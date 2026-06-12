# 007A OrderPlan 需求说明

> 适用项目：中低频趋势跟踪自动交易系统
> 目标路径：`docs/requirements/order_plan.md`  
> 状态：P0 需求定稿草案  
> 相关上游：`006A DecisionSnapshot`、`006B Binance Account Sync`、`006C PriceSnapshot`  
> 相关下游：`007B RiskCheck`、后续 `ExecutionPreparation / Execution`、`Tracking`

---

## 1. 背景与目标

在正式交易链路中，`DecisionSnapshot` 不直接表达订单动作，也不输出 `ENTER_LONG`、`ENTER_SHORT`、`EXIT` 等非正式订单动作枚举。

正式链路为：

```text
006A DecisionSnapshot
→ 006B Binance Account Sync / 006C PriceSnapshot
→ 007A OrderPlan
→ CandidateOrderIntent
→ 007B RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation / Execution
→ Tracking
```

其中：

```text
DecisionSnapshot：表达本周期目标仓位意图。
OrderPlan：把目标仓位转换成候选订单意图。
RiskCheck：审批、拒绝或阻断候选订单。
ExecutionPreparation / Execution：执行前检查与真实下单。
Tracking：跟踪订单状态、成交、部分成交、持仓变化。
```

OrderPlan 的核心目标是：

```text
把 target_position_ratio 转换为可审计、可风控、但不可直接执行的 CandidateOrderIntent。
```

OrderPlan 不是策略层，不是风控层，也不是执行层。

---

## 2. 上游 DecisionSnapshot 合约

### 2.1 DecisionSnapshot 不输出订单动作

006A 不输出非正式订单动作枚举：

```text
ENTER_LONG
ENTER_SHORT
EXIT
HOLD
NO_TRADE
```

006A 输出的是目标仓位意图，例如：

```text
target_intent
target_position_ratio
decision_policy_name
decision_policy_version
evidence_summary
```

### 2.2 target_position_ratio 含义

`target_position_ratio` 表示当前决策周期的目标总仓位比例，不是新增下单比例。

```text
+1.0  = 当前周期目标满仓做多
+0.5  = 当前周期目标半仓做多
 0.0  = 当前周期目标空仓
-0.5  = 当前周期目标半仓做空
-1.0  = 当前周期目标满仓做空
```

例如：

```text
上一轮已经是半仓；
新一轮 DecisionSnapshot 仍然给出 +0.5；
OrderPlan 仍然必须按当前权益、当前价格、当前持仓重新计算“当前半仓”。
```

### 2.3 DecisionSnapshot 如何得到 target_position_ratio

DecisionSnapshot 不应凭空生成目标仓位，而应基于：

```text
StrategySignal
DecisionPolicy
```

例如：

```text
StrategySignal.direction = bullish
StrategySignal.strength >= 某阈值
StrategySignal.confidence >= 某阈值
→ DecisionPolicy 映射为 target_position_ratio = +0.5 或 +1.0
```

该映射属于 006A 的 `DecisionPolicy`，不是 OrderPlan 的职责。

OrderPlan 只消费已经生成并通过质量校验的 `DecisionSnapshot.target_position_ratio`。

---

## 3. P0 支持范围

### 3.1 Binance One-Way Mode

OrderPlan P0 只支持 Binance One-Way Mode：

```text
position_mode = one_way
positionSide = BOTH
```

如果检测到 Hedge Mode：

```text
OrderPlan.status = blocked
blocked_reason = hedge_mode_not_supported
allows_downstream = false
```

原因：

```text
One-Way Mode 下可以用一笔净额订单完成仓位反转；
Hedge Mode 下 LONG / SHORT 独立，需要拆成独立平仓腿和开仓腿，P0 暂不支持。
```

### 3.2 支持 market_type

OrderPlan P0 必须完整支持：

```text
usds_m_futures
coin_m_futures
```

不得把 COIN-M 留成 unsupported，也不得复用 U 本位公式处理 COIN-M。

必须通过 calculator / registry 路由：

