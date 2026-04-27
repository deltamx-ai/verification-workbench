# Alma 运行时与源码实现复盘

这份文档不是基于仓库源码直接开发，而是基于一台已安装运行中的 Alma 桌面应用做**逆向式架构复盘**。目标是把已经确认到的实现事实收束成一份对后续设计有参考价值的文档。

重点回答：

- Alma 真实运行时是怎么组织的
- 核心数据模型在本地是怎么落的
- thread / message / task / subagent / memory / mission 之间如何协作
- 哪些结论已经通过运行文件、SQLite、API、解包产物确认
- 这些实现对 verification workbench 有什么启发

---

## 1. 结论先行

Alma 的真实形态不是“聊天应用 + 几个工具调用”。

更接近：

**Electron 宿主中的本地 agent runtime platform**。

它的核心由四部分组成：

1. **SQLite 本地核心库**
   - 持久化 thread、message、FTS、usage、memory、provider、plugin、MCP、mission/sprint 等结构化对象

2. **message graph 事实源**
   - 以 thread 内消息树为核心工作结构
   - 通过 `parent_id`、`slot_id`、`depth`、`parent_tool_call_id` 表达树形关系、版本槽位和工具挂接

3. **投影与治理层**
   - 从消息图派生 `fileWrites`、`diffStats`、`contextUsage`、subagent grouped view
   - 通过 snapshot / rollback / compaction / approval 控制副作用和上下文预算

4. **Electron + 本地 API + IPC 宿主层**
   - 桌面 UI、CLI、thread API、tool/session/browser/OAuth/snapshot/approval 共用同一运行时内核

一句话：

**Alma 是一个以 message graph 为统一事实源、以 SQLite 为本地结构化核心、以投影视图和宿主能力支撑可治理 agent 工作流的桌面工作台。**

---

## 2. 实际运行形态

### 2.1 桌面壳
运行中的桌面进程是 Electron 形态，表现为：

- 主程序 `/opt/Alma/alma`
- Chromium / zygote / crashpad 相关子进程
- `app.asar` 打包产物
- `out/main`、`out/preload`、`out/renderer` 三段结构

可以判断为标准 Electron 分层：

```text
Renderer
Preload
Main process
```

### 2.2 CLI 入口
`alma` 命令本身只是一个 Bun 启动器：

```bash
/opt/Alma/resources/bun/bun /opt/Alma/resources/cli/alma
```

说明 CLI 不是外挂，而是共享同一 JS/TS runtime 生态的正式入口层。

### 2.3 本地 API
桌面主进程自身监听：

- `127.0.0.1:23001`

说明本地 HTTP API 不是外部 daemon，而是 Electron 主进程内嵌服务。

因此整体运行形态是：

```text
Desktop UI / CLI / 其他入口
  -> Electron Main
    -> Local API (:23001)
    -> IPC privileged handlers
    -> SQLite + runtime services
```

---

## 3. SQLite 本地数据库事实

实际数据库文件：

- `~/.config/alma/chat_threads.db`

这不是一个只存聊天记录的小库，而是 Alma 本地核心结构化数据库。

### 3.1 已确认的重要表

#### 会话 / 消息域
- `chat_threads`
- `chat_messages`
- `messages_fts`
- `messages_fts_content`
- `messages_fts_data`
- `messages_fts_idx`
- `messages_fts_docsize`
- `messages_fts_config`

#### 计量 / 投影视图域
- `usage_records`
- `thread_diff_stats_cache`

#### 记忆域
- `memories`
- `memory_embeddings`
- `memory_embeddings_chunks`
- `memory_embeddings_info`
- `memory_embeddings_rowids`
- `memory_embeddings_vector_chunks00`
- `memory_metadata`

#### 能力与平台域
- `providers`
- `provider_models_cache`
- `model_capabilities_cache`
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

#### 多代理 / 编排域
- `agent_missions`
- `agent_handoffs`
- `agent_runs`
- `mission_sprints`
- `sprint_contracts`
- `sprint_evaluations`

这说明 Alma 不是“文件优先到没有数据库”，而是：

**SQLite 本地核心 + 文件侧日志与记忆辅助 + 运行时投影**。

---

## 4. thread / message 的真实落库模型

