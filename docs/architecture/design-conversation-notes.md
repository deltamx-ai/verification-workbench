# 设计对话纪要（仅保留 Alma 的设计输出）

这份文档把关于 **verification workbench** 的关键设计对话整理进仓库。

原则：
- 只保留 Alma 的回答与设计输出
- 去掉提问原文
- 优先保留对架构、边界、数据模型、上下文恢复、落库方向有价值的内容
- 对纯执行性回复做少量保留，用于记录仓库落档过程

---

## 1. 总体设计定稿

### 1.1 统一定位
“验证层桌面应用”不是单纯聊天工具，也不是单纯自动化执行器。

它由三条主线共同构成：
1. 验证主线：负责 Task / Run / Step / Artifact / Verdict
2. 对话记忆主线：负责 Thread / Message / Summary / Memory / Snapshot
3. 上下文编排层：负责恢复、压缩、长期记忆提取、任务关联

系统目标不是回放聊天记录，而是围绕验证任务形成一个可持续推进、可复盘、可恢复上下文的桌面协作环境。

### 1.2 已确认决策
设计已经纳入三个明确决策：
- SQLite 单库优先
- 记忆系统偏项目协作型
- 一个 thread 支持关联多个 task / run

### 1.3 设计落点
- 数据层围绕本地桌面单机一致性、事务简单性、可备份性展开
- 记忆系统优先沉淀项目决策、约束、结论、讨论上下文，而不是泛用户闲聊偏好
- Thread 与 Task / Run 采用多对多关联，支持一个讨论线程推进多个验证任务与多次执行

---

## 2. 最小正确骨架

### 2.1 结构化验证主链
最小正确骨架的第一部分是验证主链：

Task → TaskVersion → StepDefinition → Run → StepRun → Artifact → Verdict

这个链路表达的是：
- Task 负责任务本体
- TaskVersion 负责冻结某次任务定义
- StepDefinition 负责步骤定义
- Run 负责一次执行实例
- StepRun 负责步骤级执行结果
- Artifact 负责截图、日志、结构化输出等证据
- Verdict 负责结构化判定

### 2.2 协作容器
第二部分是线程化协作容器：
- ChatThread
- ChatMessage
- ThreadTaskLink
- ThreadRunLink

这个层负责把“对话推进”和“验证对象”连接起来，但不把聊天消息直接混进任务执行表。

### 2.3 摘要与记忆
第三部分是项目协作型摘要与记忆：
- ThreadSummary
- MemoryItem

这里的重点不是做一个闲聊型记忆系统，而是把项目里的长期约束、结论、决策、风险点沉淀下来。

### 2.4 上下文恢复
第四部分是上下文编排与恢复：
- recent messages
- rolling summary
- milestone summary
- long-term memory
- task / run facts
- ContextSnapshot

也就是说，恢复上下文时不是把所有历史消息一股脑重新塞进去，而是分层组装“当前工作现场”。

---

## 3. 模块边界

设计里明确划分了这些边界：

- UI Shell：只负责呈现与交互
- Application / Service：负责流程编排与事务边界
- Domain：负责核心业务模型与约束
- Execution Engine：执行步骤并产生事实
- Artifact：统一管理证据对象
- Verdict：结构化判定层
- Integration：外部适配层
- Memory / Context Orchestration：摘要、记忆、上下文组装

这里有一个很重要的点：
**业务状态不应该由 UI 解析日志得出，聊天消息也不应该直接充当任务模型。**

---

## 4. 数据模型主线

### 4.1 验证主线对象
验证主线包含：
- Task
- TaskVersion
- StepDefinition
- Run
- StepRun
- Artifact
- Verdict
- Environment
- ExecutorProfile

### 4.2 对话记忆主线对象
对话记忆主线包含：
- ChatThread
- ChatMessage
- ThreadSummary
- MemoryItem
- ContextSnapshot
- ThreadTaskLink
- ThreadRunLink

这两条主线是协作关系，不是互相污染的关系。

---

## 5. SQLite 表结构分组

为了兼顾本地 SQLite 落地与未来迁移，表结构被分成几组。

### 5.1 任务定义层
- tasks
- task_versions
- step_definitions

### 5.2 执行实例层
- runs
- step_runs
- run_state_transitions

### 5.3 产物层
- artifacts
- artifact_relations

### 5.4 判定层
- verdicts
- verdict_evidence_links

