# RiskCheck 需求文件

> 建议存放路径：`docs/requirements/risk_check.md`  
> P0 范围：基于 `CandidateOrderIntent → RiskCheck → ApprovedOrderIntent` 合同实现事前风控。  
> 模块定位：是候选订单意图进入执行准备前的事前风控层。  
> 核心原则：RiskCheck 只检查候选订单意图是否允许进入执行准备，不生成策略判断、不制定目标仓位、不真实下单、不调用 Binance 修改交易环境。

---

## 1. 链路定位

主链路为：

```text
DecisionSnapshot
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation
→ Execution
→ Tracking
```

RiskCheck 的直接输入不是 `DecisionSnapshot`。

RiskCheck 只消费：

```text
OrderPlan
CandidateOrderIntent
Binance Account Sync 快照批次
PriceSnapshot
RiskRuleDefinition / risk_config
```

RiskCheck 负责回答：

```text
这一个 CandidateOrderIntent 是否允许进入执行准备？
如果不允许，是风险拒绝、前置阻断，还是系统失败？
如果是净额反手订单，主反手失败时是否可以批准 fallback_reduce_only_intent？
```

RiskCheck 不负责回答：

```text
策略是否应该做多 / 做空
目标仓位比例是多少
系统内部全仓是多少
应该生成什么候选订单
真实订单是否会成交
```

---

## 2. 输入合同边界

RiskCheck 的正式检查对象是：

```text
CandidateOrderIntent
order_components
```

RiskCheck 禁止把以下策略动作或DecisionSnapshot 动作字段作为正式输入：

```text
DecisionSnapshot.decision_action
ENTER_LONG
ENTER_SHORT
EXIT
HOLD
NO_TRADE
```

以下规则不得作为 P0 风控规则直接使用：

```text
decision_action_allowed
position_conflict
exit_position_exists
HOLD / NO_TRADE 风控判断
反方向已有持仓直接 rejected
```

RiskCheck 必须基于候选订单及其内部风控组件判断，而不是基于策略动作判断。

---

## 3. 模块边界

RiskCheck 必须做到：

```text
1. 校验 CandidateOrderIntent / OrderPlan 是否有效。
2. 校验 Binance Account Sync 快照批次是否可消费。
3. 校验账户、余额、持仓、symbol rule、价格快照是否完整且一致。
4. 按 market_type 区分 U 本位和 COIN-M 的保证金与交易规则。
5. 按 order_components 分段评估净额订单内部的风险语义。
6. 输出 ALLOW / DENY / BLOCKED / FAILED。
7. 对所有正式订单相关结果写 AlertEvent。
8. 生成可追溯、可幂等、可审计的 RiskCheckResult。
```

RiskCheck 不得做：

```text
1. 不读取或修改策略信号。
2. 不重新生成 DecisionSnapshot。
3. 不生成目标仓位。
4. 不生成新的 CandidateOrderIntent。
5. 不做任意 MODIFY，不自动缩小订单。
6. 不真实下单。
7. 不撤单。
8. 不查订单成交。
9. 不调用 Binance REST / WebSocket。
10. 不修改 Binance leverage / margin type / position mode。
11. 不执行 30 秒 / 0.5% 的最终 price guard。
```

---

## 4. 输入契约

RiskCheck P0 输入至少包括：

```text
candidate_order_intent_id
order_plan_id
binance_sync_run_id
price_snapshot_id
symbol
market_type
account_domain
risk_rule_set / RiskRuleDefinition 集合
risk_config
reference_time_utc
trace_id
trigger_source
dry_run
```

其中：

```text
CandidateOrderIntent 必须来自有效 OrderPlan。
CandidateOrderIntent.status 必须是 pending_risk_check 或等价状态。
OrderPlan 必须是 created / usable / allows_downstream=True。
```

RiskCheck 不接受由 DecisionSnapshot 直接触发的风控输入。

---

## 5. BinanceSyncRun 事实批次边界

RiskCheck 正式链路必须使用编排层传入的：

```text
binance_sync_run_id
```

并且只读取该批次下的：

```text
BinanceSyncRun
BinanceAccountSnapshot
BinanceBalanceSnapshot
BinancePositionSnapshot
BinanceSymbolRuleSnapshot
```

该批次必须满足：

```text
BinanceSyncRun.status = succeeded
BinanceSyncRun.market_type = CandidateOrderIntent.market_type
BinanceSyncRun.account_domain = CandidateOrderIntent.account_domain
BinanceSyncRun.exchange = binance
BinanceSyncRun 未过期
snapshot_set_hash 可校验
account / balance / position / symbol rule 快照完整
```

