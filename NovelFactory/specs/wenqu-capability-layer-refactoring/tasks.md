# 实现计划：文曲能力层重构

## 概述

按设计文档的四步实施顺序，逐步在现有代码上增加薄编排层：先扩展 ActorContext 和补齐互斥控制，再新增聚合查询与聚合编辑接口，最后接入 Agent 接口层（API Key 鉴权、限流、任务轮询、各业务代理接口）。

## 任务

- [x] 1. 扩展 ActorContext，补充 roles 字段
  - [x] 1.1 修改 `ActorContext` 类，新增 `List<String> roles` 字段，保留 `source` 字段兼容现有调用
    - 从 `Authentication.getAuthorities()` 提取 roles
    - 系统级调用允许 `userId == null`，但必须携带 `ROLE_0`
    - _需求：14.1、14.2_
  - [x] 1.2 修改 `CreateNovelAppCommandFactory`，在构建 `ActorContext` 时填充 `roles`
    - _需求：14.1_
  - [ ]* 1.3 为 `ActorContext` 编写单元测试
    - 验证 roles 提取逻辑
    - 验证系统级调用场景
    - _需求：14.1_

- [x] 2. 新增全局互斥控制组件
  - [x] 2.1 新增 `EditingOperationTracker` 组件
    - 实现 `tryStart(String appId)`、`finish(String appId)`、`hasAnyEditing()`、`isEditing(String appId)` 方法
    - 使用 `ConcurrentHashMap` 维护编辑态，线程安全
    - _需求：8.9_
  - [x] 2.2 新增 `GlobalOperationGuard` 组件
    - 实现 `checkCanStartCreate()`：若有构建/发布运行中、编辑进行中或已有创建任务则抛出对应错误
    - 实现 `checkCanStartEdit(String appId)`：若有构建/发布运行中、创建进行中或同 appId 编辑进行中则抛出对应错误
    - 实现 `beforeEnqueueBuildOrPublish(String appId)`：始终允许入队
    - 实现 `runEdit(String appId, Supplier<T> action)`：进入互斥区执行，完成后释放
    - 依赖 `CreateNovelTaskManager`、`TaskQueueManager`、`EditingOperationTracker`
    - _需求：8.9、9.5、11.1、12.1_
  - [x] 2.3 扩展 `TaskQueueManager`，新增状态查询方法
    - 新增 `hasRunningTask()`、`hasQueuedTask()`、`canStartNextHeavyTask()` 方法
    - `canStartNextHeavyTask()` 内部检查：无创建进行中、无编辑进行中、当前运行槽为空
    - _需求：4.4_
  - [ ]* 2.4 为互斥控制编写单元测试
    - 创建执行中，编辑被拒绝
    - 编辑执行中，创建被拒绝
    - 创建执行中，构建进入队列但不启动
    - 构建运行中，创建被拒绝
    - _需求：8.9、9.5_

- [x] 3. 检查点 - 确保所有测试通过，如有疑问请询问用户

- [x] 4. 新增聚合查询能力
  - [x] 4.1 新增 `NovelAppAggregateView` DTO
    - 包含 `baseConfig`、`commonConfig`、`payConfig`、`uiConfig`、`adConfig`、`buildInfo`、`publishInfo`、`syncStatus`、`permissionInfo` 字段
    - 新增 `PermissionInfo`、`SyncStatus` 数据类
    - _需求：7.2_
  - [x] 4.2 新增 `NovelAppQueryFacade` 服务
    - 实现按 `appId`、`appCode`、`customer`、`platform`、`appName` 等参数查询
    - 对每个 app 聚合读取各分域配置，任一域失败则整个请求失败
    - 计算并填充 `permissionInfo` 和 `syncStatus`
    - _需求：6.1、6.2、6.3、6.4、7.1、7.2、7.3、7.4_
  - [x] 4.3 新增 `GET /api/novel-apps/query` Controller 端点
    - 支持可选查询参数：`appId`、`appCode`、`customer`、`platform`、`appName`
    - 唯一字段命中单条时返回单对象，否则返回列表
    - _需求：6.1、6.2、6.3、6.4_
  - [ ]* 4.4 为 `NovelAppQueryFacade` 编写单元测试
    - 测试各参数组合的查询逻辑
    - 测试某配置域查询失败时整个请求失败的行为
    - _需求：7.3、7.4_

