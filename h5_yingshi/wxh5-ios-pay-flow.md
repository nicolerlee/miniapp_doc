# wxh5 + iOS 支付流程分析

## 概述

wxh5 + iOS 环境下，支付需要从微信跳转到支付宝完成，涉及跨 APP 跳转。由于页面会被刷新，无法通过 JS 回调感知支付结果，因此采用缓存 + 弹框确认的兜底机制。

---

## 普通订单支付流程

### 1. 发起支付

`src/sdk/modules/pay/host/wxh5.js` 中，iOS 普通订单不直接打开支付链接，而是跳转到 transfer 中转页：

```js
if (appConfig && appConfig.os === 'ios') {
  const transferPath = `/pages/transfer/transfer?url=${encodeURIComponent(alipayUrl)}&type=jumpNormal&return_path=${encodeURIComponent(returnPath)}`
  uni.navigateTo({ url: transferPath })
}
```

### 2. transfer 中转页

`src/pages/transfer/transfer.vue`：

- 检测到 wxh5 + iOS + 微信浏览器环境 → 显示引导图，提示用户手动点击跳转支付宝
- 非微信环境 → 自动 `location.href` 跳转

```js
if (appConfig.is_wxh5 && isIos && isWeixin) {
  showGuide.value = true  // 展示引导图
  return
}
handleRedirect()  // 非微信直接跳转
```

### 3. 支付完成回到微信

transfer 页面的 `onHide` 触发，通过 `location.href` 跳回 mediaPage：

```js
onHide(() => {
  if (appConfig.is_wxh5 && isIos && isWeixin && params.value.return_path) {
    location.href = decodeURIComponent(params.value.return_path)
  }
})
```

### 4. 页面重新加载 → 弹出支付确认

mediaPage 重新加载后，`onLoad` 中检测 URL 含 `isAlipayRedirect=true` → 调用 `notifyOnLoad()` → 从缓存读取 `normal-order` → 弹出确认弹框：

```js
uni.showModal({
  title: '提示', content: '请确认支付结果',
  confirmText: '已支付', cancelText: '没有支付',
  complete: async (res) => {
    // 调用 h5ConfirmPaid 查询订单状态
  }
})
```

---

## 连包（续费）订单支付流程

### 与普通订单的区别

| 环节 | 普通订单 | 连包订单 |
|------|---------|---------|
| 缓存 key | `normal-order` | `renew-order` |
| 检测函数 | URL 含 `isAlipayRedirect` | `checkNotifyRenew()` 检查缓存 |
| 确认函数 | `notifyOnLoad()` | `notifyRenewOnLoad()` |
| 订单确认 | `h5ConfirmPaid()` | `checkWebPaid()` + `renewOrderConfirm()` |

### 触发流程

1. 创建订单成功时，缓存订单信息：

```js
// usePay.js payCallBack_ 中
if (payParams.is_renew == 1) {
  uni.setStorageSync('renew-order', { order_code, payParams, businessTAG })
}
```

2. mediaPage `onLoad` 中优先检测连包订单：

```js
if (payApi.checkNotifyRenew()) {
  payApi.notifyRenewOnLoad().then(...)
}
```

3. `checkNotifyRenew()` 判断条件：wxh5 + iOS + 缓存中存在 `renew-order`

4. `notifyRenewOnLoad()` 弹出确认弹框，用户确认后调用 `checkWebPaid` 查询订单状态

---

## 页面未刷新时的处理（onShow）

如果页面没有被刷新（Android 或部分场景），支付跳转回来时走 `onShow` → `notifyOnShow()`：

- 检测 `payment.payParams` 中是否有 `pay_url`
- 有则暂停播放，弹出确认弹框
- 查询订单状态后恢复播放

---

## 支付确认弹框触发总结

```
mediaPage onLoad
  ├── checkNotifyRenew() 命中 → notifyRenewOnLoad() → 弹框（连包订单）
  ├── URL 含 isAlipayRedirect → notifyOnLoad() → 弹框（普通订单）
  └── 都不命中 → 正常加载

mediaPage onShow
  └── payment.payParams.pay_url 存在 → notifyOnShow() → 弹框
```

---

## 相关文件

| 文件 | 作用 |
|------|------|
| `src/hooks/usePay.js` | 支付 hook，包含弹框确认逻辑 |
| `src/sdk/modules/pay/host/wxh5.js` | wxh5 支付宿主，iOS 跳 transfer 页 |
| `src/sdk/modules/pay/host/h5.js` | 通用 h5 支付宿主 |
| `src/sdk/modules/pay/index.js` | 支付入口，创建订单、查询订单状态 |
| `src/pages/transfer/transfer.vue` | 支付中转页，微信内展示引导图 |
| `src/pages/mediaPage/mediaPage.vue` | 主页面，onLoad/onShow 中触发支付确认 |
