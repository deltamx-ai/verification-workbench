# Skills 与知识库管理设计

这份文档补充说明：在 verification workbench 这类系统里，如何管理 **Skills**，以及如何管理 **知识库（Knowledge Base）**。

核心结论先说：

- **Skill** 管“系统会做什么”
- **知识库** 管“系统知道什么”
- 两者必须协作，但不能混成一层

如果把这两个东西混在一起，后面会出现两个典型问题：
- 让 skill 背太多知识，版本和行为会失控
- 让知识库承担执行责任，结果很难追踪副作用和权限边界

所以应该明确把它们拆开。

---

## 1. Skill 到底是什么

Skill 不是单纯的一段 prompt，也不是一个随便起名的工具别名。

更合理的定义是：

**Skill = 面向 Agent 的、可管理、可复用、可观测的能力单元**

一个 skill 可以很薄，也可以稍微复杂，但至少应该满足这几个条件：
- 有明确名字
- 有输入输出定义
- 有行为边界
- 有权限声明
- 有版本
- 有执行日志

### 1.1 Skill 的三种常见类型

#### Tool Skill
单步能力，通常直接映射一个动作。
例如：
- `read_confluence_page`
- `trigger_jenkins_build`
- `fetch_recent_mails`

#### Workflow Skill
把多个动作封成一个目标导向能力。
例如：
- `deploy_to_staging_and_wait`
- `summarize_unread_incident_mails`
- `append_verification_result_to_release_note`

#### Reasoning Skill
偏分析、整理、判断，不一定直接产生外部副作用。
例如：
- `extract_risks_from_release_note`
- `summarize_customer_thread`
- `propose_reply_draft`

这三种类型最好显式标出来，因为它们的权限风险和执行方式不一样。

---

## 2. Skill 怎么管理才不会乱

### 2.1 每个 Skill 都要有 Manifest

一个 skill 至少要有自己的 manifest，描述：
- `name`
- `version`
- `kind`
- `description`
- `input_schema`
- `output_schema`
- `required_actions`
- `required_capabilities`
- `risk_level`
- `owner`
- `status`

示例：

```yaml
name: append_verification_result
version: 0.1.0
kind: workflow
risk_level: medium
required_actions:
  - confluence.read_page
  - confluence.update_page
required_capabilities:
  - read
  - write
status: active
```

这一步很重要，因为后面：
- UI 展示 skill 列表
- loop 判断能不能调用
- operator 配权限
- 审计系统看谁在用什么

都依赖它。

### 2.2 Skill 必须有版本

Skill 后面一定会改。

如果没有版本管理，很容易出现这种情况：
- 上周还能用
- 这周 prompt 或逻辑改了
- 结果行为漂移
- 但 nobody knows why

所以建议至少有：
- `skills`：逻辑实体
- `skill_versions`：具体版本
- `skill_aliases`：稳定调用名到版本映射

这样可以做到：
- 默认走最新稳定版
- 某些 flow 固定老版本
- 出问题时快速回滚

### 2.3 Skill 要能启停与灰度

不要让新 skill 一上线就全量用。

建议支持：
- `draft`
- `active`
- `disabled`
- `deprecated`

再进一步可以做：
- 按 workspace 启用
- 按项目启用
- 按 flow 启用
- 按 risk level 控制是否需要人工确认

这对高风险动作特别重要，比如：
- 自动回复邮件
- 自动发版
- 自动回写 Confluence

### 2.4 Skill 必须可观测

每次 skill 执行至少要有：
- skill name / version
- 输入摘要
- 触发人 / 触发线程 / 触发任务
- 调用了哪些 action
- 成功 / 失败
- 耗时
- 失败原因
- 输出摘要

否则你永远不知道：
- skill 到底有没有价值
- 它常失败在哪
- 是 prompt 问题还是外部系统问题

---

## 3. 知识库到底是什么

知识库不是“把文件扔进去”。

更准确地说：

**知识库 = 可被检索、可被引用、可被追溯、可带版本的上下文事实集合**