如果 `binance_sync_run_id` 缺失、不存在、状态不可消费、market_type 不匹配、account_domain 不匹配、批次过期、快照不完整或 hash 校验失败：

```text
RiskCheckResult.status = BLOCKED
allows_downstream = false
写 AlertEvent
```

正式链路不得自动回退到其他 BinanceSyncRun 批次。

---

## 6. market_type 与账户域隔离

RiskCheck 必须严格隔离：

```text
usds_m_futures
coin_m_futures
```

不得跨 market_type 使用：

```text
账户余额
持仓快照
symbol rule
mark_price
contractSize
availableBalance
marginAsset
```

运行时只能消费当前 CandidateOrderIntent / OrderPlan 指定的 account_domain 与 market_type。

如果快照 market_type、symbol、account_domain 不一致：

```text
RiskCheckResult.status = BLOCKED
reason_code = market_identity_mismatch / account_domain_mismatch
```

---

## 7. CandidateOrderIntent 基本校验

RiskCheck 必须校验 CandidateOrderIntent：

```text
id 存在
status 可消费
order_plan_id 匹配
symbol 匹配
market_type 匹配
account_domain 匹配
side 合法
requested_size / requested_notional 合法
positionSide 符合 One-Way Mode 语义
reduceOnly 合法
order_components 存在且结构合法
intent_hash 可校验
未被取消、废弃、过期或已消费
```

如果结构缺失或 hash 校验失败：

```text
RiskCheckResult.status = BLOCKED
```

---

## 8. One-Way Mode 要求

RiskCheck P0 只支持 Binance One-Way Mode。

One-Way Mode 下：

```text
positionSide = BOTH
```

如果快照显示当前为 Hedge Mode，或 CandidateOrderIntent 使用 LONG / SHORT positionSide：

```text
RiskCheckResult.status = BLOCKED
reason_code = hedge_mode_not_supported
allows_downstream = false
写 AlertEvent
```

RiskCheck 不调用 Binance API 修改 position mode。

---

## 9. margin mode / leverage 只读

RiskCheck 不调用 Binance API 修改：

```text
margin type
position mode
leverage
```

这些信息只来自 Binance Account Sync 快照。

如果 margin mode / leverage 字段存在，RiskCheck 可以用于：

```text
保证金估算
交易环境审计
evidence 记录
```

如果必要字段缺失导致无法估算保证金：

```text
RiskCheckResult.status = BLOCKED
```

---

## 10. 系统内部目标仓位与 observed_exchange_leverage 的边界

系统内部目标仓位由 OrderPlan 使用以下字段生成：

```text
current_equity
max_target_notional_to_equity_ratio
target_position_ratio
```

RiskCheck 不得使用 `observed_exchange_leverage` 重新计算：

```text
系统全仓
系统半仓
target_notional
target_quantity
target_contracts
```

`observed_exchange_leverage` 的含义：

```text
Binance 当前 symbol 实际设置的交易所杠杆。
```

RiskCheck 只能将其用于：

```text
1. 估算当前交易所保证金环境是否能承接 CandidateOrderIntent。
2. 写入审计与 evidence。
```

允许出现：

```text
observed_exchange_leverage = 4
max_target_notional_to_equity_ratio = 3
```

这种设置用于给滑点、手续费和价格变化留缓冲。

---

## 11. order_components 风控语义

CandidateOrderIntent 必须包含 `order_components`。

`order_components` 是内部风控组件，不是 Binance 子订单。

真实 Binance 订单只有一套：

```text
side
quantity / contracts
reduceOnly
positionSide
```

组件中不得使用 Binance 参数 `reduceOnly` 表示局部订单属性。组件应使用：

```text
risk_effect
is_risk_reducing
position_effect
component_type
```

示例：净额反手主候选订单

```text
真实订单：
SELL 1.3
reduceOnly = false
positionSide = BOTH

order_components：
1. close_long 0.8  → reduce_risk
2. open_short 0.5 → increase_risk
```

RiskCheck 必须按组件分段评估：

```text
reduce_risk 组件：
  校验当前持仓存在、方向匹配、数量不超过当前持仓。
  校验该真实订单数量满足交易所 symbol rule，包括 minQty、stepSize、minNotional、maxQty 等。
  不按新增仓位计算保证金。
  不要求 observed_exchange_leverage 存在。

increase_risk 组件：
  校验保证金、availableBalance、交易所最大数量/最大名义、symbol rule。
  需要 observed_exchange_leverage 用于保证金估算。
```

