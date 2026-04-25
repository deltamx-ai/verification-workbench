# Search / Read / Write / Validation 接口与 Runtime 设计

这份文档补充 SRWV（Search / Read / Write / Validation）循环在运行时中的接口结构与模块 contract。

目标是回答：
- SRWV 在 runtime 里怎么执行
- 模块之间怎么传值
- 输入输出结构长什么样
- browser_live_search 这种能力应该挂在哪

---

## 1. 运行时定位

SRWV 不应该被设计成一个独立的“另一个引擎”。  
更合理的方式是把它做成 **Loop Engine 里的一个标准执行模式**。

大概关系：

```text
Thread Runtime
  -> Loop Engine
    -> SRWV Executor
      -> Search Provider
      -> Read Extractor
      -> Write Target Adapter
      -> Validation Policy
```

也就是说：
- Thread Runtime 负责接收目标
- Loop Engine 负责调度
- SRWV Executor 负责把四步跑起来
- 具体搜索、读取、写入、校验由子模块承接

---

## 2. 核心 Runtime 模块

### 2.1 SRWV Executor

它是 SRWV 模式的主控制器。  
负责：
- 创建 loop
- 驱动 iteration
- 调用 search/read/write/validate
- 接收 validation 结果
- 产出 decision
- 更新状态

### 2.2 Search Provider

负责根据当前目标找来源。

建议 provider 化：
- `local_search`
- `knowledge_search`
- `web_search`
- `browser_live_search`

其中 `browser_live_search` 负责：
- 打开页面
- 搜索关键词
- 抓取最新消息
- 返回候选来源与摘要

它仍然只是 Search 阶段的一种 provider，不直接输出最终结论。

### 2.3 Read Extractor

负责把来源转成结构化证据：
- excerpt
- facts
- confidence
- timestamp
- conflicts / uncertainties

### 2.4 Write Target Adapter

负责把产出写到不同目标：
- draft memory
- local objects
- external systems

并区分：
- `draft`
- `local`
- `external`

### 2.5 Validation Policy

负责对一轮结果做结构化校验：
- 证据够不够
- 是否最新
- 是否冲突
- 是否满足 goal
- 是否具备权限
- 是否需要审批

---

## 3. 核心接口对象

### 3.1 SRWVLoopRequest

```ts
export interface SRWVLoopRequest {
  goal: string
  threadId?: string
  taskId?: string
  runId?: string
  freshnessRequired?: boolean
  externalWriteAllowed?: boolean
  maxIterations?: number
  sourcePriority?: Array<"local" | "knowledge" | "web" | "browser_live">
  initialContext?: Record<string, unknown>
}
```

---

### 3.2 SRWVLoopState

```ts
export type SRWVLoopStatus =
  | "draft"
  | "running"
  | "blocked"
  | "needs_approval"
  | "completed"
  | "failed"
  | "cancelled"

export interface SRWVLoopState {
  loopId: string
  status: SRWVLoopStatus
  currentIterationNo: number
  goal: string
  freshnessRequired: boolean
  externalWriteAllowed: boolean
  context: Record<string, unknown>
}
```

---

### 3.3 SRWVIterationContext

```ts
export interface SRWVIterationContext {
  loopId: string
  iterationNo: number
  knownContext: Record<string, unknown>
  missingInfo: string[]
  previousIssues: ValidationIssue[]
  previousDecision?: SRWVDecisionType
}
```

---

## 4. Search Contract

### 4.1 Search Request

```ts
export interface SearchRequest {
  loopId: string
  iterationNo: number
  goal: string
  missingInfo: string[]
  freshnessRequired: boolean
  sourcePriority: Array<"local" | "knowledge" | "web" | "browser_live">
  context: Record<string, unknown>
}
```

### 4.2 Search Result

```ts
export interface SearchCandidate {
  provider: "local_search" | "knowledge_search" | "web_search" | "browser_live_search"
  sourceType: string
  sourceId: string
  title?: string
  snippet?: string
  url?: string
  timestamp?: string
  confidence?: number
  whySelected?: string
  rankScore?: number
}

export interface SearchResponse {
  candidates: SearchCandidate[]
}
```

### 4.3 BrowserLiveSearchProvider

```ts
export interface BrowserLiveSearchProvider {
  search(request: SearchRequest): Promise<SearchResponse>
}
```

这个 provider 背后可以调用浏览器自动化能力，但在 runtime contract 上它仍然只是 SearchProvider。

---

## 5. Read Contract

### 5.1 Read Request

```ts
export interface ReadRequest {
  loopId: string
  iterationNo: number
  candidates: SearchCandidate[]
  goal: string
}
```

### 5.2 Read Document / Fact

```ts
export interface ReadFact {
  factText: string
  factType: "status" | "event" | "timestamp" | "owner" | "risk" | "actionable" | "unknown"
  confidence?: number
  normalizedKey?: string
  normalizedValue?: string
  isLatest?: boolean
  conflictsWith?: string
}

export interface ReadDocument {
  sourceId: string
  excerpt?: string
  timestamp?: string
  confidence?: number
  uncertainties?: string[]
  facts: ReadFact[]
}

export interface ReadResponse {
  documents: ReadDocument[]
}
```

---

## 6. Write Contract

### 6.1 Write Type

