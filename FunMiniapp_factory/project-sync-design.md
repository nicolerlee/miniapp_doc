# FunMiniFactory 项目聚合同步方案

## 1. 背景

FunMiniFactory 需要统一展示公司内部多个小程序和 H5 项目，包括小说、短剧、漫剧、影视、创意小程序、H5 小说、H5 影视、H5 分销、H5 拉新等。

现有数据分散在多个子系统中：

- 小说小程序
- 短剧小程序
- 漫剧小程序
- 影视小程序
- 创意小程序
- H5 小说 / H5 影视 / H5 分销 / H5 拉新

如果前端列表页每次都实时请求多个子系统接口，会带来几个问题：

- 页面速度受最慢子系统影响。
- 任意子系统失败会影响总览页体验。
- 统计、搜索、筛选需要反复聚合，成本高。
- 子系统接口格式、字段和稳定性会影响前端。

因此，FunMiniFactory 应该做成“项目目录系统”，列表和统计读取自己的数据库，子系统接口只作为数据来源。

## 2. 目标

核心目标：

- 前端列表页快速打开。
- 所有项目数据格式统一。
- 支持项目数量统计、类型占比、最近新增。
- 子系统失败不影响总览页展示已有数据。
- 支持全量同步、增量同步、手动同步。
- 支持后续扩展更多子系统。

非目标：

- 不追求秒级强实时。
- 不在用户打开列表页时实时请求所有子系统。
- 第一版不引入复杂消息队列。

## 3. 总体架构

整体涉及两类系统，三张表分布在不同系统的不同库里：

```text
┌─────────────────────────────────┐    ┌─────────────────────────────────┐
│  子系统（小说/漫剧/影视/H5...） │    │  FunMiniFactoryManager           │
│  各自独立部署，各有自己的库     │    │  独立部署，自己的中央库          │
│                                 │    │                                  │
│  - novel_app (业务表，已存在)   │    │  - factory_app (项目目录) ⭐    │
│  - app_change_log (变更流水) ⭐ │    │  - factory_app_sync (同步水位)⭐│
│                                 │    │                                  │
│  对外暴露：                     │    │  对前端暴露：                    │
│    GET /api/open/apps/all       │ <- │    GET  /api/projects            │
│    GET /api/open/apps/updates   │拉取│    GET  /api/projects/stats      │
└─────────────────────────────────┘    │    POST /api/project-sync/run    │
                                       └─────────────────────────────────┘
                                                       ↑
                                                  FunMiniFactory 前端
```

⭐ 标记的是同步方案需要新增的表。

数据流：

```text
子系统业务表 -> 子系统 app_change_log -> Manager 拉取 -> Manager factory_app -> 前端读取
```

前端只请求 FunMiniFactoryManager：

```text
前端 -> FunMiniFactoryManager -> MySQL
```

不要做成：

```text
前端 -> FunMiniFactoryManager -> 多个子系统 -> 返回前端
```

也不要做成：

```text
前端 -> 多个子系统接口
```

## 4. 同步模式选择

采用 Pull 拉模式。

流程：

```text
1. 子系统新增、修改、删除项目。
2. 子系统在本地记录一条变更记录。
3. FunMiniFactoryManager 定时请求子系统增量接口。
4. FunMiniFactoryManager 将变化写入自己的 MySQL。
5. 前端列表页读取 FunMiniFactoryManager 的 MySQL 数据。
```

Pull 模式的好处：

- FunMiniFactoryManager 挂了也不丢数据，恢复后继续拉。
- 子系统不需要知道 FunMiniFactoryManager 是否同步成功。
- 同步节奏由 FunMiniFactoryManager 控制。
- 更适合项目总览这种低频变化场景。

不建议第一版使用子系统主动 Push 完整数据，因为 Push 需要处理失败重试、重复投递、顺序、幂等、限流、死信等问题。

可选优化：

```text
Push 只做通知，Pull 拉数据
```

即子系统有变化时只通知 FunMiniFactoryManager “我有变化了”，FunMiniFactoryManager 收到通知后仍然主动拉增量数据。通知丢失也没关系，因为定时同步会兜底。

