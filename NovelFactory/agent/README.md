# NovelAppManagerServer API 文档索引

> 统一返回格式: `{ "code": 200, "message": "提示信息", "data": {} }`  
> 认证接口请携带请求头: `Authorization: Bearer <token>`（JWT 认证）  
> Agent 接口请携带请求头: `X-API-Key: <api_key>`（API Key 认证）  
> 权限角色: ROLE_0=研发, ROLE_1=产品, ROLE_2=测试

## 文档列表

| 文件 | 模块 | 基础路径 |
|------|------|----------|
| [01-认证接口](01-认证接口.md) | 登录/注册/用户管理 | `/api/novel-auth` |
| [02-小说应用管理](02-小说应用管理.md) | 应用CRUD/版本管理 | `/api/novel-apps` |
| [03-创建小程序](03-创建小程序.md) | 异步创建小程序 | `/api/novel-create` |
| [04-构建小程序](04-构建小程序.md) | 单个/批量构建 | `/api/novel-build` |
| [05-发布小程序](05-发布小程序.md) | 发布/预览/二维码 | `/api/novel-publish` |
| [06-广告管理](06-广告管理.md) | 广告配置CRUD | `/api/novel-ad` |
| [07-通用配置](07-通用配置.md) | 应用通用配置 | `/api/novel-common` |
| [08-支付管理](08-支付管理.md) | 支付配置CRUD | `/api/novel-pay` |
| [09-UI配置管理](09-UI配置管理.md) | 主题/样式配置 | `/api/novel-ui` |
| [10-微剧管理](10-微剧管理.md) | Banner/Deliver管理 | `/api/novel-weiju` |
| [11-微剧Mock接口](11-微剧Mock接口.md) | 测试用Mock接口 | `/api/weiju-mock` |
| [12-代码同步](12-代码同步.md) | Git操作/npm安装 | `/api/code-sync` |
| [13-数据库导出](13-数据库导出.md) | SQL导出/提交 | `/api/database` |
| [14-操作日志](14-操作日志.md) | 操作记录查询 | `/api/op-log` |
| [15-预览管理](15-预览管理.md) | 预览配置CRUD | `/api/preview` |
| [16-任务队列](16-任务队列.md) | 队列状态/调试 | `/api/task-queue` |
| [17-系统配置与工具箱](17-系统配置与工具箱.md) | 系统配置/发版检查/资源查找 | `/api/config` `/api/novel-toolbox` `/api/test` |
| [18-AI接口](18-AI接口.md) | AI对话/AI应用管理 | `/api/ai` `/api/fun-ai/app` |
| [19-异步任务机制说明](19-异步任务机制说明.md) | 创建/构建/发布的异步原理、WebSocket通知、队列机制 | - |
| [20-Agent接口](20-Agent接口.md) | Agent应用管理/构建发布/任务查询/聚合查询/API Key管理 | `/api/agent` `/api/novel-apps` `/api/novel-auth/api-keys` |
