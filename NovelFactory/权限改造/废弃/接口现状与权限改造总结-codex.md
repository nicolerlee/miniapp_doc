# 接口现状与权限改造总结

## 1. 目的

这份文档只回答 4 个问题：

1. `NovelAppManagerServer` 现在到底有多少接口
2. 当前权限控制真实落在哪几层
3. 哪些结论已经和旧文档不一致
4. 下一步权限改造应该先改什么，后改什么

口径说明：

- 统计范围：`src/main/java/com/fun/novel/controller`
- 统计对象：方法级 `@GetMapping / @PostMapping / @PutMapping / @DeleteMapping / @PatchMapping`
- 不把类级 `@RequestMapping` 算成接口

---

## 2. 当前接口总量

当前后端一共 **126 个 HTTP 接口**，分布在 **25 个 Controller** 中。

### 2.1 各 Controller 接口数量

| Controller | 接口数 | 路径前缀 |
|---|---:|---|
| `AppWeijuController` | 14 | `/api/novel-weiju` |
| `CodeSyncController` | 13 | `/api/code-sync` |
| `PreviewController` | 11 | `/api/preview` |
| `NovelAppController` | 9 | `/api/novel-apps` |
| `NovelAppPublishController` | 9 | `/api/novel-publish` |
| `AuthController` | 7 | `/api/novel-auth` |
| `AgentAppController` | 6 | `/api/agent/app` |
| `AppAdController` | 6 | `/api/novel-ad` |
| `DatabaseExportController` | 6 | `/api/database` |
| `NovelAppBuildController` | 6 | `/api/novel-build` |
| `AppCommonConfigController` | 5 | `/api/novel-common` |
| `AppUiController` | 5 | `/api/novel-ui` |
| `AppWeijuMockController` | 5 | `/api/weiju-mock` |
| `TaskQueueController` | 5 | `/api/task-queue` |
| `AppPayController` | 4 | `/api/novel-pay` |
| `AgentBuildPublishController` | 3 | `/api/agent` |
| `ApiKeyController` | 3 | `/api/novel-auth/api-key` |
| `AgentTaskController` | 2 | `/api/agent` |
| `OpLogController` | 2 | `/api/op-log` |
| `NovelAppCreateController` | 1 | `/api/novel-create` |
| `NovelSearchController` | 1 | `/api/test` |
| `OpenApiController` | 1 | `/open/v1` |
| `SystemConfigController` | 1 | `/api/config` |
| `ToolBoxController` | 1 | `/api/novel-toolbox` |
| `TestController` | 0 | `/api/test` |

---

## 3. 当前权限控制模型

当前权限不是单点控制，而是 **三层叠加**：

### 3.1 第一层：`SecurityFilterChain` 路由级准入

文件：

- [SecurityConfig.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/config/SecurityConfig.java)

规则：

- `URL_WHITELIST` 里的路径直接 `permitAll`
- 其余请求统一 `anyRequest().authenticated()`

这意味着：

- 不是所有未加 `@PreAuthorize` 的接口都等于“裸奔”
- 很多接口虽然没写角色注解，但仍然要求先通过认证

### 3.2 第二层：认证方式

当前系统有两条认证链路：

- 后台网页链路：JWT
- Agent / OpenAPI 链路：`X-API-Key` 或 Bearer API Key

关键文件：

- [DynamicJwtAuthenticationFilter.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/security/DynamicJwtAuthenticationFilter.java)
- [ApiKeyAuthenticationFilter.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/security/ApiKeyAuthenticationFilter.java)
- [BearerApiKeyAuthenticationFilter.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/security/BearerApiKeyAuthenticationFilter.java)

`ApiKeyAuthenticationFilter` 当前已经做了两件关键事：

- 校验 API Key 是否有效、是否启用
- 校验 `user.status`，未审核/审核拒绝账号不能继续访问

### 3.3 第三层：方法级角色控制

项目当前主要使用 `@PreAuthorize("hasAnyRole(...)")` 做方法级角色限制。

从代码扫描结果看：

- 方法级接口总数：**126**
- 带 `@PreAuthorize` 的接口：**49**
- 不带 `@PreAuthorize` 的接口：**77**

注意：

