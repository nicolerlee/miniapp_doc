# 需求文档

## 介绍

文曲系统当前的 API 仅供网页前端使用，通过 JWT 登录鉴权。随着外部调用方（agent、龙虾等自动化程序）的接入需求增加，需要开放一套面向外部调用方的 API 层。

本次功能的核心目标是：为外部调用方提供 API Key 鉴权机制、统一的异步任务状态轮询接口、结构化错误码，以及一组聚合了现有业务能力的 Agent 接口层（`/api/agent/`），使 agent 无需了解内部多接口细节即可完成小程序的增删改查、构建与发布操作。

实施分三个阶段：
- **Phase 1**：API Key 认证 + 限流 + 任务状态轮询
- **Phase 2**：应用列表 / 详情 / 聚合编辑 / 创建 / 删除 / 构建 / 发布接口
- **Phase 3**：结构化错误码

---

## 术语表

- **文曲系统**：小说小程序管理系统，提供创建、配置、构建、发布等能力
- **外部调用方**：非网页前端的调用者，包括 agent、龙虾等自动化程序
- **API Key**：绑定到具体用户的长期有效凭证，用于替代 JWT 供外部调用方鉴权
- **ApiKeyAuthenticationFilter**：负责解析 `X-API-Key` 请求头并完成认证的过滤器
- **Agent 接口层**：位于 `/api/agent/` 路径下，面向外部调用方的聚合接口集合
- **TaskId**：异步任务的唯一标识符（UUID 格式）
- **任务状态机**：描述异步任务生命周期的状态集合：`submitted → pending → running → completed / failed`
- **聚合接口**：内部调用多个 Service，对外暴露为单一接口的设计模式
- **限流**：按 API Key 维度控制单位时间内的最大请求数
- **结构化错误码**：用枚举值（如 `BUILD_FAILED`）标识错误类型，替代纯文本错误信息
- **TaskQueueManager**：管理构建/发布任务队列的内存组件
- **CreateNovelTaskManager**：管理创建小程序任务状态的内存组件

---

## 需求

### 需求 1：API Key 鉴权

**用户故事：** 作为外部调用方（agent），我希望使用 API Key 而非 JWT 进行身份认证，以便无需登录流程即可调用文曲 API。

#### 验收标准

1. THE 文曲系统 SHALL 提供 `api_key` 数据库表，包含 `api_key`、`user_id`、`name`、`enabled`、`rate_limit`、`request_count`、`created_time`、`last_used_time` 字段
2. WHEN 请求头中包含 `X-API-Key` 时，THE ApiKeyAuthenticationFilter SHALL 查询 `api_key` 表并验证该 Key 是否存在且 `enabled=1`
3. WHEN API Key 验证通过时，THE ApiKeyAuthenticationFilter SHALL 通过 `user_id` 查询对应的 User 对象，并将其设置到 SecurityContext 中
4. WHEN API Key 验证通过时，THE ApiKeyAuthenticationFilter SHALL 将 `request_count` 加 1 并更新 `last_used_time`
5. IF 请求头中的 API Key 不存在或 `enabled=0`，THEN THE ApiKeyAuthenticationFilter SHALL 返回 HTTP 401，errorCode 为 `UNAUTHORIZED`
6. WHEN 请求头中不包含 `X-API-Key` 时，THE 文曲系统 SHALL 继续执行原有的 JWT 认证流程，不受影响
7. THE 文曲系统 SHALL 将 `ApiKeyAuthenticationFilter` 注册到 SecurityConfig，优先级高于 `JwtAuthenticationFilter`
8. THE 文曲系统 SHALL 要求 `/api/agent/**` 路径必须通过 `ApiKeyAuthenticationFilter` 完成认证后才能访问
9. THE 文曲系统 SHALL 允许一个用户拥有多个 API Key，每个 Key 独立管理

### 需求 2：API Key 管理接口

**用户故事：** 作为系统用户，我希望能通过接口管理自己的 API Key，以便创建、查看和删除 Key，而无需手动操作数据库。

#### 验收标准

