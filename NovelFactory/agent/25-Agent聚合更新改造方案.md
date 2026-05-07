# Agent 聚合更新接口改造方案

## 背景

当前 `POST /api/agent/app/update` 是 Agent 专用接口，但实现里仍保留了通用聚合更新 DTO 和 Agent 广告 DTO 的双层适配：

- `AgentUpdateAppRequest`
- `UpdateAppConfigRequest`
- `UpdateAppConfigRequest.agentAdConfig`
- `NovelAppConfigFacade.updateConfig(appId, request, agentAdConfig, authentication)`
- `NovelAppConfigFacade.updateConfig(appId, request, authentication)`

这导致两个问题：

1. `agentAdConfig` 只是中转字段，没有业务含义。
2. 广告更新语义不一致：`reward` 缺字段会失败，`banner/feed/interstitial` 缺字段可能不会失败，但行为依赖旧值和 MyBatis-Plus null 更新策略。

目标是把 Agent 接口收敛成明确的 PATCH 语义：

- 未传的顶层配置域不更新。
- 已传的顶层配置域只更新明确传入的字段。
- 广告配置按广告类型做 PATCH。
- 删除广告配置使用独立删除接口，不混入 update。

## 当前实现判断

入口：

```java
AgentAppController.updateApp(@RequestBody AgentUpdateAppRequest request, Authentication authentication)
```

当前调用链：

```java
AgentUpdateAppRequest
  -> toUpdateAppConfigRequest()
  -> NovelAppConfigFacade.updateConfig(appId, configRequest, request.getAgentAdConfig(), authentication)
  -> request.setAgentAdConfig(agentAdConfig)
  -> NovelAppConfigFacade.updateConfig(appId, request, authentication)
  -> doUpdate(appId, request, actor)
```

第一个 `updateConfig` 重载只是包装器：

```java
public NovelAppAggregateView updateConfig(
        String appId,
        UpdateAppConfigRequest request,
        AgentAdConfigRequest agentAdConfig,
        Authentication authentication
) {
    if (agentAdConfig != null) {
        request.setAgentAdConfig(agentAdConfig);
    }
    return updateConfig(appId, request, authentication);
}
```


## 改造原则

1. Agent 接口只承认 Agent DTO。
2. `adConfig` 在 Agent 请求里就是嵌套广告配置，不再改名为 `agentAdConfig`。
3. Facade 负责 Agent 语义：`appId -> appAdId` 查询、patch merge、互斥、事务、文件同步。
4. `AdConfigService` 继续负责单广告类型的完整更新校验，不直接接收 Agent DTO。
5. 删除是删除，不塞进 update。

## 合理性评估与业务通用性

结论：这种做法合理，且是业务系统里常见的分层方式。

更准确地说，本方案采用的是：

- 外部 Agent API：意图式 PATCH。
- Facade/Application Service：读取当前状态、merge patch、控制互斥、事务和文件同步。
- Domain/Service：执行完整更新和业务校验。
- 子资源删除：独立 DELETE。
- 内部 ID：后端隐藏，Agent 只感知 `appId + adType` 这类业务标识。

### 为什么 Agent update 应该是 PATCH

`/api/agent/app/update` 是 Agent 专用接口，不是后台页面的整页表单保存。Agent 调用的本质是表达修改意图：

```json
{
  "appId": "ttxxx",
  "baseConfig": {
    "version": "1.2.3"
  }
}
```

它不应该为了改一个字段，被迫传完整应用配置。因此 `POST /api/agent/app/update` 应明确为 partial update：

- 没传的配置域不动。
- 没传的字段不动。
- 传入的非 null 值才尝试更新。

这类接口在 BFF、Agent、自动化运维、配置聚合接口里很常见。

### 为什么 Facade merge 合理

老的 `AdConfigService.updateAdConfig(UpdateAdConfigRequest)` 本质是“完整更新某一种广告配置”，要求字段完整是合理的，因为 Service 要保证数据完整性。

Agent 侧需要支持：

```json
{
  "adConfig": {
    "rewardAd": {
      "rewardCount": 5
    }
  }
}
```

因此必须有一层把当前值查出来并合并：

