# Alma Message Graph 模型图

这份文档专门抽离 Alma 最核心的内部对象：**message graph**。

前面的几份文档已经分别讲了：
- 运行时复盘
- 总体架构图
- 数据库关系图

但 Alma 最关键的不是 SQLite，也不是 Electron，而是：

**它把整个工作流压到 thread 内的一张 message graph 上。**

---

## 1. 一句话定义

Alma 的 message graph 不是“聊天记录列表”，而是：

**一个由 thread 承载、由 message node 构成、可分支、可回滚、可压缩、可续跑、可挂接工具和子代理执行结果的工作图。**

---

## 2. 基本结构图

```text
chat_threads
  └── chat_messages[]
        ├── id
        ├── thread_id
        ├── parent_id
        ├── slot_id
        ├── depth
        ├── parent_tool_call_id
        ├── message (JSON)
        └── metadata (JSON)
```

其中：

- `parent_id`：树结构边
- `slot_id`：同一逻辑槽位 / optimistic lane / 覆盖锚点
- `depth`：消息层级
- `parent_tool_call_id`：工具调用挂点

---

## 3. MessageNode 的真实语义

可以把一条消息抽象成：

```text
MessageNode
  = structure
  + content
  + execution telemetry
  + governance anchors
  + memory linkage
  + subagent linkage
```

### 3.1 structure
- `id`
- `thread_id`
- `parent_id`
- `slot_id`
- `depth`
- `parent_tool_call_id`
- `timestamp`

### 3.2 content
来自 `message.parts[]`。

已确认至少有：
- `text`
- `reasoning`
- `step-start`
- `file`
- tool part
- artifact 相关 part

### 3.3 execution telemetry
已确认主要写在 assistant message metadata 中：
- `autoSelectedSkills`
- `autoSelectedTools`
- `reasoningDuration`
- `usage`
- `turnEndReason`
- `suggestions`

### 3.4 governance anchors
- `snapshotId`
- `isCompactionIndicator`
- `isCompactionSummary`
- `compactionSummary`

### 3.5 memory linkage
- `memoryExtracted`
- `memoryExtractedAt`
- `usedMemories`
- `createdMemories`
- `deletedMemories`

### 3.6 subagent linkage
- `subagentTaskId`
- `subagentParentMessageId`

---

## 4. user node 与 assistant node 的差异

### 4.1 user node
最典型 metadata：

```json
{ "snapshotId": "..." }
```

这说明 user node 通常承担：
- 新一轮请求起点
- rollback 边界锚点
- 对应一次新的执行现场

### 4.2 assistant node
最典型 metadata：

```json
{
  "autoSelectedSkills": [...],
  "autoSelectedTools": [...],
  "memoryExtracted": true,
  "reasoningDuration": 1.2,
  "turnEndReason": "stop",
  "usage": {...},
  "suggestions": [...]
}
```

这说明 assistant node 通常承担：
- 一轮执行结果输出
- 运行过程遥测
- 工具/技能选择记录
- 下一步建议

---

## 5. 为什么需要 `slot_id`

Alma 不是纯 append-only 聊天流。

`slot_id` 的实际作用更像：

```text
同一逻辑回复槽位
  -> optimistic message
  -> streaming update
  -> final message
  -> retry/branch 覆盖
```

Renderer 会根据 `slot_id`：
- 找同槽位旧消息
- 替换 optimistic 内容
- 合并 final 内容
- 在需要时剪掉过时 tail

所以：

**`slot_id` 不是辅助字段，而是 UI merge / version lane 的核心键。**

---

## 6. activePath 与 graph 的关系

message graph 是整张树，但 UI 不会同时把整棵树全展开。

runtime 还维护：
- `activePath`

它表示：

```text
当前 thread 正在使用/展示/推进的主线节点路径
```

activePath 的作用包括：
- 控制 UI 当前主线
- 控制 subagent 结果是否应显示
- 控制 branch / retry / rollback 的当前上下文范围

也就是：

```text
message graph = full state
activePath    = current visible execution slice
```

---

## 7. Message Graph 如何承载执行

### 7.1 工具调用
工具调用不会只存在于外部日志里，而会通过：
- tool part
- `parent_tool_call_id`
- metadata / projections

挂回 message graph。

### 7.2 fileWrites 投影
从 message parts 中的：
- `tool-Write`
- `tool-Edit`

推导出 fileWrites ledger。

### 7.3 diffStats 投影
从：
- 主消息图
- subagent 消息图

推导出 diff 统计。

所以 message graph 同时承载：
- conversation
- execution trace
- effect source

---

## 8. Message Graph 如何承载治理

### 8.1 rollback
rollback 是按 message node 做的，不是按整个 thread 粗做。

真实语义是：

```text
rollbackToMessage(messageId, mode)
```

message 通过 `snapshotId` 成为回滚锚点。

### 8.2 compact
compact 会生成：
- `isCompactionIndicator`
- `compactionSummary`

也就是把一段旧历史折成新的 summary node。

### 8.3 branch
branch API 已确认要求：
- `targetMessageId`

所以 branch 语义是：

```text
branch from target message node
```

这不是 thread 粗分叉，而是 node 级分叉。

---

## 9. Message Graph 如何承载 subagent

subagent 结果不是单独页外挂，而是：

```text
parent message
  -> subagent message node
       metadata.subagentTaskId
       metadata.subagentParentMessageId
```

UI 再按 activePath 过滤，只展示当前主线相关的子代理结果。

这说明：

**thread graph 才是所有执行结果最终汇聚的统一视图。**

---

## 10. 最终总结

如果只保留一个对 Alma 的理解，最值得保留的就是：

**Alma 的核心不是 thread 表，也不是任务表，而是 message graph。**

thread 只是容器；
message node 才是真正承载：
- 内容
- 执行
- 治理
- 记忆
- 工具
- 子代理
- 投影视图来源

的最小核心单元。
