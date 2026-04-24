# 22 - 新老接口对比

## 说明

本文对比以下两组接口：

- 旧网页接口
  - `/api/novel-create`
  - `/api/novel-build`
  - `/api/novel-publish`
- 新 Agent 接口
  - `/api/agent/app`
  - `/api/agent`

目标是回答一个问题：

> 旧网页接口和现在 Agent 接口，到底有什么区别？

---

## 一、结论先行

从第一性原理看，两组接口的核心业务能力是重叠的：

- 创建应用
- 触发构建
- 触发发布
- 查询任务状态

但它们的定位不同：

- 旧网页接口：为前端页面服务，偏“人工操作 + WebSocket + 页面交互”
- 新 Agent 接口：为 AI Agent / 脚本服务，偏“API 调用 + 轮询 taskId + 聚合能力”

一句话总结：

- 旧接口更像“后台管理页面接口”
- 新接口更像“可编排的机器调用接口”

---

## 二、总览对比

| 维度 | 旧网页接口 | 新 Agent 接口 |
|------|------------|---------------|
| **调用对象** | 后台网页前端 | AI Agent / 脚本 / 自动化 |
| **认证方式** | JWT / Session 风格页面调用 | `X-API-Key` |
| **接口组织方式** | 创建、构建、发布分散在 3 个模块 | 应用管理聚合到 `/api/agent/app`，构建发布聚合到 `/api/agent` |
| **返回风格** | 偏页面使用，接口风格不完全统一 | 更统一，围绕 `taskId` + 轮询 |
| **查询能力** | 分散，页面驱动 | 聚合查询、详情、更新、创建、删除更完整 |
| **任务查询** | 旧链路更多依赖页面/WebSocket/各自逻辑 | 统一走 `/api/agent/task/{taskId}/status` |
| **审计日志** | 已接入 `@OperationLog` | 现已补齐 `@OperationLog` |
| **业务归属** | 无 | 目前也无，刻意与旧接口保持一致 |

---

## 三、创建接口对比

### 1. 路径

旧接口：

```text
POST /api/novel-create/createNovelApp
```

新接口：

```text
POST /api/agent/app/create
```

### 2. 共同点

两者本质上都做同一件事：

1. 校验参数
2. 检查同名同平台是否已存在
3. 生成 `taskId`
4. 异步执行创建流程
5. 执行数据库、本地代码、资源文件操作

换句话说，创建业务内核没有本质变化。

### 3. 主要区别

| 项目 | 旧网页接口 | 新 Agent 接口 |
|------|------------|---------------|
| **认证** | `@PreAuthorize`，依赖网页登录态 | `X-API-Key` |
| **接口语义** | 单一创建入口 | 属于 Agent 应用管理的一部分 |
| **错误码风格** | 偏 `Result.error("...")` 文本返回 | 更明确的 `errorCode`，如 `INVALID_PARAMS` / `CREATE_IN_PROGRESS` / `DUPLICATE_APP` |
| **配套能力** | 只负责创建 | 创建后可直接用 Agent 查询、详情、更新、删除接口继续操作 |

### 4. 业务归属对比

这一点尤其重要：

- 旧网页创建接口虽然有审计日志，但主业务链路不知道“是谁创建的”
- 新 Agent 创建接口目前也一样

也就是说，两者当前都属于：

- 有审计
- 无业务归属

具体表现：

1. `user_op_log` 能记录谁调用了接口
2. 但 `user_task` 创建任务未写 `userId`
3. `novel_app` 也没有 `created_by` 字段

所以在“创建人归属”这个问题上，新接口并没有比旧接口更强。

---

## 四、构建接口对比

### 1. 路径

旧接口：

```text
POST /api/novel-build/build
```

新接口：

```text
POST /api/agent/build
```

### 2. 共同点

两者最后都进入 `TaskQueueManager` 队列，核心机制一致：

1. 认证用户
2. 生成 `taskId`
3. 构造 `TaskQueueItem`
4. 入队
5. 后台异步执行构建

所以构建任务引擎本身没有分叉成两套。

### 3. 主要区别

