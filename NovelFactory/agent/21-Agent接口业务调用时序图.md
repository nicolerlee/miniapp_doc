# Agent 接口业务调用时序图

## 说明

本文汇总 `NovelAppManagerServer/docs/20-Agent接口.md` 中所有 Agent 相关接口的业务调用时序图，便于快速理解请求入口、认证、控制器、业务服务、任务队列与数据库之间的关系。

接口分组：

- Agent 应用管理：`/api/agent/app`
- Agent 构建与发布：`/api/agent`
- Agent 任务状态查询：`/api/agent`
- API Key 管理：`/api/novel-auth/api-keys`

---

## 1. Agent 应用管理

### 1.1 应用列表

```mermaid
sequenceDiagram
    participant Agent as "Agent Client"
    participant Filter as "ApiKeyAuthenticationFilter"
    participant Controller as "AgentAppController"
    participant Facade as "NovelAppQueryFacade"
    participant DB as "DB"

    Agent->>Filter: GET /api/agent/app/list?platform=...
    Filter->>DB: 校验 X-API-Key -> user
    DB-->>Filter: user / auth info
    Filter->>Controller: 已认证请求
    Controller->>Facade: query(null, null, null, platform, null)
    Facade->>DB: 查询应用聚合视图
    DB-->>Facade: 应用列表
    Facade-->>Controller: NovelAppAggregateView[]
    Controller-->>Agent: AppListItemDTO[]
```

### 1.2 聚合查询

```mermaid
sequenceDiagram
    participant Agent as "Agent Client"
    participant Filter as "ApiKeyAuthenticationFilter"
    participant Controller as "AgentAppController"
    participant Facade as "NovelAppQueryFacade"
    participant DB as "DB"

    Agent->>Filter: GET /api/agent/app/query
    Note over Agent,Filter: appId/appCode/customer/platform/appName 可选
    Filter->>DB: 校验 X-API-Key -> user
    DB-->>Filter: user / auth info
    Filter->>Controller: 已认证请求
    Controller->>Facade: query(appId, appCode, customer, platform, appName)
    Facade->>DB: 按优先级检索聚合数据
    DB-->>Facade: 0/1/N 条结果
    Facade-->>Controller: List<NovelAppAggregateView>
    Controller-->>Agent: 单对象或数组
```

### 1.3 应用详情

```mermaid
sequenceDiagram
    participant Agent as "Agent Client"
    participant Filter as "ApiKeyAuthenticationFilter"
    participant Controller as "AgentAppController"
    participant Facade as "NovelAppQueryFacade"
    participant DB as "DB"

    Agent->>Filter: GET /api/agent/app/detail?appId=...
    Filter->>DB: 校验 X-API-Key -> user
    DB-->>Filter: user / auth info
    Filter->>Controller: 已认证请求
    Controller->>Controller: 校验 appId 非空
    Controller->>Facade: queryByAppId(appId)
    Facade->>DB: 查询单个应用聚合配置
    DB-->>Facade: 应用详情 / null
    Facade-->>Controller: NovelAppAggregateView / null
    Controller-->>Agent: 成功响应 或 INVALID_PARAMS
```

### 1.4 聚合编辑

```mermaid
sequenceDiagram
    participant Agent as "Agent Client"
    participant Filter as "ApiKeyAuthenticationFilter"
    participant Controller as "AgentAppController"
    participant ConfigFacade as "NovelAppConfigFacade"
    participant DB as "DB"
    participant FS as "Local Files"

    Agent->>Filter: POST /api/agent/app/update
    Filter->>DB: 校验 X-API-Key -> user
    DB-->>Filter: user / auth info
    Filter->>Controller: 已认证请求 + Authentication
    Controller->>Controller: 校验 appId
    Controller->>ConfigFacade: updateConfig(appId, configRequest, authentication)
    ConfigFacade->>DB: 更新一个或多个配置域
    ConfigFacade->>FS: 同步本地配置文件
    DB-->>ConfigFacade: 持久化成功
    FS-->>ConfigFacade: 文件更新成功
    ConfigFacade-->>Controller: NovelAppAggregateView
    Controller-->>Agent: 更新后的聚合配置
```

### 1.5 创建应用

