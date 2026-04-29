# Plan Runtime 实现骨架（TypeScript 导向）

> 目标：在 `plan-runtime-design.md` 的基础上，再往前走一步，给出可以直接转成代码的模块切分、接口、执行流程与伪代码骨架。

## 1. 实现目标

这一层不是再讲概念，而是回答几个工程问题：

1. 代码要分成哪些模块
2. 每个模块的输入输出是什么
3. 执行循环怎么串起来
4. replan 在哪里触发
5. UI / runtime / tool executor 怎么对接

---

## 2. 推荐目录结构

如果后面要把这套 runtime 真做出来，建议目录大概这样分：

```text
src/
├── plan-runtime/
│   ├── types.ts
│   ├── intent-parser.ts
│   ├── tool-matcher.ts
│   ├── strategy-registry.ts
│   ├── plan-generator.ts
│   ├── step-selector.ts
│   ├── step-executor.ts
│   ├── result-normalizer.ts
│   ├── state-reducer.ts
│   ├── goal-evaluator.ts
│   ├── replanner.ts
│   ├── approval-gate.ts
│   ├── runtime.ts
│   └── index.ts
│
├── tools/
│   ├── registry.ts
│   ├── capabilities.ts
│   └── adapters/
│       ├── grep.ts
│       ├── read.ts
│       ├── edit.ts
│       ├── bash.ts
│       └── task-agent.ts
│
├── ui/
│   ├── plan-view-model.ts
│   └── runtime-status-mapper.ts
│
└── shared/
    ├── ids.ts
    ├── logger.ts
    └── errors.ts
```

