# 开发前 Checklist

这份文档专门用来回答一个非常现实的问题：

**在 verification workbench 真正开始开发之前，还有哪些细节必须先钉住？**

前面的架构、运行时、SRWV loop、扩展点、实施路线，已经把大方向搭好了。  
但真正开工前，如果这些“脏细节”没定清楚，后面大概率会返工。

---

## 1. 模块边界 Checklist

在开始写代码前，先确认下面这些边界都已经明确：

- `Thread` 和 `Task / Run` 的关系是什么
- `Summary / Memory / Knowledge` 各自负责什么
- `Skill / Plugin / Rule` 分别负责什么
- 哪些对象属于稳定内核，哪些允许频繁演进
- UI 层是否只做 projection，不承载业务状态判断

如果这些边界还在摇摆，不要急着开写复杂功能。

---

## 2. 状态机 Checklist

不是只有状态名就够了，还要明确迁移规则：

- 哪些状态存在
- 谁能触发状态迁移
- 哪些迁移是自动的，哪些需要人工确认
- 失败后是否允许 retry
- 什么情况下进入 `paused`
- 什么情况下进入 `needs_approval`
- 什么情况下直接 `failed`
- 什么情况下 `handoff_to_human`
- 中断恢复后从哪一步继续

如果状态迁移没有写清楚，Runtime 很快会变成一堆 if/else。

---

## 3. 数据库 Schema Checklist

在正式建表前，至少确认这些：

- 主键策略（UUID / 字符串 ID / 自增）
- 外键关系
- 唯一索引
- 幂等键设计
- 时间字段统一规范
- 是否需要 soft delete
- JSON 字段的使用边界
- artifact 是只存 metadata，还是存本地路径 / 外链
- 审计表与主业务表如何关联
- migration 如何管理与回滚

特别是 SQLite-first 场景，后面再改 schema 成本会越来越高。

---

## 4. Runtime Contract Checklist

Runtime 真正能不能接得起来，取决于 contract 是否先写死。

至少要明确：

- loop input / output
- action envelope
- skill result
- validation result
- approval request / approval decision
- event payload
- artifact descriptor
- error envelope

如果 contract 不稳定，domain、runtime、plugin、UI 四边都会互相猜。

---

## 5. 错误模型 Checklist

错误不能只靠字符串。

至少要分类：

- 用户输入错误
- 参数校验错误
- 权限错误
- 审批缺失错误
- 数据冲突错误
- 外部系统错误
- 超时错误
- 可重试错误 / 不可重试错误
- 来源不足 / 证据冲突错误

还要明确：

- 哪些错误会中断 loop
- 哪些错误会回到 search/read
- 哪些错误需要直接转人工
- 哪些错误需要写 audit

---

## 6. 权限与审批 Checklist

所有高风险动作都要先定规则。

至少确认：

- 哪些动作属于 external write
- 哪些 skill 必须审批
- 谁能审批
- 审批是否支持过期
- 审批是否支持撤销
- 是否允许 override
- 审批后的执行窗口有多长
- 每次审批要留哪些审计字段

如果这块后补，外部写操作一放开就很危险。

---

## 7. Search / Retrieval Checklist

既然 search 可能要支持浏览器查最新消息，这块细节必须先定：

- freshness 策略怎么定义
- source priority 怎么排序
- 本地搜索和外部搜索谁优先
- browser live search 的超时与重试策略
- 动态页面内容怎么截断
- search 结果怎么缓存
- citation 格式怎么统一
- 多来源冲突如何判定
- read 阶段如何从原文抽 facts

否则 search 很容易又散又贵，还不稳定。

---

## 8. Artifact Checklist

Artifact 最好一开始就统一结构，不然后面 UI、validation、审计都会乱。

至少先统一这些类型：

- screenshot
- raw log
- extracted facts
- citation
- diff
- external URL
- report draft
- approval record

还要确认：

- artifact 是否可版本化
- artifact 是否能挂在 task / run / step_run / thread 上
- artifact 元数据怎么存
- 大文件存哪
- 外链资源失效后怎么处理

---

## 9. UI Projection Checklist

开发前先定几个核心页面看什么：

- thread 页显示哪些上下文块
- task 页显示哪些摘要与约束
- run 页显示哪些 step / artifact / verdict
- approval 页显示哪些待审批动作
- audit 页显示哪些 trace
- SRWV 中间结果是否对用户可见

如果 UI 到时才临时想，会反过来逼 runtime 改结构。

---

## 10. MVP Scope Checklist

第一版最好再收紧一次，确认：

- 哪些是必须完成的
- 哪些明确不做
- 哪些只做 read-only
- 哪些高风险动作暂不开放
- 哪些 AI 能力先不做自动化

建议 MVP 先只包含：

- SQLite schema v1
- Task / Run / Artifact / Verdict 主链
- ChatThread / Message / ContextSnapshot
- 薄 loop
- 基础 audit / approval
- 一个 read-only plugin
- 一个简单 skill
- 基础 validation

---

## 11. 测试策略 Checklist

开发前就先想好怎么测，不然越往后越难补。

至少要有：

- schema migration 测试
- state machine 测试
- plugin contract 测试
- skill integration 测试
- runtime / loop 回归测试
- validation policy 测试
- approval 流程测试
- search provider mock 测试

---

## 12. 工程组织 Checklist

建议确认：

- 代码目录结构
- 包依赖方向
- migration 放哪
- 测试数据怎么组织
- fixture / mock / sample artifact 放哪
- docs 和代码如何同步演进

推荐至少拆成：

```text
src/
  domain/
  runtime/
  plugins/
  skills/
  knowledge/
  infra/
  ui/
```

---

## 13. 一句话结论

真正开始开发前，最该先钉住的不是“我要不要再加一个能力”，而是：

**边界、状态、表结构、contract、错误模型、权限审批、artifact 规范、MVP 范围。**

这些细节一旦先定清楚，后面开发会稳很多。  
这些细节如果一直模糊，架构再漂亮，代码也会越写越散。
