# Plan Runtime 设计：预解析、工具选择、调用计划与执行循环

## 目标

实现一个可落地的 Plan Runtime，让系统在收到用户请求后，不是直接自由调用工具，而是先完成：

1. 预解析用户输入
2. 识别任务目标、约束、风险与产物
3. 匹配当前可用工具/技能/代理
4. 生成结构化执行计划
5. 进入 plan → act → observe → replan 的循环
6. 在成功、失败、阻塞、审批等待之间安全切换

这个 runtime 的核心目标不是“写一段好看的计划文本”，而是产出一个**可执行、可观测、可回滚、可重规划**的任务执行过程。

---

## 一、总体流程

```text
user input
  ↓
intent pre-parse
  ↓
task classification
  ↓
tool / skill / agent eligibility matching
  ↓
structured plan generation
  ↓
execution loop
  ├─ act
  ├─ observe
  ├─ validate
  ├─ update state
  └─ replan if needed
  ↓
completion / blocked / failed
```

---

## 二、阶段拆分

### 1. Intent Pre-Parse（预解析）

先把用户自然语言抽成结构化意图对象，而不是立刻规划。

需要抽取：
- `goal`: 用户最终目标
- `taskType`: 任务类型
- `constraints`: 明确限制
- `artifacts`: 预期产物
- `scope`: 文件/仓库/环境范围
- `riskLevel`: 风险等级
- `requiresApproval`: 是否可能需要审批
- `ambiguities`: 目前仍不明确的点

### 任务类型建议

- `qa`
- `search`
- `summarize`
- `code_change`
- `config_change`
- `file_operation`
- `workflow_build`
- `research`
- `multi_step_execution`

### 示例输出

```json
{
  "goal": "修复登录按钮点击无效",
  "taskType": "code_change",
  "constraints": ["不要修改后端 API", "优先最小改动"],
  "artifacts": ["patch", "verification_result"],
  "scope": {
    "repo": "current",
    "paths": ["src/"]
  },
  "riskLevel": "medium",
  "requiresApproval": false,
  "ambiguities": []
}
```

---

### 2. Tool Eligibility Matching（工具可用性匹配）

这一步的目的，是为 planner 提供一个**受约束工具集合**，而不是让 planner 任意幻想工具。

要判断：
- 哪些原子工具适用
- 哪些 skill 可直接覆盖
- 是否更适合委托 agent
- 是否存在权限/审批边界
- 是否需要先读环境信息

### 输出建议

```json
{
  "candidateTools": [
    {
      "name": "Grep",
      "reason": "定位登录逻辑实现位置",
      "confidence": 0.95
    },
    {
      "name": "Read",
      "reason": "检查组件代码",
      "confidence": 0.93
    },
    {
      "name": "Edit",
      "reason": "修复事件绑定",
      "confidence": 0.88
    },
    {
      "name": "Bash",
      "reason": "运行测试或构建验证",
      "confidence": 0.81
    }
  ],
  "candidateSkills": [],
  "candidateAgents": [
    {
      "name": "developer",
      "reason": "适合多文件代码修复与验证"
    }
  ],
  "delegationMode": "self",
  "needsApproval": false,
  "blockedBy": []
}
```

### 一个关键原则

planner 不应该凭空决定“我要用 Bash 干所有事”。

而应该先经过 runtime 过滤：
- 纯搜索优先 `Glob/Grep/Read`
- 精准改动优先 `Edit`
- 大规模实现优先 `Task(agent_id=developer)`
- 复杂产品构建优先走多阶段委托

---

### 3. Structured Plan Generation（结构化计划生成）

plan 不应该只是 Markdown 文本，而应是可执行步骤数组。

每一步建议包含：
- `id`
- `title`
- `kind`
- `tool`
- `args`
- `reason`
- `dependsOn`
- `successCheck`
- `failurePolicy`
- `outputs`
- `status`

### 示例

