# FunMiniFactory Redis 优先同步方案

## 1. 背景

FunMiniFactory 要展示多个子系统里的小程序和 H5 项目。

弱服务器下需要避免：

```text
前端频繁轮询 Manager
Manager 频繁查 MySQL
Manager 频繁请求多个子系统
```

因此 Redis 方案的核心不是把所有东西都改成 Redis，而是：

```text
前端读 Redis 缓存
同步任务低频更新 Redis
页面刷新尽量用 SSE 推送，减少轮询
MySQL 只保底保存关键事实数据
```

## 2. 结论

推荐方案：

```text
Redis 做前端读取层和同步状态层
MySQL 做可靠水位和可恢复数据
子系统仍然提供全量 / 增量接口
Manager 后台同步后写 Redis
前端优先读 Redis，不直接打子系统
页面更新用 SSE，轮询只作为兜底
```

不建议：

```text
前端每 2 秒轮询 /api/projects
前端每 2 秒轮询 /api/project-sync/status
每次请求都查 MySQL
每次请求都请求子系统
```

第一版建议：

```text
前端 GET /api/projects 读 Redis
前端 GET /api/projects/stats 读 Redis
前端用 SSE 监听同步版本变化
Manager 每 5 分钟同步一次子系统
手动同步时由后端异步执行
```

## 3. 总体架构

```text
子系统
  ├── GET /api/open/apps/all
  └── GET /api/open/apps/updates
        ↓
FunMiniFactoryManager 同步任务
        ↓
MySQL: factory_app_sync       保存可靠同步水位
Redis: factory:apps:*         保存项目列表 / 详情 / 统计 / 同步状态
        ↓
FunMiniFactory 前端
  ├── 首屏读 /api/projects
  ├── 统计读 /api/projects/stats
  └── SSE 监听 /api/project-sync/events
```

读链路：

```text
前端 -> Manager -> Redis
```

同步链路：

```text
Manager 定时任务 -> 子系统 -> Redis + MySQL 水位
```

不要做：

```text
前端 -> Manager -> 子系统
```

## 4. Redis 存什么

Redis 存适合高频读取、可重建的数据。

### 4.1 项目详情

每个项目一个 Hash 或 JSON：

```text
factory:app:{source}:{appId}
```

示例：

```json
{
  "source": "novel_miniapp",
  "appId": "101",
  "appName": "番茄小说",
  "category": "novel",
  "platform": "wechat",
  "status": "",
  "version": "1.0.3",
  "owner": "lijj",
  "createTime": "2026-05-29T10:00:00+08:00",
  "updateTime": "2026-05-29T10:10:00+08:00",
  "lastChangeId": 10086,
  "isDeleted": false
}
```

### 4.2 项目索引

全部项目 ID：

```text
factory:apps:index:all
```

类型索引：

```text
factory:apps:index:category:{category}
```

平台索引：

```text
factory:apps:index:platform:{platform}
```

最近新增排序：

```text
factory:apps:zset:recent
```

建议：

```text
Set 存成员：{source}:{appId}
ZSet score 存创建时间戳
```

### 4.3 统计缓存

```text
factory:apps:stats
```

示例：

```json
{
  "total": 286,
  "byCategory": [
    { "category": "novel", "count": 100 },
    { "category": "video", "count": 60 },
    { "category": "h5", "count": 40 }
  ],
  "updatedAt": "2026-05-29T19:30:00+08:00",
  "version": 108
}
```

### 4.4 同步状态

每个 source 一份：

```text
factory:sync:state:{source}
```

示例：

```json
{
  "source": "novel_miniapp",
  "status": "success",
  "syncing": false,
  "lastSuccessChangeId": 10086,
  "lastSuccessTime": "2026-05-29T19:30:00+08:00",
  "lastAttemptTime": "2026-05-29T19:30:00+08:00",
  "lastError": ""
}
```

全局版本：

```text
factory:apps:version
```

每次同步导致项目或统计变化时：

```text
INCR factory:apps:version
```

前端只需要知道版本变了，然后重新拉列表和统计。

## 5. MySQL 还要不要

建议保留 MySQL，但只保留低频、关键、不可丢的数据。

最少保留一张表：

```text
factory_app_sync
```

用于保存同步水位：

```sql
create table factory_app_sync (
    id                       bigint        primary key auto_increment,
    source                   varchar(64)   not null,
    last_success_change_id   bigint        not null default 0,
    last_success_time        datetime      null,
    last_attempt_time        datetime      null,
    last_status              varchar(32)            default '',
    last_error               varchar(1024)          default '',
    created_at               datetime      not null default current_timestamp,
    updated_at               datetime      not null default current_timestamp on update current_timestamp,
    unique key uk_source (source)
);
```

原因：

```text
Redis 可以重建
同步水位不能轻易丢
服务器重启或 Redis 清空后，Manager 要知道从哪里继续同步
```

如果想更稳，可以继续保留 `factory_app` 作为落库备份；如果服务器很弱，第一版可以不把每个项目都写 MySQL，只写 Redis + 水位 MySQL。