## 5. 子系统接口规范

每个子系统对外暴露两个接口，由 FunMiniFactoryManager 调用：

```http
GET /api/open/apps/all                                    全量
GET /api/open/apps/updates?afterChangeId=xxx&limit=500    增量
```

`/api/open/` 命名空间专门表示「子系统对外开放、给 Manager 调用」的接口，避免和 Manager 自身给前端的接口混淆。

每个子系统部署在不同的 host/端口，路径相同。Manager 在配置里维护「source -> baseUrl」映射，遍历调用即可。例如：

```text
http://172.17.7.183:8099/api/open/apps/all              漫剧子系统
http://172.17.7.183:8090/api/open/apps/all              影视子系统
http://172.17.7.183:8080/miniapp/novel/api/open/apps/all 小说子系统
http://172.17.3.118:8080/api/open/apps/all              H5 子系统
```

当前落地前提：

```text
第一版先认为各子系统已经提供 /api/open/apps/all 和 /api/open/apps/updates。
登录态、子系统鉴权、旧 appLists 接口适配暂不纳入本轮架构设计。
```

### 5.1 全量接口

用于首次同步和兜底校准。

```http
GET /api/open/apps/all?cursor=&limit=500
```

全量接口必须基于同一个快照水位返回数据。子系统收到第一页请求时，先固定当前最大变更 ID，作为本次全量同步的 `snapshotChangeId`。后续分页必须继续使用同一个 `snapshotChangeId`，不能每页重新取当前最大值。

后续分页请求带回该水位：

```http
GET /api/open/apps/all?cursor=101&limit=500&snapshotChangeId=10086
```

原因是全量同步过程中子系统仍然可能新增或修改项目。如果全量结束时直接使用最新 `maxChangeId` 推进水位，可能跳过全量过程中产生但没有被本轮分页扫到的变更。

返回：

```json
{
  "source": "novel_miniapp",
  "hasMore": false,
  "nextCursor": null,
  "snapshotChangeId": 10086,
  "items": [
    {
      "appId": "101",
      "appName": "番茄小说",
      "category": "novel",
      "platform": "wechat",
      "appToken": "",
      "version": "1.0.3",
      "status": "online",
      "owner": "lijj",
      "sourcePath": "git@gitlab.com:miniapp/novel-app.git",
      "buildCmd": "npm install && npm run build",
      "artifactDir": "dist",
      "createTime": "2026-05-29T10:00:00+08:00",
      "updateTime": "2026-05-29T10:10:00+08:00"
    }
  ]
}
```

全量接口查询子系统当前项目表，返回当前有效项目。

全量分页规则：

```text
cursor = 上一页最后一条记录的稳定排序键
排序规则 = 子系统业务主键升序
nextCursor = 本页最后一条记录的业务主键
```

建议查询方式：

```sql
select *
from app
where id > :cursor
order by id asc
limit :limit;
```

如果某个子系统业务主键不是单调递增数字，可以使用固定格式的字符串 cursor，但必须保证排序稳定、分页不会跳数据。

`cursor` 和 `snapshotChangeId` 不需要作为项目业务字段长期保存：

```text
cursor = 本次全量分页的临时位置，Manager 在当前同步任务内持有即可。
snapshotChangeId = 本次全量同步开始时的快照水位，Manager 在当前同步任务内持有；全量成功后写入 factory_app_sync.last_success_change_id。
```

子系统也不需要单独建表保存 `snapshotChangeId`，它通常就是第一页请求时读取到的 `app_change_log.id` 最大值。只有在希望全量同步任务失败后从中间页继续恢复时，才需要把 `cursor` 和 `snapshotChangeId` 存到任务状态表或 Redis；第一版可以失败后重新开始全量同步。

### 5.2 字段说明

