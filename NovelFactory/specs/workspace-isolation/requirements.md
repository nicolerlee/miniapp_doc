# Requirements Document

## Introduction

NovelAppManager 是一个小程序工厂平台（前端 Vue 3 + Vite，后端 Spring Boot 3 + MyBatis-Plus + MySQL）。当前系统已具备基于 Git Worktree 的文件级工作空间隔离，但存在两个关键缺口：

1. **数据库隔离缺失**：所有用户共享同一个数据库 `db_novelapp_agent`，用户在工作空间中修改应用配置、广告设置、支付配置等数据时，变更直接写入共享数据库，影响其他用户。
2. **文件写入路径错误**：`ConfigFilePreprocessor` 等组件在写入配置文件时使用全局 `build.workPath` / `build.testWorkPath`，而非用户工作空间路径，导致工作空间内的构建操作仍然修改主仓库文件。

本功能通过引入 **workspace_id 字段 + MyBatis-Plus 拦截器** 实现数据库级隔离：仅对 9 张配置相关表添加 `workspace_id` 字段（BIGINT, 默认 0），通过自定义 `TenantLineInnerInterceptor` 拦截器自动注入 `workspace_id` 条件，实现同库同表的逻辑隔离。`workspace_id = 0` 表示主/共享数据，`workspace_id = {userId}` 表示用户工作空间数据。结合 **工作空间感知的文件写入路径**，实现完整的工作空间级隔离。合并审批通过后，工作空间数据行覆盖回 `workspace_id = 0`。

### 需要隔离的 9 张配置表

- `ad_config`
- `app_ad`
- `app_common_config`
- `app_pay`
- `app_preview_page`
- `app_ui_config`
- `app_weiju_business_type`
- `app_weiju_public_switch`
- `novel_app`

### 不需要隔离的表（始终读写主数据）

- `workspace`, `change` — 工作空间管理元数据
- `novel_user`, `sys_role`, `sys_permission`, `sys_user_role`, `sys_role_permission`, `sys_role_apply` — 用户与权限
- `api_key` — API 密钥
- `fun_ai_app`, `fun_ai_user` — AI 相关
- `notification` — 通知
- `task_relation`, `task_template`, `user_task`, `user_task_log` — 任务
- `user_op_log` — 操作日志

## Glossary

- **Workspace_Manager**: 后端工作空间管理服务，负责工作空间的创建、复用、同步、销毁等全生命周期管理
- **Workspace_Interceptor**: 基于 MyBatis-Plus `TenantLineInnerInterceptor` 的自定义拦截器，自动为隔离表的 SQL 注入 `workspace_id` 条件
- **Workspace_Context**: 请求级上下文持有者（基于 ThreadLocal），存储当前请求关联的 userId 和 workspaceId，供 Workspace_Interceptor 和路径解析组件读取
- **Isolated_Table**: 需要数据隔离的 9 张配置表的统称，包含 `workspace_id` 字段
- **Config_File_Preprocessor**: 后端配置文件预处理器，在构建前更新 deliverConfigs 配置文件
- **Build_Task_Processor**: 后端构建任务处理器，执行小程序构建和打包命令
- **Change_Manager**: 后端变更单元管理服务，负责变更单元的创建、状态流转、审批流程
- **Merge_Engine**: 后端合并引擎，负责将用户分支合并到主库，包含冲突检测、全局锁控制
- **Data_Merger**: 后端数据合并服务，负责将工作空间数据行（workspace_id = userId）覆盖回主数据行（workspace_id = 0）
- **Conflict_Resolver**: 后端冲突解析服务，负责解析结构化配置文件的字段级冲突
- **Sync_Service**: 后端同步服务，负责检测工作空间与主库的差异并执行同步
- **Notification_Service**: 后端通知服务，通过 WebSocket 向前端推送工作空间状态变更消息
- **Workspace_UI**: 前端工作空间交互组件，包含状态栏、冲突处理面板、审批管理界面
- **Main_Repository**: 主 Git 仓库的 main 分支，作为所有小程序代码的唯一真相源
- **Main_Data**: workspace_id = 0 的数据行，作为所有小程序配置数据的唯一真相源
- **Workspace_Data**: workspace_id = {userId} 的数据行，代表用户工作空间中的配置副本
- **Worktree**: Git Worktree 创建的独立工作目录，每个用户对应一个
- **Template_Node_Modules**: 预装好依赖的模板 node_modules 目录，用于硬链接复制到用户工作空间
- **Change_Unit**: 变更单元，用 change_id 标识，代表用户一次提交合并的完整变更集
- **Global_Git_Lock**: 全局 Git 操作锁，确保写入主库的操作串行执行
- **Dry_Run**: 预检测操作，在实际合并前检测工作空间是否落后于主库及是否存在冲突
- **Hardlink_Copy**: 使用 `cp -al` 命令创建的硬链接副本，文件共享底层数据但目录结构独立
- **Workspace_Path**: 用户工作空间的文件系统路径，格式为 `/workspaces/user_{userId}`

