# Search / Read / Write / Validation 表结构设计

这份文档补全 SRWV（Search / Read / Write / Validation）循环在 SQLite-first 架构下的落库设计。

目标不是把所有运行细节都塞进一张大表，而是把：
- loop 本身
- 每一轮执行
- search 结果
- read 抽取事实
- write 产物
- validation 结果
- 最终决策

拆成可追踪、可恢复、可审计的几组表。

---

## 1. 设计原则

### 1.1 一次 loop 与每一轮尝试分开

一个 SRWV 任务可能会循环多轮。  
所以要区分：
- `srwv_loops`：一次整体目标执行
- `srwv_iterations`：其中的每一轮

### 1.2 来源、事实、产出、校验分开

Search 找的是来源。  
Read 提取的是事实。  
Write 生成的是产出。  
Validation 评估的是结果。

这四类对象不要混表。

### 1.3 每一步都要能回放

后面如果用户问：
- 为什么得出这个结论？
- 为什么又去搜了一次？
- 为什么没直接发出去？

系统必须能回看每一轮的输入、来源、事实、校验问题和决策。

---

## 2. 核心表总览

建议拆成下面这些表：

```text
srwv_loops
srwv_iterations
srwv_search_results
srwv_read_documents
srwv_read_facts
srwv_write_operations
srwv_validation_results
srwv_validation_issues
srwv_decisions
srwv_citations
```

如果再往下细化，还可以加：
- `srwv_provider_calls`
- `srwv_state_transitions`
- `srwv_approval_links`

---

## 3. srwv_loops

表示一次完整的 SRWV 目标执行。

### 字段建议
- `id`
- `thread_id`
- `task_id`
- `run_id`
- `goal`
- `status` (`draft`, `running`, `blocked`, `needs_approval`, `completed`, `failed`, `cancelled`)
- `freshness_required` (bool)
- `external_write_allowed` (bool)
- `max_iterations`
- `current_iteration_no`
- `started_at`
- `completed_at`
- `created_by`
- `created_at`
- `updated_at`

### 作用
这一张表回答：
- 这个 loop 是为了什么目标
- 关联了哪个 thread / task / run
- 现在整体进行到哪里
- 是否要求最新来源
- 是否允许外部写

---

## 4. srwv_iterations

表示 loop 中的一轮执行。

### 字段建议
- `id`
- `loop_id`
- `iteration_no`
- `state` (`planning`, `searching`, `reading`, `writing`, `validating`, `completed`, `failed`)
- `reason`
- `missing_info_summary`
- `selected_search_strategy`
- `selected_write_type` (`draft`, `local`, `external`, `none`)
- `validation_passed` (bool)
- `decision` (`continue`, `retry_current_step`, `go_to_search`, `go_to_read`, `needs_approval`, `handoff_to_human`, `stop`)
- `started_at`
- `ended_at`

### 作用
这一张表是回放核心。  
后面看历史时，最常看的就是它。

---

## 5. srwv_search_results

记录 search 阶段发现的候选来源。

### 字段建议
- `id`
- `loop_id`
- `iteration_id`
- `provider` (`local_search`, `knowledge_search`, `web_search`, `browser_live_search`)
- `source_type` (`web`, `thread`, `knowledge`, `artifact`, `email`, `confluence`, `jira`, `dashboard`)
- `source_id`
- `title`
- `snippet`
- `url`
- `timestamp`
- `confidence`
- `why_selected`
- `rank_score`
- `selected_for_read` (bool)
- `created_at`

### 说明
Search 阶段可能找到很多来源，但不一定都进入 Read。  
所以 `selected_for_read` 很有用。

---

## 6. srwv_read_documents

记录从来源中读出来的结构化文档级结果。

### 字段建议
- `id`
- `loop_id`
- `iteration_id`
- `search_result_id`
- `source`
- `excerpt`
- `timestamp`
- `confidence`
- `raw_hash`
- `has_conflict` (bool)
- `uncertainty_summary`
- `created_at`

这一层代表“从一个来源里读出了什么”。

---

## 7. srwv_read_facts

进一步把 read document 拆成事实粒度。

### 字段建议
- `id`
- `read_document_id`
- `fact_text`
- `fact_type` (`status`, `event`, `timestamp`, `owner`, `risk`, `actionable`, `unknown`)
- `normalized_key`
- `normalized_value`
- `confidence`
- `is_latest` (bool)
- `conflicts_with_fact_id`
- `created_at`

### 作用
这一层很关键，因为 validation 往往不是看整段摘录，而是看事实间是否冲突、是否过时。

