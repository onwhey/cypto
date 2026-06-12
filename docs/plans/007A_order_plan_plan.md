# 007A OrderPlan 实现计划

> 目标路径：`docs/plans/007A_order_plan_plan.md`  
> 对应需求：`docs/requirements/order_plan.md`  
> 模块编号：007A  
> 当前状态：待实现  
> 上游：`006A DecisionSnapshot`、`006B Binance Account Sync`、`006C PriceSnapshot`  
> 下游：`007B RiskCheck`  

---

## 0. 本计划的合同边界

本计划实现 `007A OrderPlan`。

OrderPlan 的唯一职责是：

```text
把 006A DecisionSnapshot 输出的目标仓位
结合当前账户、当前持仓、symbol rule、估值价格
转换成可审计的 CandidateOrderIntent。
```

正式主链路为：

```text
006A DecisionSnapshot
→ 007A OrderPlan
→ CandidateOrderIntent
→ 007B RiskCheck
→ ApprovedOrderIntent
→ 后续 ExecutionPreparation / Execution
→ 后续 Tracking
```

OrderPlan 不负责：

```text
策略判断
目标仓位决策
最终风控审批
真实下单
撤单
查订单
查成交
执行前 price guard
```

OrderPlan 不得直接调用 Binance REST / WebSocket。

所有事实必须来自已落库快照或明确传入的测试输入。

---

## 1. 实现前必须阅读

Codex 实现前必须完整阅读：

```text
AGENTS.md
README.md
docs/rules/project_invariants.md

docs/requirements/decision_snapshot.md
docs/plans/006A_decision_snapshot_plan.md

docs/requirements/binance_account_sync.md
docs/plans/006B_binance_account_sync_plan.md

docs/requirements/order_plan.md
docs/requirements/risk_check.md
docs/plans/007B_risk_check_plan.md
```

并检查当前代码：

```text
apps/decision_snapshot/
apps/binance_account_sync/
apps/risk_check/
apps/notifications/
```

注意：

```text
本任务只实现 007A OrderPlan。
不得在本任务中实现或修改 007B RiskCheck。
不得在本任务中实现 ApprovedOrderIntent 或 ExecutionPreparation。
```

---

## 2. 术语与字段统一

### 2.1 正式字段名

本计划统一使用：

```text
max_target_notional_to_equity_ratio
```

对应 settings：

```text
ORDER_PLAN_MAX_TARGET_NOTIONAL_TO_EQUITY_RATIO
```

含义：

```text
系统内部满仓目标名义 / 当前权益的比例。
```

它不是 Binance 交易所杠杆。

不得使用非正式字段名：

```text
max_target_notional_multiplier
```

如果测试中出现非正式字段名，应改为正式字段名。

### 2.2 observed_exchange_leverage 边界

OrderPlan 不使用 `observed_exchange_leverage` 计算：

```text
系统全仓
系统半仓
target_notional
target_quantity
target_contracts
```

`observed_exchange_leverage` 只属于后续 RiskCheck 的保证金估算与审计输入。

OrderPlan 可以把快照中的 leverage 字段记录进 `input_snapshot` / `evidence`，但不得用它改变目标数量。

---

## 3. 本次实现范围

本次必须实现：

```text
1. 新增 apps/order_plan/ Django app。
2. 新增 OrderPlan 模型。
3. 新增 CandidateOrderIntent 模型。
4. 实现 OrderPlanService。
5. 实现 BinanceSyncRun / account / balance / position / symbol rule / price snapshot selectors。
6. 实现 USDⓈ-M calculator。
7. 实现 COIN-M calculator。
8. 实现 One-Way Mode 净额反手 primary_intent。
9. 实现 fallback_reduce_only_intent。
10. 实现 order_components 语义拆分。
11. 实现 min_rebalance_notional = 20 的 no_order_required。
12. 实现 active_order_guard。
13. 实现幂等 key / hash。
14. 实现 dry-run。
15. 实现 AlertEvent 写入。
16. 实现 management command。
17. 补充模型、service、calculator、command 测试。
```

---

## 4. 本次不得实现

本次不得实现：

