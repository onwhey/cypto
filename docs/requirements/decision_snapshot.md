# 006A DecisionSnapshot 需求文档

## 1. 文档目的

006A DecisionSnapshot 是 StrategySignal 与 OrderPlan 之间的决策快照层。

它的目的不是生成买卖订单，而是冻结当前分析周期内的**策略目标仓位意图**。

DecisionSnapshot 回答的问题是：

```text
在当前市场信号和决策规则下，系统理想上希望处于什么目标仓位？
```

DecisionSnapshot 不回答：

```text
应该买还是卖？
应该开仓、平仓、加仓、减仓还是反手？
订单数量是多少？
保证金是否足够？
这组订单是否通过风控？
```

这些问题分别属于后续层：

```text
DecisionSnapshot
→ 目标仓位意图

OrderPlan
→ 当前持仓到目标仓位的订单计划

RiskCheck
→ 对 CandidateOrderIntent 做 ALLOW / DENY / BLOCKED / FAILED

ApprovedOrderIntent
→ 风控通过后的可执行订单意图

Execution
→ 真实交易所下单
```

RiskCheck P0 不做任意 MODIFY。净额反手场景只允许 RiskCheck 选择 OrderPlan 预生成的 `fallback_reduce_only_intent`，不得由 RiskCheck 临时缩单、拆单或生成新订单意图。

---

## 2. 核心边界

DecisionSnapshot 必须表达目标仓位意图，不得表达订单动作。

DecisionSnapshot 禁止输出或作为下游交易语义来源使用：

```text
ENTER_LONG
ENTER_SHORT
EXIT
HOLD
NO_TRADE 作为订单动作
BUY
SELL
OPEN_LONG
OPEN_SHORT
CLOSE_POSITION
decision_action 作为可执行订单动作
position_context_snapshot
position_context_hash
position_context_source
```

原因：

```text
006A 不拥有当前交易所真实持仓状态。
006A 不读取账户、余额、仓位、杠杆或交易所规则。
006A 不能可靠决定开仓、平仓、加仓、减仓、持有或反手。
```

因此：

```text
DecisionSnapshot 只能输出目标仓位合同。
OrderPlan 才能读取当前持仓并生成 CandidateOrderIntent。
RiskCheck 只能审批 CandidateOrderIntent。
Execution 才能提交真实订单。
```

---

## 3. 006A 核心职责

006A 负责：

```text
StrategySignal / StrategySignalQualityResult
→ DecisionPolicy / DecisionRuleDefinition
→ 目标仓位意图
→ 不可变 DecisionSnapshot
```

它必须做到：

1. 校验 StrategySignal 是否可以用于生成决策。
2. 校验 StrategySignalQualityResult 是否允许进入决策层。
3. 使用版本化 DecisionPolicy 计算目标仓位意图。
4. 冻结 target_intent、target_position_ratio、target_confidence、target_reason_code、input_snapshot、evidence_summary、lineage、schema_version、snapshot_key、trace_id、expires_at。
5. 生成可供 OrderPlan 消费的 DecisionSnapshot。

它不得做：

1. 读取 Binance REST 或 WebSocket。
2. 直接读取当前交易所持仓。
3. 消费 BinanceSyncRun 或 BinancePositionSnapshot。
4. 消费 BinanceAccountSnapshot、BinanceBalanceSnapshot 或 BinanceSymbolRuleSnapshot。
5. 计算订单数量。
6. 计算订单买卖方向。
7. 计算 reduce_only / close_position。
8. 计算保证金、余额、杠杆或 symbol rule 是否满足。
9. 生成 OrderPlan、CandidateOrderIntent、ApprovedOrderIntent 或交易所订单 payload。
10. 调用 RiskCheck、ExecutionPreparation 或 Execution。
11. 提交、撤销、查询或跟踪交易所订单。

---

## 4. 目标仓位意图模型

DecisionSnapshot 必须表达目标仓位意图，而不是订单动作。