说明：

```text
降风险组件虽然不新增保证金，但仍然是真实交易所订单的一部分。
因此 reduce_risk 组件不能跳过 minQty / stepSize / minNotional 等交易所规则校验。
如果残仓过小导致无法满足交易所规则，P0 不暗中放行，应返回 DENY 或 BLOCKED，并写 AlertEvent。
```

最终结果仍然是整体 RiskCheckResult。


---

## 12. 净额反手与 fallback_reduce_only

One-Way Mode 下，OrderPlan 可以为反手生成主候选订单。

示例：

```text
当前 long +0.8
目标 short -0.5
```

主候选：

```text
primary_intent:
SELL 1.3
reduceOnly = false
components = close_long 0.8 + open_short 0.5
```

同时必须有降风险 fallback：

```text
fallback_reduce_only_intent:
SELL 0.8
reduceOnly = true
components = close_long 0.8
```

RiskCheck 规则：

```text
primary_intent 全部通过
→ ALLOW primary_intent

primary_intent 的 increase_risk component 不通过，
但 fallback_reduce_only_intent 通过
→ ALLOW fallback_reduce_only_intent

fallback_reduce_only_intent 也不通过
→ 按失败原因返回 DENY / BLOCKED / FAILED
```

fallback 失败时的状态归类：

```text
当前持仓不存在、方向不一致、当前持仓数量小于 fallback 要平的数量、
或 OrderPlan 使用的持仓事实与 RiskCheck 当前快照不一致
→ BLOCKED
→ reason_code = position_snapshot_mismatch / stale_position_snapshot / current_position_insufficient

fallback 数量完整但不满足交易所 minQty / stepSize / minNotional / maxQty 等规则
→ DENY
→ reason_code = symbol_rule_violation

缺少 position snapshot / symbol rule / price snapshot 等关键字段
→ BLOCKED

系统异常、数据库异常、不可预期错误
→ FAILED
```

触发 fallback_reduce_only 时必须写 AlertEvent，说明：

```text
主反手目标未完整通过；系统只批准降低旧方向风险的平仓意图。
```

如果 primary 和 fallback 都不能通过，也必须写 AlertEvent，说明：

```text
反手主订单未通过，降风险 fallback 也未通过，当前持仓仍未按目标方向降低风险。
```

fallback_reduce_only 不是任意 MODIFY，而是 OrderPlan 预先生成的安全候选。


---

## 13. P0 不做任意 MODIFY

RiskCheck P0 不做：

```text
自动缩小订单
自动改数量
自动拆单
自动把 BUY 300 改成 BUY 260
自动保留部分组件
```

如果系统目标订单超过当前 Binance 环境可承接能力：

```text
RiskCheckResult.status = DENY
reason_code = exchange_open_capacity_insufficient 或等价语义
写 AlertEvent
```

唯一例外：

```text
净额反手场景允许选择 OrderPlan 预生成的 fallback_reduce_only_intent。
```

---

## 14. 状态定义

RiskCheck P0 状态：

```text
ALLOW
DENY
BLOCKED
FAILED
```

含义：

```text
ALLOW
= 候选订单通过风控，允许形成 ApprovedOrderIntent。

DENY
= 输入完整且可判断，但风险约束不允许。

BLOCKED
= 前置条件不满足、数据缺失、快照不一致、结构不合法，无法进行有效风控判断。

FAILED
= 系统异常、代码异常、数据库异常、不可预期错误。
```

示例：

```text
availableBalance 不足 → DENY
超过 Binance 最大可承接能力 → DENY
缺 COIN-M contractSize → BLOCKED
缺 PriceSnapshot → BLOCKED
CandidateOrderIntent hash 失败 → BLOCKED
数据库异常 → FAILED
```

模型内部可以使用小写枚举，但语义必须与上述四类一致。

---

## 15. U 本位保证金检查

U 本位使用 quote asset，例如 USDT / USDC。

OrderPlan 已生成目标数量，RiskCheck 不重新用 observed_exchange_leverage 生成目标数量。

RiskCheck 对 increase_risk 组件估算保证金：

```text
opening_notional_quote = component_size_quantity * mark_price
margin_required_quote = opening_notional_quote / observed_exchange_leverage
```

余额检查：

```text
available_balance_quote >= margin_required_quote + buffer
```

其中：