```text
真实下单
撤单
查订单
查成交
Binance REST client 初始化
Binance WebSocket
ApprovedOrderIntent 模型或服务
ExecutionPreparation
Execution
007B RiskCheck 实现
RiskCheck MODIFY
自动调整 Binance leverage
自动调整 Binance margin type
自动调整 Binance position mode
Redis
UI / dashboard
回测收益
参数扫描
```

如果需要与尚未实现的下游对象关联，只能预留字段或 selector hook，不得提前实现完整下游模块。

---

## 5. 模块结构

新增目录：

```text
apps/order_plan/
  __init__.py
  apps.py
  constants.py
  models.py
  services.py
  selectors.py
  hashes.py
  alerts.py
  calculators/
    __init__.py
    base.py
    registry.py
    usds_m.py
    coin_m.py
  management/
    __init__.py
    commands/
      __init__.py
      build_order_plan.py
  migrations/
    __init__.py
```

新增测试目录建议：

```text
tests/order_plan/
  test_models.py
  test_hashes.py
  test_selectors.py
  test_service.py
  test_usds_m_calculator.py
  test_coin_m_calculator.py
  test_active_order_guard.py
  test_management_command.py
```

如果项目当前已有不同测试组织方式，遵循项目既有结构。

---

## 6. settings 配置

新增 settings，给出默认值：

```python
ORDER_PLAN_TARGET_NOTIONAL_BASIS = "current_equity"
ORDER_PLAN_MAX_TARGET_NOTIONAL_TO_EQUITY_RATIO = "3.0"
ORDER_PLAN_MIN_REBALANCE_NOTIONAL = "20"
ORDER_PLAN_REQUIRED_POSITION_MODE = "one_way"
ORDER_PLAN_SUPPORTED_MARKET_TYPES = ["usds_m_futures", "coin_m_futures"]
ORDER_PLAN_ACTIVE_ORDER_BLOCK_ENABLED = True
ORDER_PLAN_SCHEMA_VERSION = "v1"
```

说明：

```text
target_notional_basis P0 固定为 current_equity。
max_target_notional_to_equity_ratio P0 默认 3.0。
min_rebalance_notional P0 默认 20。
```

P0 不实现：

```text
rebalance_band_bps
max_rebalance_delta_ratio_per_cycle
configured_exchange_leverage 作为 OrderPlan 规则
observed_exchange_leverage 作为 OrderPlan 规则
risk_target_effective_leverage
risk_hard_max_effective_leverage
```

---

## 7. 数据模型

### 7.1 OrderPlan

新增 `OrderPlan` 模型，用于记录一次从目标仓位到候选订单意图的计划结果。

建议字段：

```text
id
created_at
updated_at
trace_id
schema_version

status
status_reason
is_usable
allows_downstream

exchange
account_domain
market_type
symbol
position_mode

source_decision_snapshot_id
source_decision_snapshot_hash
binance_sync_run_id
binance_snapshot_set_hash
account_snapshot_id
balance_snapshot_ids
position_snapshot_id
symbol_rule_snapshot_id
price_snapshot_id
price_snapshot_hash

current_equity
current_equity_asset
available_balance
available_balance_asset
mark_price
price_source
price_as_of_utc

current_signed_position_size
current_position_size_unit
current_position_notional

target_position_ratio
target_notional_basis
max_target_notional_to_equity_ratio
target_notional
target_signed_position_size
target_position_size_unit

delta_size
delta_notional
min_rebalance_notional

primary_candidate_order_intent_id
fallback_candidate_order_intent_id
candidate_count

order_plan_key
input_hash
plan_hash
input_snapshot
plan_snapshot
evidence_items
evidence_text_zh
alert_event_ids
trigger_source
```

状态建议：

```text
created
= 已生成至少一个 CandidateOrderIntent，可进入 RiskCheck。

no_order_required
= 目标与当前仓位差值低于 min_rebalance_notional，无需生成候选订单。

blocked
= 前置条件缺失或不满足，无法生成订单计划。

failed
= 系统异常。
```

语义：

```text
created → is_usable=True, allows_downstream=True
no_order_required → is_usable=True, allows_downstream=False
blocked / failed → is_usable=False, allows_downstream=False
```

### 7.2 CandidateOrderIntent

新增 `CandidateOrderIntent` 模型。

建议字段：