```text
usds_m_futures → USDⓈ-M calculator
coin_m_futures → COIN-M calculator
```

如果 COIN-M 缺少关键字段，必须 fail-closed，不能用 U 本位公式兜底。

---

## 4. 核心仓位参数

### 4.1 target_notional_basis

P0 采用：

```text
target_notional_basis = current_equity
```

含义：

```text
每一轮目标仓位都按当前账户权益重新计算。
```

不使用：

```text
initial_equity
realized_equity
历史半仓数量
```

结果：

```text
盈利后，目标仓位随当前权益变大；
亏损后，目标仓位随当前权益变小。
```

### 4.2 max_target_notional_to_equity_ratio

字段名采用：

```text
max_target_notional_to_equity_ratio
```


含义：

```text
当 target_position_ratio = ±1.0 时，
目标名义仓位 = current_equity 的多少倍。
```

P0 暂定：

```text
max_target_notional_to_equity_ratio = 3.0
```

它不是 Binance 交易所杠杆设置，也不是实际下单杠杆。它只是系统内部的目标名义仓位上限比例。

公式：

```text
target_notional =
current_equity
* max_target_notional_to_equity_ratio
* abs(target_position_ratio)
```

实际目标名义权益比为：

```text
target_notional_to_equity_ratio =
max_target_notional_to_equity_ratio
* abs(target_position_ratio)
```

例子：

```text
current_equity = 100
max_target_notional_to_equity_ratio = 3.0
target_position_ratio = +0.5
```

则：

```text
target_notional = 100 * 3.0 * 0.5 = 150
target_notional_to_equity_ratio = 1.5
```

也就是说，半仓时目标名义权益比是 1.5 倍，不是 3 倍。

### 4.3 不引入 P0 杠杆校验字段

OrderPlan P0 不引入以下字段作为核心规则：

```text
configured_exchange_leverage
observed_exchange_leverage
risk_target_effective_leverage
risk_hard_max_effective_leverage
```

原因：

```text
系统仓位目标由 max_target_notional_to_equity_ratio 控制；
Binance 交易所杠杆设置不等于系统目标仓位比例；
effective_leverage 可作为审计字段，但不作为 P0 阻断规则。
```

Account Sync 可以保存 Binance 返回的 `leverage` 字段用于审计或后续风控，但 OrderPlan P0 不要求：

```text
observed_exchange_leverage == configured_exchange_leverage
```

也不得因为：

```text
半仓实际名义权益比 1.5 != Binance 设置 3x
```

而阻断。

---

## 5. current_equity 字段映射

### 5.1 current_equity 的定义

`current_equity` 表示当前权益，应包含未实现盈亏。

因此，P0 不使用 `walletBalance` 作为 `current_equity`，而使用 `marginBalance` 口径。

### 5.2 USDⓈ-M 映射

U 本位使用 quote asset 权益，例如 USDT / USDC。

```text
current_equity_quote = totalMarginBalance
或对应 asset.marginBalance

available_balance_quote = availableBalance
或对应 asset.availableBalance
```

`current_equity_quote` 用于 OrderPlan 计算目标仓位。

`available_balance_quote` 用于 RiskCheck 判断新增风险部分的可用保证金。

### 5.3 COIN-M 映射

COIN-M 使用 settlement asset 权益，例如 BTC / ETH。

```text
current_equity_native = assets[marginAsset].marginBalance
available_balance_native = assets[marginAsset].availableBalance
```

`current_equity_native` 用于 OrderPlan 计算目标合约张数。

`available_balance_native` 用于 RiskCheck 判断新增风险部分的可用保证金。

---

## 6. PriceSnapshot 定位

### 6.1 PriceSnapshot 不是最终下单价格

PriceSnapshot 在 OrderPlan / RiskCheck 中只解决两个问题：

```text
1. 估值：
   target_notional 如何换成 quantity / contracts。

2. 审计一致性：
   本次 OrderPlan / RiskCheck 绑定的是哪个价格快照。
```

PriceSnapshot 不是最终下单价格。