```text
available_balance_quote 来自 Binance Account Sync 快照。
observed_exchange_leverage 只用于保证金估算。
observed_exchange_leverage 不参与系统目标仓位计算。
```

如果 `observed_exchange_leverage` 缺失、为 0、为负数，且存在 increase_risk 组件：

```text
RiskCheckResult.status = BLOCKED
reason_code = observed_exchange_leverage_missing
```

对于 `is_risk_reducing = true` 的 reduce_risk 组件：

```text
不按新增仓位计算保证金；
不检查 observed_exchange_leverage 是否缺失；
仍必须检查持仓匹配、数量不超过当前持仓、symbol rule 合法性。
```


---

## 16. COIN-M 保证金检查

COIN-M 使用 settlement asset，例如 BTC / ETH。

COIN-M 的 `mark_price` 表示该 symbol / pair 的 USD 标记价格，例如 BTCUSD_PERP 的 BTC/USD 标记价格。

COIN-M 的 `contractSize` 表示每张合约对应的固定 USD 面值。

不得混淆：

```text
mark_price = 当前市场标记价格
contractSize = 每张合约固定 USD 面值
```

`contractSize` 必须满足：

```text
存在
Decimal 可解析
> 0
与当前 symbol rule 快照一致
```

`marginAsset` / settlement asset 必须能在同一 BinanceSyncRun 的余额快照中找到对应资产。不得跨 market_type 或跨 account_domain 使用余额。

increase_risk 组件的 USD 名义：

```text
opening_notional_usd = component_contracts * contractSize
```

保证金换算成 settlement asset：

```text
margin_required_native = opening_notional_usd / mark_price / observed_exchange_leverage
```

余额检查：

```text
available_balance_native >= margin_required_native + buffer
```

如果缺少以下任一字段，且存在 increase_risk 组件，必须 BLOCKED：

```text
contractSize
mark_price
marginAsset
available_balance_native
observed_exchange_leverage
```

对于 `is_risk_reducing = true` 的 reduce_risk 组件：

```text
不按新增仓位计算保证金；
不检查 observed_exchange_leverage 是否缺失；
仍必须检查持仓匹配、contracts 数量不超过当前持仓、symbol rule 合法性；
仍需要 contractSize / symbol rule 用于交易规则和审计。
```

COIN-M 不得复用 U 本位线性 PnL / 保证金公式。


### 16.1 market calculator 抽象与共享计算工具

RiskCheck 必须按 market_type 区分 market calculator。

P0 至少应支持：

```text
usds_m_futures market calculator
coin_m_futures market calculator
```

两类 calculator 可以共用中性的底层计算工具，例如：

```text
Decimal 解析
Decimal 量化
stepSize / tickSize 对齐
minQty / maxQty 检查
minNotional / maxNotional 检查
notional 摘要计算
通用 buffer 计算
通用错误码构造
```

但不得把 market_type 的语义差异抽象掉。

规则：

```text
U 本位和 COIN-M 必须有独立 market calculator。
公共 helper / shared calculator 只能做中性数值计算和通用校验。
公共 helper 不得决定 RiskCheckResult.status。
公共 helper 不得访问数据库。
公共 helper 不得调用 Binance。
公共 helper 不得写 AlertEvent。
公共 helper 不得承载 rule_code 分发。
```

market calculator 输出必须服务于 RiskRulePlugin / RuleEngine 的风控判断，不得绕过 rule plugin 直接生成 ApprovedOrderIntent。


---

## 17. Binance 最大可承接能力检查

RiskCheck 应检查当前 Binance 环境是否能承接 CandidateOrderIntent，但 P0 不实现精确模拟 Binance App 的“最大可开”计算。

原因是精确最大可开会受到以下因素共同影响：

```text
observed_exchange_leverage
availableBalance
已有持仓
已有挂单
openOrderInitialMargin
positionInitialMargin
mark_price
手续费与 buffer
保证金模式
交易所杠杆档位 / 风控档位
```

P0 将该能力拆成明确规则检查：

```text
1. symbol_rule_max_quantity_check
   检查 requested_size 是否超过 symbol rule 的 maxQty。

2. symbol_rule_max_notional_check
   检查 requested_notional 是否超过 maxNotional / maxNotionalValue 等可用字段。

3. available_margin_check
   基于 availableBalance、observed_exchange_leverage、mark_price、contractSize 估算新增保证金是否足够。

4. active_order_guard
   阻断同一 symbol / market_type / account_domain 下未完成订单链路，避免已有挂单影响保证金能力但系统重复开单。
```

