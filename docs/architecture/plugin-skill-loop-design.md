# 插件 / Skill / Agent Loop 通用设计

这份文档整理一套更通用的可扩展能力架构，用来支持 Jenkins 发布、邮件接收/回复、Confluence 读写等外部系统接入，同时避免把具体平台能力硬编码进核心验证工作台。

---

## 1. 结论先说

更稳的方向不是二选一，而是三层组合：

- **Plugin**：外部系统接入层
- **Skill**：面向 Agent 的可组合能力层
- **Agent Loop**：受约束的通用决策与调度层

简单说：

- Plugin 负责“怎么连外部系统”
- Skill 负责“把一组外部动作封装成可复用能力”
- Agent Loop 负责“下一步该调用哪个能力，以及何时停止、重试、人工接管”

最重要的一点是：
**Loop 不要知道 Jenkins、邮件、Confluence 的实现细节；Loop 只知道它可以调用哪些结构化能力。**

---

## 2. 为什么不能只做 Skills

如果只有 skills，没有统一 loop，系统很容易变成一堆分散工具：
- 会做很多动作
- 但不会稳定串联
- 缺少统一观察、重试、暂停、审计机制
- 每条业务流程都得单独手写控制逻辑

这种方式适合“人工点一下就执行某个动作”，但不适合复杂的自动协作流程。

---

## 3. 为什么也不能只做 Agent Loop

如果只有 loop，没有 plugin / skill 分层，最后会出现两个问题：

第一，所有平台细节都被塞进 loop：
- Jenkins 的构建触发逻辑
- 邮件的收发与线程归并
- Confluence 的版本冲突与写回规则

第二，loop 会越来越像一个大杂烩：
- 很多分支判断
- 很多 provider 特例
- 很多 hardcode 的动作名和参数结构

这样系统前期可能能跑，但很快就会因为外部系统增多而失控。

---

## 4. 推荐的三层结构

### 4.1 Plugin 层：外部系统适配器

Plugin 是 provider adapter。

它负责：
- API / SDK 调用
- 认证方式处理
- 平台返回值适配
- 错误映射
- 限流、重试、分页、版本差异

例如：
- Jenkins Plugin
- Email Plugin
- Confluence Plugin
- GitHub Actions Plugin
- Feishu Plugin

这一层不负责“思考下一步该做什么”，它只负责把某个平台能力变成标准动作。

### 4.2 Skill 层：面向 Agent 的能力封装

Skill 建立在 plugin action 之上。

它负责：
- 组合多个 action
- 补充业务前置校验
- 处理参数模板
- 提供更高阶的能力语义

例如：
- `deploy_to_staging_and_wait`
- `fetch_unread_incident_mails`
- `reply_with_template_after_human_approval`
- `read_confluence_release_note_and_extract_risks`

Plugin 更像“接口适配器”，Skill 更像“能完成一个目标的动作包”。

### 4.3 Agent Loop：通用决策与编排器

Loop 负责：
- 读取上下文
- 判断当前目标
- 选择下一步 skill / action
- 观察执行结果
- 决定继续 / 重试 / 暂停 / 人工接管 / 结束

标准闭环可以非常简单：

```text
plan -> act -> observe -> decide
```

这个 loop 前期不要做得太聪明。

应该先限制成：
- 一轮只执行一个 action
- action 必须来自注册表
- 每轮必须记录 reason
- 高风险动作必须人工确认
- 每轮都产出结构化 observation

这样比“全自动自由发挥”稳很多。

---

## 5. 抽象边界怎么划最合理

推荐边界：

```text
Agent Loop
  -> Skill Registry
    -> Plugin Runtime
      -> Provider Client
```

也就是：

- Loop 不直连 provider client
- Skill 不直接关心底层凭据保存
- Plugin 不负责规划业务流程

每一层都尽量只干自己的事。

---

## 6. 通用 Action 协议

无论底层是什么平台，最好统一成结构化 action 调用。

请求：

```json
{
  "action": "jenkins.trigger_pipeline",
  "input": {
    "job": "deploy-prod",
    "params": {
      "version": "1.2.3"
    }
  },
  "context": {
    "taskId": "task_001",
    "threadId": "thread_001",
    "runId": "run_001"
  },
  "authRef": "cred_jenkins_main"
}
```

响应：

```json
{
  "ok": true,
  "data": {
    "externalRunId": "12345",
    "url": "https://jenkins/job/deploy-prod/12345"
  },
  "artifacts": [
    {
      "type": "link",
      "name": "jenkins-run",
      "uri": "https://jenkins/job/deploy-prod/12345"
    }
  ],
  "error": null
}
```

这样带来的好处是：
- UI 可以统一展示
- 日志可以统一记录
- 重试策略可以统一处理
- 审计模型可以统一落库
- 后面替换 provider 更容易

---

## 7. Manifest 与注册中心

每个 plugin 都应该有自己的 manifest，告诉系统：
- 它是谁
- 它支持哪些 action
- 它需要哪些配置
- 它要什么认证方式
- 它有哪些权限范围

例如：

```json
{
  "name": "jenkins",
  "version": "0.1.0",
  "actions": [
    "trigger_pipeline",
    "get_pipeline_status",
    "stream_logs"
  ],
  "auth": {
    "type": "token"
  },
  "configSchema": {
    "baseUrl": "string"
  }
}
```

