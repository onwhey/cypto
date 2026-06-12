# 007B RiskCheck 实现计划

> 建议存放路径：`docs/plans/007B_risk_check_plan.md`
> 
> 对应需求文件：`docs/requirements/risk_check.md`
> 
> 当前目标：实现 `CandidateOrderIntent → RiskCheck → ApprovedOrderIntent` 风控合同。
> 
> 本计划只覆盖 007B RiskCheck P0，不实现真实下单、不实现撤单、不实现成交追踪、不实现 ExecutionPreparation price guard。

---

## 0. 实施结论

RiskCheck 是 OrderPlan 之后、ExecutionPreparation 之前的风控闸门。

实现上使用**一个 plan 文件，内部按阶段 A / B / C / D / E 执行**，不需要再拆成多份 RiskCheck 计划文件。

阶段划分：

```text
Phase A：模型、状态与字段
Phase B：selectors / calculators / component evaluator
Phase C：rule plugin 与 RuleEngine 接入
Phase D：RiskCheckService / command / AlertEvent / dry-run / 幂等
Phase E：测试与中文实现文档补充
```

---

## 1. Codex 开始前必须阅读

请先阅读：

```text
AGENTS.md
README.md
docs/rules/project_invariants.md
docs/requirements/risk_check.md
docs/requirements/order_plan.md
docs/requirements/binance_account_sync.md
docs/plans/007A_order_plan_plan.md
apps/risk_check/
apps/binance_account_sync/
tests/risk_check/
```

如果 `apps/order_plan/` 已经存在，也必须阅读：

```text
apps/order_plan/
tests/order_plan/
```

如果 `apps/order_plan/` 尚未实现，本计划不得自行重复创建 OrderPlan / CandidateOrderIntent 的完整业务模型；只能在 007B 中通过清晰接口约定或显式 TODO 等待 007A 完成。

---

## 2. 本阶段目标

RiskCheck P0 职责：

```text
RiskCheck 只检查 CandidateOrderIntent 是否允许进入执行准备。
```

正式链路：

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

RiskCheck 直接消费：

```text
OrderPlan
CandidateOrderIntent
Binance Account Sync 快照批次
PriceSnapshot
RiskRuleDefinition / risk_config
```

RiskCheck 不消费 DecisionSnapshot，也不消费任何策略动作枚举。

---

## 3. 非目标

本次不得实现：

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
ExecutionPreparation 30 秒 / 0.5% price guard
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

## 4. 输入入口边界

RiskCheck P0 唯一订单输入是 `CandidateOrderIntent`。

```text
允许输入：OrderPlan / CandidateOrderIntent / BinanceSyncRun / PriceSnapshot / RiskRuleDefinition
不允许输入：DecisionSnapshot、策略动作枚举、交易所实时连接、执行回报
```

RuleEngine 只能通过 `RiskRuleDefinition.rule_code` 调用已注册的 RiskRulePlugin，不得在 RuleEngine 中硬编码具体规则判断。

---

## 5. Phase A：模型、状态与字段

### 5.1 RiskCheckResult 状态

P0 使用四类状态：

```text
ALLOW
DENY
BLOCKED
FAILED
```

语义：

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

如果代码内部使用小写枚举，必须保持语义一致。

### 5.2 RiskCheckResult 关键字段

目标模型至少支持：

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

不得增加会让 RiskCheck 直接消费 DecisionSnapshot 或策略动作枚举的字段。

### 5.3 ApprovedOrderIntent 边界

本阶段必须实现 ApprovedOrderIntent 的生成或等价确认。

当 RiskCheckResult.status = ALLOW 时，RiskCheckService 必须基于被批准的 CandidateOrderIntent 生成 ApprovedOrderIntent，或写入等价的 approved intent 记录。

当 RiskCheckResult.status in DENY / BLOCKED / FAILED 时，不得生成 ApprovedOrderIntent。

RiskCheckResult 必须输出足够信息支持后续生成 ApprovedOrderIntent：

```text
selected_candidate_order_intent_id
selected_intent_type
status = ALLOW
allows_downstream = true
risk_snapshot
evidence_items
```

ApprovedOrderIntent 的订单方向、数量、reduce_only、order_type 等核心内容必须来自 CandidateOrderIntent。
RiskCheck 不得重新设计订单，只能批准、拒绝、阻断，或在规则允许时选择 OrderPlan 预生成的 fallback_reduce_only_intent。


