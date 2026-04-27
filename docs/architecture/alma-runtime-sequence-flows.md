# Alma 关键运行时序图

这份文档把 Alma 最关键的几条运行链路，用时序方式重新梳理出来。

重点不是 UI 操作，而是底层语义：
- 一轮普通对话怎么落 graph
- 工具写文件怎么投影
- rollback 怎么执行
- compact 怎么重构上下文
- subagent 怎么回写主图

---

## 1. 普通一轮对话

```text
User
  -> Renderer
  -> Electron Main / Local API
  -> create user message node
       metadata.snapshotId = ...
  -> Thread Runtime starts generation
  -> choose model / tools / skills
  -> stream assistant parts
  -> finalize assistant message
       metadata.autoSelectedSkills
       metadata.autoSelectedTools
       metadata.usage
       metadata.reasoningDuration
       metadata.turnEndReason
       metadata.suggestions
  -> save to chat_messages
  -> update messages_fts searchable content
  -> broadcast thread_updated
  -> Renderer merges by slotId / activePath
```

---

## 2. 工具写文件链路

```text
Assistant message generation
  -> tool part emitted (tool-Write / tool-Edit)
  -> message graph stores tool call trace
  -> runtime/API derives fileWrites projection
  -> runtime/API derives diffStats projection
  -> optional thread_diff_stats_cache refresh
  -> Renderer shows file changes / diff summary
```

重点：
- `fileWrites` 不是主存
- `diffStats` 不是主存
- 它们都从 message graph 派生

---

## 3. rollback 链路

```text
User message node exists
  metadata.snapshotId = X

Later user clicks rollback
  -> Renderer sends rollback request
  -> Main/API calls rollbackToMessage(messageId, mode)
  -> snapshot manager loads snapshot boundary X
  -> revert file effects
  -> thread updated
  -> Renderer refreshes graph / file projections
```

注意：
- rollback 粒度是 message node
- 默认 mode 已确认倾向 `files-only`

---

## 4. compact 链路

```text
Thread grows large
  -> compact request
  -> runtime summarizes earlier conversation
  -> create compaction indicator message
       metadata.isCompactionIndicator = true
       metadata.compactionSummary = ...
  -> older messages may be treated as compacted
  -> context usage drops / savedTokens returned
  -> later context rebuild injects summary block instead of full old history
```

这说明 compact 不是删消息，而是：
- 新增一条 summary node
- 以后重建上下文时用 summary 替代老内容

---

## 5. branch 链路

```text
User selects a message node
  -> branch(targetMessageId)
  -> runtime creates new branch/thread lineage
  -> activePath switches to new path
  -> subsequent messages continue from that node
```

重点：
- branch 不是 thread 粗拷贝
- 是以 target message node 为锚点的分叉

---

## 6. subagent 链路

```text
Main assistant decides to delegate
  -> create Task / agent run
  -> subagent execution starts
  -> subagent streaming events emitted
  -> Renderer updates subagentMessages map
  -> subagent completes
  -> final subagent message written back to chat_messages
       metadata.subagentTaskId
       metadata.subagentParentMessageId
  -> Renderer shows grouped subagent result under parent node
```

这说明 subagent 不只是后台日志，而是：
- 有自己的运行流
- 最终仍回写主 message graph

---

## 7. search 链路

```text
Query
  -> messages_fts MATCH
  -> get message_id + thread_id matches
  -> load thread + nearby messages from chat_messages
  -> build snippets
  -> return search results
```

所以搜索不是直接扫 chat_messages JSON，而是：
- 搜 messages_fts
- 回主图补上下文

---

## 8. memory 抽取链路

```text
Assistant turn completes
  -> runtime decides memoryExtracted = true/false
  -> if extracting:
       create memories row
       create memory_embeddings row
       attach memory metadata back to message
  -> future turns can search memories by vector lookup
```

所以 memory 是：
- 主图外的结构化长期层
- 但它的来源和消费都仍然回到 message 节点

---

## 9. 多代理 mission/sprint 链路

```text
Mission created
  -> agent_missions
  -> mission_sprints
  -> sprint_contracts
  -> agent_runs
  -> agent_handoffs between runs
  -> sprint_evaluations
  -> results summarized back into conversation/task context
```

这层不是直接替代 thread graph，而是执行编排侧车。

---

## 10. 最终总结

Alma 的运行时不是“问一句答一句”，而是几条链共同成立：

- 对话链
- 工具副作用链
- rollback 治理链
- compact 上下文链
- subagent 回写链
- memory 持久链
- mission/sprint 编排链

它们的共同落点都是：

**message graph 作为统一工作现场。**
