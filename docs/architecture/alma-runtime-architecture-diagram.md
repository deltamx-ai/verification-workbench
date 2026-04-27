# Alma 架构图（基于运行时与源码实现复盘）

这份文档把前面已经确认的 Alma 实现事实压缩成一份更适合快速浏览的“架构图 + 分层说明”。

它不追求穷尽细节，而是回答：

- Alma 的内核到底分成哪些层
- 哪些表和对象是核心
- 数据怎么流动
- UI、API、SQLite、message graph、agent 编排是怎么接起来的

---

## 1. 总体架构图

```text
┌─────────────────────────────────────────────────────────────┐
│                        Interface Layer                     │
│                                                             │
│  Electron Renderer UI   CLI (alma)   Prompt App Runner     │
│  Discord/other bridges  Chrome relay / browser tools       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Host / Runtime Entry                     │
│                                                             │
│  Electron Main Process                                      │
│  - Local HTTP API (:23001)                                  │
│  - IPC privileged handlers                                  │
│  - snapshot / rollback                                      │
│  - approval dialogs                                         │
│  - browser sessions / OAuth / tool sessions                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Core Runtime Layer                      │
│                                                             │
│  Thread Runtime                                             │
│  - activePath                                               │
│  - branch(targetMessageId)                                  │
│  - compact / context usage                                  │
│                                                             │
│  Message Graph Runtime                                      │
│  - parent_id / slot_id / depth                              │
│  - message JSON + metadata JSON                             │
│                                                             │
│  Execution Runtime                                          │
│  - tools / skills / providers                               │
│  - task resume                                              │
│  - subagent writeback                                       │
│                                                             │
│  Governance Runtime                                         │
│  - snapshotId                                               │
│  - rollbackToMessage(messageId, mode)                       │
│  - compactionSummary / approval                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    SQLite Persistence Core                  │
│                                                             │
│  Conversation Graph Domain                                  │
│  - chat_threads                                             │
│  - chat_messages                                            │
│  - messages_fts                                             │
│                                                             │
│  Projection / Metrics Domain                                │
│  - usage_records                                            │
│  - thread_diff_stats_cache                                  │
│                                                             │
│  Memory Domain                                              │
│  - memories                                                 │
│  - memory_embeddings                                        │
│                                                             │
│  Capability Domain                                          │
│  - providers                                                │
│  - app_settings                                             │
│  - plugins / plugin_permissions                             │
│  - mcp_servers / mcp_oauth_tokens                           │
│  - prompt_apps / prompt_app_executions                      │
│  - workspaces / thread_labels                               │
│                                                             │
│  Multi-Agent Orchestration Domain                           │
│  - agent_missions                                           │
│  - agent_runs                                               │
│  - agent_handoffs                                           │
│  - mission_sprints                                          │
│  - sprint_contracts                                         │
│  - sprint_evaluations                                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Derived Projections                      │
│                                                             │
│  fileWrites                                                 │
│  diffStats                                                  │
│  contextUsage                                               │
│  subagent grouped view                                      │
│  search snippets                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. message graph 是事实源

Alma 的核心不是一张“聊天记录表”，而是一个**消息图**。

### 2.1 主表
- `chat_threads`
- `chat_messages`

### 2.2 `chat_messages` 的关键字段
- `id`
- `thread_id`
- `parent_id`
- `slot_id`
- `depth`
- `message`（JSON）
- `metadata`（JSON）
- `parent_tool_call_id`
- `timestamp`
- `created_at`
- `updated_at`

### 2.3 图语义
- `parent_id`：树结构
- `slot_id`：同一槽位的 optimistic / final / branch merge
- `depth`：层级
- `parent_tool_call_id`：工具调用挂点

所以 Alma 的 message node 是：

```text
MessageNode = 树节点 + 内容载荷 + 运行元数据 + 治理锚点
```

---

## 3. message 的两层表示

### 3.1 完整表示：`chat_messages`
保存完整 JSON：
- `message.parts[]`
- `metadata`

### 3.2 可搜索表示：`messages_fts`
`messages_fts` 使用 FTS5：

```sql
USING fts5(
  message_id UNINDEXED,
  thread_id UNINDEXED,
  content
)
```

也就是说：
- `chat_messages` 保存完整事实
- `messages_fts` 保存从 parts 抽平后的搜索文本

### 3.3 同步方式
当前复盘更倾向于：
- 应用层在 add/update/delete message 时同步维护 FTS
- 没有依赖 SQLite trigger 做自动镜像

---

## 4. message metadata 承担的角色

### 4.1 user message metadata
已确认最核心字段：
- `snapshotId`

表示一轮用户输入的副作用边界锚点。

### 4.2 assistant message metadata
已确认字段：
- `autoSelectedSkills`
- `autoSelectedTools`
- `memoryExtracted`
- `memoryExtractedAt`
- `reasoningDuration`
- `turnEndReason`
- `usage`
- `suggestions`

这里的 `usage` 包括：
- `inputTokens`
- `outputTokens`
- `totalTokens`
- `reasoningTokens`
- `cachedInputTokens`
- `timeToFirstTokenMs`
- `streamingDurationMs`
- `tokensPerSecond`

### 4.3 治理 / 上下文字段
已确认存在：
- `usedMemories`
- `createdMemories`
- `deletedMemories`
- `isCompactionIndicator`
- `isCompactionSummary`
- `compactionSummary`
- `subagentTaskId`
- `subagentParentMessageId`

所以 metadata 是：

```text
runtime telemetry + governance glue + memory linkage + subagent linkage
```

---

## 5. 投影视图怎么从 message graph 推出来

### 5.1 fileWrites
通过遍历 tool parts：
- `tool-Write`
- `tool-Edit`

派生出：
- `filePath`
- `content` / `oldString` / `newString`
- `bytesWritten`
- `replaceAll`
- `toolCallId`
- `messageId`
- `timestamp`

### 5.2 diffStats
遍历：
- 主消息
- subagent 消息

统计：
- additions
- deletions
- filesChanged

并写入缓存表：
- `thread_diff_stats_cache`

### 5.3 contextUsage
从：
- 最近有效 assistant usage
- compaction indicator / compactionSummary
- message graph 上下文

派生：
- `totalTokens`
- `contextWindow`
- `usagePercent`

### 5.4 subagent view
从：
- 子代理消息节点
- `subagentTaskId`
- `subagentParentMessageId`

聚合成 grouped subagent runs。

---

## 6. 治理链路图

### 6.1 rollback
```text
user message created
  -> snapshotId written to metadata
