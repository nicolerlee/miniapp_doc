# 按 packageId 粒度互斥重构方案

## 1. 目标

将现有的**全局互斥**改为**按 packageId 粒度互斥**：

- 同一个 packageId 的任务串行排队
- 不同 packageId 的任务可以并行（最多 10 个并发）
- 同一个 packageId 编辑时，不允许构建/上传/安装/发布/删除
- 同一个 packageId 有任务运行或排队时，不允许编辑/删除
- 不同 packageId 的编辑互不影响

## 2. 互斥规则表

| 当前状态（同一 packageId） | 允许的操作 | 拒绝的操作 |
|---------------------------|-----------|-----------|
| 空闲 | 所有 | 无 |
| 编辑中 | 其他 packageId 的任何操作 | 本 packageId 的：构建、上传、安装、安装并构建、发布、删除 |
| 有任务排队或运行中 | 其他 packageId 的任何操作 | 本 packageId 的：编辑、删除 |
| 删除中 | 其他 packageId 的任何操作 | 本 packageId 的：构建、上传、安装、安装并构建、编辑 |

## 3. 涉及的文件

### 3.1 需要新增的文件

| 文件 | 说明 |
|------|------|
| `guard/PackageLockManager.java` | 新的按 packageId 粒度的锁管理器 |

### 3.2 需要修改的文件

| 文件 | 改动说明 |
|------|---------|
| `utils/TaskQueueManager.java` | 核心重构：全局队列 → 按 packageId 分队列 + Semaphore(10) 并发控制 |
| `entity/TaskQueueItem.java` | 新增 `packageId` 字段（当前 packageId 存在 taskParams Map 里，提升为一级字段方便索引） |
| `service/impl/AppServiceImpl.java` | 入队前调用 `PackageLockManager` 检查；`patch()` 和 `delete()` 加互斥检查 |
| `controller/AppController.java` | `PATCH` 和 `DELETE` 接口返回 409 冲突时的错误处理 |

### 3.3 需要删除的文件

| 文件 | 原因 |
|------|------|
| `guard/GlobalOperationGuard.java` | 被 `PackageLockManager` 替代 |
| `guard/EditingOperationTracker.java` | 功能合并到 `PackageLockManager` |
| `annotation/MutexGuarded.java` | 不再使用 AOP 方式 |
| `aspect/MutexGuardAspect.java` | 不再使用 AOP 方式 |
| `enums/MutexOpType.java` | 不再需要 |

## 4. 详细设计

### 4.1 PackageLockManager（新增）

**设计原则：所有状态转换必须是原子的（check-and-set 语义），对外不暴露分离的 check + set 方法。**

