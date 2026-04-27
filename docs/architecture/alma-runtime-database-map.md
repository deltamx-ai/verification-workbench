# Alma 数据库关系图（基于本地 SQLite 逆向复盘）

这份文档专门把 Alma 本地 SQLite 的核心表族、关系方向和运行语义单独收出来，作为 `alma-runtime-reverse-engineering.md` 和 `alma-runtime-architecture-diagram.md` 的补充。

目标不是穷尽所有表，而是把目前已经确认、并且对理解 Alma 内核最关键的数据库结构画清楚。

---

## 1. 核心数据库文件

已确认 Alma 的主本地数据库文件为：

```text
~/.config/alma/chat_threads.db
```

它不是只存聊天记录，而是把以下几类数据都放在同一个 SQLite 核心库里：

- thread / message 图
- 全文索引
- usage 统计
- 记忆与 embedding
- provider / settings / plugins / MCP
- prompt app
- 多代理 mission / run / sprint 编排

---

## 2. 总体分域图

```text
Conversation Graph Domain
  chat_threads
  chat_messages
  messages_fts (+ FTS附属表)

Projection / Metrics Domain
  usage_records
  thread_diff_stats_cache

Memory Domain
  memories
  memory_embeddings (+ vec附属表)
  memory_metadata

Capability / Platform Domain
  providers
  app_settings
  plugins
  plugin_permissions
  skills
  mcp_servers
  mcp_oauth_tokens
  prompt_apps
  prompt_app_executions
  workspaces
  thread_labels

Multi-Agent Orchestration Domain
  agent_missions
  agent_runs
  agent_handoffs
  mission_sprints
  sprint_contracts
  sprint_evaluations
```

---

## 3. 会话图主存

### 3.1 `chat_threads`
thread 主表。

它承载的是 thread 级上下文壳，而不是普通聊天标题行。

已确认字段包含：
- `id`
- `title`
- `model`
- `created_at`
- `updated_at`
- `enable_artifacts`
- `workspace_id`
- `artifact_workspace_id`
- `skill_ids`
- `tools_compact_view`
- `parent_thread_id`
- 以及运行时已确认存在的 prompt app / reasoning / tools 对应字段

### 3.2 `chat_messages`
message node 主表。

已确认字段：
- `id`
- `thread_id`
- `parent_id`
- `slot_id`
- `depth`
- `message`
- `timestamp`
- `metadata`
- `created_at`
- `updated_at`
- `parent_tool_call_id`

这是 Alma 最重要的主表之一。

#### 语义
- `parent_id`：树边
- `slot_id`：版本槽位 / optimistic lane / 覆盖锚点
- `depth`：树深度
- `message`：完整 message JSON
- `metadata`：治理 / 遥测 / 记忆 / subagent glue
- `parent_tool_call_id`：工具调用关系锚点

### 3.3 会话图关系图

```text
chat_threads (1)
  └── chat_messages (N)
        ├── parent_id -> chat_messages.id
        ├── slot_id   -> same logical lane / version slot
        └── parent_tool_call_id -> tool call anchor in message parts
```

也就是：

```text
thread
  -> message graph
       -> tree edge by parent_id
       -> merge lane by slot_id
```

---

## 4. 搜索与索引层

### 4.1 `messages_fts`
Alma 使用 FTS5：

```sql
USING fts5(
  message_id UNINDEXED,
  thread_id UNINDEXED,
  content
)
```

说明：
- 真正参与全文检索的是 `content`
- `message_id` / `thread_id` 是定位字段
- `content` 是从 `chat_messages.message.parts` 抽平得到的搜索表示

### 4.2 FTS 关系图

```text
chat_messages.message(parts JSON)
  -> extract searchable plain content
  -> messages_fts(message_id, thread_id, content)
```

### 4.3 相关索引
`chat_messages` 已确认索引：
- `idx_chat_messages_id`
- `idx_messages_depth`
- `idx_messages_slot_id`
- `idx_messages_parent_id`
- `idx_messages_timestamp`
- `idx_messages_thread_id`
- `idx_messages_version_info`

其中：

```sql
CREATE INDEX idx_messages_version_info
ON chat_messages(thread_id, timestamp, id, slot_id, created_at)
```

这说明 DB 层明确为以下能力做了优化：
- thread 内顺序遍历
- slot 合并
- 树结构读取
- 时间序查找

---

## 5. usage 与统计投影

### 5.1 `usage_records`
usage 的正式结构化表。

已确认字段：
- `id`
- `message_id`
- `thread_id`
- `model`
- `provider_id`
- `date`
- `input_tokens`
- `output_tokens`
- `cached_input_tokens`
- `cache_write_input_tokens`
- `reasoning_tokens`
- `total_tokens`
- `timestamp`
- `created_at`

外键：
- `message_id -> chat_messages.id`
- `thread_id -> chat_threads.id`

### 5.2 `thread_diff_stats_cache`
thread diff 聚合缓存表。

