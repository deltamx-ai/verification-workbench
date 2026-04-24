# 上下文恢复设计

## 目标
让系统在多轮协作中恢复“当前工作现场”，而不是机械回放所有历史消息。

## 五层上下文
1. Recent Messages：最近消息，保证连续性
2. Rolling Summary：近历史滚动摘要
3. Milestone Summary：关键节点摘要
4. Long-term Memory：项目长期约束、决策、风险
5. Task / Run Facts：当前任务、最近运行、最新 verdict、环境与标准

## 使用场景
### 打开 thread
- 最近消息
- 最新 rolling summary
- 最近 milestone summary
- 当前活跃 task / run links

### 继续追问失败原因
- 相关 run / step_run facts
- 失败 artifact
- 最近 verdict
- 对应摘要

### 重新开始某任务验证
- task current version
- success criteria
- environment / executor profile
- 最近历史 runs

## 快照
每次组装上下文后生成 ContextSnapshot，记录：
- 选了哪些消息
- 选了哪些摘要
- 选了哪些 memory
- 选了哪些 task / run facts
- token 预算