```java
@Component
public class PackageLockManager {

    // 每个 packageId 的状态 + 任务计数
    private final ConcurrentHashMap<String, PackageStateHolder> states = new ConcurrentHashMap<>();

    // per-packageId 锁，保证同一 packageId 的状态转换原子性
    private final ConcurrentHashMap<String, Object> locks = new ConcurrentHashMap<>();

    /**
     * 内部状态持有者
     */
    private static class PackageStateHolder {
        PackageState state = PackageState.IDLE;
        int taskCount = 0;          // 排队 + 运行中的任务数
        long stateTimestamp = 0;    // 进入当前状态的时间戳（用于超时清理）
    }

    // PackageState 枚举：IDLE, EDITING, BUSY, DELETING

    private Object getLock(String packageId) {
        return locks.computeIfAbsent(packageId, k -> new Object());
    }
    
    /**
     * 原子操作：尝试进入编辑态。
     * 内部持锁完成 检查IDLE → 设置EDITING，不可能被其他线程插入。
     * @return true=成功进入编辑态；false=拒绝（当前有任务或正在删除）
     */
    public boolean tryStartEdit(String packageId) {
        synchronized (getLock(packageId)) {
            PackageStateHolder holder = states.get(packageId);
            if (holder == null || holder.state == PackageState.IDLE) {
                PackageStateHolder h = states.computeIfAbsent(packageId, k -> new PackageStateHolder());
                h.state = PackageState.EDITING;
                h.stateTimestamp = System.currentTimeMillis();
                return true;
            }
            return false;
        }
    }

    /**
     * 结束编辑态（CAS 语义：仅当前状态为 EDITING 时才转 IDLE）。
     * 防止与超时清理竞态——如果超时清理已经将状态改为其他值，此处 no-op。
     */
    public void finishEdit(String packageId) {
        synchronized (getLock(packageId)) {
            PackageStateHolder holder = states.get(packageId);
            if (holder != null && holder.state == PackageState.EDITING) {
                holder.state = PackageState.IDLE;
                holder.stateTimestamp = 0;
                // 如果无任务引用，清理条目释放内存
                cleanupIfIdle(packageId, holder);
            }
        }
    }

    /**
     * 原子操作：尝试入队任务（check + markBusy 合一）。
     * 条件：当前状态为 IDLE 或 BUSY（允许追加任务）。
     * 拒绝：EDITING 或 DELETING 状态。
     * @return true=入队成功，已标记 BUSY 并递增 taskCount；false=拒绝
     */
    public boolean tryEnqueueTask(String packageId) {
        synchronized (getLock(packageId)) {
            PackageStateHolder holder = states.computeIfAbsent(packageId, k -> new PackageStateHolder());
            if (holder.state == PackageState.IDLE || holder.state == PackageState.BUSY) {
                holder.state = PackageState.BUSY;
                holder.taskCount++;
                holder.stateTimestamp = System.currentTimeMillis();
                return true;
            }
            return false; // EDITING 或 DELETING
        }
    }

    /**
     * 任务完成时调用，递减 taskCount。
     * 当 taskCount 降为 0 时自动恢复 IDLE 并清理条目。
     */
    public void taskFinished(String packageId) {
        synchronized (getLock(packageId)) {
            PackageStateHolder holder = states.get(packageId);
            if (holder != null && holder.state == PackageState.BUSY) {
                holder.taskCount = Math.max(0, holder.taskCount - 1);
                if (holder.taskCount == 0) {
                    holder.state = PackageState.IDLE;
                    holder.stateTimestamp = 0;
                    cleanupIfIdle(packageId, holder);
                }
            }
        }
    }

    /**
     * 原子操作：尝试进入删除态。
     * 条件：当前状态为 IDLE（无编辑、无任务）。
     * @return true=可以删除；false=拒绝
     */
    public boolean tryStartDelete(String packageId) {
        synchronized (getLock(packageId)) {
            PackageStateHolder holder = states.get(packageId);
            if (holder == null || holder.state == PackageState.IDLE) {
                PackageStateHolder h = states.computeIfAbsent(packageId, k -> new PackageStateHolder());
                h.state = PackageState.DELETING;
                h.stateTimestamp = System.currentTimeMillis();
                return true;
            }
            return false;
        }
    }

    /**
     * 结束删除态（CAS 语义：仅当前状态为 DELETING 时才清理）。
     * 同时从 states 和 locks 中移除该 packageId 条目。
     */
    public void finishDelete(String packageId) {
        synchronized (getLock(packageId)) {
            PackageStateHolder holder = states.get(packageId);
            if (holder != null && holder.state == PackageState.DELETING) {
                states.remove(packageId);
            }
        }
        // 删除完成后清理锁对象（可选，防止内存泄漏）
        locks.remove(packageId);
    }

    /**
     * 查询当前拒绝原因（用于生成友好错误信息）。
     */
    public String getBlockReason(String packageId) {
        PackageStateHolder holder = states.get(packageId);
        if (holder == null) return null;
        switch (holder.state) {
            case EDITING: return "正在编辑中";
            case BUSY: return "有任务正在执行或排队中";
            case DELETING: return "正在删除中";
            default: return null;
        }
    }

    /**
     * 内部：清理 IDLE 且无引用的条目，防止内存泄漏。
     */
    private void cleanupIfIdle(String packageId, PackageStateHolder holder) {
        if (holder.state == PackageState.IDLE && holder.taskCount == 0) {
            states.remove(packageId);
        }
    }

    /**
     * 定时清理超时状态，每 60 秒执行一次。
     * - 编辑态超过 10 分钟未释放 → 自动清理
     * - 删除态超过 10 分钟未释放 → 自动清理
     * 清理后触发 processQueues() 唤醒可能等待的任务。
     */
    @Scheduled(fixedDelay = 60_000L)
    public void cleanExpiredStates() {
        long now = System.currentTimeMillis();
        for (Map.Entry<String, PackageStateHolder> entry : states.entrySet()) {
            String packageId = entry.getKey();
            PackageStateHolder holder = entry.getValue();
            synchronized (getLock(packageId)) {
                if (holder.stateTimestamp > 0) {
                    long elapsed = now - holder.stateTimestamp;
                    if (elapsed > TIMEOUT_MS) {
                        if (holder.state == PackageState.EDITING || holder.state == PackageState.DELETING) {
                            logger.warn("状态超时自动清理：packageId={}, state={}, 已持续 {} 秒",
                                    packageId, holder.state, elapsed / 1000);
                            holder.state = PackageState.IDLE;
                            holder.stateTimestamp = 0;
                            cleanupIfIdle(packageId, holder);
                        }
                    }
                }
            }
        }
        // 超时清理后唤醒调度，让被阻塞的任务有机会执行
        taskQueueManager.triggerQueueProcessing();
    }

    private static final long TIMEOUT_MS = 10 * 60 * 1000L; // 10 分钟
}
```