如果上述 P0 明确规则任一不通过：

```text
数据完整但规则不允许 → DENY
关键字段缺失或快照不一致 → BLOCKED
写 AlertEvent
```

P0 不做：

```text
精确 max_open_capacity calculator
自动缩小到 Binance 最大可开
自动把 BUY 300 改成 BUY 260
```

更精细的交易所最大可开估算列为 P1。


---

## 18. symbol rule 检查

RiskCheck 必须基于 BinanceSymbolRuleSnapshot 检查：

```text
minQty
maxQty
stepSize
minNotional
maxNotional / maxNotionalValue
priceTick / tickSize
contractSize（COIN-M）
marginAsset（COIN-M）
```

数量必须按交易所规则量化。

如果候选订单数量不满足 stepSize / minQty / maxQty / minNotional：

```text
数据完整但规则不允许 → DENY
缺关键规则字段 → BLOCKED
```

P0 不自动修正数量。

---

## 19. PriceSnapshot 使用边界

RiskCheck 使用 PriceSnapshot 进行：

```text
notional 估算
margin_required 估算
COIN-M native margin 换算
审计追溯
```

RiskCheck 不执行最终下单前 price guard。

但 RiskCheck 不得使用明显过期的价格快照做保证金估算。

P0 增加估值价格 TTL：

```text
RISK_CHECK_PRICE_SNAPSHOT_MAX_AGE_SECONDS = 60
```

如果：

```text
reference_time_utc - price_snapshot.as_of_utc > 60 秒
```

则：

```text
RiskCheckResult.status = BLOCKED
reason_code = stale_price_snapshot
allows_downstream = false
写 AlertEvent
```

说明：

```text
60 秒 TTL 只用于 RiskCheck 估值价格是否过旧。
它不等于最终下单价格保护。
RiskCheck 不检查 0.5% 价格偏离。
```

ExecutionPreparation 负责：

```text
刷新最新 PriceSnapshot
latest_price_snapshot_age <= 30 秒
price_deviation <= 0.5%
```

30 秒不是指 OrderPlan / RiskCheck 后必须 30 秒内下单，而是 ExecutionPreparation 真正准备发单前读取到的最新价格快照不能超过 30 秒。


---

## 20. 穿仓机制 / 保险基金

Futures Insurance Fund / 保险基金是交易所强平和穿仓后的兜底机制。

它不是用户账户中预先扣留的一部分保证金。

RiskCheck 不应假设：

```text
账户权益 100，其中 20 是穿仓保证金，只有 80 可开仓。
```

RiskCheck P0 使用账户快照中的：

```text
availableBalance
initialMargin
maintMargin
positionInitialMargin
openOrderInitialMargin
marginBalance
```

不使用保险基金作为可用保证金，也不预留所谓“穿仓保证金”。

---

## 21. AlertEvent 要求

所有正式订单相关事件，不管成功还是失败，都必须写 AlertEvent。

RiskCheck 对每一次正式 CandidateOrderIntent 检查结果都必须 AlertEvent：

```text
ALLOW → 必须 AlertEvent
DENY → 必须 AlertEvent
BLOCKED → 必须 AlertEvent
FAILED → 必须 AlertEvent
fallback_reduce_only → 必须 AlertEvent
```

原因：

```text
开仓 / 加仓 = 风险增加事件
减仓 / 平仓 = 风险变化事件
反手 = 风险方向切换事件
DENY / BLOCKED / FAILED = 目标仓位未完成或交易链路异常
```

AlertEvent 内容至少包括：

```text
risk_check_result_id
candidate_order_intent_id
order_plan_id
symbol
market_type
account_domain
status
selected_intent_type
reason_code
side
requested_size
requested_notional
components 摘要
trace_id
```

dry-run 不写 AlertEvent。

P1 可以增加 AlertEvent 路由、级别、聚合、限频与降噪策略，但不得改变以下 P0 原则：

```text
所有正式订单相关结果都必须产生 AlertEvent 记录。
```


---

## 22. RiskRuleDefinition 与 rule plugin 架构

RiskCheck 采用配置化规则和 plugin 架构。

建议模型：

```text
RiskRuleDefinition
RiskCheckResult
RiskRuleResult / RiskCheckIssue
RiskCheckService
RiskRulePlugin
RiskRuleRegistry
RuleEngine
```

`RiskRuleDefinition` 字段建议：

```text
rule_code
rule_version
algorithm_name
algorithm_version
params
params_hash
definition_hash
status
enabled
severity
created_at
updated_at
```