---

## 6. Phase B：selectors / calculators / component evaluator

### 6.1 BinanceSyncRun selector

必须基于显式传入的：

```text
binance_sync_run_id
```

读取同一批次下的：

```text
BinanceSyncRun
BinanceAccountSnapshot
BinanceBalanceSnapshot
BinancePositionSnapshot
BinanceSymbolRuleSnapshot
```

必须校验：

```text
BinanceSyncRun.status = succeeded
BinanceSyncRun.exchange = binance
market_type 匹配 CandidateOrderIntent.market_type
account_domain 匹配 CandidateOrderIntent.account_domain
未过期
snapshot_set_hash 可校验
account / balance / position / symbol rule 快照完整
```

不允许自动回退到其他 BinanceSyncRun。

### 6.2 CandidateOrderIntent selector

必须校验：

```text
CandidateOrderIntent 存在
status 可消费
order_plan_id 匹配
symbol / market_type / account_domain 匹配
side 合法
requested_size / requested_notional 合法
positionSide = BOTH
reduceOnly 合法
order_components 存在且结构合法
intent_hash 可校验
未取消、未废弃、未过期、未消费
```

不接受 DecisionSnapshot 或策略动作枚举作为风控入口。

### 6.3 PriceSnapshot selector

RiskCheck 使用 PriceSnapshot 进行估值和保证金估算。

P0 新增温和新鲜度检查：

```text
RISK_CHECK_PRICE_SNAPSHOT_MAX_AGE_SECONDS = 60
```

规则：

```text
price_snapshot 不存在 → BLOCKED
price_snapshot age > 60 秒 → BLOCKED, reason_code = stale_price_snapshot
```

注意：这不是 ExecutionPreparation 的最终 price guard。

ExecutionPreparation 的 30 秒 / 0.5% 检查不在 007B 实现。

### 6.4 Component evaluator

RiskCheck 必须按 `order_components` 分段评估。

组件类型至少支持：

```text
reduce_risk
increase_risk
```

reduce_risk 组件必须校验：

```text
当前持仓存在
方向匹配
数量不超过当前持仓
side 与 position_effect 一致
数量满足交易所规则：minQty / stepSize / minNotional / maxQty
不按新增仓位计算保证金
不要求 observed_exchange_leverage 存在
```

increase_risk 组件必须校验：

```text
observed_exchange_leverage 存在且 > 0
availableBalance 足够
保证金估算通过
交易所 symbol rule 通过
maxQty / maxNotional / maxNotionalValue 通过
```

### 6.5 U 本位 calculator

U 本位增加风险组件：

```text
opening_notional_quote = component_size_quantity * mark_price
margin_required_quote = opening_notional_quote / observed_exchange_leverage
```

余额检查：

```text
available_balance_quote >= margin_required_quote + buffer
```

`observed_exchange_leverage` 只用于保证金估算，不得用于重新计算系统目标数量。

### 6.6 COIN-M calculator

COIN-M 增加风险组件：

```text
opening_notional_usd = component_contracts * contractSize
margin_required_native = opening_notional_usd / mark_price / observed_exchange_leverage
```

余额检查：

```text
available_balance_native >= margin_required_native + buffer
```

必须校验：

```text
contractSize 存在
contractSize > 0
contractSize 可 Decimal 解析
mark_price 存在且 > 0
marginAsset 存在
available_balance_native 存在
marginAsset 能匹配到对应 balance asset
```

COIN-M 不得复用 U 本位线性 PnL / 保证金公式。

### 6.7 P0 不做精确 Binance 最大可开模拟

P0 不实现一个复杂的 `max_open_capacity` 精确计算器。

不要模拟 Binance App 的最大可开。

P0 拆成明确检查：

```text
symbol_rule_max_quantity_check
symbol_rule_max_notional_check
available_margin_check
available_balance_check
```

更精细的交易所最大可开估算列为 P1。

---

## 7. Phase C：rule plugin 与 RuleEngine 接入

### 7.1 保留 plugin 架构

继续使用：

```text
RiskRuleDefinition
RiskRulePlugin
RiskRuleRegistry
RuleEngine
```

RuleEngine 只做编排：

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

### 7.2 P0 建议插件

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


### 7.3 active_order_guard 范围

检查范围必须包括同一：

