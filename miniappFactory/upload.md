# 代码包上传流程

## 1. 概述

上传功能允许将小程序源码以 zip 包的形式上传到服务器。上传成功后，该代码包即可进行构建、发布等后续操作。

每个代码包由 `packageId` 唯一标识，同一 `packageId` 可以多次上传（覆盖更新）。

---

## 2. 前端流程

**入口**：`AutoUpload.vue`

用户填写以下信息后提交：

| 字段 | 必填 | 说明 |
|------|------|------|
| packageId | ✅ | 代码包唯一标识，新建时自动生成，更新时从详情页带入 |
| platform | ✅ | 平台：weixin / douyin / kuaishou / baidu |
| file（zip） | ✅ | 小程序源码 zip 包 |
| appName | ✅ | 小程序名称，未填时自动生成默认名 |
| appId | ❌ | 小程序 AppID |
| token | ❌ | 发布用 Token |
| buildCmd | ❌ | 构建命令，如 `npm run build:mp-weixin` |
| artifactDir | ❌ | 构建产物目录，未填时按平台取默认值 |
| version | ❌ | 版本号 |

提交前，前端会在浏览器里计算 zip 文件的 SHA-256：
- HTTPS / localhost：使用浏览器原生 `crypto.subtle`
- HTTP 内网 IP：fallback 到 `js-sha256` 纯 JS 库

两种情况都能算出真实哈希，不会传 `skip`。

---

## 3. 接口

```
POST /api/app/upload
Content-Type: multipart/form-data
```

接口同步接收文件、落盘 zip，然后**异步**处理后续校验和解压，返回 `taskId`。

**响应**

```json
{
  "code": 200,
  "data": { "taskId": "xxx-uuid" }
}
```

前端拿到 `taskId` 后通过 WebSocket 订阅实时日志：`/topic/upload-logs/{taskId}`

---

## 4. 后端处理流程

处理器：`PackageUploadTaskProcessor`

### 阶段 A：参数校验 + zip 落盘（同步，在 Controller 层完成）

- 校验必填参数（file、sha256、platform、appName）
- 将 zip 写入 `{packageBase}/{packageId}/origin.zip`
- 创建任务入队，返回 taskId

### 阶段 B：文件合法性/一致性校验（异步）

**魔数校验**：读取文件头 4 个字节，必须是 `50 4B 03 04`（即 `PK\x03\x04`，zip 格式固定签名）。文件名改成 `.zip` 但内容不是 zip 的直接拒绝。

**SHA-256 校验**：服务器重新计算 zip 的哈希，与前端传来的值对比，不一致说明文件在传输中损坏，拒绝处理。

### 阶段 C：幂等检查 + 解压

先查数据库是否已存在该 `packageId`：

**已存在（覆盖更新）**：
1. 备份现有 `node_modules` 到 `.node_modules.bak`
2. 清空 `source` 目录
3. 解压新 zip
4. 删除新 zip 中可能携带的 `node_modules` 和 `dist`
5. 将备份的 `node_modules` 移回（保留已安装的依赖，避免重复安装）
6. 更新数据库记录

**幂等跳过**：如果本次 zip 的 SHA-256 与上次上传记录（`.last_upload_sha256`）完全一致，直接返回"无变化"，不做任何操作。

**不存在（新建）**：
1. 创建 `source` 目录并解压 zip
2. 删除 zip 中可能携带的 `node_modules` 和 `dist`（节省磁盘，避免跨平台二进制兼容问题）
3. 创建数据库记录

### 阶段 D：包结构校验

解压完成后验证内容是否是合法的前端项目：

- 根目录必须有 `package.json`
- 如果 zip 内有一层外壳目录（如 `my-project/package.json`），自动扁平化
- `package.json` 的 `scripts` 里必须有至少一个 `build:*` 脚本

校验失败会执行回滚（删除数据库记录 + 清理文件目录）。

### 后续处理

- 检查 `package-lock.json` 是否变化，变化时在结果中标记 `packageLockChanged: true`，提示用户重新安装依赖
- 扫描敏感文件（`.env`、`.key`、`.pem`、`.p12`），发现时记录警告
- 将本次 zip 的 SHA-256 写入 `.last_upload_sha256`，供下次幂等判断使用

---

## 5. 安全机制

| 机制 | 说明 |
|------|------|
| 魔数校验 | 防止非 zip 文件伪装上传 |
| SHA-256 校验 | 防止文件传输损坏 |
| 路径穿越防护 | zip 条目路径 normalize 后必须在目标目录内，防止 `../../etc/passwd` 类攻击 |
| zip 炸弹防护 | 解压后总大小 ≤ 2GB，文件数 ≤ 50000 |
| 自动清理 | 解压后删除 `node_modules` 和 `dist`，防止磁盘浪费 |
| 失败回滚 | 任何阶段失败都清理数据库记录和文件目录，不留残留 |

---

## 6. 文件存储结构

```
{packageBase}/
└── {packageId}/
    ├── origin.zip              # 上传的原始 zip（处理完后保留）
    ├── source/                 # 解压后的源码目录
    │   ├── package.json
    │   ├── src/
    │   └── ...
    ├── .last_upload_sha256     # 最后一次成功上传的 zip SHA-256
    └── .last_lock_hash         # 最后一次 install 时的 package-lock.json SHA-256
```

---

## 7. 为什么 zip 里带 node_modules 不能直接用

理论上可以，但实际有以下问题：

1. **平台不兼容**：部分 npm 包含原生二进制（`.node` 文件），在用户 macOS 上编译的二进制在 Linux 服务器上无法运行
2. **路径硬编码**：某些包安装时会把绝对路径写入配置，换了机器路径对不上
3. **磁盘浪费**：node_modules 几百 MB，zip 压缩率极差，上传慢、存储浪费

因此统一删除，通过 install 步骤在服务器上重新安装，保证环境一致。
