# verification workbench 整体架构总览

这份文档尝试把前面的多份设计稿收束成一份更高层的总览，目标不是重复细节，而是回答这几个核心问题：

- 这个系统到底是什么
- 它的核心模块有哪些
- 模块之间如何协作
- 哪些是稳定内核，哪些是可扩展能力
- 第一版应该落到什么程度

---

## 1. 系统定位

verification workbench 本质上不是单纯聊天工具，也不是单纯自动化执行器。

它更像一个**面向验证协作的人机工作台**，把以下几件事统一到一个桌面系统里：

- 围绕需求、改动、缺陷、发布建立验证任务
- 在执行中沉淀日志、截图、结构化结果等证据
- 在对话中持续推进、解释、复盘验证过程
- 通过知识库与摘要能力恢复上下文
- 通过技能和插件接入外部系统
- 在需要时让 Agent 自动推进，在高风险时切回人工控制

所以它不是“chat + tools”的简单拼接，
而是一个以 **Task / Run / Thread / Knowledge / Skill / Plugin** 为核心对象的协作系统。

---

## 2. 系统目标

这个系统要解决的不是单点自动化，而是“验证工作为什么总是断、散、不可追踪”的问题。

它至少要同时做到：

1. **任务可定义**
   - 明确验证对象、目标、约束、环境、步骤、成功标准

2. **执行可追踪**
   - 每次 run 做了什么、失败在哪、留下了什么证据，都能回看

3. **对话可延续**
   - 用户可以在 thread 中不断推进任务，而不是每次重新解释背景

4. **上下文可恢复**
   - 系统能从历史消息、摘要、知识和执行事实里拼出当前工作现场

5. **能力可扩展**
   - 能逐步接入 Jenkins、邮件、Confluence、GitHub、Feishu 等能力，而不污染核心架构

6. **副作用可控**
   - 发版、回邮件、写文档这类动作必须有权限边界、审批和审计

---

## 3. 总体分层

建议把整个系统拆成六个大层：

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
- task/run 界面
- artifact / verdict 展示
- skill / plugin 配置界面
- approval / audit 界面

这一层只负责交互，不负责决定真实业务状态。

### 3.2 Application / Runtime Layer
负责：
- Thread Runtime
- Task Runtime
- Loop Engine
- Retrieval Runtime
- Approval Runtime
- Audit / Event Runtime

这是系统的运行中枢，负责把用户输入、上下文恢复、技能调用、外部动作和结果回流串起来。

### 3.3 Domain Layer
负责稳定业务对象和约束：
- Task
- TaskVersion
- StepDefinition
- Run
- StepRun
- Artifact
- Verdict
- ChatThread
- ChatMessage
- ThreadSummary
- MemoryItem
- ContextSnapshot

这层决定系统“什么才算合法状态”。

### 3.4 Capability Layer
负责扩展能力：
- Skill Registry
- Skill Runtime
- Plugin Runtime
- Provider Client
- External Action Adapters

这是系统的“手”。

### 3.5 Knowledge Layer
负责上下文知识：
- 文档知识
- 运行事实
- 对话决策
- 长期记忆
- 检索索引
- 摘要与 citation

这是系统的“资料室”。

### 3.6 Infrastructure Layer
负责技术底座：
- SQLite
- 文件存储 / artifact storage
- 索引
- 事件日志
- 凭据管理
- 调度与后台 worker

---

## 4. 五条核心主线

从对象关系上看，整个系统可以归纳成五条主线。

### 4.1 验证主线
核心链路：

```text
Task -> TaskVersion -> StepDefinition -> Run -> StepRun -> Artifact -> Verdict
```

它解决：
- 要验证什么
- 按什么定义验证
- 实际跑了哪些步骤
- 产出了哪些证据
- 最终判断是什么

### 4.2 对话主线
核心链路：

```text
ChatThread -> ChatMessage -> ThreadSummary -> ContextSnapshot
```

它解决：
- 讨论是怎么推进的
- 当前 thread 的上下文是什么
- 为什么用户一句“继续”系统也知道接哪一段

