# 错误模型与治理设计

这份文档专门补全一个最容易被低估、但真正开发时一定会爆炸的部分：

**错误模型、权限治理、审批治理、审计与恢复，到底怎么统一设计。**

如果这块不提前收口，系统前面看起来再优雅，真正接上自动执行、外部写、浏览器搜索、Jenkins、邮件后，都会变得非常脆弱。

---

## 1. 错误不能只是一段字符串

开发时最常见的问题就是：
- 返回一个报错文本
- UI 自己猜怎么展示
- Runtime 自己猜要不要重试
- 审批层也不知道是不是它的问题

这样后面会一团糟。

所以错误至少要结构化成：
- `code`
- `category`
- `message`
- `retryable`
- `severity`
- `details`
- `related_object`

---

## 2. 错误分类建议

建议至少分这些大类：

### 2.1 Validation Error
例如：
- 来源不足
- 来源冲突
- 信息过时
- 缺少 citation
- 输出不满足 goal

### 2.2 Permission Error
例如：
- 当前 actor 无权限执行外部写
- workflow policy 禁止当前动作
- 缺少 capability

### 2.3 Approval Error
例如：
- 审批未完成
- 审批被拒绝
- 审批已过期

### 2.4 Provider Error
例如：
- Jenkins API 报错
- 邮件服务连接失败
- Confluence 接口失败
- browser search 页面打不开

### 2.5 Timeout Error
例如：
- 搜索超时
- 页面加载超时
- 外部调用超时

### 2.6 Conflict Error
例如：
- 数据版本冲突
- workflow 定义版本不匹配
- 外部文档版本冲突

### 2.7 Internal Error
例如：
- runtime bug
- serialization failure
- unexpected state transition

---

## 3. 每类错误要回答什么

不是只有分个类就完了。  
每类错误还要回答：

- 是否可重试
- 是否需要人工介入
- 是否要进入审批状态
- 是否要停止整个 run
- 是否要落审计事件
- 是否要对用户暴露细节

---

## 4. 错误 Envelope 建议

```ts
export interface ErrorEnvelope {
  code: string
  category: "validation" | "permission" | "approval" | "provider" | "timeout" | "conflict" | "internal"
  severity: "info" | "warning" | "error" | "critical"
  message: string
  retryable: boolean
  details?: Record<string, unknown>
  relatedObject?: {
    type: string
    id: string
  }
}
```

这样 Runtime、UI、Audit、Recovery 都可以共用。

---

## 5. 权限治理怎么设计

权限最好不要写成散落在代码里的 if/else。  
应该抽成 capability policy。

建议按能力授权：
- `read`
- `search_web`
- `search_browser_live`
- `write_local`
- `write_external`
- `trigger_deploy`
- `reply_mail`
- `update_confluence`
- `approve_high_risk_action`

### 5.1 Policy 作用对象
- user
- workflow
- skill
- plugin action
- environment

### 5.2 作用方式
例如：
- 某 workflow 只允许 local write
- 某 skill 可读 Confluence，不可写
- 某 environment 禁止 deploy action

---

## 6. 审批治理怎么设计

审批不只是“有个按钮点同意”。  
它应该是系统的一等对象。

建议审批请求至少有：
- target action / skill / workflow step
- 风险级别
- 请求原因
- 输入摘要
- 预期副作用
- 申请时间
- 过期时间
- 审批人
- 审批决策

### 6.1 适合进审批的动作
- write_external
- deploy
- reply_mail
- update_confluence
- bulk notification

### 6.2 审批状态
- `pending`
- `approved`
- `rejected`
- `expired`
- `cancelled`

---

## 7. 审计治理怎么设计

自动执行系统最怕“跑了，但没人知道为什么”。

所以审计至少要记录：
- 谁触发的
- 为什么触发
- 跑了哪个 workflow / run / loop
- 用了哪个 skill / action
- 外部系统返回了什么
- validation 出了什么 issue
- approval 怎么决策的
- 最终结果是什么

### 7.1 审计事件粒度
建议至少到 step 级。

例如：
- step started
- step completed
- validation failed
- approval requested
- approval approved
- retry scheduled
- run completed

---

## 8. 恢复治理怎么设计

治理不只是拦错，还包括“出错后怎么办”。

至少要支持：
- pause
- resume
- retry
- handoff_to_human
- cancel

### 8.1 恢复点
恢复点建议至少记录：
- 当前 step
- 当前状态
- 工作内存
- 最近 artifact
- 最近 error
- 最近 approval 状态

这样中断后才能接着走，而不是从头再来。

---

## 9. 与整个系统的关系

错误与治理模型不是外挂层，而是贯穿：

- SRWV loop
- workflow runtime
- skill runtime
- plugin runtime
- approval runtime
- audit runtime

也就是说：

```text
execution
-> validation / permission check
-> approval if needed
-> audit every key step
-> retry / recovery / handoff on failure
```

---

## 10. 一句话结论

真正稳定的系统，不是“尽量别出错”，而是：

**错误要结构化、权限要能力化、审批要对象化、审计要事件化、恢复要显式化。**

这五件事如果先设计清楚，后面自动执行、外部写、浏览器搜索这些高风险能力才能稳住。