- [x] 5. 新增聚合配置更新能力
  - [x] 5.1 新增 `UpdateAppConfigRequest` DTO
    - 包含 `baseConfig`、`commonConfig`、`payConfig`、`uiConfig`、`adConfig` 字段，均可为 null
    - _需求：8.1、8.2_
  - [x] 5.2 新增 `NovelAppConfigFacade` 服务
    - 实现"传了就改，未传（null）则不动"的语义
    - 执行流程：构建 ActorContext → 互斥检查 → 权限检查 → 读旧快照 → 逐域更新数据库 → 同步文件 → 文件失败时用旧快照补偿数据库 → 释放编辑态
    - 数据库操作使用 `@Transactional`
    - 文件操作使用 `rollbackActions` 补偿，与数据库事务独立
    - _需求：8.1、8.2、8.3、8.4、8.5、8.6、8.7、8.8、8.9、8.10_
  - [x] 5.3 新增 `PUT /api/novel-apps/{appId}/config` Controller 端点
    - 调用 `NovelAppConfigFacade` 执行聚合更新
    - 返回更新后的完整配置（与详情接口结构一致）
    - _需求：8.1、8.8_
  - [ ]* 5.4 为 `NovelAppConfigFacade` 编写单元测试
    - 测试各配置域的选择性更新逻辑
    - 测试文件失败时数据库回补行为
    - 测试互斥检查被触发时的拒绝行为
    - _需求：8.2、8.9、8.10_

- [x] 6. 检查点 - 确保所有测试通过，如有疑问请询问用户

- [x] 7. 实现 API Key 鉴权基础设施（Phase 1）
  - [x] 7.1 创建 `api_key` 数据库表及对应 Entity、Repository
    - 表字段：`api_key`、`user_id`、`name`、`enabled`、`rate_limit`（默认 60）、`request_count`、`created_time`、`last_used_time`
    - _需求：1.1_
  - [x] 7.2 实现 `ApiKeyAuthenticationFilter`
    - 解析 `X-API-Key` 请求头，查询 `api_key` 表验证 Key 是否存在且 `enabled=1`
    - 验证通过后通过 `user_id` 查询 User 并设置到 SecurityContext
    - 验证通过后将 `request_count` 加 1 并更新 `last_used_time`
    - Key 不存在或 `enabled=0` 时返回 HTTP 401，errorCode 为 `UNAUTHORIZED`
    - 请求头中无 `X-API-Key` 时直接放行，继续执行 JWT 认证流程
    - _需求：1.2、1.3、1.4、1.5、1.6_
  - [x] 7.3 实现按 API Key 维度的限流逻辑
    - 使用 `ConcurrentHashMap<String, AtomicInteger>` 维护每个 Key 的当前分钟请求计数
    - 每分钟开始时重置计数（使用 `@Scheduled` 或时间窗口判断）
    - 超过 `rate_limit` 时返回 HTTP 429，errorCode 为 `RATE_LIMIT_EXCEEDED`
    - _需求：3.1、3.2、3.3、3.4、3.5、3.6_
  - [x] 7.4 修改 `SecurityConfig`，将 `ApiKeyAuthenticationFilter` 注册到过滤器链
    - 优先级高于 `JwtAuthenticationFilter`
    - 要求 `/api/agent/**` 路径必须通过 `ApiKeyAuthenticationFilter` 认证
    - _需求：1.7、1.8_
  - [ ]* 7.5 为 `ApiKeyAuthenticationFilter` 编写单元测试
    - 测试有效 Key 的认证流程
    - 测试无效/禁用 Key 返回 401
    - 测试无 Key 时 JWT 流程不受影响
    - 测试限流触发返回 429
    - _需求：1.2、1.5、1.6、3.3_

