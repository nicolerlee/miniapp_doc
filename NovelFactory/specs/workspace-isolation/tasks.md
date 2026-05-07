# Implementation Plan: Workspace Isolation

## Overview

基于 Git Worktree 实现多用户工作空间隔离方案。后端使用 Java Spring Boot + MyBatis Plus，前端使用 Vue3。实现按照从底层基础设施到上层业务逻辑的顺序推进，先建立数据模型和 Git 操作封装，再逐步构建各业务服务，最后完成前端交互和集成联调。

## Tasks

- [x] 1. 数据库表结构与基础实体
  - [x] 1.1 创建数据库迁移脚本，包含 workspace、change、notification 三张表
    - 在 NovelAppManagerServer 中创建 SQL 迁移文件
    - workspace 表：id, user_id, workspace_path, branch_name, status, last_active_time, created_at, updated_at
    - change 表：id, change_id, user_id, branch_name, status, submitted_at, approved_at, reject_reason, created_at, updated_at
    - notification 表：id, user_id, change_id, status, message, delivered, created_at
    - 包含唯一索引 uk_user_id_active 和 uk_change_id
    - _Requirements: 14.1, 14.2, 14.3_

  - [x] 1.2 创建 Java Entity 类和 MyBatis Plus Mapper
    - 创建 `com.fun.novel.workspace.entity.Workspace`、`ChangeUnit`、`Notification` 实体类
    - 创建 `com.fun.novel.workspace.mapper.WorkspaceMapper`、`ChangeMapper`、`NotificationMapper` 接口
    - 创建 `com.fun.novel.workspace.enums.WorkspaceStatusEnum`、`ChangeStatusEnum` 枚举
    - _Requirements: 14.1, 14.2_

  - [x] 1.3 创建所有 DTO 类
    - 创建 `com.fun.novel.workspace.dto` 包下的所有 DTO：SyncResult, MergeResult, MergeRequest, RejectRequest, ConflictDetail, Resolution, ResolveRequest, ApproveResult, NotificationPayload, WorkspaceStatus
    - _Requirements: 5.2, 6.4, 7.1, 12.3_

- [x] 2. Git 操作封装层
  - [x] 2.1 实现 GitExecutor 工具类
    - 创建 `com.fun.novel.workspace.utils.GitExecutor`
    - 实现 worktreeAdd、worktreeRemove、exec、status、trialMerge、merge、mergeAbort、fetch、gc 方法
    - 基于现有 CommandLineExecutor 封装 Git 命令执行
    - 包含命令执行超时控制和错误日志记录
    - _Requirements: 1.1, 1.5, 10.1_

  - [ ]* 2.2 编写 GitExecutor 单元测试
    - 测试命令拼接正确性
    - 测试超时处理
    - 测试错误场景
    - _Requirements: 1.5_

- [x] 3. Checkpoint - 确保基础层编译通过
  - 确保所有 Entity、Mapper、DTO、GitExecutor 编译无误，ask the user if questions arise.

- [x] 4. 工作空间管理服务
  - [x] 4.1 实现 TemplateNodeModulesService
    - 创建 `com.fun.novel.workspace.service.TemplateNodeModulesService`
    - 实现 regenerate()、isTemplateOutdated()、copyToWorkspace() 方法
    - copyToWorkspace 使用 `cp -al` 硬链接复制
    - _Requirements: 13.1, 13.2, 13.3_

  - [x] 4.2 实现 WorkspaceManagerService
    - 创建 `com.fun.novel.workspace.service.WorkspaceManagerService`
    - 实现 getOrCreateWorkspace：检查数据库是否有 active 工作空间，无则创建
    - 实现 createWorkspace：git worktree add + cp -al + 数据库记录
    - 实现 destroyWorkspace：git worktree remove + 更新状态
    - 实现 resolveWorkspacePath：根据 userId 查表返回路径
    - 实现 commitWorkspaceChanges：git add -A && git commit
    - 实现 rebaseToLatestMain：切换工作空间到最新主线
    - 实现 touchLastActiveTime：更新活跃时间
    - 实现 hasUncommittedChanges：git status --porcelain 检查
    - 创建失败时清理半成品 worktree
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 2.1, 2.2, 4.2, 9.1, 9.5, 14.3, 15.1, 15.2, 15.3_

  - [ ]* 4.3 编写 WorkspaceManagerService 属性测试
    - **Property 1: Workspace reuse idempotency**
    - **Property 3: Workspace path resolution correctness**
    - **Property 14: One user one active workspace invariant**
    - **Validates: Requirements 2.1, 4.2, 14.3**

  - [ ]* 4.4 编写 WorkspaceManagerService 单元测试
    - 测试创建失败时的清理逻辑
    - 测试复用时 last_active_time 更新
    - 测试工作空间不存在时自动创建
    - _Requirements: 1.5, 1.6, 2.1, 2.2, 9.5_

