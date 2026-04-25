# 流程定义 / 定时执行 / 自动执行设计

这份文档专门回答一个关键问题：

**系统能不能支持“用户说一个流程”，然后后续按定时或事件自动执行？如果能，应该怎么设计？**

结论是：
**必须支持，而且这应该是 verification workbench 从“会执行单次任务”走向“可持续协作自动化系统”的关键能力。**

但这个能力不能做成一句自然语言临时现猜。  
更稳的做法是把流程定义、触发器、执行器、校验、审批、审计拆开，形成一条可追踪、可恢复、可治理的执行链。

---

## 1. 核心能力拆分

如果用户说：
- “每天 9 点搜最新发布消息，总结一下写日报”
- “每小时查 Jenkins 状态，异常就发通知”
- “收到特定邮件就拉取上下文并生成处理建议”

系统不能只是把这句话存起来。  
它要拆成几个明确对象：

1. **Workflow Definition**
   - 这个流程要做什么
   - 输入来源是什么
   - 处理步骤是什么
   - 输出写到哪里

2. **Trigger**
   - 定时触发（schedule）
   - 事件触发（event）
   - 手动触发（manual）

3. **Execution Plan / Runtime**
   - 实际如何执行 search/read/write/validate
   - 如何串 skill / plugin / rule

4. **Policy**
   - 是否允许外部写
   - 是否需要审批
   - 失败后如何重试

5. **Audit / Recovery**
   - 每次执行都留痕
   - 失败后可恢复
   - 可追踪到每次自动动作做了什么

---

## 2. 系统里的角色分工

这个能力应该由多个模块协作，而不是某一个 runtime 独吞。

### 2.1 Workflow Definition Layer
负责把流程定义成结构化对象。

### 2.2 Trigger Layer
负责决定**什么时候启动一次执行**。

### 2.3 Runtime Layer
负责真的去跑：
- search
- read
- write
- validate
- decide

### 2.4 Approval / Permission Layer
负责拦住高风险动作。

### 2.5 Audit / Event Layer
负责记录：
- 为什么触发
- 跑了哪些步骤
- 结果是什么
- 为什么失败

也就是说：

```text
workflow definition
-> trigger fires
-> execution runtime starts
-> SRWV / skill / plugin action executes
-> validation / approval
-> audit / result / recovery
```

---

## 3. Workflow Definition 怎么存

自然语言只是输入形式，不应是最终存储形式。

用户说一个流程后，系统应该把它解析成结构化 workflow。

推荐字段：

- `name`
- `goal`
- `trigger_type`
- `trigger_config`
- `input_sources`
- `execution_pattern`
- `steps`
- `write_targets`
- `validation_policy`
- `permission_policy`
- `retry_policy`
- `enabled`

示意：

```json
{
  "name": "daily release digest",
  "goal": "Search latest release updates and write daily summary",
  "trigger_type": "schedule",
  "trigger_config": {
    "cron": "0 9 * * *",
    "timezone": "Asia/Shanghai"
  },
  "input_sources": ["confluence", "web", "browser_live_search"],
  "execution_pattern": "srwv",
  "steps": [
    "search latest release messages",
    "read selected sources",
    "write report draft",
    "validate freshness and citations"
  ],
  "write_targets": ["daily_report"],
  "validation_policy": "freshness_required",
  "permission_policy": "local_write_only",
  "retry_policy": {
    "max_attempts": 3
  },
  "enabled": true
}
```

这样后面它才是一个真正可执行的对象。

---

## 4. Trigger 需要支持哪些类型

至少三类：

### 4.1 Manual Trigger
用户手动点一下执行。  
适合调试、补跑、回放。

### 4.2 Schedule Trigger
通过 cron / schedule 定时执行。  
例如：
- 每天 9 点
- 每小时
- 工作日早晚各一次

### 4.3 Event Trigger
事件来了就执行。  
例如：
- 收到邮件
- Jenkins 状态变化
- 新消息进入某频道
- task 状态变为 failed
- 新的 Confluence 页面更新

事件触发模式大概是：

```text
event
-> event filter
-> match workflow rule
-> create execution instance
```

---

## 5. Schedule Trigger 的设计要点

定时执行不是只存一个 cron 就完了，还要补很多细节：

- `cron` 表达式
- `timezone`
- `enabled`
- `start_at`
- `end_at`
- `next_run_at`
- `last_run_at`
- `misfire_policy`
- `max_concurrency`

### 5.1 Misfire Policy
比如机器关机、调度器暂停，错过了执行时间，怎么办？

可以支持：
- `skip`
- `run_once_when_resumed`
- `catch_up_all`

通常推荐默认：
- `run_once_when_resumed`

这样既不会补太多历史任务，也不会完全丢掉一次执行。

---

## 6. Event Trigger 的设计要点

事件触发需要的不只是“监听到了事件”，还要有过滤和去重。

例如：
- 邮件来了，但不是这个流程关心的主题
- Jenkins 频繁推状态，不应该每次都触发
- 同一条 Discord 消息不能反复触发同一流程

