# 项目推荐目录树（含 Plan Runtime 位置）

## 结论

`Plan Runtime` 建议作为**单独目录**存在，而不是塞进 `agents/`、`workflows/` 或 `tools/` 下面。

推荐放在：

```text
src/plan-runtime/
```

原因很简单：它不是某一个零散模块，而是一整套运行编排内核，负责把用户请求转成可执行步骤，并驱动工具循环、状态更新和重规划。

---

## 推荐目录树

```text
project-root/
├── package.json
├── tsconfig.json
├── README.md
├── docs/
│   ├── architecture/
│   ├── prompts/
│   └── product/
│
├── src/
│   ├── app/
│   │   ├── bootstrap.ts
│   │   ├── container.ts
│   │   └── routes.ts
│   │
│   ├── plan-runtime/
│   │   ├── index.ts
│   │   ├── runtime.ts
│   │   ├── types.ts
│   │   ├── intent-parser.ts
│   │   ├── tool-matcher.ts
│   │   ├── strategy-registry.ts
│   │   ├── plan-generator.ts
│   │   ├── step-selector.ts
│   │   ├── step-executor.ts
│   │   ├── result-normalizer.ts
│   │   ├── state-reducer.ts
│   │   ├── goal-evaluator.ts
│   │   ├── replanner.ts
│   │   └── approval-gate.ts
│   │
│   ├── tools/
│   │   ├── index.ts
│   │   ├── registry.ts
│   │   ├── capabilities.ts
│   │   └── adapters/
│   │       ├── grep.ts
│   │       ├── read.ts
│   │       ├── edit.ts
│   │       ├── bash.ts
│   │       ├── task-agent.ts
│   │       ├── web-search.ts
│   │       └── browser.ts
│   │
│   ├── skills/
│   │   ├── index.ts
│   │   ├── skill-registry.ts
│   │   ├── skill-matcher.ts
│   │   └── executors/
│   │
│   ├── agents/
│   │   ├── index.ts
│   │   ├── managed-agent-registry.ts
│   │   ├── delegation-policy.ts
│   │   └── adapters/
│   │
│   ├── workflows/
│   │   ├── index.ts
│   │   ├── scheduler.ts
│   │   ├── triggers.ts
│   │   ├── workflow-runtime.ts
│   │   └── templates/
│   │
│   ├── memory/
│   │   ├── index.ts
│   │   ├── thread-memory.ts
│   │   ├── people-memory.ts
│   │   ├── summary-writer.ts
│   │   └── retrieval.ts
│   │
│   ├── threads/
│   │   ├── index.ts
│   │   ├── thread-store.ts
│   │   ├── message-graph.ts
│   │   └── thread-events.ts
│   │
│   ├── data/
│   │   ├── index.ts
│   │   ├── db.ts
│   │   ├── migrations/
│   │   ├── repositories/
│   │   └── models/
│   │
│   ├── ui/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── hooks/
│   │   ├── stores/
│   │   └── view-models/
│   │
│   ├── services/
│   │   ├── logging/
│   │   ├── config/
│   │   ├── provider/
│   │   ├── attachment/
│   │   └── telemetry/
│   │
│   └── shared/
│       ├── ids.ts
│       ├── errors.ts
│       ├── logger.ts
│       ├── constants.ts
│       └── utils/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
└── scripts/
    ├── dev.sh
    ├── build.sh
    └── migrate.sh
```

---

## 为什么 Plan Runtime 要独立

因为它的职责横跨多个子系统：

- 它要读用户请求
- 它要判断该用什么工具
- 它要决定要不要委托 agent
- 它要驱动步骤执行
- 它要更新状态
- 它要在失败后 replan

所以它不是：
- `tools/` 的一部分
- `agents/` 的一部分
- `ui/` 的一部分
- `workflows/` 的一部分

它更像是这些东西上面的**调度与编排内核**。

---

## 边界建议

### `plan-runtime/`
只负责：
- 想怎么做
- 下一步做什么
- 成功/失败后怎么推进

不要负责：
- 具体工具实现细节
- 具体数据库读写细节
- UI 展示逻辑

### `tools/`
只负责：
- 真正执行 Grep / Read / Edit / Bash / Browser / WebSearch / Task 等动作

### `agents/`
只负责：
- 对接 managed agents / delegation policy

### `ui/`
只负责：
- 把 runtime state 渲染成用户看得懂的界面

---

## 一句话

在整个项目里，`Plan Runtime` 最合适的归位就是：

> **作为 `src/plan-runtime/` 独立存在，充当项目的执行编排中枢。**


---

## 目录树分层用途说明

下面不是只看“文件夹叫什么”，而是看**每一层在整个项目里负责什么**。

### 第 1 层：项目根层 `project-root/`

用途：
- 作为整个仓库的顶层容器
- 放项目入口配置、文档、测试、脚本
- 定义这个仓库怎么安装、怎么构建、怎么测试、怎么被理解

这一层更像“仓库外壳”和“工程边界”。

典型内容：
- `package.json`：依赖、脚本、工程入口
- `tsconfig.json`：TypeScript 编译规则
- `README.md`：仓库说明与索引
- `docs/`：设计、架构、产品文档
- `tests/`：测试代码
- `scripts/`：开发脚本、迁移脚本、构建脚本

---

### 第 2 层：应用装配层 `src/app/`

