# Requirements Document

## Introduction

NovelAppManager 平台（前端 Vue 3 + Vite，后端 Spring Boot 3 + MyBatis-Plus + MySQL）已具备基于 Git Worktree 的文件级工作空间隔离（见 `workspace-isolation` spec）。本功能在此基础上补齐三个关键缺口：

1. **Workspace_Context（请求级工作空间上下文）**：基于 ThreadLocal 的请求上下文，存储当前用户的 userId、workspaceId、workspacePath。这是文件路径修复和数据库隔离的共同基础设施。

2. **文件写入路径修复**：当前 `ConfigFilePreprocessor` 和所有 `AbstractConfigFileOperationService` 子类（AdConfigFileOperationService、AppConfigFileOperationService、BaseConfigFileOperationService、CommonConfigFileOperationService、PayConfigFileOperationService、PreFileOperationService、UiConfigFileOperationService）以及 `AppUploadCheckService` 均通过 `@Value("${build.workPath}")` 注入固定路径，导致工作空间内的构建操作仍然写入主仓库。修复方案：将 `buildWorkPath` 从固定字段改为从 Workspace_Context 动态读取的方法，有活跃工作空间时写入工作空间目录，否则回退到默认路径。

3. **数据库隔离**：为 9 张配置表添加 `workspace_id`（BIGINT, 默认 0），使用 MyBatis-Plus `TenantLineInnerInterceptor` 自动注入 workspace_id 条件。`workspace_id=0` 为主/共享数据，`workspace_id={userId}` 为工作空间数据。工作空间创建时复制数据行，合并时回写，销毁时清理。

### 需要隔离的 9 张配置表（Isolated_Tables）

- `ad_config`, `app_ad`, `app_common_config`, `app_pay`, `app_preview_page`, `app_ui_config`, `app_weiju_business_type`, `app_weiju_public_switch`, `novel_app`

### 不需要隔离的表

- `workspace`, `change` — 工作空间管理元数据
- `novel_user`, `sys_role`, `sys_permission`, `sys_user_role`, `sys_role_permission`, `sys_role_apply` — 用户与权限
- `api_key`, `fun_ai_app`, `fun_ai_user`, `notification`, `task_relation`, `task_template`, `user_task`, `user_task_log`, `user_op_log`

### 需要修复文件路径的类

- `AbstractConfigFileOperationService`（基类，`@Value("${build.workPath}") protected String buildWorkPath`）
- 7 个子类：`AdConfigFileOperationService`, `AppConfigFileOperationService`, `BaseConfigFileOperationService`, `CommonConfigFileOperationService`, `PayConfigFileOperationService`, `PreFileOperationService`, `UiConfigFileOperationService`
- `ConfigFilePreprocessor`（`@Value("${build.testWorkPath}") private String testWorkPath`）
- `AppUploadCheckService`（`@Value("${build.workPath}") private String buildWorkPath`）

## Glossary

