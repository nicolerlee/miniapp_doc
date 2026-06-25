# 支付网关（gateway_id）来源分析

> 关注问题：连包支付的网关从哪来？`appConfig/payConfigs` 里的网关配置到底有没有被用到？

## 结论速览

| 配置项 | 是否被代码使用 | 说明 |
|---|---|---|
| `normal_pay.gateway_id` | ✅ 使用 | 普通支付/默认网关来源（画音阁、风行、星光均为 `54`） |
| `normal_pay.enable` | ❌ 未读取 | 代码里没有判断该开关 |
| `renew_pay.enable` | ✅ 使用 | 仅作为"是否走连包(续费)"的开关 |
| `renew_pay.gateway_id` | ❌ 未读取 | 连包网关实际来自 `pay_list`，此配置项是死配置 |

一句话：**普通支付网关来自 `appConfig`；连包支付网关来自 `pay_list`，不来自 `appConfig`。** `renew_pay` 配置里真正生效的只有 `enable`。

## 配置如何进入 appConfig

`src/appConfig/index.js`：

```js
import payConfig from './payConfigs/dummy_brand'  // 构建时替换成实际品牌
...
pay: payConfig[host],   // appConfig.pay = payConfigs/<brand>[host]
```

因此 `appConfig.pay.normal_pay` / `appConfig.pay.renew_pay` 即 `src/appConfig/payConfigs/<brand>.js` 中对应 host 的那两段，例如 `huayinge.js`：

```js
"wxh5": {
  "normal_pay": { "enable": true, "gateway_id": { "android": 54,   "ios": 54   } },
  "renew_pay":  { "enable": true, "gateway_id": { "android": 1038, "ios": 1038 } }
}
```

## 网关获取逻辑（唯一引用处）

全项目对网关配置的引用只有 `src/hooks/usePay.js` 的 `makePayParams`（约 366-377 行）：

```js
//网关
payParams.gateway_id = appConfig.pay.normal_pay.gateway_id[appConfig.os]   // ① 默认：普通网关
if (payment.is_renew == 1 && appConfig.pay.renew_pay.enable) {             // ② 仅用 enable 判断
  if (payment.pay_list?.length > 0) {
    const payConfig = common.getItemInArray(payment.pay_list, 'code', 'alipay')
    payParams.gateway_id = payConfig?.gateway_id || ''                     // 连包网关来自 pay_list
  } else {
    payParams.gateway_id = ''
  }
  payParams.is_renew = payment.is_renew
} else if (payParams.is_renew) {
  delete payParams.is_renew
}
```

逻辑拆解：

1. 先用 `appConfig.pay.normal_pay.gateway_id[os]` 作为默认网关（普通支付）。
2. 当判定为连包（`is_renew == 1` 且 `renew_pay.enable` 为 true）时，网关被**覆盖**为 `pay_list` 中 `code === 'alipay'` 那一项的 `gateway_id`。
3. 若 `pay_list` 为空，则 `gateway_id = ''`（空串），后续 `checkPayParamsValid` 会拦截并提示"支付功能异常，稍后再试"。

## pay_list 的来源

连包能否成功下单，关键看 `pay_list` 里是否有 `alipay` 项且 `gateway_id` 有效。两种连包入口的 `pay_list` 来源不同：

- **业务16（popType `renew`）**：`payment.sel` 来自套餐接口 `getPayment` 过滤出 `is_renew == 1` 的套餐，其 `pay_list` 由服务端返回。
- **业务17（popType `step-pay-info`）**：`pay_list` 由前端 `buildPayList(step)`（见 `business_base.js`）用微距 step 里的渠道字段现拼，例如 step 配置 `alipay:1038` 会被拼成 `[{ code: 'alipay', gateway_id: 1038 }]`。

## 注意点

- `renew_pay.gateway_id`（如 1038）虽然在四个 payConfig 文件里都有配置，但**从未被代码读取**，属于死配置。下单时取的 1038 来自 `pay_list`，与配置里的 1038 数值相同纯属巧合。
- `normal_pay.enable` 同样未被读取。
- 排查连包"支付功能异常 / 网关为空"问题时，应优先检查 `pay_list` 是否包含有效的 `alipay` 项，而不是检查 `payConfigs` 里的 `renew_pay.gateway_id`。

## 相关文件

- `src/hooks/usePay.js` — `makePayParams`，网关赋值唯一处
- `src/appConfig/index.js` — `pay: payConfig[host]`
- `src/appConfig/payConfigs/*.js` — `normal_pay` / `renew_pay` 配置
- `src/sdk/modules/combo/business/business_base.js` — `buildPayList`
