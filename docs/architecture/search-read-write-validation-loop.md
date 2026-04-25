# Search / Read / Write / Validation 循环设计

这份文档专门讨论一类非常常见、也非常容易失控的执行模式：

**系统围绕一个目标，先搜索信息，再阅读与抽取，再产出内容，最后校验结果是否可信、是否足够、是否允许继续执行。**

这可以简称为：

```text
Search -> Read -> Write -> Validation
```

但真正设计时，不能把它做成一个无边界的自由循环。  
更合理的方式是把它设计成一个**受约束的执行环**，作为 Runtime / Loop Engine 里的一个标准模式。

---

## 1. 这个循环解决什么问题

很多任务本质上都符合这个模式：
- 先搜索最新信息
- 读多份来源
- 形成一版结果
- 校验结果够不够好
- 不够再继续搜 / 读 / 修正

比如：
- 搜索最近的事故消息并总结现状
- 搜索某功能的最新讨论后更新文档
- 搜索 Jenkins / 邮件 / Confluence 的最新状态后给出结论
- 搜索网页上的最新消息，再把结论写入 thread / report

所以它不是一个“特殊 case”，而是应该被抽象成系统内可复用的 loop pattern。

---

## 2. 先说结论：它应该是一个受约束 loop

推荐模式：

```text
goal
-> search
-> read
-> write
-> validate
-> decide(next / retry / stop / handoff)
```

但更精确地说，它其实不是机械的固定四步，而是：

```text
understand goal
-> identify missing info
-> search if needed
-> read and structure evidence
-> write draft / local result / external action
-> validate
-> decide continue / retry / stop / handoff
```

核心点在于：

- **Search / Read 是按需触发的**
- **Validation 是每轮必做的**
- **Write 必须区分“写草稿 / 写本地 / 写外部”**

---

## 3. 四个步骤分别负责什么

### 3.1 Search：找来源，不下结论

Search 的职责不是回答问题，而是找材料。

输入通常包括：
- 当前目标 `goal`
- 当前已知上下文 `known_context`
- 缺失信息列表 `missing_info`
- 时间约束（是否必须是最新）
- 来源偏好（本地优先 / 外部优先 / 某系统优先）

输出应该是候选来源，而不是最终答案。

示意结构：

```json
{
  "search_results": [
    {
      "source_type": "web",
      "source_id": "https://example.com/post/123",
      "title": "Latest incident update",
      "snippet": "...",
      "timestamp": "2026-04-25T08:30:00+08:00",
      "confidence": 0.82,
      "why_selected": "Contains latest update related to current goal"
    }
  ]
}
```

Search 最重要的规则：
- 只负责发现来源
- 记录为什么选这个来源
- 不把来源和结论混在一起

---

### 3.2 Read：把来源读成结构化事实

Read 负责把 Search 找回来的内容转成可供后续决策使用的结构。

它不应该把整页网页、整封邮件、整篇文档原样喂给后面步骤。  
更好的做法是统一抽取成：
- source
- excerpt
- facts
- timestamp
- confidence
- conflicts / uncertainties

例如：

```json
{
  "source": "https://example.com/post/123",
  "excerpt": "Deployment has been delayed due to database migration issues.",
  "facts": [
    "Deployment delayed",
    "Cause is database migration",
    "Update time is 08:30"
  ],
  "timestamp": "2026-04-25T08:30:00+08:00",
  "confidence": 0.88,
  "uncertainties": []
}
```

Read 阶段的目标是：
**把原始内容压缩成可比较、可引用、可验证的事实。**

---

### 3.3 Write：产出草稿、本地结果或外部动作

Write 这个词很容易误导。  
它不只是“写文档”，而是“把 loop 当前阶段的产出真正落下来”。

建议至少区分三类写操作：

#### a. write_draft
写草稿、临时答案、候选结论、工作内存。  
这是低风险写。

#### b. write_local
写入本地系统对象，比如：
- thread message
- summary
- task note
- report 草稿
- artifact metadata

#### c. write_external
写到外部系统，比如：
- 回复邮件
- 更新 Confluence
- 写回 Jira
- 触发发布

这三类写操作风险完全不同。  
设计上一定要分层，不然 loop 很容易从“写一个内部总结”直接滑到“发了一封外部邮件”。

---

### 3.4 Validation：给 loop 踩刹车

Validation 是整个循环最关键的一步。

没有 validation，loop 就很容易：
- 用过时信息写结论
- 用单一来源下判断
- 忽略来源冲突
- 未审批就执行外部写操作
- 结果看起来完整，其实并不满足目标

Validation 至少要检查：
- 信息来源是否足够
- 是否需要最新消息但实际用的是旧消息
- 多个来源之间是否冲突
- 输出是否有依据 / citation
- 输出是否覆盖目标要求
- 当前写操作是否超出权限
- 外部写是否已经审批

输出示例：

```json
{
  "pass": false,
  "issues": [
    "Missing latest source",
    "Two sources conflict on deployment status",
    "External write requires approval"
  ],
  "next_action": "search"
}
```

Validation 的本质是：
**不是证明系统多聪明，而是阻止系统在证据不足时继续装作确定。**

---

## 4. Decide 阶段：只允许少数几种决策

