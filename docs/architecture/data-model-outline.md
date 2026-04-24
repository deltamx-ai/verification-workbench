# 数据模型纲要

## 验证对象
### Task
- id, title, goal, description, status, priority, owner, current_version_id, default_environment_id, created_at, updated_at

### TaskVersion
- id, task_id, version_no, objective_snapshot, assumptions_snapshot, success_criteria_snapshot, status, published_at

### StepDefinition
- id, task_version_id, step_key, name, order_index, step_type, input_contract, expected_output_contract, retry_policy, timeout_ms

### Run
- id, task_id, task_version_id, thread_id, environment_id, executor_profile_id, status, started_at, ended_at, submitted_by, summary

### StepRun
- id, run_id, step_definition_id, status, started_at, ended_at, attempt_no, input_payload, output_payload, error_code, error_message

### Artifact
- id, owner_type, owner_id, artifact_type, title, storage_kind, storage_ref, mime_type, size_bytes, checksum, produced_at, metadata_json

### Verdict
- id, subject_type, subject_id, verdict_type, rationale, evidence_summary, source_type, confidence, created_at, created_by

## 协作与记忆对象
### ChatThread
- id, title, thread_type, status, pinned_task_id, pinned_run_id, last_message_at, last_context_snapshot_id, created_at

### ChatMessage
- id, thread_id, role, message_type, content_text, structured_payload_json, related_task_id, related_run_id, created_at

### ThreadSummary
- id, thread_id, summary_kind, time_range_start, time_range_end, summary_text, structured_facts_json, generated_at

### MemoryItem
- id, scope_type, scope_id, memory_type, title, content, normalized_fact_json, importance, freshness_score, valid_until, source_thread_id, created_at

### ContextSnapshot
- id, thread_id, snapshot_reason, target_task_id, target_run_id, included_recent_message_ids_json, included_summary_ids_json, included_memory_ids_json, included_task_ids_json, included_run_ids_json, assembled_context_json, token_budget, created_at