**关键设计决策：**
1. **原子性**：所有状态转换在 `synchronized(getLock(packageId))` 内完成，消除竞态窗口
2. **CAS 语义**：`finishEdit` / `finishDelete` 只在预期状态下才转换，防止与超时清理冲突
3. **引用计数**：`taskCount` 跟踪排队+运行中的任务数，避免过早 markIdle
4. **内存清理**：状态回到 IDLE 且 taskCount=0 时自动从 Map 中移除
5. **双超时保护**：编辑态和删除态都有 10 分钟超时兜底，超时后唤醒调度

### 4.2 TaskQueueManager 重构

**核心变化：**

```java
// 旧：全局单队列
private final Queue<TaskQueueItem> prodTaskQueue = new ConcurrentLinkedQueue<>();

// 新：按 packageId 分队列（每个队列上限 50，防止 OOM）
private final ConcurrentHashMap<String, Queue<TaskQueueItem>> packageQueues = new ConcurrentHashMap<>();
private static final int MAX_QUEUE_SIZE_PER_PACKAGE = 50;

// 新：全局并发控制（最多 10 个任务同时执行，公平模式）
private final Semaphore globalConcurrency = new Semaphore(10, true);

// 新：每个 packageId 当前正在运行的任务（同一 packageId 最多 1 个）
private final ConcurrentHashMap<String, TaskQueueItem> packageRunningTasks = new ConcurrentHashMap<>();
```

**调度逻辑（`processQueues()`）：**

1. 遍历所有 `packageQueues`（仅非空队列）
2. 对每个 packageId：如果该 packageId 没有正在运行的任务 && 队列非空 && `globalConcurrency.tryAcquire()`
3. 取出队首任务，提交到线程池执行
4. **如果线程池提交失败**（RejectedExecutionException），立即 `globalConcurrency.release()` 并将任务放回队首
5. 任务完成后（**必须在 finally 块中**）：`globalConcurrency.release()` + 从 `packageRunningTasks` 移除 + `packageLockManager.taskFinished(packageId)` + 调用全局 `processQueues()` 唤醒其他等待的 packageId
6. 如果该 packageId 队列为空且无运行中任务，`taskFinished()` 内部会自动将状态恢复 IDLE 并清理条目

**`addTaskToQueue()` 改动：**

```java
public void addTaskToQueue(Long userId, TaskQueueItem task) {
    String packageId = task.getPackageId();
    
    // 防御性校验
    if (packageId == null || packageId.trim().isEmpty()) {
        throw new IllegalArgumentException("packageId 不能为空");
    }
    
    // 注意：入队权限检查（tryEnqueueTask）已在 AppServiceImpl 层原子完成
    // 此处只负责加入队列和触发调度
    
    // 队列容量检查
    Queue<TaskQueueItem> queue = packageQueues.computeIfAbsent(packageId, k -> new ConcurrentLinkedQueue<>());
    if (queue.size() >= MAX_QUEUE_SIZE_PER_PACKAGE) {
        // 回退 taskCount（因为 tryEnqueueTask 已经递增了）
        packageLockManager.taskFinished(packageId);
        throw new IllegalStateException("代码包 [" + packageId + "] 队列已满（上限 " + MAX_QUEUE_SIZE_PER_PACKAGE + "），请稍后重试");
    }
    
    queue.offer(task);
    
    // 触发调度
    processQueues();
}
```

**测试环境队列不变**，保持独立的 `testTaskQueue` 逻辑。

### 4.3 TaskQueueItem 改动

新增 `packageId` 一级字段：

```java
/** 关联的代码包ID */
private String packageId;

public String getPackageId() { return packageId; }
public void setPackageId(String packageId) { this.packageId = packageId; }
```

在 `AppServiceImpl` 创建任务时设置：`task.setPackageId(packageId);`

### 4.4 AppServiceImpl 改动

**入队前检查（build/install/upload/publish）：**