1. THE 文曲系统 SHALL 提供 `POST /api/novel-auth/api-keys` 接口，为当前登录用户生成新的 API Key
2. THE 文曲系统 SHALL 提供 `GET /api/novel-auth/api-keys` 接口，返回当前用户的所有 API Key 列表，包含 `request_count` 和 `last_used_time`
3. THE 文曲系统 SHALL 提供 `DELETE /api/novel-auth/api-keys/{id}` 接口，删除或禁用指定的 API Key
4. WHEN 生成 API Key 时，THE 文曲系统 SHALL 生成全局唯一的 Key 值，并将其与当前用户的 `user_id` 绑定
5. WHEN 删除 API Key 时，THE 文曲系统 SHALL 仅允许操作属于当前用户的 Key，IF 尝试删除他人的 Key，THEN THE 文曲系统 SHALL 返回 HTTP 403

### 需求 3：请求限流

**用户故事：** 作为系统管理员，我希望按 API Key 维度控制请求频率，以防止单个调用方过度占用系统资源。

#### 验收标准

1. THE ApiKeyAuthenticationFilter SHALL 在认证通过后，检查当前 API Key 在当前分钟内的请求次数是否超过 `rate_limit` 值
2. WHEN 当前分钟内请求次数小于 `rate_limit` 时，THE ApiKeyAuthenticationFilter SHALL 放行请求
3. WHEN 当前分钟内请求次数大于或等于 `rate_limit` 时，THE ApiKeyAuthenticationFilter SHALL 返回 HTTP 429，errorCode 为 `RATE_LIMIT_EXCEEDED`
4. THE 文曲系统 SHALL 使用内存中的 `ConcurrentHashMap<apiKey, AtomicInteger>` 维护每个 API Key 的当前分钟请求计数，并在每分钟开始时重置
5. THE 文曲系统 SHALL 为 `api_key` 表的 `rate_limit` 字段设置默认值 60（次/分钟），并支持按 Key 单独配置
6. WHEN 某个 API Key 触发限流时，THE 文曲系统 SHALL 不影响其他 API Key 的正常请求

### 需求 4：统一任务状态轮询接口

**用户故事：** 作为外部调用方（agent），我希望通过统一的 HTTP 接口轮询任意异步任务的状态，以便无需接入 WebSocket 即可感知任务结果。

#### 验收标准

1. THE 文曲系统 SHALL 提供 `GET /api/agent/task/{taskId}/status` 接口，返回指定任务的当前状态
2. THE 文曲系统 SHALL 在响应中包含 `taskId`、`taskType`、`status`、`progress`、`description`、`startTime`、`completeTime`、`errorCode`、`errorMessage` 字段
3. THE 文曲系统 SHALL 支持 `submitted`、`pending`、`running`、`completed`、`failed` 五种任务状态
4. WHEN 查询任务状态时，THE 文曲系统 SHALL 按以下顺序查找：先查 `TaskQueueManager`（构建/发布/npm-install 任务），再查 `CreateNovelTaskManager`（创建小程序任务），最后查数据库 `user_task` 表（历史任务）
5. IF 三处均未找到该 taskId，THEN THE 文曲系统 SHALL 返回 errorCode 为 `TASK_NOT_FOUND` 的错误响应
6. THE 文曲系统 SHALL 在内存中保留已完成任务状态 24 小时，超时后从数据库 `user_task` 表查询
7. THE 文曲系统 SHALL 保持网页前端的 WebSocket 实时日志功能不受影响，两套机制并行运行

### 需求 5：创建小程序任务状态维护

**用户故事：** 作为外部调用方（agent），我希望创建小程序的任务状态也能通过统一轮询接口查询，以便与构建、发布任务保持一致的使用方式。

#### 验收标准

1. THE CreateNovelTaskManager SHALL 维护 `ConcurrentHashMap<String, TaskStatus> taskStatusMap`，记录每个创建任务的状态
2. WHEN 创建任务开始执行时，THE CreateNovelTaskManager SHALL 将对应 taskId 的状态设置为 `running`
3. WHEN 创建任务成功完成时，THE CreateNovelTaskManager SHALL 将对应 taskId 的状态设置为 `completed`，并将结果写入数据库 `user_task` 表
4. WHEN 创建任务执行失败时，THE CreateNovelTaskManager SHALL 将对应 taskId 的状态设置为 `failed`，并将失败信息写入数据库 `user_task` 表
5. THE CreateNovelTaskManager SHALL 在内存中保留任务状态 24 小时

### 需求 6：应用列表接口

**用户故事：** 作为外部调用方（agent），我希望获取当前可访问的小程序应用列表，以便了解系统中存在哪些应用。