- [x] 5. 变更单元管理服务
  - [x] 5.1 实现 ChangeManagerService
    - 创建 `com.fun.novel.workspace.service.ChangeManagerService`
    - 实现 createChange：生成唯一 change_id（UUID），创建变更记录
    - 实现 updateStatus：校验状态转换合法性（editing→submitted→approved/rejected）
    - 实现 reject：更新状态并记录拒绝原因
    - 实现 hasPendingChange：查询是否有 submitted 状态的变更
    - 实现 getApprovalQueue：查询所有 submitted 状态的变更
    - 实现 getById：根据 changeId 查询
    - 拒绝非法状态转换，拒绝重复提交
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 8.1_

  - [ ]* 5.2 编写 ChangeManagerService 属性测试
    - **Property 4: Change ID uniqueness**
    - **Property 5: Change status transition enforcement**
    - **Property 6: Duplicate submission rejection**
    - **Property 10: Approval queue filtering**
    - **Validates: Requirements 5.1, 5.3, 5.4, 8.1**

  - [ ]* 5.3 编写 ChangeManagerService 单元测试
    - 测试状态转换的合法/非法路径
    - 测试重复提交拒绝
    - 测试审批队列过滤
    - _Requirements: 5.3, 5.4, 8.1_

- [x] 6. 同步与合并服务
  - [x] 6.1 实现 SyncService
    - 创建 `com.fun.novel.workspace.service.SyncService`
    - 实现 syncWorkspace：幂等保护（30秒内防重复）+ dry-run 检测 + 自动合并
    - 实现 isBehindMain：git log main..ws_branch 检测是否落后
    - 实现 doSync：在工作空间内 merge main
    - 返回三种结果：up_to_date、merged、conflicts
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [x] 6.2 实现 MergeEngine
    - 创建 `com.fun.novel.workspace.service.MergeEngine`
    - 使用 ReentrantLock 实现全局锁，tryLock 30秒超时
    - 实现 submitMerge：获取锁 → trial merge → 检测冲突 → 创建审批记录或返回冲突
    - 实现 approve：获取锁 → merge 到 main → 更新状态 → 切换基线 → 通知用户
    - 实现 scheduledGc：定时 GC（凌晨3点）
    - 合并失败时 abort merge 恢复状态
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 8.2, 8.3, 8.5, 8.6, 8.7, 10.1, 10.2, 10.3_

  - [ ]* 6.3 编写 SyncService 属性测试
    - **Property 2: Sync endpoint idempotency**
    - **Validates: Requirements 3.5**

  - [ ]* 6.4 编写 MergeEngine 单元测试
    - 测试全局锁超时处理
    - 测试 trial merge 冲突/无冲突分支
    - 测试审批通过/失败的事务回滚
    - _Requirements: 6.1, 6.6, 8.2, 8.5, 10.1_

