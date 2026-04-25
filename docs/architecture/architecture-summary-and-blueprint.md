# 总体设计与架构总纲

这份文档用于把当前 verification workbench 已有的多份设计稿收束成一份“总纲”，方便后续作为 README 索引、架构入口和实现规划基线。

它重点回答五个问题：

- 这个系统到底是什么
- 系统的核心对象有哪些
- 总体架构如何分层
- 各条主线如何协作
- 接下来应该先实现什么

---

## 1. 系统定位

verification workbench 不是单纯的聊天工具，也不是单纯的自动化执行器。

它更像一个**面向验证协作的人机工作台**，统一承载以下能力：

- 围绕需求、缺陷、变更、发布组织验证任务
- 在执行中沉淀日志、截图、结构化结果、判定依据等证据
- 在 thread 对话中持续推进任务，而不是每次重新解释背景
- 通过摘要、记忆、知识检索恢复上下文
- 通过 skill / plugin 接入外部系统能力
- 在可自动化时自动推进，在高风险动作处切回人工审批和接管

因此它的核心不是 message，而是一组可以互相关联的稳定对象：

- `Task`
- `Run`
- `ChatThread`
- `Knowledge`
- `Skill / Plugin`
- `Workflow`
- `Approval / Audit`

---

## 2. 设计目标

整个系统不是为了解决某一个点状功能，而是要解决验证协作中最常见的几个根问题：

1. **任务可定义**
   - 验证对象、目标、约束、环境、步骤、成功标准都能被结构化描述

2. **执行可追踪**
   - 每次 run 做了什么、失败在哪、留下了什么证据，都能回看

3. **对话可延续**
   - 用户在 thread 中说“继续”“为什么失败了”，系统能恢复工作现场

4. **上下文可恢复**
   - 系统能从最近消息、滚动摘要、知识、运行事实中拼出当前上下文

5. **能力可扩展**
   - Jenkins、GitHub、Confluence、邮件、Discord、Feishu 等能力能作为插件接入

6. **副作用可治理**
   - 写回外部系统、触发发布、发送通知等动作必须具备权限边界、审批和审计

7. **准确性可保障**
   - 结论必须尽量基于证据、引用、校验，而不是单纯依赖模型即兴发挥

---

## 3. 总体分层

建议把整个系统拆成六层：

```text
Presentation Layer
Application / Runtime Layer
Domain Layer
Capability Layer
Knowledge Layer
Infrastructure Layer
```

### 3.1 Presentation Layer
负责：
- UI shell
- thread 界面
- task / run 界面
- artifact / verdict 展示
- workflow 管理界面
- skill / plugin 配置界面
- approval / audit 界面

这一层主要负责展示与交互，不负责决定真实业务状态。

### 3.2 Application / Runtime Layer
负责：
- Thread Runtime
- Task Runtime
- Loop Engine
- Retrieval Runtime
- Workflow Runtime
- Approval Runtime
- Audit / Event Runtime

这一层是运行时中枢，用于把用户输入、上下文恢复、工具调用、校验、审批、结果回流串成完整执行链。

### 3.3 Domain Layer
负责定义稳定业务对象与状态约束。

核心对象包括：
- `Task`
- `TaskVersion`
- `StepDefinition`
- `Run`
- `StepRun`
- `Artifact`
- `Verdict`
- `ChatThread`
- `ChatMessage`
- `ThreadSummary`
- `MemoryItem`
- `ContextSnapshot`
- `WorkflowDefinition`
- `WorkflowRun`
- `ApprovalRequest`
- `AuditEvent`

它决定系统里“什么才算合法状态”。

### 3.4 Capability Layer
负责扩展能力，是系统的“手”。

包括：
- Skill Registry
- Skill Runtime
- Plugin Runtime
- Provider Client
- External Action Adapter

这一层的目标是让系统可以接入外部能力，而不把外部平台的细节写死到核心内核里。

### 3.5 Knowledge Layer
负责知识、记忆和上下文资料，是系统的“资料室”。

包括：
- 文档知识
- 对话摘要
- 运行事实
- 长期记忆
- 检索索引
- citation 链路
- retrieval session/result

它负责回答“系统知道什么、从哪知道、为什么这次拿这些内容”。

### 3.6 Infrastructure Layer
负责底座能力：
- SQLite
- 文件存储 / artifact storage
- 索引系统
- 事件日志
- 调度器 / 后台 worker
- 凭据管理
- provider 调用日志
- 配置与环境隔离

---

## 4. 五条核心主线

### 4.1 验证主线

```text
Task -> TaskVersion -> StepDefinition -> Run -> StepRun -> Artifact -> Verdict
```

这条主线解决：
- 验证什么
- 按什么定义验证
- 本次具体执行了哪些步骤
- 产生了哪些证据
- 最终判定是什么

### 4.2 对话主线

```text
ChatThread -> ChatMessage -> ThreadSummary -> ContextSnapshot
```