真实下单价格、市价单保护、限价价格、成交价格，都属于后续 ExecutionPreparation / Execution。

### 6.2 OrderPlan 绑定价格快照

OrderPlan 必须记录：

```text
price_snapshot_id
price_source
mark_price
price_time
market_type
symbol / pair
```

### 6.3 30 秒与 0.5% price guard 不属于 OrderPlan

Execution 前 price guard 属于后续 ExecutionPreparation。

默认值暂定：

```text
latest_price_snapshot_age <= 30 秒
price_deviation <= 0.5%
```

注意：

```text
30 秒不是指 OrderPlan 生成后 30 秒内必须完成下单。
30 秒只约束 ExecutionPreparation 真正准备发单前读取到的 latest PriceSnapshot。
```

如果 ExecutionPreparation 发现最新价格快照超过 30 秒，应先尝试刷新最新价格。

```text
能拿到最新价格，且偏离 <= 0.5% → 可以继续执行。
能拿到最新价格，但偏离 > 0.5% → 不直接下单，要求重新 RiskCheck。
拿不到最新价格 → ExecutionPreparation blocked，reason_code = fresh_price_unavailable。
```

这些细节在后续 ExecutionPreparation 文档展开，OrderPlan 只记录边界。

---

## 7. USDⓈ-M 目标仓位换算

### 7.1 单位口径

U 本位口径：

```text
权益单位：USDT / USDC
目标名义：USDT / USDC
持仓单位：base asset quantity，例如 BTC quantity
订单单位：base asset quantity，例如 BTC quantity
PnL 单位：USDT / USDC
```

### 7.2 目标名义

```text
target_notional_quote =
current_equity_quote
* max_target_notional_to_equity_ratio
* abs(target_position_ratio)
```

### 7.3 目标数量

```text
target_abs_quantity = target_notional_quote / mark_price
```

```text
target_signed_quantity =
sign(target_position_ratio) * target_abs_quantity
```

### 7.4 当前持仓名义

估值时使用绝对值：

```text
current_position_notional_quote =
abs(current_signed_quantity) * mark_price
```

### 7.5 订单差值

```text
delta_quantity =
target_signed_quantity - current_signed_quantity
```

```text
requested_size = abs(delta_quantity)
```

方向：

```text
delta_quantity > 0 → BUY
delta_quantity < 0 → SELL
delta_quantity = 0 → no_order_required
```

---

## 8. COIN-M 目标仓位换算

### 8.1 单位口径

COIN-M 口径：

```text
权益单位：settlement asset，例如 BTC / ETH
目标名义：USD
持仓单位：contracts
订单单位：contracts
PnL 单位：settlement asset，例如 BTC
```

### 8.2 COIN-M mark_price 单位

COIN-M 的 `mark_price` 表示该 symbol / pair 的 USD 标记价格。

例如：

```text
BTCUSD_PERP 的 mark_price 表示 BTC 的 USD 标记价格。
```

注意：

```text
mark_price 是市场价格；
contractSize 是每张合约对应的固定 USD 面值；
二者不能混淆。
```

不得把 `mark_price` 理解为每张合约面值。

不得把 `contractSize` 理解为当前市场价格。

### 8.3 contractSize 来源

COIN-M 必须使用 `contractSize`。

来源：

```text
BinanceSymbolRuleSnapshot.contract_size
或 raw_payload.symbols[].contractSize
```

建议 Account Sync / Symbol Rule Sync 将以下字段规范化落库：

```text
contract_size
margin_asset
base_asset
quote_asset
```

如果缺少 `contractSize`：

```text
OrderPlan.status = blocked
blocked_reason = coin_m_contract_size_missing
allows_downstream = false
```

### 8.4 当前权益折算为 USD 价值

由于 COIN-M 账户权益是 settlement asset，而合约张数按 USD 面值计算，因此必须先折算：

```text
current_equity_usd_value =
current_equity_native * mark_price
```

也可以在实现中合并计算，但文档必须保留单位转换语义。