## Requirements

### Requirement 1: 配置表 workspace_id 字段添加

**User Story:** 作为平台开发者，我希望 9 张配置表具备 workspace_id 字段，以便通过同库同表的方式实现数据逻辑隔离。

#### Acceptance Criteria

1. THE Workspace_Manager SHALL add a `workspace_id` column of type BIGINT with default value 0 to each of the 9 Isolated_Tables: `ad_config`, `app_ad`, `app_common_config`, `app_pay`, `app_preview_page`, `app_ui_config`, `app_weiju_business_type`, `app_weiju_public_switch`, `novel_app`.
2. THE Workspace_Manager SHALL create a database index on the `workspace_id` column for each Isolated_Table to ensure query performance.
3. WHEN `workspace_id = 0`, THE data row SHALL represent Main_Data shared by all users.
4. WHEN `workspace_id = {userId}`, THE data row SHALL represent Workspace_Data belonging to that user's workspace.

### Requirement 2: MyBatis-Plus 工作空间拦截器

**User Story:** 作为平台开发者，我希望所有对隔离表的查询自动注入 workspace_id 条件，以便无需修改任何业务代码即可实现数据隔离。

#### Acceptance Criteria

1. THE Workspace_Interceptor SHALL extend MyBatis-Plus `TenantLineInnerInterceptor` and automatically append `workspace_id = {currentWorkspaceId}` condition to all SELECT, UPDATE, and DELETE statements targeting Isolated_Tables.
2. THE Workspace_Interceptor SHALL automatically set `workspace_id = {currentWorkspaceId}` for all INSERT statements targeting Isolated_Tables.
3. THE Workspace_Interceptor SHALL read the current workspace_id value from Workspace_Context ThreadLocal.
4. WHEN no workspace is active in Workspace_Context, THE Workspace_Interceptor SHALL use `workspace_id = 0` as the default condition, routing queries to Main_Data.
5. THE Workspace_Interceptor SHALL skip interception for tables not in the Isolated_Table list, allowing those tables to operate without workspace_id filtering.
6. THE Workspace_Interceptor SHALL be registered as a MyBatis-Plus plugin in the Spring configuration class.

### Requirement 3: 请求级工作空间上下文

**User Story:** 作为平台开发者，我希望每个 HTTP 请求自动携带工作空间上下文信息，以便拦截器和文件路径解析组件无需修改业务代码即可获取当前用户的工作空间信息。

#### Acceptance Criteria

1. THE Workspace_Context SHALL use a ThreadLocal variable to store the current request's workspace_id (0 for main, userId for workspace) and userId.
2. WHEN an HTTP request arrives, a servlet filter or Spring interceptor SHALL extract the userId from the JWT token, look up the user's active Workspace, and populate the Workspace_Context before the request reaches any controller.
3. WHEN the HTTP request processing completes, the filter or interceptor SHALL clear the Workspace_Context to prevent ThreadLocal leaks.
4. THE Workspace_Context SHALL provide static accessor methods (`getCurrentWorkspaceId()`, `getCurrentUserId()`, `getCurrentWorkspacePath()`) that any component can call without constructor injection.
5. WHEN a request is processed in an asynchronous thread (such as build tasks), THE Workspace_Context SHALL propagate the workspace context to the child thread before execution begins.