```text
id
created_at
updated_at
trace_id
schema_version

order_plan_id
intent_type
status
is_usable
allows_risk_check

exchange
account_domain
market_type
symbol
position_mode
position_side
side
reduce_only

requested_size
requested_size_unit
requested_notional
notional_unit
reference_price
price_snapshot_id

opening_size
closing_size
opening_notional
closing_notional
has_increase_risk_component
is_risk_reducing_total

order_components
order_components_hash

candidate_order_intent_key
intent_hash
input_snapshot
evidence_items
evidence_text_zh
alert_event_ids
```

`intent_type` 建议：

```text
primary
fallback_reduce_only
```

`status` 建议：

```text
pending_risk_check
cancelled
expired
consumed
void
```

P0 生成后默认：

```text
status = pending_risk_check
is_usable = True
allows_risk_check = True
```

### 7.3 order_components JSON 结构

`order_components` 是内部风控语义，不是 Binance 子订单。

建议结构：

```json
[
  {
    "component_type": "close_existing_position",
    "position_effect": "close_long",
    "side": "SELL",
    "size": "0.8",
    "size_unit": "quantity",
    "notional": "8000",
    "notional_unit": "USDT",
    "risk_effect": "reduce_risk",
    "is_risk_reducing": true
  }
]
```

组件中不得用 `reduceOnly` 表示局部 Binance 参数。

组件应使用：

```text
risk_effect
is_risk_reducing
position_effect
component_type
```

---

## 8. 上游输入校验

OrderPlanService 正式入口建议：

```python
build_order_plan(
    *,
    decision_snapshot_id: int,
    binance_sync_run_id: int,
    price_snapshot_id: int,
    symbol: str,
    market_type: str,
    account_domain: str,
    trace_id: str | None = None,
    trigger_source: str = "manual",
    dry_run: bool = False,
) -> OrderPlanBuildResult
```

必须校验：

```text
DecisionSnapshot 存在
DecisionSnapshot.status 可消费
DecisionSnapshot.is_usable=True
DecisionSnapshot.allows_downstream=True
DecisionSnapshot 未过期
DecisionSnapshot hash 可校验
DecisionSnapshot 提供 target_intent / target_position_ratio

BinanceSyncRun 存在
BinanceSyncRun.status=succeeded
BinanceSyncRun 未过期
BinanceSyncRun.market_type 匹配
BinanceSyncRun.account_domain 匹配
BinanceSyncRun.snapshot_set_hash 可校验

account / balance / position / symbol rule 快照完整
PriceSnapshot 存在
PriceSnapshot.symbol / market_type / account_domain 匹配
PriceSnapshot.mark_price 存在且 > 0
```

缺失或不匹配：

```text
OrderPlan.status = blocked
allows_downstream = false
写 AlertEvent
```

异常：

```text
OrderPlan.status = failed
allows_downstream = false
写 AlertEvent
```

`dry_run=True` 时不写库、不写 AlertEvent，只返回摘要。

---

## 9. active_order_guard

OrderPlan 必须在生成新 CandidateOrderIntent 前检查 active order / active plan。

检查范围：

```text
同一 symbol
同一 market_type
同一 account_domain
```

检查对象：

```text
OrderPlan
CandidateOrderIntent
ApprovedOrderIntent
ExecutionOrder
```

P0 只检查当前已经实现的模型。

对于尚未实现的 `ApprovedOrderIntent` / `ExecutionOrder`：

```text
预留 selector hook；不得为了 active_order_guard 提前实现下游模块。
```

只要存在未终结状态的对象：

```text
OrderPlan.status = blocked
status_reason = active_order_exists
allows_downstream = false
写 AlertEvent
```

终结状态按对应模型定义判断，例如：

```text
completed
cancelled
expired
denied
failed
void
```

---

## 10. Calculator Registry

实现 calculator registry：

```text
usds_m_futures → UsdsMOrderPlanCalculator
coin_m_futures → CoinMOrderPlanCalculator
```

不得在 `OrderPlanService` 中写大型 if / elif 承载计算细节。

`OrderPlanService` 只负责：

```text
输入校验
上下文构建
calculator 路由
结果落库
AlertEvent
幂等
```

calculator 负责：

```text
读取标准化 context
计算 target position
计算 delta
生成 order_components 草案
返回 plan result
```

---

