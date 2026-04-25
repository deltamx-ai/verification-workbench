# Workflow 表结构设计

这份文档补全“流程定义 / 定时执行 / 自动执行”在 SQLite-first 架构下的落库设计。

目标是把以下几个对象拆清楚：
- workflow 定义
- workflow 版本
- trigger 配置
- workflow run
- workflow step
- workflow event
- workflow artifact
- workflow approval

这样系统才能真正支持：
- 用户说一个流程后保存下来
- 后续定时执行
- 事件触发自动执行
- 每次执行都有完整回放与审计

---

## 1. 设计原则

### 1.1 定义与执行分开

`workflow_definition` 表示“流程是什么”。
`workflow_run` 表示“某次实际执行”。

这一点必须分开，不然后面一改流程定义，历史执行就全乱掉。

### 1.2 版本与 trigger 分开

workflow 会演进，但触发器也会演进。  
所以建议把：
- definition version
- trigger config

拆开存，不要都塞到一列 JSON 里就完事。

### 1.3 步骤与运行事件都要留痕

自动执行一旦出了问题，最怕的是：
- 不知道为什么被触发
- 不知道跑到哪一步失败
- 不知道哪一步卡在审批

所以 step 和 event 都需要落库。

---

## 2. 核心表总览

建议至少有这些表：

```text
workflow_definitions
workflow_definition_versions
workflow_triggers
workflow_runs
workflow_run_steps
workflow_run_events
workflow_run_artifacts
workflow_run_approvals
workflow_run_links
```

如果后面需要再细化，可以补：
- `workflow_trigger_state`
- `workflow_run_retries`
- `workflow_run_errors`

---

## 3. workflow_definitions

表示一个流程的稳定身份。

### 字段建议
- `id`
- `name`
- `slug`
- `description`
- `status` (`draft`, `active`, `paused`, `deprecated`, `archived`)
- `current_version_id`
- `owner_id`
- `created_at`
- `updated_at`

### 作用
这一层回答：
- 这个流程是谁
- 它现在是否启用
- 当前默认版本是哪一版

---

## 4. workflow_definition_versions

表示某个 workflow 在某一时刻冻结下来的定义。

### 字段建议
- `id`
- `workflow_definition_id`
- `version_no`
- `goal`
- `execution_pattern` (`srwv`, `skill_chain`, `hybrid`)
- `input_sources_json`
- `steps_json`
- `write_targets_json`
- `validation_policy_json`
- `permission_policy_json`
- `retry_policy_json`
- `enabled`
- `created_by`
- `created_at`

### 作用
这一层回答：
- 这一版流程到底怎么跑
- 当时有哪些步骤、策略、目标

每次 workflow run 都应该绑定某个 version，而不是动态读取最新配置。

---

## 5. workflow_triggers

表示触发器配置。

### 字段建议
- `id`
- `workflow_definition_id`
- `workflow_version_id`
- `trigger_type` (`manual`, `schedule`, `event`)
- `status` (`active`, `paused`, `disabled`)
- `cron_expr`
- `timezone`
- `event_type`
- `filters_json`
- `dedupe_key_expr`
- `cooldown_seconds`
- `misfire_policy` (`skip`, `run_once_when_resumed`, `catch_up_all`)
- `max_concurrency`
- `start_at`
- `end_at`
- `next_run_at`
- `last_run_at`
- `created_at`
- `updated_at`

### 说明
- 定时任务主要用 `cron_expr + timezone`
- 事件任务主要用 `event_type + filters_json`
- 手动触发可以只有 trigger_type=manual

---

## 6. workflow_runs

表示某次具体执行。

### 字段建议
- `id`
- `workflow_definition_id`
- `workflow_version_id`
- `trigger_id`
- `trigger_type` (`manual`, `schedule`, `event`)
- `trigger_event_id`
- `status` (`queued`, `running`, `paused`, `needs_approval`, `completed`, `failed`, `cancelled`)
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

### 作用
这一层是自动执行的主回放对象。

---

## 7. workflow_run_steps

表示本次 workflow run 中的具体步骤。

### 字段建议
- `id`
- `workflow_run_id`
- `step_no`
- `step_kind` (`search`, `read`, `write`, `validate`, `decision`, `skill`, `plugin_action`, `approval_wait`)
- `step_name`
- `status` (`pending`, `running`, `completed`, `skipped`, `failed`, `blocked`)
- `input_json`
- `output_json`
- `started_at`
- `ended_at`
- `error_code`
- `error_message`

### 说明
如果 workflow 内部执行的是 SRWV，这里就能把 search/read/write/validate 每一步都记录下来。

---

## 8. workflow_run_events

记录触发与运行期事件。

### 字段建议
- `id`
- `workflow_run_id`
- `event_type`
- `event_source`
- `event_payload_json`
- `created_at`

### 常见事件
- `workflow.triggered`
- `workflow.step_started`
- `workflow.step_completed`
- `workflow.approval_requested`
- `workflow.retry_scheduled`
- `workflow.completed`
- `workflow.failed`

这层很适合做 timeline 与调试。

---

## 9. workflow_run_artifacts

挂 workflow 运行产物。

### 字段建议
- `id`
- `workflow_run_id`
- `workflow_run_step_id`
- `artifact_type`
- `artifact_id`
- `artifact_uri`
- `summary`
- `created_at`

### 作用
把：
- report draft
- search citation
- external URL
- log
- screenshot
- approval record

都挂回来。

---

## 10. workflow_run_approvals

表示自动执行里卡住的审批链路。

### 字段建议
- `id`
- `workflow_run_id`
- `workflow_run_step_id`
- `approval_request_id`
- `approval_target_type` (`write_external`, `skill`, `plugin_action`)
- `approval_status` (`pending`, `approved`, `rejected`, `expired`, `cancelled`)
- `requested_at`
- `decided_at`
- `decided_by`
- `decision_reason`

### 作用
自动任务最关键的安全拦截层之一。

---

## 11. workflow_run_links

把 workflow run 和 thread / task / run / SRWV loop 这些对象串起来。

### 字段建议
- `id`
- `workflow_run_id`
- `link_type` (`thread`, `task`, `run`, `srwv_loop`, `artifact`, `approval_request`)
- `linked_object_id`
- `created_at`

### 作用
后面查询“这个 workflow run 影响了什么”就靠它。

---

## 12. 与现有主干怎么关联

workflow 不应该替代现有主干，而应该挂在主干旁边：

- workflow run 可以关联 thread
- workflow run 可以创建 / 推进 task
- workflow run 可以生成 verification run
- workflow run 可以内部启动 SRWV loop

也就是说：

```text
workflow_run
  -> workflow_run_steps
  -> workflow_run_events
  -> workflow_run_artifacts
  -> workflow_run_approvals
  -> links(thread/task/run/srwv_loop)
```

---

## 13. 一句话结论

workflow 表结构最关键的是把：

**定义、版本、触发器、执行实例、步骤、事件、产物、审批、关联对象**

拆开存。  
这样才能既支持用户“说一个流程”，又支持后续定时/自动执行，还能保证历史执行可追、定义演进可控、失败恢复可做。
