# 新老接口 Permission 拉齐方案

## 1. 目标

回答两个问题：

1. 新老接口当前的 permission 差异在哪里
2. 如果要把新 Agent 接口和老网页接口的 permission 拉齐，应该怎么做

---

## 2. 先下结论

当前状态：

- 新老接口 **认证方式不同**
  - 老接口：JWT
  - 新 Agent 接口：API Key

- 新老接口 **账号状态校验已基本对齐**
  - 老接口登录时校验 `user.status`
  - 新 Agent 接口的 `ApiKeyAuthenticationFilter` 也校验 `user.status`

- 新老接口 **角色授权没有对齐**
  - 老接口大量使用 `@PreAuthorize(...)`
  - 新 Agent 接口当前大多没有对应的角色限制

所以当前真实结论是：

**新老接口 permission 现在不一致。**

但这不是底层服务不同，而是：

**入口层的认证/授权边界不同。**

---

## 3. 当前 permission 差异

### 3.0 角色定义与映射关系

系统中的角色通过 `UserDetailsServiceImpl` 根据用户的 `type` 字段分配：

| user.type | 用户类型 | 对应角色 | 审核状态 | 创建方式 | 权限级别 |
|-----------|---------|---------|---------|---------|---------|
| **-1** | 超级管理员 | ROLE_SUPER_ADMIN | 默认审核通过 | 仅数据库/接口提升，禁止注册 | 最高权限（所有权限 + 用户管理） |
| **0** | 研发 | ROLE_0 | 默认审核通过 | 注册 | 完整权限（创建、构建、发布、编辑、删除 + 用户管理） |
| **1** | 产品 | ROLE_1 | 默认审核通过 | 注册 | 应用权限（创建、构建、发布、编辑） |
| **2** | 测试 | ROLE_2 | 默认待审核 | 注册 | 受限权限（构建、发布、编辑） |

**业务逻辑说明**：

- **ROLE_SUPER_ADMIN（超级管理员）**：最高权限，拥有所有角色权限，不允许通过注册接口创建
- **ROLE_0（研发）**：管理员角色，拥有完整的应用管理权限和用户管理权限
- **ROLE_1（产品）**：可以创建和管理应用，但不能删除应用
- **ROLE_2（测试）**：可以构建、发布和编辑应用，但不能创建和删除应用

---

### 3.1 认证层差异

| 维度 | 老接口 | 新 Agent 接口 |
|------|------|------|
| 认证凭证 | JWT | `X-API-Key` |
| 认证入口 | JWT Filter | `ApiKeyAuthenticationFilter` |
| 用户角色来源 | `UserDetailsServiceImpl` | `ApiKeyAuthenticationFilter` |

说明：

- 新接口并不是没有角色
- 新接口通过 API Key 认证后，同样能映射出 `ROLE_SUPER_ADMIN / ROLE_0 / ROLE_1 / ROLE_2`
- 只是当前没有把这些角色充分用于 Controller 授权

补充：

- 当前 API Key 管理模型已收敛为"一用户一把 key"
- 这不会影响 Agent 接口的 permission 拉齐
- 因为 permission 拉齐的关键仍然是 `@PreAuthorize`，不是 key 数量

---

### 3.2 账号状态差异

这块现在已经基本拉齐。

当前规则：

- `status = 0`：允许
- `status = 1`：拒绝
- `status = 2`：拒绝

所以：

- 网页侧未审核账号不能登录
- Agent 侧未审核账号也不能继续使用 API Key

这部分不再是主要差异点。

---

### 3.3 角色授权差异

这是当前最大差异。

#### 创建

老接口：

- `/api/novel-create/createNovelApp`
- `@PreAuthorize("hasAnyRole('ROLE_0','ROLE_1')")`

新接口：

- `/api/agent/app/create`
- 当前未显式配置同等 `@PreAuthorize`

#### 构建

老接口：

- `/api/novel-build/build`
- `@PreAuthorize("hasAnyRole('ROLE_0','ROLE_1')")`

新接口：

- `/api/agent/build`
- 当前未显式配置同等 `@PreAuthorize`

#### 发布

老接口：

- `/api/novel-publish/publish`
- `@PreAuthorize("hasAnyRole('ROLE_0','ROLE_1')")`

新接口：