```json
[
  {
    "id": "step1",
    "title": "定位登录按钮相关代码",
    "kind": "search",
    "tool": "Grep",
    "args": {
      "pattern": "login|sign in|onClick",
      "path": "src"
    },
    "reason": "先找出登录按钮组件与点击逻辑",
    "dependsOn": [],
    "successCheck": "找到候选文件列表",
    "failurePolicy": "retry_with_broader_pattern",
    "outputs": ["candidate_files"],
    "status": "pending"
  },
  {
    "id": "step2",
    "title": "阅读候选组件",
    "kind": "inspect",
    "tool": "Read",
    "args": {
      "file_path": "src/components/Login.tsx"
    },
    "reason": "确认点击事件绑定与表单提交逻辑",
    "dependsOn": ["step1"],
    "successCheck": "明确 bug 所在位置",
    "failurePolicy": "select_alternate_file",
    "outputs": ["bug_hypothesis"],
    "status": "pending"
  },
  {
    "id": "step3",
    "title": "修复点击逻辑",
    "kind": "modify",
    "tool": "Edit",
    "args": {},
    "reason": "修复按钮 onClick 或 submit handler 绑定",
    "dependsOn": ["step2"],
    "successCheck": "代码改动符合假设",
    "failurePolicy": "fallback_to_developer_agent",
    "outputs": ["patch"],
    "status": "pending"
  },
  {
    "id": "step4",
    "title": "验证修复结果",
    "kind": "verify",
    "tool": "Bash",
    "args": {
      "command": "npm test"
    },
    "reason": "确保修复没有破坏现有逻辑",
    "dependsOn": ["step3"],
    "successCheck": "测试通过或至少相关测试通过",
    "failurePolicy": "replan",
    "outputs": ["verification_result"],
    "status": "pending"
  }
]
```

---

## 三、执行循环设计

### 核心循环

runtime 不是一次性执行全部 plan，而是逐步推进。

```text
select next executable step
  ↓
run tool / skill / agent
  ↓
capture result
  ↓
validate success / failure / partial success
  ↓
update execution state
  ↓
replan if needed
  ↓
continue or finish
```

### 推荐循环伪代码

```ts
while (!state.done && state.iteration < state.maxIterations) {
  const nextStep = getNextExecutableStep(state.plan, state);

  if (!nextStep) {
    state = markBlocked(state, "No executable step available");
    break;
  }

  const executionResult = await runStep(nextStep, state);
  state = applyStepResult(state, nextStep, executionResult);

  if (goalSatisfied(state)) {
    state = markCompleted(state);
    break;
  }

  if (shouldReplan(state, nextStep, executionResult)) {
    state = await replan(state);
  }
}
```

---

## 四、状态机设计

建议把整个过程做成明确状态机，而不是散落在 if/else 里。

### Runtime 状态

- `received`
- `intent_parsed`
- `tools_matched`
- `plan_ready`
- `executing`
- `waiting_approval`
- `blocked`
- `replanning`
- `completed`
- `failed`
- `cancelled`

### Step 状态

- `pending`
- `ready`
- `running`
- `succeeded`
- `failed`
- `blocked`
- `skipped`

### 状态迁移示意

```text
received
  → intent_parsed
  → tools_matched
  → plan_ready
  → executing
      ↘ waiting_approval
      ↘ blocked
      ↘ replanning
  → completed / failed / cancelled
```

---

## 五、Replan 机制

真正好用的 plan runtime，强在失败后还能继续，而不是第一下失败就结束。

### 触发 replan 的典型场景

1. 搜不到目标文件
2. 读完后发现假设错误
3. 编辑失败或替换不匹配
4. 测试失败，说明修复不完整
5. 权限不足
6. 新信息暴露出更优路径

### Replan 要做什么

- 保留已完成步骤
- 标记失败步骤原因
- 基于最新上下文重新生成后续步骤
- 不重复做已确认完成的工作
- 如果风险提升，切换到审批或委托模式

### replan 输入

```json
{
  "currentGoal": "修复登录按钮点击无效",
  "completedSteps": ["step1", "step2"],
  "failedStep": "step4",
  "failureReason": "相关测试失败，出现表单状态回归",
  "currentContext": {
    "bugHypothesis": "点击绑定修好了，但 loading state 没有恢复"
  }
}
```

---

## 六、策略层（Strategy Templates）

为避免 planner 每次从零推理，建议内置策略模板。

### 1. Search Strategy

流程：
- narrow search
- inspect hits
- summarize findings

### 2. Code Fix Strategy

流程：
- locate code
- inspect behavior
- propose fix
- apply patch
- verify
- summarize

### 3. Build Strategy

流程：
- parse requirements
- split milestones
- create implementation slices
- build incrementally
- evaluate each slice

### 4. Ops / Config Strategy

流程：
- inspect current state
- calculate minimal change
- apply config update
- verify effect
- rollback if needed

### 策略选择逻辑

```ts
switch (intent.taskType) {
  case "search":
    return searchStrategy;
  case "code_change":
    return codeFixStrategy;
  case "workflow_build":
    return buildStrategy;
  case "config_change":
    return opsStrategy;
  default:
    return genericReasoningStrategy;
}
```