### 8.5 目标 USD 名义

```text
target_notional_usd =
current_equity_usd_value
* max_target_notional_to_equity_ratio
* abs(target_position_ratio)
```

### 8.6 目标合约张数

```text
target_abs_contracts =
target_notional_usd / contractSize
```

```text
target_signed_contracts =
sign(target_position_ratio) * target_abs_contracts
```

### 8.7 当前持仓名义

```text
current_position_notional_usd =
abs(current_signed_contracts) * contractSize
```

### 8.8 订单差值

```text
delta_contracts =
target_signed_contracts - current_signed_contracts
```

```text
requested_size = abs(delta_contracts)
```

方向：

```text
delta_contracts > 0 → BUY
delta_contracts < 0 → SELL
delta_contracts = 0 → no_order_required
```

### 8.9 COIN-M 不得复用 U 本位 PnL 公式

OrderPlan / RiskCheck P0 不主动重算 COIN-M PnL，应优先使用 Binance Account Sync 快照字段：

```text
walletBalance
marginBalance
availableBalance
unRealizedProfit
positionAmt
entryPrice
markPrice
contractSize
```

COIN-M PnL 公式只作为禁止误用 U 本位线性公式的约束和测试说明。

U 本位近似收益口径：

```text
PNL_quote = position_quantity * (mark_price - entry_price)
```

COIN-M 是反向合约，PnL 原生单位是 settlement asset：

```text
PNL_native =
contracts
* contractSize
* direction
* (1 / entry_price - 1 / mark_price)
```

不得用 U 本位线性公式推导 COIN-M 权益。

---

## 9. 最小调仓金额

P0 采用：

```text
min_rebalance_notional = 20
```

含义：

```text
如果本轮理论调仓名义金额 < 20，则不生成订单。
```

U 本位：

```text
20 USDT / USDC
```

COIN-M：

```text
20 USD contract notional
```

结果：

```text
OrderPlan.status = no_order_required
reason_code = below_min_rebalance_notional
```

P0 暂不实现：

```text
rebalance_band_bps
max_rebalance_delta_ratio_per_cycle
```

原因：

```text
P0 先完整执行 DecisionSnapshot 的目标仓位；
min_rebalance_notional 已能解决小额调仓手续费浪费问题；
比例 band 和分批调仓留到 P1。
```

---

## 10. 仓位转换与 order_components

### 10.1 为什么需要 order_components

一笔 Binance One-Way 订单可能同时包含：

```text
降低风险部分
新增风险部分
```

例如净额反手：

```text
当前 long +0.8
目标 short -0.5
OrderPlan 生成 SELL 1.3
```

真实 Binance 订单是一笔：

```text
side = SELL
quantity = 1.3
positionSide = BOTH
reduceOnly = false
```

但其内部风险语义是：

```text
close_long 0.8  → 降低风险
open_short 0.5 → 新增风险
```

因此 CandidateOrderIntent 必须包含：

```text
order_components
```

`order_components` 是内部风控组件，不是 Binance 子订单。

### 10.2 order_components 字段

每个 component 至少包含：

```text
component_index
component_type
position_effect
side
size
notional
risk_effect
is_risk_reducing
```

字段含义：

```text
component_type：
  close_existing_position
  open_new_position
  increase_existing_position
  reduce_existing_position

position_effect：
  open_long
  open_short
  increase_long
  increase_short
  reduce_long
  reduce_short
  close_long
  close_short

risk_effect：
  reduce_risk
  increase_risk

is_risk_reducing：
  true / false
```

不要在 component 上使用 `reduceOnly` 字段。

原因：

```text
reduceOnly 是 Binance 真实订单参数；
Binance 不支持一笔订单内部不同组件拥有不同 reduceOnly；
组件只能表达内部风控语义。
```

### 10.3 空仓开多 / 开空

```text
current = 0
target = +0.5
```

结果：

```text
side = BUY
requested_size = 0.5
exchange_reduce_only = false
plan_type = open_long
```

组件：

```text
opening_size = 0.5
closing_size = 0
is_risk_reducing = false
```