### 4.3 记忆与知识主线
核心链路：

```text
KnowledgeSource -> KnowledgeItem -> Chunk / Index -> Retrieval -> Citation
MemoryItem -> Summary -> ContextSnapshot
```

它解决：
- 系统知道什么
- 从哪知道的
- 为什么这次检索返回了这些东西
- 哪些结论是长期值得保留的

### 4.4 能力扩展主线
核心链路：

```text
Plugin -> Action -> Skill -> Loop Decision -> Execution Result
```

它解决：
- 系统会做什么
- 新能力怎么接进来
- 如何避免把 Jenkins / 邮件 / Confluence 写死进核心流程

### 4.5 治理主线
核心链路：

```text
Permission -> Approval -> Audit -> Event -> Recovery
```

它解决：
- 哪些动作可以执行
- 谁批准了高风险操作
- 外部副作用能否追踪
- 出错后如何恢复

---

## 5. 核心模块抽象

### 5.1 Thread Runtime
Thread Runtime 是线程级控制器。

它负责：
- 接收消息
- 解析当前意图
- 恢复 thread context
- 找到关联 task / run
- 决定是否触发 loop
- 把结果写回 thread

它不直接关心 Jenkins、邮件、Confluence 的细节。

### 5.2 Task Runtime
Task Runtime 是执行对象控制器。

它负责：
- 创建 task / version / run
- 展开 step_run
- 管理状态迁移
- 汇总 artifact / verdict
- 支持 retry / replan / cancel / resume

### 5.3 Loop Engine
Loop Engine 是受约束的决策器。

固定闭环：

```text
plan -> act -> observe -> decide
```

它负责：
- 看当前上下文
- 选下一步 skill / action
- 观察结果
- 决定继续、暂停、转人工还是结束

Loop 只做决策，不做底层系统调用。

### 5.4 Skill Runtime
Skill Runtime 把“一个高层能力”转成可执行调用。

它负责：
- 解析 manifest
- 做 schema 校验
- 补默认参数
- 执行前置检查
- 调用底层 action
- 包装结果

### 5.5 Plugin Runtime
Plugin Runtime 是外部系统适配层。

它负责：
- 解析 plugin manifest
- 选择 provider client
- 处理 authRef
- 执行 action
- 统一错误与 artifact 返回

这一层必须标准化 timeout、retry、idempotency、error normalization。

### 5.6 Retrieval Runtime
Retrieval Runtime 负责上下文检索和拼装。

它从多个池子里拿内容：
- 最近消息
- rolling summary
- milestone summary
- long-term memory
- knowledge base
- task / run facts
- artifact / verdict

然后组装出本轮 loop 和用户回复真正需要的上下文。

### 5.7 Approval Runtime
Approval Runtime 负责高风险动作审批。

比如：
- Jenkins deploy
- 邮件自动回复
- Confluence 写回
- 外部 webhook 触发

它要支持：
- 预览将要执行什么
- 参数摘要
- 风险级别
- 批准 / 拒绝 / 修改后批准

### 5.8 Audit / Event Runtime
Audit / Event Runtime 负责可观测性和恢复。

它记录：
- 每轮 loop 决策
- 每次 skill 调用
- 每个 plugin action
- 每次审批
- 每个状态变更

没有这一层，后面一定很难 debug。

---

## 6. 模块间协作方式

可以把一次典型流程理解成这样：

```text
用户在 thread 提出目标
-> Thread Runtime 恢复上下文
-> Loop Engine 判断下一步
-> Retrieval Runtime 补充知识与历史事实
-> Skill Runtime 执行 skill
-> Plugin Runtime 调用外部系统
-> 结果回流为 artifact / event / audit
-> Task Runtime 更新状态
-> Thread Runtime 给用户反馈
```

如果其中包含高风险动作，则流程中插入 Approval Runtime：

```text
Loop Decision
-> Approval Required
-> User Approves
-> Skill Runtime / Plugin Runtime Execute
```

