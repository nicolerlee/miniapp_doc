# Deliver（微距）接口分析

## 概述

Deliver 接口用于获取微距策略配置（付费融合策略），请求地址为 `pub.funshion.com/interface/deliver`。配置中定义了多种微距类型，但 H5 端实际只请求了 `business_type`。

---

## 微距配置类型

`src/appConfig/host/fun.js` 中定义了以下微距类型：

| 类型 | 说明 | H5 端启用 |
|------|------|-----------|
| `business_type` | 付费融合策略 | ✅ ksh5/tth5/wxh5 均启用 |
| `public_switch` | 公共开关 | ❌ ksh5/tth5/wxh5 均 `enable: false` |
| `pd` | - | ❌ h5 `enable: false` |
| `daren_apply` | 达人申请 | ❌ h5 `enable: false` |
| `short_video` | 短视频 | 无 H5 配置 |

---

## H5 端只请求 business_type

### 配置

```js
// src/appConfig/host/fun.js
deliver: {
  business_type: {
    ksh5: { id: 'ks_h5_movie_business_type', enable: true },
    tth5: { id: 'tt_h5_fun_business_type', enable: true },
    wxh5: { id: 'wx_h5_fun_business_type', enable: true }
  },
  public_switch: {
    ksh5: { id: 'ks_h5_movie_public_switch', enable: false },
    tth5: { id: 'tt_h5_fun_public_switch', enable: false },
    wxh5: { id: 'wx_h5_fun_public_switch', enable: false }
  },
  // ...
}
```

### 调用链路

唯一的 `getDeliver` 调用在 `src/sdk/modules/combo/handler/inner/movie.js`：

```js
const deliverParam = {
  ...appConfig.deliver.business_type,  // 只传了 business_type
  mid: media.episode.mid,
  channel,
  // ...
}
deliver.clearDeliver(appConfig.deliver.business_type);
return await deliver.getDeliver(deliverParam);
```

没有任何地方对 `public_switch` 调用 `getDeliver`。

---

## public_switch 为什么没请求？

1. H5 端配置 `enable: false`，即使调用 `getDeliver` 也会直接返回空对象
2. 代码中没有任何地方传入 `appConfig.deliver.public_switch` 去请求
3. `public_switch` 只在小程序端（tt/ks/tb）启用，H5 端完全未使用

`reportParam.js` 中有一个字段注释提到 `macro_adId // public_switch的adId`，但这只是上报字段定义，不涉及实际请求。

---

## Deliver 请求参数

`src/sdk/modules/deliver/deliver.js` 中 `queryDeliver` 拼接的参数：

| 参数 | 来源 | 说明 |
|------|------|------|
| `ap` | `deliver.business_type.id` | 微距广告位 ID |
| `deliver_ver` | 写死 `'v1'` | 版本 |
| `client` | `appConfig.app_code` | 客户端标识 |
| `cl` | `appConfig.cl` | 渠道标识 |
| `scene_id` | `appConfig.scene` | 场景 ID |
| `appname` | `appConfig.appName` | App 名称 |
| `mid` | `media.episode.mid` | 媒体 ID |
| `channel` | 业务逻辑计算 | 渠道 |
| + `makeCommonParam()` | ajax 公共参数 | customer/fudid/platform/ve/os/cl/cp/uc/user_id/token |
| + `makeAdParam()` | ajax 渠道参数 | coop_main/coop_client/coop_expert/promotion_code 等 |

注意：deliver 请求会手动调用 `ajax.makeCommonParam()` 获取公共参数并合并，同时设置 `applyCommonParam = false` 避免 ajax 内部重复拼接。如果公共参数中 `coop_main` 或 `coop_client` 为空，会兜底为 `funshion` / `fun`。

---

## 关键文件

| 文件 | 作用 |
|------|------|
| `src/appConfig/host/fun.js` | 微距配置定义（deliver 字段） |
| `src/sdk/modules/deliver/deliver.js` | deliver 请求和缓存逻辑 |
| `src/sdk/modules/combo/handler/inner/movie.js` | 唯一调用 `getDeliver` 的地方 |
