# Requirements Document

## Introduction

NovelAppManager 是一个小程序工厂平台，当前所有用户的操作（创建、编辑、构建、打包小程序）都在同一个 Git 仓库上进行，多人同时操作会产生冲突。本功能引入"工作空间隔离"机制，使用 Git Worktree 为每个用户创建独立的工作空间，实现操作隔离、并行构建、变更审批和冲突处理，从根本上解决多人协作冲突问题。

## Glossary

- **Workspace_Manager**: 后端工作空间管理服务，负责工作空间的创建、复用、同步、销毁等全生命周期管理
- **Change_Manager**: 后端变更单元管理服务，负责变更单元（change_id）的创建、状态流转、审批流程
- **Merge_Engine**: 后端合并引擎，负责将用户分支合并到主库，包含冲突检测、全局锁控制
- **Conflict_Resolver**: 后端冲突解析服务，负责解析结构化配置文件的字段级冲突并返回冲突详情
- **Sync_Service**: 后端同步服务，负责检测工作空间与主库的差异并执行同步
- **Notification_Service**: 后端通知服务，通过 WebSocket 向前端推送工作空间状态变更消息
- **Workspace_UI**: 前端工作空间交互组件，包含状态栏、冲突处理面板、审批管理界面
- **Main_Repository**: 主 Git 仓库的 main 分支，作为所有小程序代码的唯一真相源
- **Worktree**: Git Worktree 创建的独立工作目录，每个用户对应一个
- **Template_Node_Modules**: 预装好依赖的模板 node_modules 目录，用于硬链接复制到用户工作空间
- **Change_Unit**: 变更单元，用 change_id 标识，代表用户一次提交合并的完整变更集
- **Global_Git_Lock**: 全局 Git 操作锁，确保写入主库的操作（merge、fetch、gc）串行执行
- **Dry_Run**: 预检测操作，在实际合并前检测工作空间是否落后于主库及是否存在冲突
- **Hardlink_Copy**: 使用 `cp -al` 命令创建的硬链接副本，文件共享底层数据但目录结构独立

## Requirements

### Requirement 1: 工作空间创建

**User Story:** 作为平台用户，我希望登录后系统自动为我分配独立的工作空间，以便我可以在不影响其他用户的情况下进行小程序操作。

#### Acceptance Criteria

1. WHEN a user logs in and no Workspace exists for that user, THE Workspace_Manager SHALL create a new Worktree from Main_Repository using `git worktree add` and record the workspace in the database with status `active`.
2. WHEN a new Worktree is created, THE Workspace_Manager SHALL copy Template_Node_Modules to the Worktree directory using Hardlink_Copy (`cp -al`) to provide an independent node_modules directory.
3. WHEN Hardlink_Copy completes, THE Workspace_Manager SHALL verify that the Worktree directory contains a valid node_modules directory before marking workspace creation as successful.
4. THE Workspace_Manager SHALL complete workspace creation (Worktree creation + Hardlink_Copy) within 5 seconds under normal conditions.
5. IF Worktree creation fails due to a Git error, THEN THE Workspace_Manager SHALL log the error details and return a descriptive error message to the user.
6. IF Hardlink_Copy fails, THEN THE Workspace_Manager SHALL remove the partially created Worktree and return a descriptive error message to the user.

### Requirement 2: 工作空间复用

**User Story:** 作为平台用户，我希望再次登录时能直接进入之前的工作空间，以便继续之前的工作而不丢失进度。

#### Acceptance Criteria

1. WHEN a user logs in and an active Workspace already exists for that user, THE Workspace_Manager SHALL reuse the existing Workspace without creating a new one.
2. WHEN an existing Workspace is reused, THE Workspace_Manager SHALL update the `last_active_time` field to the current timestamp.
3. WHEN an existing Workspace is reused, THE Sync_Service SHALL perform a Dry_Run check to detect whether the Workspace is behind Main_Repository.

### Requirement 3: 主库代码同步

**User Story:** 作为平台用户，我希望我的工作空间能自动同步主库的最新代码，以便我始终基于最新的代码基线进行操作。

#### Acceptance Criteria

1. WHEN Dry_Run detects that the Workspace is not behind Main_Repository, THE Sync_Service SHALL allow the user to proceed without any notification.
2. WHEN Dry_Run detects that the Workspace is behind Main_Repository and no content conflicts exist, THE Sync_Service SHALL automatically merge the latest Main_Repository changes into the Workspace without user intervention.
3. WHEN Dry_Run detects that the Workspace is behind Main_Repository and content conflicts exist, THE Sync_Service SHALL notify the user and present the conflict details through Workspace_UI.
4. THE Sync_Service SHALL execute the Dry_Run check before any actual merge operation to avoid leaving the Workspace in an inconsistent state.
5. THE Sync_Service SHALL implement idempotent protection on the sync endpoint to prevent duplicate sync operations when a user triggers multiple requests within 30 seconds.

### Requirement 4: 并行构建与打包

**User Story:** 作为平台用户，我希望在自己的工作空间内独立执行构建和打包操作，以便不需要等待其他用户的构建完成。

