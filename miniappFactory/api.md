# API 接口文档

本文档列出 novel-miniapp-manager skill 使用的所有接口。

**认证**：所有请求需携带 `X-API-Key: <NOVEL_API_KEY>` 和 `X-Source: api` 请求头。

---

## 目录

1. [环境验证](#1-环境验证)
2. [应用列表](#2-应用列表)
3. [应用查询](#3-应用查询)
4. [应用详情](#4-应用详情)
5. [创建应用](#5-创建应用)
6. [更新应用](#6-更新应用)
7. [删除应用](#7-删除应用)
8. [广告配置 - 新建](#8-广告配置--新建)
9. [广告配置 - 删除](#9-广告配置--删除)
10. [支付配置 - 新建](#10-支付配置--新建)
11. [支付配置 - 删除](#11-支付配置--删除)
12. [构建](#12-构建)
13. [发布](#13-发布)
14. [发布二维码下载](#14-发布二维码下载)
15. [构建产物列表](#15-构建产物列表)
16. [任务状态查询](#16-任务状态查询)
17. [任务日志查询](#17-任务日志查询)

---

## 1. 环境验证

验证 API Key 是否有效。

```
GET /open/v1/me
Authorization: Bearer <NOVEL_API_KEY>
```

**响应**

```json
{
  "success": true,
  "result": {
    "userId": "xxx"
  }
}
```

---

## 2. 应用列表

获取所有应用的基础信息列表，支持按平台过滤。

```
GET /api/agent/app/list
GET /api/agent/app/list?platform=douyin
```

**Query 参数**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| platform | string | 否 | 平台过滤：`douyin` / `weixin` / `kuaishou` / `baidu` |

**响应**

```json
{
  "code": 200,
  "data": [
    {
      "id": 1,
      "appid": "tt025674eb9d672b7101",
      "appName": "星辰文鉴",
      "platform": "douyin",
      "version": "4.3.1",
      "appCode": "tt_miniapp_xingchennovel",
      "product": "xingchennovel",
      "customer": "tt_xingchennovel",
      "deliverId": "tt_mp_xingchen_business_type",
      "bannerId": "tt_mp_xingchen_public_switch",
      "tokenId": 22,
      "cl": "tt_miniapp_xingchennovel",
      "createTime": "2024-01-01T00:00:00",
      "updateTime": "2024-01-01T00:00:00",
      "lastBuildTime": "2024-01-01T00:00:00"
    }
  ]
}
```

---

## 3. 应用查询

按条件聚合查询，支持多条件组合。命中唯一结果时返回单对象，多个结果返回数组。

```
GET /api/agent/app/query?<params>
```

**Query 参数**（至少提供一个）

| 参数 | 类型 | 说明 |
|------|------|------|
| appId | string | 平台分配的 AppID（精确匹配，优先级最高） |
| appCode | string | 应用代码（精确匹配） |
| customer | string | 客户标识 |
| platform | string | 平台：`douyin` / `weixin` / `kuaishou` / `baidu` |
| appName | string | 应用名称（精确匹配） |

**响应 - 命中唯一**

```json
{
  "code": 200,
  "data": {
    "appid": "tt025674eb9d672b7101",
    "appName": "星辰文鉴",
    "platform": "douyin",
    "version": "4.3.1",
    "appCode": "tt_miniapp_xingchennovel",
    "customer": "tt_xingchennovel",
    "paymentConfig": { ... },
    "adConfig": { ... }
  }
}
```

**响应 - 命中多个**

```json
{
  "code": 200,
  "data": [ { ... }, { ... } ]
}
```

---

## 4. 应用详情

查询单个应用的完整聚合配置（包含所有配置域）。

```
GET /api/agent/app/detail?appId=<appId>
```

**Query 参数**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| appId | string | ✅ | 平台分配的 AppID |

**响应**

```json
{
  "code": 200,
  "data": {
    "appid": "tt025674eb9d672b7101",
    "appName": "星辰文鉴",
    "platform": "douyin",
    "version": "4.3.1",
    "baseConfig": {
      "appName": "星辰文鉴",
      "platform": "douyin",
      "appCode": "tt_miniapp_xingchennovel",
      "appid": "tt025674eb9d672b7101",
      "version": "4.3.1",
      "product": "xingchennovel",
      "customer": "tt_xingchennovel",
      "tokenId": 22,
      "cl": "tt_miniapp_xingchennovel",
      "bannerId": "tt_mp_xingchen_public_switch",
      "deliverId": "tt_mp_xingchen_business_type"
    },
    "commonConfig": {
      "buildCode": "xingchen",
      "douyinAppToken": "Ld4gqz-...",
      "contact": "https://im..."
    },
    "paymentConfig": {
      "normalPay": { "enabled": true, "gatewayAndroid": 815, "gatewayIos": 815 },
      "orderPay":  { "enabled": true, "gatewayAndroid": 815, "gatewayIos": 815 }
    },
    "adConfig": {
      "rewardAd": { "enabled": true, "rewardAdId": "xxx", "rewardCount": 3 },
      "bannerAd": { "enabled": false, "bannerAdId": "xxx" }
    },
    "uiConfig": {
      "mainTheme": "#2552F5",
      "secondTheme": "#dce7ff",
      "homeCardStyle": 1,
      "payCardStyle": 1
    }
  }
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| `400` | `INVALID_PARAMS` | appId 为空或应用不存在 |
| `401` | `UNAUTHORIZED` | API Key 无效或未提供 |

---

## 5. 创建应用

异步接口，返回 taskId，需轮询任务状态。

```
POST /api/agent/app/create
Content-Type: application/json
```

**请求体**

```json
{
  "baseConfig": {
    "appName": "星辰文鉴",
    "platform": "douyin",
    "appCode": "tt_miniapp_xingchennovel",
    "appid": "tt025674eb9d672b7101",
    "version": "4.3.1",
    "product": "xingchennovel",
    "customer": "tt_xingchennovel",
    "tokenId": 22,
    "cl": "tt_miniapp_xingchennovel",
    "bannerId": "tt_mp_xingchen_public_switch",
    "deliverId": "tt_mp_xingchen_business_type"
  },
  "commonConfig": {
    "buildCode": "xingchen",
    "douyinAppToken": "Ld4gqz-...",
    "contact": "https://im..."
  },
  "paymentConfig": {},
  "adConfig": {},
  "uiConfig": {}
}
```

**baseConfig 必填字段**：`appName`、`platform`、`appCode`、`appid`、`version`、`product`、`customer`、`tokenId`、`cl`、`bannerId`、`deliverId`

**commonConfig 必填字段**：`buildCode`、`contact`、`kuaishouClientId`、`kuaishouClientSecret`、`mineLoginType`、`readerLoginType`

**commonConfig 可选字段**：`douyinAppToken`、`weixinAppToken`、`kuaishouAppToken`、`baiduAppToken`、`douyinImId`、`iaaMode`、`iaaDialogStyle`、`hidePayEntry`、`hideScoreExchange`

**响应**

```json
{
  "code": 200,
  "data": { "taskId": "abc123" }
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| `401` | `UNAUTHORIZED` | API Key 无效或未提供 |
| `403` | `FORBIDDEN` | 当前账号无创建应用权限 |
| `409` | `DUPLICATE_APP` | 已存在同名同平台的应用 |
| `409` | `CREATE_IN_PROGRESS` | 当前有创建任务正在进行 |

---

## 6. 更新应用

同步接口，只传要修改的字段，未传字段保持原值。

```
POST /api/agent/app/update
Content-Type: application/json
```

**请求体**（只需包含要修改的域）

```json
{
  "appId": "tt025674eb9d672b7101",
  "baseConfig": {
    "version": "4.3.2"
  },
  "uiConfig": {
    "mainTheme": "#FF0000"
  },
  "adConfig": {
    "rewardAd": { "rewardCount": 5 }
  },
  "paymentConfig": {
    "payType": "normalPay",
    "gatewayAndroid": 900
  }
}
```

> ⚠️ `paymentConfig` 更新时每次只能更新一种支付方式，只需传 `payType` + 要改的字段，未传字段服务端自动保留原值。

**响应**

```json
{
  "code": 200,
  "data": null
}
```

> 更新成功后 data 为 null，如需查看最新配置请调用 `GET /api/agent/app/detail`。

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| `400` | `INVALID_PARAMS` | 参数错误 |
| `401` | `UNAUTHORIZED` | API Key 无效或未提供 |
| `409` | `CREATE_IN_PROGRESS` | 当前有创建任务正在进行 |
| `409` | `EDIT_IN_PROGRESS` | 应用正在被其他操作编辑 |

---

## 7. 删除应用

同步接口，不可逆。

```
DELETE /api/agent/app/delete?appId=<appId>
```

**Query 参数**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| appId | string | ✅ | 平台分配的 AppID |

**响应**

```json
{
  "code": 200,
  "data": null
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| `401` | `UNAUTHORIZED` | API Key 无效或未提供 |
| `403` | `FORBIDDEN` | 当前账号无删除应用权限 |
| `400` | `INVALID_PARAMS` | 应用不存在或参数错误 |
| `500` | `INVALID_PARAMS` | 删除过程中服务端异常（后端 bug，errorCode 字段值为 INVALID_PARAMS） |

---

## 8. 广告配置 - 新建

为指定应用新建一种广告配置。

```
POST /api/novel-ad/adConfig/create
Content-Type: application/json
```

**请求体**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| appId | string | ✅ | 平台分配的 AppID |
| adType | string | ✅ | `reward` / `interstitial` / `banner` / `feed` |
| rewardAdId | string | reward 时 | 激励广告 ID |
| rewardCount | number | reward 时 | 激励次数 |
| isRewardAdEnabled | boolean | reward 时 | 是否启用 |
| interstitialAdId | string | interstitial 时 | 插屏广告 ID |
| interstitialCount | number | interstitial 时 | 插屏次数 |
| isInterstitialAdEnabled | boolean | interstitial 时 | 是否启用 |
| bannerAdId | string | banner 时 | Banner 广告 ID |
| isBannerAdEnabled | boolean | banner 时 | 是否启用 |
| feedAdId | string | feed 时 | Feed 广告 ID |
| isFeedAdEnabled | boolean | feed 时 | 是否启用 |

**示例**

```json
{
  "appId": "tt025674eb9d672b7101",
  "adType": "reward",
  "rewardAdId": "ad_xxx",
  "rewardCount": 3,
  "isRewardAdEnabled": true
}
```

**响应**

```json
{
  "code": 200,
  "data": { /* 创建后的广告配置对象 */ }
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| `401` | `UNAUTHORIZED` | API Key 无效或未提供 |
| `403` | `FORBIDDEN` | 当前账号无广告配置写权限 |
| `409` | `EDIT_IN_PROGRESS` | 应用正在被其他操作编辑 |

---

## 9. 广告配置 - 删除

删除指定应用的某种广告配置。

```
GET /api/novel-ad/adConfig/deleteByAppIdAndType?appId=<appId>&adType=<adType>
```

**Query 参数**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| appId | string | ✅ | 平台分配的 AppID |
| adType | string | ✅ | `reward` / `interstitial` / `banner` / `feed` |

**响应**

```json
{
  "code": 200,
  "data": null
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| `401` | `UNAUTHORIZED` | API Key 无效或未提供 |
| `403` | `FORBIDDEN` | 当前账号无广告配置写权限 |
| `409` | `EDIT_IN_PROGRESS` | 应用正在被其他操作编辑 |

---

## 10. 支付配置 - 新建

为指定应用新建一种支付配置。

```
POST /api/novel-pay/create
Content-Type: application/json
```

**请求体**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| appId | string | ✅ | 平台分配的 AppID |
| payType | string | ✅ | 见下方枚举 |
| enabled | boolean | ✅ | 是否启用 |
| gatewayAndroid | number | ✅ | Android 支付网关 ID |
| gatewayIos | number | ✅ | iOS 支付网关 ID |

**payType 枚举**：`normalPay` / `orderPay` / `renewPay` / `douzuanPay` / `imPay` / `wxVirtualPay` / `wxVirtualRenewPay`

**示例**

```json
{
  "appId": "tt025674eb9d672b7101",
  "payType": "normalPay",
  "enabled": true,
  "gatewayAndroid": 815,
  "gatewayIos": 815
}
```

**响应**

```json
{
  "code": 200,
  "data": { /* 创建后的支付配置对象 */ }
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| `401` | `UNAUTHORIZED` | API Key 无效或未提供 |
| `403` | `FORBIDDEN` | 当前账号无支付配置写权限 |
| `409` | `EDIT_IN_PROGRESS` | 应用正在被其他操作编辑 |

---

## 11. 支付配置 - 删除

删除指定应用的某种支付配置。

```
GET /api/novel-pay/deleteAppPayByAppIdAndType?appId=<appId>&payType=<payType>
```

**Query 参数**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| appId | string | ✅ | 平台分配的 AppID |
| payType | string | ✅ | 见支付类型枚举 |

**响应**

```json
{
  "code": 200,
  "data": null
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| `401` | `UNAUTHORIZED` | API Key 无效或未提供 |
| `403` | `FORBIDDEN` | 当前账号无支付配置写权限 |
| `409` | `EDIT_IN_PROGRESS` | 应用正在被其他操作编辑 |

---

## 12. 构建

提交构建任务，异步执行，返回 taskId。

```
POST /api/agent/build
Content-Type: application/json
```

**请求体**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| cmd | string | ✅ | 构建命令，格式：`npm run build:<平台前缀>-<buildCode>` |

**示例**

```json
{ "cmd": "npm run build:tt-xingchen" }
```

**响应**

```json
{
  "code": 200,
  "data": { "taskId": "abc123" }
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| `400` | `INVALID_PARAMS` | 构建命令为空 |
| `401` | `UNAUTHORIZED` | API Key 无效或未提供 |
| `403` | `FORBIDDEN` | 当前账号无构建权限 |
| `500` | `BUILD_FAILED` | 加入构建队列失败 |

---

## 13. 发布

提交发布任务，异步执行，返回 taskId。

```
POST /api/agent/publish
Content-Type: application/json
```

**请求体**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| platformCode | string | ✅ | `mp-toutiao` / `mp-weixin` / `mp-kuaishou` / `mp-baidu` |
| appId | string | ✅ | 平台分配的 AppID |
| projectPath | string | ✅ | 构建产物目录的绝对路径 |
| version | string | ✅ | 版本号 |
| log | string | ✅ | 发布说明 |
| publishMode | string | 否 | `preview`（预览码）/ `publish`（正式发布），默认 `preview` |
| douyinAppToken | string | 抖音时必填 | 抖音平台 Token |
| weixinAppToken | string | 微信时必填| 微信平台 RSA 私钥 |
| kuaishouAppToken | string | 快手时必填| 快手平台 Token |
| baiduAppToken | string | 百度时必填| 百度平台 Token |

**示例**

```json
{
  "platformCode": "mp-toutiao",
  "appId": "tt025674eb9d672b7101",
  "projectPath": "/path/to/dist/mp-toutiao",
  "version": "4.3.2",
  "log": "修复若干问题",
  "publishMode": "preview",
  "douyinAppToken": "Ld4gqz-..."
}
```

**响应**

```json
{
  "code": 200,
  "data": { "taskId": "abc123" }
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| `400` | `INVALID_PARAMS` | 缺少必要参数 |
| `401` | `UNAUTHORIZED` | API Key 无效或未提供 |
| `403` | `FORBIDDEN` | 当前账号无发布权限 |
| `500` | `PUBLISH_FAILED` | 加入发布队列失败 |

---

## 14. 发布二维码下载

发布任务完成后获取预览/发布二维码。

```
GET /api/agent/publish/qrcode/<taskId>
```

**响应**：PNG 图片二进制流（`Content-Type: image/png`）

二维码文件由服务端在发布完成后写入构建产物目录，文件名按平台固定：

| platformCode | 文件名 |
|---|---|
| `mp-toutiao` | `tt_qrcode.png` |
| `mp-weixin` | `wx_qrcode.png` |
| `mp-kuaishou` | `ks_qrcode.png` |
| `mp-baidu` | `bd_qrcode.png` |

**注意**：文件不存在时返回 HTTP 404（无 JSON body），说明发布任务尚未完成或二维码尚未生成，需等待后重试。

---

## 15. 构建产物列表

获取所有构建产物，可按 appId 过滤。

```
GET /api/novel-publish/list
```

**响应**

```json
{
  "code": 200,
  "data": [
    {
      "platforms": [
        {
          "appId": "tt025674eb9d672b7101",
          "platformCode": "mp-toutiao",
          "projectPath": "/path/to/dist/mp-toutiao",
          "version": "4.3.1",
          "buildTime": "2024-01-01T00:00:00Z"
        }
      ]
    }
  ]
}
```

---

## 16. 任务状态查询

轮询异步任务（创建/构建/发布）的执行状态。

```
GET /api/agent/task/<taskId>/status
```

**响应**

```json
{
  "code": 200,
  "message": "查询成功",
  "data": {
    "taskId": "abc123",
    "taskType": "BUILD",
    "source": "api",
    "status": "COMPLETED",
    "progress": 100,
    "description": "构建成功",
    "startTime": "2024-01-01T10:00:00",
    "completeTime": "2024-01-01T10:02:30",
    "errorCode": null,
    "errorMessage": null
  }
}
```

**data 字段说明**

| 字段 | 类型 | 说明 |
|------|------|------|
| `taskId` | string | 任务 ID |
| `taskType` | string | 任务类型：`BUILD` / `PUBLISH` / `CREATE` |
| `source` | string | 来源：`api` / `web` |
| `status` | string | 任务状态，见下方枚举 |
| `progress` | number | 进度 0-100 |
| `description` | string | 当前状态描述 |
| `startTime` | string | 任务开始时间（ISO 8601） |
| `completeTime` | string | 任务完成时间（ISO 8601），未完成时为 null |
| `errorCode` | string | 错误码，失败时非 null |
| `errorMessage` | string | 错误描述，失败时非 null |

**status 枚举**

| status | 说明 |
|--------|------|
| `SUBMITTED` | 已提交，等待调度 |
| `PENDING` | 已入队，等待执行 |
| `RUNNING` | 执行中 |
| `COMPLETED` | 执行成功 |
| `FAILED` | 执行失败 |

**失败响应示例**

```json
{
  "code": 200,
  "message": "查询成功",
  "data": {
    "taskId": "abc123",
    "taskType": "PUBLISH",
    "source": "api",
    "status": "FAILED",
    "progress": 30,
    "description": "发布失败",
    "startTime": "2024-01-01T10:00:00",
    "completeTime": "2024-01-01T10:01:05",
    "errorCode": "PUBLISH_FAILED",
    "errorMessage": "上传失败：Token 已过期"
  }
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| `401` | - | 未登录或用户不存在 |
| `403` | - | 无权查看该任务（非任务所有者且非管理员） |
| `404` | `TASK_NOT_FOUND` | 任务不存在 |

---

## 17. 任务日志查询

获取失败任务的详细日志。

```
GET /api/agent/task/<taskId>/logs
```

**响应**

```json
{
  "code": 200,
  "data": "任务执行日志内容..."
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| `401` | - | 未登录或用户不存在 |
| `403` | - | 无权查看该任务（非任务所有者且非管理员） |
| `404` | `TASK_NOT_FOUND` | 任务不存在 |

---

## 通用响应结构

```json
{
  "code": 200,
  "message": "success",
  "errorCode": null,
  "errorMessage": null,
  "data": { ... }
}
```

失败时 `errorCode` 和 `errorMessage` 会有值，例如：

```json
{
  "code": 409,
  "message": "已存在同名同平台的小程序",
  "errorCode": "DUPLICATE_APP",
  "errorMessage": "已存在同名同平台的小程序",
  "data": null
}
```

---

## HTTP 状态码

| HTTP code | 含义 |
|-----------|------|
| `200` | 成功 |
| `400` | 参数错误或资源不存在（如 appId 无效） |
| `401` | 未登录 / API Key 无效或已过期 |
| `403` | 权限不足，当前账号角色无权执行此操作 |
| `404` | 资源不存在 |
| `409` | 冲突（重复创建、任务进行中等） |
| `500` | 服务器内部错误 |

---

## 结构化错误码（errorCode）

### 通用

| errorCode | HTTP code | 含义 | 出现场景 |
|-----------|-----------|------|---------|
| `INVALID_PARAMS` | 400 | 参数无效或缺少必填字段 | 创建/更新/删除时参数校验失败；appId 不存在 |
| `FORBIDDEN` | 403 | 权限不足 | 当前账号角色无权执行此操作（需联系管理员授权） |
| `UNAUTHORIZED` | 401 | 未授权 | API Key 无效、已过期或未提供 |
| `RATE_LIMIT_EXCEEDED` | 429 | 请求频率超限 | 短时间内请求次数过多 |

### 应用管理

| errorCode | HTTP code | 含义 | 出现场景 |
|-----------|-----------|------|---------|
| `DUPLICATE_APP` | 409 | 同名同平台应用已存在 | 创建应用时，相同 appName + platform 已有记录 |
| `CREATE_IN_PROGRESS` | 409 | 已有创建任务正在进行 | 创建应用时，上一个创建任务尚未完成 |
| `EDIT_IN_PROGRESS` | 409 | 应用正在被编辑 | 更新/广告/支付操作时，该应用有其他编辑操作正在执行 |
| `TASK_NOT_FOUND` | 404 | 任务不存在 | 查询任务状态/日志时，taskId 无效或已过期 |

### 构建 & 发布

| errorCode | HTTP code | 含义 | 出现场景 |
|-----------|-----------|------|---------|
| `BUILD_FAILED` | 500 | 构建失败 | 构建任务执行过程中出错 |
| `BUILD_INTERRUPTED` | 500 | 构建被中断 | 构建任务被手动停止或进程异常退出 |
| `PUBLISH_TOKEN_MISSING` | 400 | 发布缺少平台 Token | 发布时未提供对应平台的 Token（如 `douyinAppToken`） |
| `PUBLISH_FAILED` | 500 | 发布失败 | 发布任务执行过程中出错 |
| `STOP_FAILED` | 500 | 停止任务失败 | 停止构建/发布时操作失败 |
