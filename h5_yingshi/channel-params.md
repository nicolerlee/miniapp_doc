# 渠道参数分析

## 一、下单接口（createOrder）渠道参数

代码位置：`src/hooks/usePay.js` → `makePayParams`

下单接口被 `src/sdk/modules/utils/ajax.js` 的 `makeAdParam` 显式排除，不走公共参数拼接，改为手动拼成三层结构。

### path 级别直接参数

| 参数 | 说明 |
|------|------|
| `coop_client` | 合作客户端标识 |
| `coop_main` | 合作主渠道 |
| `coop_expert` | 达人 ID |
| `coop_appid` | 小程序 ID |
| `expert_open_id` | 达人 open_id |
| `pt_videoid` | 挂载视频 ID |
| `pt_taskid` | 挂载任务 ID |
| `x_mid` | 媒体 ID |
| `x_eid` | 剧集 ID |

### entrance 字段（编码字符串透传）

`makePayEntranceTouchuanParam` 将以下参数编码后作为 `entrance` 字段传递：

| 参数 | 说明 |
|------|------|
| `coop_code` / `coop_client` / `coop_main` / `coop_expert` / `coop_appid` | 合作渠道全链路 |
| `si` / `osi` / `osi_expert` | 会话标识 |
| `callback` | 回调标识 |
| `promotion_ad_id` | 广告 ID |
| `promotion_thirdad_id` | 第三方广告 ID |
| `promotion_code` | 推广渠道码 |
| `promotion_pt` | 推广平台 |
| `third_account_id` / `third_set_id` / `third_maccount_id` | 腾讯广点通专用（`promotion_code === 'txgdt'` 时） |
| `platform` / `dev` / `appname` | 设备/宿主信息 |
| `scene` / `ver` / `cp` | 场景值、版本、渠道包 |
| `pay_page` | 支付页面路由 |
| `mid` / `eid` | 媒体/剧集 ID |
| `source` / `clicklocation` | 来源、点击位置 |

### promotion 字段（JSON 字符串透传）

`makePayPromotionTouchuanParam` 按挂载来源生成：

| 来源 | 参数 |
|------|------|
| 广告（`from == 'ad'`）/ 推广 | `coop`（合作码）、`expert`（达人 ID）、`appid`（小程序 ID） |
| 橙星推（`from == 'appfun'`） | 不传 promotion |

---

## 二、上报接口渠道参数

代码位置：`src/sdk/modules/report/report.js` → `makeParam` + `addMountParams`

### 基础参数（makeParam）

| 参数 | 说明 |
|------|------|
| `channel` | 渠道标识 |
| `scene` | 场景值 |
| `platform` | 平台类型 android/ios |
| `ver` | 版本号 |
| `cp` | 渠道包 |
| `cl` / `client` / `product` | app_code 拆分字段 |

### 挂载渠道参数（addMountParams → CommonMount.addCommonMountReportParams）

投流渠道（`from === 'ad'`）：

| 参数 | 说明 |
|------|------|
| `coopCode` / `coopMain` / `coopClient` | 合作渠道码 |
| `popularizeId` | 达人 ID |
| `microapp_id` | 小程序 ID |
| `osi` / `si` | 会话标识 |
| `callback` | 回调标识 |
| `promotion_ad_id` / `promotion_thirdad_id` / `promotion_code` / `promotion_pt` | 广告推广参数 |

推广/橙星推（`from === 'tuiguang' / 'appfun'`）：

| 参数 | 说明 |
|------|------|
| `coop_main` / `coop_client` / `coop_expert` / `coop_appid` | 合作渠道全链路 |
| `si` / `callback` | 会话/回调标识 |
| `promotion_*` | 同上 |

---

## 三、渠道参数解析

### 解析入口

`src/sdk/modules/mount/getMountConfig.js` → `queryMountConfig`

解析结果统一存入 `AppConfig.commonMountInfo`，下单和上报都从这里取。

### 三种 Resolver（策略模式）

| Resolver | 文件 | 触发条件 |
|----------|------|----------|
| AdKey | `resolver/adKey.js` | `coopCode === 'ad'` |
| FunKey | `resolver/funKey.js` | `coop_client === 'appfun'` |
| TuiguangKey | `resolver/tuiguangKey.js` | 有 `coop_client` 但没有 `coopCode` |

