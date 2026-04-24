# 文曲 API 开放给外部调用方 — 设计方案

## 背景

文曲内部集成各种 API，这是第一层基础设施：新建小程序、编辑小程序、编译小程序、上传发布小程序、删除小程序。

之前这些 API 都是给网页前端用的，现在外部调用方（agent、龙虾等）也需要调用这些 API。

---

## 核心问题

### 问题1：鉴权

**现状**: 网页端通过 JWT 登录获取 token，请求头带 `Authorization: Bearer <token>`。

**问题**: 外部调用方（agent）没有登录流程，不方便走用户名密码换 token 的方式。

**方案: API Key 绑定用户**

核心思路：每个 API Key 绑定到一个具体的系统用户，这样你和小王各自用自己的 Key，后端能区分出是谁在操作。

```
你的 agent   → X-API-Key: ak_zhangsan_xxx → 后端查到 userId=1(张三) → 以张三身份执行
小王的 agent → X-API-Key: ak_xiaowang_xxx → 后端查到 userId=2(小王) → 以小王身份执行
```

**AI/agent 不需要登录，有 API Key 就够了**:
- JWT 登录是给网页前端用的：用户名密码 → 换 token → token 会过期 → 要重新登录
- API Key 是给 agent 用的：直接带 Key 请求 → 后端查到绑定的用户 → 有身份了
- agent 不需要调 `/login`，不需要管 token 过期，每次请求只带 `X-API-Key` 头即可

**数据库设计** — 新建 `api_key` 表:

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT | 主键 |
| api_key | VARCHAR(64) | API Key 值（唯一索引） |
| user_id | BIGINT | 关联的用户ID（外键 → user.id） |
| name | VARCHAR(100) | 备注名（如"张三的agent"） |
| enabled | TINYINT | 是否启用（0/1） |
| rate_limit | INT | 每分钟最大请求数（默认60） |
| request_count | BIGINT | 累计请求次数 |
| created_time | DATETIME | 创建时间 |
| last_used_time | DATETIME | 最后使用时间 |

**认证流程**:

```
请求进来
  │
  ├─ 有 X-API-Key 头
  │   ├─ 查 api_key 表 → 找到记录且 enabled=1
  │   │   ├─ 通过 user_id 查到对应的 User 对象
  │   │   ├─ 将 User 设置到 SecurityContext（和 JWT 登录后一样）
  │   │   ├─ 更新 request_count +1，last_used_time
  │   │   ├─ 检查限流（见问题6）
  │   │   └─ 后续的 @PreAuthorize、操作日志都能拿到正确的用户信息
  │   └─ 未找到或 disabled → 返回 401
  │
  └─ 无 X-API-Key 头 → 走原来的 JWT 认证流程
```

**认证语义**: `/api/agent/**` 路径不放行，必须通过 `ApiKeyAuthenticationFilter` 完成认证后才能访问。未携带有效 API Key 的请求返回 401。

**效果**:
- 操作日志里能看到"张三(通过API Key)创建了小程序"
- 权限控制复用现有的角色体系，API Key 继承绑定用户的角色
- 一个用户可以有多个 API Key（比如一个给 agent，一个给龙虾）
- 管理员可以随时禁用某个 Key，不影响用户的网页登录

**API Key 管理接口**（可选，也可以先手动数据库插入）:
- `POST /api/novel-auth/api-keys` — 为当前用户生成 API Key
- `GET /api/novel-auth/api-keys` — 查看自己的 API Key 列表（含 request_count）
- `DELETE /api/novel-auth/api-keys/{id}` — 删除/禁用某个 Key

**实现要点**:
- 新增 `ApiKeyAuthenticationFilter`，注册到 SecurityConfig，优先级高于 JwtAuthenticationFilter
- Filter 里查 api_key 表，拿到 user_id，再查 User，构造 Authentication 放入 SecurityContext
- 原有的 JWT 认证完全不受影响

---

### 问题2：异步任务结果感知