这套分法的核心思路是：
- **plan-runtime/** 只负责“怎么想、怎么编排”
- **tools/** 只负责“怎么执行具体工具”
- **ui/** 只负责“怎么展示 runtime 状态”

不要把工具调用细节塞进 planner，也别把 UI 状态判断写进 executor。

---

## 3. 最小类型定义

## 3.1 runtime 输入

```ts
export interface UserRequest {
  id: string;
  threadId: string;
  message: string;
  workspace?: string;
  repoPath?: string;
  attachments?: Array<{
    name: string;
    path: string;
    mimeType?: string;
  }>;
}
```

## 3.2 工具能力描述

```ts
export interface ToolCapability {
  name: string;
  category:
    | "search"
    | "read"
    | "write"
    | "execute"
    | "delegate"
    | "network"
    | "visual";
  riskLevel: "low" | "medium" | "high";
  requiresApproval: boolean;
  supportedTaskTypes: string[];
  inputSchema?: Record<string, unknown>;
}
```

## 3.3 plan step

```ts
export interface PlanStep {
  id: string;
  title: string;
  kind:
    | "search"
    | "inspect"
    | "modify"
    | "verify"
    | "summarize"
    | "delegate"
    | "approval";
  tool?: string;
  args?: Record<string, unknown>;
  reason: string;
  dependsOn: string[];
  status: "pending" | "ready" | "running" | "succeeded" | "failed" | "blocked" | "skipped";
  successCheck: string;
  failurePolicy:
    | "retry"
    | "replan"
    | "abort"
    | "fallback_to_agent"
    | "wait_for_approval";
  outputs: string[];
}
```

## 3.4 runtime state

```ts
export interface PlanRuntimeState {
  taskId: string;
  threadId: string;
  status:
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
  iteration: number;
  maxIterations: number;
  request: UserRequest;
  intent?: ParsedIntent;
  toolMatch?: ToolMatchResult;
  plan: PlanStep[];
  artifacts: Record<string, unknown>;
  logs: RuntimeLog[];
  lastError?: string;
}

export interface RuntimeLog {
  at: string;
  level: "info" | "warn" | "error";
  message: string;
  stepId?: string;
  data?: unknown;
}
```

---

## 4. 核心模块接口

## 4.1 Intent Parser

```ts
export interface IntentParser {
  parse(request: UserRequest): Promise<ParsedIntent>;
}
```

职责：
- 从用户输入抽出目标、约束、产物、风险
- 尽量不要在这一层做工具选择

---

## 4.2 Tool Matcher

```ts
export interface ToolMatcher {
  match(intent: ParsedIntent, context: MatchContext): Promise<ToolMatchResult>;
}

export interface MatchContext {
  availableTools: ToolCapability[];
  availableSkills: ToolCapability[];
  availableAgents: ToolCapability[];
  workspace?: string;
  repoPath?: string;
}
```

职责：
- 根据 taskType、风险和当前环境筛出候选工具
- 给 planner 一个受约束候选集

---

## 4.3 Strategy Registry

```ts
export interface PlanningStrategy {
  name: string;
  supports(intent: ParsedIntent): boolean;
  buildInitialPlan(input: {
    intent: ParsedIntent;
    toolMatch: ToolMatchResult;
    state: PlanRuntimeState;
  }): Promise<PlanStep[]>;

  replan(input: {
    state: PlanRuntimeState;
    failedStep?: PlanStep;
    reason?: string;
  }): Promise<PlanStep[]>;
}
```

职责：
- 给不同任务类型提供不同 plan 模板
- search / code_change / workflow_build 分别有自己的策略

---

## 4.4 Step Executor

```ts
export interface StepExecutor {
  execute(step: PlanStep, state: PlanRuntimeState): Promise<StepExecutionResult>;
}

export interface StepExecutionResult {
  stepId: string;
  ok: boolean;
  summary: string;
  rawResult?: unknown;
  derivedArtifacts?: Record<string, unknown>;
  error?: string;
  retryable?: boolean;
  shouldReplan?: boolean;
}
```

职责：
- 执行单步
- 调用具体工具 adapter
- 标准化所有工具返回结果

---

## 4.5 State Reducer

```ts
export interface StateReducer {
  applyStepResult(
    state: PlanRuntimeState,
    step: PlanStep,
    result: StepExecutionResult,
  ): PlanRuntimeState;
}
```

职责：
- 更新 step 状态
- 写入 artifacts
- 记录日志
- 切换 runtime status

---

## 4.6 Goal Evaluator

```ts
export interface GoalEvaluator {
  isSatisfied(state: PlanRuntimeState): boolean;
}
```

职责：
- 判断任务是不是已经完成
- 不要只靠“最后一步跑完了”，要结合产物和验证结果

---

## 5. 运行主循环骨架

这里建议有一个 `PlanRuntime` 类来统一调度。

```ts
export class PlanRuntime {
  constructor(
    private intentParser: IntentParser,
    private toolMatcher: ToolMatcher,
    private strategyRegistry: StrategyRegistry,
    private stepExecutor: StepExecutor,
    private stateReducer: StateReducer,
    private goalEvaluator: GoalEvaluator,
    private approvalGate: ApprovalGate,
  ) {}

  async run(request: UserRequest): Promise<PlanRuntimeState> {
    let state = this.createInitialState(request);

    state = await this.parseIntent(state);
    state = await this.matchTools(state);
    state = await this.buildInitialPlan(state);
    state = await this.executeLoop(state);

    return state;
  }
}
```

---

## 6. 每个阶段怎么写

## 6.1 parseIntent

```ts
private async parseIntent(state: PlanRuntimeState): Promise<PlanRuntimeState> {
  const intent = await this.intentParser.parse(state.request);

  return {
    ...state,
    intent,
    status: "intent_parsed",
    logs: [
      ...state.logs,
      logInfo("Intent parsed", { intent }),
    ],
  };
}
```

## 6.2 matchTools

```ts
private async matchTools(state: PlanRuntimeState): Promise<PlanRuntimeState> {
  const toolMatch = await this.toolMatcher.match(state.intent!, {
    availableTools: getBuiltinToolCapabilities(),
    availableSkills: getInstalledSkillCapabilities(),
    availableAgents: getManagedAgentCapabilities(),
    workspace: state.request.workspace,
    repoPath: state.request.repoPath,
  });

  return {
    ...state,
    toolMatch,
    status: toolMatch.needsApproval ? "waiting_approval" : "tools_matched",
    logs: [
      ...state.logs,
      logInfo("Tools matched", { toolMatch }),
    ],
  };
}
```

## 6.3 buildInitialPlan

```ts
private async buildInitialPlan(state: PlanRuntimeState): Promise<PlanRuntimeState> {
  const strategy = this.strategyRegistry.pick(state.intent!);
  const plan = await strategy.buildInitialPlan({
    intent: state.intent!,
    toolMatch: state.toolMatch!,
    state,
  });

  return {
    ...state,
    plan: hydrateReadySteps(plan),
    status: "plan_ready",
    logs: [
      ...state.logs,
      logInfo("Initial plan built", { planSize: plan.length, strategy: strategy.name }),
    ],
  };
}
```

---

## 7. 执行循环

## 7.1 主循环

```ts
private async executeLoop(state: PlanRuntimeState): Promise<PlanRuntimeState> {
  let current = {
    ...state,
    status: "executing",
  };

  while (current.iteration < current.maxIterations) {
    if (this.goalEvaluator.isSatisfied(current)) {
      return {
        ...current,
        status: "completed",
      };
    }

    const nextStep = selectNextExecutableStep(current.plan);

    if (!nextStep) {
      return {
        ...current,
        status: hasFailedStep(current.plan) ? "failed" : "blocked",
        lastError: "No executable step available",
      };
    }

    if (needsApproval(nextStep)) {
      const approved = await this.approvalGate.request(nextStep, current);
      if (!approved) {
        return {
          ...current,
          status: "waiting_approval",
        };
      }
    }

    const result = await this.stepExecutor.execute(nextStep, current);
    current = this.stateReducer.applyStepResult(current, nextStep, result);
    current = incrementIteration(current);

    if (result.shouldReplan) {
      current = await this.replan(current, nextStep, result.error ?? result.summary);
    }
  }

  return {
    ...current,
    status: "failed",
    lastError: "Max iterations reached",
  };
}
```

---

## 8. replan 触发点

以下情况建议 `shouldReplan = true`：

1. 搜索结果为空
2. 读到的文件和预期不一致
3. edit 失败（替换不到）
4. bash 验证失败但可恢复
5. 当前路径变得不合适，需要升级到 agent

### replan 骨架

```ts
private async replan(
  state: PlanRuntimeState,
  failedStep: PlanStep,
  reason: string,
): Promise<PlanRuntimeState> {
  const strategy = this.strategyRegistry.pick(state.intent!);
  const newPlan = await strategy.replan({
    state: {
      ...state,
      status: "replanning",
    },
    failedStep,
    reason,
  });

  return {
    ...state,
    status: "executing",
    plan: mergePlan(state.plan, newPlan),
    logs: [
      ...state.logs,
      logWarn("Plan replanned", {
        failedStepId: failedStep.id,
        reason,
      }),
    ],
  };
}
```

重点不是把旧 plan 整个推翻，而是：
- 保留成功步骤
- 标记失败步骤
- 给后续补新路

---

## 9. Tool Adapter 层建议

`step-executor` 不应该直接知道所有工具细节。

应该是：

```ts
export interface ToolAdapter {
  supports(toolName: string): boolean;
  execute(args: Record<string, unknown>, state: PlanRuntimeState): Promise<unknown>;
}
```

例如：
- `grep.ts`
- `read.ts`
- `edit.ts`
- `bash.ts`
- `task-agent.ts`

这样后面你加 skill、browser、web search 都不用改 planner，只扩展 adapter registry。

---

## 10. 结果标准化

工具返回五花八门，所以需要 `result-normalizer`。

### 统一成：

```ts
export interface NormalizedToolResult {
  ok: boolean;
  summary: string;
  data?: unknown;
  artifacts?: Record<string, unknown>;
  error?: string;
  signals?: {
    emptyResult?: boolean;
    permissionDenied?: boolean;
    partialSuccess?: boolean;
  };
}
```

例如：
- `Grep` 没搜到 -> `ok: false`, `signals.emptyResult = true`
- `Edit` 改到了 -> `ok: true`, `artifacts.patch = ...`
- `Bash` 退出码非 0 -> `ok: false`, `error = stderr`

这样上层状态机不需要理解每个工具的原始格式。

---

## 11. UI 怎么接

UI 不应该吃原始工具返回，而应该吃 runtime view model。

### 建议单独映射

```ts
export interface RuntimeViewModel {
  title: string;
  status: string;
  currentStepTitle?: string;
  completedCount: number;
  totalCount: number;
  timeline: Array<{
    label: string;
    status: string;
    summary?: string;
  }>;
  blocker?: string;
}
```

这样右侧详情页可以稳定展示：
- 当前计划
- 当前执行步骤
- 已完成步骤
- 为什么重规划
- 当前阻塞点

---

## 12. MVP 真实落地顺序

最省力的顺序我建议这样：

### Phase 1
- `types.ts`
- `intent-parser.ts`
- `tool-matcher.ts`
- `strategy-registry.ts`
- `plan-generator.ts`

先把 **能产出 plan** 做出来。

### Phase 2
- `step-executor.ts`
- `tools/adapters/read.ts`
- `tools/adapters/grep.ts`
- `tools/adapters/edit.ts`
- `tools/adapters/bash.ts`

先支持最基础的 4 个工具。

### Phase 3
- `state-reducer.ts`
- `goal-evaluator.ts`
- `runtime.ts`
- `replanner.ts`

把循环跑起来。

### Phase 4
- 接 UI
- 做 timeline
- 做 step 状态展示
- 做审批态展示

### Phase 5
- 接 agent delegation
- 接 skill matching
- 接 browser / web tools

---

## 13. 一个最小真实例子

输入：

> 修复登录按钮点击没反应，别改后端，改完跑测试

### 预解析

```json
{
  "goal": "修复登录按钮点击没反应",
  "taskType": "code_change",
  "constraints": ["别改后端", "改完跑测试"],
  "artifacts": ["patch", "verification_result"],
  "riskLevel": "medium"
}
```

### 初始 plan

```json
[
  {
    "id": "s1",
    "title": "搜索登录组件",
    "tool": "Grep",
    "status": "ready"
  },
  {
    "id": "s2",
    "title": "阅读候选文件",
    "tool": "Read",
    "status": "pending"
  },
  {
    "id": "s3",
    "title": "修改点击逻辑",
    "tool": "Edit",
    "status": "pending"
  },
  {
    "id": "s4",
    "title": "运行测试",
    "tool": "Bash",
    "status": "pending"
  }
]
```

### 如果测试失败

replan 成：
- 插入“检查 loading state / form submit state”
- 再补一次 edit
- 再跑测试

这就是滚动规划的价值。

---

## 14. 一句话落地原则

真正能跑的 Plan Runtime，不是“模型先写一大段计划”，而是：

**用模块化 runtime 把意图解析、工具约束、结构化步骤、执行观察和重规划串成一个可持续推进的状态机。**