Validation 结束后，loop 不能无限自由发挥。  
建议只允许输出几种有限决策：

- `continue`
- `retry_current_step`
- `go_to_search`
- `go_to_read`
- `handoff_to_human`
- `needs_approval`
- `stop`

这能大幅降低 loop 失控的概率。

---

## 5. 更合理的状态机

这个 loop 的状态机我会设计成：

```text
idle
-> planning
-> searching
-> reading
-> writing
-> validating
-> completed

异常分支：
-> blocked
-> needs_approval
-> needs_more_context
-> failed
```

### 5.1 状态含义
- `idle`：尚未开始
- `planning`：识别目标与缺失信息
- `searching`：查找来源
- `reading`：抽取事实
- `writing`：产出内容或动作
- `validating`：验证结果
- `completed`：目标已满足
- `blocked`：缺权限、缺来源、缺能力
- `needs_approval`：待人工批准
- `needs_more_context`：上下文不足
- `failed`：达到失败条件

这样后面无论是 UI 展示、日志记录还是恢复执行，都会清楚很多。

---

## 6. Search Provider 抽象

你提到“search 支持开浏览器 search 最新的消息”，这个很好，应该正好挂在 Search 阶段。

建议把 search 抽象成 provider：

- `local_search`
- `knowledge_search`
- `web_search`
- `browser_live_search`

### 6.1 local_search
查本地：
- thread
- summary
- memory
- artifacts
- 本地文档

### 6.2 knowledge_search
查结构化知识库：
- docs
- confluence sync
- incident records
- release notes

### 6.3 web_search
查公开网页和搜索引擎结果。  
适合快速发现候选来源。

### 6.4 browser_live_search
查“必须打开网页才能拿到”的最新信息，比如：
- 登录后页面
- 实时动态
- 需要交互才能展开的消息列表
- 最新 dashboard / internal portal 信息

browser_live_search 的职责是：
- 打开页面
- 导航
- 搜索关键词
- 抓取页面上的最新内容
- 返回候选来源与摘要

**它仍然只是 Search，不负责最终判断。**

---

## 7. 为什么 browser_live_search 不该直接给最终答案

因为浏览器拿回来的通常是：
- 最新但嘈杂
- 页面级而不是事实级
- 可能需要二次解释
- 可能包含暂时状态或未确认状态

所以正确方式是：

```text
browser_live_search
-> read / extract facts
-> validate freshness and conflicts
-> then write conclusion
```

不要把“能打开浏览器看到页面”误当成“已经完成验证”。

---

## 8. 接口结构建议

### 8.1 Loop Input

```json
{
  "goal": "Find latest deployment status and write summary",
  "context": {
    "threadId": "thread_001",
    "taskId": "task_001"
  },
  "constraints": {
    "freshnessRequired": true,
    "externalWriteAllowed": false
  }
}
```

### 8.2 Search Output

```json
{
  "candidates": [
    {
      "provider": "browser_live_search",
      "source": "https://internal.example.com/deployments",
      "title": "Deployment dashboard",
      "snippet": "Last updated 08:32",
      "timestamp": "2026-04-25T08:32:00+08:00"
    }
  ]
}
```

### 8.3 Read Output

```json
{
  "documents": [
    {
      "source": "https://internal.example.com/deployments",
      "facts": [
        "Current deployment is delayed",
        "Blocking issue is migration timeout"
      ],
      "confidence": 0.86
    }
  ]
}
```

### 8.4 Write Output

```json
{
  "write_type": "draft",
  "target": "thread_summary",
  "content": "Deployment is currently delayed due to migration timeout.",
  "citations": [
    "https://internal.example.com/deployments"
  ]
}
```

### 8.5 Validation Output

```json
{
  "pass": true,
  "issues": [],
  "decision": "stop"
}
```

---

## 9. 这个 loop 应该怎么嵌进整体运行时

它不应该独立漂在系统外面，而应该作为 Loop Engine 里的一个标准执行模式。

关系可以理解成：

```text
Thread Runtime
  -> Loop Engine
    -> SRWV Pattern
      -> Search Providers
      -> Read Extractor
      -> Write Targets
      -> Validation Policy
```

也就是说，SRWV 不是另一个系统，而是 Runtime 里的一个“标准战术动作”。

---

## 10. 风险控制点

这个 loop 最容易出问题的地方有三个：

### 10.1 Search 过度扩张
什么都搜，越搜越散。  
解决办法：
- 限制每轮搜索次数
- 明确 freshness requirement
- 明确 source priority

### 10.2 Write 越权
本来只是写内部草稿，结果直接回了外部邮件。  
解决办法：
- 把 write 分 draft/local/external
- external write 必须审批

### 10.3 Validation 形同虚设
只是走过场。  
解决办法：
- validation policy 可配置
- 必须输出 issues / evidence / decision
- fail 时不能直接继续写

---

## 11. 一句话结论

Search / Read / Write / Validation 这个循环最稳的设计方式是：

**Search 负责找来源，Read 负责抽事实，Write 负责分层落结果，Validation 负责给整个 loop 踩刹车。**

而“支持开浏览器搜索最新消息”就挂在 Search provider 里，作为 `browser_live_search` 能力存在。  
它负责发现最新来源，但不直接替代 Read 和 Validation。