```java
rewardAdId = patch.rewardAdId != null ? patch.rewardAdId : current.rewardAdId;
rewardCount = patch.rewardCount != null ? patch.rewardCount : current.rewardCount;
enabled = patch.enabled != null ? patch.enabled : current.enabled;
```

这层应该放在 Facade，而不是放到底层 `AdConfigService`：

- Facade 理解 Agent PATCH 语义。
- Service 继续保持完整更新和完整校验。
- Mapper 不感知业务语义。

这样不会污染老接口，也不会让 Agent DTO 下沉到通用 Service。

### 为什么删除不放进 update

删除和更新不是同一类动作。

如果允许这种请求：

```json
{
  "adConfig": {
    "rewardAd": null
  }
}
```

语义会变得不稳定：

- 是不更新 reward？
- 是删除 reward？
- 是清空 reward？
- 是非法参数？

因此删除广告配置应使用独立接口：

```http
DELETE /api/agent/app/ad-config?appId=ttxxx&adType=reward
```

这比把 `_delete` 或 `null` 塞进 update 更清晰，也更符合子资源删除语义。

### 最终语义边界

`POST /api/agent/app/update` 只负责局部更新：

| 请求 | 语义 |
|---|---|
| 不传字段 | 不更新 |
| 传字段非 null | 更新 |
| 字符串传 `""` | 明确值，按业务规则校验；广告 ID 应非法 |
| 数字传 `0` | 明确值，合法则更新 |
| 布尔传 `false` | 明确值，合法则更新 |
| 字段传 `null` | 当前阶段等同不传，不做清空 |

广告配置：

| 请求 | 语义 |
|---|---|
| 不传 `adConfig` | 广告不动 |
| `adConfig: {}` | no-op |
| `rewardAd: { "rewardCount": 5 }` | 只改 reward 次数 |
| `rewardAd: { "enabled": false }` | 只关闭 reward |
| `rewardAd: { "rewardAdId": "" }` | 返回 400 |
| 删除 reward | 走 `DELETE /api/agent/app/ad-config?appId=...&adType=reward` |

### 必须遵守的实现约束

1. 普通 Java DTO 区分不了“字段没传”和“字段传 null”。当前阶段不支持“显式清空字段”。如果以后要支持清空，需要使用 `JsonNullable`、`JsonNode`、`Map<String, Object>`，或显式动作字段如 `clearRewardAdId: true`。
2. merge 必须在 `globalOperationGuard.runEdit(appId, ...)` 互斥区内执行，不能在锁外查旧值。
3. 广告 merge 应复用互斥区内已经查出的 `oldSnapshot.getAdConfig()`，不要在 `updateAgentAdConfig` 里再次调用 `appAdService.getAppAdConfig(appId)`。
4. Facade patch 可以不完整，但传给 `AdConfigService.updateAdConfig()` 的对象必须完整。
5. `AdConfigServiceImpl` 必须校验 merge 后的 request，不能校验数据库旧对象。
6. 删除接口也必须走同一套 appId 互斥和文件同步，不能只删数据库。
7. `paymentConfig` 暂不改成 PATCH，仍要求 `payType/enabled/gatewayAndroid/gatewayIos` 完整传入，避免扩大本次改造范围。该限制是明确技术债，需要单独排期收敛。

一句话边界：`update` 负责局部更新，`delete` 负责删除，Facade 负责把 Agent 意图转成内部完整命令。

## DTO 设计

保留 `AgentUpdateAppRequest` 作为 `/api/agent/app/update` 唯一请求 DTO：

```java
public class AgentUpdateAppRequest {
    private String appId;
    private NovelApp baseConfig;
    private AppCommonConfigDTO commonConfig;
    private UpdateAppPayRequest paymentConfig;
    private AppUIConfig uiConfig;
    private AgentAdConfigRequest adConfig;
}
```

删除或废弃：

```java
AgentUpdateAppRequest.toUpdateAppConfigRequest()
UpdateAppConfigRequest.agentAdConfig
NovelAppConfigFacade.updateConfig(String, UpdateAppConfigRequest, AgentAdConfigRequest, Authentication)
```

