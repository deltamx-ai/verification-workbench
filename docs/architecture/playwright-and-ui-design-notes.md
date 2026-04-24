# Playwright 与桌面验证层设计纪要（仅保留 Alma 的设计输出）

这份文档收录与 **Playwright 验证层、桌面页面结构、执行链路、打包集成** 相关的设计对话整理稿。

---

## 1. Playwright 是否适合做验证

结论很明确：
**可行，而且很适合端到端验证。**

它适合验证：
- 页面交互是否正确
- 表单提交后的 UI / 接口行为
- 权限、跳转、状态流转
- 回归测试
- 用户真实操作路径是否能跑通

如果目标是判断：
- 需求到底有没有实现
- 改完后核心流程还能不能跑

那 Playwright 很合适。

但它不适合单独承担所有验证，尤其不适合验证：
- 深层代码逻辑
- service 内部事务
- 边界条件完整性
- 并发竞争类问题

所以最推荐的组合是：
- 静态 review 看代码问题和实现逻辑风险
- Playwright 验用户流程
- API / 集成测试验接口和系统联动
- 必要时补单测、压测或并发测试

---

## 2. AI 如何知道验证哪个功能

不要靠 AI 自己读代码“悟出来”，必须给结构化输入。

最少应提供一个验证计划输入，包含：
- Jira 标题
- Jira 描述
- 验收标准
- git diff
- 影响页面 / 接口
- 重点风险点

然后先让 AI 产出结构化验证目标，再进入执行。

核心思想是：
**验证目标必须结构化，不靠自由发挥。**

---

## 3. 登录与准备层设计

Playwright 在验证前先要解决环境问题：
- 是否需要登录
- 登录态怎么拿
- 测试账号是谁
- 是否要预置数据

有三种常见策略：
1. 录一份 `storageState`
2. 每次测试前自动执行登录脚本
3. 复用已登录浏览器状态

推荐方案是：
**`storageState + 独立测试账号`**

因为它更稳，也更适合内部系统验证。

---

## 4. 为了方便接入 Rust，需要提前考虑什么

如果想方便集成到 Rust，从一开始就该按“分层 + 标准输入输出”设计。

关键细节包括：

### 4.1 登录态
- 用 `storageState` 还是自动登录
- 测试账号是否固定
- 登录失效如何自动刷新
- 多环境账号是否隔离

### 4.2 测试数据
- 前置数据谁准备
- 测完是否清理
- 是否污染真实环境
- 幂等性怎么保证

### 4.3 验证目标
- 从 Jira 提取哪些场景
- 哪些是 happy path
- 哪些是异常 / 边界
- diff 影响范围怎么映射到验证计划

### 4.4 断言设计
断言不能只看 UI，还要考虑：
- 页面文案
- 接口返回
- 数据库状态
- 事件 / 埋点 / 队列副作用

不然就会出现“看起来成功，实际上没成功”的误判。

### 4.5 环境差异
- 本地 / dev / staging 是否一致
- feature flag 是否影响结果
- 第三方服务是否需要 mock
- 网络波动造成的 flaky case 怎么处理

### 4.6 失败归因
失败到底是：
- 代码实现问题
- 用例问题
- 数据问题
- 环境问题
- Playwright 脚本问题

如果失败归因做不好，结果回流到系统后没有实际价值。

---

## 5. Playwright 验证层下一步该怎么设计

当时给出的建议是继续往下拆这几个模块：

### 5.1 验证计划生成器
输入：
- Jira 需求
- acceptance criteria
- git diff
- review 风险点

输出：
- 要测哪些功能
- happy path
- error path
- edge cases
- 前置条件

### 5.2 登录与会话管理
- `storageState` 复用
- 测试账号管理
- 登录失效刷新
- 多环境隔离

### 5.3 Playwright Worker
职责：
- 启动测试
- 加载 storageState
- 执行 case
- 收集 screenshot / trace / console / network
- 输出统一 JSON 结果

### 5.4 验证结果结构
结果必须标准化。

核心思想是：
**执行器产物和编排器事实要分离，但必须能结构化回流。**