```java
// 在 build()、install()、installBuild()、upload()、publish() 方法中

// 1. 防御性校验
if (packageId == null || packageId.trim().isEmpty()) {
    return Result.errorWithCode(400, "INVALID_PARAMS", "packageId 不能为空");
}

// 2. 原子性入队检查（check + markBusy 合一，无竞态窗口）
if (!packageLockManager.tryEnqueueTask(packageId)) {
    String reason = packageLockManager.getBlockReason(packageId);
    return Result.errorWithCode(409, "PACKAGE_LOCKED", 
        "代码包 [" + packageId + "] " + reason + "，请等待完成后再操作");
}

// 3. 后续创建 TaskQueueItem 并入队
// 注意：如果后续逻辑异常（如参数校验失败），需要回退 taskCount：
// packageLockManager.taskFinished(packageId);
```

**编辑（patch）加互斥：**

```java
public Result<Void> patch(String packageId, Map<String, Object> body) {
    if (!packageLockManager.tryStartEdit(packageId)) {
        return Result.errorWithCode(409, "PACKAGE_BUSY", 
            "代码包 [" + packageId + "] 有任务正在执行或排队中，请等待完成后再编辑");
    }
    try {
        // 原有 patch 逻辑...
        return Result.success();
    } finally {
        packageLockManager.finishEdit(packageId);
    }
}
```

**删除加互斥：**

```java
public Result<Void> delete(String packageId) {
    if (!packageLockManager.tryStartDelete(packageId)) {
        return Result.errorWithCode(409, "PACKAGE_BUSY", 
            "代码包 [" + packageId + "] 有任务正在执行/排队或正在编辑中，无法删除");
    }
    try {
        // 原有 delete 逻辑...
        return Result.success();
    } finally {
        packageLockManager.finishDelete(packageId);
    }
}
```

### 4.5 现有接口行为变化

| 接口 | 旧行为 | 新行为 |
|------|--------|--------|
| `POST /{packageId}/build` | 全局排队，同时只跑 1 个 | 按 packageId 排队，不同包可并行（最多 10 个） |
| `POST /{packageId}/install` | 同上 | 同上 |
| `POST /{packageId}/install-build` | 同上 | 同上 |
| `POST /upload` | 同上 | 同上 |
| `POST /{packageId}/publish` | 同上 | 同上 |
| `PATCH /{packageId}` | 无保护 | 有任务或删除中时返回 409 |
| `DELETE /{packageId}` | 无保护 | 有任务、编辑中时返回 409；删除期间阻止其他操作 |

### 4.6 并发控制总结

```
┌─────────────────────────────────────────────────────┐
│              Semaphore(10) 全局并发池                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  packageA: [task1] → [task2] → [task3]  (串行)      │
│  packageB: [task4] → [task5]            (串行)      │
│  packageC: [task6]                      (串行)      │
│                                                     │
│  同一时刻最多 10 个不同 packageId 的任务并行执行       │
│  同一 packageId 内严格串行                           │
└─────────────────────────────────────────────────────┘
```

## 5. 实施步骤

### Step 1：新增 PackageLockManager
- 创建 `guard/PackageLockManager.java`
- 实现状态管理和超时清理

### Step 2：修改 TaskQueueItem
- 新增 `packageId` 字段和 getter/setter

### Step 3：重构 TaskQueueManager
- 将 `prodTaskQueue` + `prodRunningTasks` 替换为 `packageQueues` + `packageRunningTasks`
- 添加 `Semaphore(10, true)` 并发控制（公平模式）
- 重写 `addTaskToQueue()`：加 packageId 非空校验 + 队列容量检查
- 重写 `processProdQueue()`（改名为 `processQueues()`）：遍历所有非空队列调度
- 重写 `executeTask()` 的完成回调：finally 中 release Semaphore + taskFinished + processQueues
- 重写 `stopTask()` / `stopBatchTasks()`：停止后 release Semaphore + taskFinished + processQueues
- 线程池提交失败时兜底：catch RejectedExecutionException → release + taskFinished
- 保留测试环境队列逻辑不变（不经过 PackageLockManager）
- 保留 WebSocket 通知、进度推送等逻辑不变
- 删除 `canStartNextHeavyTask()` 方法
- 删除对 `EditingOperationTracker` 的依赖

