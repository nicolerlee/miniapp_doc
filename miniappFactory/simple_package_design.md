# 简化版代码包上传-构建-发布方案

> 范围：MVP，单表结构，`platform` 放开四个平台：weixin / douyin / kuaishou / baidu。

---

## 1. 背景与目标

### 现状
- 构建：工作目录与产物目录通过 `application.properties` 的 `build.workPath` / `build.buildedPath` 全局固定，构建命令从数据库表拼接。
- 发布：产物目录同样走 `build.buildedPath`，其余发布参数（appId、token 等）从数据库表读取。
- 只能管理一个固定的代码库，想切换代码库必须改配置或重启。

### 目标
提供一条**独立的轻量链路**，不动现有链路：
1. 通过上传 zip 注册一个"代码包"，每个包有独立的源码目录。
2. 每个包可以独立构建、预览、发布，互不干扰。
3. 同时存在多个包，支持列表、详情、删除管理。

---

## 2. 整体方案

### 2.1 核心思路

把"包"当成**可插拔的工作目录**：
- 上传 zip → 解压到 `<packageBase>/<packageId>/source/`
- 构建时以 `source/` 为 workDir 执行 `npm run build:xxx`
- 产物天然落在 `source/dist/<mpDir>/`
- 发布时 `projectPath` 自动等于 `source/dist/<mpDir>`

`packageId` 是**系统唯一键**（调用方自行定义并传入）：一个 packageId 对应且仅对应一个包。上传同 packageId 的 zip 会触发覆盖。
`appId` 是平台分配的应用 ID，**允许为空**（用户可以先上传代码包，后续再补填 appId）。同一个 appId 可以关联多个包（例如同一个小程序的不同版本分支）。
对外接口、批量请求、任务参数、数据库主键统一使用 `packageId`。
发布所需 `appId` 与上传 token 作为包的字段存储，发布时必须非空。四个平台都放开，后端按 `platform` 映射构建产物目录、二维码文件名与发布 token 字段。

### 2.2 目录布局

```
<packageBase>/                          # 新增配置 miniapp.packageBase
  └─ <packageId>/
       ├─ origin.zip                    # 原始 zip（可保留用于排查，或解压完即删）
       └─ source/                       # 解压后源码（= 构建 workDir）
            ├─ node_modules/            # 覆盖时保留，不随 zip 替换
            └─ dist/<mpDir>/            # 构建产物（= 发布 projectPath）
                 └─ <qrcode>.png        # 发布完成后写入
```

`<mpDir>` 由平台决定：`weixin → mp-weixin`、`douyin → mp-toutiao`、`kuaishou → mp-kuaishou`、`baidu → mp-baidu`。二维码文件名亦随平台：`wx_qrcode.png` / `tt_qrcode.png` / `ks_qrcode.png` / `bd_qrcode.png`。

### 2.3 覆盖策略（同 packageId 再次上传）

判定：上传时后端用 body 里的 `packageId` 去表里查，命中则进入覆盖流程；未命中则新建。

覆盖过程（在 `PackageUploadTaskProcessor` 里按顺序执行）：

```
1. 检查该包是否有正在运行的构建/发布任务 → 有则返回 EDIT_IN_PROGRESS
2. 把 source/node_modules 原子 rename 到 <packageId>/.node_modules.bak
3. rm -rf source/                     # 旧 dist 一并清除
4. 解压新 zip 到 source/
5. 把 .node_modules.bak 原子 rename 回 source/node_modules
6. 比较 package-lock.json 哈希（可选）
   - 变了 → 响应 result 里 `packageLockChanged: true`，调用方据此决定是否调 install
7. 更新表：original_file_name、update_time
   （name / appId / token / buildCmd 仅当 body 里传了才覆盖）
```

关键点：
- **保留 `node_modules`**：跳过 `npm install`，节省几分钟
- **不保留 `dist`**：代码都换了，旧产物不再可信，主动清空；用户下次构建会重新产出
- **用 `Files.move` 而非拷贝**：同一分区内是原子 rename，`node_modules` 有几万小文件也是瞬间完成
- 新 zip 如果自带了 `dist/` 或 `node_modules/`，也会被上述流程以备份回填的方式覆盖（node_modules 用旧的回填，dist 直接丢弃）

### 2.4 上传 zip 文件规范

#### 2.4.1 文件本身

| 项目 | 要求 | 说明 |
|------|------|------|
| 文件类型 | `.zip` | 仅支持 zip，不接受 tar.gz / rar |
| MIME | `application/zip` / `application/x-zip-compressed` / `application/octet-stream` | 按扩展名为主，MIME 做辅助校验 |
| 文件名编码 | UTF-8 | zip 内部条目名不得含非 UTF-8 字符，否则部分条目会解压失败（Java `ZipInputStream` 默认 UTF-8） |
| 文件名长度 | ≤ 255 字符 | `original_file_name` 字段限制 |
| 单文件大小 | ≤ 200 MB | `spring.servlet.multipart.max-file-size` |
| 解压后总大小 | ≤ 2 GB | `ZipExtractUtil` 内部阈值，超过视作 zip 炸弹拒绝 |
| 解压后文件数 | ≤ 50000 | 同上 |

#### 2.4.2 zip 内部目录结构

**根必须是项目根**（`package.json` 必须在 zip 的顶层），不允许把项目塞在一个子目录里。