### 10.4 同向加仓

```text
current = +0.3
target = +0.8
```

结果：

```text
side = BUY
requested_size = 0.5
exchange_reduce_only = false
plan_type = increase_long
```

组件：

```text
increase_long 0.5
risk_effect = increase_risk
```

### 10.5 同向减仓

```text
current = +0.8
target = +0.3
```

结果：

```text
side = SELL
requested_size = 0.5
exchange_reduce_only = true
plan_type = reduce_long
```

组件：

```text
reduce_long 0.5
risk_effect = reduce_risk
```

### 10.6 平仓到空仓

```text
current = +0.8
target = 0
```

结果：

```text
side = SELL
requested_size = 0.8
exchange_reduce_only = true
plan_type = close_long
```

组件：

```text
close_long 0.8
risk_effect = reduce_risk
```

### 10.7 净额反手：多转空

```text
current = +0.8
target = -0.5
```

通用差值：

```text
delta = -0.5 - (+0.8) = -1.3
requested_size = 1.3
```

真实 Binance One-Way 订单：

```text
side = SELL
requested_size = 1.3
positionSide = BOTH
exchange_reduce_only = false
plan_type = netting_reverse_long_to_short
```

内部组件：

```text
component 1:
  position_effect = close_long
  size = 0.8
  risk_effect = reduce_risk
  is_risk_reducing = true

component 2:
  position_effect = open_short
  size = 0.5
  risk_effect = increase_risk
  is_risk_reducing = false
```

### 10.8 净额反手：空转多

```text
current = -0.8
target = +0.5
```

真实 Binance One-Way 订单：

```text
side = BUY
requested_size = 1.3
positionSide = BOTH
exchange_reduce_only = false
plan_type = netting_reverse_short_to_long
```

内部组件：

```text
component 1:
  position_effect = close_short
  size = 0.8
  risk_effect = reduce_risk
  is_risk_reducing = true

component 2:
  position_effect = open_long
  size = 0.5
  risk_effect = increase_risk
  is_risk_reducing = false
```

---

## 11. CandidateOrderIntent

OrderPlan 输出的是 CandidateOrderIntent。

CandidateOrderIntent 不是可执行订单。

P0 状态：

```text
CandidateOrderIntent.status = pending_risk_check
```

只有 RiskCheck 通过后，才生成：

```text
ApprovedOrderIntent
```

只有 ApprovedOrderIntent 才能进入 ExecutionPreparation。

### 11.1 CandidateOrderIntent 关键字段

CandidateOrderIntent 至少包含：

```text
order_plan_id
symbol
market_type
account_domain
position_mode
plan_type
side
positionSide
exchange_reduce_only
requested_size
requested_notional
requested_size_unit
price_snapshot_id
mark_price
current_position_signed_size
target_position_signed_size
delta_signed_size
closing_size
opening_size
order_components
status
reason_code
evidence
```

其中：

```text
requested_size_unit：
  quantity        # U 本位，例如 BTC quantity
  contracts       # COIN-M contracts
```

`closing_size` 与 `opening_size` 是摘要字段。

真正用于 RiskCheck 分段评估的是：

```text
order_components
```

### 11.2 真实订单参数与内部组件关系

CandidateOrderIntent 中的：

```text
side
requested_size
positionSide
exchange_reduce_only
```

表示未来可能提交给 Binance 的真实订单参数。

`order_components` 只表示内部风控语义，不表示 Binance 子订单。

---

## 12. RiskCheck 边界

### 12.1 RiskCheck 不直接检查 DecisionSnapshot

RiskCheck 检查对象是：

```text
CandidateOrderIntent
OrderPlan
当前账户快照
当前持仓快照
当前价格快照
symbol rule
available balance
margin requirement
exchange max notional / max quantity
```

### 12.2 RiskCheck P0 不实现 MODIFY

P0 不实现：
    MODIFY

如果风控不通过：
    结构 / 前置条件问题 → BLOCKED
    风险约束不允许 → DENY
    系统异常 → FAILED