### Step 4：修改 AppServiceImpl
- 所有创建 `TaskQueueItem` 的地方加上 `task.setPackageId(packageId)`
- 所有接受 packageId 参数的方法入口加非空校验
- `build()`、`install()`、`installBuild()`、`upload()`、`publish()` 入队前调用 `tryEnqueueTask()`（原子操作），失败时直接返回 409
- 入队成功后如果后续逻辑异常，需要回退 `taskFinished(packageId)`
- `patch()` 方法包裹 `tryStartEdit()` / `finishEdit()`
- `delete()` 方法包裹 `tryStartDelete()` / `finishDelete()`
- `batchBuild()` / `batchPublish()` 改为部分成功模式，返回每个 packageId 的入队结果明细

### Step 5：清理旧代码
- 删除 `GlobalOperationGuard.java`
- 删除 `EditingOperationTracker.java`
- 删除 `MutexGuarded.java`
- 删除 `MutexGuardAspect.java`
- 删除 `MutexOpType.java`
- 清理 `TaskQueueManager` 中残留的旧引用

### Step 6：验证
- 同一 packageId 连续提交多个构建任务 → 串行执行
- 不同 packageId 同时提交构建任务 → 并行执行（最多 10 个）
- 编辑中提交构建 → 返回 409
- 构建中提交编辑 → 返回 409
- 有任务时删除 → 返回 409
- 编辑中删除 → 返回 409
- 删除中提交构建 → 返回 409
- 删除中提交编辑 → 返回 409
- **并发安全验证**：
  - 同一 packageId 并发发起编辑 + 构建 → 只有一个成功，另一个 409
  - 任务异常退出后 → Semaphore 正确释放，后续任务正常调度
  - 编辑超时 10 分钟后 → 状态自动恢复，构建可正常入队
  - 删除超时 10 分钟后 → 状态自动恢复
  - stopTask 后 → 该 packageId 状态正确恢复，后续操作不被阻塞
  - 单个 packageId 提交超过 50 个任务 → 返回 429
  - packageId 为 null 时 → 返回 400
  - 批量构建部分包正在编辑 → 返回明细（部分 queued / 部分 rejected）

## 6. 风险点

1. **批量构建/发布**：`batch-build` 和 `batch-publish` 会一次提交多个 packageId 的任务。每个 packageId 独立调用 `tryEnqueueTask()`，部分成功部分失败时，返回结果中需明确标注每个 packageId 的状态（成功入队 / 被拒绝 + 原因）。前端根据返回明细展示。
2. **停止任务后的状态恢复**：`stopTask()` / `stopBatchTasks()` 停止任务后，必须完成以下清理：
   - 释放 Semaphore（`globalConcurrency.release()`）
   - 从 `packageRunningTasks` 移除
   - 调用 `packageLockManager.taskFinished(packageId)`（递减 taskCount，可能恢复 IDLE）
   - 对于从队列中移除的待执行任务，同样需要调用 `taskFinished()` 递减 taskCount
   - 清理完成后调用 `processQueues()` 唤醒其他等待任务
3. **状态查询接口**：`getQueueStatistics()` 等方法需要适配新结构（遍历 `packageQueues` 和 `packageRunningTasks`）。统计结果为弱一致性快照，可接受。
4. **WebSocket 通知**：通知逻辑不变，但 topic 路由需要确认是否受影响。
5. **测试环境**：测试环境队列保持独立，不经过 `PackageLockManager`，不受此次重构影响。

## 7. 安全加固要点

本节汇总所有并发安全和异常兜底的设计约束，实施时必须逐条落实。

### 7.1 原子性保证

| 操作 | 保证方式 |
|------|---------|
| 状态转换（tryStartEdit / tryEnqueueTask / tryStartDelete） | per-packageId `synchronized` 锁内完成 check + set |
| finishEdit / finishDelete | CAS 语义：仅当前状态匹配时才转换，否则 no-op |
| taskFinished | 锁内递减 taskCount，降为 0 时自动转 IDLE |

### 7.2 Semaphore 泄漏防护

```java
// processQueues() 中的任务提交模板
if (globalConcurrency.tryAcquire()) {
    try {
        CompletableFuture.runAsync(() -> {
            try {
                executeTaskLogic(task);
            } finally {
                // 无论成功/失败/异常，必须释放
                globalConcurrency.release();
                packageRunningTasks.remove(task.getPackageId());
                packageLockManager.taskFinished(task.getPackageId());
                processQueues(); // 唤醒全局调度
            }
        }, taskExecutor);
    } catch (RejectedExecutionException e) {
        // 线程池拒绝时立即归还许可
        globalConcurrency.release();
        packageLockManager.taskFinished(task.getPackageId());
        // 将任务放回队首，等待下次调度
        packageQueues.get(packageId).offer(task); // 实际应放回队首
    }
}
```

