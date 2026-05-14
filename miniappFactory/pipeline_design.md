# 构建发布一体化 Pipeline 设计文档

## 1. 背景

当前系统中"编译"和"发布"是两个独立操作，用户需要：
1. 进入"编译小程序"，选择包 → 构建 → 等待完成
2. 再进入"发布小程序"，从构建产物中选择 → 配置 → 发布

站在用户角度，构建和发布应该是一个连贯动作。用户的真实意图是"我要把这个小程序上线"，而不是"我要先编译，再发布"。

## 2. 目标

提供 install → build → publish 三合一的 Pipeline 接口，用户一次操作即可完成从安装依赖到最终发布的全流程。

## 3. 整体架构

```
用户操作（前端）
    │
    ▼
POST /api/app/batch-pipeline
    │
    ▼
后台 Pipeline 编排器
    │
    ├── 包A: install → build → publish → ✓ completed
    ├── 包B: install → build → ✗ failed（跳过 publish）
    └── 包C: install → build → publish → ✓ completed
    │
    ▼
WebSocket 实时推送状态 + 日志
```

## 4. 接口设计

### 4.1 单个 Pipeline（install → build → publish）

```
POST /api/app/{packageId}/pipeline
Content-Type: application/json
```

**请求体**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| version | string | ✅ | 发布版本号 |
| log | string | 否 | 发布说明 |
| publishMode | string | 否 | `preview` / `publish`，默认 `preview` |

**示例**

```json
{
  "version": "1.2.0",
  "log": "修复首页加载问题",
  "publishMode": "preview"
}
```

**响应**