已确认字段：
- `id`
- `thread_updated_at`
- `additions`
- `deletions`
- `files_changed`
- `created_at`
- `updated_at`

### 5.3 投影关系图

```text
chat_messages + subagent messages
  -> usage metadata / model/provider info
  -> usage_records

chat_messages + subagent messages + tool parts
  -> diff calculation
  -> thread_diff_stats_cache
```

这说明 usage/diff 都不是主存，而是面向统计与 UI 的结构化投影层。

---

## 6. 记忆与向量层

### 6.1 `memories`
结构化记忆主表。

已确认字段：
- `id`
- `content`
- `metadata`
- `thread_id`
- `message_id`
- `created_at`
- `updated_at`
- `user_id`

说明 memory 是可以挂到：
- thread
- message
- user

### 6.2 `memory_embeddings`
向量索引表。

从运行时和 Python 直连错误可确认它依赖 SQLite 向量扩展（`vec0` 相关）。

### 6.3 关系图

```text
memories
  ├── thread_id  -> chat_threads.id
  ├── message_id -> chat_messages.id
  └── user_id    -> logical user identity

memory_embeddings
  └── memory_id -> memories.id
```

也就是：

```text
structured memory row
  + semantic vector index
```

---

## 7. provider / settings / platform 能力层

### 已确认表
- `providers`
- `app_settings`
- `plugins`
- `plugin_permissions`
- `skills`
- `mcp_servers`
- `mcp_oauth_tokens`
- `prompt_apps`
- `prompt_app_executions`
- `workspaces`
- `thread_labels`

### 关系重点

#### `providers`
已从实现层确认字段方向：
- `id`
- `name`
- `type`
- `apiKey`
- `models`
- `availableModels`
- `baseURL`
- `enabled`
- `createdAt`
- `updatedAt`

#### `plugins` / `plugin_permissions`
- 插件清单与启用状态
- 权限粒度
- granted/denied/pending 状态

#### `mcp_servers` / `mcp_oauth_tokens`
- MCP server 配置
- OAuth token 持久化
- refresh / expires / error 状态

#### `prompt_apps`
- Prompt App 已是正式对象，而不是 UI 临时态

#### `workspaces`
- thread 和 artifact workspace 都会引用它

---

## 8. 多代理编排数据库域

这部分是 Alma 区别于普通 chat app 的关键。

### 8.1 `agent_missions`
mission 总体目标与生命周期。

### 8.2 `agent_runs`
单个 agent 执行实例。

已确认字段：
- `id`
- `mission_id`
- `task_id`（UNIQUE）
- `agent_id`
- `agent_name`
- `parent_run_id`
- `spawned_by_handoff_id`
- `execution_mode`
- `model`
- `status`
- `input_summary`
- `output_summary`
- `created_at`
- `updated_at`
- `harness_role`
- `sprint_id`
- `attempt_number`

### 8.3 `agent_handoffs`
agent 之间交接。

已确认字段：
- `id`
- `mission_id`
- `from_run_id`
- `to_agent_id`
- `to_agent_name`
- `to_run_id`
- `status`
- `packet`
- `result_summary`
- `created_at`
- `updated_at`

### 8.4 `mission_sprints`
已确认 SQL：
- `mission_id`
- `sprint_number`
- `title`
- `description`
- `agent_id`
- `status`
- `created_at`
- `updated_at`

### 8.5 `sprint_contracts`
- `sprint_id`
- `version`
- `criteria`
- `negotiation_log`
- `status`

### 8.6 `sprint_evaluations`
- `sprint_id`
- `contract_id`
- `attempt_number`
- `generator_run_id`
- `evaluator_run_id`
- `grades`
- `overall_passed`
- `feedback_summary`

### 8.7 编排关系图

```text
agent_missions (1)
  ├── agent_runs (N)
  ├── agent_handoffs (N)
  └── mission_sprints (N)
         ├── sprint_contracts (N)
         └── sprint_evaluations (N)
```

这就是 Alma 的 mission/sprint/harness persistence layer。

---

## 9. 数据流视角总图

```text
User/CLI/UI input
  -> chat_threads / chat_messages
      -> messages_fts (search projection)
      -> usage_records (usage projection)
      -> thread_diff_stats_cache (diff projection)
      -> memories / memory_embeddings (memory extraction)
      -> fileWrites (runtime projection)
      -> subagent grouped view (runtime projection)

Complex execution
  -> agent_missions
      -> agent_runs
      -> agent_handoffs
      -> mission_sprints
          -> sprint_contracts
          -> sprint_evaluations
```

---

## 10. 最终一句话

**Alma 的本地 SQLite 不是“聊天记录库”，而是一个 agent workbench kernel database：`chat_threads + chat_messages` 作为消息图主存，`messages_fts`/`usage_records`/`thread_diff_stats_cache` 作为搜索与投影层，`memories + memory_embeddings` 作为长期记忆层，再由 `agent_missions / agent_runs / sprint_*` 承载多代理编排。**