### Requirement 4: 文件写入路径工作空间感知

**User Story:** 作为平台用户，我希望在工作空间中执行构建时，配置文件写入到我的工作空间目录而非主仓库目录，以便构建操作不影响主仓库文件。

#### Acceptance Criteria

1. WHEN a build or config preprocessing operation is triggered and the requesting user has an active Workspace, THE Config_File_Preprocessor SHALL resolve the file write path to the user's Workspace_Path instead of the global `build.workPath` or `build.testWorkPath`.
2. WHEN a build or config preprocessing operation is triggered and the requesting user has no active Workspace, THE Config_File_Preprocessor SHALL use the global `build.workPath` or `build.testWorkPath` as the file write path.
3. THE Config_File_Preprocessor SHALL obtain the current workspace path from Workspace_Context without requiring changes to its method signatures or callers.
4. WHEN the Build_Task_Processor executes a build command, THE Build_Task_Processor SHALL set the working directory of the build process to the user's Workspace_Path.
5. WHEN the Build_Task_Processor executes a `git checkout` command to restore config files, THE Build_Task_Processor SHALL execute the command within the user's Workspace_Path.

### Requirement 5: 工作空间数据创建（行复制）

**User Story:** 作为平台用户，我希望创建工作空间时系统自动为我复制一份配置数据，以便我修改应用配置、广告设置等数据时不影响其他用户。

#### Acceptance Criteria

1. WHEN a Workspace is created for a user, THE Workspace_Manager SHALL copy all rows from each Isolated_Table where `workspace_id = 0` and insert them with `workspace_id = {userId}`.
2. THE Workspace_Manager SHALL complete the data copy operation for all 9 Isolated_Tables within a single database transaction to ensure atomicity.
3. IF the data copy operation fails, THEN THE Workspace_Manager SHALL roll back the transaction, clean up the associated Worktree, and return a descriptive error message to the user.
4. THE Workspace_Manager SHALL record the workspace creation timestamp as the data baseline version for later conflict detection during merge.

### Requirement 6: 工作空间数据合并

**User Story:** 作为平台用户，我希望审批通过后工作空间中的配置变更自动同步回主数据，以便其他用户能看到我的修改。

#### Acceptance Criteria

1. WHEN a Change_Unit is approved, THE Data_Merger SHALL compare Workspace_Data (workspace_id = userId) against Main_Data (workspace_id = 0) for each Isolated_Table and identify all rows that were inserted, updated, or deleted since the workspace was created.
2. THE Data_Merger SHALL apply the identified changes to Main_Data (workspace_id = 0) within a single database transaction to ensure atomicity.
3. IF a data conflict is detected during merge (a Main_Data row was modified by another user after the workspace was created), THEN THE Data_Merger SHALL abort the transaction, report the conflicting table and row identifiers, and leave Main_Data unchanged.
4. WHEN the data merge succeeds, THE Data_Merger SHALL refresh the Workspace_Data by copying the updated Main_Data rows back to workspace_id = userId so the user's workspace reflects the merged state.
5. THE Data_Merger SHALL acquire a data-level merge lock before executing the merge to prevent concurrent data merges from conflicting.

### Requirement 7: 工作空间数据同步

**User Story:** 作为平台用户，我希望工作空间数据能同步主数据的最新内容，以便我始终基于最新的数据基线进行操作。

#### Acceptance Criteria