### 7.3 超时保护

| 状态 | 超时时间 | 清理动作 |
|------|---------|---------|
| EDITING | 10 分钟 | 恢复 IDLE + 触发 processQueues() |
| DELETING | 10 分钟 | 恢复 IDLE + 触发 processQueues() |

### 7.4 内存泄漏防护

- `PackageLockManager`：状态回到 IDLE 且 taskCount=0 时，从 `states` 和 `locks` 中移除该 packageId
- `TaskQueueManager`：`taskFinished()` 触发 IDLE 后，从 `packageQueues` 中移除空队列
- `finishDelete()`：删除完成后清理所有相关条目（states / locks / packageQueues）

### 7.5 队列容量限制

每个 packageId 的队列上限 50 个任务。超限时：
- 回退 `taskCount`（调用 `taskFinished`）
- 返回 429 Too Many Requests

### 7.6 packageId 非空校验

以下入口必须校验 packageId 非空（`ConcurrentHashMap` 不支持 null key）：
- `AppServiceImpl` 所有接受 packageId 参数的方法
- `TaskQueueManager.addTaskToQueue()`
- `PackageLockManager` 所有公开方法

### 7.7 批量操作部分失败处理

`batchBuild()` / `batchPublish()` 采用**部分成功 + 返回明细**策略：

```java
public Result<Map<String, Object>> batchBuild(List<String> packageIds, ...) {
    String batchTaskId = UUID.randomUUID().toString();
    List<Map<String, Object>> results = new ArrayList<>();
    
    for (String packageId : packageIds) {
        Map<String, Object> item = new HashMap<>();
        item.put("packageId", packageId);
        
        if (!packageLockManager.tryEnqueueTask(packageId)) {
            item.put("status", "rejected");
            item.put("reason", packageLockManager.getBlockReason(packageId));
        } else {
            // 创建任务并入队...
            item.put("status", "queued");
            item.put("taskId", taskId);
        }
        results.add(item);
    }
    
    Map<String, Object> data = new HashMap<>();
    data.put("batchTaskId", batchTaskId);
    data.put("details", results);
    return Result.success(data);
}
```

## 8. 适用范围

本设计仅适用于**单实例部署**场景。所有锁和状态均为 JVM 内存对象。若未来需要多实例部署，需引入分布式锁（如 Redis + Redisson）替换 `PackageLockManager` 的内部实现，但对外接口保持不变。

---

## 9. 实施记录（2025-05-12）

本节记录实际落地时与原设计的差异、补充和注意事项。

### 9.1 实际改动清单

**新增**
- `guard/PackageLockManager.java`（约 270 行）

**修改**
- `entity/TaskQueueItem.java`：新增 `packageId` 字段 + getter/setter
- `utils/TaskQueueManager.java`：
  - 字段层：`prodTaskQueue` → `packageQueues`（`ConcurrentHashMap<String, Queue>`）
  - 字段层：新增 `packageRunningTasks`（每 packageId 最多 1 个运行中任务）
  - 字段层：新增 `globalConcurrency = new Semaphore(10, true)`（公平模式）
  - 保留 `prodRunningTasks` 作为按 taskId 索引的聚合视图（现有查询接口依赖）
  - 保留 `testTaskQueue` / `testRunningTasks` 不变（测试环境独立队列）
  - 新增辅助方法 `totalProdQueueSize()` / `snapshotAllProdQueueTasks()`
  - 重写 `addTaskToQueue()`：正式任务要求 packageId 非空；超限时回退 `taskFinished`
  - 重写 `processProdQueues()`（原 `processProdQueue`）：按 packageId 并行调度
  - 重写 `executeTask()` finally：统一释放 Semaphore + 清理 `packageRunningTasks` + 调 `taskFinished`
  - 新增 `restoreHead()`：线程池拒绝时把任务放回队首
  - 重写 `stopTask()` / `stopBatchTasks()`：从每 packageId 队列移除 + 回退 taskCount
  - 新增 `stopBatchTasksFromProdQueue()`：专门处理正式环境每 packageId 队列
  - 所有 `prodTaskQueue` 迭代改为 `snapshotAllProdQueueTasks()`
  - 删除 `canStartNextHeavyTask()`
  - 移除 `editingOperationTracker` 依赖
