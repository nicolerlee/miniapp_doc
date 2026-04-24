# 构建成功误判 Bugfix 说明

## 问题背景

用户通过 `NovelAppManagerServer/agent-scripts/build.sh` 选择 `1` 执行抖音 `tt-fun` 构建时，终端显示：

- 构建任务已加入队列
- 任务状态为 `COMPLETED`
- 最终提示 `构建成功`

但实际没有生成 `dist` 构建产物。

这不是单点问题，而是两个缺陷叠加：

1. 构建入口脚本把菜单项映射到了错误命令
2. 后端任务状态判定只看“是否抛异常”，没有严格校验构建产物

## 根因分析

### 1. 菜单命令映射错误

文件：`NovelAppManagerServer/agent-scripts/build.sh`

原始映射：

```bash
VALS=("npm run build:tt" "npm run build:ks" "npm run build:bd" "npm run build:wx")
```

但实际 `funNovel_miniapp/package.json` 中不存在这些通用脚本，例如：

- 不存在 `build:tt`
- 存在 `build:tt-fun`
- 存在 `build:ks-fun`
- 存在 `build:bd-xingchen`
- 存在 `build:wx-xingchen`

因此，菜单里显示的是 `tt-fun`，真正执行的却是一个不存在的 npm script。

### 2. 后端成功判定过宽

文件：`NovelAppManagerServer/src/main/java/com/fun/novel/utils/TaskQueueManager.java`

上层队列逻辑是：

1. 调用 `executeBuildTask(task)`
2. 如果没有抛异常
3. 直接将任务标记为 `completed`

文件：`NovelAppManagerServer/src/main/java/com/fun/novel/utils/NovelAppBuildUtil.java`

原始实现存在以下问题：

1. 子进程退出码非 0 时，只记录日志，不抛异常
2. 日志中即使出现 `Build failed with errors`，也不一定中断任务
3. 即使构建命令结束后没有任何产物目录，也不会判失败

所以从 Java 调用链来看，这段代码“正常返回”，上层自然把任务标记成 `COMPLETED`。

结论：这是业务状态机设计缺陷，不是编译器能发现的语法错误。

## 修复内容

### 1. 修复构建脚本命令映射

修改文件：

- `NovelAppManagerServer/agent-scripts/build.sh`
- `NovelAppManagerServer/agent-scripts/js/build.js`

修复后映射：

```bash
VALS=("npm run build:tt-fun" "npm run build:ks-fun" "npm run build:bd-xingchen" "npm run build:wx-xingchen")
```

同时更新了脚本注释和帮助示例，避免继续误导调用方。

### 2. 强化构建失败判定

修改文件：

- `NovelAppManagerServer/src/main/java/com/fun/novel/utils/NovelAppBuildUtil.java`

新增和强化的判定规则：

1. 构建进程被中断时，直接抛异常，任务失败
2. 构建进程退出码非 0 时，直接抛异常，任务失败
3. 日志中出现 `Build failed with errors` 时，直接判失败
4. 构建结束后检查预期产物目录是否存在且非空

预期产物目录解析规则：

- `build:tt-fun` -> `dist/build/fun/mp-toutiao`
- `build:ks-fun` -> `dist/build/fun/mp-kuaishou`
- `build:wx-xingchen` -> `dist/build/xingchen/mp-weixin`
- `build:bd-xingchen` -> `dist/build/xingchen/mp-baidu`

若目录不存在或为空，则构建任务直接失败，而不是返回成功。

### 3. 修正测试中的错误示例命令

修改文件：

- `NovelAppManagerServer/src/test/java/com/fun/novel/controller/AgentBuildPublishControllerTest.java`

将示例中的错误命令：

```text
npm run build:wx
```

改为存在的脚本：

```text
npm run build:wx-xingchen
```

### 4. 补齐 Agent / API Key 接口审计日志

问题：

- `AgentAppController`
- `AgentBuildPublishController`
- `AgentTaskController`
- `ApiKeyController`

这些新接口此前只有 `@Operation`，没有 `@OperationLog`，因此不会进入统一审计表 `user_op_log`。

修复方式：

为上述 Controller 的公开接口补充 `@OperationLog` 注解，保持与原网页接口一致的审计方式。