RiskCheck P0 不缩小订单，不拆单，不自行生成新订单。

唯一例外：
净额反手场景中，OrderPlan 可以预生成 fallback_reduce_only_intent。
RiskCheck 不修改 primary_intent，但可以在 primary_intent 不通过、fallback_reduce_only_intent 通过时，选择该预生成 fallback。

也就是说：
    不允许 RiskCheck 把 SELL 1.3 自动改成 SELL 0.8；
    但允许 OrderPlan 事先生成一个独立的 fallback_reduce_only_intent = SELL 0.8；
    RiskCheck 只能在两个已存在的候选意图中选择，不得临时改数量、拆单或造单。

### 12.3 closing / opening components 的用途

即使 RiskCheck P0 整体拒绝，仍然必须使用 `order_components` 分段评估。

降低风险组件：

```text
校验当前持仓是否存在；
校验 size 是否不超过当前持仓；
识别为 reduce_risk；
不按新增仓位计算保证金。
```

新增风险组件：

```text
校验新增仓位保证金；
校验 availableBalance；
校验交易所最大数量 / 最大名义；
校验 symbol rule。
```

最终输出仍然是整体：

```text
ALLOW
DENY
BLOCKED
FAILED
```

---

## 13. U 本位与 COIN-M 风控边界

本文件只定义 OrderPlan 与 RiskCheck 的边界，RiskCheck 详细规则由 `docs/requirements/risk_check.md` 维护。

但 P0 必须遵守以下边界：

### 13.1 U 本位保证金口径

U 本位新增风险部分使用 quote asset 余额。

```text
available_balance_quote >= margin_required_quote
```

### 13.2 COIN-M 保证金口径

COIN-M 不能把 USD 名义保证金直接和 BTC / ETH 可用余额比较。

必须换算到 settlement asset：

```text
margin_required_native =
opening_notional_usd / mark_price / exchange_leverage_snapshot
```

然后比较：

```text
available_balance_native >= margin_required_native
```

注意：

```text
exchange_leverage_snapshot 可以作为保证金估算输入；
但 OrderPlan P0 不引入 configured_exchange_leverage；
RiskCheck P0 不要求 observed_exchange_leverage == configured_exchange_leverage。
```

如果缺少必要字段：

```text
available_balance_native
mark_price
contractSize
exchange_leverage_snapshot（若 RiskCheck 需要用于保证金估算）
```

必须 fail-closed。

---

## 14. Active Order 处理

如果同一：

```text
symbol
market_type
account_domain
```

已经存在未完成的：

```text
OrderPlan
CandidateOrderIntent
ApprovedOrderIntent
ExecutionOrder
```

新的 OrderPlan 必须：

```text
status = blocked
blocked_reason = active_order_exists
allows_downstream = false
```

OrderPlan 不负责撤单。

撤单、订单状态同步、部分成交处理属于：

```text
后续 Execution
后续 Tracking
```

---

## 15. 幂等性

相同输入应生成相同 OrderPlan。

OrderPlan 幂等 key 至少包含：

```text
decision_snapshot_id
binance_sync_run_id
symbol
market_type
account_domain
position_mode
current_position_snapshot_id
account_snapshot_id
price_snapshot_id 或 reference_price
target_position_ratio
target_notional_basis
max_target_notional_to_equity_ratio
min_rebalance_notional
```

重复运行：

```text
相同输入 → 返回已有 OrderPlan
输入冲突 → blocked / failed
```

---

## 16. OrderPlan 状态

OrderPlan P0 状态：

```text
created
no_order_required
blocked
failed
```

含义：

```text
created：已生成 CandidateOrderIntent，等待 RiskCheck。
no_order_required：目标差值小于最小调仓金额，或当前已经满足目标。
blocked：前置条件不满足，例如 Hedge Mode、active_order_exists、缺关键字段。
failed：系统异常。
```

---

## 17. OrderPlan 不做的事情

OrderPlan 不允许做：