## 11. USDⓈ-M calculator

### 11.1 单位

```text
权益单位：quote asset，例如 USDT / USDC
目标名义单位：quote asset
持仓单位：base asset quantity，例如 BTC
订单单位：base asset quantity
```

### 11.2 目标仓位计算

```text
current_equity_quote = account/balance snapshot 中的 marginBalance / totalMarginBalance

abs_target_notional_quote =
  current_equity_quote
  * max_target_notional_to_equity_ratio
  * abs(target_position_ratio)

target_direction_sign = sign(target_position_ratio)

target_signed_quantity =
  abs_target_notional_quote / mark_price * target_direction_sign
```

当前持仓：

```text
current_signed_quantity = BinancePositionSnapshot.position_quantity 或 positionAmt 的标准化值
current_position_notional = abs(current_signed_quantity) * mark_price
```

差值：

```text
delta_quantity = target_signed_quantity - current_signed_quantity
delta_notional = abs(delta_quantity) * mark_price
```

如果：

```text
delta_notional < min_rebalance_notional
```

则：

```text
OrderPlan.status = no_order_required
reason_code = below_min_rebalance_notional
不生成 CandidateOrderIntent
```

### 11.3 数量量化

OrderPlan 应使用 `BinanceSymbolRuleSnapshot` 中的 stepSize / quantity precision 对候选订单数量进行确定性量化。

P0 建议：

```text
所有订单数量向不超过目标 delta 的方向取整。
reduce_risk 不得超过当前持仓数量。
```

量化后如果 delta_notional 低于 `min_rebalance_notional`：

```text
no_order_required
```

交易所 minQty / minNotional / maxQty / maxNotional 的最终审批仍由 RiskCheck 完成。

OrderPlan 不因为 symbol rule 风控不通过而自动改成其他数量。

---

## 12. COIN-M calculator

### 12.1 单位

```text
权益单位：settlement asset，例如 BTC / ETH
目标名义单位：USD
持仓单位：contracts
订单单位：contracts
contractSize 单位：USD / contract
mark_price 单位：USD / settlement asset
```

### 12.2 必要字段

COIN-M 必须具备：

```text
contractSize / contract_size
contractSize > 0
mark_price > 0
marginAsset / settlement asset
current_equity_native
available_balance_native
current_position_contracts
```

如果缺少任一关键字段：

```text
OrderPlan.status = blocked
status_reason = coin_m_required_field_missing
写 AlertEvent
```

### 12.3 目标仓位计算

```text
current_equity_usd_value = current_equity_native * mark_price

abs_target_notional_usd =
  current_equity_usd_value
  * max_target_notional_to_equity_ratio
  * abs(target_position_ratio)

target_direction_sign = sign(target_position_ratio)

target_signed_contracts =
  abs_target_notional_usd / contractSize * target_direction_sign
```

当前持仓：

```text
current_signed_contracts = BinancePositionSnapshot.positionAmt 标准化值
current_position_notional_usd = abs(current_signed_contracts) * contractSize
```

差值：

```text
delta_contracts = target_signed_contracts - current_signed_contracts
delta_notional_usd = abs(delta_contracts) * contractSize
```

如果：

```text
delta_notional_usd < min_rebalance_notional
```

则：

```text
OrderPlan.status = no_order_required
reason_code = below_min_rebalance_notional
不生成 CandidateOrderIntent
```

### 12.4 contracts 量化

COIN-M 订单单位是 contracts。

P0 应按 symbol rule 中的合约数量步长量化：

```text
delta_contracts_quantized
```

量化后不得超过目标 delta，也不得让 reduce_risk 超过当前持仓 contracts。

COIN-M 不得使用 U 本位公式：

```text
quantity = notional / price
```

处理 contracts。

---

## 13. 净额订单与 order_components

OrderPlan 根据：

```text
current_signed_position
目标 target_signed_position
delta
```

生成候选订单。

### 13.1 空仓到开仓

```text
current = 0
target = +X
```

生成：

```text
BUY X
reduceOnly = false
components = open_long X increase_risk
```

### 13.2 同向加仓

```text
current = +A
target = +B
B > A
```

生成：

```text
BUY (B-A)
reduceOnly = false
components = increase_long (B-A) increase_risk
```