如果 `UpdateAppConfigRequest` 只剩 Agent update 使用，则后续可删除整个 DTO。若仍有老 Controller 调用，则保留给老接口，但 Agent 调用链不再经过它。

## Facade 入口

新增 Agent 专用入口：

```java
public void updateAgentConfig(
        String appId,
        AgentUpdateAppRequest request,
        Authentication authentication
) {
    ActorContext actor = ActorContext.fromAuthentication(resolveUserId(authentication), authentication, "web");
    log.info("updateAgentConfig 开始，appId={}, actor={}", appId, actor.summary());
    globalOperationGuard.runEdit(appId, () -> { self.doUpdateAgent(appId, request, actor); return null; });
}
```

实现时不要让 `doUpdateAgent` 和现有 `doUpdate` 平行维护两份重复逻辑。优先级如下：

1. 如果 `UpdateAppConfigRequest` 没有实际调用方，直接把现有 `doUpdate` 改成 Agent DTO 版本，不新增第二套 `doUpdateAgent`。
2. 如果必须同时保留老入口和 Agent 入口，则抽出共享私有方法，避免重复维护：
   - `updateBaseConfig(appId, baseConfig, oldSnapshot)`
   - `updateCommonConfig(appId, commonConfig)`
   - `updateUiConfig(appId, uiConfig, oldSnapshot)`
   - `updatePaymentConfig(appId, paymentConfig)`
   - `updateAgentAdConfig(appId, oldSnapshot.getAdConfig(), adConfigPatch)`
   - `syncLocalFiles(appId, latestView, touchedDomains)`

Controller 改为直接调用：

```java
novelAppConfigFacade.updateAgentConfig(
        request.getAppId(),
        request,
        authentication
);
return Result.<Void>success();
```

`doUpdateAgent` 内部仍按顶层配置域判断：

```java
if (request.getBaseConfig() != null) { ... }
if (request.getCommonConfig() != null) { ... }
if (request.getUiConfig() != null) { ... }
if (request.getPaymentConfig() != null) { ... }
if (request.getAdConfig() != null) { updateAgentAdConfig(appId, oldSnapshot.getAdConfig(), request.getAdConfig()); }
```

`POST /api/agent/app/update` 的返回契约明确为：

- 成功返回 `200`，`data` 为空。
- 如需最新聚合配置，调用 `GET /api/agent/app/detail?appId=...`。
- update 响应不回传 `NovelAppAggregateView`。

## 更新语义

### 顶层配置域

| 请求 | 语义 |
|---|---|
| 不传 `baseConfig` | 不更新基础信息 |
| `"baseConfig": null` | 不更新基础信息 |
| `"baseConfig": { "version": "1.2.3" }` | 更新版本号 |
| `"baseConfig": { "version": "" }` | 明确传空字符串，应按字段规则校验 |

`commonConfig`、`uiConfig` 同理。

### paymentConfig

短期不改支付模型，继续沿用当前完整更新：

```json
{
  "paymentConfig": {
    "payType": "normalPay",
    "enabled": true,
    "gatewayAndroid": 815,
    "gatewayIos": 815
  }
}
```

规则：

- 不传 `paymentConfig`：不更新支付配置。
- 传 `paymentConfig`：必须包含 `payType/enabled/gatewayAndroid/gatewayIos`。
- 只想改一个网关时，调用方先查详情，补齐另外两个字段。

这是本方案刻意保留的技术债：它和 Agent 接口“只传意图”的方向不完全一致。短期保留是为了控制改造范围；后续应单独把支付也改成类型级 PATCH：

```json
{
  "paymentConfig": {
    "payType": "normalPay",
    "gatewayAndroid": 900
  }
}
```

支付 PATCH 改造时同样应复用当前详情做 merge，再调用完整更新 Service。

## 广告更新设计

Agent 广告更新改为类型级 PATCH。

### 请求语义

| 请求 | 语义 |
|---|---|
| 不传 `adConfig` | 广告配置不动 |
| `"adConfig": null` | 广告配置不动 |
| `"adConfig": {}` | no-op |
| 传 `adConfig.rewardAd` | 只 patch 激励广告 |
| `rewardAd.rewardCount` 不传 | 保持原值 |
| `rewardAd.rewardCount: 0` | 明确更新为 0 |
| `rewardAd.enabled: false` | 明确关闭 |
| `rewardAd.rewardAdId: ""` | 非法 |
| `rewardAd.rewardAdId: null` | 不更新该字段 |