所以 event trigger 至少要有：
- event type
- filter condition
- dedupe key
- cooldown window
- retry policy

例如：

```json
{
  "event_type": "email.received",
  "filters": {
    "subject_contains": "incident",
    "sender_in": ["ops@example.com"]
  },
  "dedupe_key": "message_id",
  "cooldown_seconds": 300
}
```

---

## 7. 执行器怎么复用现有设计

这里的关键是：
**不要为自动执行再发明一套新的执行引擎。**

已有设计里：
- Loop Engine
- SRWV Executor
- Skill Runtime
- Plugin Runtime
- Validation Policy

这些都应该直接复用。

也就是：
- workflow 触发后
- 生成一个 execution instance
- 交给现有 runtime 跑
- 只是这次不是用户手动触发，而是 schedule / event 触发

所以整体关系是：

```text
workflow trigger
-> execution instance
-> Loop Engine
  -> SRWV Executor / Skill Runtime
  -> Plugin Runtime
  -> Validation Policy
  -> Approval / Audit
```

这点很重要，不然系统会变成“手动一套逻辑，自动又一套逻辑”。

---

## 8. 自动执行也必须走 Validation

不要因为是自动任务，就跳过 validation。

尤其是：
- search 出来的来源是不是够新
- read 出来的事实有没有冲突
- write 的目标是不是允许
- 外部写是否需要审批

自动任务如果跳过这层，很容易变成“稳定地自动犯错”。

所以建议：
- 自动流程默认必须 validation
- validation 不通过时不允许进入 external write
- validation issue 写入 audit

---

## 9. 审批模型怎么接

这块尤其重要。

你说一个流程，不代表它就可以永远无脑自动跑。  
流程里面的某些步骤可能必须审批。

比如：
- 自动生成日报草稿：可以直接跑
- 自动发外部邮件：要审批
- 自动更新 Confluence 正式页面：要审批
- 自动触发发布：强审批

所以 workflow 级别要支持：
- `local_write_only`
- `external_write_requires_approval`
- `fully_manual_external_write`

执行中一旦遇到高风险 write：

```text
workflow run
-> write_external prepared
-> validation pass but approval required
-> enter needs_approval
-> wait decision
-> continue or abort
```

---

## 10. 重试与恢复怎么设计

自动任务一定会遇到：
- timeout
- 外部系统报错
- 页面没打开
- 数据源暂时不可用
- 审批超时

所以每个 workflow run 至少要支持：
- `retry_count`
- `max_attempts`
- `next_retry_at`
- `last_error`
- `recovery_state`

### 10.1 推荐重试策略
- search / read 阶段：允许自动重试
- local write：可自动重试
- external write：谨慎重试，最好先确认幂等
- approval timeout：默认停止，不自动继续

---

## 11. 数据模型建议

如果把这块也落库，建议至少有：

```text
workflow_definitions
workflow_definition_versions
workflow_triggers
workflow_runs
workflow_run_steps
workflow_run_events
workflow_run_artifacts
workflow_run_approvals
```

### 11.1 workflow_definitions
定义流程本体。

### 11.2 workflow_definition_versions
保存流程的具体版本，避免定义改了后历史执行不可追。

### 11.3 workflow_triggers
保存 trigger 配置：schedule / event / manual。

### 11.4 workflow_runs
每次实际执行一条记录。

### 11.5 workflow_run_steps
记录本次执行中每个步骤：
- search
- read
- write
- validate
- decision

### 11.6 workflow_run_events
记录触发与运行时事件。

### 11.7 workflow_run_artifacts
把产出挂上去。

### 11.8 workflow_run_approvals
记录审批链路。

---

## 12. 用户体验怎么设计

从用户角度看，最好是这样的：

### 12.1 定义流程
用户说：
- “每天 9 点搜最新发布消息，写日报草稿”

系统把它解析成结构化 workflow，并让用户确认：
- 触发时间
- 数据来源
- 写入目标
- 是否允许外部写
- 审批规则

### 12.2 查看流程
用户能看到：
- 这个流程现在开着没
- 下次什么时候跑
- 最近一次跑的结果
- 最近失败原因
- 是否卡在审批

### 12.3 手动补跑
用户可以点：
- 立即执行一次
- 跳过下一次
- 暂停流程
- 修改 trigger

也就是说，自动执行不该是“黑箱后台魔法”，而应该是可见、可控、可追踪的。

---

## 13. 一句话结论

如果要满足“用户说一个流程，后续定时执行或自动执行”，最稳的设计方式是：

**把自然语言流程先落成结构化 workflow definition，再用 schedule / event trigger 触发 execution instance，复用现有 Loop / SRWV / Skill / Plugin Runtime 执行，并全程走 validation、approval、audit、recovery。**

不要把自动执行做成临时猜测，也不要另起一套运行时。  
手动执行和自动执行应该共用同一条主干，只是触发方式不同。