#### 验收标准

1. THE 文曲系统 SHALL 提供 `GET /api/agent/app/list` 接口，返回当前用户可访问的应用列表
2. THE 文曲系统 SHALL 支持通过可选的 `platform` 查询参数按平台筛选应用列表
3. THE 文曲系统 SHALL 在列表中每条记录包含 `id`、`appId`、`appName`、`platform`、`appCode`、`version`、`product`、`customer`、`deliverId`、`bannerId` 字段
4. WHEN 未传入 `platform` 参数时，THE 文曲系统 SHALL 返回所有平台的应用列表

### 需求 7：应用详情接口

**用户故事：** 作为外部调用方（agent），我希望通过单个接口获取某个应用的完整配置，以便无需调用多个接口拼装数据。

#### 验收标准

1. THE 文曲系统 SHALL 提供 `GET /api/agent/app/detail?appId={appId}` 接口，返回指定应用的完整聚合配置
2. THE 文曲系统 SHALL 在响应中包含 `baseConfig`、`commonConfig`、`uiConfig`、`paymentConfig`、`adConfig`、`weijuConfig` 六个配置域
3. WHEN 查询应用详情时，THE 文曲系统 SHALL 分别调用各 Service 查询对应配置域，并聚合为单一响应返回
4. IF 指定的 `appId` 不存在，THEN THE 文曲系统 SHALL 返回 errorCode 为 `INVALID_PARAMS` 的错误响应

### 需求 8：聚合编辑接口

**用户故事：** 作为外部调用方（agent），我希望通过单个接口一次性修改应用的多个配置域，以便无需了解内部各模块接口的细节。

#### 验收标准

1. THE 文曲系统 SHALL 提供 `POST /api/agent/app/update` 接口，支持在单次请求中更新应用的一个或多个配置域
2. THE 文曲系统 SHALL 采用"传了就改，未传（null）则不动"的语义处理各配置域
3. WHEN 请求中包含 `baseConfig` 时，THE 文曲系统 SHALL 调用 `NovelAppService.updateNovelApp()` 更新基础信息
4. WHEN 请求中包含 `commonConfig` 时，THE 文曲系统 SHALL 调用 `AppCommonConfigService.updateAppCommonConfig()` 更新通用配置
5. WHEN 请求中包含 `uiConfig` 时，THE 文曲系统 SHALL 调用 `AppUIConfigService.updateAppUIConfig()` 更新 UI 配置
6. WHEN 请求中包含 `paymentConfig` 时，THE 文曲系统 SHALL 遍历每种支付类型并调用 `AppPayService.updateAppPay()` 更新支付配置
7. WHEN 请求中包含 `adConfig` 时，THE 文曲系统 SHALL 遍历每种广告类型并调用 `AdConfigService.updateAdConfig()` 更新广告配置
8. WHEN 所有配置域更新完成时，THE 文曲系统 SHALL 返回更新后的完整配置（与详情接口结构一致）
9. THE 文曲系统 SHALL 对数据库操作使用 `@Transactional`，任一数据库操作失败时已执行的数据库操作全部回滚
10. THE 文曲系统 SHALL 对文件操作使用 `rollbackActions` 补偿回滚，与数据库事务独立执行

### 需求 9：创建应用接口（Agent 代理）

**用户故事：** 作为外部调用方（agent），我希望通过 Agent 接口层发起创建小程序请求，并获得 taskId 用于后续轮询。

#### 验收标准

1. THE 文曲系统 SHALL 提供 `POST /api/agent/app/create` 接口，接受创建小程序所需的完整配置参数
2. WHEN 接收到有效的创建请求时，THE 文曲系统 SHALL 立即返回 `{ "taskId": "uuid" }`，并异步执行创建任务
3. WHEN 接收到创建请求时，THE 文曲系统 SHALL 验证必需字段的完整性，IF 校验失败，THEN THE 文曲系统 SHALL 返回 errorCode 为 `INVALID_PARAMS` 的错误响应
4. IF 同名同平台的小程序已存在，THEN THE 文曲系统 SHALL 返回 errorCode 为 `DUPLICATE_APP` 的错误响应
5. IF 已有创建任务正在执行，THEN THE 文曲系统 SHALL 返回 errorCode 为 `CREATE_IN_PROGRESS` 的错误响应

### 需求 10：删除应用接口（Agent 代理）