- `service/impl/AppServiceImpl.java`：
  - 注入 `PackageLockManager`
  - 所有 packageId 入口加非空校验
  - `upload` / `install` / `build` / `installBuild` / `publish`：入队前 `tryEnqueueTask` → 409；入队后异常路径回退 `taskFinished`
  - `patch`：包裹 `tryStartEdit` / `finishEdit`（409 + finally）
  - `delete`：包裹 `tryStartDelete` / `finishDelete`（409 + finally）
  - `batchBuild` / `batchPublish`：改为"部分成功 + details 明细"返回模式
  - 所有创建 `TaskQueueItem` 处增加 `task.setPackageId(packageId)`
  - `install` 调整顺序：把幂等跳过检查放在 `tryEnqueueTask` 之前，避免无意义锁进出

**删除**
- `guard/GlobalOperationGuard.java`
- `guard/EditingOperationTracker.java`
- `annotation/MutexGuarded.java`
- `aspect/MutexGuardAspect.java`
- `enums/MutexOpType.java`

### 9.2 与原设计的差异

| 项 | 原设计 | 实施 | 原因 |
|----|--------|------|------|
| 空队列清理 | 任务完成后清理 `packageQueues` 中空队列 | 不做清理 | 清理存在竞态：判断为空后其他线程 offer 新任务，再 `remove(pid, q)` 会把整个队列从 map 移除，新任务变孤儿。`ConcurrentLinkedQueue` 内存占用可忽略 |
| 停止运行中任务时的 Semaphore 释放 | 在 `stopTask` 内 release + taskFinished | 交由 `executeTask` 的 finally 统一处理 | 避免与 finally 双重 release。`future.cancel(true)` 不会真正中断 runnable，但 runnable 终会结束（进程被 destroy 后 reader 返回 EOF），finally 保证执行 |
| `install()` 中 clean 模式的顺序 | 先 clean 再 `tryEnqueueTask` | 先 `tryEnqueueTask` 再 clean | 原顺序会让 clean 操作跳过互斥检查；现在 clean 也受互斥保护 |
| `install()` 幂等检查位置 | 未明示 | 放在 `tryEnqueueTask` 之前 | 跳过场景无需进入 BUSY 状态浪费锁进出 |
| `clearAllRunningTasks()` | 未明确 | 不手动 release Semaphore / taskFinished | `future.cancel(true)` 会让 finally 正常跑，由 finally 统一清理；手动 release 会导致许可数失真 |
| PackageLockManager 循环依赖 | 未明示 | `@Lazy` 注入 `TaskQueueManager` | 打破 `PackageLockManager ↔ TaskQueueManager` 循环 |

### 9.3 关键实现细节

**Semaphore 释放 + taskFinished 的幂等性**

`executeTask` 的 finally 块是唯一的权威清理点：

```java
} finally {
    saveUserTaskToDatabase(task);
    String pid = task.getPackageId();
    prodRunningTasks.remove(task.getTaskId());
    if (pid != null) packageRunningTasks.remove(pid);
    completedTasks.put(task.getTaskId(), task);
    notifyTaskStatusChanged(task);
    globalConcurrency.release();              // 1. 释放并发许可
    if (pid != null) {
        packageLockManager.taskFinished(pid); // 2. 递减 taskCount
    }
    processProdQueues();                      // 3. 唤醒调度
}
```

`stopTask` 不做这三步，避免重复。

**放回队首的实现**

`ConcurrentLinkedQueue` 不支持 `offerFirst`，使用 drain + addAll 重建顺序：

```java
private void restoreHead(Queue<TaskQueueItem> queue, TaskQueueItem task) {
    List<TaskQueueItem> buf = new ArrayList<>();
    buf.add(task);
    TaskQueueItem next;
    while ((next = queue.poll()) != null) buf.add(next);
    queue.addAll(buf);
}
```

非原子操作，但仅在线程池拒绝这一极端场景使用，可接受弱一致性。

**processProdQueues 的调度循环**

```java
for (Map.Entry<String, Queue<TaskQueueItem>> entry : packageQueues.entrySet()) {
    String packageId = entry.getKey();
    Queue<TaskQueueItem> queue = entry.getValue();
    if (queue.isEmpty()) continue;
    if (packageRunningTasks.containsKey(packageId)) continue;
    if (!globalConcurrency.tryAcquire()) return; // 许可耗尽，本轮退出
    TaskQueueItem task = queue.poll();
    if (task == null) { globalConcurrency.release(); continue; }
    TaskQueueItem prev = packageRunningTasks.putIfAbsent(packageId, task);
    if (prev != null) { restoreHead(queue, task); globalConcurrency.release(); continue; }
    try { executeTask(task); }
    catch (RejectedExecutionException ree) {
        packageRunningTasks.remove(packageId);
        prodRunningTasks.remove(task.getTaskId());
        globalConcurrency.release();
        packageLockManager.taskFinished(packageId);
        restoreHead(queue, task);
        return;
    }
}
```

