# 全局操作互斥机制

## 概述

系统中的创建、编辑、构建、发布四种重操作之间存在互斥关系，同一时刻只允许一种重操作执行，防止并发操作导致代码文件冲突或数据不一致。

## 互斥矩阵

| 发起 ↓ \ 已有 → | 创建 | 编辑 | 构建/发布 |
|---|---|---|---|
| **创建** | ❌ 拒绝 | ❌ 拒绝 | ❌ 拒绝 |
| **编辑** | ❌ 拒绝 | ❌ 同appId拒绝，不同appId允许 | ❌ 拒绝 |
| **构建/发布** | 入队成功，执行等待 | 入队成功，执行等待 | 入队成功，串行执行 |

说明：
- 创建和编辑是**立即拒绝**（返回 409），调用方需要提示用户稍后重试
- 构建/发布是**排队等待**（返回 200 + taskId），任务进入队列，等前置操作完成后自动执行

## 核心组件

### GlobalOperationGuard

互斥守卫的核心类，位于 `com.fun.novel.guard.GlobalOperationGuard`。

内部使用 `ReentrantLock` 保证"检查 + 占位"的原子性，消除 TOCTOU（Time-of-check to time-of-use）竞态窗口。

主要方法：

| 方法 | 用途 |
|---|---|
| `tryStartCreate(source, userId)` | 原子检查互斥条件 + 创建任务占位，返回 taskId |
| `finishCreate(taskId)` | 释放创建态 + 通知构建/发布队列恢复消费 |
| `runEdit(appId, action)` | 原子检查 + 编辑占位 → 执行 action → 释放 + 通知队列 |

### EditingOperationTracker

编辑态追踪器，位于 `com.fun.novel.guard.EditingOperationTracker`。

- 使用 `ConcurrentHashMap<String, Long>` 维护正在编辑中的 appId 集合（value 为进入编辑态的时间戳）
- `tryStart(appId)` — 尝试占位，同 appId 互斥
- `finish(appId)` — 释放编辑态
- `hasAnyEditing()` — 是否有任何 appId 在编辑中（供创建和构建/发布检查）
- 内置超时保护：编辑态超过 10 分钟未释放时自动清理（`@Scheduled` 每 60 秒检查）

### TaskQueueManager.canStartNextHeavyTask()

构建/发布队列的消费前置检查，位于 `com.fun.novel.utils.TaskQueueManager`。

`processProdQueue()` 在取出队首任务执行前调用此方法，检查：
1. 有创建任务在运行？→ 不执行，等待
2. 有编辑操作在进行？→ 不执行，等待
3. 有其他构建/发布在运行？→ 不执行，等待

当创建或编辑完成时，`finishCreate()` 和 `runEdit()` 的 finally 块会调用 `triggerQueueProcessing()`，通知队列恢复消费。

### MutexGuardAspect（AOP 切面）

位于 `com.fun.novel.aspect.MutexGuardAspect`，拦截标注了 `@MutexGuarded` 注解的 Controller 方法。

当前仅处理 `EDIT` 类型：通过 SpEL 表达式从方法参数中提取 appId，委托给 `GlobalOperationGuard.runEdit()` 执行。

CREATE 和 BUILD_PUBLISH 是异步操作，生命周期不适合用 AOP 包裹，由各 Controller 手动管理。

## 各入口的保护方式

### 创建 — 手动管理

创建是异步操作（接口立即返回 taskId，后台异步执行），需要手动管理生命周期。

```java
// Controller 中的典型用法
String taskId = globalOperationGuard.tryStartCreate("web", userId);
try {
    CompletableFuture.runAsync(() -> {
        try {
            // 执行创建逻辑
        } finally {
            globalOperationGuard.finishCreate(taskId);
        }
    });
} catch (Exception e) {
    globalOperationGuard.finishCreate(taskId);
    throw e;
}
```

已保护的入口：
- `AgentAppController.createApp()` — Agent API
- `NovelAppCreateController.createNovelApp()` — 网页端

### 编辑 — 切面 + 手动两种方式

**方式一：`@MutexGuarded` 注解（网页端 Controller）**

```java
@MutexGuarded(type = MutexOpType.EDIT, appIdExpr = "#novelApp.appid")
public Result<NovelApp> updateNovelApp(@RequestBody NovelApp novelApp) {
    // 方法体不需要任何互斥代码，切面自动处理
}
```

已标注的网页端 Controller：

| Controller | 方法 | appIdExpr |
|---|---|---|
| `NovelAppController` | `updateNovelApp` | `#novelApp.appid` |
| `AppUiController` | `updateAppUiConfig` | `#appUIConfig.appId` |
| `AppCommonConfigController` | `updateAppCommonConfig` | `#dto.appId` |
| `AppPayController` | `updateAppPay` | `#request.appId` |
| `AppAdController` | `updateAdConfig` | `#request.appAdId` |

**方式二：`GlobalOperationGuard.runEdit()` 直接调用（Agent 端）**

`AgentAppController.update()` → `NovelAppConfigFacade.updateConfig()` → `globalOperationGuard.runEdit(appId, () -> doUpdate(...))`

### 构建/发布 — 队列层保护

构建和发布通过 `TaskQueueManager` 统一管理，入队不拦截，执行前由 `canStartNextHeavyTask()` 检查互斥条件。

所有入口（网页端和 Agent 端）都调用同一个 `taskQueueManager.addTaskToQueue()`，共享同一套队列和互斥检查。

涉及的 Controller：
- `AgentBuildPublishController` — Agent API（构建 + 发布）
- `NovelAppBuildController` — 网页端构建
- `NovelAppPublishController` — 网页端发布

## 注意事项

### 单机限制

所有互斥状态保存在内存中（`ConcurrentHashMap`、`AtomicReference`、`ReentrantLock`），仅适用于单实例部署。多实例部署需要替换为分布式锁（如 Redis）。

### 编辑窗口极短

当前编辑操作是同步的数据库事务，通常几十毫秒完成。`hasAnyEditing()` 为 true 的窗口极短，其他操作几乎不可能撞上。如果未来编辑变成长操作（如用户在页面上编辑多个字段后统一保存），互斥效果会更明显。

### 新增编辑接口时

如果新增了修改应用配置的 Controller 方法，需要加上 `@MutexGuarded` 注解：

```java
@MutexGuarded(type = MutexOpType.EDIT, appIdExpr = "#request.appId")
public Result<?> updateSomething(@RequestBody SomeRequest request) {
    // ...
}
```

`appIdExpr` 是 SpEL 表达式，从方法参数中提取 appId。如果无法直接提取（如参数中只有关联 ID），可以用关联 ID 作为 key，同样能实现同资源互斥。
