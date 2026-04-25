# 准确性保障与评估设计

这份文档专门回答一个核心问题：

**verification workbench 不只是要“能跑”，更要“尽量不胡说、不误写、不误触发”。准确性怎么设计出来，而不是事后祈祷？**

在这套系统里，准确性不是单点能力，而是一条保障链：
- source 质量
- retrieval 策略
- facts 抽取
- conflict 检测
- validation policy
- citation 约束
- external write 前二次校验
- 结果评估与回归集

也就是说，准确性要被系统化，而不是交给某一个 prompt 临场发挥。

---

## 1. 准确性问题通常出在哪

如果不专门设计，系统最容易出这些错：

- 用过时消息当最新消息
- 只看单一来源就下结论
- 不同来源冲突却没发现
- 抽取事实时把“猜测”当成“事实”
- 没有 citation 也敢输出确定答案
- 外部写之前没有再次验证
- 自动流程因为一条噪声消息被误触发

所以“准确性”不能只理解成模型答题准不准。  
在这里更准确地说，它是：

**系统对来源、事实、结论、副作用的整体可信控制能力。**

---

## 2. 准确性保障链

建议把准确性拆成七层保障。

### 2.1 Source Trust Layer

先判断来源值不值得信。

给每种 source 打可信等级：
- `authoritative`
- `trusted`
- `useful`
- `unverified`
- `ephemeral`

例如：
- 正式 Confluence 页面：`authoritative`
- Jenkins 官方状态：`trusted`
- 邮件转述：`useful`
- 随手群消息：`unverified`
- 临时网页动态：`ephemeral`

后面 validation 时，不能只看“有没有来源”，还要看来源等级够不够。

---

### 2.2 Freshness Layer

特别是 search 支持浏览器查最新消息后，“最新”本身就成了系统约束。

建议为每次执行显式定义：
- 是否要求 freshness
- freshness 时间窗
- 超出时间窗是否直接 fail validation

例如：
- 发布状态：要求 30 分钟内来源
- 每日总结：允许当天来源
- 历史设计文档：不要求 freshness

---

### 2.3 Multi-source Cross-check Layer

对关键结论，不应该默认单一来源足够。  
要定义最小证据规则，例如：

- 发布成功：至少 2 个来源交叉确认
- 外部通知前：至少 1 个 authoritative + 1 个 runtime source
- 只做内部草稿：允许单一 trusted source

这样系统才能根据风险动态提升准确性门槛。

---

### 2.4 Fact Extraction Layer

Read 阶段不要只抽文段，要抽成事实。  
每条 fact 最好包含：
- fact_text
- fact_type
- source
- timestamp
- confidence
- trust_level
- is_latest
- conflict_group

后面 validation 要比对的是 fact，不是整页原文。

---

### 2.5 Conflict Detection Layer

系统必须能显式发现冲突，而不是悄悄选一个自己喜欢的。  
典型冲突：
- 一个来源说成功，一个来源说失败
- 一个来源是 08:30，一个来源是 07:10
- 一个来源说 staging，一个来源说 prod

冲突发现后，系统要支持：
- 降低 confidence
- 回到 search/read
- 转人工
- 禁止 external write

---

### 2.6 Validation Policy Layer

Validation 不是“看起来差不多就过”，而应配置成策略。

建议至少支持：
- freshness_required
- minimum_source_count
- minimum_trust_level
- citation_required
- no_unresolved_conflicts
- approval_required_for_external_write
- minimum_confidence_threshold

不同 workflow / skill / write_target 可以绑定不同策略。

---

### 2.7 Output Safeguard Layer

最终输出也要分级保护：

- `draft`：允许不完美，但必须带 caveat
- `local`：需要 citation，允许低风险不确定
- `external`：必须满足更高 validation，必要时审批

尤其 external write，一定要在执行前做二次 validation。

---

## 3. Citation 设计必须是强约束

citation 不能当装饰。  
它应该是准确性系统的一部分。

建议：
- 每条关键结论都能回链到 source
- citation 至少带 source_id / timestamp / excerpt
- 没 citation 的结论不能标成“确定”
- UI 能点开看依据

也就是说：

**citation 不是为了好看，而是为了让系统能自证。**

---

## 4. Confidence 不能只是一串数字

confidence 可以有，但不要神化。  
更推荐把它拆成几个来源：

- source_trust_score
- freshness_score
- agreement_score
- extraction_confidence
- validation_pass_ratio

最终再合成一个综合 score。  
这样后面 debug 时才知道到底哪里拉低了可信度。

---

## 5. “不知道”必须是一等结果

很多系统不准，不是因为完全错，而是因为在证据不足时也硬给答案。  
所以必须允许下面这些显式结果：

- `insufficient_evidence`
- `conflicting_sources`
- `outdated_information`
- `awaiting_approval`
- `requires_human_confirmation`

也就是说，
**不确定不是失败，乱确定才是。**

---

## 6. 评估体系怎么做

准确性不能靠感觉，要有固定评估集。

建议至少建三类 case：

### 6.1 Golden Cases

理想路径、标准来源齐全、结论明确。  
用来保证主链没退化。

### 6.2 Conflict Cases

多个来源互相冲突。  
用来验证：
- 系统是否发现冲突
- 是否阻止 external write
- 是否正确转人工

### 6.3 Failure Cases

来源缺失、超时、页面打不开、审批未过。  
用来验证系统是不是稳，而不是只在顺风局里准。

---

## 7. 回归与评分

建议每次改动都跑一套回归：
- retrieval regression
- fact extraction regression
- validation regression
- workflow external write safeguard regression

评分可以看：
- citation completeness
- freshness correctness
- conflict detection rate
- false confident answer rate
- external write block rate when evidence is insufficient

这几个指标比普通“回答像不像”更重要。

---

## 8. 与整体架构的关系

准确性保障不是一个外挂模块。  
它应该嵌在：
- Search Provider
- Read Extractor
- Validation Policy
- Workflow Runtime
- Approval / Governance
- UI Projection

UI 也要帮准确性：
- 明示来源等级
- 明示时间戳
- 明示冲突
- 明示是否已审批
- 明示是否是草稿/正式结果

---

## 9. 一句话结论

verification workbench 的准确性要靠一整条保障链来设计：

**从 source trust、freshness、multi-source cross-check、fact extraction、conflict detection、validation policy、citation 到 external write safeguard，层层收紧。**

不要把“准不准”寄托在某一次模型输出上。  
系统级准确性，必须来自系统级约束。