补充后的审计分类：

- Agent 应用管理
  - 列表、查询、详情：`QUERY_CODE`
  - 创建：`INSERT_CODE`
  - 编辑：`UPDATE_CODE`
  - 删除：`DELETE_CODE`
- Agent 构建与发布
  - 构建、发布：`OTHER_CODE`
- Agent 任务状态查询
  - 查询任务状态：`QUERY_CODE`
- API Key 管理
  - 创建：`INSERT_CODE`
  - 查询：`QUERY_CODE`
  - 删除：`DELETE_CODE`

结果：

- 这些接口现在会写入 `user_op_log`
- 通过 JWT 或 `X-API-Key` 认证进入接口的用户，会在审计日志中保留 `userId / userName / requestUrl / requestParams / responseResult`

### 5. 删除未接线的 CreateNovelAppCommandFactory

删除文件：

- `NovelAppManagerServer/src/main/java/com/fun/novel/common/CreateNovelAppCommandFactory.java`

删除原因：

1. 该类在全仓库中没有任何调用点
2. 当前创建链路并未使用 `CreateNovelAppCommand`
3. 保留该类会给人造成“创建接口已经接入 actor context”的错误印象

因此本次直接删除，避免继续保留半截重构代码。

### 6. API Key 增加用户审核状态校验

问题：

网页登录链路会检查 `novel_user.status`：

- `0` 审核通过：允许登录
- `1` 待审核：拒绝登录
- `2` 审核失败：拒绝登录

但此前 `ApiKeyAuthenticationFilter` 只校验：

1. API Key 是否存在
2. API Key 是否启用
3. API Key 绑定的用户是否存在

并没有校验该用户的 `status`。

这会导致网页和 Agent 的账号可用性判断不一致：

- 网页侧：待审核 / 审核失败账号无法登录
- Agent 侧：如果历史上已经拿到过可用 API Key，仍可能继续调用接口

修复方式：

修改文件：

- `NovelAppManagerServer/src/main/java/com/fun/novel/security/ApiKeyAuthenticationFilter.java`

新增用户状态检查逻辑：

1. `status = 1`：拒绝，返回“账号正在审核中，请耐心等待”
2. `status = 2`：拒绝，返回“账号审核未通过，请联系管理员”
3. `status = 0`：允许继续
4. `status = null`：与网页登录当前实现保持一致，暂不拒绝

结果：

- Agent 接口的账号可用性判断和网页登录保持一致
- 已经被置为待审核/审核失败的用户，即使历史上拿到过 API Key，也无法继续调用 Agent 接口

## 修复后的行为

修复前：

1. 菜单可能触发不存在的 npm script
2. 构建失败可能只体现在日志里
3. 队列层仍将任务标记为 `COMPLETED`
4. 用户看到“构建成功”，但没有 `dist`

修复后：

1. 菜单触发真实存在的品牌化构建命令
2. 退出码失败会中断任务
3. 构建日志出现失败标记会中断任务
4. 无构建产物会中断任务
5. 只有真正生成对应产物目录时才允许任务进入 `COMPLETED`
6. Agent / API Key 接口调用会进入统一操作审计日志
7. Agent / API Key 调用链会校验用户审核状态，和网页登录保持一致

## 当前已知限制

### 1. 创建链路仍然无业务归属

虽然本次已经为 Agent / API Key 接口补上了 `user_op_log` 审计，但当前“创建小程序”主业务链路仍然没有统一的业务归属模型。

具体表现：

1. 审计日志知道“谁调用了创建接口”
2. 旧网页创建接口仍然只记录任务来源，不记录统一的创建者归属字段
3. `novel_app` 本身也没有 `created_by / owner_user_id` 之类字段

因此当前系统状态是：

- 网页旧接口：有审计，无统一业务归属
- Agent 新接口：创建任务的 `user_task.user_id` 已补齐，但应用主表仍无创建者字段

这次修复只补齐了 Agent 创建任务的 `user_task.user_id`，没有扩展应用主表的业务归属模型。

---

# Agent 创建任务补齐 user_task.user_id

## 问题背景