它服务的目标不是“长期囤积信息”，而是：
- 在合适的时候把对的知识拿出来
- 给 agent / user / workflow 提供决策上下文
- 让历史结论、约束、经验可以复用

---

## 4. 知识库最好分层管理

如果不分层，后面就会把：
- PRD
- 聊天记录
- 运行日志
- 团队偏好
- 事故复盘

全混在一个检索池里，结果检索命中会越来越脏。

比较合理的是至少分四层。

### 4.1 静态文档库

放相对稳定的资料：
- PRD
- 设计文档
- API 文档
- SOP
- Confluence 页面
- 规范与模板

特点：
- 更新频率较低
- 权威性较高
- 很适合做引用依据

### 4.2 运行事实库

放系统执行中产生的事实：
- run 结果
- step_run 结果
- verdict
- logs
- screenshots
- incidents
- deployment records

特点：
- 时效性强
- 非常适合 support / debugging / verification
- 需要强关联 task / run / environment

### 4.3 对话与决策库

放讨论与结论：
- thread summary
- 决策记录
- why
- 风险判断
- 设计演进过程

特点：
- 对“为什么这么做”特别重要
- 不应该和静态文档完全混在一起
- 通常要有 summary 和 source trace

### 4.4 长期记忆库

放偏长期约束和偏好：
- 项目约定
- 团队偏好
- 环境说明
- 常见坑
- 人员协作信息

特点：
- 更接近记忆层
- 更新不频繁
- 但长期很有价值

---

## 5. 知识条目应该长什么样

每个 knowledge item 至少要有这些元数据：
- `id`
- `source_type`
- `source_ref`
- `project_id`
- `workspace_id`
- `title`
- `content`
- `summary`
- `tags`
- `created_at`
- `updated_at`
- `version`
- `confidence`
- `freshness`
- `visibility`
- `status`

如果要做得更稳，还要有：
- `owner`
- `expires_at`
- `supersedes`
- `linked_items`

原因很简单：
知识不是越多越好，关键是要知道：
- 这是谁说的
- 什么时候说的
- 还有效吗
- 被哪条更新覆盖了

---

## 6. 不要只存原文，要做分块和引用

知识库只存整篇文档不够用。

因为真正检索的时候，经常需要的是：
- 某段设计约束
- 某个 release note 的一节
- 某次对话里关于权限模型的那几句话

所以除了 `knowledge_items`，最好还要有：
- `knowledge_chunks`
- `knowledge_chunk_embeddings`
- `citations`

也就是：
- item 是原始文档实体
- chunk 是检索粒度
- citation 是回答时引用的依据

这样后面：
- 检索更准
- 引用更清晰
- 用户更容易信任结果

---

## 7. Skill 和知识库怎么协作

Skill 不该自己偷偷背知识。

更合理的流程是：

```text
goal -> retrieve context -> choose skill -> execute -> produce artifact/result -> write back knowledge if needed
```

也就是：
1. 先从知识库拿上下文
2. 再决定调用哪个 skill
3. skill 执行后产生结果
4. 必要时把结果沉淀回知识库

例如：
- `reply_customer_mail`
  - 先查客户历史、产品规则、事故背景
  - 再生成回复草稿
  - 人工确认后发送
  - 最后把回复与结论回写 thread / knowledge

- `append_verification_result`
  - 先查 Confluence 页面结构和项目约定
  - 再读取本次 run 的 verdict / artifact
  - 最后写回页面，并记录 page diff

### 7.1 明确一个原则

**Skill 不存事实，知识库存事实。**  
**Skill 只消费知识，并在执行后产出新的事实。**

这个边界一旦稳住，系统会清爽很多。

---

## 8. 检索链路怎么设计

一个比较通用的检索链路可以是：

```text
User/Agent Goal
  -> Query Builder
  -> Scope Filter (workspace/project/type/time)
  -> Hybrid Retrieval (keyword + vector + links)
  -> Re-rank
  -> Context Pack
  -> Skill / Agent Loop 消费
```