生命周期：

```text
draft
active
deprecated
retired
disabled
```

生产只使用：

```text
status = active
enabled = True
```

已用于生产的规则定义不得原地修改关键身份字段。参数变化应新增版本。

RuleEngine 只负责：

```text
读取 active / enabled 的 RiskRuleDefinition
根据 rule_code 从 RiskRuleRegistry 查找 plugin
调用 plugin
汇总 RiskRuleResult / RiskCheckIssue
聚合最终 RiskCheckResult
```

RuleEngine 不得承载具体规则判断逻辑，不得使用大型 if / elif / switch 分发具体 rule_code。

如果 `RiskRuleDefinition.rule_code` 找不到 plugin：

```text
RiskCheckResult.status = BLOCKED 或 FAILED
allows_downstream = false
写 AlertEvent
```

### 22.1 rule plugin 文件组织

RiskCheck rule 必须采用注册制。

规则：

```text
原则上一个 RiskRulePlugin 一个文件。
文件名应与 rule_code 保持可读对应关系。
RuleEngine 不写具体规则逻辑。
RuleEngine 不使用大型 if / elif / switch 分发 rule_code。
RiskRuleRegistry 负责 rule_code → plugin 的注册与查找。
```

多个 rule plugin 可以共用 shared calculator / helper。

共享代码边界：

```text
shared calculator / helper 可以提供 Decimal、数量、价格、notional、symbol rule 等通用计算。
shared calculator / helper 不能承载具体 rule_code 判断。
shared calculator / helper 不能写库。
shared calculator / helper 不能访问外部服务。
shared calculator / helper 不能写 AlertEvent。
```

### 22.2 中文风控规则说明文档要求

RiskCheck implementation 必须维护：

```text
docs/implementation/risk_check/
```

每个 P0 RiskRule plugin 都必须有对应中文说明文档。

中文说明至少包括：

```text
基本信息
中文名
用途
输入对象
依赖字段
适用 market_type
检查逻辑
ALLOW / DENY / BLOCKED / FAILED 判定边界
reason_code
AlertEvent 触发规则
dry-run 行为
关键失败条件
交易语义边界
```

规则：

```text
中文说明文档只解释风控规则和边界。
不得写真实下单逻辑。
不得写 Binance 交易 API 调用逻辑。
不得把 RiskCheck 写成订单修改模块。
不得把 shared calculator 写成 rule registry。
```

如果新增 P0 RiskRuleDefinition 或新增 plugin，必须同步新增或更新对应中文说明文档。


---

## 23. P0 建议规则插件

P0 插件建议包括：

```text
candidate_intent_valid
order_plan_valid
order_components_valid
binance_sync_run_consumable
snapshot_integrity
market_identity_consistency
one_way_position_mode_required
active_order_guard
price_snapshot_present
price_snapshot_fresh_for_risk_check
usds_m_balance_available
coin_m_balance_available
symbol_rule_min_notional
symbol_rule_quantity_step
symbol_rule_max_quantity
symbol_rule_max_notional
available_margin_check
reverse_fallback_reduce_only
```

`active_order_guard` 的检查范围必须与 OrderPlan 保持一致。

同一：

```text
symbol
market_type
account_domain
```

下如果存在未完成的：

```text
OrderPlan
CandidateOrderIntent
ApprovedOrderIntent
ExchangeOrder
```

其中 `ExchangeOrder` 指交易所订单状态记录。若当前阶段尚未实现 ExchangeOrder，RiskCheck 只能预留 selector hook；后续 ExchangeOrder 可用后，active_order_guard 必须检查未完成 ExchangeOrder / 交易所未完成订单状态，当前阶段不得越界创建 ExchangeOrder 模型。

RiskCheck 必须：

```text
RiskCheckResult.status = BLOCKED
reason_code = active_order_exists
allows_downstream = false
写 AlertEvent
```

未完成状态的具体枚举由对应模型定义，但至少应覆盖：

```text
created
pending_risk_check
approved
pending_execution_preparation
pending_execution
submitted
partially_filled
```

已终结状态可包括：

```text
no_order_required
denied
blocked
failed
completed
filled
cancelled
expired
```

以下动作合同插件不得作为 P0 直接使用：

```text
decision_action_allowed
position_conflict
exit_position_exists
```

以下插件不属于 P0 RiskRulePlugin，不得进入 P0 active registry：
decision_action_allowed
position_conflict
exit_position_exists


---

## 24. RiskCheckResult 输出字段