### 4.1 `chat_threads`
`chat_threads` 是 thread 的主表。

已确认字段包含：

- `id`
- `title`
- `model`
- `is_generating`（API/运行时已确认存在对应状态）
- `created_at`
- `updated_at`
- `enable_artifacts`
- `workspace_id`
- `artifact_workspace_id`
- `skill_ids`
- `tools_compact_view`
- `parent_thread_id`
- 以及前面 API 已确认的：`reasoningEffort`、`promptAppId`、`tools` 等对应存储

同时有这些索引：

- `idx_threads_workspace_id`
- `idx_threads_artifact_workspace_id`
- `idx_threads_prompt_app_id`
- `idx_threads_updated_at`

这说明 thread 不是单纯聊天会话，而是：

```text
conversation shell
+ workspace binding
+ artifact workspace binding
+ prompt app binding
+ model/skill/tool context
+ branching lineage
```

### 4.2 `chat_messages`
`chat_messages` 是 message node 的主表。

已确认字段：

- `id TEXT PRIMARY KEY`
- `thread_id TEXT NOT NULL`
- `parent_id TEXT`
- `slot_id TEXT`
- `depth INTEGER NOT NULL DEFAULT 0`
- `message TEXT NOT NULL`
- `timestamp TEXT NOT NULL`
- `metadata TEXT DEFAULT '{}'`
- `created_at TEXT NOT NULL`
- `updated_at TEXT NOT NULL`
- `parent_tool_call_id TEXT`

这说明一条消息在数据库里是：

```text
结构字段 + message JSON + metadata JSON
```

而不是把 role/content/tool/memory 等拆成大量子表。

### 4.3 `messages_fts`
`messages_fts` 使用 FTS5：

```sql
USING fts5(
  message_id UNINDEXED,
  thread_id UNINDEXED,
  content
)
```

说明：
- 可搜索的只有抽平后的 `content`
- `message_id` / `thread_id` 只是随行定位字段
- 搜索后再回主表/主图恢复上下文

### 4.4 `chat_messages` 索引策略
已确认索引：

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

这说明 message graph 在数据库层是用：

- `parent_id` 表示树结构
- `slot_id` 表示版本槽位 / optimistic merge lane
- `timestamp/created_at` 表示 thread-local 顺序

它不是图数据库，而是：

**关系表 + 树键 + 槽位键 + FTS + 索引优化**。

---

## 5. message graph 是统一事实源

目前已经确认很多工作台能力不是单独一张强业务表，而是从消息图投影出来。

### 5.1 file writes 投影
`/api/threads/:id/file-writes` 的实现会遍历消息里的 tool parts：

- `tool-Write`
- `tool-Edit`

并投影出：

- `type`
- `filePath`
- `content`
- `bytesWritten`
- `oldString`
- `newString`
- `replaceAll`
- `toolCallId`
- `messageId`
- `timestamp`

也就是：

```text
chat_messages.message.parts
  -> derive fileWrites ledger
```

### 5.2 diff stats 投影
`/api/threads/:id/diff-stats` 的实现会：

- 取 thread 的主消息
- 再取 `subagent messages`
- 合并后遍历 message parts
- 统计 additions / deletions / filesChanged

结果还会缓存到：

- `thread_diff_stats_cache`

表定义：

```sql
CREATE TABLE thread_diff_stats_cache (
  id TEXT PRIMARY KEY,
  thread_updated_at TEXT NOT NULL,
  additions INTEGER NOT NULL DEFAULT 0,
  deletions INTEGER NOT NULL DEFAULT 0,
  files_changed INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
)
```

这说明 diff stats 是：

```text
message graph -> projection -> cached projection table
```

### 5.3 context usage 投影
`/api/threads/:id/context-usage?model=...` 会：

- 遍历消息
- 识别 compaction indicator
- 把 `compactionSummary` 转回 `<context_from_earlier_conversation>` block
- 从最近有效 assistant message 的 usage 估值
- 计算 `totalTokens / contextWindow / usagePercent`

说明 context usage 也不是独立存储，而是：

```text
message graph + metadata + compaction markers
  -> context budget projection
```

---

## 6. message 的真实内容模型

`chat_messages.message` 存的是 JSON。

live 样本已经确认：