### 请求示例

只改激励次数：

```json
{
  "appId": "ttxxx",
  "adConfig": {
    "rewardAd": {
      "rewardCount": 5
    }
  }
}
```

关闭 banner：

```json
{
  "appId": "ttxxx",
  "adConfig": {
    "bannerAd": {
      "enabled": false
    }
  }
}
```

同时改两个广告类型：

```json
{
  "appId": "ttxxx",
  "adConfig": {
    "rewardAd": {
      "rewardCount": 5
    },
    "feedAd": {
      "enabled": true,
      "feedAdId": "feed_xxx"
    }
  }
}
```

### Facade merge 逻辑

`doUpdateAgent` 开始时已经在互斥区内查了旧快照：

```java
NovelAppAggregateView oldSnapshot = novelAppQueryFacade.queryByAppId(appId);
```

广告 merge 必须复用该快照中的 `oldSnapshot.getAdConfig()`，避免重复查询和快照不一致：

```java
private void updateAgentAdConfig(String appId, AppAdWithConfigDTO current, AgentAdConfigRequest patch) {
    if (patch == null) {
        return;
    }

    if (current == null || current.getId() == null) {
        throw new IllegalArgumentException("未找到应用的广告配置，appId=" + appId);
    }

    String appAdId = String.valueOf(current.getId());

    if (patch.getRewardAd() != null) {
        adConfigService.updateAdConfig(buildRewardUpdate(appAdId, current.getReward(), patch.getRewardAd()));
    }
    if (patch.getInterstitialAd() != null) {
        adConfigService.updateAdConfig(buildInterstitialUpdate(appAdId, current.getInterstitial(), patch.getInterstitialAd()));
    }
    if (patch.getBannerAd() != null) {
        adConfigService.updateAdConfig(buildBannerUpdate(appAdId, current.getBanner(), patch.getBannerAd()));
    }
    if (patch.getFeedAd() != null) {
        adConfigService.updateAdConfig(buildFeedUpdate(appAdId, current.getFeed(), patch.getFeedAd()));
    }
}
```

注意：不要在 `updateAgentAdConfig` 内部再次调用 `appAdService.getAppAdConfig(appId)`。一次 `oldSnapshot` 已经包含 `base/common/payment/ui/ad` 当前状态，足够支撑 merge 和失败补偿。

`reward` merge 示例：

```java
private UpdateAdConfigRequest buildRewardUpdate(
        String appAdId,
        AppAdWithConfigDTO.RewardAdConfigDetail current,
        AgentAdConfigRequest.RewardAd patch
) {
    if (current == null) {
        throw new IllegalArgumentException("未找到 reward 广告配置");
    }

    UpdateAdConfigRequest req = new UpdateAdConfigRequest();
    req.setAppAdId(appAdId);
    req.setAdType("reward");
    req.setRewardAdId(patch.getRewardAdId() != null ? patch.getRewardAdId() : current.getRewardAdId());
    req.setRewardCount(patch.getRewardCount() != null ? patch.getRewardCount() : current.getRewardCount());
    req.setIsRewardAdEnabled(patch.getEnabled() != null ? patch.getEnabled() : current.getIsRewardAdEnabled());
    return req;
}
```

### Service 校验修正

`AdConfigServiceImpl.updateAdConfig()` 当前有 bug：`interstitial/banner/feed` 校验旧对象，却更新 request。

错误模式：

```java
if (adConfig.getBannerAdId() == null || adConfig.getBannerAdId().trim().isEmpty()) {
    throw new IllegalArgumentException("banner类型的广告ID不能为空");
}
adConfig.setBannerAdId(request.getBannerAdId());
```

应统一改为校验 request：

```java
if (!StringUtils.hasText(request.getBannerAdId())) {
    throw new IllegalArgumentException("banner类型的广告ID不能为空");
}
if (request.getIsBannerAdEnabled() == null) {
    throw new IllegalArgumentException("banner类型的启用状态不能为空");
}
adConfig.setBannerAdId(request.getBannerAdId());
adConfig.setIsBannerAdEnabled(request.getIsBannerAdEnabled());
```