### 各 Resolver 解析的 key

**AdKey** 解析：`coopCode`、`ctime`、`microapp_id`、`popularizeId`、`si`、`clickid`、`callback`、`clue_token`、`promotion_ad_id`、`promotion_thirdad_id`、`promotion_code`、`promotion_pt`、`adid`、`gdt_vid`、`qz_gdt`、`third_account_id`、`third_set_id`、`third_maccount_id`

**FunKey / TuiguangKey** 解析：`source`、`mid`、`ctime`、`entrance`、`coop_main`、`coop_expert`、`coop_client`、`coop_appid`、`si`、`clickid`、`callback`、`clue_token`、`promotion_ad_id`、`promotion_thirdad_id`、`promotion_code`、`promotion_pt`、`adid`

### 特殊映射规则

| 平台 | 规则 |
|------|------|
| wxh5 + 腾讯广点通 | `gdt_vid` 或 `qz_gdt` → `callback` |
| tth5 | `promotionid` / `promotion_id` / `adid` → `promotion_thirdad_id` |
| 微信（wx/wxh5） | `clue_token` → `callback` |
| 抖音（tt/tth5） | `clickid` → `callback` |

---

## 四、下单 vs 普通接口对比

| 维度 | 普通接口（play/episode/profile 等） | 下单接口（createOrder） |
|------|------|------|
| 渠道参数来源 | `ajax.js` 的 `makeAdParam` 自动拼接 | `usePay.js` 的 `makePayParams` 手动拼接 |
| 传递方式 | 平铺在 query string | path 参数 + entrance 编码 + promotion JSON |
| `promotion_code` | 直接作为 query 参数 | 编码在 `entrance` 字符串内 |
| `coop_main` 等 | 直接作为 query 参数 | path 级别 + entrance 内都有 |


---

## 五、各接口渠道参数拼接位置汇总

| 接口 | 请求方式 | 渠道参数拼接位置 | 拼接函数 | 参数形式 |
|------|----------|-----------------|----------|----------|
| play（播放） | GET | `src/sdk/modules/utils/ajax.js` → `ajax.get` | `makeAdParam` | query string 平铺 |
| episode（剧集） | GET | `src/sdk/modules/utils/ajax.js` → `ajax.get` | `makeAdParam` | query string 平铺 |
| profile（详情） | GET | `src/sdk/modules/utils/ajax.js` → `ajax.get` | `makeAdParam` | query string 平铺 |
| payment（套餐列表） | GET | `src/sdk/modules/utils/ajax.js` → `ajax.get` | `makeAdParam`（兜底套餐有特殊处理） | query string 平铺 |
| createOrder（下单） | POST | `src/hooks/usePay.js` → `makePayParams` | `addCommonMountPath` + `makePayEntranceTouchuanParam` + `makePayPromotionTouchuanParam` | path 参数 + `entrance` 编码字符串 + `promotion` JSON |
| 上报（report） | GET | `src/sdk/modules/report/report.js` → `send` | `makeParam` + `addMountParams` + `reportParam.makeReportParam` | query string 平铺 |
| 抖音登录（douyin） | GET | `src/sdk/modules/utils/ajax.js` → `ajax.get` | `makeAdParam`（被排除，只返回 `h5_login_check: ''`） | 不传渠道参数 |

### makeAdParam 投流 vs 非投流参数对比

| 来源（from） | 传递的渠道参数 |
|-------------|--------------|
| `ad`（投流） | `coop_main`、`coop_client`、`coop_expert`、`coop_appid`、`si`、`promotion_ad_id`、`promotion_code`、`promotion_thirdad_id`、`ctime` |
| `tuiguang` / `appfun`（推广/橙星推） | `coop_main`、`coop_client`、`coop_expert`、`coop_appid`、`ctime` |
| `payment` 接口 + 兜底套餐 si | `coop_main: 'funshion'`、`coop_client: 'fun'`、`si` |
| `createOrder` / 抖音登录 | 不传（被排除） |