用途：
- 做启动装配
- 组装各个子系统
- 处理依赖注入、路由、初始化顺序

这一层不负责具体业务逻辑，更像“总装车间”。

典型职责：
- 初始化数据库连接
- 注册 tool registry / skill registry / agent registry
- 创建 `PlanRuntime` 实例
- 把 UI、runtime、data、services 连起来

---

### 第 3 层：运行编排层 `src/plan-runtime/`

用途：
- 这是项目的执行中枢
- 负责把用户请求转成 plan
- 决定下一步做什么
- 驱动 plan → act → observe → replan 循环

这一层回答的是：
- 用户到底要做什么？
- 可以用哪些工具？
- 先做哪一步？
- 失败了怎么改路？
- 什么叫完成？

它是“思考与调度层”，不是具体工具实现层。

子模块用途：
- `intent-parser.ts`：解析用户意图
- `tool-matcher.ts`：挑可用工具/技能/代理
- `strategy-registry.ts`：按任务类型挑规划策略
- `plan-generator.ts`：产出结构化 plan
- `step-selector.ts`：选下一步
- `step-executor.ts`：发起单步执行
- `state-reducer.ts`：更新 runtime 状态
- `goal-evaluator.ts`：判断任务是否完成
- `replanner.ts`：失败后重规划
- `approval-gate.ts`：高风险步骤审批

---

### 第 4 层：能力执行层 `src/tools/`

用途：
- 提供真正干活的原子能力
- 把 planner 选中的动作执行掉

这一层不负责“为什么这么做”，只负责“把这一步做掉”。

典型职责：
- 搜索文件
- 读取文件
- 编辑文件
- 跑 shell 命令
- 委托 agent
- 浏览器操作
- web search

可以理解成 runtime 的“手脚”。

---

### 第 5 层：扩展能力层 `src/skills/`

用途：
- 承接可安装、可复用、可组合的高阶能力
- 让一些复杂任务不用从原子工具一步步拼

这一层更像“经验包”或“能力插件层”。

典型职责：
- skill registry
- skill matching
- skill executor
- 社区技能/本地技能接入

它和 `tools/` 的区别是：
- `tools/` 偏原子动作
- `skills/` 偏封装好的复合能力

---

### 第 6 层：代理协作层 `src/agents/`

用途：
- 管理不同 specialist agent
- 定义什么时候该委托、委托给谁
- 对接外部/内部代理执行

这一层更像“协作接口层”。

典型职责：
- managed agent registry
- delegation policy
- developer / researcher / planner 等代理接入

它不直接规划整个任务，而是被 `plan-runtime/` 调用。

---

### 第 7 层：流程自动化层 `src/workflows/`

用途：
- 管理长期自动化、定时任务、事件触发流程
- 适合那种不是单轮对话，而是持续运行的流程

这一层偏“自动化编排”，和 `plan-runtime/` 的区别是：
- `plan-runtime/` 偏一次任务执行循环
- `workflows/` 偏长期流程与触发器

典型职责：
- scheduler
- triggers
- workflow runtime
- workflow templates

---

### 第 8 层：记忆与上下文层 `src/memory/`

用途：
- 保存、提取、汇总上下文
- 让系统记得用户、线程、历史讨论、长期偏好

这一层回答的是：
- 之前聊过什么？
- 这个人是谁？
- 这个线程做到了哪？
- 哪些信息应该长期保存？

它是 runtime 的“长期上下文供应层”。

---

### 第 9 层：线程消息层 `src/threads/`

用途：
- 管理 thread、message、message graph、线程事件
- 给 plan-runtime 和 memory 提供消息级原始数据

这一层偏“会话结构层”。

典型职责：
- thread store
- message graph
- thread events
- 父子线程关系

---

### 第 10 层：数据持久化层 `src/data/`

用途：
- 负责数据库、模型、仓储、迁移
- 提供稳定的数据读写边界

这一层不要带太多业务判断，它更像“存储抽象层”。

典型职责：
- SQLite / DB connection
- migrations
- repositories
- models

---

### 第 11 层：界面呈现层 `src/ui/`

用途：
- 把 runtime 状态、线程状态、工具执行状态变成可交互界面
- 服务桌面端工作台体验

这一层回答的是：
- 用户现在看到了什么？
- 当前步骤展示成什么样？
- 怎么显示 blocked / running / completed？

它不该自己决定 plan，而是消费 runtime 的 view model。

---

### 第 12 层：通用服务层 `src/services/`

用途：
- 放横切能力
- 给其他层提供基础设施支持

典型职责：
- logging
- config
- provider 接入
- attachment 管理
- telemetry

这一层像“基础设施服务站”。

---

### 第 13 层：共享基础层 `src/shared/`

用途：
- 放所有层都会用到的最小公共能力
- 避免重复定义常量、错误类型、工具函数

典型职责：
- ids
- errors
- logger helpers
- constants
- utils

这层要保持克制，不然很容易变成杂物箱。

---

## 一句话总结

如果把整个项目看成一台机器：

- `app/` 是装配与启动
- `plan-runtime/` 是大脑和调度中枢
- `tools/` 是手脚
- `skills/` 是经验包
- `agents/` 是外部协作者
- `workflows/` 是长期自动化流水线
- `memory/threads/` 是上下文系统
- `data/` 是底层存储
- `ui/` 是用户看到的外壳
- `services/shared/` 是基础设施
