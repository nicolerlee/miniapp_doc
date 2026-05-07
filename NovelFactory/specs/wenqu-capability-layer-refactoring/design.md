# 设计文档

## 概述

本次改造只解决三件事：

1. 统一能力层入口，给 Skill / AI 一个稳定的聚合接口
2. 补齐重操作互斥，先把创建、编辑、构建、发布的冲突收住
3. 明确数据库是真相源，本地文件只是投影

不做的事：

1. 不重写现有 Controller 体系
2. 不重构构建/发布队列实现
3. 不在本次引入 App 级分布式锁体系
4. 不为了“架构好看”删除大量现有类

原则很简单：在现有代码上加一层薄适配，优先满足需求和兼容性。

## 设计目标

### 目标

- 保持现有网站接口、请求结构、响应结构不变
- 新增统一查询与聚合编辑接口，给外部调用方使用
- 所有写操作都能拿到明确的 `ActorContext`
- 创建、编辑、构建、发布之间具备可解释的互斥规则
- 读操作全部走数据库聚合，不依赖本地文件

### 非目标

- 不把能力层彻底改造成 DDD / CQRS
- 不在本次统一所有任务模型
- 不实现乐观锁
- 不实现跨 App 并发优化

## 现状约束

从代码看，当前后端已经具备这些基础：

- 创建链路：`NovelAppCreateController` -> `CreateNovelAppUseCase` -> `NovelAppCreationService`
- 创建任务控制：`CreateNovelTaskManager`
- 构建/发布队列：`TaskQueueManager`
- 分域配置编辑 Controller 已存在
- 文件写入服务已经分层：`NovelAppLocalFileOperationService`、`NovelAppResourceFileService`、各类 `*ConfigFileOperationService`

这意味着本次设计应当采用“增量改造”，而不是把创建链路全部推翻重来。

## 总体方案

### 分层

改造后的结构保持现有主体不动，只新增一个轻量能力编排层：

```text
前端网站 / Skill / AI
        |
        v
  Controller（保留现有接口 + 新增聚合接口）
        |
        v
  Capability Facade / Orchestrator（新增薄层）
        |
        +-- 现有 Domain Service / UseCase
        |      - CreateNovelAppUseCase
        |      - NovelAppCreationService
        |      - NovelAppService
        |      - AppCommonConfigService / AppPayService / AppUIConfigService / AppAdService
        |
        +-- Operation Guard（新增）
        |
        +-- Task Managers（保留）
               - CreateNovelTaskManager
               - TaskQueueManager
```

核心判断：

- 创建链路不删 `UseCase`，只在其前后补足互斥和 ActorContext
- 聚合查询、聚合编辑新增 Facade，避免 Controller 拼装过重
- 构建/发布继续复用 `TaskQueueManager`

## 核心设计

### 1. ActorContext 统一

#### 目标

把“谁发起了写操作”从隐式信息变成显式上下文。

#### 现状

当前 `ActorContext` 只有：

- `userId`
- `username`
- `source`

这不足以支持需求里的角色校验和任务审计。

#### 改造

将 `ActorContext` 扩展为：

```java
public class ActorContext {
    private final Long userId;
    private final String username;
    private final List<String> roles;
}
```

处理规则：

- `roles` 从 `Authentication` 提取
- 系统级调用允许 `userId == null`
- 但系统级调用必须带 `ROLE_0`

保留策略：

- 可以暂时保留 `source` 字段一段时间，避免影响现有调用链
- `summary()` 输出以 `roles` 为准，`source` 仅作为兼容字段

结论：

- 本次不要求一步删除 `source`
- 先把 `roles` 加进去，让新旧调用都能工作

### 2. 能力编排层

新增两个薄服务，避免 Controller 直接写大量聚合逻辑。

#### `NovelAppQueryFacade`

职责：

- 解析查询条件
- 查询 `NovelApp`
- 聚合 common / pay / ui / ad / build / publish 信息
- 计算 `permissionInfo`

#### `NovelAppConfigFacade`