| 项目 | 旧网页接口 | 新 Agent 接口 |
|------|------------|---------------|
| **参数形式** | `@RequestParam String cmd` | JSON：`{ "cmd": "..." }` |
| **认证方式** | 页面登录态 + `@PreAuthorize` | `X-API-Key` |
| **返回格式** | 直接返回字符串 `taskId` | 返回 `{ taskId }` 对象 |
| **使用方式** | 页面多配合 WebSocket 日志订阅 | Agent 更适合脚本化调用和轮询 |

### 4. 旧接口比新接口多的能力

旧 `NovelAppBuildController` 当前能力更全，除了单次构建，还有：

- 停止构建：`GET /api/novel-build/stop`
- 批量构建：`POST /api/novel-build/batch-build`
- 批量测试构建：`POST /api/novel-build/batch-build-test`
- 停止批量构建：`GET /api/novel-build/stop-batch`
- 手动触发队列：`GET /api/novel-build/trigger-queue`

新 Agent 版本目前只暴露了：

- `POST /api/agent/build`

所以如果从“能力覆盖面”看：

- 旧网页构建接口更全
- 新 Agent 构建接口更轻、更适合标准化调用

---

## 五、发布接口对比

### 1. 路径

旧接口：

```text
POST /api/novel-publish/publish
```

新接口：

```text
POST /api/agent/publish
```

### 2. 共同点

发布主流程也没有分成两套独立引擎。两者都会：

1. 校验平台、appId、projectPath、version、log
2. 获取当前用户
3. 生成 `taskId`
4. 构造发布任务
5. 放入 `TaskQueueManager`

### 3. 主要区别

| 项目 | 旧网页接口 | 新 Agent 接口 |
|------|------------|---------------|
| **认证方式** | 页面登录态 + `@PreAuthorize` | `X-API-Key` |
| **返回结构** | 返回 `NovelAppPublishDTO(taskId)` | 返回 `{ taskId }` |
| **页面能力** | 面向后台发布流程 | 面向程序化发布调用 |
| **接口风格** | 更偏页面业务对象 | 更偏脚本调用参数 |

### 4. 旧接口比新接口多的能力

旧 `NovelAppPublishController` 额外提供了很多页面能力：

- 已构建应用列表：`GET /api/novel-publish/list`
- 预览码生成：`POST /api/novel-publish/previewQrCode`
- 停止发布：`POST /api/novel-publish/stop/{taskId}`
- 获取二维码：`GET /api/novel-publish/qrcode/{taskId}`
- 批量发布：`POST /api/novel-publish/batch-publish`
- 批量预览测试：`POST /api/novel-publish/batch-preview-test`
- 停止批量发布：`POST /api/novel-publish/stop-batch/{batchTaskId}`
- 获取批量二维码：`GET /api/novel-publish/qrcode/batch/{batchTaskId}/{appId}`

新 Agent 版本目前只暴露：

- `POST /api/agent/publish`

所以在发布领域：

- 旧网页接口更完整，覆盖后台管理全流程
- 新 Agent 接口只保留最核心的“发起发布任务”

---

## 六、任务状态查询对比

旧接口侧：

- 没有一个统一、简洁、对外明确的 Agent 风格任务状态查询入口
- 更多依赖页面侧自己的任务管理、WebSocket 日志、二维码查询等组合能力

新接口侧：

```text
GET /api/agent/task/{taskId}/status
```

这是 Agent 版本最关键的增强之一。

它统一屏蔽了底层差异，查询顺序为：

1. `TaskQueueManager`
2. `CreateNovelTaskManager`
3. `user_task` 历史记录

因此 Agent 端可以用一套轮询方式统一处理：

- 创建
- 构建
- 发布

这一点是新接口相对旧接口最实用的价值之一。

---

## 七、`source` 传递规则对比

这里的 `source` 指的是 `user_task.source`，不是 `user_op_log.source`。

当前原则很简单：

- 有明确来源，就写明确值
- 没有来源，就写 `unknown`
- 展示层不要再把空值默认成 `web`

### 1. 任务来源总表