### 13.3 同向减仓

```text
current = +A
target = +B
0 < B < A
```

生成：

```text
SELL (A-B)
reduceOnly = true
components = reduce_long (A-B) reduce_risk
```

### 13.4 平仓到空仓

```text
current = +A
target = 0
```

生成：

```text
SELL A
reduceOnly = true
components = close_long A reduce_risk
```

### 13.5 多空反手

```text
current = +A
target = -B
```

主候选：

```text
SELL (A+B)
reduceOnly = false
positionSide = BOTH
components = close_long A reduce_risk + open_short B increase_risk
```

fallback：

```text
SELL A
reduceOnly = true
positionSide = BOTH
components = close_long A reduce_risk
```

### 13.6 空多反手

```text
current = -A
target = +B
```

主候选：

```text
BUY (A+B)
reduceOnly = false
positionSide = BOTH
components = close_short A reduce_risk + open_long B increase_risk
```

fallback：

```text
BUY A
reduceOnly = true
positionSide = BOTH
components = close_short A reduce_risk
```

---

## 14. CandidateOrderIntent 生成规则

### 14.1 普通场景

非反手场景只生成一个 `primary` CandidateOrderIntent。

```text
intent_type = primary
status = pending_risk_check
```

### 14.2 反手场景

反手场景必须生成两个 CandidateOrderIntent：

```text
primary
fallback_reduce_only
```

`fallback_reduce_only` 必须引用同一个 `OrderPlan`，并且在 evidence 中说明：

```text
仅在 primary 的 increase_risk component 未通过 RiskCheck 时作为降风险候选。
```

OrderPlan 不决定 fallback 是否通过。

fallback 是否被选中由 007B RiskCheck 决定。

---

## 15. One-Way Mode 校验

P0 只支持：

```text
position_mode = one_way
positionSide = BOTH
```

如果账户快照显示 Hedge Mode，或无法确认 One-Way：

```text
OrderPlan.status = blocked
reason_code = hedge_mode_not_supported
allows_downstream = false
写 AlertEvent
```

OrderPlan 不调用 Binance API 修改 position mode。

---

## 16. PriceSnapshot 边界

OrderPlan 使用 PriceSnapshot 做估值与数量换算。

必须校验：

```text
price_snapshot_id 存在
symbol 匹配
market_type 匹配
account_domain 匹配
mark_price 存在
mark_price > 0
```

OrderPlan 不执行最终下单前 price guard。

这些属于后续 ExecutionPreparation：

```text
latest_price_snapshot_age <= 30 秒
price_deviation <= 0.5%
```

OrderPlan 不连接 WebSocket，不拉 REST ticker。

如果当前项目尚未实现 PriceSnapshot 模型：

```text
本任务不得在 OrderPlan 中临时接 Binance 获取价格；
应先实现或接入最小 PriceSnapshot 模块。
```

---

## 17. AlertEvent 要求

正式 OrderPlan 链路必须写 AlertEvent。

至少以下结果必须写：

```text
OrderPlan.created
OrderPlan.no_order_required
OrderPlan.blocked
OrderPlan.failed
CandidateOrderIntent.primary_created
CandidateOrderIntent.fallback_reduce_only_created
```

原因：

```text
生成候选订单 = 准备进入风控的订单相关事件。
no_order_required = 本轮未形成订单的正式判断结果。
blocked / failed = 交易链路未能继续。
fallback_reduce_only_created = 反手场景存在降风险候选。
```

AlertEvent 至少包含：

```text
order_plan_id
candidate_order_intent_id（如有）
symbol
market_type
account_domain
status
reason_code
side
requested_size
requested_notional
target_position_ratio
current_position
order_components 摘要
trace_id
```

`dry_run=True` 不写 AlertEvent。

---

## 18. 幂等要求

OrderPlan 必须幂等。

`order_plan_key` 至少包含：

```text
decision_snapshot_id
decision_snapshot_hash
binance_sync_run_id
binance_snapshot_set_hash
account_snapshot_id
position_snapshot_id
symbol_rule_snapshot_id
price_snapshot_id
price_snapshot_hash
symbol
market_type
account_domain
target_position_ratio
target_notional_basis
max_target_notional_to_equity_ratio
min_rebalance_notional
schema_version
```