Skill 也应该有自己的 registry，声明：
- skill 名字
- 输入 schema
- 依赖哪些 plugin action
- 是否高风险
- 是否需要人工确认

---

## 8. 凭据管理必须独立

认证信息不要散落在插件代码里。

应该统一做 credential store：
- `authRef` 引用凭据
- 插件运行时按需读取
- 支持轮转
- 支持最小权限
- 支持不同环境隔离

凭据类型可以包括：
- API token
- OAuth token
- SMTP / IMAP 用户名密码
- webhook secret
- cookie / session

这一步很 boring，但特别关键。

---

## 9. 权限模型要按能力收敛

不要一上来就让插件拿全权限。

建议按 capability 授权：
- read
- trigger
- write
- reply
- approve
- admin

例如：
- Confluence 先开放 `read_page`
- 邮件先开放 `fetch_message`
- Jenkins 先开放 `get_build_status`
- 真正的 `deploy`、`reply`、`update_page` 放到高风险区

这样即使 loop 决策错了，风险也不会直接炸穿。

---

## 10. 事件驱动比硬编码流程更通用

很多外部系统接入，最终都应该变成：

```text
event -> rule -> skill/action -> artifact/result -> next decision
```

例如：
- PR merged -> 触发 Jenkins deploy
- 收到 incident 邮件 -> 提取关键信息 -> 建任务
- 验证 run 完成 -> 回写 Confluence 页面

不要把这些流程硬写在某个 provider 里。

provider 只负责执行动作；
规则和决策应放在 loop / orchestration 层。

---

## 11. 三个例子怎么落

### 11.1 Jenkins

Plugin actions：
- `trigger_pipeline`
- `get_pipeline_status`
- `get_pipeline_logs`
- `approve_step`

Skill 示例：
- `deploy_to_env`
- `deploy_and_wait_for_result`
- `collect_failed_build_evidence`

Loop 只判断：
- 现在是否应该触发部署
- 要不要继续等待
- 是否失败后转人工

### 11.2 邮件

Plugin actions：
- `fetch_messages`
- `get_thread`
- `send_mail`
- `reply_mail`
- `download_attachments`

Skill 示例：
- `fetch_unread_incident_mails`
- `summarize_customer_thread`
- `reply_after_human_confirmation`

不建议一开始就放开“自动自由回复”。
先做：
- 拉取
- 解析
- 建议回复
- 人工确认
- 再发出

### 11.3 Confluence

Plugin actions：
- `read_page`
- `search_page`
- `create_page`
- `update_page`
- `append_section`

Skill 示例：
- `read_release_notes`
- `append_verification_result`
- `create_daily_status_page`

写操作建议支持：
- dry-run
- diff preview
- version check

不然多人协作时很容易互相覆盖。

---

## 12. 对 verification workbench 特别重要的两点

### 12.1 所有外部动作都要沉淀为 artifact

例如：
- Jenkins build URL
- Jenkins console log
- 邮件正文与附件
- Confluence page diff
- 外部 API 响应摘要

这些都应该能回流到：
- task
- run
- step_run
- thread

否则外部动作只是“发生了”，但系统里没有证据。

### 12.2 所有外部动作都必须可审计

最少记录：
- 谁触发的
- 何时触发
- 调用哪个 action
- 输入参数摘要
- 关联哪个 task / run / thread
- 成功还是失败
- 外部响应是什么
- 是否重试过

这会直接决定后面可追溯性和调试成本。

---

## 13. 最小可落地版本建议

建议按这个顺序落地：

### Phase 1：定义框架
- plugin manifest
- skill registry
- action envelope
- audit/event 模型
- credential 引用机制

### Phase 2：做 runtime
- 加载 plugin
- 注册 actions
- 调用 provider
- 统一错误处理
- 统一结果包装

### Phase 3：做一个薄 loop
- plan -> act -> observe -> decide
- 单轮单 action
- reason 必填
- 高风险动作人工确认

### Phase 4：先接低风险能力
- Confluence read-only
- Jenkins status-only
- Email fetch-only

### Phase 5：再开放高风险能力
- Jenkins deploy
- Email reply
- Confluence update

这个顺序的好处是：
系统先获得可扩展性和可观测性，再逐步开放副作用能力。

---

## 14. 目录结构建议

```text
src/
  core/
    agent-loop/
      loop.ts
      decision.ts
      observation.ts
    actions/
      action-envelope.ts
      action-result.ts
    audit/
      audit-log.ts
    credentials/
      credential-store.ts

  plugins/
    jenkins/
      manifest.ts
      client.ts
      actions/
        trigger-pipeline.ts
        get-pipeline-status.ts
        get-pipeline-logs.ts
    email/
      manifest.ts
      client.ts
      actions/
        fetch-messages.ts
        reply-mail.ts
    confluence/
      manifest.ts
      client.ts
      actions/
        read-page.ts
        update-page.ts

  skills/
    deploy-to-env/
      index.ts
    fetch-unread-incident-mails/
      index.ts
    append-verification-result/
      index.ts
```

---

## 15. 一句话结论

如果想做成一个长期可扩展的系统，最稳的方式就是：

**底层用 plugin 接外部系统，中层用 skill 封装可复用能力，外层用一个受约束的通用 agent loop 去调度。**

不要只做 skill，也不要只做 loop。  
真正耐用的是这三层分开、但协议统一。
