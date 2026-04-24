# 任务状态与日志查询机制

## 概述

Agent 脚本通过两个接口获取构建/发布任务的执行结果：

- `GET /api/agent/task/{taskId}/status` — 查询任务状态和错误信息
- `GET /api/agent/task/{taskId}/logs` — 查询任务的完整执行日志

两个接口的数据来源不同，一个从内存取，一个从数据库取。

---

## 1. Status 接口

**数据来源：内存**（`TaskQueueManager` 的 `completedTasks` Map）

### 返回字段

| 字段 | 说明 |
|------|------|
| status | 任务状态：COMPLETED / FAILED |
| errorMessage | 失败时的错误信息，成功时为 null |
| taskType | 任务类型：build / publish / preview |
| startTime | 任务开始时间 |
| completeTime | 任务完成时间 |

### 状态设置流程

```
TaskQueueManager.executeTask()
├── try {
│     executeBuildTask(task) 或 executePublishTask(task)
│     task.setStatus("completed")     ← 成功
│   }
└── catch (Exception e) {
      task.setStatus("failed")         ← 失败
      task.setErrorMessage(e.getMessage())
    }
```

### errorMessage 的传播链

构建任务：
```
NovelAppBuildUtil (抛出原始异常)
  → BuildTaskProcessor.process() (包装: "构建任务失败")
    → TaskExecutionService.executeBuildTask() (包装: "执行构建任务失败")
      → TaskQueueManager catch (存入 task.setErrorMessage)
```

发布任务：
```
NovelAppPublishUtil Handler (抛出原始异常)
  → publishNovelAppAsync catch/whenComplete (包装: "发布任务失败")
    → PublishTaskProcessor.process() (包装: "处理发布任务失败")
      → TaskExecutionService.executePublishTask() (包装: "执行发布任务失败")
        → TaskQueueManager catch (存入 task.setErrorMessage)
```

> 每一层 catch 都会包装一次异常信息，所以最终的 errorMessage 会有多层嵌套。

---

## 2. Logs 接口

**数据来源：数据库**（`user_task` 表的 `log` 字段）

### 日志收集流程

```
executeCommand() / buildNovelAppWithTaskId()
  │
  │  每读到一行进程输出
  ├──→ appendTaskLog(taskId, line)   写入内存缓冲区（Deque，最多 150 行）
  │
  │  任务完成后
  ├──→ TaskQueueManager.saveUserTaskToDatabase()
  │     │
  │     ├── 失败 → buildFailureLog(taskId, null) 从缓冲区拼接完整日志写入 user_task.log
  │     └── 成功 → 写固定文案（"发布成功" 或 "构建成功"）
  │
  └──→ removeTaskLogs(taskId)   清理缓冲区，释放内存
```

### 关键方法说明

| 方法 | 所在类 | 说明 |
|------|--------|------|
| `appendTaskLog(taskId, line)` | PublishUtil / BuildUtil | 追加一行到缓冲区，超过 150 行丢弃最早的 |
| `buildFailureLog(taskId, fallback)` | PublishUtil / BuildUtil | 将缓冲区所有行拼接成完整字符串，缓冲区为空时返回 fallback |
| `removeTaskLogs(taskId)` | PublishUtil / BuildUtil | 清除缓冲区，在 TaskQueueManager 落库后调用 |

### 落库时机

落库发生在 `TaskQueueManager.saveUserTaskToDatabase()` 中，位于 `finally` 块，无论成功失败都会执行：

```java
// TaskQueueManager
try {
    executePublishTask(task);  // 或 executeBuildTask(task)
    task.setStatus("completed");
} catch (Exception e) {
    task.setStatus("failed");
    task.setErrorMessage(e.getMessage());
} finally {
    saveUserTaskToDatabase(task);  // ← 统一落库
}
```

---

## 3. 异常抛出点详细说明

### 3.1 构建任务（NovelAppBuildUtil）

`buildNovelAppWithTaskId()` 方法中的异常抛出点：

