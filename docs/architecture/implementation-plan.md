# verification workbench 实施路线与执行方案

这份文档回答的是另一个更落地的问题：

**前面的整体架构、运行时、插件、skill、知识库这些设计，真正要开始做时，应该按什么顺序执行？**

目标不是把所有模块一次性做完，而是给出一条能逐步落地、尽量减少返工、同时保留扩展性的实施路线。

---

## 1. 执行原则

在开始分阶段之前，先定几条执行原则。

### 1.1 先做稳定内核，再接外部能力

不要一上来就先接 Jenkins、邮件、Confluence。

如果内核还没稳定：
- task / run 模型不清楚
- artifact / verdict 不清楚
- thread / context 不清楚
- 审计 / approval 没有底座

那后面每接一个新系统都会返工。

所以正确顺序应该是：

```text
先把系统骨架立住
-> 再做扩展点
-> 再接低风险能力
-> 再放高风险动作
-> 最后再做自动化与高级编排
```

---

### 1.2 先让链路跑通，再追求智能化

系统前期最重要的不是“AI 多聪明”，而是：
- 对象模型是否稳定
- 状态是否可追踪
- 结果是否可回看
- 出错是否可恢复

所以前期 loop 要薄，skill 要显式，审批要保守。

不要一开始就追求 fully autonomous。

---

### 1.3 先低风险，再高风险

低风险动作先跑通：
- read-only 查询
- 状态获取
- 文档读取
- 邮件拉取
- 构建状态查看

高风险动作后开放：
- 发版
- 回复外部邮件
- 正式文档写回
- 自动更新状态

这样能先把接入、审计、artifact 回流、上下文恢复这些链路验证清楚。

---

### 1.4 先单条主线跑通，再做自动化规则

自动化规则很容易让系统显得“很强”，但也最容易把问题藏起来。

所以应该先保证：
- 用户主动发起时能跑通
- thread -> task -> run -> artifact -> verdict -> thread 回流能闭环

闭环稳了，再挂：
- event -> rule -> skill/action

---

## 2. 实施总路线

建议按六个阶段推进。

```text
Phase 0 统一边界与目录
Phase 1 核心领域模型与存储
Phase 2 线程上下文与恢复
Phase 3 Runtime / Loop / 审批骨架
Phase 4 Plugin / Skill / Knowledge 扩展点
Phase 5 真实能力接入与打样
Phase 6 自动化规则与高级治理
```

---

## 3. Phase 0：统一边界与目录

这一阶段不追求功能，而是把边界先定死。

### 3.1 要完成什么
- 统一目录结构
- 确定模块边界
- 定义核心对象命名
- 决定 SQLite-first 落地方式
- 明确 artifact 存储策略
- 明确代码层次（domain / runtime / capability / knowledge / infra）

### 3.2 输出物
- 目录结构约定
- 模块依赖图
- migration 约定
- 事件命名约定
- action / skill / plugin 命名约定

### 3.3 为什么先做这个
因为如果连模块边界都不稳定，后面文档越多，代码越乱。

---

## 4. Phase 1：核心领域模型与存储

这是第一阶段真正要做的核心。

### 4.1 先实现的核心对象
- `Task`
- `TaskVersion`
- `StepDefinition`
- `Run`
- `StepRun`
- `Artifact`
- `Verdict`
- `ChatThread`
- `ChatMessage`
- `ThreadTaskLink`
- `ThreadRunLink`

### 4.2 数据层先做什么
- SQLite schema 第一版
- migration 机制
- repository / persistence adapter
- artifact metadata 表
- run / step 状态迁移记录

### 4.3 这一阶段不要做什么
- 不做复杂 plugin
- 不做复杂 AI 规划
- 不做自动化规则
- 不做高级知识库

### 4.4 验收标准
- 能创建 task
- 能为 task 创建 version
- 能创建 run 与 step_run
- 能附加 artifact
- 能输出 verdict
- thread 能关联 task / run

也就是说，这阶段要先把“验证主链”立住。

---

## 5. Phase 2：线程上下文与恢复

第一阶段解决“对象存在”，第二阶段解决“上下文不丢”。

### 5.1 先实现什么
- rolling summary
- milestone summary
- ContextSnapshot
- recent messages + summary + task/run facts 的组合恢复
- thread 打开时的上下文装配逻辑

### 5.2 关键设计点
恢复上下文时不要拼全量历史，而是分层取：
- recent messages
- thread summary
- active task / run facts
- latest verdict / artifact
- memory / knowledge hits

### 5.3 验收标准
- 用户一句“继续”时系统能恢复最近现场
- 用户问“为什么失败”时系统能恢复对应 run / artifact / verdict
- 每次恢复都有 ContextSnapshot 记录

这一阶段的目标是让系统开始像一个真正的协作工作台，而不是会失忆的聊天壳。

---

## 6. Phase 3：Runtime / Loop / 审批骨架

