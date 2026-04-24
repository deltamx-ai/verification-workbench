# 运行时与状态机设计

这份文档继续补全 verification workbench 的核心骨架，重点回答两个问题：

1. 系统运行时到底怎么组织
2. task / run / step / thread / plugin action 的状态到底怎么流转

这份稿子不是讲概念，而是尽量往“能直接落实现”的方向收。

---

## 1. 设计目标

这里的运行时不是单纯指一个进程。

它更像一个统一协调层，负责把这些东西接起来：
- 用户线程与对话上下文
- task / run / step 的执行状态
- skill / plugin action 的实际调用
- knowledge retrieval 的上下文补充
- artifact / verdict / audit 的结果回流

核心目标有四个：

1. **状态可追踪**：任何一步为什么执行、执行到哪、为什么停，都要说得清。
2. **副作用可控制**：邮件回复、Jenkins 发布、Confluence 写回这类动作不能乱跑。
3. **失败可恢复**：中断、报错、超时后，系统要能从中间继续，不是从头再来。
4. **线程与任务解耦但可关联**：thread 推进讨论，task / run 推进执行，二者互相引用，但不要混成一团。

---

## 2. 运行时总览

建议把运行时拆成这几个核心模块：

- **Thread Runtime**：处理对话输入、上下文恢复、线程级事件
- **Task Runtime**：管理 task / run / step 的生命周期
- **Loop Engine**：执行 `plan -> act -> observe -> decide`
- **Skill Runtime**：解析 skill，绑定输入，调用 action
- **Plugin Runtime**：真正对接外部系统
- **Retrieval Runtime**：检索知识库、摘要、历史事实
- **Audit / Event Runtime**：记录所有状态变化和外部动作
- **Approval Runtime**：高风险动作审批与人工接管

大概关系：

```text
User / Thread
  -> Thread Runtime
    -> Retrieval Runtime
    -> Loop Engine
      -> Skill Runtime
        -> Plugin Runtime
      -> Task Runtime
      -> Approval Runtime
    -> Audit / Event Runtime
```

这里最关键的一条原则是：

**Loop 负责决策，Runtime 负责执行边界，状态机负责约束流转。**

---

## 3. Thread Runtime

Thread Runtime 的职责不是“聊天 UI”。

它更像线程级控制器，负责：
- 接收消息
- 解析用户意图
- 恢复当前 thread context
- 找到关联的 task / run
- 决定是否触发 loop
- 把 loop 产出写回 thread

它需要处理三类情况：

### 3.1 普通对话
只更新 thread / message / summary / memory，不触发 task execution。

### 3.2 任务推进
用户在对话里明确要求：
- 新建一个验证任务
- 继续执行某个 run
- 补充某个 step 的约束
- 触发外部动作

这时要把 thread 和 task / run 建立 link，再交给 Task Runtime / Loop Engine。

### 3.3 恢复现场
当用户过了一段时间回来继续说：
- “继续”
- “现在怎样了”
- “为什么失败”

Thread Runtime 要优先恢复：
- recent messages
- rolling summary
- 活跃 task / run
- 最近 verdict / artifact
- 最近 loop trace / pending approval

也就是说，线程层不是执行层，但它决定从哪个执行现场接着走。

---

## 4. Task Runtime

Task Runtime 管的是执行对象本身。

建议明确分开：
- `Task`：长期存在的问题或目标
- `TaskVersion`：任务在某一时刻冻结的定义
- `Run`：某次实际执行
- `StepDefinition`：标准步骤模板
- `StepRun`：本次 run 里的具体步骤实例

### 4.1 Task Runtime 负责什么
- 创建 task / version / run
- 根据 plan 展开 step_run
- 管控状态迁移
- 处理 retry / replan / cancel
- 汇总 artifact / verdict
- 更新 run 结论

### 4.2 为什么要把 Task 和 Run 分开
因为：
- 一个任务可能执行很多次
- 不同 run 可能在不同环境、不同版本上跑
- 同一个 task 的 success criteria 可能逐步调整