- `/api/agent/publish`
- 当前未显式配置同等 `@PreAuthorize`

#### 编辑

老接口：

- `/api/novel-apps/update`
- `@PreAuthorize("hasAnyRole('ROLE_0','ROLE_1','ROLE_2')")`

新接口：

- `/api/agent/app/update`
- 当前未显式配置同等 `@PreAuthorize`

#### 删除

老接口：

- `/api/novel-apps/delete`
- `@PreAuthorize("hasAnyRole('ROLE_0')")`

新接口：

- `/api/agent/app/delete`
- 当前未显式配置同等 `@PreAuthorize`

#### 查询

老接口查询权限本来就不完全统一：

- 有的接口开放
- 有的接口限制较弱
- 有的甚至不需要登录

新接口：

- `/api/agent/app/list`
- `/api/agent/app/query`
- `/api/agent/app/detail`
- `/api/agent/task/{taskId}/status`

当前也没有统一显式角色限制。

所以查询类不能简单说"当前一致"。

---

## 4. 当前 permission 的本质问题

当前问题不是：

- 新 Agent 接口认不出用户身份

而是：

- 新 Agent 接口虽然已经能通过 API Key 识别出用户和角色
- 但没有把老接口已有的角色矩阵完整落到 Controller 上

所以本质问题是：

**授权规则缺失**

而不是：

**认证能力缺失**

---

## 5. 如果要拉齐，核心原则是什么

原则只有一句：

**保持 API Key 认证机制不动，只把 Agent 接口的角色授权规则补齐到和老接口一致。**

也就是说：

- 不需要推翻 `ApiKeyAuthenticationFilter`
- 不需要把 Agent 改回 JWT
- 不需要改底层业务服务

真正该改的是：

- Agent Controller 上的 `@PreAuthorize(...)`

---

## 6. 推荐的拉齐方式

### 6.1 第一步：明确新老接口映射关系

先按能力一一映射：

| 能力 | 老接口 | 新接口 |
|------|------|------|
| 创建 | `/api/novel-create/createNovelApp` | `/api/agent/app/create` |
| 构建 | `/api/novel-build/build` | `/api/agent/build` |
| 发布 | `/api/novel-publish/publish` | `/api/agent/publish` |
| 编辑 | `/api/novel-apps/update` | `/api/agent/app/update` |
| 删除 | `/api/novel-apps/delete` | `/api/agent/app/delete` |
| 查询 | 老查询接口集合 | `/api/agent/app/list/query/detail` |
| 任务状态 | 老任务查询链路 | `/api/agent/task/{taskId}/status` |

---

### 6.2 第二步：按新的权限矩阵对齐

写操作推荐矩阵：

| 新 Agent 接口 | 应对齐老接口 | 建议角色 |
|------|------|------|
| `/api/agent/app/create` | `/api/novel-create/createNovelApp` | `ROLE_SUPER_ADMIN, ROLE_0, ROLE_1` |
| `/api/agent/build` | `/api/novel-build/build` | `ROLE_SUPER_ADMIN, ROLE_0, ROLE_1, ROLE_2` |
| `/api/agent/publish` | `/api/novel-publish/publish` | `ROLE_SUPER_ADMIN, ROLE_0, ROLE_1, ROLE_2` |
| `/api/agent/app/update` | `/api/novel-apps/update` | `ROLE_SUPER_ADMIN, ROLE_0, ROLE_1, ROLE_2` |
| `/api/agent/app/delete` | `/api/novel-apps/delete` | `ROLE_SUPER_ADMIN, ROLE_0` |

查询类推荐矩阵：

| 新 Agent 接口 | 建议角色 |
|------|------|
| `/api/agent/app/list` | 不做角色控制（已认证即可访问） |
| `/api/agent/app/query` | `ROLE_SUPER_ADMIN, ROLE_0, ROLE_1, ROLE_2` |
| `/api/agent/app/detail` | `ROLE_SUPER_ADMIN, ROLE_0, ROLE_1, ROLE_2` |
| `/api/agent/task/{taskId}/status` | `ROLE_SUPER_ADMIN, ROLE_0, ROLE_1, ROLE_2` |

---

### 6.3 第三步：在 Agent Controller 上补 `@PreAuthorize`

实现层面就是在这些 Controller 上加角色注解：