- 这 **77** 个接口里，一部分命中白名单，另一部分只是“已认证即可访问”
- 所以“没写 `@PreAuthorize`” 和 “公开接口” 不是一回事

---

## 4. 当前接口按权限形态划分

为了避免把“白名单公开”和“已认证即可访问”混为一谈，这里只按准入形态拆分。

### 4.1 命中 `URL_WHITELIST` 的接口：24 个

这 24 个接口在路由层是 `permitAll`。

典型例子：

- 登录注册：`/api/novel-auth/login`、`/api/novel-auth/register`
- 公共查询：`/api/novel-apps/appLists`
- 预览相关：`/api/preview/**`
- 操作日志：`/api/op-log/**`
- 微剧 mock：`/api/weiju-mock/getBannersByBannerId`、`/api/weiju-mock/getDeliverByDeliverId`

但这里有一个必须单独指出的问题：

- 有 **5 个接口** 同时命中 `URL_WHITELIST` 和 `@PreAuthorize`
- 代表“路由层放开了，但方法层又在收紧”
- 这种双重配置本身不会自动说明谁更合理，但它会让权限边界不直观，后续维护成本高

这 5 个接口是：

- `/api/novel-ad/appAd/getAppAdByAppId`
- `/api/novel-common/getAppCommonConfig`
- `/api/novel-pay/getAppPayByAppId`
- `/api/novel-weiju/banner/getBannerByBannerId`
- `/api/novel-apps/appLists`

### 4.2 仅要求“已认证”的接口：58 个

这类接口的特点是：

- 不在白名单
- 也没有方法级 `@PreAuthorize`
- 因此当前规则等价于：**登录即可访问 / 携带有效 API Key 即可访问**

典型风险接口：

- `CodeSyncController` 全部 13 个接口
- `TaskQueueController` 全部 5 个接口
- `ApiKeyController` 全部 3 个接口
- `POST /api/novel-build/build`
- `POST /api/novel-publish/publish`
- `POST /api/novel-create/createNovelApp`
- `POST /api/novel-apps/update`

这一类是当前权限改造最需要优先收口的区域。

### 4.3 带方法级角色限制的接口：49 个

这类接口的特点是：

- 已经显式写了角色矩阵
- 规则相对清晰
- 适合后续抽象成统一权限码

代表性接口：

- Agent 查询/编辑/创建/删除
- Agent 构建/发布/任务查询
- 旧链路的批量构建、批量发布、数据库导出
- 微剧、UI、支付、广告部分编辑接口

---

## 5. 旧文档中已经过时的结论

目录 `docs/agent` 下面的若干文档里有一个旧结论：

- “Agent 接口当前大多没有 `@PreAuthorize`，主要依赖已认证即可访问”

这个结论 **现在已经不成立**。

以当前代码为准：

- [AgentAppController.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/controller/AgentAppController.java)
- [AgentBuildPublishController.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/controller/AgentBuildPublishController.java)
- [AgentTaskController.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/controller/AgentTaskController.java)

当前 Agent 链路的真实状态是：

- `GET /api/agent/app/list`：仅认证
- `GET /api/agent/app/query`：`ROLE_SUPER_ADMIN / ROLE_0 / ROLE_1 / ROLE_2`
- `GET /api/agent/app/detail`：`ROLE_SUPER_ADMIN / ROLE_0 / ROLE_1 / ROLE_2`
- `POST /api/agent/app/update`：`ROLE_SUPER_ADMIN / ROLE_0 / ROLE_1 / ROLE_2`
- `POST /api/agent/app/create`：`ROLE_SUPER_ADMIN / ROLE_0 / ROLE_1`
- `DELETE /api/agent/app/delete`：`ROLE_SUPER_ADMIN / ROLE_0`
- `POST /api/agent/build`：`ROLE_SUPER_ADMIN / ROLE_0 / ROLE_1 / ROLE_2`
- `POST /api/agent/publish`：`ROLE_SUPER_ADMIN / ROLE_0 / ROLE_1 / ROLE_2`
- `GET /api/agent/publish/qrcode/{taskId}`：`ROLE_SUPER_ADMIN / ROLE_0 / ROLE_1 / ROLE_2`
- `GET /api/agent/task/{taskId}/status`：`ROLE_SUPER_ADMIN / ROLE_0 / ROLE_1 / ROLE_2`
- `GET /api/agent/task/{taskId}/logs`：`ROLE_SUPER_ADMIN / ROLE_0 / ROLE_1 / ROLE_2`