- **Workspace_Context**: 基于 ThreadLocal 的请求级上下文持有者，存储当前请求关联的 userId、workspaceId 和 workspacePath，供拦截器、文件路径解析组件和业务服务读取
- **Workspace_Filter**: Servlet Filter，负责从 HTTP 请求中提取用户身份和工作空间信息，填充 Workspace_Context，并在请求完成后清理 ThreadLocal
- **Workspace_Interceptor**: 基于 MyBatis-Plus `TenantLineInnerInterceptor` 的自定义拦截器，自动为 Isolated_Table 的 SQL 注入 `workspace_id` 条件
- **Isolated_Table**: 需要数据隔离的 9 张配置表的统称，包含 `workspace_id` 字段
- **Abstract_Config_Service**: `AbstractConfigFileOperationService` 及其 7 个子类的统称，负责配置文件的创建、更新、删除操作
- **Config_File_Preprocessor**: `ConfigFilePreprocessor` 组件，在构建前更新 deliverConfigs 配置文件
- **Upload_Check_Service**: `AppUploadCheckService` 组件，在发版前检查小程序配置完整性
- **Build_Task_Processor**: `BuildTaskProcessor` 组件，执行小程序构建和打包命令，运行在异步线程中
- **Workspace_Manager**: 后端工作空间管理服务 `WorkspaceManagerService`，负责工作空间的创建、复用、销毁等全生命周期管理
- **Data_Copier**: 工作空间数据复制服务，负责在工作空间创建时将 Main_Data 复制为 Workspace_Data
- **Data_Merger**: 工作空间数据合并服务，负责将 Workspace_Data 变更回写到 Main_Data
- **Main_Data**: `workspace_id = 0` 的数据行，代表主/共享配置数据
- **Workspace_Data**: `workspace_id = {userId}` 的数据行，代表用户工作空间中的配置副本
- **Workspace_Path**: 用户工作空间的文件系统路径，格式为 `/workspaces/user_{userId}`
- **Workspace_UI**: 前端工作空间交互组件，包含 workspaceStore 和 request 拦截器

## Requirements

### Requirement 1: 请求级工作空间上下文（Workspace_Context）

**User Story:** 作为平台开发者，我希望每个 HTTP 请求自动携带工作空间上下文信息，以便拦截器和文件路径解析组件无需修改业务代码即可获取当前用户的工作空间信息。

#### Acceptance Criteria

1. THE Workspace_Context SHALL use a ThreadLocal variable to store the current request's userId (Long), workspaceId (Long, 0 for main), and workspacePath (String, null for main).
2. THE Workspace_Context SHALL provide static accessor methods `getCurrentUserId()`, `getCurrentWorkspaceId()`, and `getCurrentWorkspacePath()` that any component can call without constructor injection.
3. THE Workspace_Context SHALL provide a static `clear()` method that removes all ThreadLocal values to prevent memory leaks.
4. THE Workspace_Context SHALL provide a static `set(userId, workspaceId, workspacePath)` method for populating the context.
5. WHEN `getCurrentWorkspaceId()` is called and no context has been set, THE Workspace_Context SHALL return 0 (representing Main_Data).
6. WHEN `getCurrentWorkspacePath()` is called and no context has been set, THE Workspace_Context SHALL return null (representing the default build path).

### Requirement 2: Servlet Filter 填充 Workspace_Context

**User Story:** 作为平台开发者，我希望 Workspace_Context 在每个请求开始时自动填充、请求结束时自动清理，以便业务代码无需关心上下文的生命周期管理。

#### Acceptance Criteria

1. WHEN an HTTP request arrives, THE Workspace_Filter SHALL extract the userId from the JWT token in the Authorization header.
2. WHEN the HTTP request contains an `X-Workspace-Id` header with a non-zero value, THE Workspace_Filter SHALL look up the corresponding workspace record from the database and populate Workspace_Context with the userId, workspaceId, and workspacePath.
3. WHEN the HTTP request does not contain an `X-Workspace-Id` header or the header value is 0, THE Workspace_Filter SHALL populate Workspace_Context with userId and workspaceId=0 and workspacePath=null.
4. WHEN the HTTP request processing completes (including both success and error paths), THE Workspace_Filter SHALL call `Workspace_Context.clear()` in a finally block to prevent ThreadLocal leaks.
5. THE Workspace_Filter SHALL be registered with a filter order that executes after JWT authentication but before any business controller.
6. IF the JWT token is missing or invalid, THEN THE Workspace_Filter SHALL skip Workspace_Context population and allow the request to proceed to the authentication error handler.

### Requirement 3: 异步线程 Workspace_Context 传播

**User Story:** 作为平台开发者，我希望异步线程（如构建任务）能继承父线程的 Workspace_Context，以便 Config_File_Preprocessor 和 Workspace_Interceptor 在异步线程中也能正确解析工作空间路径和 workspace_id。