```mermaid
sequenceDiagram
    participant Agent as "Agent Client"
    participant Filter as "ApiKeyAuthenticationFilter"
    participant Controller as "AgentAppController"
    participant Guard as "GlobalOperationGuard"
    participant Service as "NovelAppCreationService"
    participant TaskMgr as "CreateNovelTaskManager"
    participant DB as "DB"
    participant FS as "Local Files/Resources"

    Agent->>Filter: POST /api/agent/app/create
    Filter->>DB: 校验 X-API-Key -> user
    DB-->>Filter: user / auth info
    Filter->>Controller: 已认证请求
    Controller->>Controller: 校验 baseConfig 必填字段
    Controller->>Guard: checkCanStartCreate()
    Guard-->>Controller: 允许/拒绝
    Controller->>DB: 校验同名同平台是否已存在
    DB-->>Controller: existing / null
    Controller->>TaskMgr: createTask()
    TaskMgr-->>Controller: taskId
    Controller-->>Agent: 立即返回 taskId
    Controller->>Service: 异步 createNovelAppOperations(taskId, params, rollbackActions)
    Service->>DB: 创建数据库配置
    Service->>FS: 创建本地代码文件
    Service->>FS: 创建资源文件
    Service-->>TaskMgr: setTaskCompleted / setTaskFailed
```

### 1.6 删除应用

```mermaid
sequenceDiagram
    participant Agent as "Agent Client"
    participant Filter as "ApiKeyAuthenticationFilter"
    participant Controller as "AgentAppController"
    participant AppSvc as "NovelAppService"
    participant FileSvc as "NovelAppLocalFileOperationService"
    participant ResSvc as "NovelAppResourceFileService"
    participant DB as "DB"
    participant FS as "Local Files/Resources"

    Agent->>Filter: DELETE /api/agent/app/delete?appId=...
    Filter->>DB: 校验 X-API-Key -> user
    DB-->>Filter: user / auth info
    Filter->>Controller: 已认证请求
    Controller->>AppSvc: getByAppId(appId)
    AppSvc->>DB: 查询应用
    DB-->>AppSvc: 应用 / null
    AppSvc-->>Controller: 应用实体
    Controller->>FileSvc: deleteAppLocalCodeFiles(params, rollbackActions, isLast)
    Controller->>ResSvc: deleteResourceFiles(params, rollbackActions, isLast)
    Controller->>AppSvc: deleteByAppId(appId)
    AppSvc->>DB: 删除数据库记录
    DB-->>AppSvc: success
    AppSvc-->>Controller: success
    Controller-->>Agent: 删除成功 / 失败回滚
```

---

## 2. Agent 构建与发布

### 2.1 构建小程序

```mermaid
sequenceDiagram
    participant Agent as "Agent Client"
    participant Filter as "ApiKeyAuthenticationFilter"
    participant Controller as "AgentBuildPublishController"
    participant UserSvc as "UserService"
    participant Queue as "TaskQueueManager"
    participant Worker as "Build Worker"
    participant DB as "DB"

    Agent->>Filter: POST /api/agent/build
    Filter->>DB: 校验 X-API-Key -> user
    DB-->>Filter: user / auth info
    Filter->>Controller: 已认证请求 + Authentication
    Controller->>UserSvc: getUserIdByUsername(authentication.name)
    UserSvc->>DB: 查询 userId
    DB-->>UserSvc: userId
    Controller->>Queue: addTaskToQueue(userId, buildTask)
    Queue-->>Controller: task accepted
    Controller-->>Agent: 返回 taskId
    Queue->>Worker: 异步执行构建任务
```

### 2.2 发布小程序

```mermaid
sequenceDiagram
    participant Agent as "Agent Client"
    participant Filter as "ApiKeyAuthenticationFilter"
    participant Controller as "AgentBuildPublishController"
    participant UserSvc as "UserService"
    participant Queue as "TaskQueueManager"
    participant Worker as "Publish Worker"
    participant DB as "DB"

    Agent->>Filter: POST /api/agent/publish
    Filter->>DB: 校验 X-API-Key -> user
    DB-->>Filter: user / auth info
    Filter->>Controller: 已认证请求 + Authentication
    Controller->>Controller: 校验 platformCode/appId/projectPath/version/log
    Controller->>UserSvc: getUserIdByUsername(authentication.name)
    UserSvc->>DB: 查询 userId
    DB-->>UserSvc: userId
    Controller->>Queue: addTaskToQueue(userId, publishTask)
    Queue-->>Controller: task accepted
    Controller-->>Agent: 返回 taskId
    Queue->>Worker: 异步执行发布任务
```

---

## 3. Agent 任务状态查询

### 3.1 查询任务状态

