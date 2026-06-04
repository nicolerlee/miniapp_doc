# 定时拉取（同步）规则说明

> 适用范围：`FunMiniFactoryManager`（后端）的项目同步定时任务。
> 关联文档：
> - 同步总体方案：[`project-sync-design.md`](./project-sync-design.md)
> - Redis 优化补充：[`project-sync-design-redis.md`](./project-sync-design-redis.md)
>
> 本文只聚焦"定时拉取"这件事：**多久拉一次、在哪里配、按什么规则拉**。

---

## 1. 一句话总结

后端有两个定时任务：**增量拉取**（高频，开发 30 秒 / 生产 2 分钟）和**全量对账**（每天凌晨 3:00 兜底）。
频率由配置项 `factory.sync.incremental-cron` 和 `factory.sync.reconcile-cron` 控制，分环境写在 `application-dev.yml` / `application-prod.yml` 里。

---

## 2. 两个定时任务

| 任务 | 方法 | 作用 | 拉取接口 | 同步模式 |
|---|---|---|---|---|
| 增量拉取 | `incrementalSync()` | 高频拉子系统的变更事件，保持数据近实时 | `GET /api/open/apps/updates` | `SyncMode.AUTO` |
| 全量对账 | `fullReconcile()` | 每天兜底跑一次全量，修正增量漏掉/错过的数据 | `GET /api/open/apps/all` | `SyncMode.FULL` |

代码位置：`scheduler/ProjectSyncScheduler.java`

```java
@Scheduled(cron = "${factory.sync.incremental-cron:0 */2 * * * *}")
public void incrementalSync() {
    if (!syncProperties.isScheduledEnabled()) return;   // 总开关
    syncOrchestrator.syncAll(SyncMode.AUTO);
}

@Scheduled(cron = "${factory.sync.reconcile-cron:0 0 3 * * *}")
public void fullReconcile() {
    if (!syncProperties.isScheduledEnabled()) return;   // 总开关
    syncOrchestrator.syncAll(SyncMode.FULL);
}
```

> 注意：`SyncMode.AUTO` 不一定走增量。若某个 source 的水位线 `lastSuccessChangeId <= 0`（即从未成功同步过），首次会自动退化为全量拉取；之后才是真正的增量。

---

## 3. 频率到底是多少（按环境）

频率取决于启动时激活的 Spring Profile（`spring.profiles.active`，默认 `dev`，线上用 `-Dspring.profiles.active=prod`）。

| 配置项 | 含义 | 开发环境 `application-dev.yml` | 生产环境 `application-prod.yml` | 代码兜底默认值 |
|---|---|---|---|---|
| `incremental-cron` | 增量拉取频率 | `0/30 * * * * *`（**每 30 秒**） | `0 */2 * * * *`（**每 2 分钟**） | `0 */2 * * * *` |
| `reconcile-cron` | 全量对账频率 | `0 0 3 * * *`（**每天 3:00**） | `0 0 3 * * *`（**每天 3:00**） | `0 0 3 * * *` |
| `scheduled-enabled` | 定时任务总开关 | `true` | `true` | `true` |

代码兜底默认值在 `config/FactorySyncProperties.java`，只有当两个 yml 都没配时才会用到。

### cron 表达式格式

Spring 的 cron 是 **6 位**：`秒 分 时 日 月 周`（注意比 Linux crontab 多了最前面的"秒"）。

- `0/30 * * * * *` → 从第 0 秒起每 30 秒触发一次
- `0 */2 * * * *` → 每 2 分钟的第 0 秒触发一次
- `0 0 3 * * *` → 每天 03:00:00 触发一次

---

## 4. 在哪里配置

按生效优先级从高到低：

### ① 环境变量（仅 dev 暴露，临时覆盖最方便）

`application-dev.yml` 采用 `${环境变量:默认值}` 写法，可在启动时用环境变量临时覆盖，不用改代码：

```bash
# 临时改成每 10 秒拉一次
FACTORY_SYNC_INCREMENTAL_CRON="0/10 * * * * *" mvn spring-boot:run

# 临时关掉所有定时任务（仍可手动触发）
FACTORY_SYNC_SCHEDULED_ENABLED=false mvn spring-boot:run
```

| 环境变量 | 对应配置 |
|---|---|
| `FACTORY_SYNC_INCREMENTAL_CRON` | 增量拉取频率 |
| `FACTORY_SYNC_SCHEDULED_ENABLED` | 定时任务总开关 |

### ② 环境配置文件（改频率的常规做法）

