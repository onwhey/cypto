# 006C PriceSnapshot 需求文件

> 建议存放路径：`docs/requirements/price_snapshot.md`  
> 模块编号：006C  
> 当前状态：待实现  
> 上游：`006B Binance Account Sync`、后续 `REST Mark Price / WebSocket PriceFeed`  
> 下游：`007A OrderPlan`、`007B RiskCheck`、后续 `ExecutionPreparation`  
> 核心原则：PriceSnapshot 是价格事实层，不是策略分析层，不是实时行情系统，也不是最终成交价格。

---

## 1. 模块定位

PriceSnapshot 用于把某一时刻的市场价格固化成可追溯、可审计、可幂等引用的系统事实。

正式交易链路中，PriceSnapshot 被消费于：

```text
006A DecisionSnapshot
→ 006B Binance Account Sync / 006C PriceSnapshot
→ 007A OrderPlan
→ CandidateOrderIntent
→ 007B RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation
→ Execution
```

其中：

```text
OrderPlan 使用 PriceSnapshot 把目标仓位换算成真实数量 / 合约张数。
RiskCheck 使用 PriceSnapshot 估算 notional / margin_required。
ExecutionPreparation 使用最新 PriceSnapshot 做最终 price guard。
```

PriceSnapshot 不负责：

```text
策略判断
K 线分析
生成目标仓位
生成候选订单
风控审批
真实下单
成交价记录
盘口深度
滑点估算
强平监控
```

---

## 2. 与 Kline / MarketSnapshot 的区别

Kline / MarketSnapshot 服务于策略分析：

```text
4h / 1d Kline
→ MarketSnapshot
→ FeatureLayer
→ AtomicSignal
→ StrategySignal
→ DecisionSnapshot
```

PriceSnapshot 服务于订单链路估值：

```text
OrderPlan
RiskCheck
ExecutionPreparation
```

不得使用 4h / 1d K 线收盘价直接作为下单数量、保证金估算或执行前价格保护的依据。

---

## 3. 与成交价的区别

PriceSnapshot 是下单前或风控前的参考价格事实。

成交价属于 Execution / Tracking：

```text
fill_price
average_fill_price
fee
slippage
order_trade_update
```

PriceSnapshot 不得伪造成交价，也不得作为成交结果使用。

---

## 4. P0 数据来源

P0 不接 WebSocket。

P0 价格来源允许：

```text
1. Binance Account Sync 快照中的 mark_price
2. 显式传入的测试 fixture / manual fixture
3. 后续可扩展 REST mark price，但本模块 P0 不直接实现 Binance REST 调用
```

P0 禁止：

```text
WebSocket 连接
盘口深度订阅
实时价格缓存服务
订单簿 best bid / best ask 实时维护
Binance REST client 初始化
```

后续 WebSocket 接入后，WebSocket 只是 PriceSnapshot 的数据来源之一。OrderPlan / RiskCheck / ExecutionPreparation 仍然只依赖 PriceSnapshot，不直接依赖 WebSocket。

---

## 5. 支持的价格类型

P0 只要求支持：

```text
price_type = mark_price
```

字段：

```text
mark_price
```

后续可扩展：

```text
last_price
index_price
best_bid
best_ask
mid_price
```

但 P0 不实现盘口价格和滑点价格。

---

## 6. Market Identity

每条 PriceSnapshot 必须绑定完整市场身份：

```text
exchange = binance
symbol
market_type
account_domain
```

`market_type` P0 支持：

```text
usds_m_futures
coin_m_futures
```

PriceSnapshot 不得跨 market_type / account_domain 复用。

例如：

```text
usds_m_futures BTCUSDT 的 mark_price
不得用于 coin_m_futures BTCUSD_PERP。
```

---

## 7. PriceSnapshot 模型字段

建议新增模型：

```text
PriceSnapshot
```

建议字段：