```text
symbol
market_type
account_domain
```

下未完成的：

```text
OrderPlan
CandidateOrderIntent
ApprovedOrderIntent
ExecutionOrder
```

如果存在未终结状态：

```text
RiskCheckResult.status = BLOCKED
reason_code = active_order_exists
写 AlertEvent
```

终结状态的具体枚举按模型实现确定，但必须至少覆盖：

```text
completed
cancelled
expired
denied
failed
```

---

## 8. Phase D：RiskCheckService 主流程

### 8.1 主流程

RiskCheckService 执行顺序建议：

```text
1. 标准化输入。
2. 加载 CandidateOrderIntent / OrderPlan。
3. 加载 BinanceSyncRun 与同批次快照。
4. 加载 PriceSnapshot，并检查 60 秒估值 TTL。
5. 加载 active / enabled RiskRuleDefinition。
6. 计算 rule_set_hash / risk_config_hash。
7. 检查幂等 key，已有相同结果则返回。
8. 执行 RuleEngine。
9. 对 primary_intent 进行组件级风控。
10. 如 primary 为净额反手且 increase_risk 不通过，尝试 fallback_reduce_only_intent。
11. 聚合最终 ALLOW / DENY / BLOCKED / FAILED。
12. 写 RiskCheckResult / RuleResult / Issue。
13. 正式链路写 AlertEvent。
14. 返回结果摘要。
```

### 8.2 fallback_reduce_only 聚合规则

净额反手：

```text
primary_intent 全部通过
→ ALLOW primary_intent
```

```text
primary_intent 的 increase_risk component 不通过，fallback_reduce_only_intent 通过
→ ALLOW fallback_reduce_only_intent
→ selected_intent_type = fallback_reduce_only
→ 必须 AlertEvent
```

```text
primary_intent 的 reduce_risk component 本身不通过
→ 不应继续批准 fallback
→ 按原因 BLOCKED / DENY / FAILED
```

状态归类：

```text
当前持仓数量不足 / 持仓方向不一致 / 快照不一致
→ BLOCKED
→ reason_code = position_snapshot_mismatch 或 stale_position_snapshot

fallback 数量不满足交易所规则
→ DENY

fallback 缺关键字段
→ BLOCKED

系统异常
→ FAILED
```

### 8.3 AlertEvent

正式链路所有结果都必须 AlertEvent：

```text
ALLOW
DENY
BLOCKED
FAILED
fallback_reduce_only
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

P1 可增加 AlertEvent 路由、级别、聚合、限频，但 P0 必须写 AlertEvent。

### 8.4 dry-run

dry-run 时：

```text
不写 RiskCheckResult
不写 RiskRuleResult / RiskCheckIssue
不写 AlertEvent
不生成 ApprovedOrderIntent
不修改数据库
返回风控摘要
```

dry-run 仍必须执行与正式流程一致的输入校验、规则校验和风险计算。

### 8.5 幂等

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

---

## 9. risk_measures 最小字段集

`risk_measures` 至少包含：

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

COIN-M 额外包含：

```text
contract_size
current_equity_native
current_equity_usd_value
margin_required_native
available_balance_native
```

这些字段必须能支持后续审计、复盘和 AlertEvent 摘要。

---

## 10. management command

建议新增命令：

```text
python manage.py run_risk_check \
  --candidate-order-intent-id <id> \
  --order-plan-id <id> \
  --binance-sync-run-id <id> \
  --price-snapshot-id <id> \
  --symbol BTCUSDT \
  --market-type usds_m_futures \
  --account-domain <domain> \
  --trigger-source manual \
  [--dry-run]
