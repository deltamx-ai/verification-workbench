# Workflow 接口与 Runtime 设计

这份文档补全 workflow definition / scheduling / automation 在运行时中的接口 contract。

目标是回答：
- workflow 在 runtime 里怎么启动
- schedule / event / manual trigger 怎么进来
- workflow run 怎么和 SRWV / skill / plugin runtime 组合
- approval / audit / retry / recovery 怎么挂进去

---

## 1. Runtime 定位

workflow runtime 不应该成为“另一个平行系统”。  
更合理的方式是：

```text
Trigger Layer
  -> Workflow Runtime
    -> Loop Engine
      -> SRWV Executor / Skill Runtime
        -> Plugin Runtime
    -> Approval Runtime
    -> Audit / Event Runtime
```

也就是说：
- Trigger Layer 负责触发
- Workflow Runtime 负责把 workflow definition 变成一次 execution instance
- 真正执行能力仍然复用现有 runtime

---

## 2. 核心 Runtime 模块

### 2.1 Workflow Registry
负责：
- 加载 workflow definition
- 解析当前有效版本
- 读取 trigger 配置
- 校验是否可执行

### 2.2 Trigger Dispatcher
负责：
- 接 schedule 触发
- 接 event 触发
- 接 manual 触发
- 做 dedupe / cooldown / concurrency check

### 2.3 Workflow Executor
负责：
- 创建 workflow run
- 按 definition 生成 step plan
- 调用 SRWV / skill / plugin action
- 汇总状态
- 决定 retry / pause / fail / complete

### 2.4 Workflow Recovery Manager
负责：
- 从 paused / failed / interrupted 恢复
- 识别能否继续
- 找回上次卡住的 step / approval

---

## 3. 核心接口对象

### 3.1 WorkflowTriggerRequest

```ts
export interface WorkflowTriggerRequest {
  workflowDefinitionId: string
  triggerType: "manual" | "schedule" | "event"
  triggerId?: string
  triggerEventId?: string
  reason?: string
  initiatedBy?: string
  payload?: Record<string, unknown>
}
```

### 3.2 WorkflowRunState

```ts
export type WorkflowRunStatus =
  | "queued"
  | "running"
  | "paused"
  | "needs_approval"
  | "completed"
  | "failed"
  | "cancelled"

export interface WorkflowRunState {
  workflowRunId: string
  workflowDefinitionId: string
  workflowVersionId: string
  status: WorkflowRunStatus
  currentStepNo: number
  retryCount: number
  maxAttempts: number
  context: Record<string, unknown>
}
```

### 3.3 WorkflowExecutionContext

```ts
export interface WorkflowExecutionContext {
  workflowRunId: string
  threadId?: string
  taskId?: string
  runId?: string
  triggerType: "manual" | "schedule" | "event"
  triggerPayload?: Record<string, unknown>
  workingMemory: Record<string, unknown>
}
```

---

## 4. Trigger Contract

### 4.1 Schedule Trigger Input

```ts
export interface ScheduleTriggerTick {
  triggerId: string
  now: string
}
```

### 4.2 Event Trigger Input

```ts
export interface EventTriggerInput {
  eventType: string
  source: string
  payload: Record<string, unknown>
  dedupeKey?: string
  occurredAt: string
}
```

### 4.3 Trigger Decision

```ts
export interface TriggerDispatchResult {
  matched: boolean
  workflowDefinitionIds: string[]
  skipReason?: string
}
```

---

## 5. Workflow Definition Contract

```ts
export type WorkflowExecutionPattern = "srwv" | "skill_chain" | "hybrid"

export interface WorkflowDefinition {
  id: string
  name: string
  goal: string
  executionPattern: WorkflowExecutionPattern
  inputSources: string[]
  steps: WorkflowStepDefinition[]
  writeTargets: string[]
  validationPolicy: Record<string, unknown>
  permissionPolicy: Record<string, unknown>
  retryPolicy: {
    maxAttempts: number
    backoffSeconds?: number
  }
}

export interface WorkflowStepDefinition {
  stepNo: number
  kind: "search" | "read" | "write" | "validate" | "decision" | "skill" | "plugin_action"
  name: string
  config?: Record<string, unknown>
}
```

---

## 6. Workflow Executor Contract

```ts
export interface WorkflowExecutor {
  start(request: WorkflowTriggerRequest): Promise<WorkflowRunState>
  runNextStep(state: WorkflowRunState): Promise<WorkflowStepExecutionResult>
  resume(workflowRunId: string): Promise<WorkflowRunState>
  cancel(workflowRunId: string, reason?: string): Promise<void>
}
```

### 6.1 Step Execution Result

```ts
export interface WorkflowStepExecutionResult {
  workflowRunId: string
  stepNo: number
  status: "completed" | "failed" | "blocked" | "needs_approval"
  output?: Record<string, unknown>
  artifacts?: ArtifactDescriptor[]
  nextAction?: "continue" | "retry" | "pause" | "complete" | "fail"
  error?: RuntimeErrorEnvelope
}
```

---

## 7. 与 SRWV / Skill 的衔接

workflow runtime 不应该重新发明 search/read/write/validate 逻辑。  
而应该在 step 层直接调用现有 runtime：

### 7.1 SRWV Step

```ts
export interface SRWVStepAdapter {
  execute(request: SRWVLoopRequest): Promise<SRWVLoopState>
}
```

### 7.2 Skill Step

```ts
export interface SkillStepAdapter {
  execute(skillName: string, input: Record<string, unknown>): Promise<SkillExecutionResult>
}
```

### 7.3 Plugin Action Step

```ts
export interface PluginActionStepAdapter {
  execute(action: string, input: Record<string, unknown>): Promise<ActionExecutionResult>
}
```

所以 workflow 更像 orchestration shell，而不是底层执行器。

---

## 8. Approval Contract

```ts
export interface WorkflowApprovalRequest {
  workflowRunId: string
  stepNo: number
  targetType: "write_external" | "skill" | "plugin_action"
  targetSummary: string
  reason: string
}

export interface WorkflowApprovalDecision {
  approvalRequestId: string
  decision: "approved" | "rejected" | "expired"
  decidedBy?: string
  decidedAt?: string
  comment?: string
}
```

workflow runtime 遇到高风险动作时，必须切到：
- `needs_approval`
- 等 decision
- 再决定 resume / abort

---

## 9. Retry / Recovery Contract

```ts
export interface RetryPolicy {
  maxAttempts: number
  backoffSeconds?: number
  retryableErrorCodes?: string[]
}

export interface RecoveryPoint {
  workflowRunId: string
  stepNo: number
  recoveryState: Record<string, unknown>
  createdAt: string
}
```

建议：
- search/read/local write 允许自动重试
- external write 只在确认幂等时允许自动重试
- approval timeout 默认不自动重试

---

## 10. 错误与审计接口

```ts
export interface RuntimeErrorEnvelope {
  code: string
  category: "validation" | "permission" | "approval" | "provider" | "timeout" | "conflict" | "internal"
  message: string
  retryable: boolean
  details?: Record<string, unknown>
}

export interface WorkflowAuditEvent {
  workflowRunId: string
  stepNo?: number
  eventType: string
  payload: Record<string, unknown>
  createdAt: string
}
```

---

## 11. 一句话结论

workflow runtime 最稳的设计方式是：

**trigger 负责触发，workflow runtime 负责把 definition 变成 execution instance，而真正的 search/read/write/validate、skill、plugin action 仍复用已有 runtime。**

这样手动执行、定时执行、事件执行走的是同一条主干，只是入口不同。