接口返回的项目对象（`items[]` 或 `data`）字段约定：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `appId` | string | 是 | 子系统内唯一 ID，主键。和 `source` 一起作为 FunMiniFactoryManager 的唯一定位 |
| `appName` | string | 是 | 项目展示名 |
| `category` | enum | 是 | 业务分类，取值见下 |
| `platform` | enum | 是 | 投放平台，取值见下 |
| `appToken` | string | 否 | 项目 token / appKey / 接入凭证。敏感字段，不允许在普通前端列表接口返回明文 |
| `version` | string | 否 | 当前版本号，无版本概念可传空字符串 |
| `status` | enum | 否 | 项目状态，取值见下。部分子系统暂无状态控制时可为空 |
| `owner` | string | 否 | 项目负责人姓名 / 花名 |
| `sourcePath` | string | 否 | 项目源码地址，例如 Git 仓库 URL |
| `buildCmd` | string | 否 | 构建命令，例如 `npm install && npm run build` |
| `artifactDir` | string | 否 | 构建产物目录，例如 `dist` / `build` |
| `createTime` | datetime | 是 | 项目在子系统的创建时间，ISO 8601，必须带时区 |
| `updateTime` | datetime | 是 | 项目在子系统的最近更新时间，ISO 8601，必须带时区 |

`category` 枚举（H5 类目按方案 B 全部拆成顶级）：

```text
novel            小说小程序
drama            短剧小程序
comic            漫剧小程序
video            影视小程序
h5_novel         H5 小说
h5_yingshi       H5 影视
h5_fx            H5 分销
h5_lx            H5 拉新
```

注意：**没有顶级的 `h5` 值**。如果要统计「所有 H5 项目」，前端用 `category startsWith 'h5_'`，或后端在 `/api/projects/stats` 里聚合 `H5*` 这一组。

`platform` 枚举：

```text
weixin  微信小程序
douyin   抖音小程序
kuaishou 快手小程序
baidu    百度小程序
web      H5 / 普通网页
```

`status` 枚举：

```text
online    上线 / 启用
offline   下线
disabled  禁用
draft     草稿 / 未发布
```

子系统按需扩展时只新增枚举值，不复用旧值的语义。

第一版兼容部分子系统没有状态控制：

```text
子系统没有状态字段 -> 返回 null 或空字符串。
Manager 入库 status 允许为空。
前端展示时可将空状态显示为 unknown 或不展示状态标签。
统计时不要把空状态当成 online。
```

`status` 和 `DELETE` 的语义必须分开：

```text
online / offline / disabled / draft = 项目仍然存在，只是状态变化，写 UPSERT。
DELETE = 项目从目录中移除，Manager 标记 is_deleted = 1。
```

也就是说，小程序下线或禁用时不要发 `DELETE`，应该发 `UPSERT`，并把 `status` 更新为 `offline` 或 `disabled`。

敏感字段约束：

```text
appToken / sourcePath / buildCmd 可以同步到 Manager 供内部管理使用，但普通前端列表接口默认不返回。
appToken 不允许作为必填字段，保存和展示时需要按权限控制，必要时脱敏。
```

跳转地址约束：

```text
entryUrl 不属于子系统接口范畴，子系统不需要返回此字段。
Manager 在 /api/projects 出参时，按 source 从 factory-sources.yml 的 entry-url 配置中读取并拼到结果里。
新增子系统时，只需在 factory-sources.yml 加一行 entry-url 即可，无需做数据迁移。

video_miniapp -> http://172.17.7.183:8080/miniapp/autocreate/video/apps
drama_miniapp / comic_miniapp -> http://172.17.7.183:5177/apps
novel_miniapp -> http://172.17.7.183:8080/miniapp/novel/home
h5 -> http://172.17.3.118/h5novelwebconfig/
```

### 5.3 增量接口

用于日常同步。

```http
GET /api/open/apps/updates?afterChangeId=10000&limit=500
```

含义：

- `afterChangeId`: FunMiniFactoryManager 上次成功处理到的变更 ID。
- `limit`: 本次最多返回多少条变更。

返回：

