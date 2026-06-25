# 策略拦截逻辑分析（什么时候能播放，什么时候拦截）

## 概述

播放过程中，`useCombo` 的 `playPosUpdate` 在每次播放进度更新时调用 `applyBusiness()`，判断是否需要拦截播放并弹出付费弹窗。拦截判断分三层：身份/内容检查 → 策略有效性校验 → 试看时间检查。

---

## 拦截判断流程

```
playPosUpdate（播放进度更新）
  └── applyBusiness()
        ├── selectBusiness()          // 选择当前策略 step
        ├── baseChecking()            // 策略66专用：只检查身份+内容，不检查试看时间
        │     ├── isForceDropBase()   // 身份/内容检查
        │     └── dropInvalidPolicy() // 策略白名单校验
        ├── checking()                // 通用检查：身份+内容+试看时间
        │     ├── isForceDrop()       // 身份/内容/试看时间检查
        │     └── dropInvalidPolicy() // 策略白名单校验
        └── applyBusiness(step)       // 应用策略，弹出对应弹窗
```

---

## 第一层：身份和内容检查（isForceDrop / isForceDropBase）

### isForceDropBase（不检查试看时间）

返回 `true` 表示丢弃策略（不拦截，允许播放）：

| 条件 | 结果 | 说明 |
|------|------|------|
| `user.isVip === true` | 丢弃 ✅ | VIP 用户不拦截 |
| `episodeIsFree()` | 丢弃 ✅ | 免费内容不拦截 |
| `episodeIsNeedPay()` | 不丢弃 ❌ | 需要付费的内容，继续走拦截 |
| 其他情况 | 丢弃 ✅ | 兜底不拦截 |

### isForceDrop（检查试看时间）

在 `isForceDropBase` 基础上，对需要付费的内容增加试看时间判断：

```js
// 试看时长来自微距配置 try_dur
// frist_try_dur: 第1集试看时长（默认300秒）
// second_try_dur: 第2集及以后试看时长（默认300秒）
const tryDuration = media.episode.playIndex <= 1 ? frist_try_dur : second_try_dur

// 拦截条件：播放位置超出试看区间 [offsetPos, offsetPos + tryDuration]
const execute = position != 0 && (position > (offsetPos + tryDuration) || position < offsetPos)
return !execute  // execute=true 表示需要拦截，返回 false（不丢弃）
```

| 条件 | 结果 |
|------|------|
| VIP 用户 | 不拦截 |
| 免费内容 | 不拦截 |
| 付费内容 + 播放位置在试看区间内 | 不拦截 |
| 付费内容 + 播放位置超出试看区间 | 拦截 |
| 付费内容 + position == 0 | 不拦截（刚开始播放） |

---

## 第二层：策略白名单校验（dropInvalidPolicy → checkStepsIsAcceptable）

通过第一层后，还要校验微距返回的策略是否在白名单内。不在白名单的策略会被丢弃（step 清空），等同于不拦截。

### 白名单规则（stepCheckMap.js）

H5 端允许的 business_type（android 和 ios 一致）：

| business_type | 说明 | vip 内容 | free 内容 | fee 内容 |
|---------------|------|----------|-----------|----------|
| 6 | 半屏支付弹窗（单片+会员） | ✅ | ❌ | ❌ |
| 8 | 挽留弹窗（vip挽留） | ✅ | ❌ | ❌ |
| 11 | 免费播放 | ✅ | ✅ | ✅ |
| 16 | 连续包月弹框 | ✅ | ❌ | ❌ |
| 17 | 连续包月弹窗（套餐可配） | ✅ | ❌ | ❌ |
| 66 | 半屏支付弹窗（进入播放页立即弹出） | ✅ | ❌ | ❌ |
| 其他所有 | 广告/激励/分享等 | ❌ | ❌ | ❌ |

关键结论：
- H5 端只允许付费类策略（6/8/16/17/66）和免费播放（11）
- 所有广告/激励相关策略（1-5, 7, 9-10, 12-15, 21-29）在 H5 端全部禁用
- 付费策略只对 vip 内容生效，free 和 fee 内容不拦截

