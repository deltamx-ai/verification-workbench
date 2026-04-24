# 验证层桌面应用整体设计

## 1. 系统定位
这是一个以验证协作为中心的桌面验证工作台，不是单纯聊天工具，也不是单纯自动化执行器。

系统由三条主线构成：
1. 验证主线：Task / Run / Step / Artifact / Verdict
2. 对话记忆主线：Thread / Message / Summary / Memory / Snapshot
3. 上下文编排层：恢复、压缩、长期记忆提取、任务关联

## 2. 已确认决策
- SQLite 单库优先
- 记忆系统偏项目协作型
- 一个 thread 支持关联多个 task / run

## 3. 最小正确骨架
1. 结构化验证主链：Task → TaskVersion → StepDefinition → Run → StepRun → Artifact → Verdict
2. 线程化协作容器：ChatThread → ChatMessage → ThreadTaskLink / ThreadRunLink
3. 项目协作型记忆与摘要：ThreadSummary + MemoryItem
4. 上下文编排与恢复：recent messages + rolling summary + milestone summary + long-term memory + task/run facts + ContextSnapshot

## 4. 模块边界
- UI Shell：只负责呈现与交互
- Application / Service：负责流程编排与事务边界
- Domain：核心业务模型与约束
- Execution Engine：执行步骤并产生事实
- Artifact：统一管理证据对象
- Verdict：结构化判定层
- Integration：外部适配层
- Memory / Context Orchestration：摘要、记忆、上下文组装

## 5. 数据模型
### 验证主线
- Task
- TaskVersion
- StepDefinition
- Run
- StepRun
- Artifact
- Verdict
- Environment
- ExecutorProfile

### 对话记忆主线
- ChatThread
- ChatMessage
- ThreadSummary
- MemoryItem
- ContextSnapshot
- ThreadTaskLink
- ThreadRunLink

## 6. SQLite 表结构分组
### 任务定义层
- tasks
- task_versions
- step_definitions

### 执行实例层
- runs
- step_runs
- run_state_transitions

### 产物层
- artifacts
- artifact_relations

### 判定层
- verdicts
- verdict_evidence_links

### 环境配置层
- environments
- executor_profiles
- environment_snapshots

### 对话记忆层
- chat_threads
- chat_messages
- thread_summaries
- memory_items

### 上下文编排层
- context_snapshots
- context_snapshot_items
- thread_task_links
- thread_run_links

### 事件与集成层
- domain_events
- integration_jobs
- integration_logs

## 7. 上下文恢复逻辑
恢复时不拼全量历史，而是按以下五层组装：
1. recent messages
2. rolling summary
3. milestone summary
4. long-term memory
5. task / run facts

并在每次组装后落一份 ContextSnapshot，用于恢复与审计。

## 8. 关键反模式
- UI 解析日志决定业务状态
- 聊天消息混进任务表
- 执行状态与 verdict 混淆
- 全量消息直接进入上下文
- 长期记忆做成垃圾桶
- run 直接读取 task 最新配置而不绑定 task_version

## 9. MVP
### 必须先做
- Task / TaskVersion / StepDefinition / Run / StepRun / Artifact / Verdict
- ChatThread / ChatMessage / ThreadTaskLink / ThreadRunLink
- rolling summary
- ContextSnapshot

### 二期扩展
- 高级记忆治理
- 多执行器并发
- AI verdict
- 多 run 对比与更强判定
