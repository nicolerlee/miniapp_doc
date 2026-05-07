# Implementation Plan: Workspace DB Isolation

## Overview

本实现计划将工作空间数据库隔离功能分解为增量式编码任务。实现顺序遵循设计文档的依赖关系：先建立 WorkspaceContext 基础设施，再修复文件路径，然后实现数据库隔离（字段迁移 → 拦截器 → 数据复制/合并/清理），最后完成前端 header 注入。每个任务构建在前一个任务之上，确保无孤立代码。

## Tasks

- [x] 1. 实现 WorkspaceContext 请求级上下文
  - [x] 1.1 创建 `WorkspaceContext` 类
    - 创建 `com.fun.novel.workspace.context.WorkspaceContext` 类
    - 实现三个 ThreadLocal 字段：`USER_ID`（默认 0L）、`WORKSPACE_ID`（默认 0L）、`WORKSPACE_PATH`（默认 null）
    - 实现 `set(userId, workspaceId, workspacePath)` 静态方法
    - 实现 `getCurrentUserId()`、`getCurrentWorkspaceId()`、`getCurrentWorkspacePath()` 静态访问器
    - 实现 `clear()` 静态方法，移除所有 ThreadLocal 值
    - 实现 `capture()` 方法返回不可变 `Snapshot` record
    - 实现 `restore(Snapshot)` 方法从快照恢复上下文
    - 使用 Java 17 record 定义 `Snapshot(Long userId, Long workspaceId, String workspacePath)`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 3.1, 3.2, 3.4_

  - [ ]* 1.2 编写 WorkspaceContext 属性测试：set/get/clear round-trip
    - **Property 1: WorkspaceContext set/get/clear round-trip**
    - 使用 jqwik 随机生成 userId/workspaceId/workspacePath 三元组
    - 验证 `set()` 后 getter 返回相同值，`clear()` 后 `getCurrentWorkspaceId()` 返回 0、`getCurrentWorkspacePath()` 返回 null
    - 创建测试类 `WorkspaceContextPropertyTest`，最少 100 次迭代
    - **Validates: Requirements 1.1, 1.3, 1.4, 1.5, 1.6**

  - [ ]* 1.3 编写 WorkspaceContext 属性测试：capture/restore 跨线程 round-trip
    - **Property 2: WorkspaceContext capture/restore cross-thread round-trip**
    - 使用 jqwik 随机生成上下文值，在主线程 set + capture，在子线程 restore 后验证 getter 返回相同值
    - 在测试类 `WorkspaceContextPropertyTest` 中添加，最少 100 次迭代
    - **Validates: Requirements 3.1, 3.2, 3.4**

- [x] 2. 实现 WorkspaceFilter Servlet 过滤器
  - [x] 2.1 创建 `WorkspaceFilter` 类
    - 创建 `com.fun.novel.workspace.filter.WorkspaceFilter`，继承 `OncePerRequestFilter`
    - 使用 `@Order(Ordered.HIGHEST_PRECEDENCE + 100)` 确保在 JWT Filter 之后执行
    - 在 `doFilterInternal` 中：从 JWT 提取 userId；读取 `X-Workspace-Id` header；查询 workspace 表获取 workspacePath
    - 有活跃工作空间时调用 `WorkspaceContext.set(userId, userId, workspacePath)`
    - 无工作空间时调用 `WorkspaceContext.set(userId, 0L, null)`
    - JWT 缺失或无效时跳过填充，放行到认证错误处理
    - 在 `finally` 块中始终调用 `WorkspaceContext.clear()`
    - 注入 `JwtUtil` 和 `WorkspaceMapper` 依赖
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6_

  - [ ]* 2.2 编写 WorkspaceFilter 单元测试
    - 创建 `WorkspaceFilterTest` 测试类
    - 测试用例：JWT 缺失时跳过填充、X-Workspace-Id header 解析（有效值/无效值/缺失）、finally 块清理验证、workspaceId=0 回退
    - 使用 Mockito mock JwtUtil 和 WorkspaceMapper
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.6_