### 字段校验

白名单通过后，还会校验 step 的必要字段：

| business_type | 必要字段 | 缺失则丢弃 |
|---------------|---------|-----------|
| 8 | `package_id` + `package_price` + `leave_pop_pics` | ✅ |
| 17 | `package_id` + `package_price` + `pop_pics` | ✅ |
| 5 | 必须是电视剧频道 | ✅ |

---

## 第三层：策略选择和应用

### 选择策略组（selectBusiness）

根据当前内容类型选择 vip 策略组还是 free 策略组：

```
episodeIsVip() 或 episodeIsFee() → 使用 weiJuInfo.vip.step
其他（免费内容）                  → 使用 weiJuInfo.free.step
```

### 内容类型判断

```js
episodeIsVip: profile.isfee != 1 && episode.isvip == 1   // VIP 内容
episodeIsFee: profile.isfee == 1 && episode.isfee == 1    // 单片付费内容
episodeIsFree: 既不是 vip 也不是 fee                       // 免费内容
episodeIsNeedPay: (vip || fee) && episode.user_paid == 0  // 需要付费（未购买）
```

### 策略应用（business_type → 弹窗类型）

| business_type | 对应 Business 类 | 弹窗 |
|---------------|-----------------|------|
| 6 | Business6 | 半屏支付弹窗（paymentPop） |
| 8 | Business8 | 挽留弹窗/低价图片弹窗（retainLpPop） |
| 16/17 | Business16 | 连续包月弹框（retainLpPop） |
| 66 | Business66 | 半屏支付弹窗（进入即弹，不暂停播放） |
| 其他/兜底 | BusinessBase | 默认走 business6 逻辑 |

---

## 策略66的特殊处理

策略66 走 `baseChecking()` 而非 `checking()`，区别是不检查试看时间：

```js
// useCombo.js applyBusiness()
const applyInPreview = business_type == 66
if (applyInPreview) {
  if (!businessHandler.baseChecking()) return false  // 只检查身份+内容
  await businessHandler.applyBusiness(step)           // 试看期间就弹窗
}
if (!businessHandler.checking()) return false          // 其他策略要检查试看时间
```

所以策略66在试看期间就会弹出，但不暂停播放。

---

## 完整拦截判断总结

```
是否拦截？
  │
  ├── VIP 用户？ → 不拦截 ✅
  ├── 免费内容？ → 不拦截 ✅
  ├── 付费内容但已购买（user_paid=1）？ → 不拦截 ✅
  │
  ├── 付费内容未购买：
  │     ├── 微距策略为空？ → 走兜底 business6（半屏支付弹窗）
  │     ├── 策略不在白名单？ → 策略丢弃，不拦截 ✅
  │     ├── 策略必要字段缺失？ → 策略丢弃，不拦截 ✅
  │     │
  │     ├── 策略66？ → 不检查试看时间，直接弹窗（不暂停播放）
  │     │
  │     ├── 播放位置在试看区间内？ → 不拦截 ✅
  │     └── 播放位置超出试看区间？ → 拦截，弹出对应弹窗 ❌
  │
  └── 其他情况 → 不拦截 ✅
```

---

## 相关文件

| 文件 | 作用 |
|------|------|
| `src/hooks/useCombo.js` | 策略 hook，playPosUpdate 触发拦截判断 |
| `src/sdk/modules/combo/handler/base.js` | checking / baseChecking / selectBusiness |
| `src/sdk/modules/combo/handler/inner/movie.js` | isForceDrop / isForceDropBase / calcTryDuration |
| `src/sdk/modules/combo/businessValidCheck/index.js` | checkStepsIsAcceptable 策略校验 |
| `src/sdk/modules/combo/businessValidCheck/stepCheckMap.js` | 白名单配置 |
| `src/sdk/customize/mediaApi/api/movie.js` | episodeIsVip / episodeIsFree / episodeIsNeedPay |