建议字段：

```text
risk_check_key
status
is_usable
allows_downstream
selected_candidate_order_intent_id
selected_intent_type
order_plan_id
primary_candidate_order_intent_id
fallback_candidate_order_intent_id
binance_sync_run_id
binance_snapshot_set_hash
account_snapshot_id
balance_snapshot_ids
position_snapshot_id
symbol_rule_snapshot_id
price_snapshot_id
blocked_reason
deny_reason_code
deny_reason_text_zh
error_code
error_message
rule_set_hash
checked_rules
risk_measures
risk_config_snapshot
input_snapshot
risk_snapshot
evidence_items
evidence_text_zh
order_components_evidence
alert_event_ids
trace_id
trigger_source
created_at
updated_at
```

其中：

```text
checked_rules
= 本次执行的所有规则及其结果。

risk_measures
= 保证金占用、余额使用、交易所限制、数量规则等风险度量。

input_snapshot
= 本次读取的 CandidateOrderIntent、OrderPlan、BinanceSyncRun、PriceSnapshot、规则版本和配置摘要。

risk_snapshot
= 本次计算得出的保证金、notional、拒绝/阻断证据、fallback 选择证据。
```

`risk_measures` 至少建议包含：

```text
current_equity
available_balance
order_notional
requested_size
margin_required_total
margin_required_by_component
observed_exchange_leverage
estimated_leverage_after_order
is_risk_reducing_total
has_increase_risk_component
price_snapshot_id
mark_price
market_type
margin_asset
```

COIN-M 还应包含：

```text
contract_size
current_equity_native
current_equity_usd_value
margin_required_native
available_balance_native
```

这些字段用于解释为什么本次 RiskCheck 是 ALLOW / DENY / BLOCKED / FAILED，也用于后续复盘。


---

## 25. 幂等要求

RiskCheckResult 必须幂等。

幂等 key 至少基于：

```text
candidate_order_intent_id
order_plan_id
binance_sync_run_id
binance_snapshot_set_hash
selected_account_snapshot_hash
selected_position_snapshot_hash
selected_symbol_rule_snapshot_hash
price_snapshot_id
price_snapshot_hash
rule_set_hash
risk_config_hash
risk_schema_version
```

相同输入重复执行：

```text
返回已有 RiskCheckResult
不得重复写入等价结果
不得重复生成等价 ApprovedOrderIntent
不得重复发送等价正式 AlertEvent
```

如果使用新的 BinanceSyncRun、PriceSnapshot、规则版本或风险配置，应生成新的幂等 key。

`price_snapshot_id` 必须是全局唯一且可追溯的价格快照标识。价格、时间或来源变化生成新的 PriceSnapshot 时，应产生新的 `price_snapshot_id`。

同一个 CandidateOrderIntent 使用不同 PriceSnapshot 重新执行 RiskCheck 时，可以生成新的 RiskCheckResult，因为价格变化可能导致 notional、保证金和风控结果不同。


---

## 26. dry-run 要求

dry-run 时：

```text
不写 RiskCheckResult
不写 RiskRuleResult / RiskCheckIssue
不写 AlertEvent
不生成 ApprovedOrderIntent
不修改数据库
返回风控摘要
```

dry-run 仍必须执行与正式流程一致的输入校验、规则校验和风险计算，只是不落库。

---

## 27. ApprovedOrderIntent 边界

ApprovedOrderIntent 是：

```text
OrderPlan 生成 CandidateOrderIntent
RiskCheck 检查通过
形成的风控通过订单意图
```

它表示：

```text
这笔订单意图已经通过风控，可以进入 ExecutionPreparation。
```

它不表示：

```text
已经真实下单
已经成交
一定会成交
```

RiskCheckResult.status = ALLOW 时，RiskCheckService 必须生成或等价确认 ApprovedOrderIntent。
RiskCheckResult.status in DENY / BLOCKED / FAILED 时，不得生成 ApprovedOrderIntent。
ApprovedOrderIntent 的核心订单内容必须来自被批准的 CandidateOrderIntent。

---

## 28. P0 检查清单

RiskCheck P0 至少检查：