| 场景 | 异常信息 | 说明 |
|------|----------|------|
| 工作目录不存在 | `"工作目录不存在: {path}"` | 构建开始前检查 |
| 线程被中断 | `"Build process for task {taskId} was interrupted"` | 任务被用户停止 |
| 进程退出码非0 | `"Build process for task {taskId} exited with code: {exitCode}"` | 命令执行失败 |
| 日志中包含错误 | `"Build process for task {taskId} reported build errors in logs"` | exitCode=0 但日志含 "Build failed with errors" |
| 构建产物为空 | `"Build process for task {taskId} produced no build output at: {path}"` | exitCode=0 但输出目录为空 |
| 其他异常 | `"Build error for task {taskId}: {message}"` | catch 块兜底 |

所有异常都是 `RuntimeException`，会通过 `CompletableFuture.join()` 传播为 `CompletionException`。

### 3.2 发布任务（NovelAppPublishUtil）

#### 3.2.1 公共异常（publishNovelAppAsync）

| 场景 | 异常信息 | 说明 |
|------|----------|------|
| 项目路径不存在 | `"项目路径不存在或不是目录: {path}"` | 发布开始前检查 |
| 发布过程异常 | `"发布过程发生错误: {message}"` | catch 块包装 Handler 抛出的异常 |

#### 3.2.2 抖音 Handler（DouyinPublishHandler）

| 场景 | 异常信息 | 说明 |
|------|----------|------|
| Token 设置失败 | `"[抖音]设置小程序Token失败 -> {detail}"` | `buildFailMessage` 拼接 executeCommand 的具体错误 |
| 上传失败 | `"[抖音]上传小程序失败 -> {detail}"` | 上传命令执行失败 |
| 二维码生成失败 | `"[抖音] 二维码生成失败 -> {detail}"` | 预览命令执行失败 |

#### 3.2.3 快手 Handler（KuaishouPublishHandler）

| 场景 | 异常信息 | 说明 |
|------|----------|------|
| 密钥生成失败 | `"[快手] 密钥文件生成失败: {message}"` | 写密钥文件异常 |
| 预览命令生成失败 | `"[快手] 生成预览命令失败: {message}"` | 构建预览命令时异常 |
| 上传失败 | `"[快手]上传小程序失败 -> {detail}"` | 上传命令执行失败 |
| 二维码生成失败 | `"[快手] 二维码生成失败 -> {detail}"` | 预览命令执行失败 |

#### 3.2.4 微信 Handler（WeixinPublishHandler）

| 场景 | 异常信息 | 说明 |
|------|----------|------|
| 密钥生成失败 | `"[微信] 密钥文件生成失败: {message}"` | 写密钥文件异常 |
| 预览命令生成失败 | `"[微信] 生成预览命令失败: {message}"` | 构建预览命令时异常 |
| 发布失败 | `"[微信]发布小程序失败 -> {detail}"` | 上传命令执行失败 |
| 预览二维码失败 | `"[微信]预览二维码失败 -> {detail}"` | 预览命令执行失败 |

#### 3.2.5 百度 Handler（BaiduPublishHandler）

| 场景 | 异常信息 | 说明 |
|------|----------|------|
| 上传失败 | `"[百度]上传小程序失败 -> {detail}"` | 上传命令执行失败 |
| 预览二维码失败 | `"[百度]预览二维码失败 -> {detail}"` | 预览命令执行失败 |

#### 3.2.6 支付宝 Handler（AlipayPublishHandler）

| 场景 | 异常信息 | 说明 |
|------|----------|------|
| 发布/预览失败 | `"[支付宝] {action}失败 -> {detail}"` | action 为 "发布" 或 "预览" |

#### 3.2.7 辅助方法异常

| 方法 | 异常信息 | 触发条件 |
|------|----------|----------|
| `buildKuaishouKeyFile()` | `"生成快手密钥文件失败: {message}"` | 写文件失败 |
| `buildWeixinKeyFile()` | `"生成微信密钥文件失败: {message}"` | 写文件失败 |
| `prepareAlipayConfigJson()` | `"准备支付宝配置文件失败: {message}"` | 配置文件生成失败 |
| `buildXxxCommand()` | `"密钥文件不存在: {path}"` | 快手/微信密钥文件不存在 |