#### Acceptance Criteria

1. THE Workspace_Context SHALL provide a `capture()` method that returns a snapshot of the current thread's context (userId, workspaceId, workspacePath) as an immutable object.
2. THE Workspace_Context SHALL provide a `restore(snapshot)` method that populates the current thread's ThreadLocal with the captured snapshot values.
3. WHEN Build_Task_Processor dispatches a build task to an asynchronous thread, THE Build_Task_Processor SHALL call `capture()` in the parent thread and `restore(snapshot)` at the beginning of the child thread, followed by `clear()` when the child thread completes.
4. THE Workspace_Context snapshot SHALL be safe to pass between threads without synchronization issues.

### Requirement 4: 文件写入路径工作空间感知（AbstractConfigFileOperationService）

**User Story:** 作为平台用户，我希望在工作空间中执行配置文件操作时，文件写入到我的工作空间目录而非主仓库目录，以便操作不影响主仓库文件。

#### Acceptance Criteria

1. THE Abstract_Config_Service SHALL replace the `@Value("${build.workPath}") protected String buildWorkPath` field with a `protected String getBuildWorkPath()` method that reads from Workspace_Context.
2. WHEN `getBuildWorkPath()` is called and Workspace_Context contains a non-null workspacePath, THE Abstract_Config_Service SHALL return the workspacePath value.
3. WHEN `getBuildWorkPath()` is called and Workspace_Context contains a null workspacePath, THE Abstract_Config_Service SHALL return the default `build.workPath` value from application properties.
4. THE Abstract_Config_Service SHALL retain the default `build.workPath` value as a private field injected via `@Value` for use as the fallback path.
5. ALL 7 subclasses of Abstract_Config_Service (AdConfigFileOperationService, AppConfigFileOperationService, BaseConfigFileOperationService, CommonConfigFileOperationService, PayConfigFileOperationService, PreFileOperationService, UiConfigFileOperationService) SHALL replace all direct references to the `buildWorkPath` field with calls to `getBuildWorkPath()`.

### Requirement 5: 文件写入路径工作空间感知（ConfigFilePreprocessor）

**User Story:** 作为平台用户，我希望在工作空间中执行构建预处理时，配置文件读写操作指向我的工作空间目录，以便预处理不影响主仓库文件。

#### Acceptance Criteria

1. THE Config_File_Preprocessor SHALL replace the direct use of `testWorkPath` field with a `getTestWorkPath()` method that reads from Workspace_Context.
2. WHEN `getTestWorkPath()` is called and Workspace_Context contains a non-null workspacePath, THE Config_File_Preprocessor SHALL return the workspacePath value.
3. WHEN `getTestWorkPath()` is called and Workspace_Context contains a null workspacePath, THE Config_File_Preprocessor SHALL return the default `build.testWorkPath` value from application properties.
4. WHEN Config_File_Preprocessor executes a `git checkout` command to restore a config file, THE Config_File_Preprocessor SHALL set the working directory of the process to the value returned by `getTestWorkPath()`.

### Requirement 6: 文件写入路径工作空间感知（AppUploadCheckService）

**User Story:** 作为平台用户，我希望在工作空间中执行发版前检查时，检查操作读取我的工作空间目录中的配置文件，以便检查结果反映工作空间的实际状态。

#### Acceptance Criteria

1. THE Upload_Check_Service SHALL replace the `@Value("${build.workPath}") private String buildWorkPath` field with a `getBuildWorkPath()` method that reads from Workspace_Context.
2. WHEN `getBuildWorkPath()` is called and Workspace_Context contains a non-null workspacePath, THE Upload_Check_Service SHALL return the workspacePath value.
3. WHEN `getBuildWorkPath()` is called and Workspace_Context contains a null workspacePath, THE Upload_Check_Service SHALL return the default `build.workPath` value from application properties.

### Requirement 7: 构建任务工作目录感知