相同输入重复执行：

```text
返回已有 OrderPlan
不得重复生成等价 CandidateOrderIntent
不得重复发送等价 AlertEvent
```

如果更换：

```text
BinanceSyncRun
PriceSnapshot
DecisionSnapshot
配置
schema_version
```

则应生成新的 `order_plan_key`。

---

## 19. dry-run 要求

`dry_run=True` 时：

```text
不写 OrderPlan
不写 CandidateOrderIntent
不写 AlertEvent
不修改数据库
返回完整计划摘要
```

但仍必须执行：

```text
输入校验
快照一致性校验
calculator 计算
active_order_guard 模拟检查
order_components 生成
```

---

## 20. management command

新增命令：

```text
python manage.py build_order_plan \
  --decision-snapshot-id <id> \
  --binance-sync-run-id <id> \
  --price-snapshot-id <id> \
  --symbol BTCUSDT \
  --market-type usds_m_futures \
  --account-domain usds_m_futures
```

参数：

```text
--dry-run
--trace-id
--trigger-source
```

命令行为：

```text
成功：打印 OrderPlan id、status、candidate ids、summary。
blocked / failed：打印 reason_code，并按正式 / dry-run 规则决定是否落库和 alert。
```

---

## 21. 测试要求

### 21.1 模型测试

覆盖：

```text
OrderPlan 字段默认值
CandidateOrderIntent 字段默认值
status / allows_downstream 语义
hash 字段非空
JSON 字段可序列化
```

### 21.2 USDⓈ-M calculator 测试

至少覆盖：

```text
空仓开多
空仓开空
同向加仓
同向减仓
平仓到 0
多转空 primary + fallback
空转多 primary + fallback
价格上涨后 current_position_notional 变化但 quantity 不变
min_rebalance_notional below threshold → no_order_required
mark_price 缺失 / <=0 → blocked
```

### 21.3 COIN-M calculator 测试

至少覆盖：

```text
用 current_equity_native * mark_price 计算 USD 权益
用 target_notional_usd / contractSize 计算 target_contracts
用 abs(position_contracts) * contractSize 计算 current_position_notional_usd
空仓开仓
同向加仓
同向减仓
平仓
反手 primary + fallback
缺 contractSize → blocked
contractSize <=0 → blocked
缺 marginAsset balance → blocked
缺 mark_price → blocked
COIN-M 不使用 U 本位 quantity = notional / price 公式
min_rebalance_notional below threshold → no_order_required
```

### 21.4 order_components 测试

至少覆盖：

```text
increase_risk component 字段完整
reduce_risk component 字段完整
反手同时包含 reduce_risk + increase_risk
fallback_reduce_only 只包含 reduce_risk
component 不包含 reduceOnly 字段
order_components_hash 稳定
```

### 21.5 active_order_guard 测试

至少覆盖：

```text
同 symbol / market_type / account_domain 存在 active OrderPlan → blocked
同 symbol / market_type / account_domain 存在 active CandidateOrderIntent → blocked
不同 symbol 不阻断
不同 market_type 不阻断
不同 account_domain 不阻断
已终结状态不阻断
尚未实现 ApprovedOrderIntent / ExecutionOrder 时不越界创建模型
```

### 21.6 幂等测试

至少覆盖：

```text
相同输入重复运行返回同一个 OrderPlan
相同输入不重复生成 CandidateOrderIntent
相同输入不重复写 AlertEvent
更换 price_snapshot_id 生成新 OrderPlan
更换 binance_sync_run_id 生成新 OrderPlan
更换 max_target_notional_to_equity_ratio 生成新 OrderPlan
```

### 21.7 dry-run 测试

至少覆盖：

```text
dry_run=True 不写 OrderPlan
dry_run=True 不写 CandidateOrderIntent
dry_run=True 不写 AlertEvent
dry_run=True 返回 primary / fallback 摘要
dry_run=True 仍执行输入校验和 calculator 计算
```

### 21.8 management command 测试

至少覆盖：

```text
正常生成 OrderPlan
blocked 输出 reason_code
dry-run 不落库
缺必要参数报错
```

---

## 22. 实现与接入步骤

### Phase A：app 与模型