#### Acceptance Criteria

1. THE Workspace_Manager SHALL ensure each Workspace has its own independent node_modules directory so that build cache files (such as `.vite/`) do not interfere across Workspaces.
2. WHEN a build or publish request is received, THE Workspace_Manager SHALL resolve the requesting user's workspace_path and execute the build or publish command within that Workspace directory.
3. WHEN multiple users trigger build operations simultaneously, THE Workspace_Manager SHALL execute all builds in parallel without queuing.
4. THE Workspace_Manager SHALL pass the user's workspace_path to existing build and publish interfaces by adding a `userId` parameter, keeping other logic unchanged.

### Requirement 5: 变更单元管理

**User Story:** 作为平台用户，我希望提交合并时系统为我的变更生成唯一标识，以便追踪变更的审批和合并状态。

#### Acceptance Criteria

1. WHEN a user submits a merge request, THE Change_Manager SHALL create a new Change_Unit with a unique change_id, associate it with the user's Workspace branch, and set its status to `submitted`.
2. THE Change_Manager SHALL record the following fields for each Change_Unit: change_id, user_id, branch name, submitted_at timestamp, and status.
3. THE Change_Manager SHALL enforce the following status transitions for a Change_Unit: `editing` → `submitted` → `approved` or `rejected`.
4. IF a user attempts to submit a merge request while a previous Change_Unit for the same Workspace is in `submitted` status, THEN THE Change_Manager SHALL reject the new submission and inform the user that a pending change already exists.

### Requirement 6: 冲突检测与合并

**User Story:** 作为平台用户，我希望系统在合并前自动检测冲突，以便我能在合并失败前了解并处理冲突。

#### Acceptance Criteria

1. WHEN a merge request is submitted, THE Merge_Engine SHALL acquire the Global_Git_Lock before executing any merge operation on Main_Repository.
2. WHEN the Global_Git_Lock is acquired, THE Merge_Engine SHALL perform a trial merge to detect content conflicts before committing.
3. WHEN no content conflicts are detected, THE Merge_Engine SHALL create a pending approval record for the Change_Unit.
4. WHEN content conflicts are detected in structured configuration files (JSON/YAML), THE Merge_Engine SHALL abort the merge, restore the Workspace to its pre-merge state, and return field-level conflict details.
5. WHEN content conflicts are detected in non-structured files (templates, code files), THE Merge_Engine SHALL abort the merge and instruct the user to contact an administrator for manual resolution.
6. THE Merge_Engine SHALL release the Global_Git_Lock after the merge operation completes, regardless of success or failure.

### Requirement 7: 冲突处理

**User Story:** 作为平台用户，我希望能在网页上直观地查看和解决配置文件冲突，以便快速完成合并。

#### Acceptance Criteria

1. WHEN content conflicts are returned by the Merge_Engine, THE Conflict_Resolver SHALL parse each conflicting structured configuration file and extract field-level differences, including the Main_Repository value and the user's value for each conflicting field.
2. WHEN the user selects a resolution for each conflicting field (use Main_Repository value or keep user value), THE Conflict_Resolver SHALL apply the selected resolutions to the Workspace files.
3. WHEN all conflicts in a Change_Unit are resolved, THE Conflict_Resolver SHALL allow the user to resubmit the merge request.
4. THE Workspace_UI SHALL display a conflict resolution panel showing each conflicting field with the Main_Repository value and the user's value side by side, along with selection buttons for each field.

### Requirement 8: 审批流程

**User Story:** 作为管理员，我希望能审批用户提交的变更，以便控制合并到主库的代码质量。

#### Acceptance Criteria

1. WHEN a Change_Unit is in `submitted` status, THE Change_Manager SHALL make it visible to administrators in the approval queue.
2. WHEN an administrator approves a Change_Unit, THE Merge_Engine SHALL acquire the Global_Git_Lock and merge the Change_Unit branch into Main_Repository.
3. WHEN the merge to Main_Repository succeeds, THE Change_Manager SHALL update the Change_Unit status to `approved`.
4. WHEN an administrator rejects a Change_Unit, THE Change_Manager SHALL update the Change_Unit status to `rejected` and record the rejection reason.
5. IF the merge to Main_Repository fails during approval, THEN THE Change_Manager SHALL roll back the Change_Unit status and return an error to the administrator.
6. WHEN a Change_Unit is approved and merged, THE Workspace_Manager SHALL rebase the user's Workspace to the latest Main_Repository baseline.
7. WHEN a Change_Unit is approved and merged, THE Notification_Service SHALL send a WebSocket notification to the user indicating that the change has been merged and the Workspace has been updated.

### Requirement 9: 工作空间生命周期管理

**User Story:** 作为平台运维人员，我希望系统自动回收长期不活跃的工作空间，以便释放服务器磁盘资源。

#### Acceptance Criteria