- [x] 8. 实现 API Key 管理接口
  - [x] 8.1 新增 `ApiKeyService`，实现创建、查询、删除 API Key 的业务逻辑
    - 创建时生成全局唯一 Key 值，绑定当前用户 `user_id`
    - 删除时校验 Key 属于当前用户，否则返回 HTTP 403
    - _需求：2.1、2.2、2.3、2.4、2.5_
  - [x] 8.2 新增 `ApiKeyController`，暴露管理接口
    - `POST /api/novel-auth/api-keys`：生成新 API Key
    - `GET /api/novel-auth/api-keys`：返回当前用户所有 Key 列表（含 `request_count`、`last_used_time`）
    - `DELETE /api/novel-auth/api-keys/{id}`：删除指定 Key
    - _需求：2.1、2.2、2.3_
  - [ ]* 8.3 为 `ApiKeyController` 编写单元测试
    - 测试创建、查询、删除的正常流程
    - 测试删除他人 Key 返回 403
    - _需求：2.4、2.5_

- [x] 9. 实现统一任务状态轮询接口（Phase 1）
  - [x] 9.1 定义 `TaskStatusResponse` DTO
    - 包含 `taskId`、`taskType`、`status`、`progress`、`description`、`startTime`、`completeTime`、`errorCode`、`errorMessage` 字段
    - 定义任务状态枚举：`submitted`、`pending`、`running`、`completed`、`failed`
    - _需求：4.2、4.3_
  - [x] 9.2 实现 `AgentTaskStatusService`
    - 按顺序查找：先查 `TaskQueueManager`，再查 `CreateNovelTaskManager`，最后查数据库 `user_task` 表
    - 三处均未找到时返回 errorCode 为 `TASK_NOT_FOUND` 的错误
    - _需求：4.4、4.5、4.6_
  - [x] 9.3 新增 `GET /api/agent/task/{taskId}/status` Controller 端点
    - 调用 `AgentTaskStatusService` 查询并返回任务状态
    - _需求：4.1_
  - [x] 9.4 改造 `CreateNovelTaskManager`，补充任务状态维护能力
    - 维护 `ConcurrentHashMap<String, TaskStatus> taskStatusMap`
    - 任务开始时设置 `running`，成功时设置 `completed` 并写入 `user_task` 表，失败时设置 `failed` 并写入 `user_task` 表
    - 内存中保留任务状态 24 小时
    - _需求：5.1、5.2、5.3、5.4、5.5_
  - [ ]* 9.5 为任务状态查询编写单元测试
    - 测试三处查找顺序
    - 测试 taskId 不存在时返回 TASK_NOT_FOUND
    - _需求：4.4、4.5_

- [x] 10. 检查点 - 确保所有测试通过，如有疑问请询问用户