这条主线解决：
- 讨论是如何推进的
- 当前 thread 的工作上下文是什么
- 用户一句“继续”到底接着哪一段继续

### 4.3 知识主线

```text
KnowledgeSource -> KnowledgeItem -> Chunk / Index -> Retrieval -> Citation
MemoryItem -> Summary -> ContextSnapshot
```

这条主线解决：
- 系统知道什么
- 知识从哪里来
- 检索结果为什么可信
- 哪些结论值得沉淀为长期记忆

### 4.4 能力扩展主线

```text
Plugin -> Action -> Skill -> Loop Decision -> Execution Result
```

这条主线解决：
- 系统会做什么
- 新能力如何接进来
- 如何避免把外部系统能力硬编码到核心流程中

### 4.5 治理主线

```text
Permission -> Approval -> Audit -> Event -> Recovery
```

这条主线解决：
- 哪些动作可以执行
- 哪些动作必须审批
- 每次副作用如何留痕
- 失败和中断后如何恢复

---

## 5. 运行时总览

整个系统建议围绕几个核心 runtime 协作：

```text
User / Thread
  -> Thread Runtime
    -> Retrieval Runtime
    -> Loop Engine
      -> Skill Runtime
        -> Plugin Runtime
      -> Task Runtime
      -> Workflow Runtime
      -> Approval Runtime
    -> Audit / Event Runtime
```

核心分工建议如下：

### 5.1 Thread Runtime
负责：
- 接收消息
- 解析当前意图
- 恢复 thread context
- 找到关联 task / run / workflow
- 决定是否触发 loop
- 把结果写回 thread

### 5.2 Task Runtime
负责：
- 创建 task / task_version / run
- 管理 step 定义与 step run
- 汇总 artifact
- 生成 verdict
- 跟踪 task / run 状态迁移

### 5.3 Loop Engine
负责：
- 决策当前下一步做什么
- 组织 `plan -> act -> observe -> decide`
- 驱动 search / read / write / validate 闭环
- 决定何时继续、重试、暂停、升级审批或交回人工

### 5.4 Retrieval Runtime
负责：
- 检索知识源
- 取回历史对话和摘要
- 汇总最近运行事实
- 形成可引用上下文输入

### 5.5 Skill Runtime / Plugin Runtime
负责：
- 把能力包装成稳定 action
- 执行具体外部操作
- 统一输入输出、错误和 trace
- 把副作用记录回审计链路

### 5.6 Workflow Runtime
负责：
- 承载长期流程定义
- 处理 manual / schedule / event trigger
- 创建 workflow run
- 将自动化流程纳入同样的验证、审批、审计体系

### 5.7 Approval Runtime / Audit Runtime
负责：
- 拦截高风险动作
- 人工审批/接管
- 记录状态变化、外部写入、副作用事件、失败原因与恢复过程

---

## 6. 标准体系

目前设计已经明确需要先统一四类标准。

### 6.1 身份标准
- 所有核心对象有稳定字符串 ID / UUID
- 所有关联字段统一命名为 `<object>_id`
- 运行链路统一可挂到 `thread / task / run / workflow_run / trace` 之一

### 6.2 状态标准
统一生命周期状态集合，避免各层自行命名：
- `draft`
- `active`
- `paused`
- `queued`
- `pending`
- `running`
- `completed`
- `failed`
- `cancelled`
- `blocked`
- `needs_approval`

### 6.3 时间标准
所有运行对象统一具备：
- `created_at`
- `updated_at`
- `started_at`
- `ended_at` / `completed_at`

定时流程额外具备：
- `next_run_at`
- `last_run_at`
- `start_at`
- `end_at`

### 6.4 审计标准
所有产生副作用、决策和状态变化的对象，都应支持：
- `created_by`
- `reason`
- `trace_id`
- `error_code`
- `error_message`
- `status`

---

## 7. 数据模型总体分域

### 7.1 核心验证域
- `tasks`
- `task_versions`
- `step_definitions`
- `runs`
- `step_runs`
- `artifacts`
- `verdicts`

### 7.2 对话与上下文域
- `chat_threads`
- `chat_messages`
- `thread_summaries`
- `memory_items`
- `context_snapshots`
- `thread_task_links`
- `thread_run_links`

### 7.3 SRWV 执行域
- `srwv_loops`
- `srwv_iterations`
- `srwv_search_results`
- `srwv_read_documents`
- `srwv_read_facts`
- `srwv_write_operations`
- `srwv_validation_results`
- `srwv_validation_issues`
- `srwv_decisions`
- `srwv_citations`

### 7.4 Workflow 自动执行域
- `workflow_definitions`
- `workflow_definition_versions`
- `workflow_triggers`
- `workflow_runs`
- `workflow_run_steps`
- `workflow_run_events`
- `workflow_run_artifacts`
- `workflow_run_approvals`
- `workflow_run_links`

### 7.5 能力扩展域
- `plugins`
- `plugin_versions`
- `plugin_actions`
- `skills`
- `skill_versions`
- `skill_bindings`
- `skill_execution_logs`