### 3.3 buildFailMessage 机制

Handler 中大部分异常通过 `buildFailMessage(taskId, defaultMsg)` 构建：

```java
private String buildFailMessage(String taskId, String defaultMsg) {
    String detail = getAndClearLastError(taskId);  // 从 lastErrorMap 取 executeCommand 的具体错误
    if (detail != null && !detail.isEmpty()) {
        return defaultMsg + " -> " + detail;       // 拼接: "[微信]发布小程序失败 -> executeCommand fail 退出码: 1"
    }
    return defaultMsg;
}
```

`lastErrorMap` 由 `executeCommand` 在以下情况写入：
- 检测到日志中包含 "Upload Error" 或 "Publish error"
- 命令执行超时
- 命令退出码非0
- 命令执行异常

---

## 4. 数据库表结构（user_task）

| 字段 | 类型 | 说明 |
|------|------|------|
| task_id | varchar(255) | 任务ID |
| task_type | varchar(255) | 任务类型（build/publish） |
| status | varchar(255) | 任务状态（completed/failed） |
| error_message | text | 错误信息 |
| log | text | 执行日志（失败时为完整进程输出，成功时为固定文案） |
| platform_code | varchar(255) | 平台代码（mp-weixin/mp-toutiao 等） |
| app_id | varchar(255) | 应用ID |
| project_path | varchar(255) | 项目路径 |
| version | varchar(255) | 版本号 |

---

## 5. 注意事项

1. **Status 接口是内存数据**，服务重启后 `completedTasks` Map 会清空，历史任务状态查不到。
2. **Logs 接口是数据库数据**，持久化的，服务重启后仍可查询。
3. **缓冲区清理时机**：`removeTaskLogs` 在 `saveUserTaskToDatabase` 之后调用，确保先读后清。
4. **日志最多 150 行**：由 `MAX_LOG_LINES` 控制，超出时丢弃最早的行，保留最近的输出。
5. **errorMessage 多层嵌套**：异常经过 Handler → publishNovelAppAsync → TaskProcessor → TaskExecutionService → TaskQueueManager 多层包装，最终的 errorMessage 会很长。

---

## 6. 命令退出码（exitCode）机制

### 什么是退出码

每个进程结束时都会返回一个退出码（exit code）给操作系统：
- `0` = 成功
- 非 `0` = 失败（具体数字由程序自己定义，1 通常表示一般性错误）

这是操作系统的标准机制，不是 Java 实现的。

### Java 如何获取退出码

```java
process = processBuilder.start();          // 启动子进程
int exitCode = process.waitFor();          // 等待进程结束，获取退出码
if (exitCode != 0) {
    // 命令执行失败
}
```

### 各 CLI 工具的退出码

| 工具 | 命令示例 | 成功 | 失败 |
|------|----------|------|------|
| uni-app CLI | `uni build -p wx-xingyu --minify` | exitCode=0 | exitCode=1 |
| 微信 CI | `miniprogram-ci upload ...` | exitCode=0 | exitCode=1 |
| 抖音 CLI | `tma upload ...` | exitCode=0 | exitCode≠0 |
| 快手 CI | `ks-miniprogram-ci upload ...` | exitCode=0 | exitCode≠0 |
| 百度 CLI | `swan upload ...` | exitCode=0 | exitCode≠0 |

退出码由这些 CLI 工具自己设置（内部调用 `process.exit(1)`），不需要我们手动处理。

---

## 7. 构建失败的三重检测机制

`NovelAppBuildUtil.buildNovelAppWithTaskId()` 中，判断构建是否失败有三层检查：

### 第一层：exitCode != 0

```java
int exitCode = process.waitFor();
if (exitCode != 0) {
    // 进程自己报告失败，最常见的情况
    throw new RuntimeException("exited with code: " + exitCode);
}
```

### 第二层：日志中包含 "Build failed with errors"

```java
// 读进程输出时标记（第 253 行）
if (line.contains("Build failed with errors")) {
    localBuildEncounterError = true;
}

// exitCode == 0 时检查
if (localBuildEncounterError) {
    // exitCode=0 但日志含错误关键字，仍视为失败
    throw new RuntimeException("reported build errors in logs");
}
```