### 5.5 环境配置层
- environments
- executor_profiles
- environment_snapshots

### 5.6 对话记忆层
- chat_threads
- chat_messages
- thread_summaries
- memory_items

### 5.7 上下文编排层
- context_snapshots
- context_snapshot_items
- thread_task_links
- thread_run_links

### 5.8 事件与集成层
- domain_events
- integration_jobs
- integration_logs

这个分组方式本身就是一种边界声明：
验证执行、对话记忆、上下文编排、集成事件各自有落点，不混写。

---

## 6. 上下文恢复设计

上下文恢复的核心思想已经明确：

恢复时不拼全量历史，而是按五层组装：
1. recent messages
2. rolling summary
3. milestone summary
4. long-term memory
5. task / run facts

### 6.1 使用场景
打开 thread 时，需要优先恢复：
- 最近消息
- 最新 rolling summary
- 最近 milestone summary
- 当前活跃 task / run links

继续追问失败原因时，需要恢复：
- 相关 run / step_run facts
- 失败 artifact
- 最近 verdict
- 对应摘要

重新开始某任务验证时，需要恢复：
- task current version
- success criteria
- environment / executor profile
- 最近历史 runs

### 6.2 ContextSnapshot
每次组装上下文后生成 ContextSnapshot，记录：
- 选了哪些消息
- 选了哪些摘要
- 选了哪些 memory
- 选了哪些 task / run facts
- token 预算

这个设计让恢复链路具备可审计性，也方便后面调优。

---

## 7. 明确避开的反模式

设计里已经明确列出几类反模式：
- UI 解析日志决定业务状态
- 聊天消息混进任务表
- 执行状态与 verdict 混淆
- 全量消息直接进入上下文
- 长期记忆做成垃圾桶
- run 直接读取 task 最新配置而不绑定 task_version

这些反模式基本就是后续实现时最该防的坑。

---

## 8. 路线建议

### Phase 1
- SQLite schema 第一版
- 验证主链核心对象
- thread / message / link 基础能力
- 最小 UI shell

### Phase 2
- rolling summary
- ContextSnapshot
- 最近消息 + 摘要 + task/run facts 恢复

### Phase 3
- MemoryItem
- milestone summary
- 项目协作型长期记忆

### Phase 4
- 多 run 对比
- verdict evidence link
- 环境快照
- 更强判定能力

这个路线本质上是先把“验证闭环”和“上下文恢复闭环”做起来，再上更复杂的记忆治理与判定增强。

---

## 9. 仓库落档过程中的关键说明

除了设计本身，仓库落档过程里也有几条值得保留的上下文。

### 9.1 首次落档结果
当时已经完成：
- 初始化仓库
- 整理并落档设计文档
- 提交初始提交

初始提交为：
- `e093ad3` — `docs: initialize verification workbench architecture`

主要文件包括：
- `README.md`
- `docs/architecture/overall-design.md`
- `docs/architecture/data-model-outline.md`
- `docs/architecture/context-recovery.md`
- `docs/architecture/roadmap.md`

### 9.2 路径修正
后来确认过一次，最初把仓库初始化在 Alma 当前工作目录下，这不对。

更正确的目标目录应该是：
- `~/workspace/ai/verification-workbench`

之后已经完成迁移，新的本地位置是：
- `/home/delta/workspace/ai/verification-workbench`

### 9.3 git 配置上下文
当时查看到的全局 git 配置包括：
- `user.name=deltamx`
- `user.email=deltamx.ai@gmail.com`
- `core.sshcommand=ssh -i ~/.ssh/id_ed25519_ai`
- `credential.helper=store`

### 9.4 远端接入与推送
仓库已经接到远端：
- `git@github.com:deltamx-ai/verification-workbench.git`

并已完成：
- `git branch -M main`
- `git push -u origin main`

---

## 10. 为什么要保留这份纪要

当前仓库里的正式文档更像“定稿版架构说明”。

而这份纪要保留的是设计推进过程里更像“脑回路”的部分：
- 为什么系统不是聊天工具也不是执行器
- 为什么要三条主线并存
- 为什么 thread 要支持多 task / run
- 为什么上下文恢复必须做分层组装
- 为什么记忆系统要偏项目协作型
- 为什么需要把 snapshot 作为恢复与审计抓手

这部分内容对后续做 schema、服务边界、上下文系统实现时，参考价值会更高。