`reward/interstitial/banner/feed` 全部按 request 校验。

Facade merge 之后传给 Service 的一定是完整字段，所以 Service 可以继续保持完整更新模型。

## 广告删除设计

删除广告配置不放进 `POST /api/agent/app/update`。

新增 Agent 删除接口：

```http
DELETE /api/agent/app/ad-config?appId={appId}&adType={adType}
```

参数：

| 参数 | 必填 | 说明 |
|---|---|---|
| `appId` | 是 | 小程序 appId |
| `adType` | 是 | `reward` / `interstitial` / `banner` / `feed` |

示例：

```http
DELETE /api/agent/app/ad-config?appId=ttxxx&adType=reward
```

实现：

1. 校验 `appId` 和 `adType`。
2. 进入 `globalOperationGuard.runEdit(appId, ...)`。
3. 在互斥区内调用 `novelAppQueryFacade.queryByAppId(appId)` 查旧快照。
4. 从 `oldSnapshot.getAdConfig().getId()` 获取 `appAdId`。
5. 调 `adConfigService.deleteAdConfigByAppAdIdAndType(appAdId, adType)`。
6. 查最新聚合配置。
7. 调 `novelAppLocalFileOperationService.updateAdConfigLocalCodeFiles(...)` 同步文件。
8. 返回更新后的聚合配置或删除结果。

推荐返回完整聚合视图，和 update 保持一致：

```java
public Result<NovelAppAggregateView> deleteAgentAdConfig(
        @RequestParam String appId,
        @RequestParam String adType
)
```

不建议让 Agent 调老接口：

```http
GET /api/novel-ad/adConfig/deleteByAppAdIdAndType?appAdId={appAdId}&adType={adType}
```

原因：Agent 不应该知道 `appAdId`。

### 老删除接口一致性风险

当前老接口存在两个问题：

1. 删除动作用 `GET` 表达，HTTP 语义不正确，容易被缓存、预取或误调用。
2. 删除逻辑分散，后续容易出现数据库和本地文件同步行为不一致。

按当前代码看：

- `GET /api/novel-ad/adConfig/deleteByAppAdIdAndType` 删除单个广告类型后已经调用 `updateAdConfigLocalCodeFiles(...)` 同步文件。
- `GET /api/novel-ad/appAd/deleteAppAdByAppId` 删除整个 AppAd 时只删除数据库记录，没有同等文件同步逻辑。

改造时应抽出共享删除编排方法，至少保证新 Agent DELETE 和老后台删除入口复用同一套：

```java
deleteAdConfigAndSyncFiles(appId, adType)
```

如果暂时不改老接口，也要在文档和测试里明确：后台页面继续调用老 GET 接口存在 HTTP 语义债；删除整个 AppAd 的文件同步需要单独补齐。

## 支付删除设计

删除支付配置不放进 `POST /api/agent/app/update`，理由与广告删除相同：删除和更新是不同类动作，语义不应混用。

网页端已有删除入口（`GET /api/novel-pay/deleteAppPayByAppIdAndType`），Agent 侧同样需要支持。

新增 Agent 删除接口：

```http
DELETE /api/agent/app/pay-config?appId={appId}&payType={payType}
```

参数：

| 参数 | 必填 | 说明 |
|---|---|---|
| `appId` | 是 | 小程序 appId |
| `payType` | 是 | `normalPay` / `orderPay` / `renewPay` / `douzuanPay` / `imPay` / `wxVirtualPay` / `wxVirtualRenewPay` |

示例：

```http
DELETE /api/agent/app/pay-config?appId=ttxxx&payType=normalPay
```

实现：

1. 校验 `appId` 和 `payType`。
2. 进入 `globalOperationGuard.runEdit(appId, ...)`。
3. 调 `appPayService.deleteAppPayByAppIdAndType(appId, payType)`。
4. 查最新聚合配置。
5. 调 `novelAppLocalFileOperationService.deletePayConfig(...)` 同步文件。
6. 返回更新后的完整聚合视图。

返回值与广告删除保持一致：