这层检查是为了兜底：某些版本的 uni-app/Vite 构建失败了但退出码仍然返回 0。`"Build failed with errors"` 是 Vite 构建工具在编译失败时输出到标准输出（stdout）的固定文案，不是我们自己写的。

### 第三层：构建产物目录为空

```java
Path expectedOutputPath = resolveExpectedOutputPath(finalCmd, useTestPath);
if (!isBuildOutputValid(expectedOutputPath)) {
    // exitCode=0，日志也没报错，但构建产物目录为空或不存在
    throw new RuntimeException("produced no build output at: " + expectedOutputPath);
}
```

三层检查依次执行，任何一层命中都会抛异常标记为失败。

### buildEncounterError vs localBuildEncounterError

| 变量 | 作用域 | 用途 |
|------|--------|------|
| `localBuildEncounterError` | 局部变量（lambda 内） | 方法内部判断当前任务是否失败，不受并发影响 |
| `buildEncounterError` | 实例变量（类字段） | `BuildTaskProcessor` 通过 `isBuildEncounterError()` 从外部读取，作为 `.join()` 后的二次检查 |

两个变量同时设置，但 `localBuildEncounterError` 是线程安全的（局部变量），`buildEncounterError` 在并发场景下可能被其他任务覆盖。

---

## 8. 进程标准输出的读取

Java 通过 `process.getInputStream()` 读取子进程的标准输出（stdout）。由于设置了 `processBuilder.redirectErrorStream(true)`，标准错误（stderr）也会合并到 stdout 一起读取。

```java
processBuilder.redirectErrorStream(true);  // stderr 合并到 stdout
process = processBuilder.start();

BufferedReader reader = new BufferedReader(
    new InputStreamReader(process.getInputStream(), StandardCharsets.UTF_8));
String line;
while ((line = reader.readLine()) != null) {
    // line 就是子进程输出的每一行文字
    // 跟你在终端里手动执行命令看到的输出一模一样
    appendTaskLog(taskId, line);  // 写入缓冲区
    messagingTemplate.convertAndSend("/topic/build-logs/" + taskId, line);  // 推送给前端
}
```

所有你在终端里看到的构建/发布日志（包括 Vite 的 `"Build failed with errors."`、微信 CI 的上传进度等），都是通过这种方式逐行读取并处理的。

---

## 9. 发布任务的失败检测机制

### executeCommand 的返回值

`NovelAppPublishUtil.executeCommand()` 返回 `boolean`：`true` 表示命令执行成功，`false` 表示失败。各 Handler 根据这个返回值判断每个步骤是否成功。

### 两层失败检测

#### 第一层：日志关键字提前返回（第 850 行）

```java
if (logMessage.contains("Upload Error") || logMessage.contains("Publish error")) {
    // 检测到错误关键字，不等命令执行完就直接返回 false
    return false;
}
```

在逐行读取进程输出时，如果某一行包含 `"Upload Error"` 或 `"Publish error"`，立即返回失败，不等进程结束。这些关键字是根据各平台 CLI 的实际错误输出逐个添加的（代码注释：`//TODO 都快微的报错log都不一致，碰到一个加一个`）。

#### 第二层：退出码检查

```java
int exitCode = process.exitValue();
if (exitCode != 0) {
    // 命令退出码非0，返回 false
    return false;
}
return true;  // 退出码为0且日志无错误关键字，返回 true
```

进程正常结束后，检查退出码。非 0 返回失败，0 返回成功。

### 与构建任务的区别

| | 构建任务（BuildUtil） | 发布任务（PublishUtil） |
|---|---|---|
| 退出码检查 | ✅ | ✅ |
| 日志关键字检查 | ✅ `"Build failed with errors"` | ✅ `"Upload Error"` / `"Publish error"` |
| 构建产物检查 | ✅ 检查输出目录是否为空 | ❌ 不检查 |
| 判断方式 | 三重检测，方法内部抛异常 | 两层检测，`executeCommand` 返回 boolean，Handler 根据返回值抛异常 |

