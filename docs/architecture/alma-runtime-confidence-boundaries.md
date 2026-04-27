# Alma 复盘中的已证实 / 高概率推断边界

这份文档专门把目前关于 Alma 的结论按置信度分层，避免后续引用时把“已确认事实”和“合理推断”混在一起。

---

## 1. 已证实事实

以下内容已经通过至少一种强证据确认：
- 运行进程/端口
- 解包后的 app 代码片段
- 本地 SQLite schema / 样例行
- live API 返回
- renderer / main 的实现片段

### 1.1 宿主与运行形态
- Alma 是 Electron 桌面应用
- 本地 API 跑在 `127.0.0.1:23001`
- CLI 是 Bun 启动器
- 有 `out/main` / `out/preload` / `out/renderer`

### 1.2 SQLite 核心库
- 主数据库文件是 `~/.config/alma/chat_threads.db`
- 已确认表：`chat_threads`、`chat_messages`、`messages_fts`、`usage_records`、`thread_diff_stats_cache`、`memories`、`memory_embeddings`、`providers`、`plugins`、`mcp_servers`、`prompt_apps`、`agent_missions`、`agent_runs`、`agent_handoffs`、`mission_sprints`、`sprint_contracts`、`sprint_evaluations` 等

### 1.3 message graph 核心字段
- `chat_messages` 确实有：`thread_id`、`parent_id`、`slot_id`、`depth`、`message`、`metadata`、`parent_tool_call_id`
- `messages_fts` 确实是 FTS5，并包含 `message_id`、`thread_id`、`content`

### 1.4 metadata 关键字段
- user message 上确实有 `snapshotId`
- assistant message 上确实有 `autoSelectedSkills`、`autoSelectedTools`、`reasoningDuration`、`usage`、`turnEndReason`、`suggestions`
- 确实存在 compaction / memory / subagent 相关 metadata 字段

### 1.5 rollback / branch / compact
- rollback 的实现入口确实是 `rollbackToMessage(messageId, mode)`
- branch API 确实要求 `targetMessageId`
- compact API 确实返回 `savedTokens`
- compaction 会写 indicator / summary 相关 metadata

### 1.6 subagent
- subagent final message 会写回主消息图
- live 样本确认 `subagentTaskId`、`subagentParentMessageId`
- renderer 确实监听 subagent 更新/完成事件

### 1.7 usage / memory
- `usage_records` 是正式表，并有外键到 `chat_messages` / `chat_threads`
- `memories` 是正式表
- `memory_embeddings` 依赖 SQLite 向量扩展

---

## 2. 高概率推断

以下内容虽然没有每一条都看到完整源码闭环，但已经有较强证据支持，可信度高。

### 2.1 FTS 同步是应用层维护
未抓到 SQLite trigger，且运行时存在大量 add/update/delete message 逻辑，因此高概率是应用层同步更新 `messages_fts`。

### 2.2 `chat_threads` 中还有更多与 API 一一对应的字段
例如 model、prompt app、tools、reasoning 等，虽然部分表结构片段被截断，但从 API 和索引已基本可以推断这些字段存在或有等价存储。

### 2.3 `slot_id` 的核心语义是 optimistic merge lane
数据库索引、renderer merge 逻辑、同槽位覆盖逻辑都支持这一点，虽然没看到一段专门的设计注释，但实现证据已经很强。

### 2.4 message graph 是多数 UI/投影的统一事实源
fileWrites、diffStats、contextUsage、subagent views 都从消息图派生，这一点没有单独的“架构宣言”，但从实现看几乎成立。

---

## 3. 暂未完全证实，但值得保留的推测

### 3.1 thread 相关部分辅助表
目前还没确认是否存在额外的 thread/message 关系辅助表（例如某些中间缓存或审计表），因为主要关注点放在主表和核心投影上。

### 3.2 更细的 tool part schema
已确认存在 `tool-Write`、`tool-Edit`、`tool-Task`、`dynamic-tool` 相关语义，但还没把所有 tool part 的字段完全列尽。

### 3.3 部分 mission/harness 表之间的完整外键图
已确认核心表及多数列，但还没把全部索引、唯一约束和所有跨表边完整抠全。

---

## 4. 如何引用这些结论

建议后续引用时遵守：

### 可以直接当事实说的
- 表名
- 已确认字段
- 已确认 API 行为
- 已确认 runtime 行为

### 最好写成“推断/更倾向于/高概率”的
- FTS 同步的精确实现方式
- `slot_id` 的产品命名语义
- 某些被压缩代码片段未完整展开的字段补全

---

## 5. 最终结论

这次复盘里最值得相信的结论是：

**Alma 的核心确实是“SQLite + message graph + projections + multi-agent orchestration”的本地 agent workbench 内核。**

而这不是高层想象，是已经被运行时、数据库、API、解包代码四层证据交叉支持的。