```java
public Result<NovelAppAggregateView> deleteAgentPayConfig(
        @RequestParam String appId,
        @RequestParam String payType
)
```

不建议让 Agent 调老接口：

```http
GET /api/novel-pay/deleteAppPayByAppIdAndType?appId={appId}&payType={payType}
```

原因：老接口用 GET 表达删除，HTTP 语义不正确；且后续如果老接口逻辑变更，Agent 行为会被动受影响。

## 接口清单

### 保留

```http
POST /api/agent/app/update
```

用途：Agent 聚合 PATCH。

### 新增

```http
DELETE /api/agent/app/ad-config?appId={appId}&adType={adType}
```

用途：Agent 删除某一种广告配置。

```http
DELETE /api/agent/app/pay-config?appId={appId}&payType={payType}
```

用途：Agent 删除某一种支付配置。

### 老接口保留给后台页面

```http
POST /api/novel-ad/adConfig/update
GET  /api/novel-ad/adConfig/deleteByAppAdIdAndType
GET  /api/novel-ad/appAd/deleteAppAdByAppId
POST /api/novel-pay/updateAppPay
GET  /api/novel-pay/deleteAppPayByAppIdAndType
```

这些接口可保留，不作为 Agent 首选入口。

## 代码改动清单

### AgentAppController

1. `updateApp` 直接调用 `novelAppConfigFacade.updateAgentConfig(...)`。
2. 新增 `DELETE /ad-config`。
3. 新增 `DELETE /pay-config`。
4. 不再调用 `request.toUpdateAppConfigRequest()`。

### NovelAppConfigFacade

1. 删除 `updateConfig(String, UpdateAppConfigRequest, AgentAdConfigRequest, Authentication)`。
2. 新增 `updateAgentConfig(String, AgentUpdateAppRequest, Authentication)`。
3. 不平行维护两套 `doUpdate`。无旧调用方时改造现有 `doUpdate`；有旧调用方时抽出共享私有方法。
4. 新增或改造 `updateAgentAdConfig(appId, currentAdConfig, adPatch)`，其中 `currentAdConfig` 来自 `oldSnapshot.getAdConfig()`。
5. 新增四个 merge 方法：
   - `buildRewardUpdate`
   - `buildInterstitialUpdate`
   - `buildBannerUpdate`
   - `buildFeedUpdate`
6. 新增 `deleteAgentAdConfig(...)` 或由 Controller 组合调用。
7. 新增或抽出 `deleteAdConfigAndSyncFiles(appId, adType)`，供 Agent DELETE 和后台删除复用。
8. 新增 `deleteAgentPayConfig(...)` 或由 Controller 组合调用。
9. 新增或抽出 `deletePayConfigAndSyncFiles(appId, payType)`，供 Agent DELETE 和后台删除复用。

### AgentUpdateAppRequest

1. 删除 `toUpdateAppConfigRequest()`。
2. 删除 `getAgentAdConfig()`。
3. 保留 `private AgentAdConfigRequest adConfig;`。

### UpdateAppConfigRequest

1. 删除 `agentAdConfig` 字段。
2. 若无调用方，删除整个 DTO。
3. 若仍有调用方，只保留老接口所需字段。

### AdConfigServiceImpl

1. 修正 `interstitial/banner/feed` 校验旧对象的问题，但必须先排查老接口调用方是否依赖不完整 request。
2. 所有类型统一校验 request。
3. 保持完整更新，不实现 Agent PATCH。

## 兼容性

对 Agent 调用方：

- `POST /api/agent/app/update` 路径不变。
- `adConfig` 入参结构不变。
- 广告更新能力变强：可以只传要改的字段。
- update 成功响应明确为 `Result<Void>`；如需最新配置，更新后单独调 detail。
- 删除广告改走新的 `DELETE /api/agent/app/ad-config`。

对后台页面：

- `/api/novel-ad/*` 老接口不删除。
- 原有创建、更新、删除逻辑继续可用。
- 修复校验 bug 后，传不完整广告更新请求会更早失败。因此修复前必须排查后台页面和脚本是否会发送不完整 request。
- 老 `GET /api/novel-ad/adConfig/deleteByAppAdIdAndType` 当前已有文件同步，但 HTTP 语义仍是技术债。
- 老 `GET /api/novel-ad/appAd/deleteAppAdByAppId` 删除整个 AppAd 时需要补齐文件同步，否则可能出现数据库和本地文件不一致。

