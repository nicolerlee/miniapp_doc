# API Key 认证说明

## 1. 目的

本项目里有两套认证入口：

- 网页/后台管理接口：使用 `JWT`
- Agent 业务接口：使用 `X-API-Key`

两者不是互斥关系，而是针对不同调用方：

- `/api/novel-auth/api-key/**` 是 **管理 API Key**，走 JWT
- `/api/agent/**` 是 **使用 API Key 调业务接口**，走 `X-API-Key`

---

## 2. API Key 管理接口

基础路径：

```text
/api/novel-auth/api-key
```

当前共 3 个接口。

当前业务模型已收敛为：

- 一个用户只保留一把 API Key
- `POST /api/novel-auth/api-key` 语义不是“永远新建”，而是“有则返回，无则创建”
- `GET /api/novel-auth/api-key` 仍保留列表结构以兼容现有调用方，但最多返回 1 条

### 2.1 获取或生成 API Key

```http
POST /api/novel-auth/api-key
```

请求体：

```json
{
  "name": "my-agent-key"
}
```

说明：

- 为当前登录用户获取唯一的 API Key
- 若当前用户尚无 API Key，则创建一把新的
- 若当前用户已有 API Key，则直接返回已有记录
- 创建后的默认状态为 `enabled = 1`
- `name` 参数用于标识这个 API Key 的用途，方便用户识别（例如：`my-agent-key`、`test-key`、`production-key`）

### 2.2 查询 API Key 列表

```http
GET /api/novel-auth/api-key
```

说明：

- 返回当前登录用户名下的 API Key 列表
- 无需请求参数，系统自动根据当前登录用户查询
- 当前列表最多 1 条
- 列表中包含 `enabled / rateLimit / requestCount / createdTime / lastUsedTime`

### 2.3 删除 API Key

```http
DELETE /api/novel-auth/api-key/{id}
```

说明：

- 删除指定 API Key
- 仅允许删除当前用户自己的 key

---

## 3. Agent 接口为什么能识别 API Key

核心原因是：

- [ApiKeyAuthenticationFilter.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/security/ApiKeyAuthenticationFilter.java)

这是一个 Spring Security 过滤器，继承：

```java
OncePerRequestFilter
```

它会在请求进入 Controller 之前执行，完成这些动作：

1. 读取请求头 `X-API-Key`
2. 查询 `api_key` 表
3. 校验 key 是否存在且启用
4. 查询绑定用户
5. 校验用户状态
6. 构造 `Authentication`
7. 放入 `SecurityContextHolder`

一旦放入 `SecurityContext`，后续 Controller / `@PreAuthorize` / `Authentication` 参数注入都能像正常登录用户一样工作。

所以本质上不是 Controller 自己识别 API Key，而是：

**过滤器先把 API Key 转成了 Spring Security 能理解的认证信息。**

---

## 4. 过滤器是怎么接进 Spring Security 的

位置：

- [SecurityConfig.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/config/SecurityConfig.java)

关键代码：

```java
.addFilterBefore(dynamicJwtAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class)
.addFilterBefore(apiKeyAuthenticationFilter(), DynamicJwtAuthenticationFilter.class);
```

实际顺序是：

1. `ApiKeyAuthenticationFilter`
2. `DynamicJwtAuthenticationFilter`
3. `UsernamePasswordAuthenticationFilter`

这意味着：

- 带 `X-API-Key` 的请求，先走 API Key 认证
- 没有 `X-API-Key` 的请求，再继续走 JWT 认证

“API Key 认证优先于 JWT 认证”的意思不是二选一，而是：

**过滤器执行顺序里，API Key 在 JWT 前面。**

---

## 5. API Key 校验链路

`ApiKeyAuthenticationFilter` 当前的核心逻辑顺序如下：

### 5.1 先校验 key 本身

查询条件：

```java
queryWrapper.eq("api_key", apiKeyValue).eq("enabled", 1);
```

含义：

- key 必须存在
- 且必须 `enabled = 1`

如果查不到，直接返回：

```json
{
  "code": 401,
  "errorCode": "UNAUTHORIZED",
  "message": "API Key 无效或已禁用"
}
```

### 5.2 再校验 key 绑定的用户

当 key 合法后，才继续查：

- `user_id`
- `user.status`
- `user.type`

### 5.3 最后才把用户身份放进 Spring Security

映射关系：

- `type = 0` -> `ROLE_0` -> 研发
- `type = 1` -> `ROLE_1` -> 产品
- `type = 2` -> `ROLE_2` -> 测试

---

## 6. 为什么 disabled key 不会返回“账号审核中”

这是两层不同的校验：

### 第一层：校验 key 是否可用

- key 不存在
- key 已禁用