1. WHEN the Sync_Service synchronizes a Workspace and Main_Data has been updated since the Workspace_Data was last synchronized, THE Sync_Service SHALL refresh the Workspace_Data with the latest Main_Data content.
2. WHEN refreshing Workspace_Data, THE Sync_Service SHALL preserve any uncommitted changes the user has made in Workspace_Data by applying a diff-based merge strategy.
3. IF a data conflict is detected during data sync (user modified a row that was also modified in Main_Data), THEN THE Sync_Service SHALL notify the user of the conflicting records and allow the user to choose which version to keep.
4. THE Sync_Service SHALL record a `db_baseline_version` timestamp in the workspace record to track when the Workspace_Data was last synchronized with Main_Data.

### Requirement 8: 工作空间数据清理

**User Story:** 作为平台运维人员，我希望工作空间销毁时其配置数据也被清理，以便释放数据库存储空间。

#### Acceptance Criteria

1. WHEN a Workspace is destroyed (either manually or by the scheduled reclamation task), THE Workspace_Manager SHALL execute `DELETE FROM {table} WHERE workspace_id = {userId}` for each of the 9 Isolated_Tables.
2. THE Workspace_Manager SHALL execute the data cleanup within a single database transaction to ensure atomicity.
3. IF deleting workspace data fails, THEN THE Workspace_Manager SHALL log the error and add the userId to a cleanup retry queue for later processing.

### Requirement 9: 工作空间创建（文件系统）

**User Story:** 作为平台用户，我希望登录后系统自动为我分配独立的文件工作空间，以便我可以在不影响其他用户的情况下进行小程序操作。

#### Acceptance Criteria

1. WHEN a user logs in and no Workspace exists for that user, THE Workspace_Manager SHALL create a new Worktree from Main_Repository using `git worktree add` and record the workspace in the database with status `active`.
2. WHEN a new Worktree is created, THE Workspace_Manager SHALL copy Template_Node_Modules to the Worktree directory using Hardlink_Copy to provide an independent node_modules directory.
3. WHEN Hardlink_Copy completes, THE Workspace_Manager SHALL verify that the Worktree directory contains a valid node_modules directory before marking workspace creation as successful.
4. THE Workspace_Manager SHALL complete workspace creation (Worktree creation + Hardlink_Copy + data row copy) within 60 seconds under normal conditions.
5. IF Worktree creation fails due to a Git error, THEN THE Workspace_Manager SHALL log the error details and return a descriptive error message to the user.
6. IF Hardlink_Copy fails, THEN THE Workspace_Manager SHALL remove the partially created Worktree and return a descriptive error message to the user.

### Requirement 10: 工作空间复用

**User Story:** 作为平台用户，我希望再次登录时能直接进入之前的工作空间，以便继续之前的工作而不丢失进度。

#### Acceptance Criteria

1. WHEN a user logs in and an active Workspace already exists for that user, THE Workspace_Manager SHALL reuse the existing Workspace without creating a new one.
2. WHEN an existing Workspace is reused, THE Workspace_Manager SHALL update the `last_active_time` field to the current timestamp.
3. WHEN an existing Workspace is reused, THE Sync_Service SHALL perform a Dry_Run check to detect whether the Workspace is behind Main_Repository and whether the Workspace_Data is behind Main_Data.

### Requirement 11: 主库代码同步

**User Story:** 作为平台用户，我希望我的工作空间能自动同步主库的最新代码，以便我始终基于最新的代码基线进行操作。

#### Acceptance Criteria

1. WHEN Dry_Run detects that the Workspace is not behind Main_Repository, THE Sync_Service SHALL allow the user to proceed without any notification.
2. WHEN Dry_Run detects that the Workspace is behind Main_Repository and no content conflicts exist, THE Sync_Service SHALL automatically merge the latest Main_Repository changes into the Workspace without user intervention.
3. WHEN Dry_Run detects that the Workspace is behind Main_Repository and content conflicts exist, THE Sync_Service SHALL notify the user and present the conflict details through Workspace_UI.
4. THE Sync_Service SHALL execute the Dry_Run check before any actual merge operation to avoid leaving the Workspace in an inconsistent state.
5. THE Sync_Service SHALL implement idempotent protection on the sync endpoint to prevent duplicate sync operations when a user triggers multiple requests within 30 seconds.