- [x] 7. 冲突解析服务
  - [x] 7.1 实现 ConflictResolver
    - 创建 `com.fun.novel.workspace.service.ConflictResolver`
    - 实现 parseConflicts：解析 JSON/YAML 文件的字段级差异
    - 实现 applyResolution：根据用户选择应用冲突解决方案
    - 实现 isStructuredConfig：判断文件扩展名是否为 .json/.yaml/.yml
    - 使用 Jackson 解析 JSON，SnakeYAML 解析 YAML
    - _Requirements: 6.4, 6.5, 7.1, 7.2, 7.3_

  - [ ]* 7.2 编写 ConflictResolver 属性测试
    - **Property 7: Structured config conflict field extraction**
    - **Property 8: Structured file type classification**
    - **Property 9: Conflict resolution application**
    - **Validates: Requirements 6.4, 6.5, 7.1, 7.2**

  - [ ]* 7.3 编写 ConflictResolver 单元测试
    - 测试 JSON 字段级差异提取
    - 测试 YAML 字段级差异提取
    - 测试非结构化文件的判断
    - 测试解决方案应用后字段值正确
    - _Requirements: 6.4, 6.5, 7.1, 7.2_

- [x] 8. Checkpoint - 确保所有后端服务编译通过并通过单元测试
  - 确保所有 Service 层代码编译无误，ask the user if questions arise.

- [x] 9. 通知服务与定时任务
  - [x] 9.1 实现 NotificationService
    - 创建 `com.fun.novel.workspace.service.NotificationService`
    - 实现 notify：通过 STOMP WebSocket 推送通知给指定用户
    - 实现 storeOfflineNotification：WebSocket 不可用时存储离线通知
    - 实现 deliverOfflineNotifications：用户重连时投递离线通知并标记已投递
    - 复用现有 WebSocket 基础设施
    - _Requirements: 8.7, 12.1, 12.2, 12.3_

  - [x] 9.2 实现 WorkspaceScheduledTasks 定时任务
    - 创建 `com.fun.novel.workspace.config.WorkspaceScheduledTasks`
    - 实现工作空间超时回收：查询 last_active_time > 7天的工作空间
    - 回收前检查未提交改动，有改动则发通知跳过回收
    - 无改动则 git worktree remove + 更新状态
    - _Requirements: 9.2, 9.3, 9.4_

  - [ ]* 9.3 编写 NotificationService 属性测试
    - **Property 12: Offline notification delivery completeness**
    - **Property 13: Entity data integrity**
    - **Property 16: Notification payload completeness**
    - **Validates: Requirements 12.2, 12.3, 14.1, 14.2**

  - [ ]* 9.4 编写定时回收任务属性测试
    - **Property 11: Reclamation query correctness**
    - **Validates: Requirements 9.2**

- [x] 10. 后端 Controller 层
  - [x] 10.1 实现 WorkspaceController
    - 创建 `com.fun.novel.workspace.controller.WorkspaceController`
    - 实现所有 REST 端点：sync, merge, approve, reject, conflicts, resolve, status, approval-queue, destroy
    - 统一使用 `com.fun.novel.common.Result` 返回格式
    - 每个 API 调用时通过 touchLastActiveTime 更新活跃时间
    - _Requirements: 1.1, 3.5, 5.1, 6.1, 7.4, 8.1, 8.2, 8.4, 9.1, 11.1_

  - [x] 10.2 改造现有 AgentBuildPublishController
    - 在现有构建/发布接口新增 userId 参数
    - 通过 WorkspaceManagerService.resolveWorkspacePath(userId) 获取工作目录
    - 将工作目录传递给构建/发布命令执行
    - 保持其他逻辑不变
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [ ]* 10.3 编写 WorkspaceController 集成测试
    - 测试 sync 接口的幂等性
    - 测试 merge 接口的冲突/无冲突分支
    - 测试 approve/reject 接口
    - _Requirements: 3.5, 5.1, 8.2, 8.4_

- [x] 11. Checkpoint - 后端完整编译通过
  - 确保所有后端代码编译通过，Controller 层接口可正常启动，ask the user if questions arise.