---

## 7. 为什么要把稳定内核和扩展能力分开

系统里真正稳定的，不是“Jenkins”或者“Confluence”这些名字，
而是这些抽象：
- task
- run
- artifact
- verdict
- thread
- context
- approval
- audit

外部平台会变，插件会变，skill 会变，
但这些核心对象不会轻易变。

所以应该：

### 稳定内核
- domain objects
- runtime orchestration
- state machine
- audit / approval / credential boundary
- retrieval / context composition

### 可扩展外层
- plugins
- provider actions
- skills
- external integrations
- channel adapters

这样未来接 Jenkins、邮件、Confluence、GitHub、Feishu，
都不会破坏系统骨架。

---

## 8. 状态机在整体架构里的位置

这个系统不是简单 CRUD 系统，必须用状态机约束运行过程。

至少有四类状态机：

### 8.1 Task 状态
例如：
- draft
- ready
- active
- blocked
- completed
- archived

### 8.2 Run 状态
例如：
- queued
- planning
- executing
- waiting_approval
- paused
- needs_replan
- completed
- failed
- canceled

### 8.3 StepRun 状态
例如：
- pending
- ready
- running
- succeeded
- failed
- blocked
- skipped

### 8.4 External Action 状态
例如：
- requested
- approved
- dispatched
- running
- succeeded
- failed
- timed_out
- rolled_back

状态机的意义是：
- 避免乱跳状态
- 支持恢复
- 支持审计
- 支持 UI 正确展示

---

## 9. 数据与存储视角

目前方向仍然是：
- **SQLite 单库优先**

原因：
- 本地桌面应用天然适合单机一致性
- schema 演进更简单
- 备份和迁移成本更低
- 对 MVP 足够稳

建议存储分工：

### 9.1 SQLite
存结构化元数据：
- task / run / step
- thread / message / summary / memory
- plugin / skill / knowledge 元信息
- approval / audit / event
- retrieval / citation / snapshot

### 9.2 文件存储
存大对象：
- screenshots
- traces
- raw logs
- attachments
- exported thread contents

SQLite 里只存引用与索引。

---

## 10. 第一版真正该做什么

如果想第一版就跑起来，我会把范围收成下面这些：

### 必做内核
- Task / Run / StepRun / Artifact / Verdict
- ChatThread / ChatMessage / ThreadTaskLink / ThreadRunLink
- rolling summary
- ContextSnapshot
- Loop Engine v1
- Skill Registry v1
- Plugin Runtime v1
- Audit log v1
- Approval flow v1

### 先接低风险能力
- Confluence read-only
- Jenkins status-only
- Email fetch-only

### 暂缓
- 自动回复邮件
- 自动部署到生产
- 高自治多步 loop
- 复杂长期记忆治理
- 多 provider 并发调度

这样第一版就能先验证：
- 核心对象对不对
- loop 能不能稳定推进
- context 恢复值不值
- 外部能力是否能统一接入

---

## 11. 后续扩展方向

第一版稳定后，可以逐步往下扩：

### 11.1 更强执行层
- 多 executor
- 多 run 对比
- 更细粒度 verdict
- 更复杂 retry / replan

### 11.2 更强能力层
- Jenkins deploy
- Email reply
- Confluence update
- GitHub / GitLab / Feishu / Jira 插件

### 11.3 更强知识层
- 分层知识治理
- 过期检测
- 可信度模型
- retrieval trace 可视化

### 11.4 更强治理层
- 审批模板
- 策略引擎
- workspace 级权限模型
- 更完整的审计报表

---

## 12. 一句话总结

verification workbench 的核心，不是做一个“会聊天的工具调用器”，而是做一个：

**以任务为主线、以对话为界面、以上下文恢复为支撑、以 skill/plugin 为扩展、以审批审计为护栏的人机协作验证系统。**

只要这个骨架立住，后面接 Jenkins、邮件、Confluence，或者更复杂的 Agent Loop，都只是外层扩展问题，不会再把系统本身做乱。