```mermaid
sequenceDiagram
    participant Agent as "Agent Client"
    participant Filter as "ApiKeyAuthenticationFilter"
    participant Controller as "AgentTaskController"
    participant StatusSvc as "AgentTaskStatusService"
    participant Queue as "TaskQueueManager"
    participant CreateMgr as "CreateNovelTaskManager"
    participant DB as "DB"

    Agent->>Filter: GET /api/agent/task/{taskId}/status
    Filter->>DB: 校验 X-API-Key -> user
    DB-->>Filter: user / auth info
    Filter->>Controller: 已认证请求
    Controller->>StatusSvc: getTaskStatus(taskId)
    StatusSvc->>Queue: findTaskById(taskId)
    alt 队列中存在
        Queue-->>StatusSvc: TaskQueueItem
    else 队列中不存在
        StatusSvc->>CreateMgr: getTaskStatus(taskId) / getCurrentTaskId()
        alt 创建任务内存中存在
            CreateMgr-->>StatusSvc: CREATE task status
        else 仍未命中
            StatusSvc->>DB: select user_task by taskId
            DB-->>StatusSvc: UserTask / null
        end
    end
    StatusSvc-->>Controller: TaskStatusResponse / TASK_NOT_FOUND
    Controller-->>Agent: 查询结果
```

---

## 4. API Key 管理

### 4.1 生成 API Key

```mermaid
sequenceDiagram
    participant Web as "Web/JWT Client"
    participant JWT as "JWT/Auth Filter"
    participant Controller as "ApiKeyController"
    participant UserSvc as "UserService"
    participant ApiKeySvc as "ApiKeyService"
    participant DB as "DB"

    Web->>JWT: POST /api/novel-auth/api-keys
    JWT-->>Controller: 已认证请求 + Authentication
    Controller->>UserSvc: getUserIdByUsername(authentication.name)
    UserSvc->>DB: 查询 userId
    DB-->>UserSvc: userId
    Controller->>ApiKeySvc: createApiKey(userId, name)
    ApiKeySvc->>DB: 插入 api_key 记录
    DB-->>ApiKeySvc: ApiKey
    ApiKeySvc-->>Controller: ApiKey
    Controller-->>Web: API Key 创建成功
```

### 4.2 查询 API Key 列表

```mermaid
sequenceDiagram
    participant Web as "Web/JWT Client"
    participant JWT as "JWT/Auth Filter"
    participant Controller as "ApiKeyController"
    participant UserSvc as "UserService"
    participant ApiKeySvc as "ApiKeyService"
    participant DB as "DB"

    Web->>JWT: GET /api/novel-auth/api-keys
    JWT-->>Controller: 已认证请求 + Authentication
    Controller->>UserSvc: getUserIdByUsername(authentication.name)
    UserSvc->>DB: 查询 userId
    DB-->>UserSvc: userId
    Controller->>ApiKeySvc: listApiKeys(userId)
    ApiKeySvc->>DB: 查询 api_key 列表
    DB-->>ApiKeySvc: List<ApiKey>
    ApiKeySvc-->>Controller: List<ApiKey>
    Controller-->>Web: 查询成功
```

### 4.3 删除 API Key

```mermaid
sequenceDiagram
    participant Web as "Web/JWT Client"
    participant JWT as "JWT/Auth Filter"
    participant Controller as "ApiKeyController"
    participant UserSvc as "UserService"
    participant ApiKeySvc as "ApiKeyService"
    participant DB as "DB"

    Web->>JWT: DELETE /api/novel-auth/api-keys/{id}
    JWT-->>Controller: 已认证请求 + Authentication
    Controller->>UserSvc: getUserIdByUsername(authentication.name)
    UserSvc->>DB: 查询 userId
    DB-->>UserSvc: userId
    Controller->>ApiKeySvc: deleteApiKey(id, userId)
    ApiKeySvc->>DB: 查询并校验 Key 归属
    alt 归属当前用户
        ApiKeySvc->>DB: 删除 api_key
        DB-->>ApiKeySvc: success
        ApiKeySvc-->>Controller: success
        Controller-->>Web: 删除成功
    else 非当前用户
        ApiKeySvc-->>Controller: 403 Forbidden
        Controller-->>Web: 无权删除该 API Key
    end
```

---

## 5. 总结

统一规律如下：

1. Agent 业务接口统一先经过 `ApiKeyAuthenticationFilter` 做 `X-API-Key` 认证，再进入 Controller。
2. 读类接口主要落到 `NovelAppQueryFacade` 或 `AgentTaskStatusService`。
3. 写类接口分两种：
   - 直接修改配置：同步调用 Facade / Service
   - 创建、构建、发布：先返回 `taskId`，再走异步任务链路
4. API Key 管理接口不走 `X-API-Key`，而是走 JWT 鉴权后通过 `ApiKeyController` 直接调用 `ApiKeyService`。
