# 支付确认弹框逻辑分析

## 概述

用户从支付宝返回后，页面通过弹框让用户确认支付结果，再轮询订单状态。三个入口触发弹框，各自的拦截条件、查询次数、后续处理存在差异。

---

## 弹框触发条件（什么时候拦截）

### mediaPage onLoad 中的判断优先级

```
onLoad
  ├── 1. checkNotifyRenew() 命中？ → notifyRenewOnLoad()
  ├── 2. URL 含 isAlipayRedirect？ → notifyOnLoad()
  └── 3. 都不命中 → 正常加载，不弹框
```

### 入口 1：`checkNotifyRenew()` → `notifyRenewOnLoad()`

拦截条件（全部满足才弹）：
- `appConfig.is_wxh5 === true`（微信 H5）
- `appConfig.os === 'ios'`
- 缓存 `renew-order` 存在且含 `order_code` 和 `payParams`

不依赖 URL 参数，只看缓存。优先级最高。

### 入口 2：`notifyOnLoad()`

拦截条件：
- URL 含 `&isAlipayRedirect=true`
- 缓存 `normal-order` 存在且含 `order_code` 和 `payParams`

两个条件都要满足。URL 参数在发起支付时通过 `history.pushState` 写入。

### 入口 3：`notifyOnShow()`（onShow 触发）

拦截条件：
- `payChecking.value === false`（onLoad 没有在查询中）
- `payment.payParams` 存在
- `payment.payParams.pay_url` 有值
- 如果是非续费订单，还要求缓存 `normal-order` 存在

这个入口针对页面没有刷新的场景（Android 或部分 iOS 情况），支付跳转回来时 JS 上下文还在，`payment.payParams` 是内存中的值。

---

## 三个入口的完整对比

| 对比项 | notifyRenewOnLoad | notifyOnLoad | notifyOnShow |
|--------|-------------------|--------------|--------------|
| 场景 | 连包订单，页面刷新后 | 普通订单，页面刷新后 | 页面未刷新，onShow 回来 |
| 平台限制 | wxh5 + iOS | 无 | 无 |
| 拦截依据 | 缓存 `renew-order` | URL `isAlipayRedirect` + 缓存 `normal-order` | 内存 `payment.payParams.pay_url` |
| 调用的确认函数 | `checkWebPaid()` | `h5ConfirmPaid()` | `checkWebPaid()` |
| 取消查询次数 | 5 次（默认值） | 5 次（默认值） | 5 次（默认值） |
| 确认查询次数 | 10 次 | 10 次 | 10 次 |
| 查询间隔 | 3 秒 | 3 秒 | 3 秒 |
| 查询成功 → renewConfirm | ✅ 调用 | ✅ 仅 `is_renew=='1'` 时调用 | ✅ 调用 |
| 查询失败 → renewConfirm | ✅ 也调用（state='success'） | ❌ 不调用 | ✅ 也调用（state='success'） |
| 删除缓存时机 | `renewConfirm` 内删 `renew-order` | 弹框前就删 `normal-order` | 支付成功后清空 `payment.payParams` |

---

## 查询次数说明

`payStatusConfirm` 函数的默认参数：

```js
const PAY_CONFIRM_COUNTS = 5   // 默认查询次数
const CHECK_TIMEOUT = 3000     // 每次间隔 3 秒
```

- 用户点"没有支付"（取消）→ 不设 `checkCount` → 走默认值 **5 次**
- 用户点"已支付"（确认）→ `checkCount = 10` → **10 次**

每次查询间隔 3 秒，最多耗时：取消 15 秒，确认 30 秒。

查询提前终止条件：
- 接口返回 `status === 'paid'` → 立即返回 `'success'`
- 接口请求失败且达到最大次数 → 返回 `'requestErr'`
- 接口返回非 `paid` 状态且达到最大次数 → 返回实际 status

---

## renewConfirm 说明

```js
// moviePay.renewConfirm
async renewConfirm(url, {order_code, state}) {
  let {success, data} = await ajax.get(url, {
    sign_code: order_code,
    state: state == 'confirm' ? 'success' : 'failed'
  })
  uni.removeStorageSync('renew-order')  // 删除缓存
  return success
}
```

注意：`checkWebPaid` 中不管订单查询成功还是失败，传给 renewConfirm 的 state 都是 `'success'`。而 renewConfirm 内部判断 `state == 'confirm'` 才传 `'success'`，否则传 `'failed'`。由于传入的是 `'success'` 而非 `'confirm'`，实际发给后端的是 `state: 'failed'`。

---

## 相关文件

| 文件 | 作用 |
|------|------|
| `src/hooks/usePay.js` | 三个弹框入口 + makePayParams |
| `src/sdk/modules/pay/index.js` | checkWebPaid / h5ConfirmPaid / payStatusConfirm |
| `src/sdk/modules/pay/product/movie.js` | renewConfirm 实现 |
| `src/pages/mediaPage/mediaPage.vue` | onLoad / onShow 中调用弹框入口 |
