# 标准与数据库设计总稿

这份文档把前面分散的设计进一步收口成一份“开发前可以对齐”的总稿，重点回答：

- 这套系统的标准应该怎么定
- 哪些对象必须落数据库
- 每类对象需要哪些字段
- 这些字段之间怎么对齐
- 哪些信息适合结构化存储，哪些只做 artifact / JSON 承载

目标不是一次性把所有实现写完，而是让后面写代码时，**模型、表、contract、runtime、UI 都能围绕同一套对象说同一种语言。**

---

## 1. 先定四类标准

整个 verification workbench 最少要统一四类标准。

### 1.1 身份标准
所有核心对象都要有稳定 ID，并且跨层一致。

推荐：
- 主业务对象统一用字符串 ID / UUID
- 所有表都有 `id`
- 关联字段统一命名成 `<object>_id`
- 所有运行对象都能挂到 `thread / task / run / workflow_run / trace` 之一

例如：
- `thread_id`
- `task_id`
- `run_id`
- `workflow_definition_id`
- `workflow_run_id`
- `trace_id`

### 1.2 状态标准
状态不能各层随便命名。

至少统一：
- 生命周期状态：`draft`, `active`, `paused`, `completed`, `failed`, `cancelled`
- 等待类状态：`queued`, `pending`, `needs_approval`, `blocked`
- 步骤级状态：`running`, `completed`, `skipped`, `failed`

### 1.3 时间标准
所有运行对象统一带时间字段：
- `created_at`
- `updated_at`
- `started_at`
- `ended_at` / `completed_at`

定时调度对象额外带：
- `next_run_at`
- `last_run_at`
- `start_at`
- `end_at`

### 1.4 审计标准
所有会产生副作用、决策、状态变化的对象，都必须能被审计。

至少统一：
- `created_by`
- `reason`
- `status`
- `error_code`
- `error_message`
- `trace_id`

---

## 2. 哪些东西必须进数据库

不是所有内容都适合进数据库，但下面这些对象**必须结构化保存**。

### 2.1 核心业务对象
- `tasks`
- `task_versions`
- `step_definitions`
- `runs`
- `step_runs`
- `artifacts`
- `verdicts`

### 2.2 对话与上下文对象
- `chat_threads`
- `chat_messages`
- `thread_summaries`
- `memory_items`
- `context_snapshots`
- `thread_task_links`
- `thread_run_links`

### 2.3 SRWV 执行对象
- `srwv_loops`
- `srwv_iterations`
- `srwv_search_results`
- `srwv_read_documents`
- `srwv_read_facts`
- `srwv_write_operations`
- `srwv_validation_results`
- `srwv_validation_issues`
- `srwv_decisions`
- `srwv_citations`

### 2.4 Workflow 自动执行对象
- `workflow_definitions`
- `workflow_definition_versions`
- `workflow_triggers`
- `workflow_runs`
- `workflow_run_steps`
- `workflow_run_events`
- `workflow_run_artifacts`
- `workflow_run_approvals`
- `workflow_run_links`

### 2.5 能力扩展对象
- `plugins`
- `plugin_versions`
- `plugin_actions`
- `skills`
- `skill_versions`
- `skill_bindings`
- `skill_execution_logs`

### 2.6 知识对象
- `knowledge_sources`
- `knowledge_items`
- `knowledge_chunks`
- `knowledge_citations`
- `knowledge_links`
- `retrieval_sessions`
- `retrieval_results`

### 2.7 治理与运行保障对象
- `approval_requests`
- `approval_decisions`
- `audit_events`
- `runtime_errors`
- `trace_spans`
- `provider_call_logs`
- `credentials`（或仅 metadata / reference）
- `environment_profiles`
- `config_entries`

---

## 3. 哪些内容不必重度结构化

有些内容不必拆太细，但仍然要留下可引用入口。

适合走 artifact / blob / JSON 的有：
- 原始网页内容
- 原始邮件正文
- 截图
- trace 原始 payload
- provider 原始响应
- 长文草稿
- 临时工作内存快照