- [x] 3. 修复文件写入路径工作空间感知
  - [x] 3.1 改造 `AbstractConfigFileOperationService` 文件路径
    - 将 `@Value("${build.workPath}") protected String buildWorkPath` 改为 `private String defaultBuildWorkPath`
    - 新增 `protected String getBuildWorkPath()` 方法：优先从 `WorkspaceContext.getCurrentWorkspacePath()` 读取，null 时回退到 `defaultBuildWorkPath`
    - 在基类中将所有 `buildWorkPath` 直接引用替换为 `getBuildWorkPath()` 调用
    - 检查并更新 7 个子类（AdConfigFileOperationService、AppConfigFileOperationService、BaseConfigFileOperationService、CommonConfigFileOperationService、PayConfigFileOperationService、PreFileOperationService、UiConfigFileOperationService）中所有 `buildWorkPath` 引用为 `getBuildWorkPath()` 调用
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_

  - [x] 3.2 改造 `ConfigFilePreprocessor` 文件路径
    - 将 `@Value("${build.testWorkPath}") private String testWorkPath` 改为 `private String defaultTestWorkPath`
    - 新增 `private String getTestWorkPath()` 方法：优先从 `WorkspaceContext.getCurrentWorkspacePath()` 读取，null 时回退到 `defaultTestWorkPath`
    - 将所有 `testWorkPath` 直接引用替换为 `getTestWorkPath()` 调用
    - 确保 `git checkout` 命令的 `processBuilder.directory()` 使用 `getTestWorkPath()` 返回值
    - _Requirements: 5.1, 5.2, 5.3, 5.4_

  - [x] 3.3 改造 `AppUploadCheckService` 文件路径
    - 将 `@Value("${build.workPath}") private String buildWorkPath` 改为 `private String defaultBuildWorkPath`
    - 新增 `private String getBuildWorkPath()` 方法：优先从 `WorkspaceContext.getCurrentWorkspacePath()` 读取，null 时回退到 `defaultBuildWorkPath`
    - 将所有 `buildWorkPath` 直接引用替换为 `getBuildWorkPath()` 调用
    - _Requirements: 6.1, 6.2, 6.3_

  - [ ]* 3.4 编写文件路径解析属性测试
    - **Property 3: Workspace-aware path resolution**
    - 使用 jqwik 随机生成 workspacePath 字符串
    - 验证：WorkspaceContext 有非 null workspacePath 时，`getBuildWorkPath()` / `getTestWorkPath()` 返回该值；null 时返回默认 `@Value` 路径
    - 创建测试类 `WorkspacePathResolutionPropertyTest`，最少 100 次迭代
    - 需要通过反射或测试子类设置 defaultBuildWorkPath / defaultTestWorkPath 值
    - **Validates: Requirements 4.2, 4.3, 5.2, 5.3, 6.2, 6.3**

- [ ] 4. Checkpoint - 确保基础设施层测试通过
  - 确保所有测试通过，ask the user if questions arise.

- [x] 5. 实现 BuildTaskProcessor 异步上下文传播
  - [x] 5.1 修改 `BuildTaskProcessor` 支持 WorkspaceContext 传播
    - 在 `process()` 方法开头添加 `WorkspaceContext.Snapshot snapshot = WorkspaceContext.capture()`
    - 在异步执行（`CompletableFuture.runAsync` 或 `novelAppBuildUtil.buildNovelAppWithTaskId`）前，将 snapshot 传递到异步线程
    - 在异步线程开头调用 `WorkspaceContext.restore(snapshot)`，在 finally 块中调用 `WorkspaceContext.clear()`
    - 注意：当前 `process()` 本身已在异步线程中被调用，需要在任务调度层（提交任务的 Controller/Service）完成 capture，在 `process()` 中 restore
    - 确保构建命令的工作目录使用 `WorkspaceContext.getCurrentWorkspacePath()` 回退到默认路径
    - _Requirements: 3.3, 7.1, 7.2, 7.3_

  - [ ]* 5.2 编写 BuildTaskProcessor 上下文传播单元测试
    - 验证 capture/restore 在异步线程中正确传播 WorkspaceContext
    - 验证构建命令使用正确的工作目录
    - 使用 Mockito mock NovelAppBuildUtil 和 ConfigFilePreprocessor
    - _Requirements: 7.1, 7.2, 7.3_