### Requirement 12: 并行构建与打包

**User Story:** 作为平台用户，我希望在自己的工作空间内独立执行构建和打包操作，以便不需要等待其他用户的构建完成。

#### Acceptance Criteria

1. THE Workspace_Manager SHALL ensure each Workspace has its own independent node_modules directory so that build cache files do not interfere across Workspaces.
2. WHEN a build or publish request is received, THE Build_Task_Processor SHALL resolve the requesting user's Workspace_Path from Workspace_Context and execute the build or publish command within that Workspace directory.
3. WHEN multiple users trigger build operations simultaneously, THE Build_Task_Processor SHALL execute all builds in parallel without queuing.
4. WHEN a build task is dispatched to an asynchronous thread, THE Build_Task_Processor SHALL propagate the Workspace_Context to the build thread so that Config_File_Preprocessor and Workspace_Interceptor resolve the correct workspace path and workspace_id.

### Requirement 13: 变更单元管理

**User Story:** 作为平台用户，我希望提交合并时系统为我的变更生成唯一标识，以便追踪变更的审批和合并状态。

#### Acceptance Criteria

1. WHEN a user submits a merge request, THE Change_Manager SHALL create a new Change_Unit with a unique change_id, associate it with the user's Workspace branch, and set its status to `submitted`.
2. THE Change_Manager SHALL record the following fields for each Change_Unit: change_id, user_id, branch_name, submitted_at timestamp, and status.
3. THE Change_Manager SHALL enforce the following status transitions for a Change_Unit: `editing` → `submitted` → `approved` or `rejected`.
4. IF a user attempts to submit a merge request while a previous Change_Unit for the same Workspace is in `submitted` status, THEN THE Change_Manager SHALL reject the new submission and inform the user that a pending change already exists.

### Requirement 14: 冲突检测与合并

**User Story:** 作为平台用户，我希望系统在合并前自动检测冲突（包括文件冲突和数据冲突），以便我能在合并失败前了解并处理冲突。

#### Acceptance Criteria

1. WHEN a merge request is submitted, THE Merge_Engine SHALL acquire the Global_Git_Lock before executing any merge operation on Main_Repository.
2. WHEN the Global_Git_Lock is acquired, THE Merge_Engine SHALL perform a trial merge to detect content conflicts before committing.
3. WHEN no file content conflicts and no data conflicts are detected, THE Merge_Engine SHALL create a pending approval record for the Change_Unit.
4. WHEN content conflicts are detected in structured configuration files (JSON/YAML), THE Merge_Engine SHALL abort the merge, restore the Workspace to its pre-merge state, and return field-level conflict details.
5. WHEN content conflicts are detected in non-structured files, THE Merge_Engine SHALL abort the merge and instruct the user to contact an administrator for manual resolution.
6. THE Merge_Engine SHALL release the Global_Git_Lock after the merge operation completes, regardless of success or failure.
7. WHEN checking for data conflicts, THE Data_Merger SHALL compare the Main_Data modification timestamps against the workspace's db_baseline_version to identify rows modified by other users since the workspace was created.

### Requirement 15: 冲突处理

**User Story:** 作为平台用户，我希望能在网页上直观地查看和解决配置文件冲突，以便快速完成合并。

#### Acceptance Criteria

1. WHEN content conflicts are returned by the Merge_Engine, THE Conflict_Resolver SHALL parse each conflicting structured configuration file and extract field-level differences, including the Main_Repository value and the user's value for each conflicting field.
2. WHEN the user selects a resolution for each conflicting field (use Main_Repository value or keep user value), THE Conflict_Resolver SHALL apply the selected resolutions to the Workspace files.
3. WHEN all conflicts in a Change_Unit are resolved, THE Conflict_Resolver SHALL allow the user to resubmit the merge request.
4. THE Workspace_UI SHALL display a conflict resolution panel showing each conflicting field with the Main_Repository value and the user's value side by side, along with selection buttons for each field.

