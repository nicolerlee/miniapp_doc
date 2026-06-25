# 自动播放相关逻辑分析

## 1. 配置层：按宿主环境开关

`src/appConfig/host/fun.js` 中：

```js
enableAutoPlay: { ksh5: true, tth5: true, wxh5: true }
```

不同宿主（快手H5、抖音H5、微信H5）分别控制是否启用自动播放。

---

## 2. 触发层：`onLoadedMetadata` 是自动播放的入口

视频元数据加载完成后才尝试自动播放，而不是直接依赖 `autoplay` 属性：

```js
// src/pages/mediaPage/mediaPage.vue
} else if (action == 'onLoadedMetadata') {
  // 排除 PC 端
  if (appConfig.systeminfo.deviceType != 'pc' && appConfig.webapp.enableAutoPlay) {
    data.supportAutoPlay = true
  }
  if (data.supportAutoPlay) {
    try {
      videoHook.toPlay(() => {
        console.error('mediaPage autoplay failed')
        data.supportAutoPlay = false  // 失败则关闭自动播放标志
      })
    } catch (e) {
      console.error('无法自动播放')
    }
  }
}
```

特殊处理：
- PC 端不触发自动播放
- 用 `supportAutoPlay` 标志位做状态管理
- 自动播放失败时降级，关闭标志位

---

## 3. 播放层：直接操作原生 video 元素绕过框架

### 为什么不用 uni-app 的 API？

模板里声明的是 uni-app 的 `<video>` 组件，编译成 H5 后本质是原生 `<video>` 标签加一层 uni-app 封装：

```
uni-app <video>  →  编译后  →  原生 <video> + uni-app 控制层
```

uni-app 提供的控制方式：
```js
const ctx = uni.createVideoContext('funVideo')
ctx.play()  // 没有返回值，无法感知是否被浏览器拦截
```

浏览器原生 API：
```js
const video = document.querySelector('video')
video.play()  // 返回 Promise，可以 .catch() 捕获自动播放被拦截的错误
```

浏览器会拦截自动播放并抛出错误，uni-app 封装的 `ctx.play()` 没有返回 Promise，所以根本不知道有没有被拦截。因此代码绕过 uni-app，直接操作底层原生 DOM 来拿到这个 Promise。

### 实现代码

`src/hooks/useVideo.js` 中 `toPlay()`：

```js
function toPlay(error = () => {}) {
  const video = document.querySelector('video'); // 直接获取原生video元素
  if (video) {
    video.play().then(_ => {
      console.log('toPlay success')
    }).catch(err => {
      const evt = {
        detail: {
          errMsg: {
            errorcode: -1,
            url: media.url,
            errorname: err.name,
            errormsg: err.message
          }
        }
      }
      playStatics.onPlayerError(media.episode, evt);
      error()
    });
  }
}
```

暂停则仍走 uni-app 的方式（暂停不需要感知结果）：
```js
function toPause() {
  video.toPause(videoId) // uni-app API
}
```

---

## 4. 状态层：`supportAutoPlay` 标志位的多处维护

`supportAutoPlay` 在整个生命周期内追踪自动播放的可用状态：

| 场景 | 操作 |
|------|------|
| `onLoadedMetadata`，满足条件 | 设为 `true` |
| `onLoadedMetadata`，自动播放失败 | 设为 `false` |
| `onPlay` 事件触发，配置允许 | 设为 `true`（确认浏览器真的播起来了）|
| 支付流程开始 | 设为 `false`（支付期间禁止自动播放）|
| 支付完成 / 刷新播放后 | 设为 `true`（恢复）|
| 切集 / 换媒体时 | 若为 `false` 则重置为 `true` |

---

## 5. 拦截层：弹窗出现时强制暂停

`src/sdk/modules/combo/combo.vue` 中：

```js
watch(() => data.combo.comboWhich, (newVal) => {
  // 策略66（平面支付）即使弹窗也不暂停视频，其他弹窗都暂停
  if (newVal && newVal !== data.combo.modal.planePayment66.id) videoHook.toPause()
})
```

支付弹窗弹出时暂停视频，防止付费墙弹出时视频仍在播放。策略66（平面支付）是例外，因为它平铺在页面上，不需要暂停。

---

## 6. Loading 图标显示依赖 `supportAutoPlay`

```js
// src/pages/mediaPage/mediaPage.vue
const showLoadingIcon = computed(() => {
  return data.supportAutoPlay && (
    videoHook.playState.value === videoHook.STATE.NONE
    || videoHook.playState.value === videoHook.STATE.INIT
    || videoHook.playState.value === videoHook.STATE.PREPARED
  )
})
```

只有在自动播放模式下，缓冲 loading 图标才会显示。用户手动点播时不显示。

---

## 整体流程

```
页面加载
  ↓
video 标签设置 :autoplay="true"（框架层）
  ↓
onLoadedMetadata 触发
  ↓
检查：非 PC 端 && enableAutoPlay 配置为 true
  ↓
supportAutoPlay = true
  ↓
调用 videoHook.toPlay()
  → document.querySelector('video').play()
  → 成功：onPlay 事件触发，确认 supportAutoPlay = true
  → 失败：error 回调，supportAutoPlay = false，上报错误
```

---

## 7. 为什么 H5 用 MP4 而不是 HLS 流媒体

### 结论：根据环境自动选择，H5 强制走 MP4

`src/sdk/modules/decode/decoder.js` 中的选择逻辑：

```js
// decodePlayInfo 主流程
let playinfoH264 = playinfo.filter(i => { return i.codec == 'h.264' })       // MP4
let playinfoTS   = playinfo.filter(i => { return i.codec == 'h.265_hls_ts' }) // HLS TS

item.playinfo = playinfoH264[0]; // 默认先取 H.264（MP4）

if (is_h5 || is_dev || (wxIos && manyingApi)) {
  // H5 / 开发环境 / 微信iOS：保持 H.264 MP4，不切换
} else if (playinfoTS && playinfoTS.length > 0) {
  item.playinfo = playinfoTS[0] // 原生 App 等其他环境：切换到 H.265 HLS TS
}
```

| 环境 | 格式 |
|------|------|
| H5 浏览器（`is_h5`） | H.264 MP4 |
| 微信 iOS（`wxIos`） | H.264 MP4 |
| 开发环境（`is_dev`） | H.264 MP4 |
| 原生 App 等其他环境 | H.265 HLS TS |

### 为什么 H5 不用 HLS？

HLS（`.m3u8`）的浏览器支持情况：
- Safari（iOS/macOS）：原生支持
- Android Chrome / 其他浏览器：**不原生支持**，需要引入 hls.js 等额外库来解析

MP4 则所有浏览器都原生支持，直接赋给 `<video src>` 即可播放，无需任何额外依赖。H5 场景覆盖的 Android 机型多、浏览器碎片化严重，用 MP4 是最稳妥的兼容方案。

---

## 关键文件索引

| 文件 | 作用 |
|------|------|
| `src/appConfig/host/fun.js` | `enableAutoPlay` 配置 |
| `src/pages/mediaPage/mediaPage.vue` | `onLoadedMetadata` 触发入口、`supportAutoPlay` 状态管理 |
| `src/hooks/useVideo.js` | `toPlay()` 原生 DOM 播放实现 |
| `src/sdk/modules/combo/combo.vue` | 弹窗出现时暂停视频 |