**现状**: 创建/构建/发布都是异步的，接口只返回 taskId，真正的结果通过 WebSocket (STOMP) 推送给前端。

**问题**: agent 不方便接 WebSocket，需要一个 HTTP 轮询接口来查询任务状态。

**WebSocket 不受影响**: 网页前端继续用 WebSocket 实时看日志，agent 用轮询接口查最终状态，两套并行互不影响。

**任务状态机**:

```
submitted → pending → running → completed
                              → failed
```

| 状态 | 说明 |
|------|------|
| `submitted` | 接口已接受请求，任务对象已创建（仅 create 任务有此状态） |
| `pending` | 已进入队列，等待执行 |
| `running` | 正在执行中 |
| `completed` | 执行成功 |
| `failed` | 执行失败，见 errorCode 和 errorMessage |

**方案: 统一的任务状态轮询接口**

不管是创建、构建还是发布，agent 都用同一个接口查状态：

```
GET /api/agent/task/{taskId}/status
```

响应:
```json
{
  "code": 200,
  "data": {
    "taskId": "uuid-string",
    "taskType": "build | publish | preview | create | npm-install",
    "status": "pending | running | completed | failed",
    "progress": 85,
    "description": "批量构建任务",
    "startTime": "2026-04-13T10:00:00",
    "completeTime": "2026-04-13T10:05:00",
    "errorCode": null,
    "errorMessage": null
  }
}
```

**agent 调用模式 — 三种异步操作完全一样的用法**:

```
创建: POST /api/agent/app/create  → { taskId } → 轮询 GET /api/agent/task/{taskId}/status
构建: POST /api/agent/build       → { taskId } → 轮询 GET /api/agent/task/{taskId}/status
发布: POST /api/agent/publish     → { taskId } → 轮询 GET /api/agent/task/{taskId}/status
```

agent 不需要关心 taskId 是哪种类型的任务，只看 `status` 字段就行。想区分可以看 `taskType`。

**后端内部查找顺序**:

```
收到 taskId
  │
  ├─ 1. 查 TaskQueueManager（构建/发布/npm-install 任务）
  │     ├─ prodTaskQueue / testTaskQueue → status=pending
  │     ├─ prodRunningTasks / testRunningTasks → status=running
  │     └─ completedTasks → status=completed/failed（内存保留24小时）
  │
  ├─ 2. 没找到 → 查 CreateNovelTaskManager（创建小程序任务）
  │     └─ 通过 taskStatusMap 查状态（内存保留24小时）
  │
  └─ 3. 还没找到 → 查数据库 user_task 表（历史任务）
        └─ 找到返回历史状态，找不到返回 TASK_NOT_FOUND
```

**完成态保留策略**: 内存中保留 24 小时，超时后从数据库 `user_task` 表查询。

**创建小程序的特殊处理**:
- 目前 `CreateNovelTaskManager` 只维护了 taskId，没有状态
- 方案A（先做）: 给 CreateNovelTaskManager 加一个 `ConcurrentHashMap<String, TaskStatus> taskStatusMap`，创建开始时设为 running，成功设为 completed，失败设为 failed，同时写入数据库
- 方案B（后续）: 创建小程序也改用 TaskQueueManager 统一管理，彻底消除两套任务管理的差异

---

### 问题3：错误信息

**现状**: 同步接口用 `Result.error("xxx")` 返回错误，异步任务的错误散落在 WebSocket 日志里。

**问题**: agent 需要结构化的错误信息，不能去解析日志文本。

**方案: 结构化错误码 + 错误信息**

在任务状态接口的响应中增加:
```json
{
  "status": "failed",
  "errorCode": "BUILD_FAILED",
  "errorMessage": "构建失败: npm ERR! code ELIFECYCLE"
}
```

**标准错误码定义**:

| errorCode | 说明 | 触发场景 |
|-----------|------|----------|
| `DUPLICATE_APP` | 同名同平台已存在 | 创建小程序 |
| `CREATE_IN_PROGRESS` | 已有创建任务在运行 | 创建小程序 |
| `BUILD_FAILED` | 构建命令执行失败 | 构建小程序 |
| `BUILD_INTERRUPTED` | 构建被中断 | 停止构建 |
| `PUBLISH_TOKEN_MISSING` | 缺少平台Token | 发布小程序 |
| `PUBLISH_FAILED` | 发布失败 | 发布小程序 |
| `TASK_NOT_FOUND` | 任务不存在 | 查询任务状态 |
| `INVALID_PARAMS` | 参数校验失败 | 所有接口 |
| `RATE_LIMIT_EXCEEDED` | 请求频率超限 | 所有接口 |
| `UNAUTHORIZED` | API Key 无效或已禁用 | 所有接口 |

**实现要点**:
- `TaskQueueItem` 增加 `errorCode` 字段
- 各 Processor 在 catch 异常时设置对应的 errorCode
- 任务状态接口返回时带上 errorCode 和 errorMessage

---

### 问题4：Agent 聚合接口

**现状**: 编辑一个小程序的完整配置，前端网页是分模块调不同接口的：

| 编辑内容 | 前端调的接口 |
|---------|-------------|
| 基础信息(名称/版本/appid等) | `POST /api/novel-apps/update` |
| 通用配置(buildCode/token等) | `POST /api/novel-common/updateAppCommonConfig` |
| UI配置(主题色/卡片样式) | `POST /api/novel-ui/updateUiConfig` |
| 支付配置(每种类型一次) | `POST /api/novel-pay/updateAppPay` |
| 广告配置(每种类型一次) | `POST /api/novel-ad/adConfig/update` |
| 微剧Banner | `POST /api/novel-weiju/banner/updateBanner` |
| 微剧Deliver | `POST /api/novel-weiju/deliver/updateDeliver` |

**问题**: agent 要改一个小程序的版本号+主题色+支付，得调 3 个接口，还得知道每个接口的入参结构，太碎了。

**方案: 新增 Agent 聚合接口层**

在 `/api/agent/` 下新建一组面向 agent 的聚合接口，内部复用现有 Service，不重写业务逻辑。

#### 4.1 Agent 接口汇总

| 接口 | 方法 | 响应模式 | 说明 |
|------|------|---------|------|
| `/api/agent/app/list` | GET | 同步，直接返回结果 | 获取可访问应用列表 |
| `/api/agent/app/detail?appId=xxx` | GET | 同步，直接返回结果 | 获取完整配置(聚合所有模块) |
| `/api/agent/app/update` | POST | 同步，直接返回结果 | 聚合编辑(传了就改) |
| `/api/agent/app/create` | POST | 异步，返回 taskId | 创建小程序 |
| `/api/agent/app/delete?appId=xxx` | DELETE | 同步，直接返回结果 | 删除小程序 |
| `/api/agent/build` | POST | 异步，返回 taskId | 构建小程序 |
| `/api/agent/publish` | POST | 异步，返回 taskId | 发布小程序 |
| `/api/agent/task/{taskId}/status` | GET | 同步，直接返回结果 | 查询任务状态(统一入口) |

**响应模式说明**:
- **同步**: 接口执行完直接返回结果，`data` 字段包含业务数据
- **异步**: 接口立即返回 `{ "taskId": "uuid" }`，通过轮询 `/api/agent/task/{taskId}/status` 获取最终结果

#### 4.2 获取可访问应用列表

```
GET /api/agent/app/list
```

**入参** (Query, 可选):

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| platform | String | 否 | 按平台筛选(mp-weixin/mp-toutiao等) |

**出参**:

```json
{
  "code": 200,
  "data": [
    {
      "id": 1,
      "appId": "wx123456",
      "appName": "阅读小说",
      "platform": "mp-weixin",
      "appCode": "novel_wx",
      "version": "1.0.0",
      "product": "小说",
      "customer": "funshion",
      "deliverId": "deliver_001",
      "bannerId": "banner_001"
    }
  ]
}
```

