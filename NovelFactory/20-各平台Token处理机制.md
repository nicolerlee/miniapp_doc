# 各平台 Token 处理机制

发布小程序时，各平台的 Token 用途和处理方式各不相同。本文档说明后台在执行发布任务时，如何使用传入的 Token。

---

## 概览

| 平台 | Token 字段 | 处理方式 | Token 类型 |
|---|---|---|---|
| 抖音 | `douyinAppToken` | 执行 `tma set-app-config` 写入 CLI 本地配置 | 普通字符串 |
| 快手 | `kuaishouAppToken` | 写成文件 `private.{appId}.key`，命令用 `--pkp` 引用 | RSA 私钥（多行） |
| 微信 | `weixinAppToken` | 写成文件 `private.{appId}.key`，命令用 `--private-key-path` 引用 | RSA 私钥（多行） |
| 百度 | `baiduAppToken` | 直接通过 `--token` 参数传入 `swan` 命令 | 普通字符串 |

---

## 抖音

发布前先执行 `tma set-app-config` 将 Token 写入 tma CLI 的本地配置，后续上传/预览命令自动读取：

```bash
# Step 1: 设置 Token
tma set-app-config {appId} --token {douyinAppToken}

# Step 2: 上传（正式发布）
tma upload -v {version} -c "{log}" {projectPath}

# Step 2: 预览（生成预览码）
tma preview {projectPath}
```

Token 是普通字符串，无换行符，可以直接通过命令行参数传递。

---

## 快手

Token 是 RSA 私钥内容（多行字符串），发布前写成文件 `private.{appId}.key` 放到 `projectPath` 下，并设置文件权限 `chmod 600`，上传/预览命令通过 `--pkp`（private key path）引用：

```bash
# Step 1: 写密钥文件（后台自动完成）
echo "{kuaishouAppToken}" > {projectPath}/private.{appId}.key
chmod 600 {projectPath}/private.{appId}.key

# Step 2: 上传（正式发布）
ks-miniprogram-ci upload \
  --pp {projectPath} \
  --appid {appId} \
  --pkp {projectPath}/private.{appId}.key \
  --uv {version} \
  --ud "{log}"

# Step 2: 预览（生成预览码）
ks-miniprogram-ci preview \
  --pp {projectPath} \
  --appid {appId} \
  --pkp {projectPath}/private.{appId}.key \
  --qrcode-format image \
  --qrcode-output-dest {projectPath}/ks_qrcode.png
```

> ⚠️ Token 包含换行符，**不能**通过命令行参数直接传递，必须通过 stdin 以 JSON 格式传入发布接口，由后台写文件处理。

---

## 微信

与快手相同，Token 是 RSA 私钥内容，写成文件 `private.{appId}.key`，命令通过 `--private-key-path` 引用：

```bash
# Step 1: 写密钥文件（后台自动完成）
echo "{weixinAppToken}" > {projectPath}/private.{appId}.key

# Step 2: 上传 / 预览（微信 miniprogram-ci）
# 命令中通过 --private-key-path 引用密钥文件
```

> ⚠️ 同快手，Token 包含换行符，必须通过 stdin 以 JSON 格式传入。

---

## 百度

Token 是普通字符串，直接通过 `--token` 参数传入 `swan` CLI：

```bash
# 上传（正式发布）
swan upload \
  --project-path {projectPath} \
  --release-version {version} \
  --min-swan-version 3.360.34 \
  --desc {log} \
  --token {baiduAppToken}

# 预览（生成预览码）
swan preview \
  --project-path {projectPath} \
  --min-swan-version 3.360.34 \
  --token {baiduAppToken}
```

---

## 调用发布接口时的注意事项

快手和微信的 Token 包含多行 RSA 私钥，**直接通过 shell 命令行参数传递会导致换行符被转义，私钥格式损坏**。

正确做法是通过 stdin 传入 JSON：

```bash
echo '{
  "platformCode": "mp-kuaishou",
  "appId": "ks717479719227682172",
  "projectPath": "/path/to/dist",
  "version": "4.3.2",
  "log": "修复已知问题",
  "publishMode": "preview",
  "kuaishouAppToken": "-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----"
}' | node scripts/publish.js
```

抖音和百度的 Token 是普通字符串，也可以通过命令行参数传递，但统一用 stdin 更安全。

---

## 二维码文件位置

发布/预览完成后，各平台二维码保存在 `projectPath` 下：

| 平台 | 文件名 |
|---|---|
| 抖音 | `tt_qrcode.png` |
| 快手 | `ks_qrcode.png` |
| 微信 | `wx_qrcode.png` |
| 百度 | `bd_qrcode.png` |