- [x] 6. 数据库 Schema 迁移与 Entity 更新
  - [x] 6.1 创建 SQL 迁移脚本
    - 创建迁移脚本文件 `NovelAppManagerServer/src/main/resources/db/migration/V1__workspace_db_isolation.sql`
    - 为 workspace 表添加 `db_baseline_version DATETIME DEFAULT NULL` 字段
    - 为 9 张隔离表（ad_config, app_ad, app_common_config, app_pay, app_preview_page, app_ui_config, app_weiju_business_type, app_weiju_public_switch, novel_app）各添加 `workspace_id BIGINT NOT NULL DEFAULT 0` 字段
    - 为每张隔离表创建 `idx_workspace_id` 索引
    - 注意 MySQL 8 不支持 `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`，使用存储过程或条件判断实现幂等性
    - 注意 `change` 表使用反引号转义（但 change 表不在隔离范围内）
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 14.1_

  - [x] 6.2 更新 Entity 类添加 workspaceId 字段
    - 为 9 个隔离表对应的 Entity 类添加 `@TableField("workspace_id") private Long workspaceId;` 字段
    - 涉及 Entity：NovelApp、AdConfig、AppAd、AppCommonConfig、AppPay、AppPreviewPage、AppUiConfig、AppWeijuBusinessType、AppWeijuPublicSwitch
    - 为 Workspace Entity 添加 `@TableField("db_baseline_version") private LocalDateTime dbBaselineVersion;` 字段
    - _Requirements: 8.1, 14.1_

- [x] 7. 实现 WorkspaceInterceptor MyBatis-Plus 拦截器
  - [x] 7.1 创建 `WorkspaceInterceptor` 类
    - 创建 `com.fun.novel.workspace.interceptor.WorkspaceInterceptor`，继承 `TenantLineInnerInterceptor`
    - 定义 `ISOLATED_TABLES` 常量 Set，包含 9 张隔离表名（小写）
    - 重写 `getTenantId()` 返回 `new LongValue(WorkspaceContext.getCurrentWorkspaceId())`
    - 重写 `getTenantIdColumn()` 返回 `"workspace_id"`
    - 重写 `ignoreTable(String tableName)` 返回 `!ISOLATED_TABLES.contains(tableName.toLowerCase())`
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6_

  - [x] 7.2 注册 WorkspaceInterceptor 到 MybatisPlusConfig
    - 修改 `com.fun.novel.config.MybatisPlusConfig`
    - 在 `mybatisPlusInterceptor()` 方法中，在 `PaginationInnerInterceptor` **之前**添加 `WorkspaceInterceptor`
    - MyBatis-Plus 要求 TenantLine 拦截器在 Pagination 之前注册
    - _Requirements: 9.7_

  - [ ]* 7.3 编写 WorkspaceInterceptor 属性测试：表隔离过滤
    - **Property 4: Table isolation filter correctness**
    - 使用 jqwik 随机生成表名字符串
    - 验证 `ignoreTable()` 对 9 张隔离表返回 false，对其他表名返回 true
    - 测试大小写不敏感（如 "NOVEL_APP"、"Novel_App"）
    - 创建测试类 `WorkspaceInterceptorPropertyTest`，最少 100 次迭代
    - **Validates: Requirements 9.3**

  - [ ]* 7.4 编写 WorkspaceInterceptor 单元测试
    - 创建 `WorkspaceInterceptorTest` 测试类
    - 验证 `getTenantId()` 返回 WorkspaceContext 中的值
    - 验证 `getTenantIdColumn()` 返回 `"workspace_id"`
    - 验证无活跃工作空间时 `getTenantId()` 返回 0
    - _Requirements: 9.1, 9.2, 9.6_