```json
{
  "code": 200,
  "data": {
    "taskId": "task_001"
  }
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| 400 | INVALID_PARAMS | 参数不合法 |
| 401 | UNAUTHORIZED | 未登录 |
| 403 | FORBIDDEN | 无构建或发布权限 |
| 409 | TASK_CONFLICT | 该包已有正在执行的任务 |
| 500 | PIPELINE_FAILED | 创建任务失败 |

**停止**

```
POST /api/app/{packageId}/pipeline/stop/{taskId}
```

**WebSocket 推送**

| Topic | 说明 |
|-------|------|
| `/topic/pipeline-status/{taskId}` | 阶段状态变更 |
| `/topic/pipeline-logs/{taskId}` | 实时日志 |

消息格式与批量 Pipeline 一致（见 4.3），只是粒度为单个包。

---

### 4.2 批量 Pipeline

```
POST /api/app/batch-pipeline
Content-Type: application/json
```

**请求体**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| packageIds | string[] | ✅ | 要处理的代码包 ID 列表 |
| version | string | ✅ | 发布版本号 |
| log | string | 否 | 发布说明 |
| publishMode | string | 否 | `preview`（仅预览）/ `publish`（正式发布），默认 `preview` |

**示例**

```json
{
  "packageIds": ["pkg_001", "pkg_002", "pkg_003"],
  "version": "1.2.0",
  "log": "【发布人员：张三】修复首页加载问题",
  "publishMode": "publish"
}
```

**响应**

```json
{
  "code": 200,
  "data": {
    "batchTaskId": "pipeline_20250514_001",
    "taskIds": ["task_001", "task_002", "task_003"]
  }
}
```

**错误码**

| HTTP code | errorCode | 说明 |
|-----------|-----------|------|
| 400 | INVALID_PARAMS | packageIds 为空或参数不合法 |
| 401 | UNAUTHORIZED | 未登录 |
| 403 | FORBIDDEN | 无构建或发布权限 |
| 500 | PIPELINE_FAILED | 创建 Pipeline 任务失败 |

---

### 4.3 停止批量 Pipeline

```
POST /api/app/batch-pipeline/stop/{batchTaskId}
```

**响应**

```json
{
  "code": 200,
  "message": "已发送停止命令"
}
```

停止逻辑：
- 正在 install/build 的包：终止当前进程，标记失败
- 正在 publish 的包：终止发布进程，标记失败
- 等待中的包：直接标记取消，不再执行

---

### 4.4 WebSocket 实时推送

复用现有 STOMP over SockJS 机制，新增 topic：

| Topic | 说明 |
|-------|------|
| `/topic/pipeline-status/{batchTaskId}` | 每个包的阶段状态变更 |
| `/topic/pipeline-logs/{batchTaskId}` | 每个包的实时日志 |

**状态消息格式**

```json
{
  "taskId": "task_001",
  "packageId": "pkg_001",
  "phase": "building",
  "status": "running",
  "progress": 45,
  "message": "正在编译..."
}
```

**phase 枚举**

| phase | 说明 |
|-------|------|
| `installing` | 正在安装依赖 |
| `building` | 正在构建 |
| `publishing` | 正在发布 |
| `completed` | 全部完成 |
| `failed` | 某阶段失败，已终止 |
| `cancelled` | 被用户取消 |

**日志消息格式**

```json
{
  "taskId": "task_001",
  "packageId": "pkg_001",
  "phase": "building",
  "log": "[2025-05-14 10:23:45] npm run build:tt-xingchen ...",
  "timestamp": "2025-05-14T10:23:45"
}
```

---

## 5. 后台执行逻辑

### 5.1 Pipeline 编排器伪代码

```java
public void executePipeline(String packageId, PipelineConfig config) {
    try {
        // 阶段1: 安装依赖
        updatePhase(packageId, "installing");
        TaskResult installResult = executeInstall(packageId);
        if (installResult.isFailed()) {
            markFailed(packageId, "installing", installResult.getError());
            return; // 不继续
        }

        // 阶段2: 构建
        updatePhase(packageId, "building");
        TaskResult buildResult = executeBuild(packageId);
        if (buildResult.isFailed()) {
            markFailed(packageId, "building", buildResult.getError());
            return; // 不继续
        }

        // 阶段3: 发布
        updatePhase(packageId, "publishing");
        TaskResult publishResult = executePublish(packageId, config);
        if (publishResult.isFailed()) {
            markFailed(packageId, "publishing", publishResult.getError());
            return;
        }

        // 全部成功
        markCompleted(packageId);
    } catch (CancelledException e) {
        markCancelled(packageId);
    }
}
```

### 5.2 与现有逻辑的关系

| 现有接口 | Pipeline 中复用方式 |
|----------|-------------------|
| install 逻辑 | 直接调用现有 installDependencies 服务方法 |
| build 逻辑 | 直接调用现有 buildPackage 服务方法 |
| publish 逻辑 | 直接调用现有 publishPackage 服务方法 |
| 任务队列 | 复用现有 TaskQueue，Pipeline 任务作为一个整体入队 |
| WebSocket 推送 | 新增 topic，复用现有 STOMP 基础设施 |

### 5.3 并发策略

- 每个包的 pipeline 独立执行，互不阻塞
- 受限于现有任务队列的并发数（concurrency 配置）
- 同一个包不能同时有两个 pipeline 在跑（复用现有的 per-package mutex）

---

## 6. 前端改造

### 6.1 新增页面

新增 `BatchPipeline.vue`（或命名为 `BatchBuildPublish.vue`），路由：

```js
{
  path: 'batch-pipeline',
  name: 'BatchPipeline',
  component: () => import('../views/batch/BatchPipeline.vue'),
  meta: { title: '一键构建发布' }
}
```

### 6.2 用户流程

```
步骤1: 选择小程序（复用 AppSelectionStep 组件）
    ↓
步骤2: 配置参数（版本号 + 发布模式 + 发布说明）
    ↓
步骤3: 确认预览（展示所有选中包的配置摘要）
    ↓
步骤4: 执行 & 进度展示（三阶段进度条）
```

### 6.3 进度展示设计

每个小程序显示三阶段进度：

```
┌─────────────────────────────────────────────────────────┐
│ 小程序A (抖音)                                           │
│ [安装 ✓] ──→ [构建 ✓] ──→ [发布 ████░░] 60%            │
│                                                         │
│ 小程序B (微信)                                           │
│ [安装 ✓] ──→ [构建 ✗ 失败] ──→ [发布 跳过]              │
│                                                         │
│ 小程序C (快手)                                           │
│ [安装 ███░] 等待中...                                    │
└─────────────────────────────────────────────────────────┘
```

### 6.4 复用现有组件

| 组件 | 复用方式 |
|------|---------|
| AppSelectionStep | 直接复用，传入 requiredFields 校验 |
| BatchBuildProgressStep | 参考其结构，扩展为三阶段 |
| batchBuildWebSocket | 扩展支持 pipeline topic |
| batchBuildStore | 扩展 phase 字段 |

### 6.5 首页入口调整

将首页的"批量编译"和"批量发布"合并为一个"一键构建发布"入口，原有的单独编译/发布保留作为高级选项或在详情页中提供。

---

## 7. 权限设计

### 7.1 权限点定义

| 权限点 | 说明 |
|--------|------|
| `ci:build:run` | 允许执行 install + build |
| `ci:build:stop` | 允许停止构建任务 |
| `ci:publish:preview` | 允许发布预览版（生成预览码） |
| `ci:publish:submit` | 允许正式发布上线 |

### 7.2 Pipeline 权限校验逻辑

接口不拆分，后台根据 `publishMode` 参数在不同阶段校验不同权限：

```java
// 接口入口：统一校验构建权限
requirePermission("ci:build:run");