```ts
export type WriteType = "draft" | "local" | "external"
```

### 6.2 Write Request

```ts
export interface WriteRequest {
  loopId: string
  iterationNo: number
  writeType: WriteType
  targetType: string
  targetId?: string
  content: string
  citations?: string[]
  metadata?: Record<string, unknown>
}
```

### 6.3 Write Response

```ts
export interface WriteResponse {
  status: "prepared" | "pending_approval" | "written" | "skipped" | "failed"
  targetType: string
  targetId?: string
  approvalRequired?: boolean
  approvalRequestId?: string
}
```

### 6.4 Write Adapter

```ts
export interface WriteTargetAdapter {
  write(request: WriteRequest): Promise<WriteResponse>
}
```

---

## 7. Validation Contract

### 7.1 Validation Request

```ts
export interface ValidationRequest {
  loopId: string
  iterationNo: number
  goal: string
  freshnessRequired: boolean
  writeType: WriteType
  readDocuments: ReadDocument[]
  writeResult?: WriteResponse
}
```

### 7.2 Validation Issue / Result

```ts
export interface ValidationIssue {
  issueType:
    | "missing_source"
    | "stale_information"
    | "source_conflict"
    | "missing_citation"
    | "goal_not_met"
    | "permission_denied"
    | "approval_missing"
  message: string
  severity: "low" | "medium" | "high" | "critical"
  suggestedNextAction?: SRWVDecisionType
}

export interface ValidationResult {
  pass: boolean
  evidenceSufficiencyScore?: number
  freshnessOk?: boolean
  conflictDetected?: boolean
  citationsComplete?: boolean
  goalCoverageOk?: boolean
  permissionOk?: boolean
  approvalOk?: boolean
  issues: ValidationIssue[]
  nextAction?: SRWVDecisionType
  summary?: string
}
```

---

## 8. Decision Contract

```ts
export type SRWVDecisionType =
  | "continue"
  | "retry_current_step"
  | "go_to_search"
  | "go_to_read"
  | "needs_approval"
  | "handoff_to_human"
  | "stop"
  | "fail"
```

```ts
export interface DecisionResult {
  decision: SRWVDecisionType
  reason: string
  nextState:
    | "searching"
    | "reading"
    | "writing"
    | "validating"
    | "needs_approval"
    | "completed"
    | "failed"
    | "blocked"
}
```

---

## 9. SRWV Executor 主流程

```ts
export interface SRWVExecutor {
  start(request: SRWVLoopRequest): Promise<SRWVLoopState>
  resume(loopId: string): Promise<SRWVLoopState>
  cancel(loopId: string): Promise<void>
}
```

内部流程可以理解为：

```text
create loop
-> planning
-> search
-> read
-> write
-> validate
-> decide
-> continue or stop
```

但实现时最好按 iteration 执行，每轮落库：
- iteration state
- search results
- read documents
- write operation
- validation result
- decision

这样中途崩了也能恢复。

---

## 10. 状态机建议

```text
idle
-> planning
-> searching
-> reading
-> writing
-> validating
-> completed

异常分支：
-> blocked
-> needs_approval
-> needs_more_context
-> failed
```

### 状态迁移规则
- `planning -> searching`：存在缺失信息
- `planning -> reading`：已有足够候选来源
- `searching -> reading`：search 返回候选来源
- `reading -> writing`：已有足够结构化事实
- `writing -> validating`：write 完成或生成草稿
- `validating -> searching`：缺少来源 / 信息过时
- `validating -> reading`：来源存在但抽取不够
- `validating -> needs_approval`：外部写待审批
- `validating -> completed`：校验通过且目标满足
- `validating -> failed`：达到失败条件或无法恢复

---

## 11. Browser Live Search 怎么接进来

你关心的点其实就在这里。

设计上不要把“浏览器搜索最新消息”写死在 loop 里，  
而是把它作为一个 Search Provider：

```text
SearchProvider
  ├── LocalSearchProvider
  ├── KnowledgeSearchProvider
  ├── WebSearchProvider
  └── BrowserLiveSearchProvider
```

这样：
- Search 阶段可以按 sourcePriority 选择 provider
- browser_live_search 只负责发现最新来源
- read / validate 仍然走统一后处理链路

也就是：
**浏览器能力只是 search 的一种实现，不应该变成独立流程。**

---

## 12. 与主系统的集成点

SRWV Executor 应该和这些 runtime 协同：

### 12.1 Thread Runtime
- 从 thread 触发 loop
- 把结果回写 thread / summary

### 12.2 Task Runtime
- SRWV 作为某个 run / step 的执行模式
- 产出的 artifact / verdict 回流到 task/run

### 12.3 Approval Runtime
- `write_external` 时创建 approval request
- 批准后 resume loop

### 12.4 Audit Runtime
- 记录 provider 调用
- 记录 decision 和状态迁移
- 记录失败原因与耗时

---

## 13. 一句话结论

SRWV 的 runtime 设计重点不是“把四步串起来”这么简单，  
而是要把它做成一个**可插 provider、可落库回放、可审批中断、可恢复继续**的标准执行模式。

而“开浏览器搜最新消息”就应该挂在 Search Provider 层，作为 `BrowserLiveSearchProvider` 存在，  
它负责发现最新来源，但仍然服从统一的 Read / Write / Validation contract。
