# H5 支付流程文档

> 文件：`src/view/home.vue`，下单入口：`doCreateOrder(payment_type)`

---

## 一、整体流程

```
用户点击领取按钮
  → onTapEvent('receive')
    → 校验手机号 / 协议勾选
    → 读取 activity.css_style.payment_type 决定支付方式
    → doCreateOrder(payment_type)
      → 拼接公共参数 + 各支付方式专属 params 字段
      → POST fxcreateorder（application/x-www-form-urlencoded，超时 3s）
      → 根据 payment_type 跳转到对应支付页
```

---

## 二、公共下单参数

所有支付方式共享以下参数，从 `activity.goods_list` 中匹配 `pay_agent === payment_type` 的商品项取值：

| 参数 | 来源 | 说明 |
|---|---|---|
| `activity_id` | activity | 活动 ID |
| `activity_type` | activity，默认 `'1'` | 活动类型 |
| `entrance` | activity + 挂载参数 | 透传入口参数（URL encode 的 key=value 串） |
| `service_id` | goods_item.goods_id | 商品 ID |
| `gateway_id` | goods_item.gateway_id | 支付网关 ID |
| `now_price` | goods_item.discount_price | 实付价格 |
| `key` | goods_item.goods_id | 同 service_id |
| `is_renew` | 固定 `1` | 是否连包 |
| `cl` | activity.business.code | 业务线标识 |
| `customer` | business.code（去掉 `_miniapp_`） | 客户标识 |
| `goods_type` | goods_item.goods_type | 商品类型 |
| `goods_id` | goods_item.goods_id | 商品 ID |
| `ctime` | `Date.now() / 1000` 取整 | 当前时间戳（秒） |
| `si` | 固定 `''` | session id |
| `mobile` | phone.value | 用户手机号 |
| `extra` | query.outerargs（可选） | 外部透传参数 |
| `coop_client` | CommonMount | 挂载：合作客户端 |
| `coop_main` | CommonMount | 挂载：主推广方 |
| `coop_expert` | CommonMount | 挂载：达人 ID |
| `coop_appid` | CommonMount | 挂载：小程序 appid |

---

## 三、各支付方式详解

### 3.1 支付宝（alipay）— 默认

`payment_type` 未配置时默认走支付宝。

**专属 params 字段**：无，不追加 `params`。

**跳转方式**：`jumpRenew({ schemaUrl: data.pay_url })`

- 微信 iOS：通过 `https://ulink.alipay.com/?scheme=<encoded_url>` 跳转
- 微信 Android：触发文件下载（`paps.funshion.com/v1/res/appresources`）引导用户打开支付宝
- 其他浏览器：`location.href = pay_url` 直跳

---

### 3.2 苏宁（suning）

**特殊前置流程**：苏宁支付不使用页面上的手机号输入框（`isSuningPay` 为 true 时隐藏），而是弹出三要素实名认证弹窗（`suningVerify.vue`），收集：

- `acct_name`：真实姓名
- `id_no`：身份证号（18 位）
- `bind_mob`：手机号

弹窗前端校验通过后，emit `confirm` 事件，由 `onSuningVerifyConfirm` 触发下单。

**专属 params 字段**：

```js
const b64 = (str) => btoa(String.fromCharCode(...new TextEncoder().encode(str)))
params.params = `para:id_no:${b64(id_no)};acct_name:${b64(acct_name)};bind_mob:${b64(bind_mob)}`
```

格式：`para:id_no:<base64>;acct_name:<base64>;bind_mob:<base64>`

每个字段用 `TextEncoder` 先转 UTF-8 字节再 base64 编码，支持中文姓名，避免使用已废弃的 `unescape`。

**跳转方式**：`jumpNormal(data.pay_url)` → `location.href = pay_url`

`pay_url` 实际是**连连支付**（`openweb.lianlianpay.com`）的绑卡/支付页面，直接跳转，无中间层包装。示例：
```
https://openweb.lianlianpay.com/bind-card/index?token=xxx&order_code=xxx&sign_code=xxx&lap_key=xxx&product_id=snwh0001&product_type=2
```

> ⚠️ 注意：在抖音 APP 内打开苏宁支付链接时，抖音会弹出风险警告提示，影响用户体验。

---

### 3.3 京东（jingdong）

**专属 params 字段**：无，不追加 `params`。

**跳转方式**：`jumpNormal(data.pay_url)` → `location.href = pay_url`

---

### 3.4 微信（wechat）

**专属 params 字段**：

```js
params.params = `display_account:${phone.value}`
```

透传用户手机号给微信支付网关做账号展示。

**跳转方式**：通过 fun.tv 中间页跳转，根据 `gateway_id` 选择域名：

```js
// 默认
let joinPath = 'https://m.fun.tv/pay/pay.html?payurl='
// gateway_id === '1033' 时
joinPath = 'https://gw.fun.tv/pay/pay.html?payurl='

jumpNormal(`${joinPath}${encodeURIComponent(data.pay_url)}`)
```

---

### 3.5 抖音小程序（douyin）

走小程序跳转，不走 `doCreateOrder`，直接构造 schema 跳转：

```js
// schema 方式
const schemaUrl = `snssdk1128://microapp?app_id=${appId}&start_page=<encoded_path>`
// 同时请求服务端获取 schemaV2 和 linkV2
const schemaV2Url = await getDouyinMiniPath('schemaV2', params)
const linkV2Url  = await getDouyinMiniPath('linkV2', params)
jumpDouyin({ schemaUrl, schemaV2Url, linkV2Url })
```

跳转优先级：
- 抖音浏览器内：`schemaV2Url`
- 微信 iOS：`linkV2Url`
- 微信 Android：文件下载方式触发 `schemaUrl`
- 其他：`schemaUrl`

---

### 3.6 支付宝小程序（默认非 h5Jump 场景）

```js
const miniPath = `alipays://platformapi/startapp?${activity.path}&query=${encodeURIComponent(objToString(query))}`
jumpAlipayMini({ schemaUrl: miniPath })
```

跳转优先级：
- 微信 iOS：`https://ulink.alipay.com/?scheme=<encoded_url>`
- 微信 Android：文件下载方式
- 其他：`location.href = schemaUrl`

---

## 四、跳转工具函数说明（jumpLink.js）

| 函数 | 用途 |
|---|---|
| `jumpNormal(url)` | 直接 `location.href = url` |
| `jumpRenew({ schemaUrl })` | 支付宝连包，微信内做兼容处理 |
| `jumpAlipayMini({ schemaUrl })` | 支付宝小程序，微信内做兼容处理 |
| `jumpDouyin({ schemaUrl, schemaV2Url, linkV2Url })` | 抖音小程序，多环境兼容 |

微信 Android 的兼容方案：通过触发文件下载（`paps.funshion.com/v1/res/appresources?url=<encoded>`）引导用户离开微信打开目标 App。

---

## 五、支付方式对比速查

| payment_type | 前置弹窗 | params 字段 | 跳转方式 |
|---|---|---|---|
| `alipay`（默认） | 无 | 无 | `jumpRenew` |
| `suning` | 三要素实名弹窗 | `para:id_no:<b64>;acct_name:<b64>;bind_mob:<b64>` | `jumpNormal` |
| `jingdong` | 无 | 无 | `jumpNormal` |
| `wechat` | 无 | `display_account:<phone>` | `jumpNormal`（经 fun.tv 中间页） |
| `douyin` | 无 | 不走 doCreateOrder | `jumpDouyin` |
| 支付宝小程序 | 无 | 不走 doCreateOrder | `jumpAlipayMini` |