**User Story:** 作为平台用户，我希望在工作空间中触发构建时，构建命令在我的工作空间目录中执行，以便构建产物输出到工作空间而非主仓库。

#### Acceptance Criteria

1. WHEN a build task is dispatched to an asynchronous thread, THE Build_Task_Processor SHALL propagate the Workspace_Context to the build thread using the capture/restore mechanism.
2. WHEN Build_Task_Processor executes a build command (e.g., `npm run build:wx-changjiang`), THE Build_Task_Processor SHALL set the working directory of the process to the value from Workspace_Context, falling back to the default build path when no workspace is active.
3. WHEN Build_Task_Processor executes a `git checkout` command to restore config files after build, THE Build_Task_Processor SHALL execute the command within the workspace directory obtained from Workspace_Context.

### Requirement 8: 配置表 workspace_id 字段添加

**User Story:** 作为平台开发者，我希望 9 张配置表具备 workspace_id 字段，以便通过同库同表的方式实现数据逻辑隔离。

#### Acceptance Criteria

1. THE database migration SHALL add a `workspace_id` column of type BIGINT NOT NULL with default value 0 to each of the 9 Isolated_Tables: `ad_config`, `app_ad`, `app_common_config`, `app_pay`, `app_preview_page`, `app_ui_config`, `app_weiju_business_type`, `app_weiju_public_switch`, `novel_app`.
2. THE database migration SHALL create an index on the `workspace_id` column for each Isolated_Table to ensure query performance.
3. WHEN `workspace_id = 0`, THE data row SHALL represent Main_Data shared by all users.
4. WHEN `workspace_id = {userId}`, THE data row SHALL represent Workspace_Data belonging to that user's workspace.
5. THE database migration SHALL be provided as a SQL migration script that can be executed idempotently (using `IF NOT EXISTS` or equivalent guards).

### Requirement 9: MyBatis-Plus 工作空间拦截器

**User Story:** 作为平台开发者，我希望所有对隔离表的查询自动注入 workspace_id 条件，以便无需修改任何业务代码即可实现数据隔离。

#### Acceptance Criteria

1. THE Workspace_Interceptor SHALL extend MyBatis-Plus `TenantLineInnerInterceptor` and override `getTenantId()` to return the workspace_id value from Workspace_Context.
2. THE Workspace_Interceptor SHALL override `getTenantIdColumn()` to return the string `"workspace_id"`.
3. THE Workspace_Interceptor SHALL override `ignoreTable(String tableName)` to return true for tables not in the Isolated_Table list, allowing those tables to operate without workspace_id filtering.
4. THE Workspace_Interceptor SHALL automatically append `workspace_id = {currentWorkspaceId}` condition to all SELECT, UPDATE, and DELETE statements targeting Isolated_Tables.
5. THE Workspace_Interceptor SHALL automatically set `workspace_id = {currentWorkspaceId}` for all INSERT statements targeting Isolated_Tables.
6. WHEN no workspace is active in Workspace_Context (workspaceId is 0), THE Workspace_Interceptor SHALL use `workspace_id = 0` as the default condition, routing queries to Main_Data.
7. THE Workspace_Interceptor SHALL be registered as a MyBatis-Plus plugin in the Spring configuration class, added to the `MybatisPlusInterceptor` plugin chain.

### Requirement 10: 工作空间数据创建（行复制）

**User Story:** 作为平台用户，我希望创建工作空间时系统自动为我复制一份配置数据，以便我修改应用配置、广告设置等数据时不影响其他用户。

#### Acceptance Criteria

1. WHEN a Workspace is created for a user, THE Data_Copier SHALL execute `INSERT INTO {table} (..., workspace_id) SELECT ..., {userId} FROM {table} WHERE workspace_id = 0` for each of the 9 Isolated_Tables.
2. THE Data_Copier SHALL complete the data copy operation for all 9 Isolated_Tables within a single database transaction to ensure atomicity.
3. IF the data copy operation fails for any table, THEN THE Data_Copier SHALL roll back the entire transaction and propagate the error to the caller so that the Workspace_Manager can clean up the associated Worktree.
4. THE Data_Copier SHALL record the workspace creation timestamp as the `db_baseline_version` in the workspace record for later conflict detection during merge.