建议方式：
- 数据库存 metadata + path / hash / summary
- 文件本体放文件系统或对象存储

---

## 4. 核心主业务表字段标准

### 4.1 tasks
建议字段：
- `id`
- `title`
- `description`
- `status`
- `priority`
- `owner_id`
- `current_version_id`
- `created_by`
- `created_at`
- `updated_at`

### 4.2 task_versions
建议字段：
- `id`
- `task_id`
- `version_no`
- `goal`
- `success_criteria_json`
- `constraints_json`
- `input_context_json`
- `created_by`
- `created_at`

### 4.3 step_definitions
建议字段：
- `id`
- `task_version_id`
- `step_no`
- `name`
- `goal`
- `input_schema_json`
- `output_schema_json`
- `acceptance_json`
- `depends_on_json`
- `retry_policy_json`
- `created_at`

### 4.4 runs
建议字段：
- `id`
- `task_id`
- `task_version_id`
- `status`
- `environment_id`
- `executor_profile_id`
- `started_at`
- `completed_at`
- `final_verdict_id`
- `created_by`
- `created_at`
- `updated_at`

### 4.5 step_runs
建议字段：
- `id`
- `run_id`
- `step_definition_id`
- `step_no`
- `status`
- `input_json`
- `output_json`
- `error_code`
- `error_message`
- `started_at`
- `ended_at`

### 4.6 artifacts
建议字段：
- `id`
- `artifact_type`
- `owner_type`
- `owner_id`
- `title`
- `summary`
- `storage_uri`
- `content_hash`
- `mime_type`
- `metadata_json`
- `created_at`

### 4.7 verdicts
建议字段：
- `id`
- `run_id`
- `status` (`pass`, `fail`, `inconclusive`, `blocked`)
- `summary`
- `confidence`
- `evidence_count`
- `created_by`
- `created_at`

---

## 5. 对话与上下文字段标准

### 5.1 chat_threads
- `id`
- `title`
- `status`
- `source_platform`
- `source_channel_id`
- `created_at`
- `updated_at`

### 5.2 chat_messages
- `id`
- `thread_id`
- `sender_type`
- `sender_id`
- `content`
- `message_type`
- `source_message_id`
- `created_at`

### 5.3 thread_summaries
- `id`
- `thread_id`
- `summary_type` (`rolling`, `milestone`)
- `content`
- `message_range_start_id`
- `message_range_end_id`
- `created_at`

### 5.4 memory_items
- `id`
- `scope_type`
- `scope_id`
- `memory_type`
- `content`
- `importance`
- `expires_at`
- `created_at`

### 5.5 context_snapshots
- `id`
- `thread_id`
- `task_id`
- `run_id`
- `summary_ids_json`
- `memory_ids_json`
- `fact_refs_json`
- `token_budget`
- `created_at`

---

## 6. SRWV 表字段标准

### 6.1 srwv_loops
- `id`
- `thread_id`
- `task_id`
- `run_id`
- `goal`
- `status`
- `freshness_required`
- `external_write_allowed`
- `max_iterations`
- `current_iteration_no`
- `started_at`
- `completed_at`
- `created_by`
- `created_at`
- `updated_at`

### 6.2 srwv_iterations
- `id`
- `loop_id`
- `iteration_no`
- `state`
- `reason`
- `missing_info_summary`
- `selected_search_strategy`
- `selected_write_type`
- `validation_passed`
- `decision`
- `started_at`
- `ended_at`

### 6.3 srwv_search_results
- `id`
- `loop_id`
- `iteration_id`
- `provider`
- `source_type`
- `source_id`
- `title`
- `snippet`
- `url`
- `timestamp`
- `confidence`
- `why_selected`
- `rank_score`
- `selected_for_read`
- `created_at`

### 6.4 srwv_read_documents
- `id`
- `loop_id`
- `iteration_id`
- `search_result_id`
- `source`
- `excerpt`
- `timestamp`
- `confidence`
- `raw_hash`
- `has_conflict`
- `uncertainty_summary`
- `created_at`