通过 `/api/agent/app/create` 创建小程序后，`user_task` 表中的创建任务记录只写入了：

- `task_id`
- `source`
- `task_type`
- `task_name`
- `status`
- `error_message`

但 `user_id` 始终为空，导致在数据库里无法追溯这条创建任务是由哪个用户发起的。

网页侧的 `build / publish` 链路之所以正常，是因为它们进入的是 `TaskQueueManager`，创建 `TaskQueueItem` 时已经携带了 `userId`，最终落库时会写入 `user_task.user_id`。

而 `agent/create` 走的是 `CreateNovelTaskManager`，原始实现只保存了 `source`，没有保存用户信息，所以 `user_task.user_id` 会显示为 `NULL`。

## 根因分析

### 1. Agent 创建链路没有传递用户 ID

文件：`NovelAppManagerServer/src/main/java/com/fun/novel/controller/AgentAppController.java`

原始实现只调用了：

```java
String taskId = createNovelTaskManager.createTask("api");
```

没有从 `Authentication` 中提取用户名，也没有转换成 `userId` 传入任务管理器。

### 2. CreateNovelTaskManager 只保存了 source，没有保存 userId

文件：`NovelAppManagerServer/src/main/java/com/fun/novel/utils/CreateNovelTaskManager.java`

原始实现中：

1. `createTask(String source)` 只会把 `source` 放进内存映射
2. `persistTaskStatus(...)` 只会写入 `source`
3. `user_task.user_id` 没有任何赋值来源

因此只要走这条链路，创建任务记录就会出现 `user_id = NULL`

## 修复内容

### 1. Agent 创建接口补齐 userId 获取与传递

修改文件：

- `NovelAppManagerServer/src/main/java/com/fun/novel/controller/AgentAppController.java`

修复方式：

1. 从 `Authentication` 取当前用户名
2. 通过 `userService.getUserIdByUsername(username)` 获取 `userId`
3. 调用 `createNovelTaskManager.createTask("api", userId)`

这样 Agent 创建任务和 build / publish 一样，都会把发起用户带进任务链路。

### 2. CreateNovelTaskManager 持久化时写入 userId

修改文件：

- `NovelAppManagerServer/src/main/java/com/fun/novel/utils/CreateNovelTaskManager.java`

修复方式：

1. 新增 `taskId -> userId` 的内存映射
2. 新增 `createTask(String source, Long userId)` 重载
3. 在 `persistTaskStatus(...)` 中补写 `userTask.setUserId(getTaskUserId(taskId))`
4. 任务移除时同步清理 `taskUserIdMap`

修复后，`/api/agent/app/create` 产生的 `user_task` 记录会携带 `user_id`

## 修复后的行为

修复前：

- Agent 创建任务能生成 `taskId`
- `user_task.source` 能记录 `api`
- `user_task.user_id` 为空

修复后：

- Agent 创建任务能生成 `taskId`
- `user_task.source` 继续记录 `api`
- `user_task.user_id` 也会记录对应用户 ID

## 影响范围

- 仅影响 `/api/agent/app/create` 这条创建链路
- 不影响网页旧创建接口
- 不影响 build / publish 链路
- 不影响任务状态查询接口

## 影响范围

本次修复影响以下链路：

- Agent 构建脚本入口
- 后端异步构建任务状态判定
- 构建产物存在性校验
- 构建接口测试示例命令
- Agent / API Key 接口统一审计日志
- 清理未接线的创建命令工厂
- API Key 认证补充用户审核状态校验

本次修复不会改变发布逻辑，也不会修改已有构建产物目录结构，只是让“成功”的定义从“进程返回”收紧为“进程成功且产物存在”。

## 验证结果

已验证：

- `mvn -q -DskipTests compile` 通过
- `mvn spring-boot:run -Dmaven.test.skip=true` 可用于当前本地运行验证

未完整验证：

- `mvn test` 当前无法完整执行

原因：

- 仓库现有测试依赖缺少 `spring-security-test`
- 多个测试类在 `testCompile` 阶段即失败

这属于仓库现有测试环境问题，不是本次 bugfix 引入的问题。

## 建议

后续建议继续补两件事：