前两阶段有了对象和上下文，这一阶段开始做运行中的控制层。

### 6.1 先做哪些 Runtime
- Thread Runtime
- Task Runtime
- Loop Engine（薄版）
- Audit Runtime
- Approval Runtime

### 6.2 Loop 第一版怎么约束
- 一轮只执行一个 action
- 每轮必须有 reason
- 每轮必须产出 observation
- 不能直接绕过审批执行高风险动作
- 必须支持 pause / resume / retry / handoff

### 6.3 审批骨架至少要有
- approval request
- approver
- target action / skill
- decision（approved / rejected / expired）
- audit trace

### 6.4 验收标准
- 系统能在 thread 中触发一个 run
- run 过程中能暂停、继续、失败重试
- 高风险动作能进入审批
- 所有关键状态变化有 audit/event

这一步完成后，系统才算有了“能执行但不乱跑”的骨架。

---

## 7. Phase 4：Plugin / Skill / Knowledge 扩展点

这一步才开始做真正的扩展框架。

### 7.1 Plugin 层
先做：
- plugin manifest
- action registry
- provider runtime
- authRef 解析
- error normalization
- timeout / retry / idempotency

### 7.2 Skill 层
先做：
- skill manifest
- skill registry
- skill versioning
- skill execution log
- risk level / permission mapping

### 7.3 Knowledge 层
先做：
- knowledge source adapter
- knowledge item / chunk / citation
- retrieval session
- basic indexing / metadata filter

### 7.4 验收标准
- 新增一个 plugin 不需要改 loop 主体
- 新增一个 skill 不需要改 domain 主模型
- 新增一个 knowledge source 不需要改 thread / run 主链

这一阶段是在给未来新增功能留插槽。

---

## 8. Phase 5：真实能力接入与打样

前面已经把地基、骨架、插槽都做好了，这一步开始接真实系统。

### 8.1 第一批建议只接低风险能力
建议优先：
- Confluence read-only
- Jenkins status-only
- Email fetch-only

### 8.2 为什么选这三个方向
因为它们能验证三种典型能力：
- 文档知识源
- 外部执行状态源
- 外部消息源

而且都可以先只读，不急着放副作用。

### 8.3 打样目标
- 能读取 Confluence 页面并沉淀 citation
- 能查询 Jenkins run 状态并生成 artifact
- 能拉取邮件线程并生成摘要 / task suggestion

### 8.4 这一步不要急着做什么
- 不要马上自动回复邮件
- 不要马上自动发版
- 不要马上自动改正式文档

### 8.5 验收标准
- 三类外部能力都能回流 artifact / audit
- thread 能引用这些外部结果
- loop 能基于这些能力做下一步决策

---

## 9. Phase 6：自动化规则与高级治理

这一步才开始做更强的自动化。

### 9.1 规则引擎
实现：
- event -> rule -> skill/action
- filter conditions
- cooldown / retry
- failure routing

### 9.2 高级治理
实现：
- capability permission policy
- multi-step approval
- environment isolation
- credential scoping
- operator review dashboard

### 9.3 高风险能力逐步开放
到这一步，才逐步开放：
- Jenkins deploy
- Email reply
- Confluence update
- 外部状态写回

### 9.4 验收标准
- 自动化规则不会绕过审批
- 高风险动作都有审计与回滚策略
- 失败时能准确归因到 rule / skill / plugin / external system

---

## 10. 实施时的目录落点建议

可以按这类结构推进：

```text
src/
  domain/
  runtime/
  plugins/
  skills/
  knowledge/
  infra/
  ui/

docs/architecture/
  system-architecture-overview.md
  runtime-and-state-machine-design.md
  plugin-skill-loop-design.md
  skill-and-knowledge-management.md
  extension-points-and-feature-integration.md
  implementation-plan.md
```

这样前期文档、模型、代码会一一对齐，不容易失控。

---

## 11. 第一版最小可执行切片

如果只能先做一个最小可执行版本，我建议切成：

### MVP Slice
- SQLite schema 第一版
- Task / Run / Artifact / Verdict 主链
- ChatThread / ChatMessage / ThreadTaskLink / ThreadRunLink
- rolling summary
- ContextSnapshot
- 薄 loop
- audit log
- 一个 read-only plugin
- 一个简单 skill

### 这个切片的价值
它已经能证明：
- thread 和 task/run 真能一起工作
- 系统能恢复上下文
- skill / plugin 真的能接进去
- 结果能回流为 artifact / verdict / audit

如果这个切片跑不通，后面做再多高级设计都没意义。

---

## 12. 一句话执行策略

最稳的执行顺序就是：

**先做核心主链，再做上下文恢复，再做运行时与审批骨架，再做 plugin/skill/knowledge 插槽，最后才做真实接入和自动化。**

不要倒着来。  
先接一堆外部系统再补内核，最后一定返工。