```json
{
  "source": "novel_miniapp",
  "hasMore": false,
  "nextChangeId": 10003,
  "minChangeId": 1,
  "maxChangeId": 10003,
  "items": [
    {
      "changeId": 10001,
      "operation": "UPSERT",
      "appId": "101",
      "changedAt": "2026-05-29T10:20:00+08:00",
      "data": {
        "appId": "101",
        "appName": "番茄小说 Pro",
        "category": "novel",
        "platform": "wechat",
        "appToken": "",
        "version": "1.0.4",
        "status": "online",
        "owner": "lijj",
        "sourcePath": "git@gitlab.com:miniapp/novel-app.git",
        "buildCmd": "npm install && npm run build",
        "artifactDir": "dist",
        "createTime": "2026-05-29T10:00:00+08:00",
        "updateTime": "2026-05-29T10:20:00+08:00"
      }
    },
    {
      "changeId": 10002,
      "operation": "DELETE",
      "appId": "102",
      "changedAt": "2026-05-29T10:21:00+08:00",
      "data": null
    }
  ]
}
```

增量接口查询子系统的变更流水表，而不是直接查当前项目表。

分页规则：

```text
items 必须按 changeId 升序返回。
nextChangeId = 本页最后一条 item.changeId。
hasMore = true 时，Manager 下一页请求 afterChangeId = nextChangeId。
```

建议 SQL：

```sql
select *
from app_change_log
where id > :afterChangeId
order by id asc
limit :limit;
```

## 6. 子系统变更流水表

每个子系统在自己的库里增加一张项目变更流水表，作为 `/api/open/apps/updates` 接口的数据源。

> 这张表归属于子系统，由各子系统团队负责建表和写入，**不在 FunMiniFactoryManager 的中央库里**。

建议表名：

```text
app_change_log
```

建议字段：

```sql
create table app_change_log (
    id bigint primary key auto_increment,            -- 自增主键，对外暴露为 changeId
    app_id varchar(64) not null,                     -- 变更涉及的项目 ID
    operation varchar(20) not null,                  -- UPSERT / DELETE
    changed_at datetime not null,                    -- 业务侧实际变更时间
    payload json null,                               -- UPSERT 保存完整项目快照，DELETE 可为空
    created_at datetime not null default current_timestamp,
    index idx_app_id (app_id),
    index idx_changed_at (changed_at)
);
```

`operation` 使用：

```text
UPSERT
DELETE
```

新增项目时：

```text
业务项目表 insert
app_change_log insert UPSERT
```

修改项目时：

```text
业务项目表 update
app_change_log insert UPSERT
```

删除或从目录移除项目时：

```text
业务项目表 delete / update deleted_flag
app_change_log insert DELETE
```

下线、禁用、草稿等状态变化不算删除：

```text
业务项目表 update status
app_change_log insert UPSERT，payload 里 status = offline / disabled / draft
```

注意：

- 不是更新原来的 change_log，而是新增一条变更记录。
- 业务表变更和 change_log 写入必须在同一个事务内。
- UPSERT 事件必须保存完整 `payload` 快照，避免项目后续被删除时无法还原该条变更的数据。
- DELETE 事件 `payload` 可以为空，但必须保留 `app_id`。
- change_log 不需要永久保存，可以保留 30 天或 90 天。

## 7. FunMiniFactoryManager 数据表

> 以下两张表都在 FunMiniFactoryManager 的中央库里，由 Factory 团队负责。

### 7.1 项目目录表 factory_app

汇聚所有子系统的项目，**前端列表/统计直接读这张**。

表名：

```text
factory_app
```

建表 SQL：