### Requirement 11: 工作空间数据合并

**User Story:** 作为平台用户，我希望审批通过后工作空间中的配置变更自动同步回主数据，以便其他用户能看到我的修改。

#### Acceptance Criteria

1. WHEN a merge is approved, THE Data_Merger SHALL compare Workspace_Data (`workspace_id = userId`) against Main_Data (`workspace_id = 0`) for each Isolated_Table and identify all rows that were inserted, updated, or deleted since the workspace was created (using `db_baseline_version` as the comparison baseline).
2. THE Data_Merger SHALL apply the identified changes to Main_Data (`workspace_id = 0`) within a single database transaction to ensure atomicity.
3. IF a data conflict is detected during merge (a Main_Data row was modified by another user after the workspace's `db_baseline_version`), THEN THE Data_Merger SHALL abort the transaction, report the conflicting table name and row identifiers, and leave Main_Data unchanged.
4. WHEN the data merge succeeds, THE Data_Merger SHALL refresh the Workspace_Data by re-copying the updated Main_Data rows to `workspace_id = userId` so the user's workspace reflects the merged state.

### Requirement 12: 工作空间数据清理

**User Story:** 作为平台运维人员，我希望工作空间销毁时其配置数据也被清理，以便释放数据库存储空间。

#### Acceptance Criteria

1. WHEN a Workspace is destroyed (either manually or by the scheduled reclamation task), THE Workspace_Manager SHALL execute `DELETE FROM {table} WHERE workspace_id = {userId}` for each of the 9 Isolated_Tables.
2. THE Workspace_Manager SHALL execute the data cleanup within a single database transaction to ensure atomicity.
3. IF deleting workspace data fails for any table, THEN THE Workspace_Manager SHALL log the error with the table name and userId, and add the userId to a cleanup retry queue for later processing.

### Requirement 13: 前端请求工作空间标识传递

**User Story:** 作为平台开发者，我希望前端请求自动携带工作空间标识，以便后端 Workspace_Filter 能正确填充 Workspace_Context。

#### Acceptance Criteria

1. WHEN a user has an active Workspace, THE Workspace_UI request interceptor SHALL include an `X-Workspace-Id` header with the workspace identifier value in every API request sent via the axios instance.
2. WHEN a user has no active Workspace, THE Workspace_UI request interceptor SHALL omit the `X-Workspace-Id` header, causing the backend to use `workspace_id = 0` (Main_Data) and default file paths.
3. THE Workspace_UI SHALL store the current workspace identifier in the workspaceStore upon login or workspace creation, and clear it upon workspace destruction.
4. THE `X-Workspace-Id` header injection SHALL be implemented in the existing axios request interceptor in `request.js`, reading the value from the workspaceStore.

### Requirement 14: workspace 表 db_baseline_version 字段

**User Story:** 作为平台开发者，我希望 workspace 表记录数据基线版本，以便数据合并时能检测冲突。

#### Acceptance Criteria

1. THE database migration SHALL add a `db_baseline_version` column of type DATETIME to the `workspace` table, representing the timestamp when the Workspace_Data was last synchronized with Main_Data.
2. WHEN a Workspace is created, THE Workspace_Manager SHALL set `db_baseline_version` to the current timestamp.
3. WHEN a data merge succeeds and Workspace_Data is refreshed, THE Workspace_Manager SHALL update `db_baseline_version` to the current timestamp.
4. WHEN the Data_Merger checks for conflicts, THE Data_Merger SHALL compare the `updated_at` timestamps of Main_Data rows against the workspace's `db_baseline_version` to identify rows modified by other users since the workspace data was last synchronized.