**后端内部**: 调 `NovelAppService.getNovelAppsByPlatform()` 获取所有应用，扁平化返回。

#### 4.3 获取应用详情

```
GET /api/agent/app/detail?appId=wx123456
```

**入参** (Query):

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| appId | String | 是 | 应用ID |

**出参** — 一次返回完整配置：

```json
{
  "code": 200,
  "data": {
    "appId": "wx123456",
    "baseConfig": { "appName": "...", "version": "...", "platform": "...", ... },
    "commonConfig": { "buildCode": "...", "contact": "...", ... },
    "uiConfig": { "homeCardStyle": 1, "mainTheme": "#FF6B35", ... },
    "paymentConfig": { "normalPay": {...}, "orderPay": {...}, ... },
    "adConfig": { "rewardAd": {...}, "bannerAd": {...}, ... },
    "weijuConfig": { "banners": [...], "delivers": [...] }
  }
}
```

**后端内部**: 分别调各 Service 查询，聚合返回：
- `NovelAppService.getByAppId()` → baseConfig
- `AppCommonConfigService.getAppCommonConfig()` → commonConfig
- `AppUIConfigService.getByAppId()` → uiConfig
- `AppPayService.getAppPayByAppId()` → paymentConfig
- `AppAdService.getAppAdByAppId()` → adConfig
- `AppWeijuBannerService` + `AppWeijuDeliverService` → weijuConfig

#### 4.4 聚合编辑接口

```
POST /api/agent/app/update
```

**入参** — 传了就改，没传(null)就不动：

```json
{
  "appId": "wx123456",
  "baseConfig": { "version": "1.0.1" },
  "uiConfig": { "mainTheme": "#FF0000" },
  "paymentConfig": {
    "normalPay": { "enabled": true, "gatewayAndroid": "1", "gatewayIos": "2" }
  }
}
```

完整字段参考 `CreateNovelAppRequest` 结构，所有子对象均可选。

**后端内部分发逻辑**：

```
收到请求，解析 appId
  │
  ├─ baseConfig != null    → 调 NovelAppService.updateNovelApp()
  ├─ commonConfig != null  → 调 AppCommonConfigService.updateAppCommonConfig()
  ├─ uiConfig != null      → 调 AppUIConfigService.updateAppUIConfig() + 文件操作
  ├─ paymentConfig != null → 遍历每种支付类型，调 AppPayService.updateAppPay()
  ├─ adConfig != null      → 遍历每种广告类型，调 AdConfigService.updateAdConfig()
  └─ 所有操作完成 → 返回更新后的完整配置（同 detail 接口结构）
```

**一致性语义**:
- 数据库操作：使用 `@Transactional`，任一数据库操作失败则已执行的数据库操作全部回滚
- 文件操作：使用 `rollbackActions` 补偿回滚，与数据库事务独立
- 两者不在同一个事务中，极端情况下可能出现数据库回滚但文件已修改的情况（概率极低，可接受）

#### 4.5 删除应用

```
DELETE /api/agent/app/delete?appId=wx123456
```

**入参** (Query):

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| appId | String | 是 | 应用ID |

**出参**: `{ "code": 200, "message": "应用删除成功" }`

**后端内部**: 代理到现有的删除逻辑（删数据库 + 删本地代码文件 + 删资源文件）。

---

### 问题5：权限与资源归属

> **当前阶段暂不实现，后续规划。**

待决策：agent 是否只能操作 API Key 绑定用户自己的 app，还是可以访问所有 app？

当前实现：不做隔离，所有 app 对所有有效 API Key 可见（与现有网页端行为一致）。

后续如需隔离，需要在 `novel_app` 表增加 `owner_user_id` 字段，并在 Agent 接口层加过滤逻辑。

---

### 问题6：请求统计与限流

**统计**: `api_key` 表的 `request_count` 字段记录累计调用次数，`last_used_time` 记录最后调用时间。管理员通过 `GET /api/novel-auth/api-keys` 查看。

