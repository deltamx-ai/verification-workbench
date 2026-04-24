# 更早期设计脉络纪要（仅保留 Alma 的设计输出）

这份文档收录在 verification workbench 明确成型之前，那些对后续架构有直接影响的早期设计思路，包括：
- AGT 编排思路
- Plan / Execute / Evaluate / Replan 闭环
- 状态机驱动
- Code Review 规划与上下文组织
- 风险拆解

这些内容虽然比最终“验证层桌面应用”更早，但其实提供了很多底层设计前提。

---

## 1. 编排是不是要自己写

结论是：
**要，但先写薄。**

不要一开始做重型 DAG 平台，而是选：
**轻编排 + 强状态机 + 可重规划**

推荐三层结构：
- Planner：只负责拆步骤
- Orchestrator：只负责调度、状态、重试、replan
- Worker / Agent：只负责执行单步

关键原因是：
AGT 这类系统最重要的不是普通任务调度，而是：
- 什么时候重新规划
- 什么时候补上下文
- 什么时候换模型
- 什么时候人工接管
- 什么时候并行，什么时候串行

这都是 agent-native 逻辑。

---

## 2. 核心闭环

最核心的闭环不是静态流程，而是：

```text
Plan -> Execute -> Evaluate -> Replan
```

它代表的不是普通 workflow，而是一个执行后仍可反思和修正计划的认知型编排闭环。

---

## 3. 为什么状态机优先

建议先做状态机，不先做复杂 DAG。

因为真实场景里更重要的是：
- 任务是否完成
- 是否失败
- 是否缺上下文
- 是否需要人工介入
- 是否需要重新规划

这比复杂图调度更关键。

---

## 4. 状态机怎么设计

建议分两层：

### 4.1 Flow 级状态
整个任务级状态：
- `draft`
- `planned`
- `executing`
- `paused`
- `needs_replan`
- `human_review`
- `completed`
- `failed`

### 4.2 Step 级状态
步骤级状态：
- `planned`
- `ready`
- `running`
- `done`
- `failed`
- `blocked`
- `skipped`

驱动方式不要写死成“硬编码流程”，而应该是 event loop：

```text
load state -> 找到 ready step -> 执行 -> evaluator 判断结果 -> 更新状态 -> 如果失败/缺上下文，决定 retry 还是 replan -> 继续下一轮
```

---

## 5. Step 为什么要强约束

每个 step 至少应带：
- `goal`
- `input`
- `output`
- `acceptance`
- `depends_on`
- `retry_policy`

没有验收条件的 step，后面一定会乱。

---

## 6. 实施节奏

建议：
**先串行，后并行。**

v1 先串行跑通闭环，再逐步增强并发能力。

原因很简单，系统最开始最需要验证的是：
- 计划能不能拆对
- 步骤能不能执行
- 结果能不能验收
- 失败能不能触发 replan

不是一上来就追求复杂调度。

---

## 7. Code Review 规划需要什么上下文

为了让 AI review 不瞎说，当时建议从三块准备：
- 上下文
- 工具
- 组织方式

### 7.1 需求上下文
最少要有：
- Jira 标题
- Jira 描述
- 验收标准 / acceptance criteria
- 业务规则
- 测试说明
- 优先级 / 风险标签
- 关联 issue / 历史问题

最好还能补：
- 这次为什么改
- 不改会出什么问题
- 有没有历史事故
- 哪些行为绝不能被破坏

### 7.2 代码变更上下文
- git diff
- changed files
- commit message
- PR 描述
- 涉及模块

想看出实现逻辑问题，还应补：
- 被调用函数
- 调用方
- DTO / schema / model
- config
- test 文件
- 同目录关键文件

### 7.3 运行与技术上下文
- 技术栈
- 框架版本
- 数据库 / 缓存 / MQ / 外部 API
- 权限规则
- 部署 / 环境信息

核心思想是：
**只看 diff 不够，必须给 AI 提供足够但不过量的上下文。**

---

## 8. Code Review Checklist

当时已经沉淀过一版 checklist，核心包括：

### 8.1 需求上下文
- Jira 标题
- Jira 描述
- 验收标准
- 核心业务规则
- 测试说明 / QA 备注
- 风险标签 / 优先级
- 关联 issue / 历史问题
- 本次改动目的
- 不改的风险

### 8.2 代码变更上下文
- git diff
- changed files
- commit message
- PR 描述
- 涉及模块列表
- 被调用函数 / 调用方
- DTO / schema / model
- config / migration / contract
- test 文件

### 8.3 技术与运行上下文
- 技术栈
- 框架版本
- 数据库
- 缓存
- MQ / 外部 API
- 权限规则

---

## 9. 需要提前想清楚的风险

### 9.1 规划质量风险
- plan 拆得太粗，执行时信息不够
- plan 拆得太细，系统变重
- 验收条件不清晰
- planner 幻觉，拆出无意义步骤

要想清楚：
- step 粒度怎么定
- 每个 step 必须有哪些字段
- 什么叫完成
- 什么情况触发 replan

### 9.2 上下文风险
- 只看 diff，抓不到业务问题
- 上下文太少，误判多
- 上下文太多，prompt 污染、成本高、噪音大
- Jira 描述不靠谱，影响整个规划

要想清楚：
- 哪些上下文是 P0
- 哪些是 P1
- 自动扩展到什么程度
- 什么时候停止扩展

### 9.3 工具可靠性风险
这套流程会强依赖 Jira / git / 文件读取工具。

因此不能默认工具永远可靠，而要把失败、超时、缺权限、信息缺失当成正常情况处理。

---

## 10. 这些早期设计对 verification workbench 的意义

虽然这些内容最初不是以 verification workbench 命名的，但它们对后续架构直接产生了影响：
- 验证系统为什么要有 Planner / Orchestrator / Worker 的角色分离
- 为什么状态机比复杂 DAG 更优先
- 为什么需要结构化 step 和验收条件
- 为什么上下文要分层治理
- 为什么系统最终走向“验证主线 + 对话记忆主线 + 上下文编排层”

换句话说，后来的 verification workbench 不是凭空出现的，它是从这些更早的编排与上下文设计判断演进出来的。