1. 在 Agent 构建接口层增加命令白名单校验，禁止提交不存在的 npm script
2. 为构建成功判定增加单元测试或集成测试，覆盖“退出成功但无产物”的场景


---

# Agent 接口路由重构

## 问题背景

在 Agent 接口设计中，存在以下问题：

1. **路由不统一**：`GET /api/novel-apps/query` 是 Agent 专用接口，但路径在 `/api/novel-apps` 下，与网页接口混在一起
2. **接口冗余**：`PUT /api/novel-apps/{appId}/config` 是网页专用接口，但实际前端代码中没有使用
3. **职责不清**：`NovelAppQueryController` 同时包含 Agent 接口和网页接口，职责混乱

## 修复内容

### 1. 接口路由重构

**修改前**：
```
/api/novel-apps/query          # Agent 专用（X-API-Key）
/api/novel-apps/{appId}/config # 网页专用（JWT）
```

**修改后**：
```
/api/agent/app/query           # Agent 专用（X-API-Key）
# 删除了 /api/novel-apps/{appId}/config
```

### 2. 代码重构

**删除的文件**：
- `NovelAppManagerServer/src/main/java/com/fun/novel/controller/NovelAppQueryController.java`
- `NovelAppManagerServer/src/test/java/com/fun/novel/controller/NovelAppQueryControllerTest.java`

**修改的文件**：
- `NovelAppManagerServer/src/main/java/com/fun/novel/controller/AgentAppController.java`
  - 将 `query` 接口从 `NovelAppQueryController` 移到 `AgentAppController`
  - 统一 Agent 接口路由到 `/api/agent/app` 下

- `NovelAppManagerServer/agent-scripts/js/novel-app-query.js`
  - 更新接口路径：`/api/novel-apps/query` → `/api/agent/app/query`

- `NovelAppManagerServer/docs/20-Agent接口.md`
  - 将"聚合查询"接口移到"Agent 应用管理"章节
  - 删除"小说应用聚合查询"和"网页专用接口"章节
  - 更新目录结构

### 3. 最终接口结构

修复后，Agent 应用管理接口结构更加清晰：

```
/api/agent/app
├── GET  /list          # 应用列表
├── GET  /query         # 聚合查询（新位置）
├── GET  /detail        # 应用详情
├── POST /update        # 聚合编辑
├── POST /create        # 创建应用
└── DELETE /delete      # 删除应用
```

所有接口都统一在 `/api/agent/app` 路径下，使用 `X-API-Key` 认证。

## 修复理由

### 1. 为什么要移动 `/api/novel-apps/query`？

- **职责统一**：这是 Agent 专用接口，应该放在 `/api/agent/app` 下
- **路由清晰**：`/api/novel-apps` 路径下应该只放网页接口（JWT 认证）
- **易于维护**：所有 Agent 接口集中在一个 Controller 中

### 2. 为什么要删除 `/api/novel-apps/{appId}/config`？

- **未被使用**：前端代码中没有任何调用
- **功能重复**：与 `POST /api/agent/app/update` 功能重复
- **设计待定**：文档中标注"设计待优化"，说明这个接口本身就不稳定

### 3. 为什么要删除 `NovelAppQueryController`？

- **接口已迁移**：`query` 接口已移到 `AgentAppController`
- **接口已删除**：`config` 接口已删除
- **控制器为空**：没有其他接口，保留无意义

## 影响范围

本次重构影响以下内容：

- Agent 脚本调用路径（已同步更新）
- API 文档结构（已同步更新）
- 后端控制器结构（已优化）

**不影响**：
- 网页前端（因为前端没有使用这些接口）
- 其他 Agent 接口（功能保持不变）
- 接口功能（只是路径变化，功能完全一致）

## 验证结果

已验证：

- ✅ `AgentAppController.java` 编译通过，无诊断错误
- ✅ `novel-app-query.js` 脚本路径已更新
- ✅ 文档结构已更新，目录链接正确

## 建议

后续建议：

1. 在 Spring Security 配置中，明确区分 `/api/agent/**` 和 `/api/novel-apps/**` 的认证方式
2. 考虑为 `/api/novel-apps/**` 添加 JWT 认证要求，避免混用
3. 统一 Agent 接口的命名规范，避免出现 `novel-apps` 这样的混淆命名