```sql
create table factory_app (
    id                  bigint        primary key auto_increment,
    source              varchar(64)   not null,           -- 数据来源，例如 novel_miniapp / drama_miniapp / h5
    app_id              varchar(64)   not null,           -- 子系统内项目 ID，对应接口 appId
    app_name            varchar(255)  not null,           -- 项目名
    category            varchar(32)   not null,           -- 业务分类 novel / drama / comic / video / h5_novel / h5_yingshi / h5_fx / h5_lx
    platform            varchar(32)   not null,           -- 投放平台 wechat / douyin / kuaishou / web
    -- 注意：entryUrl 不入库，由 Manager 在出参时按 source 从 factory-sources.yml 动态拼上去
    app_token           varchar(255)           default '',-- 项目 token / appKey
    version             varchar(64)            default '',-- 当前版本号
    status              varchar(32)            default '',-- online / offline / disabled / draft；子系统无状态控制时允许为空
    owner               varchar(64)            default '',-- 项目负责人
    source_path         varchar(512)           default '',-- 源码地址，对应接口 sourcePath
    build_cmd           varchar(512)           default '',-- 构建命令，对应接口 buildCmd
    artifact_dir        varchar(255)           default '',-- 构建产物目录，对应接口 artifactDir
    source_create_time  datetime      not null,           -- 项目在子系统的创建时间
    source_update_time  datetime      not null,           -- 项目在子系统的最近更新时间
    last_change_id      bigint                 default 0, -- 最后一次同步用到的 changeId
    is_deleted          tinyint(1)    not null default 0, -- 软删除标记
    deleted_at          datetime      null,               -- 软删除时间
    last_seen_at        datetime      null,               -- 最近一次全量校准看见该项目的时间
    missing_count       int           not null default 0, -- 连续全量校准缺失次数
    created_at          datetime      not null default current_timestamp,
    updated_at          datetime      not null default current_timestamp on update current_timestamp,
    unique key uk_source_app (source, app_id),
    key idx_category (category),
    key idx_platform (platform),
    key idx_status (status),
    key idx_source_update_time (source_update_time)
);
```

写入规则：

```text
UPSERT 事件 -> event.changeId > factory_app.last_change_id 时才 insert / update
DELETE 事件 -> event.changeId > factory_app.last_change_id 时才 soft delete
event.changeId <= factory_app.last_change_id -> 跳过
```

原因：

```text
同步任务失败重试、手动同步和定时同步重叠时，可能重复处理旧事件。
所有写入必须以 changeId 为准，禁止旧事件覆盖新状态。
```

UPSERT 更新时必须同时恢复软删除标记：

```text
is_deleted = 0
deleted_at = null
last_change_id = event.changeId
```

DELETE 更新时：

```text
is_deleted = 1
deleted_at = now()
last_change_id = event.changeId
```

列表页默认过滤：

```sql
where is_deleted = 0
```

### 7.2 同步水位表 factory_app_sync

每个 source 一行，记录 Manager 同步到哪了。

表名：

```text
factory_app_sync
```

建表 SQL：

```sql
create table factory_app_sync (
    id                       bigint        primary key auto_increment,
    source                   varchar(64)   not null,           -- 数据来源
    last_success_change_id   bigint        not null default 0, -- 上次成功处理到的 changeId
    last_success_time        datetime      null,               -- 上次成功完成时间
    last_attempt_time        datetime      null,               -- 上次尝试时间
    last_status              varchar(32)            default '',-- success / failed / running
    last_error               varchar(1024)          default '',-- 上次失败原因
    created_at               datetime      not null default current_timestamp,
    updated_at               datetime      not null default current_timestamp on update current_timestamp,
    unique key uk_source (source)
);
```

示例数据：

```text
source              last_success_change_id    last_status    last_success_time
novel_miniapp       10086                     success        2026-05-29 14:20:00
drama_miniapp       8801                      success        2026-05-29 14:19:55
comic_miniapp       5532                      success        2026-05-29 14:20:01
video_miniapp       12099                     failed         2026-05-29 14:15:00
h5                  3021                      success        2026-05-29 14:20:03
```

## 8. 同步流程

### 8.1 首次同步

首次同步使用全量接口：

```text
1. 请求子系统 /api/open/apps/all。
2. 第一页响应里读取 snapshotChangeId，并在后续分页中保持同一个快照水位。
3. 写入 factory_app。
4. 分页拉取全部项目并全部写入成功。
5. 更新 factory_app_sync.last_success_change_id = snapshotChangeId。
```

首次全量同步不能使用同步结束时的最新 `maxChangeId` 推进水位，只能使用全量开始时固定下来的 `snapshotChangeId`。否则全量过程中产生的新变更可能既没被全量扫到，又被后续增量跳过。

首次同步后，后续走增量接口。