1. THE Workspace_Manager SHALL update the `last_active_time` field of a Workspace each time the user performs an operation (edit, build, submit, or any API call that uses the Workspace).
2. THE Workspace_Manager SHALL run a scheduled task to identify Workspaces where `last_active_time` is older than 7 days.
3. WHEN a Workspace is identified for reclamation and has no uncommitted changes, THE Workspace_Manager SHALL remove the Worktree using `git worktree remove` and update the workspace status to `destroyed`.
4. WHEN a Workspace is identified for reclamation but has uncommitted changes, THE Workspace_Manager SHALL send a notification to the user reminding them to submit their changes, and skip reclamation for that Workspace.
5. WHEN any API request arrives and the requesting user's Workspace does not exist or has status `destroyed`, THE Workspace_Manager SHALL automatically create a new Workspace for the user transparently.

### Requirement 10: 全局 Git 操作串行队列

**User Story:** 作为平台开发者，我希望所有写入主库的 Git 操作串行执行，以便避免 Git 锁冲突导致数据损坏。

#### Acceptance Criteria

1. THE Merge_Engine SHALL use a Global_Git_Lock (ReentrantLock for single-node deployment) to serialize all operations that write to Main_Repository, including merge, fetch, and garbage collection.
2. WHILE the Global_Git_Lock is held by one operation, THE Merge_Engine SHALL queue subsequent write operations until the lock is released.
3. THE Merge_Engine SHALL execute Git garbage collection (`git gc --auto`) as a scheduled task during off-peak hours (e.g., 3:00 AM) while holding the Global_Git_Lock.
4. THE Workspace_Manager SHALL ensure that user-level operations (git add, git commit, git status, git diff within a Worktree) execute independently without acquiring the Global_Git_Lock.

### Requirement 11: 网页端工作空间状态展示

**User Story:** 作为平台用户，我希望在网页上随时看到我的工作空间状态，以便了解当前的工作进度和待处理事项。

#### Acceptance Criteria

1. THE Workspace_UI SHALL display a persistent status bar at the top of the page showing the current Workspace status, uncommitted change indicator, last operation timestamp, and action buttons (submit merge, discard).
2. WHEN no Workspace exists for the current user, THE Workspace_UI SHALL display a "Start Editing" button that triggers Workspace creation upon click.
3. WHEN the Workspace has uncommitted changes, THE Workspace_UI SHALL display an "Uncommitted Changes" indicator in the status bar.
4. WHEN a Change_Unit is approved and merged, THE Workspace_UI SHALL display a toast notification informing the user that the change has been merged and the Workspace has been updated to the latest baseline.

### Requirement 12: WebSocket 实时通知

**User Story:** 作为平台用户，我希望在审批结果出来后立即收到通知，以便及时了解变更状态而不需要手动刷新页面。

#### Acceptance Criteria

1. WHEN a Change_Unit status changes (approved, rejected, or conflict detected), THE Notification_Service SHALL push a WebSocket message to the affected user's connected client.
2. WHEN the user's WebSocket connection is not active at the time of notification, THE Notification_Service SHALL store the notification and deliver it when the user reconnects.
3. THE Notification_Service SHALL include the change_id, new status, and a human-readable message in each WebSocket notification payload.

### Requirement 13: 模板 node_modules 维护

**User Story:** 作为平台运维人员，我希望模板 node_modules 能在依赖变更时自动更新，以便新创建的工作空间始终使用最新的依赖。

#### Acceptance Criteria

1. WHEN the package.json or package-lock.json file in Main_Repository changes, THE Workspace_Manager SHALL regenerate Template_Node_Modules by running `npm install` in the template directory.
2. THE Workspace_Manager SHALL ensure that Template_Node_Modules regeneration does not affect existing Workspaces that are currently in use.
3. THE Workspace_Manager SHALL use the newly generated Template_Node_Modules for all Workspace creation operations after regeneration completes.

### Requirement 14: 数据库表结构

**User Story:** 作为平台开发者，我希望有明确的数据库表结构来持久化工作空间和变更单元信息，以便系统重启后状态不丢失。

#### Acceptance Criteria

1. THE Workspace_Manager SHALL persist workspace records in a `workspace` table containing at minimum: user_id, workspace_path, branch_name, status, last_active_time, created_at, and updated_at fields.
2. THE Change_Manager SHALL persist change records in a `change` table containing at minimum: change_id, user_id, branch_name, submitted_at, approved_at, status, and reject_reason fields.
3. THE Workspace_Manager SHALL enforce a one-to-one relationship between a user and an active Workspace (one user has at most one active Workspace at any time).

### Requirement 15: 审批通过后工作空间基线切换

**User Story:** 作为平台用户，我希望审批通过后工作空间自动切换到最新基线，以便后续操作基于最新代码进行，减少下次提交时的冲突概率。

#### Acceptance Criteria

1. WHEN a Change_Unit is approved and merged to Main_Repository, THE Workspace_Manager SHALL switch the user's Worktree branch to the latest Main_Repository baseline.
2. THE Workspace_Manager SHALL keep the Worktree directory path unchanged during baseline switch so that the user's workflow is not disrupted.
3. IF the baseline switch fails, THEN THE Workspace_Manager SHALL log the error and allow the Sync_Service to repair the Workspace on the user's next login.