✅ 正确：
```
xingchen.zip
├── package.json
├── package-lock.json       (推荐一并带上)
├── src/
├── vite.config.js
└── ...
```

❌ 错误（多了一层外壳）：
```
xingchen.zip
└── xingchen/               ← 这种会被拒绝
     ├── package.json
     └── ...
```

校验规则：解压后 `source/package.json` 必须存在，否则 upload 任务失败，返回 `INVALID_PACKAGE`。

#### 2.4.3 package.json 要求

| 字段 | 要求 | 说明 |
|------|------|------|
| `scripts` | 必须包含至少一个 `build:*` 脚本 | 否则无法构建 |
| 当前平台对应构建脚本 | 推荐 | 例如微信 `build:mp-weixin`、抖音 `build:mp-toutiao`；若脚本名不同，用户需在 PATCH 里改 `buildCmd` |

上传时后端会解析 `package.json` 的 `scripts`，抽出所有 `build:*` 键返回在 `availableBuildScripts`，前端列出来供用户选。

#### 2.4.4 推荐（但不强制）的内容

- **带上 `package-lock.json`**：决定 node_modules 是否需要重装的依据，没有则不做依赖变更检测
- **不要带上 `node_modules/`**：zip 里的 node_modules 会被丢弃（覆盖流程里会被旧的回填，首次上传也会被忽略），只会徒增体积和上传时间
- **不要带上 `dist/`**：覆盖时会被清空；首次上传会被清空
- **不要带上 `.git/`**：徒增体积，无业务价值
- **不要带上 `.env`、密钥、证书**：安全风险

前端上传前可以给用户一个提示："zip 请以项目根为起点，不要包含 `node_modules`、`dist`、`.git`"。

#### 2.4.5 文件名建议（软规则）

不做强校验，但给用户推荐命名模板，便于排查：

```
<appName>-<version>-<yyyyMMdd>.zip
例：xingchen-4.3.2-20260508.zip
```

上传表单可以让用户填 `name` 字段做展示名；不填则取 zip 文件名（去掉 `.zip` 后缀）作为 `name` 默认值。

#### 2.4.6 上传前端校验清单

前端在 `file` 选择后、调 `/upload` 前做以下本地检查：

- 扩展名是否为 `.zip`
- 文件大小是否超过 200 MB
- `packageId` 是否已填写且非空
- `appId`（如果填了）是否符合格式（多平台先放宽为 `^[a-zA-Z0-9_]{8,64}$`）

不通过直接在前端报错，不要发请求。

#### 2.4.7 完整校验流程

上传前后分四个阶段校验，任何一步失败都要清理已落盘的临时文件：

**阶段 A：前端上传前（本地）**

1. 扩展名必须是 `.zip`
2. 文件大小 ≤ 200 MB
3. 用 Web Crypto API 计算文件 SHA-256

```js
const buf = await file.arrayBuffer();
const hash = await crypto.subtle.digest('SHA-256', buf);
const sha256 = Array.from(new Uint8Array(hash))
  .map(b => b.toString(16).padStart(2, '0')).join('');
```

**阶段 B：后端接收时（同步）**

4. zip 文件头魔数必须是 `PK\x03\x04`，否则立即拒绝（`INVALID_PACKAGE`）
5. 落盘后重算 SHA-256，与 body 中的 `sha256` 字段比对，不一致返回 `CHECKSUM_MISMATCH`

**阶段 B 优化（幂等上传）**：同一 `packageId` 下 SHA-256 命中当前包时，跳过解压，直接返回该 `packageId` 和 `mode: "no_change"`。

**阶段 C：解压时（异步任务中）**

6. 每个 zip 条目路径做 `Path.resolve` 后必须 `startsWith` 目标根（防 `../` 穿越）
7. 解压后总字节数 ≤ 2 GB
8. 解压后文件总数 ≤ 50000

**阶段 D：解压后（异步任务中）**

9. `source/package.json` 必须存在
10. `package.json` 必须能被 `JSON.parse`，且 `scripts` 字段是对象
11. `scripts` 中至少包含一个 `build:*` 脚本，否则 `INVALID_PACKAGE: no build scripts found`
12. `package-lock.json` 不存在时给 warning（不拒绝）
13. 扫描敏感文件（`.env`、`*.key`、`*.pem`、`*.p12`），发现即给 warning
14. （可选）按平台读取 `src/manifest.json` 中对应平台的 appid 作为 appId 兜底来源（见 11.1）

检查结果通过 task status 接口返回，见 `4.1` 的响应示例。

---

### 2.5 链路

```
┌────────┐  upload     ┌──────────┐  build   ┌───────────┐  publish  ┌──────────┐
│ 前端   │ ─────────►  │ 新 API    │ ───────► │ 复用队列   │ ────────► │ 复用队列  │
│ (浏览) │  zip        │ (本文档)  │  taskId  │ Build...  │  taskId   │ Publish..│
└────────┘             └──────────┘          └───────────┘           └──────────┘
                                  ▲                                        │
                                  │ 轮询 /api/agent/task/{taskId}/status   │
                                  └────────────────────────────────────────┘
```

所有异步任务统一走现有 `TaskQueueManager` + `/api/agent/task/{taskId}/status` 轮询。

---

## 3. 数据模型

**单表，无外键。**