### packageSize 解析（仅抖音）

抖音 Handler 在上传时会从命令输出中解析包大小信息：

```
正在准备上传...
主包 1.2MB
packageA/ 0.8MB
总体积：2.0MB
```

解析结果拼成 JSON：`{"main": "1.2MB", "sub": "0.8MB", "total": "2.0MB"}`

这是"顺便"做的，解析不到不影响上传成功失败的判断。上传是否成功完全取决于 `executeCommand` 的返回值。


---

## 10. 发布二维码获取机制

### 概述

发布任务完成后，各平台会在项目目录下生成二维码 PNG 图片。Agent 通过以下接口获取：

```
GET /api/agent/publish/qrcode/{taskId}
```

返回 `image/png` 图片字节流，需携带 `X-API-Key` 认证。

### 各平台二维码生成方式

| 平台 | 文件名 | 生成方式 | 说明 |
|------|--------|----------|------|
| 抖音 | `tt_qrcode.png` | CLI 输出 URL → 服务端用 zxing 编码为 PNG | CLI 输出 `二维码信息：{url}`，服务端解析后调用 `saveQrCodeImage()` 生成本地文件 |
| 快手 | `ks_qrcode.png` | CLI 直接生成 | `ks-miniprogram-ci preview` 命令直接输出文件 |
| 微信 | `wx_qrcode.png` | CLI 直接生成 | `miniprogram-ci preview` 命令直接输出文件 |
| 百度 | `bd_qrcode.png` | CLI 输出 URL → 服务端用 zxing 编码为 PNG | 与抖音同理，调用 `saveQrCodeImage()` 生成 |

### 二维码生成时机

二维码在发布流程的最后一步生成（预览命令执行时）：

```
发布流程
├── 步骤1：设置 Token / 生成密钥
├── 步骤2：上传代码（previewOnly 模式跳过此步）
└── 步骤3：执行预览命令 → 生成二维码文件
```

### 接口查找逻辑

`AgentBuildPublishController.getPublishQrcode()` 的查找顺序：

```
1. PublishTaskManager.getPlatformCode(taskId) / getProjectPath(taskId)
   ↓ 如果为 null
2. TaskQueueManager.findTaskById(taskId) → taskParams 中提取 platformCode + projectPath
   ↓ 如果仍为 null
3. 返回 404
```

拿到 `platformCode` 和 `projectPath` 后，根据平台映射文件名，读取本地文件返回。

### saveQrCodeImage 通用方法

抖音和百度的二维码都是从 CLI 输出的 URL 编码而来，共用 `NovelAppPublishUtil.saveQrCodeImage()`：

```java
private void saveQrCodeImage(String content, String filePath) {
    // 使用 zxing 将 content（URL）编码为 300x300 的 PNG 二维码图片
    // 保存到 filePath
}
```

### Agent 调用流程

```
1. POST /api/agent/publish          → 拿 taskId
2. GET  /api/agent/task/{taskId}/status  → 轮询等 COMPLETED
3. GET  /api/agent/publish/qrcode/{taskId}  → 下载二维码 PNG
```

脚本端（`publish.js`）在发布成功后自动执行第 3 步，将图片保存到 `agent-scripts/qrcode_output/{taskId}.png`。可通过 `--no-qrcode` 跳过。

### 代码改动清单

#### 为什么微信和快手不需要改？

微信和快手的 CLI 工具本身就会在 `projectPath` 下直接生成 PNG 文件：

- 微信：`miniprogram-ci preview` → 直接输出 `wx_qrcode.png`
- 快手：`ks-miniprogram-ci preview` → 直接输出 `ks_qrcode.png`

而抖音和百度的 CLI 不生成文件，只输出 URL 文本：

- 抖音：CLI 输出 `二维码信息：https://...`，之前没有生成本地文件
- 百度：CLI 输出 URL，之前已经用 zxing 编码成 `bd_qrcode.png`

所以这次只需要让抖音也跟百度一样，把 URL 编码成本地 PNG。

#### 二维码文件存放位置

四个平台的二维码文件都在 `projectPath`（发布时传入的项目路径）目录下，只是文件名不同：