```text
CandidateOrderIntent 有效
OrderPlan 有效
order_components 结构合法
BinanceSyncRun 可消费
账户快照存在
余额快照存在
持仓快照存在
symbol rule 快照存在
PriceSnapshot 存在
PriceSnapshot 未超过 RiskCheck 估值 TTL
snapshot_set_hash 可校验
market_type 一致
account_domain 一致
symbol 一致
One-Way Mode
active order 状态一致
availableBalance 足够
U 本位保证金估算
COIN-M 保证金估算
reduce_risk 组件持仓存在且数量不超过当前持仓
reduce_risk 组件满足 minQty / stepSize / minNotional / maxQty 等 symbol rule
increase_risk 组件 observed_exchange_leverage 存在且 > 0
minQty
stepSize
minNotional
maxQty
maxNotional / maxNotionalValue / maxQty
COIN-M contractSize 存在且 > 0
COIN-M marginAsset 可匹配余额资产
COIN-M mark_price
反手 fallback_reduce_only 可用性
```


---

## 29. 非目标

RiskCheck P0 不实现：

```text
真实下单
撤单
查订单
查成交
Binance REST / WebSocket 调用
Binance client 初始化
自动划转
修改 leverage
修改 margin type
修改 position mode
执行前 30 秒 / 0.5% price guard
实时盘口深度
实时滑点判断
任意 MODIFY
自动缩小订单
自动拆单
重新生成 DecisionSnapshot
重新判断策略方向
UI / dashboard
回测收益
参数扫描
保险基金处理
强平处理
```

---

## 30. P1 后续增强项

以下能力记录为 P1，不进入 RiskCheck P0：

```text
任意 MODIFY / 自动缩小订单
部分保留 reduce component
精确模拟 Binance 最大可开 / max_open_capacity calculator
AlertEvent 路由、级别、聚合、限频与降噪
交易冷却期
同一方向开仓间隔
每日最大开仓次数
重复订单意图拦截
当日累计亏损限制
连续亏损暂停
总保证金占用上限
单标集中度限制
组合风险敞口
rebalance_band_bps
max_rebalance_delta_ratio_per_cycle
```


---

## 31. P2 后续能力

以下属于后续高级能力：

```text
组合最大回撤
跨品种相关性风险
重大事件前禁交易
动态风险缩放
根据历史表现调整风险比例
高级组合级风控
交易后滑点、费用、执行质量复盘
参数扫描
策略收益评估
```

---

## 32. 验收标准

实现 RiskCheck 时，至少满足：

```text
1. 不直接消费 DecisionSnapshot.decision_action。
2. 不使用 ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE 作为风控输入合同。
3. 只消费 CandidateOrderIntent / OrderPlan。
4. 不调用 Binance API。
5. 不真实下单。
6. 不撤单。
7. 不修改 Binance leverage / margin type / position mode。
8. BinanceSyncRun 必须显式传入且可消费。
9. 不自动 fallback 到其他 BinanceSyncRun。
10. U 本位和 COIN-M 严格隔离。
11. COIN-M 不复用 U 本位公式。
12. observed_exchange_leverage 不参与系统目标仓位计算。
13. increase_risk 组件缺少 observed_exchange_leverage 时 BLOCKED。
14. reduce_risk 组件不因 observed_exchange_leverage 缺失而阻断。
15. 系统目标订单超过 P0 明确规则可承接范围时 DENY + AlertEvent。
16. P0 不实现精确 Binance 最大可开模拟。
17. 净额反手 opening 不通过时，优先检查 fallback_reduce_only。
18. fallback_reduce_only 不通过时，按原因返回 DENY / BLOCKED / FAILED。
19. 所有正式结果 ALLOW / DENY / BLOCKED / FAILED 均 AlertEvent。
20. dry-run 不写库、不告警。
21. RiskCheckResult 幂等。
22. Rule plugin 架构可扩展、可版本化、可审计。
23. PriceSnapshot 超过 RiskCheck 估值 TTL 时 BLOCKED。
24. reduce_risk 组件同样校验 minQty / stepSize / minNotional 等 symbol rule。
25. COIN-M contractSize 必须存在、可解析且 > 0。
26. risk_measures 至少包含 current_equity、available_balance、order_notional、margin_required_total、estimated_leverage_after_order 等审计字段。
27. U 本位和 COIN-M 使用独立 market calculator，公共中性计算工具可复用。
28. RiskRulePlugin 采用注册制，原则上一个 rule plugin 一个文件。
29. RuleEngine 不写具体规则逻辑，不使用大型 if / elif / switch 分发 rule_code。
30. 多个 rule plugin 可以共用 shared calculator / helper，但 shared calculator 不写库、不访问外部服务、不承载 rule_code 分发。
31. docs/implementation/risk_check/ 中文风控规则说明文档存在。
32. 每个 P0 RiskRule plugin 均有中文说明。
```
