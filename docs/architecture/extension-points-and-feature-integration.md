# 新增功能接入与扩展点设计

这份文档专门回答一个很实际的问题：

**当 verification workbench 后续要不断新增功能时，应该怎么兼容进去，通常需要改动哪里，哪些地方尽量不要动？**

这不是一份“怎么写代码”的细节说明，而是一份面向架构演进的接入指南。

---

## 1. 结论先说

新增功能时，最重要的原则不是“能不能做进去”，而是：

**能不能沿着既定扩展点接进去，而不是把核心主干越改越乱。**

也就是说：
- 先判断新功能属于哪一层
- 再决定它应该挂在哪个扩展点
- 尽量优先扩 Capability / Knowledge / Rule 层
- 只有当它改变系统核心语义时，才去改 Domain / Runtime

如果每来一个新功能都直接改 thread、loop、数据库、UI 全链路，系统很快就会失控。

---

## 2. 先分清两类新增功能

### 2.1 能力型新增

这类新增的本质是：
**系统原本的世界观不变，只是多了一只“手”。**

典型例子：
- Jenkins 发布
- 邮件接收 / 回复
- Confluence 读写
- GitHub Issue / PR 接入
- Feishu 消息发送
- Jira 查询与回写

这类功能通常不应该改动核心验证主链。

更适合通过下面这些扩展点接入：
- Plugin
- Action
- Skill
- Permission / Approval Policy
- Audit / Artifact 映射

### 2.2 领域型新增

这类新增的本质是：
**系统对“验证是什么”“任务是什么”“结果是什么”的理解发生了变化。**

典型例子：
- 新的验证对象类型
- 新的 Verdict 结构
- 新的 Run 编排方式
- 新的知识实体类型
- 新的审批流模型
- 新的上下文恢复策略

这类新增通常不能只靠 plugin / skill 解决，往往需要改：
- Domain Model
- Runtime State Machine
- Schema
- Retrieval / Context Assembly
- UI Projection

所以第一步不是问“怎么接”，而是先问：

**这是在扩能力，还是在改系统理解世界的方式？**

---

## 3. 系统里的固定扩展点

为了让新增功能有稳定插槽，系统设计上应该明确这些 extension points。

### 3.1 Plugin Extension Point

这是最外层的 provider 接入点。

适合承接：
- Jenkins
- Email
- Confluence
- GitHub
- Feishu
- Jira
- Slack

在这里新增功能时，通常要补：
- provider manifest
- provider config schema
- auth requirements
- action handlers
- error mapping
- artifact extraction

这一层只回答：
**怎么接外部系统。**

不要在这里写任务逻辑、状态流转或审批规则。

---

### 3.2 Action Extension Point

Plugin 暴露的不是随意函数，而应该是标准化 action。

例如：
- `jenkins.trigger_pipeline`
- `jenkins.get_pipeline_status`
- `mail.fetch_messages`
- `mail.reply_mail`
- `confluence.read_page`
- `confluence.update_page`

新增 action 时，通常需要定义：
- action name
- input schema
- output schema
- timeout
- retry policy
- idempotency behavior
- risk level

这一层回答：
**系统把外部能力抽象成什么动作。**

---

### 3.3 Skill Extension Point

Skill 是能力编排层。

它把多个 action 封成更接近用户目标的能力。

例如：
- `deploy_to_staging_and_wait`
- `fetch_unread_incident_mails`
- `append_verification_result_to_release_note`
- `summarize_recent_failures`

新增 skill 时，通常要补：
- skill manifest
- required actions
- input / output schema
- risk level
- preconditions
- optional approval requirement
- execution log mapping

这一层回答：
**用户想做的事，系统怎么把它打包成一个可调用能力。**

---

### 3.4 Rule / Automation Extension Point

很多新增功能不是“用户点一下”，而是“某个事件发生后自动触发”。

例如：
- PR merged 后触发部署
- 收到 incident 邮件后生成 task
- run 完成后回写 Confluence
- verdict 失败后发通知

这类能力最好挂在 rule 层，而不是塞进某个 plugin 里。

新增 rule 时，通常会定义：
- trigger event
- filter conditions
- selected skill / action
- approval requirement
- retry / cooldown policy

这一层回答：
**什么时候自动触发这个功能。**

---

### 3.5 Knowledge Source Extension Point

如果新增功能是“接更多知识来源”，最好走知识源扩展点。

例如：
- Confluence 文档源
- 邮件线程源
- 事故复盘源
- 发布记录源
- Jira issue 源

新增 knowledge source 时，通常要补：
- source adapter
- fetch / sync strategy
- chunking strategy
- metadata mapping
- retention / freshness policy
- citation format

这一层回答：
**系统从哪里知道新信息。**

---

### 3.6 Approval / Permission Extension Point

不是所有新功能都能直接放开。

像下面这些动作通常都属于高风险：
- 发版
- 回复外部邮件
- 修改 Confluence 正式页面
- 大规模通知
- 写回 Jira / GitHub 状态

所以新增功能时，经常还要补：
- capability permissions
- approval policy
- actor restrictions
- audit fields

这一层回答：
**谁可以做、什么情况下可以做、做完怎么追踪。**

---

## 4. 需要尽量稳定不动的核心主干

要想让新增功能兼容进去，前提是先把“尽量不该动的地方”守住。

这些通常属于稳定内核：
- `Task`
- `TaskVersion`
- `Run`
- `StepRun`
- `Artifact`
- `Verdict`
- `ChatThread`
- `ContextSnapshot`
- 核心 loop 闭环
- 基础 audit/event 模型

也就是说：

### 不要因为接 Jenkins 就改 Task 定义
### 不要因为接邮件就改 Thread 主模型
### 不要因为接 Confluence 就把知识库和 skill 混成一层