### 7.6 知识与检索域
- `knowledge_sources`
- `knowledge_items`
- `knowledge_chunks`
- `knowledge_citations`
- `knowledge_links`
- `retrieval_sessions`
- `retrieval_results`

### 7.7 治理与可观测域
- `approval_requests`
- `approval_decisions`
- `audit_events`
- `runtime_errors`
- `trace_spans`
- `provider_call_logs`
- `environment_profiles`
- `config_entries`
- `credentials`（建议仅保存 metadata / reference）

---

## 8. 数据库存储边界

### 8.1 必须结构化入库的内容
必须进数据库的，主要是：
- 状态
- 关系
- 生命周期对象
- 执行记录
- 检索结果
- 审批审计
- workflow / run / step
- verdict / evidence linkage

### 8.2 适合 artifact / blob / JSON 的内容
不必重度结构化，但必须留下引用入口：
- 原始网页内容
- 原始邮件正文
- 截图
- 原始 provider 响应
- 长文草稿
- 临时上下文快照
- 大体积 trace payload

推荐方式：
- 数据库存 metadata + `storage_uri` / hash / summary
- 文件本体存文件系统或对象存储

也就是说：
**数据库负责关系、状态、治理与索引；文件系统负责大内容本体。**

---

## 9. 自动化与流程定义能力

设计上已经明确系统必须支持“用户定义一个流程，后续按定时或事件自动执行”。

因此 workflow 能力应包含：

### 9.1 Workflow Definition
定义：
- 流程目标
- 输入来源
- 执行步骤
- 输出写入位置
- 校验策略
- 权限策略
- 重试策略

### 9.2 Trigger
至少支持三类：
- `manual`
- `schedule`
- `event`

### 9.3 Workflow Runtime
负责：
- trigger 触发后创建 workflow run
- 驱动 SRWV / skill / plugin 执行
- 处理中间审批和失败重试
- 将执行过程纳入 audit / recovery 体系

### 9.4 调度设计关注点
schedule trigger 需要明确：
- `cron`
- `timezone`
- `enabled`
- `start_at`
- `end_at`
- `next_run_at`
- `last_run_at`
- `misfire_policy`
- `max_concurrency`

---

## 10. 准确性保障思路

这套系统要强调准确性，不能只靠模型临场发挥。

当前设计里，准确性主要依赖这几层：

1. **SRWV 闭环**
   - 先 search / read，再 write，最后 validate

2. **citation 机制**
   - 结论必须尽量能回溯到知识、文档、运行事实或 artifact

3. **结构化 evidence**
   - screenshot、log、report、step output、verdict reason 都要能挂接

4. **validation policy**
   - 对 freshness、完整性、来源可信度、格式一致性做校验

5. **人工接管点**
   - 在高风险、低置信度、冲突证据场景下切回人工

---

## 11. 当前阶段的实现优先级

结合现有 roadmap，建议实现顺序是：

### Phase 1：内核最小闭环
- SQLite schema 第一版
- 核心验证对象落库
- thread / message / link 基础能力
- 最小 UI shell

### Phase 2：上下文恢复
- rolling summary
- ContextSnapshot
- recent messages + summaries + task/run facts 恢复

### Phase 3：长期记忆
- MemoryItem
- milestone summary
- 项目协作型长期记忆

### Phase 4：更强判定与自动化
- 多 run 对比
- verdict evidence link
- 环境快照
- workflow 自动执行
- 更强 validation 与 accuracy 保障

---

## 12. 现阶段一句结论

如果用一句话总结当前整体设计：

**verification workbench 正在从“带工具调用的聊天体”收束成“以 Task / Run / Thread / Knowledge / Workflow / Audit 为核心对象的验证协作操作系统雏形”。**

而且当前方向不是继续先堆功能，而是优先把：

- 标准
- 数据模型
- 运行时模型
- 自动化模型
- 治理模型
- 准确性保障模型

这几个底盘先打稳。

---

## 13. 推荐阅读顺序

建议把当前文档按这个顺序阅读：

1. `docs/architecture/architecture-summary-and-blueprint.md`
2. `docs/architecture/system-architecture-overview.md`
3. `docs/architecture/runtime-and-state-machine-design.md`
4. `docs/architecture/standards-and-database-design.md`
5. `docs/architecture/search-read-write-validation-loop.md`
6. `docs/architecture/search-read-write-validation-schema.md`
7. `docs/architecture/search-read-write-validation-runtime-contract.md`
8. `docs/architecture/workflow-scheduling-and-automation-design.md`
9. `docs/architecture/workflow-schema-design.md`
10. `docs/architecture/workflow-runtime-contract.md`
11. `docs/architecture/error-and-governance-model.md`
12. `docs/architecture/accuracy-assurance-and-evaluation-design.md`
13. `docs/architecture/observability-and-debuggability-design.md`
14. `docs/architecture/roadmap.md`