### 6.5 srwv_read_facts
- `id`
- `read_document_id`
- `fact_text`
- `fact_type`
- `normalized_key`
- `normalized_value`
- `confidence`
- `is_latest`
- `conflicts_with_fact_id`
- `created_at`

### 6.6 srwv_write_operations
- `id`
- `loop_id`
- `iteration_id`
- `write_type`
- `target_type`
- `target_id`
- `content`
- `content_hash`
- `status`
- `approval_required`
- `approval_request_id`
- `executed_at`
- `created_at`

### 6.7 srwv_validation_results
- `id`
- `loop_id`
- `iteration_id`
- `policy_name`
- `passed`
- `score`
- `issue_count`
- `decision`
- `created_at`

### 6.8 srwv_validation_issues
- `id`
- `validation_result_id`
- `issue_code`
- `severity`
- `message`
- `details_json`
- `created_at`

### 6.9 srwv_decisions
- `id`
- `loop_id`
- `iteration_id`
- `decision_type`
- `reason`
- `created_at`

### 6.10 srwv_citations
- `id`
- `loop_id`
- `iteration_id`
- `source_type`
- `source_id`
- `excerpt`
- `timestamp`
- `created_at`

---

## 7. Workflow 表字段标准

### 7.1 workflow_definitions
- `id`
- `name`
- `slug`
- `description`
- `status`
- `current_version_id`
- `owner_id`
- `created_at`
- `updated_at`

### 7.2 workflow_definition_versions
- `id`
- `workflow_definition_id`
- `version_no`
- `goal`
- `execution_pattern`
- `input_sources_json`
- `steps_json`
- `write_targets_json`
- `validation_policy_json`
- `permission_policy_json`
- `retry_policy_json`
- `enabled`
- `created_by`
- `created_at`

### 7.3 workflow_triggers
- `id`
- `workflow_definition_id`
- `workflow_version_id`
- `trigger_type`
- `status`
- `cron_expr`
- `timezone`
- `event_type`
- `filters_json`
- `dedupe_key_expr`
- `cooldown_seconds`
- `misfire_policy`
- `max_concurrency`
- `start_at`
- `end_at`
- `next_run_at`
- `last_run_at`
- `created_at`
- `updated_at`

### 7.4 workflow_runs
- `id`
- `workflow_definition_id`
- `workflow_version_id`
- `trigger_id`
- `trigger_type`
- `trigger_event_id`
- `status`
- `reason`
- `started_at`
- `completed_at`
- `retry_count`
- `max_attempts`
- `next_retry_at`
- `last_error_code`
- `last_error_message`
- `created_by`
- `created_at`
- `updated_at`

### 7.5 workflow_run_steps
- `id`
- `workflow_run_id`
- `step_no`
- `step_kind`
- `step_name`
- `status`
- `input_json`
- `output_json`
- `started_at`
- `ended_at`
- `error_code`
- `error_message`

### 7.6 workflow_run_events
- `id`
- `workflow_run_id`
- `event_type`
- `event_source`
- `event_payload_json`
- `created_at`

### 7.7 workflow_run_artifacts
- `id`
- `workflow_run_id`
- `artifact_id`
- `created_at`

### 7.8 workflow_run_approvals
- `id`
- `workflow_run_id`
- `approval_request_id`
- `status`
- `created_at`

### 7.9 workflow_run_links
- `id`
- `workflow_run_id`
- `thread_id`
- `task_id`
- `run_id`
- `created_at`

---

## 8. 能力与知识表字段标准

### 8.1 plugins
- `id`
- `name`
- `provider_type`
- `status`
- `current_version_id`
- `created_at`
- `updated_at`

### 8.2 plugin_versions
- `id`
- `plugin_id`
- `version_no`
- `manifest_json`
- `created_at`