### 8.2 日常增量同步

流程：

```text
1. 读取 factory_app_sync.last_success_change_id。
2. 请求子系统 /api/open/apps/updates?afterChangeId=xxx&limit=500。
3. 按 changeId 从小到大处理。
4. UPSERT 写 factory_app。
5. DELETE 标记 is_deleted=true。
6. 如果 hasMore=true，使用 afterChangeId = response.nextChangeId 继续拉下一页。
7. 全部成功后，更新 last_success_change_id。
```

关键规则：

```text
只有 factory_app 写成功后，才能推进 last_success_change_id。
```

如果中途失败：

```text
不更新 last_success_change_id
下次从旧水位重新拉
```

因为 factory_app 使用 `source + app_id` 做唯一定位，并且所有写入都要求 `event.changeId > last_change_id`，重复处理不会出问题。

### 8.3 全量校准

建议每天凌晨执行一次全量校准：

```text
1. 拉取子系统全量项目。
2. 和 factory_app 中该 source 的项目集合对比。
3. 子系统存在，本地不存在 -> 新增，last_seen_at = now(), missing_count = 0。
4. 子系统存在，本地存在 -> 更新，last_seen_at = now(), missing_count = 0。
5. 本地存在，子系统不存在 -> missing_count += 1。
6. missing_count 连续达到阈值后，再标记 is_deleted = 1。
```

不要因为一次全量没返回就物理删除或软删除项目。

推荐阈值：

```text
missing_count >= 3 -> is_deleted = 1
```

原因是一次全量接口异常、分页漏数据、临时过滤条件变化，都可能导致某些项目没有返回。连续多次缺失再删除，可以降低误删风险。

## 9. FunMiniFactoryManager 什么时候拉

推荐触发时机：

```text
1. 定时拉。
2. 启动后异步拉。
3. 手动拉。
4. 收到子系统通知后拉，可选。
```

### 9.1 定时拉

主流程。

建议：

```text
每 1 到 5 分钟执行一次增量同步。
```

项目变化低频时，每 5 分钟即可。

### 9.2 启动后异步拉

服务启动完成后，后台异步拉一次。

注意：

- 不阻塞服务启动。
- 子系统慢或失败不能导致 Manager 启动失败。
- 列表页先展示 MySQL 里上一次同步的数据。

### 9.3 手动拉

管理后台提供按钮：

```text
立即同步全部
同步小说小程序
同步 H5
```

接口示例：

```http
POST /api/project-sync/run
POST /api/project-sync/run/{source}
GET  /api/project-sync/tasks/{taskId}
```

手动同步必须异步执行：

```text
POST /api/project-sync/run 或 /run/{source} 只负责创建同步任务并立刻返回 taskId。
前端用 GET /api/project-sync/tasks/{taskId} 查询 running / success / failed。
任务成功后重新请求 projects 和 stats。
```

### 9.4 通知触发拉，可选

子系统变化后可以通知：

```http
POST /api/project-sync/notify
```

请求：

```json
{
  "source": "novel_miniapp",
  "latestChangeId": 10088
}
```

FunMiniFactoryManager 收到后仍然主动请求子系统增量接口。

通知只作为触发信号，不作为数据来源。

## 10. 前端页面刷新机制

同步完成后，页面不会天然自动刷新。

推荐第一版使用轻量轮询。

页面加载：

```text
GET /api/projects
GET /api/projects/stats
GET /api/project-sync/status
```

页面运行中：

```text
每 30 秒请求一次 /api/project-sync/status
如果 lastSyncedAt 变化，重新请求 projects 和 stats
```

手动同步时：

```text
1. 前端 POST /api/project-sync/run。
2. 后端创建异步同步任务并立刻返回 taskId。
3. 前端每 2 秒请求 GET /api/project-sync/tasks/{taskId} 查询任务状态。
4. 任务成功后重新请求列表和统计。
```

第一版不需要 WebSocket。

小程序工厂列表当前只展示这些列：

```text
平台
小程序名称
APPID
版本号
类别
操作
```

对应 `/api/projects` 默认返回字段：

