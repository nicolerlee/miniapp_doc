miniappManager 是前台功能
miniappManagerServer 是后台功能

//现在构建时， 
1. 小程序的代码库地址， 是在miniappManagerServer/src/main/resources/application.properties的build.workPath, build.buildedPath配置的。 
2. 小程序的构建命令，是从数据库表中拼接的

// 现在发布接口，
1. 小程序构建产物， miniappManagerServer/src/main/resources/application.properties的build.buildedPath配置中获取的
2. 其他的一些参数，是从数据库表中获取的。


我想实现一个简化功能， 有一个上传接口，把小程序代码库zip包的形式上传。 然后针对这个代码zip包解压后，就可以实现构建，上传。 

1. 我可以上传多个zip包，每个zip包就是一个小程序代码库
2. 然后每个小程序可以构建，预览，上传

你觉得我要怎么做？ 



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