**用户故事：** 作为外部调用方（agent），我希望通过 Agent 接口层删除指定应用，以便完成应用生命周期管理。

#### 验收标准

1. THE 文曲系统 SHALL 提供 `DELETE /api/agent/app/delete?appId={appId}` 接口，删除指定应用
2. WHEN 删除成功时，THE 文曲系统 SHALL 返回 `{ "code": 200, "message": "应用删除成功" }`
3. IF 指定的 `appId` 不存在，THEN THE 文曲系统 SHALL 返回 errorCode 为 `INVALID_PARAMS` 的错误响应
4. WHEN 执行删除时，THE 文曲系统 SHALL 同步删除数据库记录、本地代码文件和资源文件

### 需求 11：构建接口（Agent 代理）

**用户故事：** 作为外部调用方（agent），我希望通过 Agent 接口层发起构建请求，并获得 taskId 用于后续轮询。

#### 验收标准

1. THE 文曲系统 SHALL 提供 `POST /api/agent/build` 接口，接受构建所需参数并将任务加入构建队列
2. WHEN 接收到有效的构建请求时，THE 文曲系统 SHALL 立即返回 `{ "taskId": "uuid" }`
3. WHEN 构建任务失败时，THE 文曲系统 SHALL 在任务状态中设置 errorCode 为 `BUILD_FAILED`
4. WHEN 构建任务被中断时，THE 文曲系统 SHALL 在任务状态中设置 errorCode 为 `BUILD_INTERRUPTED`

### 需求 12：发布接口（Agent 代理）

**用户故事：** 作为外部调用方（agent），我希望通过 Agent 接口层发起发布请求，并获得 taskId 用于后续轮询。

#### 验收标准

1. THE 文曲系统 SHALL 提供 `POST /api/agent/publish` 接口，接受发布所需参数并将任务加入发布队列
2. WHEN 接收到有效的发布请求时，THE 文曲系统 SHALL 立即返回 `{ "taskId": "uuid" }`
3. IF 缺少平台 Token，THEN THE 文曲系统 SHALL 在任务状态中设置 errorCode 为 `PUBLISH_TOKEN_MISSING`
4. WHEN 发布任务失败时，THE 文曲系统 SHALL 在任务状态中设置 errorCode 为 `PUBLISH_FAILED`

### 需求 13：结构化错误码

**用户故事：** 作为外部调用方（agent），我希望所有错误响应都包含结构化的错误码，以便程序能精确判断失败原因，而无需解析文本日志。

#### 验收标准

1. THE 文曲系统 SHALL 定义标准错误码枚举，包含：`DUPLICATE_APP`、`CREATE_IN_PROGRESS`、`BUILD_FAILED`、`BUILD_INTERRUPTED`、`PUBLISH_TOKEN_MISSING`、`PUBLISH_FAILED`、`TASK_NOT_FOUND`、`INVALID_PARAMS`、`RATE_LIMIT_EXCEEDED`、`UNAUTHORIZED`
2. WHEN 异步任务失败时，THE 文曲系统 SHALL 在任务状态响应的 `errorCode` 字段中返回对应的错误码
3. WHEN 同步接口发生错误时，THE 文曲系统 SHALL 在响应体的 `errorCode` 字段中返回对应的错误码，并在 `errorMessage` 字段中返回可读的错误描述
4. THE TaskQueueItem SHALL 包含 `errorCode` 字段（默认为 null），各 Processor 在捕获异常时设置对应的错误码
5. FOR ALL 错误码，THE 文曲系统 SHALL 保证同一错误场景始终返回相同的错误码，不因调用方式（agent 或网页）而不同

### 需求 14：对现有功能的兼容性保障

**用户故事：** 作为网页前端开发者，我希望 Agent 接口层的新增不影响现有网页功能，无需修改任何前端代码。

#### 验收标准

1. THE 文曲系统 SHALL 保持所有现有 Controller 不做任何修改
2. THE 文曲系统 SHALL 保持所有现有 Service 不做任何修改
3. THE 文曲系统 SHALL 保持 WebSocket 实时日志推送功能正常工作
4. THE 文曲系统 SHALL 保持数据库现有表结构不变，仅新增 `api_key` 表
5. WHEN 网页前端通过 JWT 调用现有接口时，THE 文曲系统 SHALL 提供与改造前完全一致的行为