```text
1. 新增 apps/order_plan。
2. 注册 Django app。
3. 新增 OrderPlan / CandidateOrderIntent 模型。
4. 增加 migrations。
5. 增加 constants / hashes。
6. 增加基础模型测试。
```

### Phase B：selectors 与 context builder

```text
1. 读取 DecisionSnapshot。
2. 读取 BinanceSyncRun。
3. 读取 account / balance / position / symbol rule 快照。
4. 读取 PriceSnapshot。
5. 校验 market_type / account_domain / symbol。
6. 实现 active_order_guard。
7. 生成标准化 OrderPlanContext。
```

### Phase C：calculator

```text
1. 实现 base calculator dataclass。
2. 实现 USDⓈ-M calculator。
3. 实现 COIN-M calculator。
4. 实现 registry。
5. 实现 order_components builder。
6. 补 calculator 单元测试。
```

### Phase D：service 与落库

```text
1. 实现 OrderPlanService.build_order_plan。
2. 实现幂等查询与创建。
3. 实现 CandidateOrderIntent 创建。
4. 实现 primary / fallback 关联。
5. 实现 AlertEvent 写入。
6. 实现 dry-run。
7. 补 service 测试。
```

### Phase E：management command

```text
1. 实现 build_order_plan command。
2. 支持 dry-run。
3. 输出清晰 summary。
4. 补 command 测试。
```

### Phase F：回归测试

运行：

```powershell
.\.venv\Scripts\python.exe manage.py makemigrations --check --dry-run
.\.venv\Scripts\python.exe manage.py test
.\.venv\Scripts\pytest.exe
```

---

## 23. 与 007B RiskCheck 的接口约定

OrderPlan 输出给 RiskCheck 的最小合同：

```text
OrderPlan.id
OrderPlan.status = created
OrderPlan.is_usable = True
OrderPlan.allows_downstream = True

CandidateOrderIntent.id
CandidateOrderIntent.status = pending_risk_check
CandidateOrderIntent.intent_type = primary / fallback_reduce_only
CandidateOrderIntent.side
CandidateOrderIntent.positionSide = BOTH
CandidateOrderIntent.reduceOnly
CandidateOrderIntent.requested_size
CandidateOrderIntent.requested_notional
CandidateOrderIntent.order_components
CandidateOrderIntent.intent_hash
```

RiskCheck 只审批 CandidateOrderIntent。

OrderPlan 不判断 CandidateOrderIntent 是否能通过风控。

---

## 24. 验收标准

实现完成后必须满足：

```text
1. 006A DecisionSnapshot 不得被当作订单动作使用。
2. OrderPlan 只读取 target_position_ratio / target_intent。
3. 不出现策略动作枚举作为订单入口。
4. 不使用 observed_exchange_leverage 计算目标仓位。
5. 使用 max_target_notional_to_equity_ratio 计算系统目标名义。
6. U 本位与 COIN-M 走不同 calculator。
7. COIN-M 使用 contractSize + mark_price + native equity，不使用 U 本位公式。
8. One-Way Mode 反手生成 primary + fallback_reduce_only。
9. order_components 可审计，且不把 component 当 Binance 子订单。
10. min_rebalance_notional 生效。
11. active_order_guard 生效。
12. 正式结果写 AlertEvent。
13. dry-run 不写库、不告警。
14. OrderPlan 幂等。
15. CandidateOrderIntent 幂等。
16. 不实现真实下单、撤单、查订单、查成交。
17. 不修改 Binance leverage / margin type / position mode。
18. 测试通过。
```

---

## 25. 完成后汇总要求

Codex 完成后需要汇总：

```text
1. 修改了哪些文件。
2. 新增了哪些模型和字段。
3. OrderPlan 如何计算 U 本位目标 quantity。
4. OrderPlan 如何计算 COIN-M target contracts。
5. max_target_notional_to_equity_ratio 如何使用。
6. 为什么 OrderPlan 不使用 observed_exchange_leverage。
7. primary / fallback_reduce_only 如何生成。
8. order_components 如何表达风险语义。
9. active_order_guard 如何实现。
10. AlertEvent 写入了哪些场景。
11. dry-run 行为。
12. 新增 / 修改了哪些测试。
13. 测试结果。
14. 是否发现需求与当前实现冲突。
15. 是否有任何越界实现。
```