### user message
```json
{
  "id": "user-...",
  "role": "user",
  "parts": [
    { "type": "text", "text": "..." }
  ]
}
```

### assistant message
已经确认可以包含：

- `step-start`
- `reasoning`
- `text`
- `file`
- artifact 相关 part
- tool / dynamic-tool / Task 相关 part

所以 message content 不是纯字符串，而是：

```text
role + typed parts[]
```

这也是为什么：
- 内容展示
- reasoning 展示
- file write 投影
- tool call 追踪
- artifact 提取

都能统一从一条 message 里恢复。

---

## 7. message metadata 的真实语义

### 7.1 user message metadata
已确认真实样本：

```json
{ "snapshotId": "..." }
```

说明用户发起一轮请求时，运行时会在该消息上挂 snapshot 锚点，后续 rollback 会以它为边界。

### 7.2 assistant message metadata
已确认字段：

- `autoSelectedSkills`
- `autoSelectedTools`
- `memoryExtracted`
- `memoryExtractedAt`
- `reasoningDuration`
- `turnEndReason`
- `usage`
- `suggestions`

`usage` 的结构已确认：

- `inputTokens`
- `outputTokens`
- `totalTokens`
- `reasoningTokens`
- `cachedInputTokens`
- `timeToFirstTokenMs`
- `streamingDurationMs`
- `tokensPerSecond`

也就是说 assistant message metadata 本质上就是一轮 runtime execution telemetry。

### 7.3 治理类 metadata
已确认或从 renderer / main 代码确认存在：

- `snapshotId`
- `usedMemories`
- `createdMemories`
- `deletedMemories`
- `isCompactionIndicator`
- `isCompactionSummary`
- `compactionSummary`
- `subagentTaskId`
- `subagentParentMessageId`

所以 message node 真正的语义是：

```text
内容节点
+ 树节点
+ 执行节点
+ 治理锚点
+ 记忆影响记录
```

---

## 8. rollback / compact / branch 的真实实现语义

### 8.1 rollback
main 里已确认存在：

- `rollbackToMessage(e, t)`

结合 API 和 renderer 可知：
- rollback 入口是 message 级
- body 带 `mode`
- renderer 默认 `rollbackMode = "files-only"`
- 是否可 rollback 与 `message.metadata.snapshotId` 直接相关

所以 rollback 语义是：

```text
rollbackToMessage(messageId, mode)
```

也就是按 message 节点边界撤销副作用，而不是按整线程粗暴回滚。

### 8.2 compact
已确认：

- `POST /api/threads/:id/compact`
- 返回 `savedTokens`
- message metadata 会出现：
  - `isCompactionIndicator`
  - `isCompactionSummary`
  - `compactionSummary`

说明 compact 的真实流程是：

```text
long thread
  -> compact
  -> create compaction indicator message
  -> store summary into metadata
  -> later context reconstruction injects summary block
```

不是简单删历史。

### 8.3 branch
branch API 已确认会要求：

- `targetMessageId is required`

这说明 branch 不是按 thread 粗分支，而是：

```text
branch(targetMessageId)
```

即以消息节点为锚点分叉。

---

## 9. subagent 是图内节点，不是外挂结果

已确认 live 样本：

```json
{
  "subagentTaskId": "...",
  "subagentParentMessageId": "..."
}
```

子代理消息的 message id 也明确是挂在线程下的特殊节点。

renderer 会：
- 监听 `subagent_message_updated`
- 监听 `subagent_message_completed`
- 只在 `parentMessageId` 落在当前 `activePath` 上时显示
- 以 collapsible 方式展示 grouped subagent runs

这说明：

```text
subagent task execution
  -> final/streaming messages
  -> write back into message graph
  -> UI filters by activePath
```

所以 thread graph 才是最终统一视图。

---

## 10. usage 是双层存储

### 10.1 message metadata 用于当前轮展示
assistant message metadata 里有 `usage`，方便 thread UI 直接展示。

### 10.2 `usage_records` 用于结构化统计
`usage_records` 是正式表，外键到：
- `chat_messages`
- `chat_threads`

并有这些索引：
- `idx_usage_records_thread_id`
- `idx_usage_records_message_id`
- `idx_usage_records_date`
- `idx_usage_records_model_date`
- `idx_usage_records_provider_date`