如果不分开，后面所有历史都会搅在一起。

---

## 5. Loop Engine

Loop Engine 是运行时里最像“大脑”的部分，但也不要让它太自由。

推荐做成一个**受约束 loop**：

```text
load context
-> evaluate objective
-> choose next step
-> choose skill/action
-> execute
-> observe result
-> update state
-> decide continue / pause / retry / handoff / finish
```

### 5.1 Loop 每轮必须产出什么
每轮都应该有结构化记录：
- `reason`：为什么做这步
- `selected_action`：选了哪个 skill / action
- `expected_outcome`：预期得到什么
- `observation`：实际结果如何
- `decision`：下一步是继续、暂停、重试还是转人工

### 5.2 Loop 不该直接做什么
- 不直接操作数据库细节
- 不直接保存凭据
- 不直接调用外部 SDK
- 不直接越过审批执行高风险动作

它只下决策，不碰底层脏活。

---

## 6. Skill Runtime

Skill Runtime 负责把 loop 选出来的 skill 真正变成一次可执行调用。

它处理：
- skill manifest 解析
- 参数校验
- 默认值填充
- 前置条件检查
- 依赖 action 展开
- 执行结果包装

### 6.1 Skill 运行的三种形态

#### a. 单 action skill
比如：
- 读取 Confluence 页面
- 查询 Jenkins 构建状态

#### b. 组合 skill
比如：
- 触发部署并等待结束
- 拉取未读邮件并抽取问题摘要

#### c. reasoning skill
比如：
- 总结最近失败原因
- 从知识库中提取与当前任务相关的风险

### 6.2 Skill Runtime 要做保护
- schema 校验失败直接拦住
- 缺少 authRef 不允许执行外部动作
- 高风险 skill 必须进入审批流程
- 同一个 request_id 支持幂等

---

## 7. Plugin Runtime

Plugin Runtime 是外部系统适配层。

它只负责：
- 找到 provider client
- 注入配置和凭据引用
- 执行 action
- 统一处理错误
- 统一返回 artifact / data / error

它不要关心：
- 任务业务逻辑
- 为什么要执行这个动作
- 下一步怎么办

这是上层的事。

### 7.1 Plugin Runtime 最重要的统一能力
- timeout
- retry
- rate limit
- idempotency key
- auth resolution
- error normalization
- artifact extraction

如果这一层不统一，未来每接一个新插件都要重新造轮子。

---

## 8. Retrieval Runtime

Retrieval Runtime 不只是“搜知识库”。

它要统一组织几类来源：
- 最近 thread messages
- thread summaries
- long-term memory
- task / run facts
- 知识库文档
- 外部系统返回的结构化记录

### 8.1 检索结果不能只返回文本
至少应该返回：
- 内容片段
- 来源
- 类型
- 时间
- 可信度
- 是否属于当前 task / thread

### 8.2 检索的目标
不是召回越多越好，而是给 loop 一份“够决策用”的上下文包。

也就是：
- 少而准
- 可引用
- 可审计

---

## 9. Approval Runtime

这是整个系统里非常容易被忽略、但实际特别关键的一层。

凡是有副作用的动作，都不该直接放行。

### 9.1 建议进入审批的动作
- Jenkins deploy / rollback
- 邮件 reply / send
- Confluence update / create
- 修改生产环境配置
- 删除或覆盖外部数据

### 9.2 审批对象要包含什么
- 谁发起的
- 来自哪个 task / run / thread
- 准备执行哪个 skill / action
- 输入参数摘要
- 预期副作用
- dry-run / diff preview
- 超时时间

### 9.3 审批后的状态
- approved
- rejected
- expired
- cancelled

审批通过后，Loop 才能继续往下跑。

---

## 10. Audit / Event Runtime

建议把事件和审计拆开但关联。

### 10.1 Event
Event 偏“系统发生了什么”：
- run created
- step started
- step finished
- approval requested
- plugin action completed