### 4.1 target_intent

必填字段。

P0 允许值：

```text
TARGET_POSITION
NO_TARGET_CHANGE
NO_TRADE
```

#### TARGET_POSITION

表示策略和决策规则明确提出一个目标仓位比例。

当：

```text
target_intent = TARGET_POSITION
```

必须满足：

```text
target_position_ratio 必填
```

#### NO_TARGET_CHANGE

表示本轮决策规则没有提出新的目标仓位变化。

它不等于目标空仓。

它的含义是：

```text
本轮不基于该 DecisionSnapshot 生成新的候选订单计划。
```

如果账户当前已有持仓，仅凭这个 DecisionSnapshot 不要求改变该持仓。

#### NO_TRADE

表示本轮信号不足、信号冲突或不可形成有效交易决策。

NO_TRADE 只能作为 DecisionSnapshot 的 target_intent / 目标仓位意图语义，表示不产生目标仓位变化或不进入订单计划。
NO_TRADE 不得作为下游订单动作输入 RiskCheck / Execution。

它也不等于目标空仓。

它的含义是：

```text
本轮不应由该 DecisionSnapshot 生成交易链路。
```

### 4.2 target_position_ratio

条件字段。

仅当：

```text
target_intent = TARGET_POSITION
```

时必填。

当：

```text
target_intent = NO_TARGET_CHANGE
target_intent = NO_TRADE
```

时必须为空。

范围：

```text
-1.0 <= target_position_ratio <= +1.0
```

含义：

```text
+1.0 = 决策规则目标仓位预算下的最大多头目标
+0.5 = 半仓多头目标
 0.0 = 目标空仓
-0.5 = 半仓空头目标
-1.0 = 决策规则目标仓位预算下的最大空头目标
```

必须明确区分：

```text
target_position_ratio = 0.0
```

表示目标空仓。若当前已有持仓，后续 OrderPlan 可以据此生成平仓或减仓类 CandidateOrderIntent。

而：

```text
target_intent = NO_TARGET_CHANGE
```

表示本轮未提出新的目标仓位变化，不得被解释为目标空仓。

### 4.3 target_confidence

可选但建议保留。

范围：

```text
0.0 <= target_confidence <= 1.0
```

表示决策规则对本次目标仓位意图的置信度。

### 4.4 target_reason_code

必填，用于审计。

示例：

```text
trend_following_bullish
trend_following_bearish
signal_conflict_no_trade
weak_signal_no_target_change
risk_off_target_flat
```

具体 reason code 必须由版本化 DecisionPolicy 定义。

### 4.5 target_reason_summary

面向人类阅读的摘要，用于审计和调试。

它不能替代结构化 evidence。

### 4.6 target_schema_version

必填。

P0 值：

```text
v2
```

用于标识目标仓位合同版本。

---

## 5. DecisionPolicy

006A 必须使用版本化 DecisionPolicy，将 StrategySignal 转换为目标仓位意图。

DecisionPolicy 负责把以下输入：

```text
strategy direction
strategy strength
strategy confidence
market regime descriptor
quality result
policy params
```

转换为：

```text
target_intent
target_position_ratio
target_confidence
target_reason_code
target_reason_summary
```

P0 阶段，DecisionPolicy 不允许把当前交易所真实持仓作为输入。

示例行为：

```text
bullish + strong + high confidence
→ TARGET_POSITION, target_position_ratio > 0

bearish + strong + high confidence
→ TARGET_POSITION, target_position_ratio < 0

neutral / weak / conflicted
→ 根据策略规则输出 NO_TRADE 或 NO_TARGET_CHANGE

strategy signal 明确 risk-off / exit
→ TARGET_POSITION, target_position_ratio = 0.0
```

具体映射必须由 policy 定义并版本化，不得写成未版本化的 service 分支硬编码。

---

## 6. 输入要求

P0 必需输入：