| 平台 | 文件路径 | 生成者 |
|------|----------|--------|
| 抖音 | `{projectPath}/tt_qrcode.png` | 服务端 `saveQrCodeImage()` |
| 快手 | `{projectPath}/ks_qrcode.png` | CLI 工具直接生成 |
| 微信 | `{projectPath}/wx_qrcode.png` | CLI 工具直接生成 |
| 百度 | `{projectPath}/bd_qrcode.png` | 服务端 `saveQrCodeImage()` |

#### 接口返回方案的选择

讨论过三种方案：

1. **统一返回 JSON**（type 区分 image base64 / url）— 灵活但调用方需要判断类型
2. **统一返回 image/png**（抖音也在服务端生成 PNG 文件）— 最终采用，调用方最简单
3. **返回本地文件路径或 URL** — Agent 和服务器不在同一台机器上，本地路径无法访问；返回 HTTP URL 本质上还是需要一个图片接口，多了一层间接

最终选择方案 2：所有平台统一产出本地 PNG 文件，接口统一返回 `image/png`。

#### 对现有功能的影响

无影响：

- 旧网页发布接口（`NovelAppPublishController`）没有任何修改
- WebSocket 日志推送逻辑没有改动，抖音的 `[抖音] 二维码生成成功: {url}` 日志仍然正常推送
- 旧的二维码接口（`GET /api/novel-publish/qrcode/{taskId}`）不受影响
- 抖音 handler 只是在解析到 URL 后多加了一行 `saveQrCodeImage()`，多生成一个文件，不影响已有逻辑

#### 具体文件改动

为实现统一的本地二维码文件产出，涉及以下代码改动：

#### 1. NovelAppPublishUtil.java

**新增通用方法** `saveQrCodeImage(String content, String filePath)`：

```java
// 类级别的通用方法，抖音和百度共用
private void saveQrCodeImage(String content, String filePath) {
    // 使用 zxing 将 content 编码为 300x300 PNG 二维码，保存到 filePath
}
```

**抖音 DouyinPublishHandler — 两处改动**（预览模式 + 正常模式）：

```java
// 改动前：只推送 WebSocket 日志
if (line.contains("二维码信息：")) {
    String qrCodeUrl = line.substring(line.indexOf("二维码信息：") + 6);
    messagingTemplate.convertAndSend(...);
}

// 改动后：多加一行，将 URL 编码为本地 PNG 文件
if (line.contains("二维码信息：")) {
    String qrCodeUrl = line.substring(line.indexOf("二维码信息：") + 6);
    saveQrCodeImage(qrCodeUrl.trim(), projectPath + File.separator + "tt_qrcode.png");  // ← 新增
    messagingTemplate.convertAndSend(...);
}
```

**百度 BaiduPublishHandler.generateAndSaveQrCode** — 改为委托通用方法：

```java
// 改动前：方法内部自己实现 zxing 编码逻辑（约 20 行）
private void generateAndSaveQrCode(String content, String projectPath) throws Exception {
    // ... 完整的 zxing 编码实现
}

// 改动后：委托给通用方法
private void generateAndSaveQrCode(String content, String projectPath) throws Exception {
    saveQrCodeImage(content, projectPath + File.separator + "bd_qrcode.png");
}
```

#### 2. AgentBuildPublishController.java

**新增接口** `GET /api/agent/publish/qrcode/{taskId}`：

- 从 `PublishTaskManager` 或 `TaskQueueManager` 获取 platformCode + projectPath
- 根据平台映射二维码文件名
- 读取本地文件，返回 `image/png`

#### 3. agent-scripts/js/lib.js

**新增方法** `downloadBinary(method, url, apiKey, destPath)`：

- 用于下载二进制文件（二维码图片），通过 stream 写入本地

#### 4. agent-scripts/js/publish.js

**发布成功后新增二维码下载逻辑**：

- 调用 `GET /api/agent/publish/qrcode/{taskId}` 下载 PNG
- 保存到 `agent-scripts/qrcode_output/{taskId}.png`
- `--no-qrcode` 跳过，`--qrcode-dir` 自定义保存目录