## 测试用例

### 聚合 update

1. 只传 `appId`：成功返回 `200`，不改任何字段，`data` 为空。
2. 只传 `baseConfig.version`：只更新版本号。
3. 只传 `commonConfig.douyinAppToken`：只更新抖音 token。
4. 只传 `uiConfig.mainTheme`：只更新主色。

### 广告 PATCH

1. 只传 `rewardAd.rewardCount=5`：保留原 `rewardAdId/enabled`，只改次数。
2. 只传 `rewardAd.enabled=false`：保留原 `rewardAdId/rewardCount`，只关闭。
3. 只传 `bannerAd.enabled=false`：保留原 `bannerAdId`，只关闭。
4. 传 `bannerAd.bannerAdId=""`：返回 400。
5. 传 `rewardAd.rewardCount=-1`：返回 400。
6. 传不存在的广告类型对应配置：返回 400。

### 广告删除

1. 删除存在的 `reward`：数据库删除对应 `ad_config`，文件同步，返回最新聚合配置。
2. 删除不存在的 `reward`：返回 400 或业务错误。
3. 删除非法 `adType`：返回 400。
4. 删除时有编辑互斥：返回 409。

### 支付删除

1. 删除存在的 `normalPay`：数据库删除对应支付配置，文件同步，返回最新聚合配置。
2. 删除不存在的 `normalPay`：返回 400 或业务错误。
3. 删除非法 `payType`：返回 400。
4. 删除时有编辑互斥：返回 409。

### 文件同步

1. 广告 PATCH 后检查本地广告配置文件已更新。
2. 广告删除后检查本地广告配置文件移除对应类型。
3. 文件同步失败时执行 rollbackActions。

## 落地顺序

1. 排查 `/api/novel-ad/adConfig/update` 的后台页面、脚本、测试调用方，确认是否都传完整 request。
2. 抽出 Facade 共享私有方法，避免 `doUpdate` 和 `doUpdateAgent` 形成两套重复逻辑。
3. 新增 Agent Facade 入口 `updateAgentConfig`，Controller 切到该入口。
4. 实现 `updateAgentAdConfig(appId, oldSnapshot.getAdConfig(), patch)`，复用互斥区内旧快照做 merge。
5. 为 Agent 广告 PATCH 补测试，覆盖只传一个字段、传空字符串、传 `false`、传 `0`。
6. 新增 `DELETE /api/agent/app/ad-config`，并抽出删除+文件同步共享逻辑。
7. 新增 `DELETE /api/agent/app/pay-config`，并抽出删除+文件同步共享逻辑。
8. 补老 GET 删除接口一致性测试，确认单类型删除和整 AppAd 删除的文件同步行为。
8. 在确认老调用方兼容后，再修 `AdConfigServiceImpl.updateAdConfig()` 校验旧对象的 bug。
9. 删除无调用方的包装方法和 DTO 中转字段。

## 最终目标

`/api/agent/app/update` 变成真正的 Agent PATCH 接口：

- 调用方只传意图。
- 后端负责补齐旧值、校验、同步文件。
- Agent 不感知 `appAdId`。
- 删除广告用独立 DELETE。
- 删除支付用独立 DELETE。
- Facade 里不再出现 `agentAdConfig` 中转字段。


## 当前调用链（完整）

```
HTTP POST /api/agent/app/update
    ↓
AgentAppController.updateApp(AgentUpdateAppRequest, Authentication)
    ↓
request.toUpdateAppConfigRequest()          // AgentUpdateAppRequest 转成 UpdateAppConfigRequest
    ↓
novelAppConfigFacade.updateConfig(appId, configRequest, request.getAgentAdConfig(), authentication)
    ↓  // 三参数重载，只是个包装器
    request.setAgentAdConfig(agentAdConfig)
    ↓
novelAppConfigFacade.updateConfig(appId, request, authentication)
    ↓  // 真正干活的
    globalOperationGuard.runEdit(appId, () -> self.doUpdate(...))
```