职责：

- 接收聚合配置更新请求
- 做互斥检查
- 做权限检查
- 顺序调用现有分域 service 更新数据库
- 数据库成功后同步本地文件
- 失败时执行补偿

这两个 Facade 只负责编排，不承载底层持久化逻辑。

### 3. 全局互斥控制

#### 目标

按需求实现“全局互斥，但不过度设计”。

#### 结论

新增 `GlobalOperationGuard`，但只做三类状态检查：

1. 创建中
2. 编辑中
3. 构建/发布运行中

其中：

- 创建中：复用 `CreateNovelTaskManager`
- 构建/发布运行中：复用 `TaskQueueManager`
- 编辑中：新增一个极轻量内存态 `EditingOperationTracker`

原因：

- 需求明确要求“编辑操作执行时，创建拒绝，构建/发布等待”
- 当前系统没有编辑态追踪，必须补一个最小实现
- 没必要上 Redis，也没必要做复杂锁管理

#### 组件草图

```java
@Component
public class GlobalOperationGuard {

    public void checkCanStartCreate();

    public void checkCanStartEdit(String appId);

    public void beforeEnqueueBuildOrPublish(String appId);

    public <T> T runEdit(String appId, Supplier<T> action);
}
```

再加一个编辑追踪器：

```java
@Component
public class EditingOperationTracker {
    boolean tryStart(String appId);
    void finish(String appId);
    boolean hasAnyEditing();
    boolean isEditing(String appId);
}
```

#### 互斥规则落地

创建：

- 若有构建/发布运行中，拒绝
- 若有编辑进行中，拒绝
- 若已有创建任务，拒绝

编辑：

- 若有构建/发布运行中，拒绝
- 若有创建进行中，拒绝
- 若同一时刻已有编辑进行中，拒绝

构建/发布：

- 始终允许入队
- 真正开始执行前检查：
  - 若有创建进行中，则继续等待
  - 若有编辑进行中，则继续等待
- 只有在“无创建、无编辑、当前环境无运行任务”时，才从队列拉起执行

这比“提交时直接拒绝构建/发布”更符合需求原文，因为需求明确说构建/发布应加入队列等待。

### 4. 创建能力设计

创建链路继续保留现有主干：

```text
NovelAppCreateController
    -> CreateNovelAppCommandFactory
    -> CreateNovelAppUseCase
    -> NovelAppCreationService
```

改造点只放在边界：

#### Controller 层

- 从 `Authentication` 构建带 `roles` 的 `ActorContext`
- 保持 `/api/novel-create/createNovelApp` 不变

#### UseCase 层

- 新增互斥检查
- 新增同名同平台创建检查
- 创建任务时把 `ActorContext` 写入任务记录或任务日志

#### 创建过程

创建执行仍然遵循：

1. 先数据库
2. 后文件
3. 文件失败后执行 rollbackActions
4. 补偿数据库状态

#### 同名同平台并发创建

按需求，需要增加：

`lock:create:{platform}:{appName}`

但本次不做完整分布式锁框架，只在创建入口加一个创建幂等/互斥封装：

- 单机阶段：先用 JVM 级锁或 DB 唯一校验兜底
- 如果项目已接 Redis，再把该锁替换成 Redis 分布式锁

设计上留出接口：

```java
public interface AppCreationLockService {
    <T> T withCreateLock(String platform, String appName, Supplier<T> action);
}
```

先定义边界，不展开实现细节。

### 5. 统一查询接口

新增：

`GET /api/novel-apps/query`

#### 查询参数

唯一字段：

- `appId`
- `appCode`
- `cl`
- `buildCode`

非唯一字段：

- `customer`
- `platform`
- `appName`

#### 设计原则

- 参数全部可选，但至少传一个
- 同时传多个参数时，按 AND 组合过滤
- 若包含唯一字段且只命中一个，返回单对象
- 其余情况返回列表

#### 返回模型