```

命令只允许显式传入以下正式入口参数：

```text
--candidate-order-intent-id
--order-plan-id
--binance-sync-run-id
--price-snapshot-id
--symbol
--market-type
--account-domain
--trigger-source
--dry-run
```

---

## 11. 测试计划

### 11.1 输入与快照测试

覆盖：

```text
CandidateOrderIntent hash 失败 → BLOCKED
OrderPlan 不可用 → BLOCKED
BinanceSyncRun.status != succeeded → BLOCKED
market_type mismatch → BLOCKED
account_domain mismatch → BLOCKED
snapshot_set_hash 失败 → BLOCKED
PriceSnapshot 缺失 → BLOCKED
PriceSnapshot age > 60 秒 → BLOCKED
```

### 11.2 U 本位测试

覆盖：

```text
increase_risk 保证金足够 → ALLOW
increase_risk availableBalance 不足 → DENY
observed_exchange_leverage 缺失且存在 increase_risk → BLOCKED
reduce_risk 不因 observed_exchange_leverage 缺失被阻断
reduce_risk 仍校验 minQty / stepSize / minNotional
超过 maxQty / maxNotional → DENY
```

### 11.3 COIN-M 测试

覆盖：

```text
contractSize 存在且 > 0 → 可计算
contractSize 缺失 → BLOCKED
contractSize = 0 → BLOCKED
mark_price 缺失 → BLOCKED
marginAsset 缺失 → BLOCKED
marginAsset 对应余额缺失 → BLOCKED
increase_risk native margin 足够 → ALLOW
increase_risk native margin 不足 → DENY
reduce_risk 不要求 observed_exchange_leverage
不得复用 U 本位线性保证金公式
```

### 11.4 净额反手与 fallback 测试

覆盖：

```text
primary close + open 全部通过 → ALLOW primary
opening component 不通过，fallback_reduce_only 通过 → ALLOW fallback_reduce_only + AlertEvent
primary reduce component 当前持仓不足 → BLOCKED
fallback_reduce_only 数量不符合 stepSize / minNotional → DENY
fallback 缺关键字段 → BLOCKED
fallback 触发必须写 AlertEvent
```

### 11.5 AlertEvent 测试

覆盖正式链路：

```text
ALLOW 写 AlertEvent
DENY 写 AlertEvent
BLOCKED 写 AlertEvent
FAILED 写 AlertEvent
fallback_reduce_only 写 AlertEvent
dry-run 不写 AlertEvent
幂等重复调用不重复写等价 AlertEvent
```

### 11.6 幂等测试

覆盖：

```text
相同输入重复运行返回同一 RiskCheckResult
不同 price_snapshot_id 生成新 RiskCheckResult
不同 binance_sync_run_id 生成新 RiskCheckResult
不同 rule_set_hash 生成新 RiskCheckResult
```

### 11.7 plugin 架构测试

覆盖：

```text
active / enabled 规则被执行
disabled 规则不执行
找不到 plugin → fail-closed + AlertEvent
RuleEngine 不包含具体 rule_code 大型 if/elif 分发
```

---

## 12. 中文实现文档

建议更新或新增：

```text
docs/implementation/risk_check/README.md
docs/implementation/risk_check/rule_engine.md
docs/implementation/risk_check/order_components.md
docs/implementation/risk_check/usds_m_margin.md
docs/implementation/risk_check/coin_m_margin.md
docs/implementation/risk_check/reverse_fallback_reduce_only.md
docs/implementation/risk_check/alert_events.md
```

文档必须说明：

```text
1. RiskCheck 只消费 CandidateOrderIntent，不消费 DecisionSnapshot。
2. observed_exchange_leverage 不参与系统目标仓位计算。
3. reduce_risk 不需要 observed_exchange_leverage。
4. increase_risk 需要 observed_exchange_leverage 进行保证金估算。
5. COIN-M contractSize / mark_price / marginAsset 的单位与用途。
6. fallback_reduce_only 为什么不是任意 MODIFY。
7. 所有风控阻断、拒绝、失败结果为什么必须 AlertEvent。
```

---

## 13. 完成后运行

请运行：

```powershell
.\.venv\Scripts\python.exe manage.py makemigrations --check --dry-run
.\.venv\Scripts\python.exe manage.py test
.\.venv\Scripts\pytest.exe
```

如果测试环境中 007A OrderPlan 尚未实现，必须说明阻塞原因，不得伪造模型或绕过测试。

---

## 14. 最后汇总要求

完成后请汇总：

```text
1. 修改了哪些文件。
2. CandidateOrderIntent 风控入口如何实现。
4. order_components 如何分段风控。
5. U 本位保证金如何估算。
6. COIN-M 保证金如何估算。
7. observed_exchange_leverage 如何使用，如何避免参与目标仓位计算。
8. fallback_reduce_only 如何触发。
9. 所有正式结果 AlertEvent 如何实现。
10. 新增 / 修改了哪些测试。
11. 测试结果。
12. 是否发现需求与当前实现冲突。
13. 是否有任何越界实现。
```