```sql
CREATE TABLE miniapp_package (
  package_id          VARCHAR(64)  NOT NULL PRIMARY KEY, -- 调用方自行定义的包标识，唯一主键
  app_id              VARCHAR(64)  DEFAULT NULL,         -- 平台分配的 AppID，允许为空（发布时必填）
  name                VARCHAR(128) NOT NULL,         -- 包名（用户自定义）
  platform            VARCHAR(16)  NOT NULL,         -- weixin / douyin / kuaishou / baidu
  original_file_name  VARCHAR(255),                  -- 上传的原 zip 文件名
  source_path         VARCHAR(512) NOT NULL,         -- 解压后源码绝对路径

  app_token           VARCHAR(2048) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL, -- 平台上传凭证（微信为 RSA 私钥；其余为 token 字符串）

  build_cmd           VARCHAR(255),                  -- 构建命令
  version             VARCHAR(32),                   -- 版本号（发布时使用，如 1.0.0）

  owner_id            BIGINT NOT NULL,               -- 创建人，仅用于审计，本期不做包级权限过滤
  create_time         DATETIME DEFAULT CURRENT_TIMESTAMP,
  update_time         DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

说明：
- **`package_id` 是唯一主键**：接口入参、返回值、任务参数、前端路由、数据库主键均使用 `packageId`。由调用方自行定义并在上传时传入。
- **`app_id` 允许为空**：用户可以先上传代码包，后续通过 PATCH 补填 appId。发布时 appId 必须非空，否则返回 `PUBLISH_APP_ID_MISSING`。
- **不存 status**：包的"构建中/成功/失败"由任务表动态计算，避免双写不一致。
- **token 明文存储**：内部工具场景，暂不加密，直接 `VARCHAR(2048)`。
- **覆盖逻辑**：上传时 `packageId` 命中已有记录则触发覆盖，未命中则新建。
- **`owner_id` 仅用于审计**：本期不做包级权限校验，不按 owner_id 过滤列表、详情、构建、发布、修改、删除。
- **`platform` 字段**: weixin, douyin, kuaishou, baidu
- `source_path` 存绝对路径，便于构建/发布直接使用，不依赖其他配置。

---

## 4. 接口清单

统一前缀 `/api/miniapp-pkg`，一共新增 **15 个接口**（单包 11 个 + 批量 4 个）。

### 4.1 异步接口（返回 taskId，前端轮询状态）

| # | 方法 | 路径 | 作用 |
|---|------|------|------|
| 1 | POST | `/upload` | multipart：`file` + `sha256` + `platform` + `packageId`(必填) + `appId`(可选) + `name/token/buildCmd/version`(可选)，`packageId` 命中则覆盖，否则新建 |
| 2 | POST | `/{packageId}/install` | 安装依赖（`npm ci` 或 `npm install`），幂等 |
| 3 | POST | `/{packageId}/build` | body `{cmd}`，以该包 `source_path` 为 workDir 执行构建（前置检查：node_modules 必须存在） |
| 4 | POST | `/{packageId}/publish` | body 见下，发布小程序 |

**upload 请求**
```
POST /api/miniapp-pkg/upload
Content-Type: multipart/form-data