关键不是单纯搞向量检索，关键是：
- 要先收 scope
- 要先知道当前是哪个项目/线程/run
- 要优先命中相关类型的数据

比如在 verification workbench 里：
- 如果当前在排查 run 失败，就优先 run facts / incident / recent decision
- 如果当前在写设计方案，就优先 architecture / conversation notes / decision records

检索一定要带场景感知，不然会越来越像大海捞针。

---

## 9. 两套表怎么落

### 9.1 Skills 相关表

#### `skills`
记录 skill 的逻辑身份。
- `id`
- `name`
- `kind`
- `owner`
- `status`
- `created_at`
- `updated_at`

#### `skill_versions`
记录每个 skill 的具体版本。
- `id`
- `skill_id`
- `version`
- `manifest_json`
- `prompt_template`
- `execution_config_json`
- `created_at`

#### `skill_bindings`
记录某个环境/项目/flow 绑定哪个版本。
- `id`
- `scope_type`
- `scope_id`
- `skill_id`
- `skill_version_id`
- `enabled`
- `rollout_ratio`

#### `skill_run_logs`
记录 skill 执行情况。
- `id`
- `skill_id`
- `skill_version_id`
- `thread_id`
- `task_id`
- `run_id`
- `input_summary`
- `output_summary`
- `status`
- `latency_ms`
- `error_text`
- `created_at`

### 9.2 知识库相关表

#### `knowledge_items`
原始知识实体。
- `id`
- `project_id`
- `workspace_id`
- `type`
- `source_type`
- `source_ref`
- `title`
- `summary`
- `content_ref`
- `version`
- `confidence`
- `freshness_score`
- `status`
- `created_at`
- `updated_at`

#### `knowledge_chunks`
可检索切片。
- `id`
- `knowledge_item_id`
- `chunk_index`
- `text`
- `token_count`
- `embedding_ref`

#### `knowledge_links`
知识之间的关联。
- `id`
- `from_item_id`
- `to_item_id`
- `link_type`

#### `knowledge_versions`
用于处理文档更新与覆盖。
- `id`
- `knowledge_item_id`
- `version`
- `diff_summary`
- `created_at`

#### `retrieval_sessions`
记录每次检索行为。
- `id`
- `thread_id`
- `task_id`
- `goal`
- `query_text`
- `filters_json`
- `result_item_ids`
- `created_at`

#### `citations`
记录回答或 skill 调用了哪些知识依据。
- `id`
- `session_id`
- `knowledge_item_id`
- `knowledge_chunk_id`
- `used_by_type`
- `used_by_id`

---

## 10. 目录结构建议

```text
src/
  skills/
    registry/
    runtime/
    manifests/
    workflows/
    reasoning/

  knowledge/
    ingestion/
    indexing/
    retrieval/
    ranking/
    citations/

  context/
    query-builder/
    context-pack/
```

如果要落磁盘或本地目录，也可以这样：

```text
knowledge/
  docs/
  runs/
  decisions/
  memory/

skills/
  manifests/
  templates/
  tests/
```

---

## 11. 最小可落地顺序

建议不要一口气做太满，先这样：

### Phase 1
- skills manifest
- skill registry
- skill run log
- knowledge item / chunk 基础表
- keyword retrieval + simple filters

### Phase 2
- knowledge type 分层
- retrieval session / citation
- skill version binding
- context pack 组装

### Phase 3
- vector retrieval
- rerank
- freshness / confidence 策略
- 自动 write-back 规则

### Phase 4
- 灰度发布 skill
- 高风险 skill 审批
- 知识过期与 supersede 管理

---

## 12. 一句话结论

如果要把系统做稳：

- **Skill 系统负责能力管理**
- **知识库系统负责事实与上下文管理**
- **Agent Loop 负责在合适的时候把两者接起来**

也就是：

**skill 是工具箱，知识库是资料室，loop 是调度大脑。**

这三层分开，系统才不会越长越乱。