| 场景 | 入口接口 | 当前传入值 | 说明 |
|------|----------|------------|------|
| 旧网页创建 | `/api/novel-create/createNovelApp` | `web` | `CreateNovelTaskManager.createTask()` 默认等价于 `createTask("web")` |
| 新 Agent 创建 | `/api/agent/app/create` | `api` | `AgentAppController.createApp()` 显式传 `createTask("api")` |
| 旧网页构建 | `/api/novel-build/build`、`/api/novel-build/batch-build`、`/api/novel-build/batch-build-test` | `web` | 创建 `TaskQueueItem` 时显式设置 |
| 新 Agent 构建 | `/api/agent/build` | `api` | 创建 `TaskQueueItem` 时显式设置 |
| 旧网页发布 | `/api/novel-publish/publish`、`/api/novel-publish/batch-publish`、`/api/novel-publish/batch-preview-test` | `web` | 创建 `TaskQueueItem` 时显式设置 |
| 新 Agent 发布 | `/api/agent/publish` | `api` | 创建 `TaskQueueItem` 时显式设置 |
| 兜底情况 | 所有任务入口 | `unknown` | 没传来源时，`TaskQueueManager` 不再默认记成 `web` |

### 2. 创建链路对比

| 项目 | 旧网页创建接口 | 新 Agent 创建接口 |
|------|---------------|-----------------|
| 路径 | `/api/novel-create/createNovelApp` | `/api/agent/app/create` |
| 调用链 | `NovelAppCreateController.createNovelApp()` → `CreateNovelTaskManager.createTask()` | `AgentAppController.createApp()` → `CreateNovelTaskManager.createTask("api")` |
| `source` 值 | `web` | `api` |
| 结论 | 网页创建任务按 `web` 记 | Agent 创建任务按 `api` 记 |

### 3. 构建链路对比

| 项目 | 旧网页构建接口 | 新 Agent 构建接口 |
|------|---------------|-----------------|
| 路径 | `/api/novel-build/build`、`/api/novel-build/batch-build`、`/api/novel-build/batch-build-test` | `/api/agent/build` |
| 任务承载 | `TaskQueueItem` | `TaskQueueItem` |
| `source` 值 | `web` | `api` |
| 落库方式 | 由 `TaskQueueManager` 写入 `user_task.source` | 由 `TaskQueueManager` 写入 `user_task.source` |

### 4. 发布链路对比

| 项目 | 旧网页发布接口 | 新 Agent 发布接口 |
|------|---------------|-----------------|
| 路径 | `/api/novel-publish/publish`、`/api/novel-publish/batch-publish`、`/api/novel-publish/batch-preview-test` | `/api/agent/publish` |
| 任务承载 | `TaskQueueItem` | `TaskQueueItem` |
| `source` 值 | `web` | `api` |
| 落库方式 | 由 `TaskQueueManager` 写入 `user_task.source` | 由 `TaskQueueManager` 写入 `user_task.source` |

### 5. 落库与展示规则

| 环节 | 规则 |
|------|------|
| 数据库落库 | `TaskQueueManager` 读取 `TaskQueueItem.source` 写入 `user_task.source` |
| 空值兜底 | 没有传来源时，统一记为 `unknown` |
| 前端展示 | `source` 有值就原样显示，空值显示 `unknown` |
| 语义原则 | 入口传什么，数据库就记什么；入口没传，就不要假装成 `web` |

---

## 七、是否可以统一成一套接口

可以，而且从架构上更合理。

### 1. 统一目标

目标不是把 Web 和外部调用混成一种客户端，而是：

- Web 前端和外部程序都调用同一套后端接口
- 后端只保留一套业务入口
- 任务日志、状态查询、权限控制都复用同一套实现

### 2. 推荐统一方向

建议把 **新 Agent 接口** 作为主入口：

- `/api/agent/app/create`
- `/api/agent/app/update`
- `/api/agent/app/delete`
- `/api/agent/build`
- `/api/agent/publish`
- `/api/agent/task/{taskId}/status`

Web 前端也可以直接调用这套接口，不必再依赖旧网页接口。

### 3. WebSocket 和外部调用如何共存

这里要区分“发起任务”和“消费任务过程”：