这两种都统一返回：

```text
API Key 无效或已禁用
```

### 第二层：校验用户是否可用

只有 key 本身是合法且启用的，才会继续看用户状态：

- `status = 1` -> `账号正在审核中，请耐心等待`
- `status = 2` -> `账号审核未通过，请联系管理员`

所以：

- `enabled = 0` 的 key 根本不会进入 `user.status` 分支
- 它会在更前面被拦掉

这是合理行为，因为失败原因本来就是：

**key 不可用**

而不是：

**用户状态不可用**

---

## 7. user.status 在 API Key 认证里的作用

网页登录和 API Key 认证现在已经对齐：

- `status = 0`：允许
- `status = 1`：拒绝，提示“账号正在审核中，请耐心等待”
- `status = 2`：拒绝，提示“账号审核未通过，请联系管理员”
- `status = null`：当前保持兼容，不拒绝

说明：

- 正常流程里，待审核/审核拒绝用户本来就无法登录网页，自然也不该生成新的 API Key
- 这层校验主要防止“历史上已经拿到过 key，后来账号状态被改坏”的情况继续调用 Agent 接口

---

## 8. request_count / rate_limit / last_used_time 是什么

`api_key` 表里几个关键字段含义如下：

### 8.1 enabled

是否启用：

- `1`：启用
- `0`：禁用

### 8.2 rate_limit

每分钟请求上限。

例如：

- `60` 表示每分钟最多 60 次请求

### 8.3 request_count

累计请求总数，不是分钟计数。

当前实现是：

```java
apiKey.getRequestCount() + 1
```

所以它会一直累计增长，只要这把 key 持续被使用。

它的作用是长期统计，不是实时限流。

### 8.4 created_time

这把 key 的创建时间。

### 8.5 last_used_time

这把 key 最近一次被使用的时间。

---

## 9. 限流是怎么做的

真正的“每分钟限流”不是靠数据库里的 `request_count`，而是靠过滤器内存里的两个计数器：

- `minuteRequestCount`
- `lastResetMinute`

逻辑是：

1. 取当前分钟
2. 如果跨分钟了，重置本分钟计数
3. 当前请求计数 `+1`
4. 如果超过 `rateLimit`，返回 `429`

所以要区分：

- `request_count`：累计总调用次数，存数据库
- 分钟级限流计数：运行时内存数据，不存数据库

---

## 10. 为什么需要把这些数据存数据库

数据库字段的目的不是做实时限流，而是做长期管理和审计。

例如：

- `request_count`：看这把 key 一共用了多少次
- `last_used_time`：看这把 key 最近一次什么时候使用
- `created_time`：看 key 是什么时候生成的
- `enabled`：控制这把 key 是否仍可继续使用

所以数据库存的是：

**长期状态**

而内存存的是：

**瞬时运行态**

---

## 11. 典型结论

### 11.1 一个用户能不能有多把 key

按当前业务规则，不应该。

当前服务层语义已调整为：

- 有 key 就直接返回
- 没有才创建

为兼容历史数据，如果数据库里仍存在某个用户多条 key，系统当前会按下面优先级选一条作为主 key 返回：

1. `enabled = 1` 优先
2. `last_used_time` 最新优先
3. `created_time` 最新优先
4. `id` 最大优先

数据库层唯一约束已在初始化 SQL [novel_user.sql](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/sql_novel/novel_user.sql) 中定义：

- `api_key.user_id` 唯一索引 `uk_api_key_user_id`

当前收口方式是：

- 服务层：有则返回，无则创建
- 初始化 SQL：`user_id` 唯一

### 11.2 disabled key 请求 Agent 接口会返回什么

返回：

```text
API Key 无效或已禁用
```

不会返回“账号正在审核中”，因为它在 key 校验阶段就已经被拦截。

### 11.3 request_count 会不会一直变大

会。

它是累计总量，不会自动清零。

### 11.4 Agent 接口为什么能直接识别 X-API-Key

因为 `ApiKeyAuthenticationFilter` 已经接入 Spring Security 过滤链，并在请求进入 Controller 前完成了认证。

---

## 12. 相关代码位置

- 认证过滤器：
  [ApiKeyAuthenticationFilter.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/security/ApiKeyAuthenticationFilter.java)

- 安全配置：
  [SecurityConfig.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/config/SecurityConfig.java)

- API Key Controller：
  [ApiKeyController.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/controller/ApiKeyController.java)

- API Key Service：
  [ApiKeyService.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/service/ApiKeyService.java)
  [ApiKeyServiceImpl.java](/Users/nicolerli/nico_ai/NovelFactoryAI_async/NovelAppManagerServer/src/main/java/com/fun/novel/service/impl/ApiKeyServiceImpl.java)