- 本地改频率 → `FunMiniFactoryManager/src/main/resources/application-dev.yml`
- 线上改频率 → `FunMiniFactoryManager/src/main/resources/application-prod.yml`

```yaml
factory:
  sync:
    scheduled-enabled: true            # 定时任务总开关
    incremental-cron: "0 */2 * * * *"  # 增量拉取频率
    reconcile-cron: "0 0 3 * * *"      # 全量对账频率
```

### ③ 代码默认值（兜底）

`FunMiniFactoryManager/src/main/java/com/funshion/funminifactory/config/FactorySyncProperties.java`

```java
private boolean scheduledEnabled = true;
private String incrementalCron = "0 */2 * * * *";
private String reconcileCron   = "0 0 3 * * *";
```

---

## 5. 拉取时遵循的规则

定时任务最终都调 `service/SyncOrchestrator.java`，核心规则如下：

1. **逐 source 拉取，可单独关闭**
   `syncAll()` 遍历 `factory.sync.sources`，`sync-enabled: false` 的直接跳过。
   - dev：`playlet_miniapp` 关闭，其余开启。
   - prod：仅 `novel_miniapp`、`yingshi_miniapp` 开启，其余关闭。

2. **同一 source 同一时刻只跑一个任务**
   通过 `SourceLockRegistry`（JVM 内 `ReentrantLock`）加锁；抢不到锁直接返回 `source is already syncing`，避免定时任务和手动触发撞车。

3. **增量靠水位线幂等推进**
   水位线 = `factory_app_sync.last_success_change_id`。只处理 `changeId > 水位线` 的事件，处理完把水位推进到最大 `changeId`（或响应里的 `nextChangeId`）。

4. **增量发现断层自动回退全量**
   若本地水位 `< 响应 minChangeId - 1`（说明中间漏了事件），自动 fallback 到全量同步补齐。

5. **全量用 snapshotChangeId 推进水位**
   全量分页期间要求 `snapshotChangeId` 全程一致，用它（而非结束时的 maxChangeId）推进水位，保证快照一致性。

6. **全量后处理下线数据**
   全量结束后调 `markMissingAfterFull(...)`，连续 `missing-delete-threshold`（默认 3）次全量都没出现的应用判定为下线/软删除。

7. **事件类型**
   - `UPSERT`：构建 `FactoryApp` 后 `upsertIfNewer`（按 changeId 比较，旧的不覆盖新的）。
   - `DELETE`：`softDeleteIfNewer` 软删除。
   - 其它操作类型直接抛异常。

8. **入库前校验与归类**
   `appId/appName` 为空、`platform/status/category` 非法、缺 `createTime/updateTime` 的条目会被跳过。
   归类规则：配了 `category-allowed` 的只收白名单内分类（如 `h5_novel`）；否则用 `category-default` 兜底。

9. **失败处理**
   单个 source 失败会 `markFailure` 记录错误信息并打日志，但不影响其它 source 继续同步。

---

## 6. 手动触发（与定时任务对照）

定时任务之外，前端也可手动触发同步（不受 cron 频率限制，但同样受第 5 节第 2 条的并发锁约束）：

| 接口 | 作用 |
|---|---|
| `POST /api/project-sync/run` | 异步触发全部 source 同步，返回 `taskId` |
| `POST /api/project-sync/run/{source}` | 异步触发单个 source 同步，返回 `taskId` |
| `GET /api/project-sync/tasks/{taskId}` | 查询手动任务进度 |
| `GET /api/project-sync/status` | 查询各 source 的同步水位与状态（前端默认每 30 秒轮询） |

前端封装见 `FunMiniFactory/src/api/sync.js`。

---

## 7. 常见操作速查

| 我想… | 怎么做 |
|---|---|
| 本地改增量频率 | 改 `application-dev.yml` 的 `incremental-cron`，或启动加环境变量 `FACTORY_SYNC_INCREMENTAL_CRON` |
| 线上改增量频率 | 改 `application-prod.yml` 的 `incremental-cron` 后重启 |
| 临时停掉所有定时拉取 | `scheduled-enabled: false`（或环境变量 `FACTORY_SYNC_SCHEDULED_ENABLED=false`） |
| 只停某个子系统的拉取 | 把对应 `sources[].sync-enabled` 设为 `false` |
| 改全量对账时间 | 改 `reconcile-cron`（默认每天 3:00） |
| 立即拉一次 | 前端点同步，或直接 `POST /api/project-sync/run` |