- [AgentAppController.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/controller/AgentAppController.java)
- [AgentBuildPublishController.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/controller/AgentBuildPublishController.java)
- [AgentTaskController.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/controller/AgentTaskController.java)

示例：

```java
@PreAuthorize("hasAnyRole('ROLE_SUPER_ADMIN','ROLE_0','ROLE_1')")
```

或：

```java
@PreAuthorize("hasAnyRole('ROLE_SUPER_ADMIN','ROLE_0','ROLE_1','ROLE_2')")
```

这一步才是真正的"permission 拉齐"。

---

## 7. 为什么不应该改 API Key 机制本身

因为 API Key 机制当前已经完成了它该做的事情：

1. 识别当前是谁
2. 识别当前用户的角色
3. 校验当前账号状态
4. 把认证结果放进 `SecurityContext`

这意味着：

**Agent 接口已经具备按角色做授权的技术条件。**

所以要拉齐 permission，不应该去改：

- `ApiKeyAuthenticationFilter`
- `SecurityConfig`
- JWT 机制

而应该直接在 Agent Controller 上补角色限制。

---

## 8. 是否要改 `PermissionInfo`

短答案：

**不是拉齐 permission 的第一步。**

原因：

- `PermissionInfo` 现在只是返回字段
- 当前不参与真实授权
- 现在即使把它改成动态计算，也不等于真正拦住了接口

所以优先级应当是：

1. 先补 `@PreAuthorize`
2. 再考虑要不要把 `PermissionInfo` 计算结果也和角色规则对齐

否则会出现：

- 返回里写着 `canDelete=false`
- 但接口实际上还能删

这种假权限现象。

---

## 9. 推荐的落地顺序

### 方案 A：最小闭环

只做这些：

1. `AgentAppController.create` 加 `ROLE_SUPER_ADMIN/0/1`
2. `AgentBuildPublishController.build` 加 `ROLE_SUPER_ADMIN/0/1/2`
3. `AgentBuildPublishController.publish` 加 `ROLE_SUPER_ADMIN/0/1/2`
4. `AgentAppController.update` 加 `ROLE_SUPER_ADMIN/0/1/2`
5. `AgentAppController.delete` 加 `ROLE_SUPER_ADMIN/0/1`

优点：

- 覆盖最核心的写操作
- 和老接口最重要的权限边界先对齐
- 风险最小

### 方案 B：连查询类一起统一

在方案 A 基础上，再把：

- `/api/agent/app/list` — 不做角色控制（已认证即可访问）
- `/api/agent/app/query` — 加 `ROLE_SUPER_ADMIN/0/1/2`
- `/api/agent/app/detail` — 加 `ROLE_SUPER_ADMIN/0/1/2`
- `/api/agent/task/{taskId}/status` — 加 `ROLE_SUPER_ADMIN/0/1/2`

优点：

- 所有 Agent 接口权限规则可读
- 不再依赖"没写注解所以默认能进"
- 查询类权限边界明确

---

## 10. 不建议的做法

### 10.1 直接让所有 Agent 接口都开放给所有角色

问题：

- 会和老接口权限边界冲突
- 尤其创建 / 删除 这些敏感写操作

### 10.2 直接把 `PermissionInfo` 当权限系统

问题：

- 它现在只是返回字段
- 不会真正拦截请求

### 10.3 去改底层 Service 做角色判断

问题：

- 权限本应优先在入口层拦截
- 放到 service 会让边界混乱
- 也不符合当前项目已有的 Spring Security 风格

### 10.4 允许通过注册接口创建超级管理员

问题：

- 任何人都可以注册超级管理员账号，严重安全漏洞
- 超级管理员只能通过数据库直接插入或由现有超级管理员提升

---

## 11. 最终建议

如果目标是：

**"让新 Agent 接口和老网页接口 permission 拉齐"**

最合理的方案是：

1. 保持 API Key 认证机制不动
2. 新增 ROLE_SUPER_ADMIN（type=-1）超级管理员角色
3. 按新的权限矩阵，补齐 Agent Controller 上的 `@PreAuthorize`
4. 先对齐写操作
5. 再对齐查询类
6. 最后再决定是否让 `PermissionInfo` 跟着返回真实权限

一句话总结：

**拉齐 permission，改的应该是 Agent Controller 的授权注解，而不是 API Key 认证机制。**