### Requirement 16: 审批流程

**User Story:** 作为管理员，我希望能审批用户提交的变更（包括文件变更和数据变更），以便控制合并到主库的代码和数据质量。

#### Acceptance Criteria

1. WHEN a Change_Unit is in `submitted` status, THE Change_Manager SHALL make it visible to administrators in the approval queue.
2. WHEN an administrator approves a Change_Unit, THE Merge_Engine SHALL acquire the Global_Git_Lock and merge the Change_Unit branch into Main_Repository.
3. WHEN the file merge to Main_Repository succeeds, THE Data_Merger SHALL merge the Workspace_Data changes into Main_Data.
4. WHEN both file merge and data merge succeed, THE Change_Manager SHALL update the Change_Unit status to `approved`.
5. WHEN an administrator rejects a Change_Unit, THE Change_Manager SHALL update the Change_Unit status to `rejected` and record the rejection reason.
6. IF the file merge or data merge fails during approval, THEN THE Merge_Engine SHALL roll back both the file merge and data merge, revert the Change_Unit status, and return an error to the administrator.
7. WHEN a Change_Unit is approved and merged, THE Workspace_Manager SHALL rebase the user's Workspace to the latest Main_Repository baseline.
8. WHEN a Change_Unit is approved and merged, THE Notification_Service SHALL send a WebSocket notification to the user indicating that the change has been merged and the Workspace has been updated.

### Requirement 17: 工作空间生命周期管理

**User Story:** 作为平台运维人员，我希望系统自动回收长期不活跃的工作空间及其数据，以便释放服务器磁盘和数据库资源。

#### Acceptance Criteria

1. THE Workspace_Manager SHALL update the `last_active_time` field of a Workspace each time the user performs an operation that uses the Workspace.
2. THE Workspace_Manager SHALL run a scheduled task to identify Workspaces where `last_active_time` is older than 7 days.
3. WHEN a Workspace is identified for reclamation and has no uncommitted changes, THE Workspace_Manager SHALL remove the Worktree, delete the Workspace_Data from all Isolated_Tables, and update the workspace status to `destroyed`.
4. WHEN a Workspace is identified for reclamation but has uncommitted changes, THE Workspace_Manager SHALL send a notification to the user reminding them to submit their changes, and skip reclamation for that Workspace.
5. WHEN any API request arrives and the requesting user's Workspace does not exist or has status `destroyed`, THE Workspace_Manager SHALL automatically create a new Workspace (including Workspace_Data copy) for the user transparently.

### Requirement 18: 全局 Git 操作串行队列

**User Story:** 作为平台开发者，我希望所有写入主库的 Git 操作串行执行，以便避免 Git 锁冲突导致数据损坏。

#### Acceptance Criteria

1. THE Merge_Engine SHALL use a Global_Git_Lock (ReentrantLock for single-node deployment) to serialize all operations that write to Main_Repository, including merge, fetch, and garbage collection.
2. WHILE the Global_Git_Lock is held by one operation, THE Merge_Engine SHALL queue subsequent write operations until the lock is released.
3. THE Merge_Engine SHALL execute Git garbage collection (`git gc --auto`) as a scheduled task during off-peak hours while holding the Global_Git_Lock.
4. THE Workspace_Manager SHALL ensure that user-level operations (git add, git commit, git status, git diff within a Worktree) execute independently without acquiring the Global_Git_Lock.

### Requirement 19: 网页端工作空间状态展示

**User Story:** 作为平台用户，我希望在网页上随时看到我的工作空间状态，以便了解当前的工作进度和待处理事项。

#### Acceptance Criteria