---

## 8. srwv_write_operations

记录 write 阶段的产出。

### 字段建议
- `id`
- `loop_id`
- `iteration_id`
- `write_type` (`draft`, `local`, `external`)
- `target_type` (`thread_message`, `summary`, `report`, `artifact_note`, `email`, `confluence_page`, `jira_comment`, `none`)
- `target_id`
- `content`
- `content_hash`
- `status` (`prepared`, `pending_approval`, `written`, `skipped`, `failed`)
- `approval_required` (bool)
- `approval_request_id`
- `executed_at`
- `created_at`

### 说明
这里既记录草稿，也记录真正落出去的外部写操作。

---

## 9. srwv_validation_results

记录一轮 validation 的总体结果。

### 字段建议
- `id`
- `loop_id`
- `iteration_id`
- `pass` (bool)
- `evidence_sufficiency_score`
- `freshness_ok` (bool)
- `conflict_detected` (bool)
- `citations_complete` (bool)
- `goal_coverage_ok` (bool)
- `permission_ok` (bool)
- `approval_ok` (bool)
- `next_action`
- `summary`
- `created_at`

### 作用
给 decide 阶段一个结构化输入，而不是只给一段模糊说明。

---

## 10. srwv_validation_issues

把 validation 发现的问题单独拆出来。

### 字段建议
- `id`
- `validation_result_id`
- `issue_type` (`missing_source`, `stale_information`, `source_conflict`, `missing_citation`, `goal_not_met`, `permission_denied`, `approval_missing`)
- `message`
- `severity` (`low`, `medium`, `high`, `critical`)
- `suggested_next_action`
- `related_source_id`
- `related_fact_id`
- `created_at`

这样后面 UI 和 loop 都更容易用。

---

## 11. srwv_decisions

记录 validation 之后的最终决策。

### 字段建议
- `id`
- `loop_id`
- `iteration_id`
- `decision_type` (`continue`, `retry_current_step`, `go_to_search`, `go_to_read`, `needs_approval`, `handoff_to_human`, `stop`, `fail`)
- `reason`
- `next_state`
- `created_at`

虽然 `iterations` 里也可以有 decision，但单独拆出来更利于审计与后续分析。

---

## 12. srwv_citations

记录 write 产出引用了哪些来源。

### 字段建议
- `id`
- `write_operation_id`
- `source_type`
- `source_id`
- `read_document_id`
- `fact_id`
- `citation_label`
- `created_at`

### 作用
回答：
- 最终这段输出引用了什么
- 来自哪个 read document / fact
- 是否有引用依据

---

## 13. 可选增强表

### 13.1 srwv_provider_calls
记录每一次 provider 调用，比如 browser_live_search 真正打开了哪个页面。

字段可包括：
- `provider`
- `request_payload`
- `response_summary`
- `status`
- `duration_ms`
- `error_code`

### 13.2 srwv_state_transitions
记录状态变化：
- `from_state`
- `to_state`
- `reason`
- `created_at`

### 13.3 srwv_approval_links
把 write external 跟 approval runtime 明确关联。

---

## 14. 与主系统对象的关系

SRWV 不应该是一个孤立小系统。  
它应该挂在主干对象旁边：

- 一个 `thread` 可以触发多个 `srwv_loops`
- 一个 `task` 可以关联多个 `srwv_loops`
- 一个 `run` 可以包含多个 `srwv_loops`
- `write_local` 产出可以回写 thread / summary / artifact
- `write_external` 产出可以挂接 approval / audit / artifact

也就是说，SRWV 是 Loop Engine 的一种标准执行模式，不是另起炉灶。

---

## 15. 最小落地版本建议

如果先做 MVP，可以先只落这 6 张：

- `srwv_loops`
- `srwv_iterations`
- `srwv_search_results`
- `srwv_read_documents`
- `srwv_write_operations`
- `srwv_validation_results`

然后第二阶段再补：
- `srwv_read_facts`
- `srwv_validation_issues`
- `srwv_citations`
- `srwv_provider_calls`

这样能先把闭环跑通，不至于一开始过重。

---

## 16. 一句话结论

SRWV 的表结构设计重点不是“把流程记录下来”这么简单，  
而是要让系统在任何时候都能回答这几个问题：

- 这次目标是什么？
- 搜了哪些来源？
- 读出了哪些事实？
- 写了什么内容？
- 为什么通过或不通过？
- 下一步为什么这样决定？

只有这样，这个 loop 才真的可恢复、可审计、可扩展。