```text
id
exchange
symbol
market_type
account_domain
price_type
mark_price
price_unit
source
source_snapshot_type
source_snapshot_id
source_event_id
as_of_utc
received_at_utc
created_at
schema_version
price_snapshot_hash
is_usable
blocked_reason
raw_payload
trace_id
```

字段说明：

```text
mark_price
= Decimal，必须 > 0。

price_unit
= 价格计价单位，例如 USDT / USD。

source
= account_sync / manual_fixture / rest_mark_price / websocket_mark_price 等。

source_snapshot_type / source_snapshot_id
= 如果来源是 BinancePositionSnapshot 或其他快照，记录来源对象。

as_of_utc
= 价格事实对应的时间。

received_at_utc
= 系统接收到 / 创建该 PriceSnapshot 的时间。

price_snapshot_hash
= 对市场身份、价格、来源、时间、schema_version 等关键字段计算出的 hash。

raw_payload
= 原始来源数据摘要，不能存放敏感密钥。
```

---

## 8. PriceSnapshot 状态语义

PriceSnapshot 本身是事实记录，原则上创建后不可变。

建议字段：

```text
is_usable
blocked_reason
```

当创建时发现关键字段缺失或非法：

```text
is_usable = False
blocked_reason = invalid_mark_price / missing_market_identity / missing_as_of_utc
```

正式 OrderPlan / RiskCheck 不得消费：

```text
is_usable = False
```

---

## 9. 价格有效性要求

P0 必须校验：

```text
mark_price 存在
mark_price > 0
mark_price 可转 Decimal
as_of_utc 存在
exchange / symbol / market_type / account_domain 完整
```

如果不满足：

```text
PriceSnapshot.is_usable = False
blocked_reason = 对应原因
```

如果消费者在正式链路中发现价格快照缺失或不可用：

```text
OrderPlan / RiskCheck = BLOCKED
写 AlertEvent
```

---

## 10. TTL 与消费边界

PriceSnapshot 不自动代表永远可用。

消费者必须按场景检查新鲜度。

P0 建议配置：

```text
ORDER_PLAN_PRICE_SNAPSHOT_MAX_AGE_SECONDS = 60
RISK_CHECK_PRICE_SNAPSHOT_MAX_AGE_SECONDS = 60
EXECUTION_PRICE_GUARD_MAX_AGE_SECONDS = 30  # 后续 ExecutionPreparation 使用
```

含义：

```text
OrderPlan 使用的价格快照超过 60 秒 → OrderPlan BLOCKED，要求刷新价格。
RiskCheck 使用的价格快照超过 60 秒 → RiskCheck BLOCKED，要求刷新价格。
ExecutionPreparation 真正准备发单前必须刷新最新价格，并使用 30 秒 TTL。
```

注意：

```text
30 秒 / 0.5% price guard 不属于 PriceSnapshot P0，也不属于 OrderPlan / RiskCheck。
它属于后续 ExecutionPreparation。
```

PriceSnapshot P0 只提供：

```text
价格事实
as_of_utc
hash
TTL 校验辅助
```

---

## 11. U 本位消费规则

U 本位 OrderPlan 使用：

```text
target_quantity = target_notional_quote / mark_price
current_position_notional = abs(position_quantity) * mark_price
```

U 本位 RiskCheck 使用：

```text
opening_notional_quote = component_size_quantity * mark_price
margin_required_quote = opening_notional_quote / observed_exchange_leverage
```

PriceSnapshot 不计算目标仓位，也不计算风控结果，只提供 mark_price。

---

## 12. COIN-M 消费规则

COIN-M 中：

```text
mark_price = symbol / pair 的 USD 标记价格
contractSize = 每张合约固定 USD 面值
```

两者不得混淆。

COIN-M OrderPlan 使用：

```text
current_equity_usd_value = current_equity_native * mark_price
target_notional_usd = current_equity_usd_value * max_target_notional_to_equity_ratio * abs(target_position_ratio)
target_contracts = target_notional_usd / contractSize
```