- [ ] 8. Checkpoint - 确保拦截器和 Schema 迁移正确
  - 确保所有测试通过，ask the user if questions arise.

- [x] 9. 实现 DataCopierService 数据复制
  - [x] 9.1 创建 `DataCopierService` 类
    - 创建 `com.fun.novel.workspace.service.DataCopierService`
    - 注入 `JdbcTemplate`（绕过 TenantLine 拦截器执行跨 workspace_id 的 SQL）
    - 定义 `ISOLATED_TABLES` 列表，包含 9 张隔离表名
    - 实现 `copyMainDataToWorkspace(Long userId)` 方法，使用 `@Transactional(rollbackFor = Exception.class)`
    - 对每张表执行 `INSERT INTO {table} (...columns, workspace_id) SELECT ...columns, {userId} FROM {table} WHERE workspace_id = 0`
    - 实现 `getColumnsExceptWorkspaceId(String table)` 辅助方法，通过 `DatabaseMetaData` 获取列名
    - _Requirements: 10.1, 10.2, 10.3_

  - [x] 9.2 集成 DataCopierService 到 WorkspaceManagerService
    - 在 `WorkspaceManagerService.createWorkspace()` 中，Git Worktree 创建成功后调用 `dataCopierService.copyMainDataToWorkspace(userId)`
    - 数据复制成功后设置 `workspace.setDbBaselineVersion(LocalDateTime.now())`
    - 数据复制失败时清理已创建的 Git Worktree
    - _Requirements: 10.2, 10.3, 10.4, 14.2_

  - [ ]* 9.3 编写 DataCopierService 属性测试：数据复制正确性
    - **Property 5: Data copy preserves Main_Data with correct workspace_id**
    - 使用 H2 内存数据库和 jqwik 随机生成测试数据行
    - 验证复制后 workspace_id=userId 的行数等于 workspace_id=0 的行数
    - 验证非 workspace_id 列值完全匹配
    - 创建测试类 `DataCopierPropertyTest`，最少 100 次迭代
    - **Validates: Requirements 10.1**

- [x] 10. 实现 DataMergerService 数据合并
  - [x] 10.1 创建 `DataMergerService` 类
    - 创建 `com.fun.novel.workspace.service.DataMergerService`
    - 注入 `JdbcTemplate`
    - 实现 `mergeWorkspaceToMain(Long userId, LocalDateTime dbBaselineVersion)` 方法，使用 `@Transactional(rollbackFor = Exception.class)`
    - 实现冲突检测：查询 Main_Data 中 `updated_at > dbBaselineVersion` 的行
    - 检测到冲突时抛出 `DataConflictException`（包含冲突表名和行标识）
    - 无冲突时将 Workspace_Data 变更应用到 Main_Data
    - 合并成功后刷新 Workspace_Data（重新复制 Main_Data）
    - _Requirements: 11.1, 11.2, 11.3, 11.4_

  - [x] 10.2 创建 `DataConflictException` 异常类
    - 创建 `com.fun.novel.workspace.exception.DataConflictException`
    - 包含冲突表名和行标识信息
    - _Requirements: 11.3_

  - [x] 10.3 集成 DataMergerService 到合并审批流程
    - 在 `MergeEngine.approve()` 流程中调用 `dataMergerService.mergeWorkspaceToMain()`
    - 合并成功后更新 workspace 的 `dbBaselineVersion` 为当前时间戳
    - 处理 `DataConflictException`，报告冲突信息
    - _Requirements: 11.1, 11.2, 11.3, 11.4, 14.3_

  - [ ]* 10.4 编写 DataMergerService 属性测试：冲突检测
    - **Property 6: Conflict detection via timestamp comparison**
    - 使用 H2 内存数据库和 jqwik 随机生成时间戳
    - 验证 Main_Data 行 `updated_at > dbBaselineVersion` 时检测为冲突并中止
    - 验证 Main_Data 行 `updated_at <= dbBaselineVersion` 时不标记为冲突
    - 创建测试类 `DataMergerPropertyTest`，最少 100 次迭代
    - **Validates: Requirements 11.3, 14.4**