- [x] 12. 前端 API 服务层与状态管理
  - [x] 12.1 创建 workspaceApi.js
    - 创建 `NovelAppManager/src/services/workspaceApi.js`
    - 封装所有后端接口调用：syncWorkspace, getWorkspaceStatus, submitMerge, approve, reject, getConflicts, resolveConflict, getApprovalQueue, destroyWorkspace
    - 复用现有 request.js 的 axios 实例
    - _Requirements: 3.5, 5.1, 7.4, 8.1, 11.1_

  - [x] 12.2 创建 workspaceStore.js (Pinia)
    - 创建 `NovelAppManager/src/stores/workspaceStore.js`
    - 定义 state：workspaceStatus, hasUncommittedChanges, lastActiveTime, currentChangeId, changeStatus, conflicts
    - 实现 actions：syncWorkspace, submitMerge, resolveConflict, getStatus, connectWebSocket
    - WebSocket 连接管理：接收通知并更新状态
    - _Requirements: 11.1, 11.3, 12.1_

- [x] 13. 前端组件实现
  - [x] 13.1 实现 WorkspaceStatusBar.vue
    - 创建 `NovelAppManager/src/components/workspace/WorkspaceStatusBar.vue`
    - 展示工作空间状态、未提交改动指示器、最后操作时间
    - 包含"提交合并"和"放弃"操作按钮
    - 无工作空间时显示"开始编辑"按钮
    - 从 workspaceStore 获取状态
    - _Requirements: 11.1, 11.2, 11.3_

  - [x] 13.2 实现 ConflictResolutionPanel.vue
    - 创建 `NovelAppManager/src/components/workspace/ConflictResolutionPanel.vue`
    - 逐字段展示冲突对比（主库值 vs 用户值）
    - 每个字段提供"使用主库的值"和"保留我的值"选择按钮
    - 全部选择后允许重新提交
    - _Requirements: 7.1, 7.2, 7.3, 7.4_

  - [x] 13.3 实现 ApprovalQueue.vue
    - 创建 `NovelAppManager/src/components/workspace/ApprovalQueue.vue`
    - 展示待审批变更列表
    - 每条变更显示用户、提交时间、操作按钮（通过/拒绝）
    - 拒绝时弹出原因输入框
    - _Requirements: 8.1, 8.2, 8.4_

  - [x] 13.4 实现 WorkspaceApproval.vue 页面并注册路由
    - 创建 `NovelAppManager/src/views/workspace/WorkspaceApproval.vue`
    - 引入 ApprovalQueue 组件
    - 在 router 中注册审批管理页面路由
    - _Requirements: 8.1_

- [x] 14. 前端集成与全局接入
  - [x] 14.1 在布局中集成 WorkspaceStatusBar
    - 在 App.vue 或主布局组件中引入 WorkspaceStatusBar.vue
    - 确保状态栏在所有页面顶部持久展示
    - 页面加载时自动调用 syncWorkspace
    - _Requirements: 11.1, 11.4, 3.1, 3.2, 3.3_

  - [x] 14.2 集成 WebSocket 通知
    - 在 workspaceStore 中实现 WebSocket 连接（复用现有 STOMP 基础设施）
    - 接收审批结果通知并更新 UI 状态
    - 显示 toast 通知："上次提交已合并到主库 ✅，工作区已更新"
    - 处理离线通知：重连时拉取未投递通知
    - _Requirements: 11.4, 12.1, 12.2, 12.3_

- [x] 15. Checkpoint - 前后端联调验证
  - 确保前端组件能正确调用后端接口，WebSocket 通知正常推送，ask the user if questions arise.

- [ ] 16. 属性测试补充（Property 15: Path stability）
  - [ ]* 16.1 编写基线切换路径稳定性属性测试
    - **Property 15: Path stability during baseline switch**
    - 验证审批通过后 workspace_path 不变
    - **Validates: Requirements 15.2**

- [x] 17. Final checkpoint - 全部完成
  - 确保所有代码编译通过，后端服务可正常启动，前端页面可正常渲染，ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- 后端代码统一放在 `com.fun.novel.workspace` 包下
- 前端代码统一放在 `components/workspace/`、`views/workspace/`、`stores/workspaceStore.js`、`services/workspaceApi.js`
- 复用现有的 `CommandLineExecutor`、`Result`、WebSocket 基础设施
- 属性测试使用 jqwik 库，每个属性至少 100 次迭代
- 每个 Checkpoint 确保增量验证，避免问题积累