- 发起任务：统一走同一套 API
- 消费过程：
  - Web 前端可以继续订阅 WebSocket，看实时日志
  - 外部程序通常不需要 WebSocket，只轮询任务状态接口即可

也就是说：

- WebSocket 是前端体验层，不是接口体系的一部分
- 外部 API 调用不接 WebSocket 也完全正常

### 4. 为什么这样可行

因为当前日志推送本身就是按 `taskId` 组织的，而不是按“旧接口/新接口”组织的：

- 创建日志：`/topic/novel-create-log/{taskId}`
- 构建日志：`/topic/build-logs/{taskId}`
- 发布日志：`/topic/publish-logs/{taskId}`

只要新接口也返回 `taskId`，Web 前端就可以继续按 `taskId` 订阅日志。

外部程序则直接跳过 WebSocket，改为：

- 调接口拿 `taskId`
- 轮询 `/api/agent/task/{taskId}/status`

### 5. 统一后的边界

如果要真正做到“一套到底”，需要满足两点：

- 新接口补齐旧网页接口的能力差异
- Web 前端迁移到新接口后，旧接口只保留过渡兼容

这样后面就不会再出现：

- 一套接口给网页
- 另一套接口给外部

的双轨分叉问题。

### 6. 最终结论

- 可以统一成一套接口
- Web 和外部都可以调用同一套新接口
- Web 继续用 WebSocket 看日志
- 外部只轮询任务状态，不需要 WebSocket
- 旧接口适合做过渡层，不适合长期并行维护

---

## 七、认证与审计对比

### 1. 认证方式

旧网页接口：

- 面向网页用户
- 依赖登录后的 JWT / Spring Security 登录态
- 常见形式是 `@PreAuthorize`

新 Agent 接口：

- 面向 Agent / 脚本
- 通过 `X-API-Key` 认证
- 由 `ApiKeyAuthenticationFilter` 将请求映射为当前用户

### 2. 审计日志

旧网页接口：

- 已有 `@OperationLog`

新 Agent 接口：

- 之前缺失
- 现在已经补齐 `@OperationLog`

因此现在两组接口在审计层面是一致的：

- 谁调用了接口
- 调了哪个 URL
- 传了什么参数
- 返回了什么结果

这些都可以进入 `user_op_log`

---

## 八、为什么要有 Agent 新接口

如果旧接口已经能做创建、构建、发布，为什么还要有 Agent 版本？

原因不是“旧接口不能做”，而是“旧接口不适合机器调用”。

Agent 接口的价值主要在于：

1. **认证更适合程序**
   - 用 `X-API-Key`
   - 不依赖网页登录流程

2. **接口组织更适合编排**
   - `/api/agent/app/*`
   - `/api/agent/build`
   - `/api/agent/publish`
   - `/api/agent/task/{taskId}/status`

3. **任务轮询模型更统一**
   - 先拿 `taskId`
   - 再统一查状态

4. **聚合能力更完整**
   - 查询、详情、编辑、创建、删除都在 Agent 视角下可直接串联

所以新接口不是简单“复制旧接口”，而是把旧网页能力整理成一套更适合自动化系统使用的 API 形态。

---

## 九、当前差异的本质

把所有差异压缩成一句话：

- **旧接口重页面交互**
- **新接口重自动化编排**

再具体一点：

- 旧接口能力更散、更全、更贴后台页面
- 新接口能力更收敛、更标准、更贴 AI Agent

所以两者不是简单替换关系，而是：

- 旧接口仍然适合网页后台
- 新接口适合 Agent / API / 脚本调用

---

## 十、建议

当前最合理的策略不是删除旧接口，而是明确边界：

1. 旧接口继续服务后台页面
2. 新接口继续服务 Agent 和自动化调用
3. 新增能力优先考虑是否需要同步暴露 Agent 版本

后续若要继续收敛，建议优先补齐 Agent 侧缺失但高价值的能力：

1. 构建停止
2. 发布停止
3. 已构建应用列表
4. 预览/二维码能力
5. 批量构建 / 批量发布

这样 Agent 接口才能逐步从“核心骨架”演进成“完整自动化接口层”。