**限流方案（按 API Key 维度）**:

在 `ApiKeyAuthenticationFilter` 里，用内存计数器按 API Key 控制时间窗口内的请求数：

```
每个 API Key 独立计数
  │
  ├─ 当前分钟内请求数 < rate_limit → 放行
  └─ 当前分钟内请求数 >= rate_limit → 返回 429，errorCode=RATE_LIMIT_EXCEEDED
```

- `rate_limit` 存在 `api_key` 表，默认 60 次/分钟，可按 Key 单独配置
- 计数器用 `ConcurrentHashMap<apiKey, AtomicInteger>` 存内存，每分钟重置
- 你的 Key 超限不影响小王的 Key

---

### 问题7：对现有功能的影响

**结论：不影响。整个方案都是纯新增，不改现有代码的行为。**

#### 7.1 新增的文件

| 文件 | 说明 |
|------|------|
| `AgentAppController.java` | Agent 聚合接口 Controller |
| `AgentTaskController.java` | 任务状态查询 Controller |
| `ApiKeyAuthenticationFilter.java` | API Key 认证 + 限流过滤器 |
| `ApiKey.java` | API Key 实体类 |
| `ApiKeyMapper.java` | API Key 数据库映射 |
| `AgentAppDetailDTO.java` | 聚合查询的返回对象 |
| `AgentAppUpdateRequest.java` | 聚合编辑的请求对象 |

#### 7.2 修改的文件（最小改动）

| 文件 | 改动内容 | 影响 |
|------|---------|------|
| `SecurityConfig.java` | 注册 ApiKeyAuthenticationFilter，`/api/agent/**` 保持 authenticated，由 Filter 完成认证 | 不影响现有 JWT 流程 |
| `CreateNovelTaskManager.java` | 加 `taskStatusMap` 字段和 3 个方法 | 不改现有方法行为 |
| `TaskQueueItem.java` | 加 `errorCode` 字段(默认null) | 现有代码不读此字段 |

#### 7.3 不改动的部分

- 所有现有 Controller **不动**
- 所有现有 Service **不动**
- WebSocket 推送逻辑 **不动**
- 前端代码 **不需要改**
- 数据库现有表 **不动**（只新增 `api_key` 表）

---

## 实施计划

| 阶段 | 内容 | 优先级 | 工作量 |
|------|------|--------|--------|
| **Phase 1** | 新增 API Key 认证 + 限流过滤器 | P0 | 小 |
| **Phase 1** | 新增 `GET /api/agent/task/{taskId}/status` 轮询接口 | P0 | 小 |
| **Phase 1** | CreateNovelTaskManager 增加状态维护 + 写库 | P0 | 小 |
| **Phase 2** | 新增 `GET /api/agent/app/list` 应用列表接口 | P0 | 小 |
| **Phase 2** | 新增 `GET /api/agent/app/detail` 聚合查询接口 | P0 | 中 |
| **Phase 2** | 新增 `POST /api/agent/app/update` 聚合编辑接口 | P0 | 中 |
| **Phase 2** | 新增 create/build/publish/delete 代理接口 | P0 | 小 |
| **Phase 3** | TaskQueueItem 增加 errorCode 字段 | P1 | 小 |
| **Phase 3** | 各 Processor 设置结构化 errorCode | P1 | 中 |
| **Phase 4** | 权限与资源归属（按用户隔离） | P2 | 中 |
| **Phase 4** | 支持 Webhook 回调（可选） | P2 | 中 |
| **Phase 4** | 创建小程序统一接入 TaskQueueManager（可选） | P2 | 大 |

Phase 1 做完：agent 能鉴权 + 限流 + 轮询拿异步结果。
Phase 2 做完：agent 能完整操作小程序（增删改查 + 构建发布）。
Phase 3 做完：错误信息结构化，agent 能精确判断失败原因。
Phase 4 是后续规划，看需要再做。