两种落地选择：

```text
轻量版：MySQL 只存 factory_app_sync，项目数据只在 Redis。
稳妥版：MySQL 存 factory_app_sync + factory_app，Redis 是读取缓存。
```

弱服务器优先选轻量版。

## 6. 子系统接口

子系统接口不变：

```http
GET /api/open/apps/all?cursor=&limit=500
GET /api/open/apps/updates?afterChangeId=xxx&limit=500
```

字段仍然统一：

```text
appId
appName
category
platform
appToken
version
status
owner
sourcePath
buildCmd
artifactDir
createTime
updateTime
```

`status` 可以为空。

敏感字段：

```text
appToken / sourcePath / buildCmd / artifactDir 不在普通前端列表接口返回。
```

## 7. 同步策略

### 7.1 首次全量同步

流程：

```text
1. Manager 读取 MySQL factory_app_sync.last_success_change_id。
2. 如果 source 没有水位，执行全量同步。
3. 请求子系统 /api/open/apps/all。
4. 子系统返回 snapshotChangeId。
5. Manager 分页拉取所有项目。
6. Manager 写 Redis 项目详情和索引。
7. Manager 重建统计缓存。
8. 全部成功后，MySQL 水位更新为 snapshotChangeId。
9. INCR factory:apps:version。
10. 发布 Redis Pub/Sub 事件。
```

`cursor` 和 `snapshotChangeId` 都不需要长期存库。同步任务失败后，第一版直接重跑全量。

### 7.2 日常增量同步

流程：

```text
1. Manager 从 MySQL 读取 last_success_change_id。
2. 请求子系统 /api/open/apps/updates?afterChangeId=xxx&limit=500。
3. 按 changeId 升序处理事件。
4. UPSERT 写 Redis 项目详情和索引。
5. DELETE 从 Redis 索引移除，并将详情标记 isDeleted=true。
6. hasMore=true 时继续用 nextChangeId 拉下一页。
7. 全部成功后，MySQL 水位更新为最后处理的 changeId。
8. 重建或局部更新统计。
9. INCR factory:apps:version。
10. 发布 Redis Pub/Sub 事件。
```

写入规则：

```text
event.changeId <= Redis 中 lastChangeId -> 跳过
event.changeId > Redis 中 lastChangeId -> 应用变更
```

这样重复同步不会把旧数据覆盖新数据。

## 8. 如何减少 HTTP 轮询

### 8.1 不要轮询列表

前端不要定时请求：

```http
GET /api/projects
```

列表只在这些时机请求：

```text
页面首次打开
用户切换筛选条件
收到同步版本变化事件
用户手动点刷新
```

### 8.2 用 SSE 推送版本变化

前端建立一个长连接：

```http
GET /api/project-sync/events
```

后端用 Server-Sent Events 返回：

```text
event: sync-version
data: {"version":109,"updatedAt":"2026-05-29T19:35:00+08:00"}
```

前端收到后：

```text
如果 version 比本地大 -> 重新请求 /api/projects 和 /api/projects/stats
```

这样平时没有变化时，不会反复打 HTTP 接口。

### 8.3 SSE 失败时低频兜底

SSE 断开或浏览器不支持时，再低频轮询：

```text
每 60 秒请求一次 /api/project-sync/version
```

接口只读一个 Redis key：

```text
GET factory:apps:version
```

这个请求非常轻。

## 9. Redis Pub/Sub 和 SSE

Manager 同步成功后发布事件：

```text
PUBLISH factory:apps:events {"type":"sync-version","version":109}
```

Manager 的 SSE 服务订阅：

```text
SUBSCRIBE factory:apps:events
```

收到事件后转发给前端 SSE 连接。

如果 Manager 只有一个实例，也可以不用 Pub/Sub，直接在同步完成后通知本进程里的 SSE 客户端。

如果 Manager 未来多实例，Pub/Sub 更合适。

## 10. Redis Key 设计

```text
factory:app:{source}:{appId}                  项目详情
factory:apps:index:all                        全部项目 Set
factory:apps:index:category:{category}        分类 Set
factory:apps:index:platform:{platform}        平台 Set
factory:apps:zset:recent                      最近新增 ZSet
factory:apps:stats                            统计 JSON
factory:apps:version                          全局数据版本
factory:sync:state:{source}                   同步状态 JSON
factory:sync:lock:{source}                    同步锁
factory:apps:events                           Pub/Sub channel
```

成员 ID 统一：

```text
{source}:{appId}
```

例如：

```text
novel_miniapp:101
```

## 11. Redis TTL 策略

项目目录数据不建议设置短 TTL。

原因：

```text
如果 key 过期，页面会突然没数据。
```

建议：

```text
项目详情：不设置 TTL
项目索引：不设置 TTL
统计缓存：不设置 TTL
同步状态：不设置 TTL
同步锁：必须设置 TTL，比如 10 分钟
```

Redis 数据丢失时：

```text
Manager 启动后检测 factory:apps:version 不存在
触发全量重建 Redis
```

## 12. 同步锁