1. THE Workspace_UI SHALL display a persistent status bar at the top of the page showing the current Workspace status, data isolation indicator, uncommitted change indicator, last operation timestamp, and action buttons.
2. WHEN no Workspace exists for the current user, THE Workspace_UI SHALL display a "Start Editing" button that triggers Workspace creation upon click.
3. WHEN the Workspace has uncommitted changes, THE Workspace_UI SHALL display an "Uncommitted Changes" indicator in the status bar.
4. WHEN a Change_Unit is approved and merged, THE Workspace_UI SHALL display a toast notification informing the user that the change has been merged and the Workspace has been updated to the latest baseline.

### Requirement 20: WebSocket 实时通知

**User Story:** 作为平台用户，我希望在审批结果出来后立即收到通知，以便及时了解变更状态而不需要手动刷新页面。

#### Acceptance Criteria

1. WHEN a Change_Unit status changes (approved, rejected, or conflict detected), THE Notification_Service SHALL push a WebSocket message to the affected user's connected client.
2. WHEN the user's WebSocket connection is not active at the time of notification, THE Notification_Service SHALL store the notification and deliver it when the user reconnects.
3. THE Notification_Service SHALL include the change_id, new status, and a human-readable message in each WebSocket notification payload.

### Requirement 21: 模板 node_modules 维护

**User Story:** 作为平台运维人员，我希望模板 node_modules 能在依赖变更时自动更新，以便新创建的工作空间始终使用最新的依赖。

#### Acceptance Criteria

1. WHEN the package.json or package-lock.json file in Main_Repository changes, THE Workspace_Manager SHALL regenerate Template_Node_Modules by running `npm install` in the template directory.
2. THE Workspace_Manager SHALL ensure that Template_Node_Modules regeneration does not affect existing Workspaces that are currently in use.
3. THE Workspace_Manager SHALL use the newly generated Template_Node_Modules for all Workspace creation operations after regeneration completes.

### Requirement 22: 数据库表结构

**User Story:** 作为平台开发者，我希望有明确的数据库表结构来持久化工作空间和变更单元信息，以便系统重启后状态不丢失。

#### Acceptance Criteria

1. THE Workspace_Manager SHALL persist workspace records in a `workspace` table containing at minimum: user_id, workspace_path, branch_name, db_baseline_version, status, last_active_time, created_at, and updated_at fields.
2. THE Change_Manager SHALL persist change records in a `change` table containing at minimum: change_id, user_id, branch_name, submitted_at, approved_at, status, and reject_reason fields.
3. THE Workspace_Manager SHALL enforce a one-to-one relationship between a user and an active Workspace (one user has at most one active Workspace at any time).

### Requirement 23: 前端请求工作空间标识传递

**User Story:** 作为平台开发者，我希望前端请求自动携带工作空间标识，以便后端能正确设置 Workspace_Context 用于拦截器和文件路径解析。

#### Acceptance Criteria

1. WHEN a user has an active Workspace, THE Workspace_UI SHALL include the workspace identifier in every API request via an HTTP header (e.g., `X-Workspace-Id`).
2. WHEN a user has no active Workspace, THE Workspace_UI SHALL omit the workspace header, causing the backend to use workspace_id = 0 (Main_Data) and Main_Repository paths.
3. THE Workspace_UI SHALL store the current workspace identifier in the frontend workspace store upon login or workspace creation, and clear it upon workspace destruction.

### Requirement 24: 审批通过后工作空间基线切换

**User Story:** 作为平台用户，我希望审批通过后工作空间自动切换到最新基线（包括文件和数据），以便后续操作基于最新代码和数据进行。

#### Acceptance Criteria

1. WHEN a Change_Unit is approved and merged to Main_Repository, THE Workspace_Manager SHALL switch the user's Worktree branch to the latest Main_Repository baseline.
2. WHEN a Change_Unit is approved and merged, THE Data_Merger SHALL refresh the Workspace_Data by re-copying Main_Data rows to workspace_id = userId.
3. THE Workspace_Manager SHALL keep the Worktree directory path unchanged during baseline switch so that the user's workflow is not disrupted.
4. IF the baseline switch fails, THEN THE Workspace_Manager SHALL log the error and allow the Sync_Service to repair the Workspace on the user's next login.