```java
public class NovelAppAggregateView {
    private NovelApp baseConfig;
    private AppCommonConfig commonConfig;
    private AppPay payConfig;
    private AppUIConfig uiConfig;
    private AppAd adConfig;
    private BuildInfo buildInfo;
    private PublishInfo publishInfo;
    private SyncStatus syncStatus;
    private PermissionInfo permissionInfo;
}
```

注意：

- `permissionInfo` 是需求明确要求的，不应遗漏
- `syncStatus` 也在需求里，不应只停留在 build / publish

#### 查询实现

1. 查询 `novel_app` 主表，拿到 app 集合
2. 对每个 app 聚合读取分域配置
3. 严格模式：任一域失败则整个请求失败

先不做并发 fan-out 优化，先保证正确性。只有在列表查询量大时，再考虑批量 SQL 优化。

### 6. 聚合配置更新接口

新增：

`PUT /api/novel-apps/{appId}/config`

#### 请求模型

```java
public class UpdateAppConfigRequest {
    private NovelApp baseConfig;
    private AppCommonConfigDTO commonConfig;
    private UpdateAppPayRequest payConfig;
    private AppUIConfig uiConfig;
    private UpdateAdConfigRequest adConfig;
}
```

#### 执行流程

1. 构建 `ActorContext`
2. `GlobalOperationGuard.checkCanStartEdit(appId)`
3. 校验权限
4. 读取旧配置快照
5. 逐域更新数据库
6. 同步本地配置文件
7. 若文件失败，用旧快照补偿数据库
8. 释放编辑态

#### 为什么不用事务包住所有文件操作

因为文件系统不支持数据库事务。

因此采用：

- 数据库写入
- 文件同步
- 失败补偿

这与现有创建链路保持一致，也最现实。

### 7. 构建/发布队列适配

`TaskQueueManager` 是已有能力，本次不重写。

只补两个能力：

1. 对外暴露“当前是否可执行重任务”的状态方法
2. 在拉起下一个任务前检查创建/编辑状态

建议新增：

```java
public boolean hasRunningTask();
public boolean hasQueuedTask();
public boolean canStartNextHeavyTask();
```

其中 `canStartNextHeavyTask()` 内部依赖：

- `!createNovelTaskManager.hasRunningTask()`
- `!editingOperationTracker.hasAnyEditing()`
- 当前环境运行槽为空

这能保证“构建/发布可以先排队，但启动时必须遵守全局互斥”。

### 8. 删除能力

删除不新增新链路，仍走现有 `NovelAppController.deleteNovelApp()`。

只补检查：

1. 应用是否存在
2. 该应用是否有关联的创建/构建/发布任务
3. 当前是否有编辑进行中

因为本次采用全局互斥最简方案，所以删除也应视为重操作：

- 执行前必须进入互斥区
- 执行后释放互斥态

如果当前代码改造成本过高，删除能力可以作为第二阶段落地，但设计上要明确它属于重操作，不应遗漏。

### 9. 任务状态统一

现有创建任务状态接口保留：

- `/api/novel-create/taskStatus`

本次不强行统一所有任务接口，但要把状态模型对齐到需求语义：

- `PENDING`
- `RUNNING`
- `SUCCEEDED`
- `FAILED`
- `CANCELLED`

兼容策略：

- 旧字段继续保留，如 `finished`、`success`
- 新状态字段优先使用标准枚举语义

这一步是“模型对齐”，不是“接口重写”。

## 数据一致性策略

### 基本原则

- 数据库是真相源
- 本地文件是数据库投影
- 查询优先读数据库
- 文件写入失败要显式补偿，不允许悄悄吞掉

### 写路径策略

创建：

1. 数据库插入应用及配置记录
2. 生成本地代码与资源文件
3. 失败时执行 rollbackActions
4. 更新数据库状态为失败

编辑：

1. 读取旧快照
2. 更新数据库
3. 更新文件
4. 文件失败时用旧快照回补数据库

删除：

1. 更新数据库状态为 `DELETING` / `DELETED`
2. 清理文件和资源
3. 失败时记录错误并按状态机补偿