```text
platform
appId
appName
category
version
entryUrl
```

`entryUrl` 不单独展示成一列，只给「进入子系统」操作使用。该字段不在 `factory_app` 表里存储，由 Manager 在出参时按 `source` 从 `factory-sources.yml` 的 `entry-url` 配置读取拼上去。

以下字段属于内部管理字段，默认不在普通列表接口返回：

```text
appToken
sourcePath
buildCmd
artifactDir
```

如果管理页确实需要查看这些字段，必须单独做权限控制；`appToken` 返回时默认脱敏。

## 11. Redis 使用建议

Redis 可以用，但不要作为核心数据源。

MySQL 保存：

```text
factory_app
factory_app_sync
子系统 app_change_log
```

Redis 只做：

```text
列表缓存
统计缓存
同步任务分布式锁
同步进度临时状态
失败冷却
```

第一版可以不使用 Redis，但仍然需要做 source 级互斥。

即使只有一个 Manager 实例，也可能出现这些并发：

```text
定时同步正在跑
启动后异步同步正在跑
用户又点击手动同步
```

因此同一个 source 同一时间只能有一个同步任务运行。单实例第一版可以用 JVM 内存锁；多实例时换 Redis 分布式锁。

如果 FunMiniFactoryManager 部署多实例，建议用 Redis 锁避免重复同步：

```text
factory:sync:lock:novel_miniapp
factory:sync:lock:h5
```

## 12. change_log 清理策略

change_log 会越来越大，但增量同步每次只查：

```sql
where id > :last_success_change_id
order by id asc
limit 500
```

只要 `id` 是主键，查询不会从头扫描整张表。

change_log 不需要永久保存。

建议：

```text
保留 90 天
```

如果 FunMiniFactoryManager 的 `afterChangeId` 早于子系统最早保留的 changeId，说明增量断档。

子系统返回：

```json
{
  "code": "CHANGE_LOG_EXPIRED",
  "message": "change log expired, please run full sync",
  "minChangeId": 5000,
  "maxChangeId": 9000
}
```

FunMiniFactoryManager 收到后执行全量同步。

## 13. 接口清单

### 13.1 子系统接口

子系统实现，FunMiniFactoryManager 调用：

```http
GET /api/open/apps/all
GET /api/open/apps/updates?afterChangeId=xxx&limit=500
```

### 13.2 FunMiniFactoryManager 对前端接口

FunMiniFactoryManager 实现，FunMiniFactory 前端调用：

```http
GET  /api/projects                      项目列表
GET  /api/projects/stats                统计数据
GET  /api/project-sync/status           同步状态
POST /api/project-sync/run              手动异步触发全部同步，返回 taskId
POST /api/project-sync/run/{source}     手动异步触发单个 source 同步，返回 taskId
GET  /api/project-sync/tasks/{taskId}   查询同步任务进度
```

可选：

```http
POST /api/project-sync/notify           子系统通知触发
```

## 14. 推荐落地顺序

第一阶段：

```text
1. FunMiniFactoryManager 建 factory_app。
2. 对接各子系统已提供的 /api/open/apps/all。
3. 定时全量同步到 MySQL。
4. 前端列表和统计只查 Manager。
```

第二阶段：

```text
1. 对接各子系统已提供的 /api/open/apps/updates。
2. Manager 增加 factory_app_sync。
3. 定时增量同步。
4. 每天凌晨全量校准。
```

第三阶段：

```text
1. 管理后台增加异步手动同步和任务进度展示。
2. 前端增加同步状态和轮询刷新。
3. 多实例时增加 Redis 分布式锁。
4. 需要更实时再增加 notify 触发。
```

## 15. 最终结论

推荐方案：

```text
MySQL 保存项目目录
Pull 拉模式同步子系统
首次全量，后续增量
增量基于 change_log + afterChangeId
定时同步为主，手动同步为辅
页面读取本地库，通过轮询感知同步完成
Redis 只做缓存和锁，不做事实数据源
```

这个方案稳定、易排查、对子系统耦合低，适合 FunMiniFactory 作为公司内部项目总览系统长期演进。