### 8.3 plugin_actions
- `id`
- `plugin_version_id`
- `action_name`
- `input_schema_json`
- `output_schema_json`
- `risk_level`
- `timeout_seconds`
- `retry_policy_json`
- `created_at`

### 8.4 skills
- `id`
- `name`
- `kind`
- `status`
- `current_version_id`
- `created_at`
- `updated_at`

### 8.5 skill_versions
- `id`
- `skill_id`
- `version_no`
- `manifest_json`
- `created_at`

### 8.6 skill_execution_logs
- `id`
- `skill_version_id`
- `workflow_run_id`
- `status`
- `input_summary`
- `output_summary`
- `started_at`
- `ended_at`
- `error_code`

### 8.7 knowledge_sources
- `id`
- `source_type`
- `name`
- `trust_level`
- `freshness_policy_json`
- `created_at`

### 8.8 knowledge_items
- `id`
- `knowledge_source_id`
- `external_ref`
- `title`
- `content_summary`
- `updated_at`
- `created_at`

### 8.9 knowledge_chunks
- `id`
- `knowledge_item_id`
- `chunk_no`
- `content`
- `embedding_ref`
- `metadata_json`
- `created_at`

### 8.10 retrieval_sessions
- `id`
- `thread_id`
- `task_id`
- `goal`
- `query_text`
- `created_at`

### 8.11 retrieval_results
- `id`
- `retrieval_session_id`
- `knowledge_chunk_id`
- `score`
- `selected`
- `created_at`

---

## 9. 治理与运行保障表字段标准

### 9.1 approval_requests
- `id`
- `target_type`
- `target_id`
- `risk_level`
- `reason`
- `input_summary`
- `status`
- `requested_by`
- `requested_at`
- `expires_at`

### 9.2 approval_decisions
- `id`
- `approval_request_id`
- `decision`
- `decided_by`
- `comment`
- `decided_at`

### 9.3 audit_events
- `id`
- `trace_id`
- `event_type`
- `object_type`
- `object_id`
- `actor_id`
- `payload_json`
- `created_at`

### 9.4 runtime_errors
- `id`
- `trace_id`
- `category`
- `code`
- `severity`
- `message`
- `retryable`
- `details_json`
- `created_at`

### 9.5 trace_spans
- `id`
- `trace_id`
- `parent_span_id`
- `span_type`
- `object_type`
- `object_id`
- `started_at`
- `ended_at`
- `status`

### 9.6 provider_call_logs
- `id`
- `trace_id`
- `provider_name`
- `action_name`
- `request_summary`
- `response_summary`
- `latency_ms`
- `status`
- `error_code`
- `retry_count`
- `created_at`

### 9.7 environment_profiles
- `id`
- `name`
- `environment_type`
- `config_json`
- `created_at`

### 9.8 config_entries
- `id`
- `scope_type`
- `scope_id`
- `config_key`
- `config_value_json`
- `updated_at`

---

## 10. 字段怎么对齐

为了减少后面各层互相猜，建议统一这几个对齐原则。

### 10.1 状态字段统一叫 `status`
不要一层叫 `state` 一层叫 `phase` 一层叫 `result`。  
特殊子阶段可以额外补 `step_kind`、`decision_type`。

### 10.2 错误字段统一 `error_code` / `error_message`
复杂细节进 `details_json` 或 `runtime_errors`。

### 10.3 JSON 字段后缀统一 `_json`
例如：
- `filters_json`
- `metadata_json`
- `payload_json`
- `steps_json`

### 10.4 关联字段统一 `<object>_id`
例如：
- `workflow_run_id`
- `approval_request_id`
- `trace_id`

### 10.5 时间字段统一 UTC 存储
展示时再按时区转换。

---

## 11. 最后一句话

这套标准的核心不是“表越多越高级”，而是：

**把定义、执行、上下文、能力、知识、治理、可观测性这些层真正分开，并且让它们通过统一字段和统一对象关系对齐。**

这样后面无论你接新功能、做自动化、加浏览器搜索、加审批、加新 provider，都不会把系统主干搞乱。