```text
StrategySignal
StrategySignalQualityResult
DecisionRuleDefinition / DecisionPolicyDefinition
通过 StrategySignal 继承的 MarketSnapshot lineage 引用
trace_id 或 cycle identity
```

DecisionSnapshot 必须校验：

1. StrategySignal 存在。
2. StrategySignal 可用。
3. StrategySignalQualityResult 存在。
4. StrategySignalQualityResult 允许进入下游决策。
5. DecisionPolicyDefinition 已启用、处于 active 状态，并且 hash 一致。
6. 源 MarketSnapshot lineage 可追踪。

P0 禁止输入：

```text
BinanceSyncRun
BinanceAccountSnapshot
BinanceBalanceSnapshot
BinancePositionSnapshot
BinanceSymbolRuleSnapshot
position_context_json
exchange REST / WebSocket data
```

---

## 7. 输出快照要求

DecisionSnapshot 必须冻结足够信息以支持后续审计。

逻辑必需字段：

```text
strategy_signal_id
strategy_signal_quality_result_id
decision_rule_definition_id
trace_id
cycle identity / market_as_of_utc
target_intent
target_position_ratio
target_confidence
target_reason_code
target_reason_summary
target_schema_version
input_snapshot
evidence_summary
snapshot_key
expires_at
status
is_usable
allows_downstream
blocked_reason
failed_reason
created_at
```

`input_snapshot` 必须包含：

```text
strategy direction
strategy strength
strategy confidence
strategy mode / definition references
quality result references
decision policy params hash
source snapshot lineage references
```

`input_snapshot` 不得包含当前账户持仓作为决策输入。

---

## 8. Hash 与幂等性

DecisionSnapshot 的幂等性必须基于目标仓位合同字段。

hash / snapshot_key 至少包含：

```text
strategy_signal_id
strategy_signal_quality_result_id
decision_rule_definition_id
decision policy definition hash
target_schema_version
target_intent
target_position_ratio
target_confidence
target_reason_code
market_as_of_utc / cycle identity
input lineage hashes
```

不得包含：

```text
position_context_hash
BinancePositionSnapshot
BinanceSyncRun
decision_action
```

相同输入和相同 decision policy 必须返回相同 DecisionSnapshot 或相同幂等摘要。

不同的 target_intent 或 target_position_ratio 必须生成不同 snapshot_key。

---

## 9. 下游行为

### 9.1 TARGET_POSITION

当：

```text
target_intent = TARGET_POSITION
```

且快照可用、未过期时，OrderPlan 可以消费该 DecisionSnapshot。

OrderPlan 负责读取当前持仓，并把目标仓位比例转换为 CandidateOrderIntent。

### 9.2 NO_TARGET_CHANGE

当：

```text
target_intent = NO_TARGET_CHANGE
```

OrderPlan 不应为本周期生成 CandidateOrderIntent。

这是正常终止状态，不是系统失败。

### 9.3 NO_TRADE

当：

```text
target_intent = NO_TRADE
```

不得基于该 DecisionSnapshot 生成 OrderPlan、CandidateOrderIntent、RiskCheck 或 Execution。

除非该状态由无效数据或质量检查失败导致，否则它是正常终止状态。

---

## 10. 过期与生命周期

DecisionSnapshot 是按分析周期冻结的决策快照。

它必须包含：

```text
expires_at
status
is_usable
allows_downstream
blocked_reason
failed_reason
```

过期的 DecisionSnapshot 不得被 OrderPlan 消费。

如果下游流程尝试消费过期 DecisionSnapshot，必须 fail-closed，并写 AlertEvent；dry-run 除外。

不得仅因为快照过期就对相同事实重新生成同周期 DecisionSnapshot。新的分析周期应产生新的快照。

---

## 11. AlertEvent 行为

006A 必须在以下情况写 AlertEvent：

```text
StrategySignal 缺失
StrategySignal 不可用
StrategySignalQualityResult 缺失
quality result 不允许下游
DecisionPolicy 缺失、禁用、retired 或 hash 不一致
target policy 计算失败
target_position_ratio 非法
snapshot 生成失败
```