如果外围扩能力总要把内核拆开重来，说明扩展点设计失败了。

---

## 5. 新增功能通常要改哪些地方

这里给一个比较实用的 checklist。

### 5.1 新增一个外部能力（例如 Jenkins / 邮件 / Confluence）

通常改这些地方：

1. `plugins/<provider>/`
   - provider client
   - actions
   - manifest

2. `skills/`
   - 新增一个或多个 skill
   - 把 provider action 组合成目标导向能力

3. `registry`
   - action registry
   - skill registry

4. `auth / permission`
   - 新增 credential type 或 capability mapping

5. `audit / artifact mapping`
   - 外部调用日志
   - 外部链接 / diff / 附件映射为 artifact

6. `UI config`
   - provider 配置页
   - skill 启停页
   - action/approval 状态展示

通常**不需要**直接改：
- Task 主模型
- Run 主链
- Thread 主结构

---

### 5.2 新增一个自动流程

例如：
- 邮件来了自动建 task
- 部署完成自动回写文档
- verdict 失败自动通知

除了上面那些，还通常要改：

7. `event definitions`
8. `rule engine / automation config`
9. `loop decision policy`
10. `approval policy`

也就是说，自动流程一般比“单点能力”多一层事件与规则。

---

### 5.3 新增一个核心业务概念

例如：
- 引入新的验证结果类型
- 引入新的 environment / executor 维度
- 引入新的任务生命周期
- 引入新的知识对象层次

这类通常要改：

11. `domain model`
12. `schema migration`
13. `state machine`
14. `runtime projection`
15. `retrieval/context assembly`
16. `UI read model`

这类变更要更谨慎，因为它已经不是“接功能”，而是在改系统骨架。

---

## 6. 典型接入示例

### 6.1 Jenkins 发布接入

通常改动：
- Plugin：Jenkins client + action handlers
- Skills：`deploy_to_env` / `deploy_and_wait`
- Approval：生产发布需要审批
- Artifact：build url / console log / deployment record
- Audit：谁触发、何时触发、结果如何

通常不需要改：
- Task / Run 核心结构

原因是：
Jenkins 发布只是系统多了一只“执行外部动作的手”，并没有改变 verification 的核心主链。

---

### 6.2 邮件收件与回复接入

通常改动：
- Plugin：mail fetch / thread / reply
- Skills：`fetch_unread_incident_mails`、`draft_reply`、`reply_after_approval`
- Knowledge Source：把邮件线程接入知识检索
- Rule：收到高优先级 incident 自动建 task
- Approval：自动回复前必须人工确认

这里要注意：
邮件既可能是外部动作能力，也可能是知识来源。  
所以它会同时接入 Capability Layer 和 Knowledge Layer。

---

### 6.3 Confluence 读写接入

通常改动：
- Plugin：read / search / update / append section
- Skills：`read_release_note`、`append_verification_result`
- Knowledge Source：Confluence 页面接入检索与引用
- Approval：正式空间写入需要审批
- Artifact：page diff / page url / version id

如果只是“读”，通常主要落在知识层。  
如果是“写回”，就还要接审批和审计。

---

## 7. 判断新功能该落哪层的简单方法

可以直接问四个问题：

### Q1. 这是不是在接一个新的外部系统？
如果是，优先走 plugin。

### Q2. 这是不是在组合已有动作来完成一个用户目标？
如果是，优先走 skill。

### Q3. 这是不是在增加一个新的自动触发流程？
如果是，优先走 event/rule。

### Q4. 这是不是在改变系统对任务、运行、结果、知识的核心理解？
如果是，才去改 domain/runtime/schema。

这个判断能挡掉大量无意义的“大改”。

---

## 8. 设计上的几个硬原则

### 8.1 主流程靠协议扩展，不靠 if/else 扩展

不要在 loop 里写：
- `if provider == jenkins`
- `if provider == confluence`
- `if provider == mail`

这些都应该通过：
- registry
- manifest
- action schema
- capability model

来解耦。

---

### 8.2 领域稳定，外围可变

稳定的是：
- task / run / artifact / verdict
- thread / snapshot
- audit / approval

易变的是：
- plugin
- skill
- rule
- knowledge source adapter

这就是为什么扩展点要尽量放外围。

---

### 8.3 知识不等于能力

接一个 Confluence 文档源，不等于自动会写文档。  
接一个邮件源，不等于自动会回邮件。

读取 / 检索 与 写入 / 副作用 是两类事情，权限和风险完全不同。

---

### 8.4 决策不等于执行

Loop 可以决定“该调用 deploy_to_env”，  
但真正执行部署的是 skill/runtime/plugin，  
并且可能被 approval policy 拦住。

这个边界一定要守住。

---

## 9. 一个建议的接入流程模板

以后每新增一个功能，都可以按这个模板走：

### Step 1：功能归类
- 能力型新增
- 领域型新增
- 知识源新增
- 自动化规则新增

### Step 2：确定扩展点
- plugin
- action
- skill
- rule
- knowledge source
- domain/runtime

### Step 3：列出改动范围
- 目录
- schema
- runtime
- UI
- permission
- audit

### Step 4：确定风险等级
- read-only
- internal write
- external write
- production side effect

### Step 5：确定 rollout 方式
- draft
- internal only
- approval required
- limited project rollout
- full rollout

这个模板本身就能减少很多架构争议。

---

## 10. 一句话总结

新增功能最稳的接入方式是：

**优先把变化吸收到 plugin / skill / rule / knowledge source 这些外围扩展点里；只有当新功能改变系统核心语义时，才去动 domain、runtime 和 schema。**

说白了就是：

**先往插槽里接，别先拆主梁。**