同一个 source 同一时间只允许一个同步任务。

Redis 锁：

```text
SET factory:sync:lock:{source} {uuid} NX EX 600
```

拿到锁：

```text
执行同步
```

没拿到锁：

```text
跳过
```

释放锁时必须校验 value：

```text
只有 value 等于本次 uuid 才删除锁
```

防止误删别的同步任务的锁。

## 13. 前端接口

### 13.1 项目列表

```http
GET /api/projects?category=&platform=&keyword=&page=1&pageSize=20
```

后端逻辑：

```text
1. 根据 category / platform 从 Redis Set 取候选 ID。
2. 批量 MGET / HMGET 项目详情。
3. 过滤 isDeleted=true。
4. 在内存里做 keyword 过滤。
5. 分页返回。
```

弱服务器注意：

```text
pageSize 限制最大 50。
keyword 搜索第一版只做简单包含匹配。
项目数量很大时再加 RediSearch 或 MySQL 搜索。
```

### 13.2 统计

```http
GET /api/projects/stats
```

直接读：

```text
factory:apps:stats
```

### 13.3 同步版本

```http
GET /api/project-sync/version
```

返回：

```json
{
  "version": 109
}
```

只读：

```text
factory:apps:version
```

### 13.4 SSE 事件

```http
GET /api/project-sync/events
```

返回：

```text
event: sync-version
data: {"version":109}
```

### 13.5 手动同步

```http
POST /api/project-sync/run
POST /api/project-sync/run/{source}
```

不要同步完成才返回。

正确方式：

```text
1. 接口收到请求。
2. 投递后台任务。
3. 立即返回 taskId。
4. 同步状态写 Redis。
5. 前端通过 SSE 或低频 version 接口感知完成。
```

## 14. 统计怎么维护

第一版建议同步完成后全量重算统计。

原因：

```text
项目数量通常不大
逻辑简单
不容易因为增删改导致计数错
```

流程：

```text
1. 扫 factory:apps:index:all。
2. 批量读取项目详情。
3. 过滤 isDeleted=true。
4. 计算 total / byCategory / byPlatform / recent。
5. 写 factory:apps:stats。
```

如果项目很多，再改成增量维护计数。

## 15. Redis 持久化建议

如果项目数据只放 Redis，必须开启持久化。

建议：

```text
appendonly yes
appendfsync everysec
```

也就是 AOF 每秒落盘。

如果服务器磁盘也很弱，可以接受 Redis 丢失后全量重建，则可以只保留 MySQL 水位，Redis 重启后触发全量重建。

## 16. 降级策略

Redis 不可用时：

```text
前端返回“数据暂不可用”
不要实时打所有子系统兜底
不要让用户请求触发全量同步
```

Manager 后台持续尝试恢复 Redis。

Redis 数据为空时：

```text
Manager 异步全量重建
前端显示“数据同步中”
```

子系统不可用时：

```text
保留 Redis 旧数据
sync state 标记 failed
页面仍展示旧数据和最后同步时间
```

## 17. 和纯 MySQL 方案的区别

纯 MySQL 方案：

```text
前端请求 -> Manager -> MySQL
页面通过轮询 status 感知变化
```

Redis 优先方案：

```text
前端请求 -> Manager -> Redis
页面通过 SSE 感知变化
MySQL 只保存同步水位
```

对弱服务器更友好：

```text
少查 MySQL
少轮询 HTTP
少实时聚合
列表和统计都读 Redis
```

但代价是：

```text
Redis key 设计和重建逻辑要写清楚
Redis 持久化要配置好
Redis 丢失后要能全量重建
```

## 18. 推荐落地顺序

第一阶段：

```text
1. 子系统统一全量接口。
2. Manager 全量同步写 Redis。
3. MySQL 只建 factory_app_sync 保存水位。
4. 前端列表和统计读 Redis。
5. 手动同步异步执行。
```

第二阶段：

```text
1. 子系统增加 app_change_log。
2. Manager 增量同步写 Redis。
3. 同步成功后 INCR factory:apps:version。
4. 前端低频请求 /api/project-sync/version。
```

第三阶段：

```text
1. 增加 SSE /api/project-sync/events。
2. 同步完成后发布 sync-version 事件。
3. 前端收到事件后刷新列表和统计。
4. 多实例时使用 Redis Pub/Sub。
```

第四阶段：

```text
1. 增加 Redis 分布式锁。
2. 增加 Redis AOF 配置。
3. 增加 Redis 数据丢失后的全量重建。
```

## 19. 最终建议

如果你的主要担心是服务器扛不住频繁 HTTP 轮询，优先改这两点：

```text
1. 列表和统计全部读 Redis。
2. 前端用 SSE 监听版本变化，轮询只做 60 秒兜底。
```

不要把“用户打开页面”变成“触发子系统同步”。

最终数据流：

```text
子系统 -> Manager 后台同步 -> Redis 项目目录 -> 前端读取
                             -> MySQL 保存同步水位
```

这个方案对弱服务器更友好，同时还能保留同步可靠性和失败恢复能力。