006A 不应对正常的以下状态写高严重级别告警：

```text
NO_TARGET_CHANGE
NO_TRADE
```

除非这些状态是由上游数据缺失、无效或质量检查失败导致。

dry-run 不得写 DecisionSnapshot 或 AlertEvent。

---

## 12. Management command 要求

生成 DecisionSnapshot 的 management command 必须只接收 StrategySignal / StrategySignalQuality / DecisionPolicy 相关输入。

建议命令：

```text
build_decision_snapshot
```

命令行为：

```text
接收 StrategySignal / quality 引用或 cycle selector
应用 DecisionPolicy
输出 target_intent 和 target_position_ratio
支持 dry-run
支持 trace_id / trigger_source
```

命令不得接收或要求：

```text
position_context_json
当前持仓
账户余额
交易所订单信息
```

命令输出不得展示 `decision_action` 作为主要结果。

---

## 13. 非目标

006A 不得实现：

```text
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
真实 Binance REST / WebSocket 调用
提交订单
撤单
查订单
成交追踪
持仓对账
回测收益
UI / dashboard
```

---

## 14. 测试要求

至少覆盖：

```text
StrategySignal 缺失 → blocked / failed
StrategySignal 不可用 → blocked
StrategySignalQualityResult 缺失 → blocked / failed
quality result 不允许下游 → blocked
DecisionPolicy 缺失 / disabled / retired → blocked
DecisionPolicy hash 不一致 → blocked
bullish 策略信号生成 TARGET_POSITION 且 target_position_ratio > 0
bearish 策略信号生成 TARGET_POSITION 且 target_position_ratio < 0
risk-off / exit 策略信号生成 TARGET_POSITION 且 target_position_ratio = 0.0
neutral / weak / conflicted 策略信号生成 NO_TRADE 或 NO_TARGET_CHANGE
target_position_ratio 超出 [-1.0, +1.0] → failed
TARGET_POSITION 时 target_position_ratio 必填
NO_TARGET_CHANGE / NO_TRADE 时 target_position_ratio 必须为空
target_position_ratio = 0.0 与 NO_TARGET_CHANGE 语义区分
snapshot_key 幂等
不同 target_intent 或 target_position_ratio 生成不同 snapshot_key
hash 不包含 position_context_hash / BinancePositionSnapshot / BinanceSyncRun / decision_action
input_snapshot 不包含当前账户持仓作为决策输入
过期 DecisionSnapshot 不允许 OrderPlan 消费
dry-run 不写 DecisionSnapshot 或 AlertEvent
不读取 Binance account / position / order / execution 逻辑
不生成 OrderPlan / CandidateOrderIntent / ApprovedOrderIntent
不调用 RiskCheck / ExecutionPreparation / Execution
management command 不接收 position_context_json
```

---

## 15. 验收标准

006A 完成后必须满足：

1. DecisionSnapshot 被定义为目标仓位快照。
2. DecisionSnapshot 不把当前持仓作为必需决策输入。
3. DecisionSnapshot 不输出 `ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE` 作为下游可执行动作。
4. 目标仓位意图模型支持 `TARGET_POSITION / NO_TARGET_CHANGE / NO_TRADE`。
5. `target_position_ratio` 仅在 `TARGET_POSITION` 时必填，且范围限制在 `[-1.0, +1.0]`。
6. 明确区分 `target_position_ratio = 0.0` 和 `NO_TARGET_CHANGE`。
7. Hash / 幂等性不依赖 position context、Binance position snapshot、Binance sync run 或 decision_action。
8. Management command 不接收 `position_context_json`。
9. 006A 不引入 Binance account / position / order / execution 逻辑。
10. 006A 不生成 OrderPlan、CandidateOrderIntent、ApprovedOrderIntent 或交易所订单 payload。
11. 006A 只能被 OrderPlan 作为下游交易链路入口消费。