### 10.2 Audit
Audit 偏“谁做了什么、用了什么参数、造成了什么副作用”：
- 谁触发了邮件回复
- 用了哪个 authRef
- 写回了哪个 page
- 改动前后摘要是什么

Event 更适合驱动流程。Audit 更适合追责、复盘、合规。

---

## 11. 核心状态机设计

这里至少要有五套状态机：
- thread state
- task state
- run state
- step_run state
- approval state

必要时还可以补：
- plugin_action state
- retrieval_session state

---

## 12. Thread 状态

Thread 不建议做太重，但可以保留少量协作状态：

- `active`
- `waiting_user`
- `waiting_system`
- `blocked`
- `archived`

解释：
- `active`：线程正常进行中
- `waiting_user`：系统需要用户补信息或审批
- `waiting_system`：后台还有 run / approval / plugin action 在推进
- `blocked`：线程依赖外部条件，暂时推进不了
- `archived`：线程已结束，只读保留

---

## 13. Task 状态

Task 表示长期目标，不应频繁跳状态。

建议：
- `draft`
- `ready`
- `in_progress`
- `blocked`
- `completed`
- `cancelled`

### 13.1 转移逻辑
- 新建任务 -> `draft`
- 补齐目标和约束 -> `ready`
- 创建 run 开始执行 -> `in_progress`
- 外部依赖缺失 / 需要人工处理 -> `blocked`
- 达到 success criteria -> `completed`
- 被放弃 -> `cancelled`

---

## 14. Run 状态

Run 是最关键的一套状态机，因为它代表一次实际推进。

建议：
- `created`
- `planned`
- `ready`
- `running`
- `waiting_approval`
- `waiting_external`
- `needs_replan`
- `paused`
- `completed`
- `failed`
- `cancelled`

### 14.1 为什么要把 waiting_approval 和 waiting_external 分开
因为它们不是一回事：
- `waiting_approval`：卡在人
- `waiting_external`：卡在系统或第三方

后面看监控、超时、提醒策略时，这两个必须能分开看。

### 14.2 needs_replan 什么时候出现
例如：
- 原计划执行后发现前提不成立
- 检索到新信息导致方案失效
- 连续失败但不是致命错误
- 某个 skill 返回“目标不明确/条件不足”

这时候不该直接 failed，而是要求 loop 回到 planning。

---

## 15. StepRun 状态

StepRun 建议更细一些：
- `planned`
- `ready`
- `running`
- `waiting_approval`
- `waiting_external`
- `done`
- `failed`
- `blocked`
- `skipped`
- `cancelled`

### 15.1 blocked 和 failed 的区别
- `blocked`：不是这一步做错了，而是暂时做不了
- `failed`：已经执行了，但结果明确失败

这两个不能混。

### 15.2 skipped 的用处
replan 或条件变化后，有些原步骤不需要做了。
这不是 failed，也不是 cancelled，更适合记成 skipped。

---

## 16. Approval 状态

建议：
- `pending`
- `approved`
- `rejected`
- `expired`
- `cancelled`
- `executed`

这里 `approved` 和 `executed` 也要分开。

因为：
- 有可能审批通过后还没真正执行
- 执行时可能又失败

审批只是允许，不等于已经成功完成副作用。

---

## 17. PluginAction 状态

如果系统里外部动作很多，建议单独建 plugin action execution 记录。

状态可设为：
- `queued`
- `running`
- `succeeded`
- `failed`
- `timed_out`
- `retrying`
- `cancelled`

这样可以更好地观察：
- 哪些外部系统最慢
- 哪些 action 最容易失败
- 是否是 provider 问题还是上层 loop 问题

---

## 18. 状态迁移原则

状态迁移不要散落在前端和业务逻辑里硬改。

建议统一经过 transition service：
- 校验当前状态是否合法跳转
- 记录 transition reason
- 记录 operator / source
- 发 domain event
- 写 audit log

比如：

```text
run.running -> run.waiting_approval
reason: selected skill requires approval
source: loop_engine
```