---

## 6. 验证层完整架构

验证层不只是“跑个 UI 自动化”，而是要同时解决四件事：
1. 知道该测什么
2. 知道怎么登录 / 准备环境
3. 知道怎么执行验证
4. 知道怎么把结果回流给上层系统做判断

因此它应被设计成一个独立层，而不是几条 Playwright 脚本。

整体链路可以理解为：

Jira / Diff / Review Result
→ Validation Planner
→ Validation Plan
→ Verification Orchestrator
→ Auth Manager / Test Data Manager / Playwright Worker / API Verify Worker / Result Collector / Evaluation Adapter
→ 输出验证事实与证据

---

## 7. Playwright 用什么实现

结论：
**直接用 Node.js + TypeScript。**

不建议用 Rust 直接控浏览器。

原因是：
- Rust 适合做编排，不适合处理浏览器自动化细节
- Playwright 官方生态在 Node.js / TypeScript 上最成熟
- 登录态、trace、video、screenshot 都现成
- 后面 AI 生成测试代码也更方便
- 出问题时资料最多

推荐组合：
- `playwright`
- `@playwright/test`
- TypeScript

推荐目录形态：
```text
verify/
  package.json
  playwright.config.ts
  auth/
    login.setup.ts
    storageState/
  specs/
    generated/
    shared/
  fixtures/
  report/
```

---

## 8. 后续如何打包进桌面应用

设计里已经明确：
- `Tauri` 负责桌面壳与 UI
- `Rust core` 负责编排、状态、汇总
- `Playwright runner` 负责具体验证执行
- 中间通过 `JSON contract` 通信
- 输出可以包含 `json + trace + video + screenshot + logs`

也就是说，Playwright 不应该嵌进 UI 逻辑里，而应该作为一个可替换 runner 被 Rust core 调度。

---

## 9. axcli 的判断

当时顺手评估过 `axcli`，结论很直接：
- 它是 macOS Accessibility API 的 CLI
- 更像是自动化原生 macOS 应用的工具
- 不适合当 Web 验证层主方案

因此判断是：
**主方案仍然是 Playwright + TypeScript。**

`axcli` 只能作为以后补充原生 macOS 桌面控件自动化的旁路工具，不能放核心链路。

---

## 10. Figma 要不要现在接入

判断是：
**要，但不是现在就强依赖。**

建议分两层：

### 10.1 前期架构设计阶段
先不强依赖 Figma，先定这些：
- 页面树
- 页面结构
- 流程图
- 状态流转
- 组件分层

### 10.2 UI 落地阶段
再接 Figma，把前面的结构稿转成：
- 信息架构
- 页面线框图
- 关键页面高保真
- 设计规范

一句话：
**架构先行，Figma 后接。**

---

## 11. 桌面应用页面结构

桌面应用应采用：
**左侧主导航 + 右侧工作区**

一级页面不要太多，重点是围绕这条主线组织：

**任务管理 → 执行监控 → 结果判断 → 结论回流 → 环境配置**

产品定位不是“写自动化脚本”，而是：
- 创建验证任务
- 选择环境 / 输入验证目标
- 启动执行
- 查看实时进度、日志、截图、步骤
- 执行后查看结果与判定
- 输出可回流的结论

页面结构本质上是在支撑这条闭环，而不是做一个普通测试工具面板。

---

## 12. 更底层的系统判断

更深一层的判断是：
这个系统本质上不应该被定义成“一个带页面的 Playwright 工具”。

它更应该被定义为：
**一个面向验证任务的本地编排与执行内核（Verification Orchestration Kernel）**

含义是：
- UI 只是壳
- Playwright 只是首个执行器
- 日志 / 截图只是产物的一部分
- 结果页只是某种投影视图

如果一开始就做成“桌面端调用 Playwright 脚本并展示结果”，后面会遇到典型问题：
- 新 runner 接不进去
- 执行事实和页面视图耦合
- 证据与判定能力没法演进
- 结果回流只能靠拼日志

所以最重要的设计判断其实是：
**先定义内核，再定义 UI，再接入 Playwright。**