## 接口设计

### 新增接口

#### 统一查询

```http
GET /api/novel-apps/query?appId=xxx
GET /api/novel-apps/query?customer=xxx&platform=douyin
```

#### 聚合更新

```http
PUT /api/novel-apps/{appId}/config
```

### 保持不变的接口

- `POST /api/novel-create/createNovelApp`
- `GET /api/novel-create/taskStatus`
- 现有各分域查询与更新接口
- `POST /api/novel-build/build`
- `POST /api/novel-build/batch-build`
- `POST /api/novel-publish/publish`
- `POST /api/novel-publish/batch-publish`

## 组件清单

### 新增组件

- `GlobalOperationGuard`
- `EditingOperationTracker`
- `NovelAppQueryFacade`
- `NovelAppConfigFacade`
- `NovelAppAggregateView`
- `UpdateAppConfigRequest`
- `PermissionInfo`
- `SyncStatus`
- `AppCreationLockService`（接口级预留）

### 改造组件

- `ActorContext`
- `CreateNovelAppCommandFactory`
- `CreateNovelAppUseCase`
- `TaskQueueManager`
- `NovelAppController`
- `NovelAppCreateController`
- 各配置域 Controller

### 暂不删除的组件

- `CreateNovelAppUseCase`
- `CreateNovelAppUseCaseImpl`
- `CreateNovelAppCommand`
- `CreateNovelAppCommandFactory`

原因：

- 它们已经接在真实代码链路上
- 本次目标是能力层收口，不是内部大清洗
- 等接口与互斥稳定后，再考虑第二阶段收缩类层次

## 实施顺序

按最小风险拆成四步：

1. 扩展 `ActorContext`，把 `roles` 打通
2. 增加 `GlobalOperationGuard` + `EditingOperationTracker`，先把互斥补齐
3. 新增 `/api/novel-apps/query` 与聚合 DTO / Facade
4. 新增 `/api/novel-apps/{appId}/config`，复用现有分域更新服务

这样做的原因：

- 第 1、2 步先解决系统性风险
- 第 3、4 步再暴露新能力接口
- 每一步都能独立测试，不会把改动缠成一团

## 测试重点

### 互斥

- 创建执行中，编辑被拒绝
- 编辑执行中，创建被拒绝
- 创建执行中，构建进入队列但不启动
- 编辑执行中，发布进入队列但不启动
- 构建运行中，创建被拒绝
- 发布运行中，编辑被拒绝

### 一致性

- 创建时文件失败，数据库状态进入失败态且无脏文件残留
- 编辑时文件失败，数据库回补为旧值
- 查询时某配置域失败，整个聚合请求失败

### 兼容性

- 现有网站接口 URL、参数、响应结构不变
- 创建任务轮询行为不变
- 构建/发布 WebSocket 行为不变

## 风险与取舍

### 风险 1：编辑态是单机内存实现

这是有意取舍。

原因：

- 需求要求的是“先解决冲突”
- 当前代码显然也以单机本地状态为主
- 上 Redis 锁会把本次范围直接拉大

结论：

- 本次先用单机内存态
- 后续若系统扩容为多实例，再把 `EditingOperationTracker` 替换成分布式实现

### 风险 2：聚合查询可能产生多次数据库访问

这是可以接受的第一版成本。

原因：

- 当前首要目标是统一入口与正确性
- 性能问题可以在稳定后做批量查询优化

### 风险 3：任务状态模型暂时存在新旧并存

这也是可接受的过渡状态。

原因：

- 强推一次性统一，兼容风险高
- 先把语义统一，再慢慢收口字段

## 最终结论

本次设计采用“保留现有主体 + 增加薄编排层 + 补齐互斥”的方案。

关键点只有四个：

1. `ActorContext` 补 `roles`
2. 新增 `GlobalOperationGuard` 和 `EditingOperationTracker`
3. 新增统一查询接口
4. 新增聚合配置更新接口

其余能力全部尽量复用现有代码，不做大拆，不追求一步到位。