file:      <xxx.zip>
sha256:    a3f5c8...              # 必填，前端用 Web Crypto 算的文件 SHA-256
platform:  weixin                 # 必填：weixin / douyin / kuaishou / baidu
packageId: a1b2c3d4e5f6...       # 必填，调用方自行定义的包标识，命中已有则覆盖
appId:     wx1234567890abcdef    # 可选，允许为空
name:      xingchen-test         # 可选，首次未传取 zip 文件名
token:     -----BEGIN ...        # 可选
buildCmd:  npm run build:mp-weixin # 可选
```

**upload 响应**
```json
{ "code": 200, "data": { "taskId": "abc123" } }
```
完成后 `GET /api/agent/task/{taskId}/status` 的 `data.result` 返回：
```json
{
  "packageId": "a1b2c3d4e5f6789012345678abcdef00",
  "appId": "wx1234567890abcdef",
  "mode": "create",                    // create / overwrite / no_change
  "sha256": "a3f5c8...",
  "availableBuildScripts": ["build:mp-weixin"],
  "packageLockChanged": false,         // overwrite 时才有意义
  "warnings": [
    "missing package-lock.json, dependency lock not available",
    "found sensitive file: src/.env"
  ]
}
```

失败示例：
```json
{
  "taskId": "abc123",
  "status": "FAILED",
  "errorCode": "INVALID_PACKAGE",
  "errorMessage": "package.json not found in zip root"
}
```

**upload 错误码**

| errorCode | HTTP code | 说明 |
|-----------|-----------|------|
| `CHECKSUM_MISMATCH` | 400 | body 的 sha256 与落盘后重算不一致 |
| `INVALID_PACKAGE` | 400 | zip 结构或项目结构不合法（非 zip / 路径穿越 / 缺 package.json / 无 build 脚本） |
| `EDIT_IN_PROGRESS` | 409 | 命中已有包但该包正在构建/发布 |
| `INVALID_PARAMS` | 400 | 缺 file / packageId / platform / sha256 等必填字段 |
| `UNSUPPORTED_PLATFORM` | 400 | `platform` 不在 `weixin/douyin/kuaishou/baidu` 枚举内 |

**build 请求**
```json
{ "cmd": "npm run build:mp-weixin" }
```

**build 前置检查**：`node_modules` 不存在 → 返回 400：

```json
{
  "code": 400,
  "errorCode": "DEPENDENCIES_NOT_INSTALLED",
  "errorMessage": "node_modules 不存在，请先安装依赖",
  "actionHint": {
    "action": "install",
    "endpoint": "POST /api/miniapp-pkg/{packageId}/install",
    "then": "install 完成后重新调用 build"
  }
}
```

Agent 收到 `DEPENDENCIES_NOT_INSTALLED` 后自动调 install → 轮询完成 → 重新调 build。

**install 请求**

```
POST /api/miniapp-pkg/{packageId}/install
```

无 body。异步，返回 `{taskId}`。

行为（幂等）：
- `node_modules` 不存在 → 有 lock 跑 `npm ci`，无 lock 跑 `npm install`
- `node_modules` 存在但 lock 哈希变了 → 跑 `npm ci`
- `node_modules` 存在且 lock 没变 → 跳过，直接返回成功

**install 响应**

正常入队：
```json
{ "code": 200, "data": { "taskId": "abc123" } }
```

依赖已最新（跳过，同步返回）：
```json
{ "code": 200, "data": { "skipped": true, "reason": "dependencies up to date" } }
```

install 任务完成后 `GET /api/agent/task/{taskId}/status` 的 `data.result`：
```json
{
  "command": "npm ci",
  "duration": "44s"
}
```

**install 内部逻辑**：
```
1. currentHash = sha256(source/package-lock.json)，不存在则 null
2. lastHash = read(<packageDir>/.last_lock_hash)，不存在则 null
3. needInstall = (node_modules 不存在) || (currentHash != lastHash)
4. if !needInstall → 返回 skipped
5. 有 lock → npm ci；无 lock → npm install
6. 成功 → 写 currentHash 到 .last_lock_hash
7. 失败 → 任务失败
```

**publish 请求**
```json
{
  "version": "1.0.0",
  "log": "修复若干问题",
  "publishMode": "preview"
}
```
`version` 可选：不传则用包表里的 `version` 字段；传了则用本次传入值并回写到包表。两者都为空 → 返回 `INVALID_PARAMS`。
后端自动装配（按包表的 `platform` 取值）：

| platform | platformCode | projectPath 后缀 | token 字段（发给底层） |
|----------|--------------|-----------------|-----------------------|
| weixin   | `mp-weixin`   | `/dist/mp-weixin`   | `weixinAppToken` |
| douyin   | `mp-toutiao`  | `/dist/mp-toutiao`  | `douyinAppToken` |
| kuaishou | `mp-kuaishou` | `/dist/mp-kuaishou` | `kuaishouAppToken` |
| baidu    | `mp-baidu`    | `/dist/mp-baidu`    | `baiduAppToken`  |

四个平台均在 controller/service 层放开；不因当前阶段只测过某个平台而提前拒绝。实际构建/发布是否成功由包内构建脚本、CLI 环境、平台 token 与底层发布工具决定。

appId 或 token 为空 → 返回 `PUBLISH_TOKEN_MISSING`，引导用户去 `PATCH` 接口补齐。

### 4.2 同步接口

| # | 方法 | 路径 | 作用 |
|---|------|------|------|
| 5 | GET    | `/list` | 所有包列表 |
| 6 | GET    | `/{packageId}` | 包详情
| 7 | PATCH  | `/{packageId}` | 修改 name / appId / token / buildCmd / version | platform
| 8 | DELETE | `/{packageId}` | 删包（同时递归删 source 目录） |
| 9 | GET    | `/{packageId}/artifacts` | 该包已有构建产物信息 |
| 10 | POST  | `/{packageId}/build/stop` | body `{taskId}`，停止构建 |
| 11 | POST  | `/{packageId}/publish/stop/{taskId}` | 停止发布 |

**PATCH 请求**
```json
{
  "name": "xingchen-test",               // 可选
  "appId": "wx1234567890abcdef",         // 可选；不传保持原值，传空串视为清空
  "token": "-----BEGIN PRIVATE KEY-----\n...", // 可选；不传保持原值，传空串视为清空
  "buildCmd": "npm run build:mp-weixin", // 可选
  "version": "1.0.1"                     // 可选
}
```

**/list token 返回规则**
- `/list` 返回全量包列表。
- 列表项保留 `token` 字段，但不返回原文。
- `app_token` 有值时，`token` 固定返回 `"***"`；`app_token` 为空时，`token` 返回 `null`。

> `packageId` 是包的唯一系统标识，**不允许通过 PATCH 修改**。
> `appId` **允许通过 PATCH 修改**（补填或更正平台 AppID）。
> `platform` 字段**不允许通过 PATCH 修改**：平台变了等价于换了个包，应当删包重新上传。

**权限口径**
- 本期不做包级权限校验，也不区分普通用户/管理员。
- `/list` 返回全量包列表，token 字段脱敏返回：有值为 `"***"`，无值为 `null`。
- `详情 / 构建 / 发布 / PATCH / DELETE / stop` 只校验 `packageId` 是否存在、请求参数是否合法；stop 额外校验 `taskId` 属于该 `packageId`，避免误停其它包任务。
- `owner_id` 只记录创建人，作为审计信息，不参与接口访问控制。

### 4.3 批量接口

为复用原"批量构建 / 批量发布"前端 UI 组件，新链路提供批量入口。底层仍然是 for 循环入队单包任务，共享同一个 `batchTaskId`。

#### 异步

| # | 方法 | 路径 | 作用 |
|---|------|------|------|
| 12 | POST | `/batch-build` | body `{packageIds, cmd?}`，逐包入队构建 |
| 13 | POST | `/batch-publish` | body `{packageIds, version, log, publishMode}`，逐包入队发布（token 从包表读） |

**batch-build 请求**
```json
{
  "packageIds": ["a1b2c3d4e5f6789012345678abcdef00", "b2c3d4e5f6789012345678abcdef0011"],
  "cmd": "npm run build:mp-weixin"    // 可选；不传则各包用自身 build_cmd
}
```

**batch-publish 请求**
```json
{
  "packageIds": ["a1b2c3d4e5f6789012345678abcdef00", "b2c3d4e5f6789012345678abcdef0011"],
  "version": "1.0.0",
  "log": "批量灰度",
  "publishMode": "preview"
}
```

**响应（两者一致）**
```json
{
  "code": 200,
  "data": {
    "batchTaskId": "uuid-batch",
    "taskIds": ["uuid-1", "uuid-2", "uuid-3"]
  }
}
```

前端订阅 WebSocket `/topic/batch-build-status/{batchTaskId}` 或 `/topic/batch-publish-status/{batchTaskId}` 看整批进度（与旧批量接口同 topic，前端组件完全复用）。

#### 同步

| # | 方法 | 路径 | 作用 |
|---|------|------|------|
| 14 | POST | `/batch-build/stop/{batchTaskId}` | 停止该批次下所有未完成构建 |
| 15 | POST | `/batch-publish/stop/{batchTaskId}` | 停止该批次下所有未完成发布 |

实现上复用 `taskQueueManager.stopBatchTask(batchTaskId)` 的同一套机制（和 `BatchBuildController.stopBatchBuild` 内部逻辑一致）。

### 4.4 接口实现说明

| 接口 | 底层调用 |
|------|---------|
| `/upload` | 新 `PackageUploadTaskProcessor`（zip 解压 + 新建/覆盖流程） |
| `/{packageId}/install` | 检测 lock 哈希 → 按需 `npm ci` / `npm install` → 写 `.last_lock_hash`；幂等，依赖已最新时同步返回 skipped |
| `/{packageId}/build` | 前置检查 node_modules 存在性 → 不存在返回 `DEPENDENCIES_NOT_INSTALLED`；通过后装配 taskParams 调 `taskQueueManager.addTaskToQueue`，执行层是 `NovelAppBuildUtil.buildNovelAppWithWorkPath` |
| `/{packageId}/publish` | 装配后调 `taskQueueManager.addTaskToQueue`，执行层复用现有 `NovelAppPublishUtil.publishNovelAppAsync`，完全不动 |
| `/{packageId}/build/stop` | 校验 taskId 属于该 packageId 后，转发 `taskQueueManager.stopTask(taskId)`（兜底 `novelAppBuildUtil.stopBuild(taskId)`） |
| `/{packageId}/publish/stop/{taskId}` | 校验 taskId 属于该 packageId 后，转发 `taskQueueManager.stopTask(taskId)`（兜底 `novelAppPublishUtil.stopPublish(taskId)`） |
| `/{packageId}/artifacts` | 纯文件系统读：按包的 `platform` 决定目录名，扫描 `source/dist/<mpDir>/`，返回存在性、最后修改时间、是否有二维码文件 |
| `/batch-build` | 同 `/build`，for 循环构造 `TaskQueueItem`，共享 `batchTaskId` 入队 |
| `/batch-publish` | 同 `/publish`，for 循环构造 `TaskQueueItem`，共享 `batchTaskId` 入队 |
| `/batch-build/stop/{batchTaskId}` | 复用 `taskQueueManager.stopBatchTask(batchTaskId)` |
| `/batch-publish/stop/{batchTaskId}` | 同上 |

两个 stop 接口**不直接让前端调 `/api/agent/...`**，而是自己包一层做 `taskId` 与 `packageId` 的关联校验，避免误停其它包的任务；这不是用户权限校验。

### 4.5 复用现有接口

| 用途 | 接口 |
|------|------|
| 异步任务状态 | `GET /api/agent/task/{taskId}/status` |
| 异步任务日志 | `GET /api/agent/task/{taskId}/logs` |
| 二维码下载   | `GET /api/agent/publish/qrcode/{taskId}` |
| 实时构建日志（单包） | WebSocket `/topic/build-logs/{taskId}` |
| 批量构建状态 | WebSocket `/topic/batch-build-status/{batchTaskId}` |
| 批量发布状态 | WebSocket `/topic/batch-publish-status/{batchTaskId}` |
| 批量发布日志 | WebSocket `/topic/batch-publish-logs/{batchTaskId}` |

### 4.6 Agent 调用流程

Agent（或任何自动化脚本）调用这些接口的标准流程：

```
1. POST /api/miniapp-pkg/upload       → taskId → 轮询 → 完成（packageId 由调用方传入）
2. POST /api/miniapp-pkg/{packageId}/install  → taskId（或 skipped）→ 轮询 → 完成
3. POST /api/miniapp-pkg/{packageId}/build    → taskId → 轮询 → 完成
4. POST /api/miniapp-pkg/{packageId}/publish  → taskId → 轮询 → 完成
5. GET  /api/agent/publish/qrcode/{taskId} → 二维码图片
```

**错误自愈**：
- build 返回 `DEPENDENCIES_NOT_INSTALLED` → Agent 自动调 install → 轮询完成 → 重试 build
- publish 返回 `PUBLISH_TOKEN_MISSING` → Agent 调 `PATCH /{packageId}` 补 token 和 appId → 重试 publish
- upload 返回 `EDIT_IN_PROGRESS` → Agent 等待一段时间后重试

**幂等安全**：
- install 可以无脑调，依赖已最新时秒回 `skipped: true`
- upload 同 packageId 同 sha256 时秒回 `mode: "no_change"`

**批量场景**：
```
1. 多次 POST /upload（各自带不同 packageId）
2. POST /batch-build { packageIds: [...] }  → batchTaskId → 轮询
3. POST /batch-publish { packageIds: [...] } → batchTaskId → 轮询
```

---

## 5. 任务队列类型

`TaskQueueItem.type` 新增 2 种，复用 2 种：

| type | 用途 | 处理器 |
|------|------|--------|
| `pkg-upload` | 新增：zip 落盘 + 解压 + 解析 `package.json` | 新增 `PackageUploadTaskProcessor` |
| `pkg-install` | 新增：`npm ci` / `npm install` + 写 `.last_lock_hash` | 新增 `PackageInstallTaskProcessor` |
| `build` | 复用：`taskParams` 里新增 `workPath` 字段，如果存在则用指定目录而非全局 `build.workPath` | `BuildTaskProcessor` 加分支 |
| `publish` | 复用：装配工作在 controller 层完成，taskParams 结构不变 | `PublishTaskProcessor` 无改动 |

---

## 6. 后端改动范围

### 6.1 新增文件

| 路径 | 作用 |
|------|------|
| `src/main/resources/db/migration/V20260508_01__create_miniapp_package.sql` | 建表 |
| `entity/MiniappPackage.java` | Entity |
| `mapper/MiniappPackageMapper.java` | MyBatis-Plus Mapper |
| `service/MiniappPackageService.java` + Impl | 业务层：CRUD、装配构建/发布任务 |
| `controller/MiniappPackageController.java` | 15 个接口（11 单包 + 4 批量） |
| `service/impl/PackageUploadTaskProcessor.java` | 处理 `pkg-upload` 类型任务 |
| `service/impl/PackageInstallTaskProcessor.java` | 处理 `pkg-install` 类型任务（npm ci / npm install） |
| `utils/ZipExtractUtil.java` | 安全解压（防 zip 炸弹 / 路径穿越） |

### 6.2 修改文件

| 路径 | 改动 |
|------|------|
| `utils/NovelAppBuildUtil.java` | 新增方法 `buildNovelAppWithWorkPath(String cmd, String taskId, String workPath, ...)`，把 workPath 从 `@Value` 注入改为参数传入。旧方法保留。 |
| `service/impl/BuildTaskProcessor.java`（或等价类） | 读取 `taskParams.workPath`，有值走新方法，无值走旧方法。 |
| `utils/TaskQueueManager.java`（如需） | 注册 `pkg-upload` 任务类型分发到 `PackageUploadTaskProcessor`。 |
| `entity/TaskQueueItem.java` | 如无 `result` 字段需补充，供 upload 完成后回写 `packageId`、`appId`、`mode`、`warnings` 等结果。 |
| `src/main/resources/application.properties` | 新增 `miniapp.packageBase`。 |
| `pom.xml` | 无新增依赖。 |

### 6.3 过渡期保留（不再演进）

> 旧链路进入"冻结"状态：保留运行，不再接受功能演进。等新链路稳定并切换完毕后，进入 Phase 3 统一清理（见第 12 节）。

- `NovelAppPublishUtil` / 发布处理器 / 二维码接口：完全复用。
- 现有 `/api/agent/build` `/api/agent/publish`：继续服务过渡期，不受影响。
- `/api/novel-build/*`、`/api/novel-publish/*` 批量接口：保留，不改造。
- `NovelAppBuildUtil` 旧方法（`buildNovelApp`、`buildNovelAppWithTaskId`、`@Value("${build.workPath}")`）：保留。
- `novel_app` 等旧表：零改动。

---

## 7. 前端改动范围（`miniappManager`）

新增一个"代码包管理"页面，独立路由，不影响现有页面。旧的"批量构建 / 批量打包 / 构建 / 打包"页面暂不动，等 Phase 2 统一切换到新 API（详见第 12 节）。

### 7.1 新增组件

| 组件 | 作用 |
|------|------|
| `views/MiniappPackage/PackageList.vue` | 包列表 + 上传入口 + 批量选择/构建/发布 |
| `views/MiniappPackage/PackageDetail.vue` | 包详情 + 构建/预览/发布按钮 |
| `components/miniappPackage/UploadDialog.vue` | 上传 zip 弹窗 |
| `components/miniappPackage/ConfigDialog.vue` | 配置 appId / token 的弹窗 |
| `components/miniappPackage/BuildLogPanel.vue` | 构建日志实时面板（WS 订阅） |
| `components/miniappPackage/BatchProgressPanel.vue` | 批量任务进度（订阅 `/topic/batch-build-status` / `/topic/batch-publish-status`），可直接从旧批量页面的组件复制后调整字段映射 |

### 7.2 新增 API 封装
`src/api/miniappPackage.js`，封装上面 15 个接口 + 7 个复用接口。

### 7.3 路由与菜单
新增一级菜单项"代码包"，路由 `/miniapp-package`。

---

## 8. 安全与健壮性清单

| 问题 | 处理 |
|------|------|
| 上传超大 zip | 在 `application.properties` 限制 `spring.servlet.multipart.max-file-size`，比如 200MB |
| zip 传输损坏 | 前端计算 SHA-256 随请求上送，后端重算比对，不一致返回 `CHECKSUM_MISMATCH`；对上传失败可触发前端重传 |
| zip 文件头伪装 | 接收后先校验魔数 `PK\x03\x04`，非 zip 立即拒绝 |
| 幂等上传 | SHA-256 命中已有包时跳过解压，直接返回 `mode: "no_change"` |
| zip 炸弹（解压后超大） | 解压时限制总字节数 + 文件数（ZipExtractUtil 内部阈值，默认 2GB/50000 文件） |
| zip 路径穿越（`../`） | 解压时 `Path.resolve` 后必须 `startsWith` 目标根目录 |
| 包结构不合法 | 缺 `package.json` / 无 `build:*` 脚本等 → 失败并返回 `INVALID_PACKAGE` |
| 包内敏感文件 | 扫描 `.env`、`*.key`、`*.pem`、`*.p12`，发现即给 warning（不拒绝） |
| 构建命令注入 | `cmd` 做前缀白名单：必须以 `npm run ` 开头 |
| 同一包并发构建 | MVP 只检查同一 `packageId` 是否已有正在运行的构建任务，有则返回 `EDIT_IN_PROGRESS` |
| 覆盖时正在构建/发布 | MVP 只在 upload 任务入队前检查该包是否有正在运行的构建/发布任务，有则立即返回 `EDIT_IN_PROGRESS`，不入队 |
| 完整 packageId 互斥 | 暂不处理 queued 任务、PATCH、DELETE 与 upload/build/publish 的统一互斥；后续有实际冲突再升级 |
| 覆盖过程失败 | `node_modules` 备份在 `.node_modules.bak`，源码解压失败时回滚：恢复 `.node_modules.bak` 并保留上一次的 `source/` 镜像（可用 `source/` rename 成 `source.old` 的策略实现） |
| 删包误删 | 仅删 `<packageBase>/<packageId>/` 子目录；路径做 `startsWith(packageBase)` 校验 |

---

## 9. 交付顺序（工作拆解）

按依赖关系排：

1. **建表 + Entity + Mapper**（半天）
   - SQL migration
   - `MiniappPackage` entity
   - MyBatis-Plus Mapper
2. **上传链路**（2 天）
   - `ZipExtractUtil`（含路径穿越/炸弹防护）
   - `PackageUploadTaskProcessor`（含覆盖策略：备份 node_modules → 清空 → 解压 → 回填）
   - 完整校验流程（见 2.4.7）：SHA-256 校验 + 魔数校验 + 幂等上传 + package.json 校验 + 敏感文件扫描
   - `/upload` + `/list` + `/{packageId}` + DELETE 接口
3. **构建链路（含 install）**（1.5 天）
   - `PackageInstallTaskProcessor`（npm ci / npm install + lock 哈希检测 + `.last_lock_hash` 写入）
   - `NovelAppBuildUtil.buildNovelAppWithWorkPath`（新方法，旧方法保留）
   - `BuildTaskProcessor` 分支
   - `/{packageId}/install` + `/{packageId}/build`（含 `DEPENDENCIES_NOT_INSTALLED` 前置检查）+ `/build/stop` 接口
4. **发布链路**（半天）
   - controller 装配逻辑
   - `/{packageId}/publish` + `/publish/stop/{taskId}`
   - `PATCH /{packageId}`（含 name/appId/token/buildCmd/version）
5. **批量链路**（半天）
   - `/batch-build` + `/batch-publish` + 两个 stop
   - 复用 `taskQueueManager.stopBatchTask` 和现有批量 WS topic
6. **前端页面**（1.5 天）
   - 列表、详情、上传弹窗、配置弹窗、日志面板
   - 批量选择 + 批量进度面板（从旧批量页组件复制后调整）
7. **联调 + 边界测试**（1 天）
   - 大 zip / 路径穿越 / 构建失败 / token 缺失 / 并发构建 / 批量中途失败

**合计约 7 个工作日。**

---

## 10. 不在本期范围

- 多租户凭证共享
- 代码包版本历史（只保留最新解压内容）
- 包之间的依赖缓存（node_modules 共享）
- CI 触发（Webhook / GitLab 集成）

多平台在本期直接放开：controller/service 接受 `weixin / douyin / kuaishou / baidu`，publish 装配层按 `platform` 选择 token 字段、`projectPath` 和 `platformCode`。

---

## 11. 待定决策

### 11.1 appId 从哪里来？

由于 `appId` 不再是主键且允许为空，用户可以：
- 上传时直接填写 `appId`（可选）
- 上传后通过 `PATCH /{packageId}` 补填 `appId`
- 从 `src/manifest.json` 自动提取（可选增强）

发布时 `appId` 必须非空，否则返回 `PUBLISH_APP_ID_MISSING`。

#### 可选增强：自动提取 appId

上传解压后，`PackageUploadTaskProcessor` 可以尝试从 `src/manifest.json` 中按当前平台读取 appid 字段，作为 `appId` 的自动填充来源（仅当用户未手动传入 `appId` 时生效）。

- manifest.json 可能是 JSONC（含注释/尾逗号），解析要用宽松 JSON parser
- 提取到的 appId 写入 task result 的 `detectedAppId` 字段，供前端展示确认
- 不强制覆盖用户手动传入的值

待和团队对齐后决定是否实现此增强。

---

## 12. 清理路线图（旧链路下线）

新链路上线后，旧的"批量构建 / 批量打包 / 构建 / 打包"等功能进入冻结期，不再接受演进。分三阶段推进清理：

### Phase 1：新旧并存（本期，本设计文档范围）

- 新增 `/api/miniapp-pkg/*` 共 15 个接口
- `NovelAppBuildUtil` 仅**新增**方法 `buildNovelAppWithWorkPath`，旧方法、`@Value("${build.workPath}")` 全部保留
- 旧 controller（`BatchBuildController` / `BatchPublishController` / `AgentBuildPublishController`）保持现状，不改造
- 前端新增"代码包"页面；旧的批量构建 / 批量打包 / 构建 / 打包页面不动

### Phase 2：前端切换（新链路稳定后）

**进入门禁**：新链路生产环境运行 ≥ 2 周，无 P0 / P1 问题。

- 前端旧的"批量构建 / 批量打包 / 构建 / 打包"入口接入 `/api/miniapp-pkg/batch-build` 和 `/batch-publish`
  - 数据源从 `novel_app` 切换为 `miniapp_package`
  - 可以采用"组件复制 + 字段映射调整"，也可以让旧页面直接改 API
- 旧入口在页面上打 deprecated 标识，灰度一段时间（建议 1 周）
- 验证所有业务场景都能由新链路承接

### Phase 3：后端清理（灰度期通过后）

**进入门禁**：前端所有旧入口已切换 ≥ 1 周，且旧后端端点访问量接近 0。

一次性清理，不保留兼容代码：

| 清理项 | 说明 |
|--------|------|
| `BatchBuildController` / `BatchPublishController` / `AgentBuildPublishController` | 删除整个类及其 DTO |
| `NovelAppBuildUtil` 旧方法 | `buildNovelApp`、`buildNovelAppWithTaskId` 删除 |
| `@Value("${build.workPath}")` 等注入 | 删除，所有 workPath 均由参数传入 |
| `application.properties` | 删除 `build.workPath` / `build.buildedPath` / `build.testWorkPath` / `build.testBuildedPath` |
| `NovelAppService` 等旧业务 | 确认无引用后删除 |
| `novel_app` 及相关旧表 | 确认无读写后，SQL 迁移脚本 `DROP TABLE` |
| `agent-scripts` 中旧 `build.sh` / `publish.sh` | 若仍需 CLI 入口，改为调用 `/api/miniapp-pkg/*` |

清理期提交独立 PR，便于 review 和 revert；不和本期新增的代码混在一起。

---

## 变更记录

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-05-08 | v1 | 初稿：单包上传/构建/发布 + 凭证池 |
| 2026-05-08 | v2 | 简化：单表 + appId 业务唯一 + 覆盖策略（保留 node_modules，清除 dist） |
| 2026-05-08 | v3 | 加入 zip 文件规范、appId 来源候选方案（待定）、批量接口 4 个、清理路线图 |
| 2026-05-08 | v4 | 加入完整校验流程（SHA-256 / 魔数 / 幂等 / 敏感文件扫描），新增错误码 `CHECKSUM_MISMATCH` / `INVALID_PACKAGE` |
| 2026-05-08 | v5 | 单表加 `platform` 枚举字段；`weixin_app_token` 改名 `app_token`；主键使用 `app_id` |
| 2026-05-08 | v6 | 对外标识统一为 `appId`，批量接口改为 `appIds` |
| 2026-05-08 | v7 | 明确本期不做包级权限校验，`owner_id` 仅用于审计 |
| 2026-05-08 | v8 | `/list` 返回全量包列表，token 字段脱敏为 `***` |
| 2026-05-08 | v9 | 四个平台在 controller/service 层放开，不写死 weixin |
| 2026-05-08 | v10 | 新增 `version` 字段；新增 `POST /{appId}/install` 接口（幂等，npm ci/install + lock 哈希检测）；build 加前置检查 `DEPENDENCIES_NOT_INSTALLED` + `actionHint`；新增 Agent 调用流程示例（4.6）；接口总数 14→15；工作量 6.5→7 天 |
| 2026-05-08 | v10 | 完整 appId 互斥暂不处理，MVP 只检查正在运行的构建/发布任务 |
| 2026-05-08 | v11 | **主键改为 `package_id`**；`app_id` 降为普通字段，允许为空；全链路标识从 `appId` 切换为 `packageId`；`packageId` 由调用方自行定义并在上传时传入（必填），命中已有则覆盖、未命中则新建；`appId` 可通过 PATCH 修改；批量接口改为 `packageIds`；接口总数保持 15 |