所以当前问题不是“Agent 完全没做角色控制”，而是：

- **新老接口的控制粒度仍然不统一**
- **一部分高风险旧接口仍然只做了 authenticated，没有做角色限制**

---

## 6. 当前真正的权限问题

从第一性原理看，权限问题只分三类。

### 6.1 问题一：同类能力在新老接口上的权限边界不一致

例如：

- Agent 创建：`ROLE_SUPER_ADMIN / ROLE_0 / ROLE_1`
- 老创建：`ROLE_0 / ROLE_1`

又如：

- Agent 构建/发布：允许到 `ROLE_2`
- 老构建/发布：很多地方仍是 `ROLE_0 / ROLE_1`

这会导致：

- 同一个业务动作，因为入口不同，权限结果不同
- 前端切换新老接口时，行为可能发生隐式变化

### 6.2 问题二：大量高风险接口只有 authenticated，没有角色收口

最典型的是：

- 代码同步
- 任务队列调度
- API Key 管理
- 构建 / 发布 / 创建 / 编辑的部分旧接口

这类接口如果没有更细角色边界，实际上就是“任何已登录用户都能操作”。

### 6.3 问题三：权限定义散落在路径白名单、过滤器、`@PreAuthorize` 三处

后果：

- 权限规则很难一眼看清
- 文档极易过时
- 同一个接口容易出现“白名单放开 + 方法再限制”的双重配置

---

## 7. 改造目标

改造目标不应该是“把所有接口都改成 Agent”或者“把所有权限都塞进注解”。

正确目标只有两个：

1. **同类业务能力只有一套权限语义**
2. **高风险接口必须显式可读，不能靠默认 authenticated 混过去**

落到实现上，就是：

- 认证方式可以继续双轨并存：JWT + API Key
- 但授权语义必须统一
- 同时逐步从“角色硬编码”演进到“权限码”

---

## 8. 建议的改造顺序

### 8.1 第一阶段：先把高风险 authenticated-only 接口补齐角色限制

优先级最高：

- `CodeSyncController`
- `TaskQueueController`
- `ApiKeyController`
- `NovelAppBuildController.build`
- `NovelAppPublishController.publish`
- `NovelAppCreateController.createNovelApp`
- `NovelAppController.update`

原则：

- 先把边界补上
- 暂时不做大重构
- 先消除“登录即可做高风险操作”的口子

### 8.2 第二阶段：把新老同类接口权限矩阵拉齐

至少统一以下能力：

- 创建
- 编辑
- 删除
- 构建
- 发布
- 任务查询
- 应用查询

建议先出一张“能力 -> 角色矩阵”，再改代码，不要继续按接口零散补注解。

### 8.3 第三阶段：从角色注解过渡到权限码

当角色矩阵稳定后，再引入：

- `permission_code`
- `user_permission`
- `@RequirePermission(...)`

原因很简单：

- 现在直接上权限码，等于把当前不统一的问题原样复制一遍
- 先统一语义，再抽象模型，风险更低

---

## 9. 最小可执行方案

如果只做一轮最小改造，建议只做下面 4 件事：

1. 给所有高风险 authenticated-only 接口补 `@PreAuthorize`
2. 清理白名单与 `@PreAuthorize` 重叠的 5 个接口，明确它们到底应该公开还是受控
3. 输出一张“业务能力 -> 角色矩阵”并作为唯一口径
4. 等角色矩阵稳定后，再决定是否引入权限码体系

---

## 10. 结论

当前 `NovelAppManagerServer` 的核心事实是：

- 一共 **126 个接口**
- 权限控制已经存在，但 **不统一**
- 当前最大问题不是“完全没鉴权”，而是：
  - 同类能力新老接口口径不同
  - 大量高风险接口只要求 authenticated
  - 白名单、过滤器、`@PreAuthorize` 三处同时配，维护成本高

所以这次“接口权限改造”的正确路线不是先做大设计，而是：

- **先收口高风险接口**
- **再统一新老能力的授权语义**
- **最后再抽象成权限码体系**