// 进入 publish 阶段前：根据 publishMode 校验发布权限
if ("preview".equals(publishMode)) {
    requirePermission("ci:publish:preview");
} else if ("publish".equals(publishMode)) {
    requirePermission("ci:publish:submit");
}
```

### 7.3 权限矩阵

| publishMode | install + build 阶段 | publish 阶段 |
|-------------|---------------------|-------------|
| `preview` | `ci:build:run` | `ci:publish:preview` |
| `publish` | `ci:build:run` | `ci:publish:submit` |

| 操作 | 所需权限 |
|------|---------|
| 启动 Pipeline（预览模式） | `ci:build:run` + `ci:publish:preview` |
| 启动 Pipeline（正式发布） | `ci:build:run` + `ci:publish:submit` |
| 停止 Pipeline | `ci:build:stop` |
| 查看进度 | 登录即可 |

### 7.4 前端权限控制

- 用户同时拥有 `ci:build:run` + `ci:publish:submit` → 可选"仅预览"或"正式发布"
- 用户只有 `ci:build:run` + `ci:publish:preview` → 只能选"仅预览"，"正式发布"选项置灰
- 用户缺少 `ci:build:run` → 整个 Pipeline 按钮置灰，提示"无操作权限，请联系管理员"

```vue
<!-- 前端示例：发布模式选择 -->
<el-radio-group v-model="publishMode">
  <el-radio value="preview" :disabled="!hasPermission('ci:publish:preview')">
    仅预览
  </el-radio>
  <el-radio value="publish" :disabled="!hasPermission('ci:publish:submit')">
    正式发布
  </el-radio>
</el-radio-group>
```

---

## 8. 异常处理

| 场景 | 处理方式 |
|------|---------|
| install 失败 | 标记该包失败，跳过 build 和 publish |
| build 失败 | 标记该包失败，跳过 publish |
| publish 失败 | 标记该包失败 |
| 用户主动停止 | 终止当前阶段进程，标记取消 |
| WebSocket 断连 | 前端自动重连，重连后通过轮询补齐状态 |
| 服务端重启 | 任务状态持久化到数据库，重启后可恢复或标记失败 |

---

## 9. 与现有功能的关系

| 现有功能 | 保留/废弃 | 说明 |
|----------|----------|------|
| 单个编译（AutoBuild） | 保留 | 适用于只想构建不发布的场景 |
| 单个发布（AutoPublish） | 保留 | 适用于重新发布已有产物的场景 |
| 批量编译（BatchBuild） | 逐步废弃 | Pipeline 覆盖此场景 |
| 批量发布（BatchPublish） | 逐步废弃 | Pipeline 覆盖此场景 |
| 一键构建发布（Pipeline） | **新增** | 主推入口 |

---

## 10. 改造计划

### 第一阶段：后台接口（优先）

1. 新增 `PipelineService`，编排 install → build → publish
2. 新增 `/api/app/batch-pipeline` 接口
3. 新增 `/api/app/batch-pipeline/stop/{batchTaskId}` 接口
4. 新增 WebSocket topic `/topic/pipeline-status` 和 `/topic/pipeline-logs`
5. 扩展任务状态枚举，增加 phase 字段

### 第二阶段：前端页面

1. 新增 `BatchPipeline.vue` 页面
2. 扩展 `batchBuildStore`，增加 phase 状态管理
3. 扩展 `batchBuildWebSocket`，支持 pipeline topic
4. 新增三阶段进度展示组件
5. 调整首页入口卡片

### 第三阶段：收尾

1. 首页默认展示"一键构建发布"，弱化单独的批量编译/发布入口
2. 补充操作日志和审计记录
3. 性能优化和异常场景测试