- [x] 11. 实现数据清理（工作空间销毁）
  - [x] 11.1 创建数据清理逻辑
    - 在 `DataCopierService` 中添加 `cleanupWorkspaceData(Long userId)` 方法
    - 使用 `@Transactional(rollbackFor = Exception.class)` 对 9 张隔离表执行 `DELETE FROM {table} WHERE workspace_id = {userId}`
    - 使用 `JdbcTemplate` 绕过 TenantLine 拦截器
    - 失败时记录 ERROR 日志（含表名和 userId），加入清理重试队列
    - _Requirements: 12.1, 12.2, 12.3_

  - [x] 11.2 集成数据清理到 WorkspaceManagerService.destroyWorkspace()
    - 在 `destroyWorkspace()` 方法中，Git Worktree 移除前先执行数据清理
    - 数据清理失败时记录日志但不阻塞 Worktree 移除
    - _Requirements: 12.1, 12.2, 12.3_

  - [ ]* 11.3 编写数据清理属性测试
    - **Property 7: Data cleanup removes all workspace rows**
    - 使用 H2 内存数据库和 jqwik 随机生成 userId 和数据行
    - 验证清理后 workspace_id=userId 的行数为 0
    - 验证 workspace_id=0 的 Main_Data 行数不变
    - 创建测试类 `DataCleanupPropertyTest`，最少 100 次迭代
    - **Validates: Requirements 12.1**

- [ ] 12. Checkpoint - 确保数据生命周期测试通过
  - 确保所有测试通过，ask the user if questions arise.

- [x] 13. 实现前端 X-Workspace-Id Header 注入
  - [x] 13.1 修改 `request.js` 请求拦截器
    - 在 `NovelAppManager/src/utils/request.js` 的请求拦截器中添加 `X-Workspace-Id` header 注入
    - 从 `workspaceStore` 读取 `currentWorkspaceId`，非空时设置 `config.headers['X-Workspace-Id']`
    - 同步修改 `requestWithTimeout` 函数中的请求拦截器，保持一致
    - 无活跃工作空间时不添加 header
    - _Requirements: 13.1, 13.2, 13.4_

  - [x] 13.2 更新 `workspaceStore.js` 添加 currentWorkspaceId 状态
    - 在 `workspaceStore` 中新增 `currentWorkspaceId` ref 状态字段
    - 在 `syncWorkspace` 成功后设置 `currentWorkspaceId`（从响应数据中获取 userId 或 workspaceId）
    - 在 `getStatus` 成功后设置 `currentWorkspaceId`（status 为 active 时设置，none 时清除）
    - 在 workspace 销毁时清除 `currentWorkspaceId`
    - 将 `currentWorkspaceId` 添加到 store 的 return 对象中
    - _Requirements: 13.3_

  - [ ]* 13.3 编写前端 header 注入单元测试
    - 创建 `request.spec.js` 测试文件
    - 使用 Vitest mock workspaceStore
    - 验证有活跃 workspace 时请求包含 `X-Workspace-Id` header
    - 验证无活跃 workspace 时请求不包含该 header
    - _Requirements: 13.1, 13.2_

- [ ] 14. Final checkpoint - 全部测试通过
  - 确保所有测试通过，ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties from the design document
- Unit tests validate specific examples and edge cases
- `JdbcTemplate` is used for DataCopier/DataMerger to bypass TenantLine interceptor for cross-workspace SQL operations
- MySQL 8 does not support `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` — the migration script should use stored procedures or conditional logic for idempotency
- WorkspaceInterceptor must be registered before PaginationInnerInterceptor in the MyBatis-Plus plugin chain
- The `change` table uses backtick escaping but is not in the isolation scope