这说明 usage 既服务当前 UI，也服务统计/分析/历史聚合。

---

## 11. memory 是结构化对象 + 向量索引

`MemoryService` 实际落库逻辑确认：

- 先插入 `memories`
- 再插入 `memory_embeddings`

`memories` 已确认字段：

- `id`
- `content`
- `metadata`
- `thread_id`
- `message_id`
- `user_id`
- `created_at`
- `updated_at`

并且 `memory_embeddings` 依赖 SQLite 向量扩展（当前系统 Python 直读会报 `vec0` 模块缺失）。

所以 memory 体系不是 markdown 备忘，而是：

```text
structured memory row
+ thread/message foreign keys
+ vector embedding index
```

---

## 12. 多代理编排是正式数据库域

### 12.1 `agent_missions`
负责 mission 级目标与整体编排状态。

### 12.2 `agent_runs`
每次 agent 执行实例。

已确认字段：
- `task_id UNIQUE`
- `agent_id`
- `agent_name`
- `parent_run_id`
- `spawned_by_handoff_id`
- `execution_mode`
- `model`
- `status`
- `input_summary`
- `output_summary`
- `harness_role`
- `sprint_id`
- `attempt_number`

### 12.3 `agent_handoffs`
用于 agent 之间的交接。

已确认字段：
- `mission_id`
- `from_run_id`
- `to_agent_id`
- `to_agent_name`
- `to_run_id`
- `status`
- `packet`
- `result_summary`

### 12.4 `mission_sprints`
已确认 SQL：

```sql
CREATE TABLE mission_sprints (
  id TEXT PRIMARY KEY,
  mission_id TEXT NOT NULL REFERENCES agent_missions(id) ON DELETE CASCADE,
  sprint_number INTEGER NOT NULL,
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  agent_id TEXT,
  status TEXT NOT NULL DEFAULT 'pending',
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
)
```

### 12.5 `sprint_contracts` / `sprint_evaluations`
也都已确认存在，用于：
- 验收 criteria
- negotiation log
- evaluation grades
- pass/fail and feedback

因此 multi-agent harness 不是 UI 假象，而是**有正式持久化状态机**。

---

## 13. 已确认的实现原则

到目前为止，可以认为以下原则已被真实实现确认：

1. **message graph 是统一事实源**
2. **FTS / diff / usage / context / subagent 等都围绕消息图派生**
3. **rollback / branch / compact 都是 message 级治理动作**
4. **UI 的 activePath / slotId 是 runtime 核心语义，不是显示小技巧**
5. **多代理编排有独立数据库域，不是 prompt 层模拟**
6. **长期记忆是结构化对象 + 向量索引，不是散文本缓存**

---

## 14. 对 verification workbench 的启发

如果把 Alma 的实现经验投回 verification workbench，最值得借鉴的是：

### 14.1 “统一事实源”比“每个功能单独建模”更稳
Alma 把 message graph 做成统一事实源，然后围绕它投影出：
- 搜索
- diff
- context
- file write
- subagent view

这个思路对验证工作台很有价值。

### 14.2 治理动作最好落到 node / step 级
Alma 的 branch / rollback / compact 都不是 thread 粗粒度，而是 message node 粒度。

验证工作台后续做 run/step/artifact 时，也应该优先考虑 node / step 锚点，而不是只做整任务级治理。

### 14.3 metadata 是极高杠杆的设计位
Alma 没有把所有执行细节都拆成独立表，而是让 message metadata 承担了很多 runtime telemetry / governance glue。

对于 workbench，task/run/step/artifact 的 metadata 设计也应该预留足够弹性。

---

## 15. 最终总结

如果用一句话总结这次逆向复盘的最终结论：

**Alma 的真实源码实现，是以 `chat_threads + chat_messages` 构成的本地 message graph 为核心事实源，用 `messages_fts`、`usage_records`、`thread_diff_stats_cache`、`memories + memory_embeddings` 做检索/统计/记忆投影，再叠加 Electron 宿主能力和 `agent_missions / agent_runs / mission_sprints / sprint_*` 多代理编排域，形成一个可分支、可回滚、可压缩、可搜索、可续跑的本地 agent 工作台内核。**