assistant/tool execution
  -> file/tool side effects
user clicks rollback
  -> rollbackToMessage(messageId, mode)
  -> main process executes revert
```

### 6.2 compact
```text
thread too long
  -> compact
  -> create compaction indicator message
  -> write compactionSummary into metadata
  -> later context rebuild injects summary block
```

### 6.3 branch
```text
user chooses a node
  -> branch(targetMessageId)
  -> new thread/path from target node
```

注意：branch 已确认要求 `targetMessageId`，说明它是 message 级分叉，不是 thread 粗分叉。

---

## 7. UI 同步链

Renderer 并不是每次全量刷新，而是维护当前 thread 的本地状态：

- `currentThread`
- `activePathRef`
- `subagentMessages`
- optimistic message / deletion snapshots

### 7.1 关键事件
- `thread_updated`
- `message_deleted`
- `subagent_message_updated`
- `subagent_message_completed`

### 7.2 activePath 的作用
activePath 不只是导航状态，它还是：
- 当前主线过滤器
- subagent 是否显示的判定条件
- 当前执行上下文切片

### 7.3 slotId 的作用
slotId 是：
- optimistic merge lane
- final message 覆盖锚点
- 分支替换锚点

数据库对 `slot_id` 和 `parent_id` 都建了索引，说明它们是运行时核心键。

---

## 8. 多代理编排层

Alma 的 harness / crew 是正式数据库域，不是 prompt 层假象。

### 8.1 主要表
- `agent_missions`
- `agent_runs`
- `agent_handoffs`
- `mission_sprints`
- `sprint_contracts`
- `sprint_evaluations`

### 8.2 语义
```text
mission
  -> many sprints
  -> contracts / evaluations
  -> runs
  -> handoffs between runs/agents
```

### 8.3 作用
把 agent 编排从“即时提示词协作”提升成：
- 可追踪
- 可恢复
- 可评估
- 可交接
- 可分 sprint 管理

---

## 9. 本地平台能力层

SQLite 之外，Alma 还有完整的本地平台能力层：

- `providers`
- `app_settings`
- `plugins`
- `plugin_permissions`
- `mcp_servers`
- `mcp_oauth_tokens`
- `prompt_apps`
- `workspaces`
- `thread_labels`

这说明它不是只围绕 chat 做，而是逐步成长成一个本地 agent platform。

---

## 10. 最终一句话

**Alma 的真实架构是：用 SQLite 中的 `chat_threads + chat_messages` 构成消息图主存，用 `messages_fts` 做全文检索，用 usage/diff/memory/subagent 等投影视图把消息图转成工作台能力，再由 Electron 主进程、本地 API 和多代理编排域把这张图驱动成一个可分支、可压缩、可回滚、可续跑的本地 agent runtime。**
