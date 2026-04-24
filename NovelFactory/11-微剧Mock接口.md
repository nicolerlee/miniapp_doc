# 微剧Mock接口

> 基础路径: `/api/weiju-mock`  
> 用于小程序测试时模拟微剧接口返回

## GET /getBannersByBannerId — Mock获取Banner列表

**入参** (Query): ap (String, 必填, public_switch_id值)  
**出参**: 直接返回JSON数组(非Result包装)
```json
[{ "ap": "public_switch_id", "ad_list": [{ ... }] }]
```

---

## GET /getDeliverByDeliverId — Mock获取Deliver列表

**入参** (Query): ap (String, 必填, business_type值)  
**出参**: 直接返回JSON数组
```json
[{ "ap": "business_type", "ad_list": [{ ... }] }]
```

---

## GET /getBusinessTypeMockUrl — 获取business_type的mock URL

**入参** (Query): businessType (String, 必填)  
**出参**: `Result<String>` 返回完整mock URL

---

## GET /getPublicSwitchMockUrl — 获取public_switch的mock URL

**入参** (Query): publicSwitch (String, 必填)  
**出参**: `Result<String>` 返回完整mock URL

---

## GET /getWeijuTestMockUrls — 同时获取两个mock URL

**入参** (Query): businessType (String, 可选), publicSwitch (String, 可选)  
**出参**: `Result<Map<String, String>>`
```json
{
  "weiju_test_business_type": "http://ip:port/api/weiju-mock/getDeliverByDeliverId?ap=xxx",
  "weiju_test_public_switch": "http://ip:port/api/weiju-mock/getBannersByBannerId?ap=xxx"
}
```