- [x] 11. 实现 Agent 业务代理接口（Phase 2）
  - [x] 11.1 新增 `AgentAppController`，实现应用列表接口
    - `GET /api/agent/app/list`：返回当前用户可访问的应用列表，支持可选 `platform` 参数筛选
    - 列表字段：`id`、`appId`、`appName`、`platform`、`appCode`、`version`、`product`、`customer`、`deliverId`、`bannerId`
    - _需求：6.1、6.2、6.3、6.4_
  - [x] 11.2 在 `AgentAppController` 中实现应用详情接口
    - `GET /api/agent/app/detail?appId={appId}`：复用 `NovelAppQueryFacade` 返回完整聚合配置
    - appId 不存在时返回 errorCode 为 `INVALID_PARAMS` 的错误
    - _需求：7.1、7.2、7.3、7.4_
  - [x] 11.3 在 `AgentAppController` 中实现聚合编辑接口
    - `POST /api/agent/app/update`：复用 `NovelAppConfigFacade` 执行聚合更新
    - _需求：8.1、8.2、8.8_
  - [x] 11.4 在 `AgentAppController` 中实现创建应用接口
    - `POST /api/agent/app/create`：验证必需字段，立即返回 `{ "taskId": "uuid" }`，异步执行创建任务
    - 校验失败返回 `INVALID_PARAMS`，同名同平台已存在返回 `DUPLICATE_APP`，已有创建任务返回 `CREATE_IN_PROGRESS`
    - _需求：9.1、9.2、9.3、9.4、9.5_
  - [x] 11.5 在 `AgentAppController` 中实现删除应用接口
    - `DELETE /api/agent/app/delete?appId={appId}`：同步删除数据库记录、本地代码文件和资源文件
    - appId 不存在时返回 `INVALID_PARAMS`
    - _需求：10.1、10.2、10.3、10.4_
  - [x] 11.6 新增 `AgentBuildPublishController`，实现构建和发布代理接口
    - `POST /api/agent/build`：将任务加入构建队列，立即返回 `{ "taskId": "uuid" }`
    - `POST /api/agent/publish`：将任务加入发布队列，立即返回 `{ "taskId": "uuid" }`
    - _需求：11.1、11.2、12.1、12.2_
  - [ ]* 11.7 为 Agent 业务接口编写集成测试
    - 测试应用列表、详情、创建、删除的正常流程
    - 测试构建/发布接口返回 taskId 并可通过轮询接口查询状态
    - _需求：9.2、11.2、12.2_

- [x] 12. 实现结构化错误码（Phase 3）
  - [x] 12.1 定义 `AgentErrorCode` 枚举
    - 包含：`DUPLICATE_APP`、`CREATE_IN_PROGRESS`、`BUILD_FAILED`、`BUILD_INTERRUPTED`、`PUBLISH_TOKEN_MISSING`、`PUBLISH_FAILED`、`TASK_NOT_FOUND`、`INVALID_PARAMS`、`RATE_LIMIT_EXCEEDED`、`UNAUTHORIZED`
    - _需求：13.1_
  - [x] 12.2 扩展 `TaskQueueItem`，新增 `errorCode` 字段（默认 null）
    - 各 Processor 在捕获异常时设置对应的错误码：构建失败设置 `BUILD_FAILED`，构建中断设置 `BUILD_INTERRUPTED`，发布缺少 Token 设置 `PUBLISH_TOKEN_MISSING`，发布失败设置 `PUBLISH_FAILED`
    - _需求：11.3、11.4、12.3、12.4、13.4_
  - [x] 12.3 确保所有 Agent 接口的同步错误响应体包含 `errorCode` 和 `errorMessage` 字段
    - 统一错误响应格式，保证同一错误场景始终返回相同的错误码
    - _需求：13.2、13.3、13.5_
  - [ ]* 12.4 为结构化错误码编写单元测试
    - 验证各错误场景返回正确的 errorCode
    - _需求：13.5_

- [x] 13. 最终检查点 - 确保所有测试通过，验证兼容性
  - 确保所有现有 Controller 未被修改
  - 确保 WebSocket 实时日志推送功能正常工作
  - 确保网页前端通过 JWT 调用现有接口行为不变
  - 确保所有测试通过，如有疑问请询问用户

## 备注

- 标有 `*` 的子任务为可选测试任务，可在快速交付时跳过
- 每个任务均引用了对应的需求条款，便于追溯
- 实施顺序遵循设计文档的四步策略：ActorContext → 互斥控制 → 聚合查询 → 聚合编辑，再叠加 Agent 接口层
- 所有新增组件均不修改现有 Controller 和 Service，保持向后兼容
