# 可观测性与调试设计

这份文档专门解决一个很现实的问题：

**verification workbench 一旦跑起来，出了错、卡住、结果不对、自动流程误触发时，怎么查？怎么定位？怎么回放？**

如果没有可观测性，前面所有 runtime、workflow、SRWV、plugin、validation 设计最后都会变成黑箱。

所以这里要做的不是“加日志”这么简单，而是设计一套：
- trace
- timeline
- event
- metrics
- artifact
- replay

共同组成的调试体系。

---

## 1. 可观测性要覆盖哪些层

至少覆盖：

- Thread Runtime
- Workflow Runtime
- SRWV Executor
- Skill Runtime
- Plugin Runtime
- Validation Policy
- Approval Runtime
- Retrieval Layer

也就是说，不能只看“最终失败了”，而是要能一路看到：
- 谁触发的
- 为什么触发
- 查了哪些来源
- 抽了哪些事实
- 为什么 validation 没过
- 是否卡在 approval
- 哪个 provider 超时了

---

## 2. Trace Model

建议把一次完整执行抽象成一条 trace。

### 2.1 Trace 范围
一条 trace 可以覆盖：
- 一个 workflow run
- 一个 SRWV loop
- 一个 skill execution
- 一个 plugin action call

### 2.2 关联方式
每层都要有：
- `trace_id`
- `parent_trace_id`
- `span_id`

这样后面能看出调用树：

```text
workflow run trace
  -> srwv trace
    -> search provider span
    -> read extractor span
    -> validation span
  -> approval span
  -> external write span
```

---

## 3. Timeline 视图

日志只是原始材料，真正对人有用的是 timeline。

建议每个 run 都能生成 timeline：
- triggered
- queued
- step started
- source fetched
- facts extracted
- validation failed / passed
- approval requested
- approval approved / rejected
- external write executed
- completed / failed

这会极大提升调试效率。

---

## 4. Search / Read / Write / Validate 的观测点

SRWV 是核心执行模式，所以最好单独定义观测字段。

### 4.1 Search
记录：
- search provider
- query / missing info
- source candidates count
- selected candidates
- provider latency
- timeout / fallback 情况

### 4.2 Read
记录：
- read documents count
- fact count
- conflict count
- extraction confidence 分布

### 4.3 Write
记录：
- write type（draft / local / external）
- target
- approval required?
- write payload hash
- execution result

### 4.4 Validate
记录：
- policy name
- checks executed
- issues count
- pass / fail
- next decision

---

## 5. Provider Call Log

外部集成要想好调试，provider call log 必须独立可查。

至少记录：
- provider name
- action name
- request summary
- response summary
- latency
- status
- error code
- retry count
- idempotency key

注意是 summary，不是盲目把所有敏感信息打进日志。

---

## 6. Approval Trace

审批是最容易让用户觉得“系统卡住”的地方。  
所以它必须非常可见。

至少要记录：
- 谁发起审批
- 为什么需要审批
- 审批目标是什么
- 当前状态
- 已等待多久
- 最终决策
- 决策备注

这样用户一看就知道：
不是系统死了，是卡在审批。

---

## 7. Failure Attribution

错误定位不能只显示“失败了”。  
要尽量归因到层级：

- definition problem
- trigger problem
- retrieval problem
- validation problem
- approval problem
- provider problem
- runtime bug
- environment/config problem

这点很重要，因为它会直接决定下一步到底该找谁处理。

---

## 8. Replay 能力

可观测性做到后面，最有价值的是 replay。  
建议支持：
- replay a workflow run
- replay from failed step
- replay an SRWV iteration
- replay validation with same facts

这样后面：
- 修了一个 bug
- 调了 validation policy
- 改了 provider adapter

就能拿历史 case 回放，不用全靠猜。

---

## 9. Metrics 指标

建议至少有这些指标：

- workflow success rate
- workflow failure rate
- average run latency
- provider timeout rate
- approval wait time
- validation fail rate
- external write blocked rate
- search freshness violation rate
- source conflict rate

这些指标不是为了好看，而是用来发现系统在哪一层最脆。

---

## 10. UI 怎么帮助调试

易用性和可观测性其实是连在一起的。  
UI 最少要提供：

- run timeline
- trace tree
- source / citation 面板
- validation issue 面板
- approval status 面板
- provider call summary
- replay / retry / resume 入口

如果这些都不可见，用户只能看一个“失败”，那基本等于没设计。

---

## 11. 一句话结论

verification workbench 的可观测性不能停留在“多打点日志”。  
更合理的目标是：

**每一次 workflow / SRWV / skill / plugin 执行，都能通过 trace、timeline、provider log、validation issue、approval trace、failure attribution、replay 机制被完整回放和定位。**

只有做到这点，这个系统才真正可维护、可调试、可进化。