---

# 创建任务失败时接口不返回错误信息

## 问题背景

通过 `agent-scripts/app-create.sh` 创建小程序，任务失败后轮询 `/api/agent/task/{taskId}/status` 接口，返回的 `errorMessage` 始终为 `null`：

```json
{
  "code": 200,
  "data": {
    "taskId": "fc3030a4-...",
    "status": "FAILED",
    "description": "创建小程序任务失败",
    "errorMessage": null
  }
}
```

但服务端日志中明确记录了失败原因（如 `抖音预取文件处理失败: ...`）。

## 根因分析

`AgentTaskStatusService.buildCreateTaskResponse()` 在构建 FAILED 状态的响应时，只设置了固定文案 `description`，没有从内存状态中取出 `errorMessage`：

```java
private TaskStatusResponse buildCreateTaskResponse(String taskId, AgentTaskStatus status) {
    resp.setDescription("创建小程序任务失败");
    // ← 缺少 resp.setErrorMessage(...)
    return resp;
}
```

而 `CreateNovelTaskManager` 虽然在 `setTaskFailed()` 时将 `errorMessage` 存入了内存 `taskStatusMap`，但只暴露了 `getTaskStatus()` 方法返回枚举，没有方法获取错误信息。

## 修复内容

### 1. CreateNovelTaskManager 新增 getTaskErrorMessage 方法

修改文件：`NovelAppManagerServer/src/main/java/com/fun/novel/utils/CreateNovelTaskManager.java`

新增方法，从内存状态映射中取出错误信息：

```java
public String getTaskErrorMessage(String taskId) {
    TaskStatusEntry entry = taskStatusMap.get(taskId);
    return entry != null ? entry.errorMessage : null;
}
```

### 2. AgentTaskStatusService 在 FAILED 状态时填充 errorMessage

修改文件：`NovelAppManagerServer/src/main/java/com/fun/novel/service/AgentTaskStatusService.java`

在 `buildCreateTaskResponse` 方法中，FAILED 状态时调用 `getTaskErrorMessage` 并设入响应：

```java
if (status == AgentTaskStatus.FAILED) {
    String errorMsg = createNovelTaskManager.getTaskErrorMessage(taskId);
    if (errorMsg != null) {
        resp.setErrorMessage(errorMsg);
    }
}
```

## 修复后的行为

修复前：任务失败时 `errorMessage` 始终为 `null`，只能去看服务端日志定位原因。

修复后：任务失败时接口返回具体错误信息，脚本终端直接展示失败原因。

## 影响范围

- 仅影响创建小程序任务的状态查询响应
- 不影响构建/发布任务（走 `TaskQueueManager`，已有 `errorMessage`）
- 不影响数据库历史任务查询（走 `fromUserTask`，已有 `errorMessage`）


---

# 发布任务状态误报 COMPLETED

## 问题背景

通过 `agent-scripts/publish.sh` 发布小程序，轮询 `/api/agent/task/{taskId}/status` 返回 `COMPLETED`，但小程序实际并未发布到平台后台。

## 根因分析

`TaskQueueManager.executeTask()` 的逻辑：

```java
try {
    executePublishTask(task);
    task.setStatus("completed");  // 只要没抛异常就设 completed
} catch (Exception e) {
    task.setStatus("failed");
}
```

但 `NovelAppPublishUtil` 中各平台 handler（抖音、快手、微信、百度、支付宝）在发布失败时只发送了 WebSocket 消息然后 `return`，没有抛异常：

```java
if(!uploadExecuteCommandResult){
    messagingTemplate.convertAndSend(..., "Publish error ...");
    return;  // ← 静默返回，没有 throw
}
```

handler 正常返回后，上层 `executeTask()` 没有捕获到异常，直接设置 `task.setStatus("completed")`。

对比 build 链路：`NovelAppBuildUtil.buildNovelAppWithTaskId()` 在所有失败场景都会 `throw RuntimeException`，所以 build 不会出现这个问题。

## 修复内容

修改文件：`NovelAppManagerServer/src/main/java/com/fun/novel/utils/NovelAppPublishUtil.java`

