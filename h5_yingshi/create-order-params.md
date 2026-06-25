# 下单接口渠道参数分析

## 概述

下单接口（`createOrder`）的渠道参数传递方式与其他 API 接口不同。普通接口通过 `ajax.js` 的公共参数自动拼接渠道信息，而下单接口被显式排除，改为在 `usePay.js` 中按特定格式自行拼接。

---

## 为什么下单接口不走公共参数？

`src/sdk/modules/utils/ajax.js` 中 `makeAdParam` 函数：

```js
const makeAdParam = (url, params = {}) => {
  // 抖音登录接口和下单接口，跳过渠道参数拼接
  if (url == `${appConfig.ttH5Login}douyin` || url == appConfig.createOrder)
    return { h5_login_check: '' }
  // ... 其他接口正常拼接 promotion_code、coop_main 等
}
```

普通接口（play、episode、profile、payment 等）走 `ajax.get()` 时，会自动合并 `makeCommonParam()` + `makeAdParam()` 两组公共参数，`makeAdParam` 里会平铺 `promotion_code`、`promotion_ad_id`、`coop_main` 等渠道参数到 query string。

下单接口被排除的原因：后端要求下单的渠道参数按特定格式传递（entrance 编码字符串、promotion JSON），跟普通接口直接平铺在 query 参数里的方式不同。如果两边都传，可能造成参数冲突或重复计费归因。

---

## 下单接口的参数来源

参数在 `src/hooks/usePay.js` 的 `makePayParams` 函数中拼接，分为四个层面：

### 1. path 级别直接参数

| 参数 | 来源 | 说明 |
|------|------|------|
| `goods_type` | payment 对象 | member / token / vod |
| `goods_id` | payment 对象 | 商品 ID |
| `price` / `amount` | payment 对象 | 价格（元 / 分） |
| `platform` | `appConfig.platform` | 平台标识 |
| `gateway_id` | `appConfig.pay` 配置 | 支付网关，按 OS 和支付类型区分 |
| `openid` | `appConfig.open_id` | 用户 open_id |
| `params` | 拼接 open_id + return_url | 快手用 `open_id:`，其他用 `openid:` |
| `curr_url` | 当前页面 URL | 用于支付回调 |
| `pt_videoid` | `commonMountInfo.videoid` | 挂载视频 ID |
| `pt_taskid` | `commonMountInfo.taskid` | 挂载任务 ID |
| `x_mid` / `x_eid` | `commonMountInfo` | 媒体/剧集 ID |

### 2. `addCommonMountPath` 拼接的挂载参数

```js
function addCommonMountPath(params) {
  Object.assign(params, {
    expert_open_id: appConfig.commonMountInfo.expert_gamma_id,
    coop_client: addMountInfo.coopClient,
    coop_main: addMountInfo.coopMain,
    coop_expert: addMountInfo.popularizeId,
    coop_appid: addMountInfo.microapp_id,
  })
  // 推广客户端额外传 clicklocation
}
```

### 3. `entrance` 透传参数（编码为字符串）

`makePayEntranceTouchuanParam` 将以下参数编码后作为 `entrance` 字段传递：

| 参数 | 说明 |
|------|------|
| `platform` / `dev` | 操作系统 |
| `appname` | 宿主 App 名称 |
| `scene` / `ver` / `cp` | 场景值、版本号、渠道包 |
| `pay_page` | 支付页面路由 |
| `mid` / `eid` | 媒体/剧集 ID |
| `cm_id` / `ce_id` | 客户资源 ID |
| `pt_videoid` / `pt_taskid` | 挂载视频/任务 |
| `expert_open_id` | 达人 ID |
| `source` | 来源 |
| `coop_code` / `coop_client` / `coop_main` / `coop_expert` / `coop_appid` | 合作渠道全链路 |
| `si` / `osi` / `osi_expert` | 会话标识 |
| `callback` | 回调标识 |
| `promotion_ad_id` / `promotion_thirdad_id` / `promotion_code` / `promotion_pt` | 推广广告参数 |
| `third_account_id` / `third_set_id` / `third_maccount_id` | 腾讯广点通专用（`promotion_code === 'txgdt'` 时） |
| `is_vip_policy` | 是否 VIP 策略 |
| `clicklocation` | 点击位置 |

### 4. `promotion` 透传参数（JSON 字符串）

`makePayPromotionTouchuanParam` 按挂载来源生成不同内容：

| 来源 | 参数 |
|------|------|
| 广告挂载（`from == 'ad' / 'not_ad'`） | `coop`（合作码）、`expert`（达人 ID）、`appid`（小程序 ID） |
| 推广客户端 | `coop`（取 `coop_main`）、`expert`、`appid` |
| 橙星推（`from == 'appfun'`） | 不传 promotion（需求 #56389） |

---

## `return_url` 白名单（H5 非续费场景）

H5 环境下（tth5 / ksh5 / wxh5），非续费订单会从当前页面 URL 中提取白名单参数拼入回调地址：

```js
const whiteList = [
  'mid', 'eid', 'source', 'ctime', 'entrance', 'si',
  'coopCode', 'microapp_id', 'popularizeId', 'clickid',
  'promotionid', 'promotion_ad_id', 'promotion_code',
  'coop_main', 'coop_client', 'coop_expert', 'coop_appid',
]
```

微信 H5（`wxh5`）不传 `params` 字段（即不传 return_url）。

---

## 与普通接口的对比

| 维度 | 普通接口（play/episode/profile 等） | 下单接口（createOrder） |
|------|------|------|
| 渠道参数来源 | `ajax.js` 的 `makeAdParam` 自动拼接 | `usePay.js` 的 `makePayParams` 手动拼接 |
| 传递方式 | 平铺在 query string | path 参数 + entrance 编码 + promotion JSON |
| `promotion_code` | 直接作为 query 参数 | 编码在 `entrance` 字符串内 |
| `coop_main` 等 | 直接作为 query 参数 | path 级别 + entrance 内都有 |

---

## 关键文件

| 文件 | 作用 |
|------|------|
| `src/sdk/modules/utils/ajax.js` | 公共参数拼接，`createOrder` 被排除 |
| `src/hooks/usePay.js` → `makePayParams` | 下单参数主逻辑 |
| `src/hooks/usePay.js` → `makePayEntranceTouchuanParam` | entrance 透传参数拼接 |
| `src/hooks/usePay.js` → `makePayPromotionTouchuanParam` | promotion 透传参数拼接 |
| `src/hooks/usePay.js` → `addCommonMountPath` | 挂载参数拼接到 path |