---

## 七、数据结构建议（TypeScript）

```ts
export type TaskType =
  | "qa"
  | "search"
  | "summarize"
  | "code_change"
  | "config_change"
  | "file_operation"
  | "workflow_build"
  | "research"
  | "multi_step_execution";

export type RiskLevel = "low" | "medium" | "high";

export type RuntimeStatus =
  | "received"
  | "intent_parsed"
  | "tools_matched"
  | "plan_ready"
  | "executing"
  | "waiting_approval"
  | "blocked"
  | "replanning"
  | "completed"
  | "failed"
  | "cancelled";

export type StepStatus =
  | "pending"
  | "ready"
  | "running"
  | "succeeded"
  | "failed"
  | "blocked"
  | "skipped";

export interface ParsedIntent {
  goal: string;
  taskType: TaskType;
  constraints: string[];
  artifacts: string[];
  scope?: {
    repo?: string;
    paths?: string[];
  };
  riskLevel: RiskLevel;
  requiresApproval: boolean;
  ambiguities: string[];
}

export interface CandidateTool {
  name: string;
  reason: string;
  confidence: number;
}

export interface ToolMatchResult {
  candidateTools: CandidateTool[];
  candidateSkills: CandidateTool[];
  candidateAgents: CandidateTool[];
  delegationMode: "self" | "skill" | "agent" | "hybrid";
  needsApproval: boolean;
  blockedBy: string[];
}

export interface PlanStep {
  id: string;
  title: string;
  kind: string;
  tool?: string;
  args?: Record<string, unknown>;
  reason: string;
  dependsOn: string[];
  successCheck: string;
  failurePolicy:
    | "retry"
    | "retry_with_broader_pattern"
    | "select_alternate_file"
    | "replan"
    | "fallback_to_agent"
    | "abort";
  outputs: string[];
  status: StepStatus;
}

export interface StepExecutionResult {
  stepId: string;
  ok: boolean;
  summary: string;
  rawResult?: unknown;
  derivedArtifacts?: Record<string, unknown>;
  error?: string;
}

export interface PlanRuntimeState {
  taskId: string;
  status: RuntimeStatus;
  iteration: number;
  maxIterations: number;
  intent?: ParsedIntent;
  toolMatch?: ToolMatchResult;
  plan: PlanStep[];
  completedStepIds: string[];
  failedStepIds: string[];
  artifacts: Record<string, unknown>;
  logs: string[];
}
```

---

## 八、模块拆分建议

### 1. `intent-parser`
职责：
- 把用户输入转成 `ParsedIntent`

### 2. `tool-matcher`
职责：
- 根据 intent 和上下文产出候选工具集合

### 3. `plan-generator`
职责：
- 用策略模板生成 `PlanStep[]`

### 4. `step-executor`
职责：
- 执行单步工具调用
- 标准化结果

### 5. `state-reducer`
职责：
- 将 step result 合并进 runtime state

### 6. `replanner`
职责：
- 基于失败上下文补后续步骤或改写路径

### 7. `goal-evaluator`
职责：
- 判断任务是否已经完成

---

## 九、落地原则

### 原则 1：工具优先级要硬编码约束，不要全靠模型自由发挥
比如：
- 搜索文件优先 `Glob/Grep`
- 看文件优先 `Read`
- 小改动优先 `Edit`
- 复杂编码优先 agent

### 原则 2：plan 必须可观测
每一步都要能展示：
- 为什么做
- 用了什么工具
- 成功没
- 失败在哪
- 下一步是什么

### 原则 3：失败不是终点
失败只意味着：
- 要么扩大搜索范围
- 要么换工具
- 要么升级委托
- 要么等审批

### 原则 4：plan 与 UI 对齐
UI 应能直接渲染：
- 当前状态
- 当前步骤
- 已完成步骤
- 阻塞原因
- replan 次数
- 最终结果摘要

---

## 十、最小可行版本（MVP）

第一版不用一步到位做太重，建议最小实现：

1. `intent-parser`
2. `tool-matcher`
3. `plan-generator`
4. `step-executor`
5. `replanner`
6. 简单状态机

先支持两类任务：
- `search`
- `code_change`

先支持四个工具：
- `Grep`
- `Read`
- `Edit`
- `Bash`

这样就已经能覆盖大量真实任务。

---

## 十一、一句话总结

Plan 的本质不是“先写一段计划”，而是：

**把用户请求转成受约束的执行状态机，用结构化步骤驱动工具调用，并在每次观察结果后持续重规划。**