如果未来要查“为什么停在这里”，这条记录会非常值钱。

---

## 19. Retry / Replan / Human Handoff

这三件事要明确区分。

### 19.1 Retry
适用于短暂错误：
- 网络抖动
- provider timeout
- 页面偶发 flaky

特点：
- 目标没变
- 步骤没变
- 只是再试一次

### 19.2 Replan
适用于计划需要调整：
- 当前方案走不通
- 缺少前置条件
- 新信息推翻旧计划

特点：
- 目标不一定变
- 但步骤结构会变

### 19.3 Human Handoff
适用于系统不该继续自己决定：
- 风险太高
- 缺少关键信息
- 连续失败无法判因
- 涉及审批或业务判断

特点：
- 需要明确等人
- 恢复后可继续原 run，也可新开 run

---

## 20. 超时与恢复

运行时要天然支持中断恢复。

至少要保存：
- 当前 run state
- 当前 step_run state
- 最近一轮 loop decision
- 待执行 approval
- 最近 action result
- 当前 context snapshot

这样系统重启后才能继续，而不是丢现场。

### 20.1 恢复策略
- 如果 run 在 `waiting_approval`，恢复后直接检查审批结果
- 如果 run 在 `waiting_external`，恢复后轮询外部状态或重连监听
- 如果 run 在 `running` 但没有心跳，转成 `paused` 或 `needs_replan`

---

## 21. 幂等性设计

凡是有副作用的动作，都要考虑重复执行。

建议：
- 每次 action execution 都带 `idempotency_key`
- 外部调用前先查本地执行记录
- 对同一请求重复提交时优先返回已有结果

尤其适用于：
- 发邮件
- 发版
- 创建 Confluence 页面
- 写回评论

不然恢复或重试时很容易一口气发两遍。

---

## 22. 建议的数据对象

运行时最少还需要这些对象：
- `loop_sessions`
- `loop_iterations`
- `approvals`
- `plugin_action_executions`
- `state_transitions`
- `event_outbox`
- `audit_logs`
- `retrieval_sessions`

这些对象让“过程”变成可落库、可恢复、可分析的东西。

---

## 23. 一条完整链路示例

以“用户要求发 Jenkins 部署，再把结果写回 Confluence”为例：

```text
User message
-> Thread Runtime 识别为任务推进
-> Retrieval Runtime 恢复当前 task / run / release context
-> Loop Engine 生成下一步：deploy_to_env
-> Skill Runtime 解析 deploy_to_env
-> Approval Runtime 创建 deploy 审批
-> 用户批准
-> Plugin Runtime 调用 Jenkins trigger_pipeline
-> PluginAction 进入 running
-> Jenkins 返回 run url
-> Loop Engine 观察结果，进入 waiting_external
-> 外部状态完成后，Loop Engine 决定 append_verification_result
-> Approval Runtime 检查写回 Confluence 是否需要审批
-> Plugin Runtime 调用 Confluence append_section
-> Artifact / Audit / Event 全部写回
-> Run completed
```

如果其中 Jenkins 失败：
- step_run -> failed
- run -> needs_replan 或 failed
- loop 写明失败归因
- thread 进入 waiting_user 或 active

---

## 24. MVP 建议

第一版不要把所有状态都做满。

推荐先落：

### 必做
- task / run / step_run 状态机
- approval 状态机
- state transition log
- plugin action execution log
- loop iteration log

### 第二阶段再做
- thread 协作状态
- retrieval session 精细追踪
- event outbox 与 rule engine 深度集成

先把执行主链做稳，比一开始做很多外围状态更重要。

---

## 25. 一句话结论

如果想让 verification workbench 真的可落地，运行时必须是：

**线程负责承接对话，task/runtime 负责推进执行，loop 负责决策，skill/plugin 负责动作落地，状态机负责约束和恢复。**

真正稳的系统，不是因为它会“自动做事”，而是因为它能清楚地记录：
**为什么做、做到哪、卡在哪、接下来该由谁接手。**
