1. JWT是怎么用的？ 网页调用接口的时候， 是怎么把token带上去的？
2. 新建小程序是怎么查进度的？

进度追踪分两条通道：**HTTP 轮询任务状态** + **WebSocket 实时日志推送**。

### 调用时序图

```
前端 AutoCreate.vue          后端 NovelAppCreateController      CreateNovelTaskManager
        |                               |                               |
        |-- POST /api/novel-create/createNovelApp ------------------>  |
        |                               |-- createTask() ----------->  |
        |                               |<-- taskId (UUID) ----------  |
        |<-- { taskId } --------------- |                               |
        |                               |                               |
        |  (异步线程启动，500ms 后开始执行)                              |
        |                               |-- setTaskRunning(taskId) -->  |
        |                               |   ... 执行创建流程 ...        |
        |                               |-- setTaskCompleted/Failed --> |
        |                               |-- removeTask(taskId) ------>  |
        |                               |                               |

前端 AutoCreateGenerateApp.vue    WebSocket /topic/novel-create-log/{taskId}
        |                               |
        |-- 建立 SockJS 连接 ---------->|
        |-- subscribe /topic/novel-create-log/{taskId} ------------>  |
        |                               |                               |
        |<-- 实时日志消息 (type/message/timestamp) -------------------- |
        |   渲染日志时间线 UI                                            |

前端 agentManager (轮询)         后端 AgentTaskController
        |                               |
        |-- GET /api/agent/task/{taskId}/status (每隔 N 秒) ------->  |
        |                               |-- AgentTaskStatusService.getTaskStatus()
        |                               |     1. 查 TaskQueueManager (构建/发布任务)
        |                               |     2. 查 CreateNovelTaskManager 内存状态
        |                               |        - taskStatusMap (RUNNING/COMPLETED/FAILED)
        |                               |        - 兜底：currentTaskId 匹配 → RUNNING
        |                               |     3. 查数据库 user_task 表 (历史任务)
        |<-- TaskStatusResponse --------|
        |   { taskId, status, progress, description, errorCode }
```

### 关键类说明

| 类 | 职责 |
|---|---|
| `NovelAppCreateController` | 接收创建请求，调用 `createTask()` 获取 taskId，立即返回，异步执行创建流程 |
| `CreateNovelTaskManager` | 内存中维护当前任务 ID 和状态映射（TTL 24h），任务完成/失败时同步写库 |
| `CreateNovelTaskLogger` | 通过 `SimpMessagingTemplate` 向 `/topic/novel-create-log/{taskId}` 推送日志 |
| `AgentTaskStatusService` | 按优先级三级查找：内存队列 → CreateNovelTaskManager → `user_task` 表 |
| `AgentTaskController` | `GET /api/agent/task/{taskId}/status`，返回 `TaskStatusResponse` |
| `AutoCreateGenerateApp.vue` | 前端日志页，订阅 WebSocket 实时渲染日志时间线 |

### 任务状态流转

```
SUBMITTED → PENDING → RUNNING → COMPLETED
                              ↘ FAILED
```

完成/失败后状态持久化到 `user_task` 表，内存中保留 24 小时供查询兜底。