```text
真实下单
撤单
查订单
查成交
连接 Binance REST
连接 Binance WebSocket
修改杠杆
修改保证金模式
最终风控审批
进入执行队列
```

OrderPlan 只负责：

```text
把 DecisionSnapshot 的目标仓位转换为可审计的 CandidateOrderIntent。
```

---

## 18. P0 配置汇总

```text
position_mode = one_way

supported_market_types =
  - usds_m_futures
  - coin_m_futures

target_notional_basis = current_equity

max_target_notional_to_equity_ratio = 3.0

min_rebalance_notional = 20

rebalance_band_bps = P1，不实现

max_rebalance_delta_ratio_per_cycle = P1，不实现

configured_exchange_leverage = 不进入 OrderPlan P0

observed_exchange_leverage = 可作为 Account Sync 快照字段，不作为 OrderPlan P0 规则

effective_leverage = 可作为审计字段，不作为 OrderPlan P0 阻断规则

RiskCheck MODIFY = P1，不实现

Execution price guard = 后续 ExecutionPreparation 负责，默认 30 秒 / 0.5%
```

---

## 19. 验收测试要求

### 19.1 USDⓈ-M 测试

必须覆盖：

```text
current_equity_quote → target_notional_quote → target_quantity
空仓开多
空仓开空
同向加仓
同向减仓
平仓到空仓
多转空净额反手
空转多净额反手
min_rebalance_notional = 20
CandidateOrderIntent pending_risk_check
order_components 生成
```

### 19.2 COIN-M 测试

必须覆盖：

```text
current_equity_native * mark_price → current_equity_usd_value
target_notional_usd → target_contracts
contractSize 缺失 blocked
mark_price 缺失 blocked
marginAsset 余额缺失 blocked
空仓开多
空仓开空
同向加仓
同向减仓
平仓到空仓
多转空净额反手
空转多净额反手
min_rebalance_notional = 20 USD contract notional
order_components 生成
不得复用 U 本位线性 PnL / 数量公式
```

### 19.3 RiskCheck 边界测试

必须覆盖：

```text
RiskCheck P0 不使用 MODIFY
RiskCheck 不临时缩单、不拆单、不自行生成 fallback
净额反手 primary_intent opening component 不通过时：
    如果没有 fallback_reduce_only_intent → primary 整体 DENY
    如果存在 fallback_reduce_only_intent 且其 reduce_risk 校验通过 → ALLOW fallback_reduce_only_intent
    如果 fallback_reduce_only_intent 也不通过 → DENY / BLOCKED
净额反手 closing component 不被当作新增风险保证金计算
CandidateOrderIntent 不是可执行订单
ApprovedOrderIntent 才能进入 ExecutionPreparation
```

### 19.4 PriceSnapshot 边界测试

必须覆盖：

```text
OrderPlan 绑定 price_snapshot_id
OrderPlan 不执行 30 秒 price guard
ExecutionPreparation price guard 不属于 OrderPlan P0
```

---

## 20. 官方字段依据

以下为字段语义依据，实际实现仍应以项目内同步快照模型为准：

- USDⓈ-M Account V3：`totalMarginBalance`、`availableBalance`、asset `marginBalance`、asset `availableBalance`。
- COIN-M Account：assets 中包含 `walletBalance`、`unrealizedProfit`、`marginBalance`、`availableBalance`。
- COIN-M Exchange Information：symbols 中包含 `contractSize`、`quoteAsset`、`baseAsset`、`marginAsset`。
- COIN-M Mark Price：`markPrice` 表示 symbol / pair 的标记价格，返回 `time`。

参考：

```text
https://developers.binance.com/docs/derivatives/usds-margined-futures/account/rest-api/Account-Information-V3
https://developers.binance.com/docs/derivatives/coin-margined-futures/account/rest-api/Account-Information
https://developers.binance.com/docs/derivatives/coin-margined-futures/market-data/rest-api/Exchange-Information
https://developers.binance.com/docs/derivatives/coin-margined-futures/market-data/rest-api/Index-Price-and-Mark-Price
```