将所有平台 handler 中失败路径的 `return;` 改为 `throw new RuntimeException(...)`，涉及：

- **DouyinPublishHandler**：Token设置失败、上传失败、二维码生成失败（预览模式和发布模式各一处）
- **KuaishouPublishHandler**：密钥生成失败、预览命令生成失败、上传失败、二维码生成失败（预览模式和发布模式各一处）
- **WeixinPublishHandler**：密钥生成失败、预览命令生成失败、发布失败、二维码生成失败（预览模式和发布模式各一处）
- **BaiduPublishHandler**：上传失败、二维码生成失败（预览模式和发布模式各一处）
- **AlipayPublishHandler**：发布/预览失败

成功路径的 `return;` 保持不变（预览模式成功后提前返回）。

WebSocket 消息在 throw 之前已经发出，网页端不受影响。`whenComplete` 回调会多推一条重复错误日志，但前端已处理第一条，不影响功能。

## 修复后的行为

修复前：handler 失败静默返回 → `task.setStatus("completed")` → Agent 拿到 `COMPLETED`

修复后：handler 失败抛异常 → `task.setStatus("failed")` + `task.setErrorMessage(...)` → Agent 拿到 `FAILED` 及具体错误信息

## 影响范围

- `NovelAppPublishUtil.java`：所有平台 handler 的失败路径
- 网页端发布功能不受影响（WebSocket 消息在 throw 之前已发出）
- 网页端构建功能不受影响（build 链路本身已正确抛异常）

## 具体改动位置

### 新增成员变量（第36行）

```java
private final ConcurrentHashMap<String, String> lastErrorMap = new ConcurrentHashMap<>();
```

用于存储每个任务 `executeCommand` 失败时的具体错误信息。

### executeCommand 中写入错误详情（4处）

| 行号 | 场景 |
|---|---|
| 795 | 日志含 `Upload Error` 或 `Publish error` 关键字 |
| 826 | 命令执行超时（85秒） |
| 847 | 进程退出码非0 |
| 866 | 执行过程抛异常 |

### handler 中 `return;` → `throw`（18处）

| 行号 | 平台 | 失败场景 |
|---|---|---|
| 959 | 抖音 | Token设置失败 |
| 1034 | 抖音 | 预览模式二维码失败 |
| 1090 | 抖音 | 上传失败 |
| 1182 | 抖音 | 发布模式二维码失败 |
| 1343 | 快手 | 预览模式二维码失败 |
| 1410 | 快手 | 密钥生成失败 |
| 1464 | 快手 | 上传失败 |
| 1583 | 快手 | 发布模式二维码失败 |
| 1738 | 微信 | 预览模式二维码失败 |
| 1804 | 微信 | 密钥生成失败 |
| 1857 | 微信 | 发布失败 |
| 1978 | 微信 | 发布模式二维码失败 |
| 2106 | 百度 | 预览模式二维码失败 |
| 2172 | 百度 | 上传失败 |
| 2281 | 百度 | 发布模式二维码失败 |
| 2493 | 支付宝 | 发布/预览失败 |

### 新增辅助方法（第2985-2998行）

```java
private String getAndClearLastError(String taskId) {
    return lastErrorMap.remove(taskId);
}

private String buildFailMessage(String taskId, String defaultMsg) {
    String detail = getAndClearLastError(taskId);
    if (detail != null && !detail.isEmpty()) {
        return defaultMsg + " -> " + detail;
    }
    return defaultMsg;
}
```

### 异常传播链路示例

以微信发布失败（第1857行）为例：

```
WeixinPublishHandler.handlePublish (1857行)
  throw "[微信]发布小程序失败 -> Upload Error: ..."
    → publishNovelAppAsync lambda catch → throw new RuntimeException(e)
      → .join() 抛出 CompletionException
        → PublishTaskProcessor.process catch → throw "处理发布任务失败: ..."
          → TaskExecutionService catch → throw "执行发布任务失败: ..."
            → TaskQueueManager.executeTask catch → task.setStatus("failed") ✅
```

Agent 轮询 `/api/agent/task/{taskId}/status` 拿到 `FAILED` 及完整错误信息。