COIN-M RiskCheck 使用：

```text
opening_notional_usd = component_contracts * contractSize
margin_required_native = opening_notional_usd / mark_price / observed_exchange_leverage
```

PriceSnapshot 不保存 contractSize。contractSize 来自 BinanceSymbolRuleSnapshot。

---

## 13. Hash 与幂等

PriceSnapshot 必须有稳定 hash。

`price_snapshot_hash` 至少基于：

```text
schema_version
exchange
symbol
market_type
account_domain
price_type
mark_price
price_unit
source
source_snapshot_type
source_snapshot_id
source_event_id
as_of_utc
```

相同来源、相同市场身份、相同价格事实重复创建时，允许返回已有 PriceSnapshot。

不同 as_of_utc、不同 source_snapshot_id、不同 mark_price 应生成不同 PriceSnapshot。

---

## 14. 与 Binance Account Sync 的关系

P0 可从 Binance Account Sync 的持仓快照中提取 mark_price 生成 PriceSnapshot。

要求：

```text
只使用指定 binance_sync_run_id 下的快照。
market_type / account_domain / symbol 必须一致。
mark_price 必须存在且 > 0。
```

如果该 sync run 中没有可用 mark_price：

```text
PriceSnapshot 创建失败或返回 unusable summary。
OrderPlan / RiskCheck 不得自行伪造价格。
```

---

## 15. dry-run 要求

PriceSnapshot dry-run：

```text
执行相同校验
返回将要创建的价格快照摘要
不写 PriceSnapshot
不写 AlertEvent
不修改数据库
```

如果 dry-run 被 OrderPlan / RiskCheck 调用，只能用于测试或命令验证，不得被正式交易链路消费。

---

## 16. AlertEvent 要求

PriceSnapshot 创建本身不属于订单事件，P0 不要求创建成功必须 AlertEvent。

但以下场景由消费者写 AlertEvent：

```text
OrderPlan 因 PriceSnapshot 缺失 / 过期 / 不可用而 BLOCKED
RiskCheck 因 PriceSnapshot 缺失 / 过期 / 不可用而 BLOCKED
ExecutionPreparation 因最新价格缺失 / 过期 / 偏离过大而 BLOCKED
```

PriceSnapshotService 自身遇到系统异常，可按通用系统异常规则写 AlertEvent。

---

## 17. 非目标

PriceSnapshot P0 不实现：

```text
WebSocket
实时行情服务
盘口深度
best bid / best ask
滑点估算
成交价记录
真实 Binance REST 调用
Binance client 初始化
执行前 30 秒 / 0.5% price guard
策略分析
K 线采集
订单状态追踪
强平监控
UI / dashboard
```

---

## 18. P1 后续增强项

P1 可加入：

```text
REST mark price source
WebSocket mark price source
最新价格缓存
价格源优先级
价格源健康检查
价格断流 AlertEvent
多 source 价格偏离检查
best bid / best ask
```

---

## 19. P2 后续能力

P2 可加入：

```text
盘口深度
实时滑点估算
流动性检查
闪崩保护
组合级实时风险价格流
高频价格采样压缩
```

---

## 20. 验收标准

实现 PriceSnapshot P0 时，至少满足：

```text
1. 能创建可审计 PriceSnapshot。
2. mark_price 必须存在且 > 0。
3. market identity 完整且不可跨 market_type / account_domain 复用。
4. 能从 Binance Account Sync 快照生成 PriceSnapshot。
5. 能用 manual fixture 创建测试 PriceSnapshot。
6. 有稳定 price_snapshot_hash。
7. 支持 TTL 校验辅助。
8. OrderPlan / RiskCheck 可以通过 price_snapshot_id 引用价格事实。
9. dry-run 不写库、不告警。
10. 不实现 WebSocket。
11. 不实现真实 Binance REST 调用。
12. 不实现 Execution price guard。
```
