# 公众号投流 H5 全链路文档

## 概述

公众号投流场景下，用户从微信公众号菜单/推文进入 H5 页面，经过授权登录后跳转到阅读（观看）页。整个流程涉及 wxAuthPage 授权页和 mediaPage 播放页两个核心页面。

## 链路流程

```
公众号菜单/推文
  ↓ 带 union_id 等参数
wxAuthPage（授权页）
  ↓ 三种情况分支
  ├─ 情况0：已登录且有 union_id 缓存 → 直接调 callback 接口 → 跳 mediaPage
  ├─ 情况1：URL 带完整四参数（union_id/user_id/token/open_id）→ 静默登录 → 跳 mediaPage
  └─ 情况2：无授权参数 → 展示引导页 → 用户点击授权 → 微信 OAuth → 回调回情况1
```

## 授权流程详解

### 1. 发起授权

- 接口：`puser.funshion.com/v2/oauth/wxpublicno_silence`
- 方式：页面跳转（非 AJAX），`location.href` 跳到波塞冬授权地址
- 参数：`u`（回调地址）、`app_code`、`pt`
- 微信 OAuth 完成后，带 `union_id`、`user_id`、`token`、`open_id`、`code` 重定向回 wxAuthPage

### 2. 静默登录（授权回调）

- 入口：`processWxH5AuthCallback(options)`
- 检查 `options.code === '200'` 后调 `wxH5OpenidLogin`
- 登录接口：`appConfig.getUserinfo`，传入 `user_id`、`token`、`open_id`
- 登录成功后将 `union_id` 写入 pinia store（持久化到 localStorage）

### 3. 获取跳转参数

- 接口：`appConfig.wxh5AdCallback`（`v1/report/wxh5adcallbackquery`）
- 请求参数：`cl`（miniapp 版本）、`union_id`、`receiver=jlgg_wx`
- 返回：`data.data` 为带 query 的 URL 字符串，包含 mid、eid、coopCode、si、fudid 等

### 4. 参数过滤与跳转

- 用 `adKey.keys` 白名单过滤返回参数，只保留合法字段
- 白名单：coopCode、ctime、microapp_id、popularizeId、si、clickid、callback、clue_token、promotion_ad_id、promotion_thirdad_id、promotion_code、mid、promotion_id、promotionid、promotion_pt、adid、gdt_vid、qz_gdt、third_account_id、third_set_id、third_maccount_id、source
- 额外保留：mid、eid
- `fudid` 不拼到 URL，而是写入 `localStorage`（`uni.setStorageSync('fudid', fudid)`）
- 跳转方式：`window.location.replace`（替换历史记录，防止回退到授权页）

## union_id 存储

- 存储位置：pinia store → localStorage（key: `userInfo`）
- 写入时机：`processWxH5AuthCallback` 授权成功后
- 清除时机：用户退出登录（`user.clear()`）或浏览器缓存被清
- 再次进入 wxAuthPage 时，若已登录且有 union_id 缓存，直接跳播放页，不展示引导页

## 上报逻辑

- `source=h5_gzh` 的公众号渠道不做波塞冬 submit 上报
- 判断逻辑：`getMountType(appConfig)` 检查 `original_source`，返回 `adGzhH5` 时跳过 `report_submit`
- `original_source` 在挂载参数解析时保存原始 source 值，之后 source 统一改为 `h5`

## 跳转小程序（兜底/获取权益后）

- 优先调 `createSchema` 接口获取微信 URL Scheme
- 失败降级：`weixin://dl/business/?appid=xxx&path=xxx&query=xxx`
- mediaPage 跳小程序时携带 mid、eid 及 h5JumpParams 中的广告参数（coopCode、si、promotion_ad_id 等）
- wxAuthPage 兜底跳小程序固定跳 `pages/homePage/homePage`

## 关键文件

| 文件 | 职责 |
|------|------|
| `src/pages/wxAuthPage/wxAuthPage.vue` | 授权页，处理三种进入情况 |
| `src/hooks/useUser.js` | 用户登录逻辑，wxH5LoginRedirectTo / processWxH5AuthCallback |
| `src/sdk/modules/user/host/wxh5.js` | 微信 H5 授权跳转实现 |
| `src/store/user.js` | 用户状态持久化（pinia + localStorage） |
| `src/sdk/modules/mount/resolver/adKey.js` | 广告参数白名单、getMountType 判断 |
| `src/sdk/modules/mount/resolver/h5Jump.js` | H5 跳转参数解析 |
| `src/sdk/modules/report/report.js` | 上报逻辑，submit 上报过滤 |
| `src/sdk/modules/utils/common.js` | getMiniappSchemaUrl 等工具方法 |
| `src/appConfig/host/fun.js` | fun 品牌配置，含 wxH5Auth / wxh5AdCallback 等接口地址 |