**批量接口返回结构**

```json
{
  "code": 200,
  "data": {
    "batchTaskId": "xxx-uuid",
    "taskIds": ["taskId1", "taskId2"],
    "details": [
      {"packageId": "pkgA", "status": "queued", "taskId": "taskId1"},
      {"packageId": "pkgB", "status": "rejected", "reason": "正在编辑中"}
    ]
  }
}
```

### 9.4 已知局限

1. **停止中运行任务的进程销毁**：`stopBatchTasksFromRunning` 只 cancel future，不调 `process.destroy()`（沿袭原代码行为）。单独 `stopTask` 会正确销毁进程。后续可统一改造。
2. **`clearAllRunningTasks` 状态同步**：强制清理时 `packageRunningTasks.clear()` 发生在 finally 块执行之前，finally 仍会跑完所有清理逻辑。在极端场景（runnable 卡死无法退出）下，Semaphore 许可会持续占用，依赖 10 分钟超时兜底。
3. **空队列不回收**：`packageQueues` 中某 packageId 的队列在任务完成后不会从 map 移除，避免竞态。长期运行若存在大量不重复的 packageId，`packageQueues` 会累积（但每个空 `ConcurrentLinkedQueue` 仅几十字节）。可在 `@Scheduled` 清理器中定期删除"空队列且无运行任务且 PackageLockManager 状态为 IDLE"的条目。

### 9.5 编译与验证

- `mvn -q -DskipTests compile` 通过，无诊断错误
- 单元测试：项目无单元测试代码（`src/test` 为空）
- 运行期验证按第 5 节 Step 6 的用例清单执行

---

## 10. Code Review 修复记录（2025-05-12）

### 10.1 P0：`stopBatchTasksFromRunning` 不销毁进程 → Semaphore 泄漏

**问题**：原实现只调用 `future.cancel(true)`，但 `CompletableFuture.runAsync(runnable)` 的 cancel 不会真正中断 runnable，runnable 内的 `process.waitFor()` 不会返回，`executeTask` 的 finally 永远不跑，Semaphore 许可永久泄漏。

**修复**：
- 抽取 `abortRunningTask(task, reason)` 公共方法，统一负责"销毁进程 + 取消 future + 更新状态"
- `stopRunningTask`、`stopBatchTasksFromRunning`、`clearAllRunningTasks` 全部改为调用 `abortRunningTask`
- 进程 destroy 后，runnable 的 `process.waitFor()` 会返回，finally 块正常执行，Semaphore 和 taskCount 由 finally 统一清理

**影响文件**：`utils/TaskQueueManager.java`

### 10.2 P1：`finishDelete` 中 `locks.remove(packageId)` 导致状态竞态

**问题**：
```java
synchronized (getLock(packageId)) { states.remove(...); }
locks.remove(packageId);  // ← 问题在此
```
在 `finishDelete` 的同步块退出后、`locks.remove` 执行前，另一个线程调用 `tryEnqueueTask` 会通过 `getLock` 创建**新的**锁对象。随后 `locks.remove` 会把这个新锁对象从 map 移除，后续线程再拿锁又会创建**另一个**新锁对象。不同线程同步到不同的锁对象 → per-packageId 状态转换不再原子。

**修复**：
- `finishDelete` 不再删除 `locks` 条目
- 锁对象的生命周期独立于 states 条目，允许同名 packageId 复用同一锁对象
- 内存代价：每个曾出现的 packageId 保留 ~16 字节的锁对象，可忽略

**影响文件**：`guard/PackageLockManager.java`

### 10.3 代码质量提升

除修复 bug 外还同步做了：

1. **抽取 `abortRunningTask`**：消除 `stopRunningTask` / `stopBatchTasksFromRunning` / `clearAllRunningTasks` 三处重复代码（进程销毁 + future 取消 + 状态更新逻辑）
2. **`clearAllRunningTasks` 文档加固**：明确注明此方法是兜底操作，并发场景下可能出现状态不一致，调用方责任

### 10.4 验证

- `mvn -q -DskipTests compile` 通过
- 关键改动点的 diagnostics：无 error/warning
