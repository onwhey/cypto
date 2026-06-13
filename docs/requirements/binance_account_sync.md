# Binance Account Sync / 币安账户信息同步模块需求文档

> 建议路径：`docs/requirements/binance_account_sync.md`  
> 当前定位：006C PriceSnapshot、007A OrderPlan、007B RiskCheck 的只读 Binance 账户事实来源。
> P0 范围：作为 OrderPlan、PriceSnapshot、RiskCheck 的只读 Binance 账户事实来源。  
> 适用交易所：Binance / 币安  
> 核心原则：只读同步、单 active account domain、数据库快照化、可 hash 追溯；不下单、不撤单、不调杠杆、不调保证金模式、不做风控批准。

---

## 1. 模块定位

`Binance Account Sync` 是明确的币安专属账户信息同步模块。模块名称、app 名、表名、配置项必须包含 `binance` 或 `Binance` 语义，避免后续接入其他交易所时与 OKX / Bybit / Deribit 等混淆。

建议 app 名：

```text
apps/binance_account_sync/
```

本模块负责从 Binance 官方 API 读取账户、余额、持仓和交易规则信息，标准化后写入数据库快照，为后续链路提供可追溯输入：

```text
PriceSnapshot：从 BinancePositionSnapshot.mark_price 派生估值价格快照。
007A OrderPlan：读取 current_equity、当前持仓、position_mode、symbol rule、contract_size。
007B RiskCheck：读取 available_balance、observed_exchange_leverage、margin_asset、contract_size、symbol rule。
```

本模块只负责账户事实同步，不负责订单 / 成交 / 执行结果事实：

```text
Binance Account Sync 不负责订单状态同步。
Binance Account Sync 不负责成交状态同步。
Binance Account Sync 不生成或处理 ExchangeOrder。
Binance Account Sync 不生成或处理 TradeFill。
Binance Account Sync 不生成或处理 ExecutionResult。
订单状态与成交状态属于后续 Execution / ExchangeOrder / TradeFill / Tracking 边界。
```

本模块只做：

```text
读取 Binance 数据
标准化字段
生成数据库快照
生成同步批次 BinanceSyncRun
生成 hash
记录 raw_payload
记录 as_of_utc / source / endpoint
提供 selector 给 OrderPlan / PriceSnapshot / RiskCheck 查询指定 sync_run 的完整快照
异常时写 AlertEvent
```

本模块不做：

```text
DecisionSnapshot
OrderPlan
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
ExchangeOrder
TradeFill
ExecutionResult
订单状态同步
成交状态同步
订单追踪
下单
撤单
开仓
平仓
调整杠杆
调整保证金模式
资金划转
自动补保证金
风控批准
仓位计算
ExecutionPreparation / Execution
收益回测
UI
```

---

## 2. 核心原则

### 2.1 Binance 专属，不做泛交易所抽象

P0 只面向 Binance，禁止把模块命名成过于抽象的 `exchange_account_sync` 或 `account_gateway`。

后续如接入其他交易所，应新增独立模块或在更高层做统一接口，而不是让 Binance 模块承担多交易所职责。

### 2.2 只读权限

P0 API Key 只能使用读取账户、余额、持仓、交易规则所需权限。

禁止请求或使用：

```text
交易权限
Universal Transfer / 统一划转权限
提现权限
调杠杆权限
调整保证金权限
```

如果 active 账户余额不足、保证金不足或资产缺失：

```text
写 AlertEvent
后续 OrderPlan / RiskCheck blocked 或 denied
由用户人工处理资金划转
系统不得自动划转或自动补保证金
```

### 2.3 单 active market_type / account_domain

系统运行时只能选择一个 active Binance account domain / market type。

配置示例：

```text
BINANCE_ACTIVE_MARKET_TYPE=usds_m_futures
```

或：

```text
BINANCE_ACTIVE_MARKET_TYPE=coin_m_futures
```

P0 支持的合法值：

```text
usds_m_futures
coin_m_futures
```

P0 中：

```text
account_domain == market_type
```

运行时只同步并服务 active market_type / account_domain。非 active account domain 不参与 OrderPlan、RiskCheck、订单或交易链路。

### 2.4 U 本位和币本位账户域独立

USDⓈ-M Futures 和 COIN-M Futures 是两个独立账户域。它们的账户、余额、持仓和接口路径不同。

系统必须通过 `market_type` 与 `account_domain` 强隔离：

```text
usds_m_futures -> /fapi/*
coin_m_futures -> /dapi/*
```

禁止混用：

```text
U 本位余额不足时读取币本位余额兜底
币本位 BTC 持仓参与 U 本位 BTCUSDT 风控
U 本位持仓被当成币本位持仓
U 本位 mark_price 被 COIN-M 订单规划使用
COIN-M contract_size 被 U 本位目标数量计算使用
```

---

## 3. 官方文档约束

所有 Binance API endpoint、字段映射、权限说明必须以 Binance 官方开发者文档为准。

禁止以第三方 SDK、博客、非官方示例项目作为字段来源。

P0 必须参考的 Binance 官方文档：

### 3.1 USDⓈ-M Futures / U 本位

```text
Account Information V3:
GET /fapi/v3/account
https://developers.binance.com/docs/derivatives/usds-margined-futures/account/rest-api/Account-Information-V3

Futures Account Balance V3:
GET /fapi/v3/balance
https://developers.binance.com/docs/derivatives/usds-margined-futures/account/rest-api/Futures-Account-Balance-V3

Position Information V3:
GET /fapi/v3/positionRisk
https://developers.binance.com/docs/derivatives/usds-margined-futures/trade/rest-api/Position-Information-V3

Exchange Information:
GET /fapi/v1/exchangeInfo
https://developers.binance.com/docs/derivatives/usds-margined-futures/market-data/rest-api/Exchange-Information
```

### 3.2 COIN-M Futures / 币本位

```text
Account Information:
GET /dapi/v1/account
https://developers.binance.com/docs/derivatives/coin-margined-futures/account/rest-api/Account-Information

Futures Account Balance:
GET /dapi/v1/balance
https://developers.binance.com/docs/derivatives/coin-margined-futures/account/rest-api/Futures-Account-Balance

Position Information:
GET /dapi/v1/positionRisk
https://developers.binance.com/docs/derivatives/coin-margined-futures/trade/rest-api/Position-Information

Exchange Information:
GET /dapi/v1/exchangeInfo
https://developers.binance.com/docs/derivatives/coin-margined-futures/market-data/rest-api/Exchange-Information
```

如 Binance 官方 endpoint 或字段发生变化，以官方 changelog / endpoint 文档为准。

---

## 4. P0 API 使用策略

### 4.1 U 本位接口

P0 使用：

```text
GET /fapi/v3/account
GET /fapi/v3/balance
GET /fapi/v3/positionRisk
GET /fapi/v1/exchangeInfo
```

用途：

```text
/fapi/v3/account      -> 账户总体信息、资产和原始账户 payload
/fapi/v3/balance      -> 余额明细
/fapi/v3/positionRisk -> 正式持仓快照主要来源
/fapi/v1/exchangeInfo -> symbol 交易规则
```

### 4.2 币本位接口

P0 使用：

```text
GET /dapi/v1/account
GET /dapi/v1/balance
GET /dapi/v1/positionRisk
GET /dapi/v1/exchangeInfo
```

用途：

```text
/dapi/v1/account      -> 账户总体信息、资产和原始账户 payload
/dapi/v1/balance      -> 币本位资产余额明细
/dapi/v1/positionRisk -> 正式持仓快照主要来源
/dapi/v1/exchangeInfo -> symbol 交易规则，COIN-M contractSize 来源
```

### 4.3 PositionSnapshot 来源

`PositionSnapshot` P0 应以 `/positionRisk` 为主要来源。

`/account` 中返回的 positions 可以保存进 raw_payload 或作为辅助校验，但不得替代 `/positionRisk` 生成正式持仓快照。

原因：`/positionRisk` 是专门的当前持仓信息接口，字段更适合 OrderPlan / RiskCheck 使用，包括 positionAmt、entryPrice、markPrice、unRealizedProfit、liquidationPrice、leverage、marginType、positionSide 等。

---

## 5. 数据模型设计

P0 使用同一组表支持 U 本位和币本位，通过 `market_type` 与 `account_domain` 字段区分，不为两个账户域拆两套表。

建议模型：

```text
BinanceSyncRun
BinanceAccountSnapshot
BinanceBalanceSnapshot
BinancePositionSnapshot
BinanceSymbolRuleSnapshot
BinanceSyncHealth / 或健康状态 selector
```

所有快照表必须包含：

```text
exchange = binance
market_type = usds_m_futures / coin_m_futures
account_domain = usds_m_futures / coin_m_futures
sync_run_id
as_of_utc
source
endpoint
raw_payload
snapshot_hash
created_at
```

同一个 `BinanceSyncRun` 下所有快照必须满足：

```text
snapshot.account_domain == sync_run.account_domain
snapshot.market_type == sync_run.market_type
```

### 5.1 BinanceSyncRun

`BinanceSyncRun` 是账户同步批次边界，也是 OrderPlan / RiskCheck 判断账户上下文是否完整可用的核心对象。

字段建议：

```text
sync_run_key
exchange
market_type
account_domain
active_market_type_at_runtime
active_account_domain_at_runtime
position_mode = one_way / hedge / unknown
status = running / succeeded / failed
started_at_utc
completed_at_utc
as_of_utc
expires_at_utc
source = rest_api / fixture / manual_import
trigger_source
trace_id
account_snapshot_id
balance_snapshot_ids
position_snapshot_ids
symbol_rule_snapshot_ids
snapshot_set_hash
error_code
error_message
raw_request_summary
raw_response_summary
created_at
updated_at
```

P0 规则：

```text
只有 status=succeeded 的 BinanceSyncRun 才代表这一批账户上下文完整可用。
status=running / failed 不得被正式 OrderPlan / RiskCheck 使用。
P0 不实现 partial_failed 主状态。
部分 API 成功、部分 API 失败时，整个批次按 failed 处理。
partial_failed 可作为 P1 预留状态，但 P0 不产生、下游也不得消费。
```

### 5.2 BinanceAccountSnapshot

用于保存账户总体状态。

字段建议：

```text
sync_run_id
exchange
market_type
account_domain
account_mode
position_mode
multi_assets_mode
asset_summary
native_asset
wallet_balance
available_balance
margin_balance
current_equity
unrealized_pnl
used_margin
free_margin
can_trade
can_deposit
can_withdraw
as_of_utc
endpoint
raw_payload
snapshot_hash
```

说明：不同接口字段在 U 本位和币本位之间不完全一致，必须保存 raw_payload，并将标准字段映射到统一字段。

`current_equity` 口径：

```text
优先使用 marginBalance / totalMarginBalance 语义。
不得默认用 walletBalance 替代 current_equity。
```

### 5.3 BinanceBalanceSnapshot

用于保存资产余额明细。

字段建议：

```text
sync_run_id
exchange
market_type
account_domain
asset
native_balance
wallet_balance
available_balance
margin_balance
current_equity
cross_wallet_balance
cross_unrealized_pnl
withdraw_available
max_withdraw_amount
update_time_utc
as_of_utc
endpoint
raw_payload
snapshot_hash
```

U 本位常见资产：USDT / USDC。  
币本位常见结算资产：BTC 等币种。

### 5.4 BinancePositionSnapshot

用于保存某个 symbol 的当前持仓状态。

字段建议：

```text
sync_run_id
exchange
market_type
account_domain
symbol
pair
margin_asset
settlement_asset
position_side = both / long / short / none
normalized_position_side = long / short / none
position_mode_observed = one_way / hedge / unknown
position_amount
entry_price
break_even_price
mark_price
mark_price_as_of_utc
unrealized_pnl
liquidation_price
leverage
raw_leverage
observed_exchange_leverage
margin_type
isolated_margin
notional
notional_value
max_quantity
update_time_utc
as_of_utc
endpoint
raw_payload
snapshot_hash
```

说明：

```text
One-way Mode 通常使用 BOTH。
Hedge Mode 可能返回 LONG / SHORT。
系统应标准化 position_side，供 OrderPlan / RiskCheck 判断当前持仓方向。
position_amount = 0 时 normalized_position_side 可为 none，但 positionSide 仍应用于 position_mode 推断。
```

#### 5.4.1 position_mode 推断

P0 推断规则：

```text
如果目标 market_type 的 positionRisk/account positions 仅出现 BOTH：
  position_mode = one_way

如果出现 LONG 或 SHORT：
  position_mode = hedge

如果没有足够 positionSide 信息：
  position_mode = unknown
```

注意：

```text
position_amount = 0 不等于没有 positionSide。
即使 position_amount 为 0，Binance 仍可能返回 BOTH / LONG / SHORT。
Account Sync 应根据 positionSide 推断 position_mode。
```

下游语义：

```text
position_mode = hedge / unknown
→ OrderPlan / RiskCheck P0 blocked
→ reason_code = hedge_mode_not_supported / position_mode_unknown
```

Account Sync 只记录 position_mode，不修改 Binance position mode。

#### 5.4.2 observed_exchange_leverage

`observed_exchange_leverage` 来自 Binance `/positionRisk` 返回的：

```text
leverage
```

标准化后写入：

```text
BinancePositionSnapshot.observed_exchange_leverage
```

解析规则：

```text
如果 leverage 可解析为 Decimal 且 > 0：
  observed_exchange_leverage = parsed value

如果 leverage 缺失、为空、无法解析、<=0：
  observed_exchange_leverage = null
  记录 normalization warning / evidence
```

Account Sync 不应因为单个 `observed_exchange_leverage` 缺失就伪造值，也不得用系统配置替代 Binance 观测值。

下游规则写在 `risk_check.md`：

```text
RiskCheck 中存在 increase_risk component 时，observed_exchange_leverage 缺失会导致 BLOCKED。
reduce_risk component 不因 observed_exchange_leverage 缺失而阻断。
```

### 5.5 BinanceSymbolRuleSnapshot

用于保存 symbol 交易规则。

字段建议：

```text
sync_run_id
exchange
market_type
account_domain
symbol
pair
contract_type
status
base_asset
quote_asset
margin_asset
settlement_asset
contract_size
price_tick
min_price
max_price
quantity_step
min_quantity
max_quantity
min_notional
max_notional
max_notional_value
max_leverage
raw_filters
as_of_utc
endpoint
raw_payload
snapshot_hash
```

`exchangeInfo` 字段和 filters 以 Binance 官方文档和实际 raw_payload 为准。P0 必须保留 raw_filters，避免字段变更时丢失信息。

#### 5.5.1 COIN-M contract_size

COIN-M `contract_size` 来自 `/dapi/v1/exchangeInfo` 的：

```text
symbols[].contractSize
```

对于：

```text
market_type = coin_m_futures
```

`contract_size` 必须满足：

```text
存在
可解析为 Decimal
> 0
```

如果目标 symbol 的 `contract_size` 缺失或无效：

```text
selector 应返回 symbol_rule_unavailable / contract_size_missing
OrderPlan / RiskCheck 必须 fail-closed
```

不得使用：

```text
0
None
空字符串
配置默认值
硬编码 100
```

伪造 `contract_size`。

对于：

```text
market_type = usds_m_futures
```

`contract_size` 可以为空。

U 本位不得使用 COIN-M `contract_size` 计算目标数量。

#### 5.5.2 margin_asset / settlement_asset

COIN-M 保证金检查需要知道：

```text
margin_asset / settlement_asset
```

并在余额快照中找到对应资产。

P0 中：

```text
settlement_asset = margin_asset
```

如果 Binance raw payload 没有单独 settlementAsset 字段，可以由 `marginAsset` 标准化得到。

---

## 6. U 本位 / 币本位收益与估值口径

U 本位和币本位的收益、余额和未实现盈亏结算资产不同。

P0 必须保存原生资产口径：

```text
native_asset
native_amount
settlement_asset
wallet_balance
available_balance
margin_balance
unrealized_pnl
```

示例：

```text
U 本位：settlement_asset = USDT / USDC，余额和 PnL 以 USDT / USDC 计
币本位：settlement_asset = BTC，余额和 PnL 以 BTC 计
```

不得用统一估值字段覆盖原生字段。

### 6.1 U 本位 current_equity / available_balance

U 本位账户权益建议标准化：

```text
current_equity = totalMarginBalance 或目标 asset.marginBalance
available_balance = availableBalance 或目标 asset.availableBalance
```

必须保留 asset-level 明细，尤其是 USDT / USDC。

### 6.2 COIN-M current_equity / available_balance

COIN-M 账户权益以 settlement asset 原生币种保存：

```text
current_equity_native = assets[marginAsset].marginBalance
available_balance_native = assets[marginAsset].availableBalance
```

Account Sync 不强制换算 USD 价值。

USD 估值由 OrderPlan / RiskCheck / PriceSnapshot 基于 `mark_price` 计算。

### 6.3 不得混用

禁止：

```text
U 本位用 COIN-M BTC availableBalance 兜底
COIN-M 用 U 本位 USDT availableBalance 兜底
用 walletBalance 替代 marginBalance 作为 current_equity
```

除非后续有明确文档重定义，否则 current_equity 口径应优先使用 marginBalance，而不是 walletBalance。

P1/P2 可增加统一估值口径：

```text
valuation_currency = USD / USDT
valuation_price
valuation_price_source
wallet_balance_value
available_balance_value
unrealized_pnl_value
```

统一估值只服务复盘和跨账户比较，不作为替代原始账户数据的事实来源。

---

## 7. mark_price 与 PriceSnapshot 派生

### 7.1 字段目的

PriceSnapshot P0 可以从 Account Sync 的 positionRisk `markPrice` 派生。

因此 `BinancePositionSnapshot` 必须保留：

```text
mark_price
mark_price_as_of_utc
endpoint = /fapi/v3/positionRisk 或 /dapi/v1/positionRisk
```

### 7.2 语义说明

U 本位：

```text
mark_price = symbol quote price，例如 BTCUSDT 的 USDT 标记价格
```

COIN-M：

```text
mark_price = symbol / pair 的 USD 标记价格，例如 BTCUSD_PERP 的 BTC/USD 标记价格
```

COIN-M 中：

```text
mark_price != contract_size
```

不得混用。

### 7.3 模块边界

Account Sync 不要求直接写 PriceSnapshot 表。

P0 推荐边界：

```text
Account Sync 负责保存 BinancePositionSnapshot.mark_price。
PriceSnapshotService 负责基于 position_snapshot 派生 PriceSnapshot。
```

如实现时选择 Account Sync 同步后自动派生 PriceSnapshot，也必须保持：

```text
PriceSnapshot 是独立模型
PriceSnapshot 有独立 hash
PriceSnapshot 不能反向修改 BinancePositionSnapshot
```

---

## 8. 同步流程与一致性

### 8.1 P0 不引入 Redis

P0 以数据库为唯一事实来源。

Redis 不进入第一版。Redis 可作为 P2 优化项，仅用于 latest_success 缓存或同步锁，不作为账户上下文事实来源。

### 8.2 同步批次流程

P0 同步流程：

```text
1. 读取 BINANCE_ACTIVE_MARKET_TYPE。
2. 校验 active market_type 合法且唯一。
3. 设置 account_domain = active market_type。
4. 创建 BinanceSyncRun(status=running)。
5. 只初始化 active market_type 对应 client。
6. 拉取 active market_type 的 account / balance / positionRisk / exchangeInfo。
7. 标准化字段并构建快照对象。
8. 推断 position_mode。
9. 在 transaction.atomic 中写入：
   - BinanceAccountSnapshot
   - BinanceBalanceSnapshot
   - BinancePositionSnapshot
   - BinanceSymbolRuleSnapshot
   - BinanceSyncRun(status=succeeded)
10. 任一步失败：
   - BinanceSyncRun(status=failed)
   - 写 AlertEvent
```

### 8.3 succeeded 是发布标记

同步任务必须在账户、余额、持仓、symbol rule 快照全部写入成功后，才允许将 `BinanceSyncRun.status` 标记为 `succeeded`。

如果只有部分数据写入成功：

```text
status = failed
不得供 OrderPlan / RiskCheck 使用
写 AlertEvent
```

P0 不产生 `partial_failed` 主状态。

### 8.4 下游如何消费

正式交易链路中，OrderPlan / RiskCheck 必须接收由编排层传入的 `binance_sync_run_id`。

下游只能读取该 `sync_run_id` 对应的一整批快照。

selector 必须校验：

```text
BinanceSyncRun.status = succeeded
BinanceSyncRun.market_type = requested_market_type
BinanceSyncRun.account_domain = requested_account_domain
BinanceSyncRun 未过期
account / balance / position / symbol_rule 快照完整
snapshot_set_hash 可校验
```

如果 `binance_sync_run_id` 缺失、无效、过期、market_type 不匹配、account_domain 不匹配或快照不完整：

```text
selector 返回 unavailable / invalid context
下游 OrderPlan.status = blocked 或 RiskCheckResult.status = BLOCKED
reason_code = invalid_or_stale_binance_sync_run / account_domain_mismatch / market_type_mismatch
写 AlertEvent
```

### 8.5 latest succeeded 的使用限制

`latest succeeded BinanceSyncRun` 只能用于：

```text
开发调试
手动查询
监控展示
非交易流程
明确配置允许的非交易场景
```

正式交易链路不得自动回退到上一条 `succeeded` 的同步批次。

原因：如果当前同步批次仍是 `running`，自动读取上一条 succeeded 可能导致 OrderPlan / RiskCheck 使用过期账户数据。

---

## 9. 新鲜度与过期判断

每个 `BinanceSyncRun` 必须有：

```text
as_of_utc
completed_at_utc
expires_at 或 freshness deadline
```

配置建议：

```text
BINANCE_ACCOUNT_SYNC_MAX_AGE_SECONDS
BINANCE_SYMBOL_RULE_MAX_AGE_SECONDS
```

账户、余额、持仓快照的新鲜度要求通常高于 symbol rule。

如果同步批次过期：

```text
不得供 OrderPlan / RiskCheck 正式链路使用
写 AlertEvent
返回 stale_binance_sync_run
```

同步过期不代表系统可以自动划转或下单。只允许阻断和告警。

### 9.1 API 限频保护

P0 必须实现简单的 Binance REST API 请求间隔控制，避免同步任务过密导致 Binance 限频或封禁风险。

配置项建议：

```text
BINANCE_API_REQUEST_INTERVAL_SECONDS
```

同步服务在连续请求 Binance REST API 时，应至少间隔该配置值。默认值应保守，且不得硬编码在业务逻辑中。

如果收到 HTTP 429、限频、请求权重超限等错误：

```text
停止本次同步
BinanceSyncRun.status = failed
error_code / error_message 非空
写 AlertEvent
由后续调度或人工处理决定是否重试
```

P0 不实现复杂动态限频器。P1/P2 可基于 Binance 官方 request weight、响应 header 和官方文档做增强。

---

## 10. API 权限与安全

### 10.1 P0 权限

P0 API Key 只需要读取账户信息所需权限。

禁止：

```text
trade
withdraw
universal transfer
margin transfer
change leverage
change margin type
```

### 10.2 密钥管理

API Key / Secret 只能来自环境变量或安全配置。

禁止：

```text
写入代码
写入测试 fixture
写入日志
写入 AlertEvent
写入 raw_payload
```

### 10.3 错误处理

以下场景必须写 AlertEvent：

```text
API 权限不足
签名失败
timestamp / recvWindow 错误
HTTP 429 限频
网络超时
响应字段缺失
字段类型异常
active market_type 配置非法
余额不足
可用保证金不足
无 active 账户快照
快照过旧
sync_run failed
```

AlertEvent 不得包含 API secret、签名、完整敏感 header。

---

## 11. 资金不足处理

如果 active market_type 下：

```text
账户无对应资产
available_balance <= 0
available_balance 小于后续风控最低要求
保证金不足
```

本模块应：

```text
正常保存快照
写 AlertEvent 提示资金/保证金不足
不自动划转
不自动补保证金
不尝试读取非 active market_type 作为资金兜底
```

OrderPlan / RiskCheck 后续应基于该快照 blocked 或 denied。

---

## 12. 对外接口 / selector

本模块给 OrderPlan / PriceSnapshot / RiskCheck 提供的 selector 必须以 `binance_sync_run_id` 为主输入。

基础 selector：

```text
get_sync_run(sync_run_id) -> BinanceSyncRun
get_account_snapshot(sync_run_id) -> BinanceAccountSnapshot
get_balance_snapshots(sync_run_id) -> list[BinanceBalanceSnapshot]
get_position_snapshot(sync_run_id, symbol, position_side=None) -> BinancePositionSnapshot
get_symbol_rule_snapshot(sync_run_id, symbol) -> BinanceSymbolRuleSnapshot
get_sync_context_bundle(sync_run_id, symbol) -> BinanceAccountContextBundle
```

新增 / 增强 selector：

```text
get_balance_snapshot_for_asset(sync_run_id, asset) -> BinanceBalanceSnapshot
get_symbol_trading_context(sync_run_id, symbol) -> BinanceSymbolTradingContextBundle
```

`get_symbol_trading_context` 至少返回：

```text
sync_run
account_snapshot
balance_snapshot_for_margin_asset
position_snapshot
symbol_rule_snapshot
position_mode
account_domain
market_type
```

selector 必须校验：

```text
requested_account_domain == sync_run.account_domain
requested_market_type == sync_run.market_type
symbol 属于该 sync_run 的 symbol rule / position 快照
balance asset 与 symbol_rule.margin_asset 匹配
```

如果 `symbol_rule.margin_asset` 找不到对应余额快照：

```text
selector 返回 unavailable
reason_code = margin_asset_balance_missing
```

下游必须 fail-closed。

可以提供非交易用途 selector：

```text
get_latest_succeeded_sync_run(market_type)
```

但必须在文档和代码注释中标明：正式交易链路不得自动 fallback 到 latest succeeded。

---

## 13. 幂等与 hash

每个快照必须有稳定 hash：

```text
snapshot_hash
```

每个同步批次必须有：

```text
snapshot_set_hash
```

hash 应基于标准化后的核心字段和 raw payload 摘要，确保同一批次可复核。

新增字段必须进入对应快照 hash：

```text
account_domain
position_mode
position_mode_observed
observed_exchange_leverage
contract_size
margin_asset
settlement_asset
mark_price
```

其中：

```text
BinanceSyncRun.snapshot_set_hash
```

也必须间接反映这些字段变化。

如果 Binance 原始数据变化导致这些标准字段变化，应生成新的 `snapshot_hash` 与新的 `snapshot_set_hash`。

`BinanceSyncRun` 应有唯一 key，例如：

```text
sync_run_key = hash(exchange, market_type, account_domain, started_at_utc, trigger_source, trace_id)
```

对于重复触发同步：

```text
允许生成新的 sync_run
不得覆盖旧快照
不得修改已 succeeded 的快照核心字段
```

---

## 14. 不可变性

快照一旦生成，不得被后续业务流程原地修改核心字段。

如果 Binance 后续数据变化，应生成新的 `BinanceSyncRun` 和新快照。

允许更新的字段仅限：

```text
非核心审计字段
内部处理状态字段
错误信息字段
```

但 `succeeded` 后的快照核心数据不应被改写。

---

## 15. 健康状态

本模块应提供健康状态查询能力。

健康状态至少包含：

```text
active_market_type
active_account_domain
latest_sync_status
latest_succeeded_sync_run_id
latest_succeeded_as_of_utc
is_fresh
consecutive_failure_count
last_error_code
last_error_message
position_mode
```

如果连续失败超过配置阈值：

```text
写 AlertEvent
health_status = unhealthy
```

P0 可以先做轻量健康状态，不需要复杂熔断状态机。

---

## 16. 同步触发方式

P0 支持：

```text
management command 单次同步
service 单次同步
fixture / fake client 测试模式
```

P1 支持：

```text
Celery Beat 定时同步
```

P2 支持：

```text
User Data Stream / WebSocket
REST 快照 + WebSocket 增量合并
```

---

## 17. 测试与模拟模式

P0 必须支持 fake client / fixture，不得在测试中调用 Binance 真接口。

原有测试应覆盖：

```text
active_market_type = usds_m_futures 时只使用 /fapi endpoint adapter
active_market_type = coin_m_futures 时只使用 /dapi endpoint adapter
非法 active_market_type blocked / failed + AlertEvent
成功同步生成 succeeded BinanceSyncRun
任一必需快照写入失败时 transaction.atomic 回滚，BinanceSyncRun 标记 failed，不可供下游使用
部分 API 成功、部分 API 失败时生成 failed，不可供下游使用
positionRisk 生成 PositionSnapshot
exchangeInfo 生成 SymbolRuleSnapshot
market_type 强隔离
sync_run_id selector 只能读取同一批次快照
正式交易链路不得 fallback latest succeeded
余额不足写 AlertEvent 但不划转
API 权限错误写 AlertEvent
快照 hash 稳定
raw_payload 保存但不泄露 secret
API 请求间隔控制生效
健康状态 selector / service 返回预期字段
sync_run_id 非 succeeded 时健康检查为 unhealthy
```

P0 测试必须覆盖以下 OrderPlan / RiskCheck / PriceSnapshot 依赖场景：

```text
1. U 本位 sync run 写入 account_domain=usds_m_futures。
2. COIN-M sync run 写入 account_domain=coin_m_futures。
3. positionSide 只有 BOTH 时 position_mode=one_way。
4. positionSide 出现 LONG / SHORT 时 position_mode=hedge。
5. positionRisk.leverage 正确写入 observed_exchange_leverage。
6. leverage 缺失 / 0 / 非法时 observed_exchange_leverage=null，不伪造。
7. COIN-M exchangeInfo.contractSize 正确写入 contract_size。
8. COIN-M contractSize 缺失 / <=0 时 selector 对目标 symbol 返回不可消费。
9. marginAsset 可以匹配到对应 BinanceBalanceSnapshot。
10. marginAsset 找不到余额时 selector 返回 margin_asset_balance_missing。
11. BinancePositionSnapshot.mark_price 可用于 PriceSnapshotService 派生。
12. 新字段进入 snapshot_hash；字段变化会导致 snapshot_hash 变化。
13. 正式 selector 不 fallback 到 latest succeeded。
14. active market_type 与 account_domain 不一致时 selector unavailable。
```

---

## 18. P0 范围

P0 必须实现：

```text
1. app: apps/binance_account_sync/
2. env/config 单选 BINANCE_ACTIVE_MARKET_TYPE
3. 支持 usds_m_futures 和 coin_m_futures 两套 REST endpoint adapter
4. 只同步 active market_type / account_domain
5. BinanceSyncRun 批次模型
6. Account / Balance / Position / SymbolRule 数据库快照
7. account_domain 标准字段
8. position_mode 推断与标准化字段
9. observed_exchange_leverage 标准字段
10. COIN-M contract_size 标准字段
11. margin_asset / settlement_asset 标准字段
12. mark_price 标准字段，供 PriceSnapshotService 派生
13. current_equity / available_balance 口径字段
14. transaction.atomic 保证 succeeded 批次完整
15. raw_payload / snapshot_hash / snapshot_set_hash
16. selector 通过 sync_run_id 返回同一批次快照
17. get_balance_snapshot_for_asset / get_symbol_trading_context selector
18. AlertEvent
19. management command 单次同步
20. fake client / fixture 测试
21. 不使用 Redis
22. 不使用内存缓存作为事实来源
23. 简单 API 请求间隔 / 限频保护
24. 不划转、不交易、不下单
```

006B 只输出：

```text
账户事实
余额事实
仓位事实
交易规则事实
同步结果 / 错误状态
```

006B 不输出：

```text
ExchangeOrder
TradeFill
ExecutionResult
订单状态事实
成交状态事实
```

006B 不输出 ExchangeOrder / TradeFill / ExecutionResult；订单状态事实和成交状态事实不归属 Binance Account Sync。

---

## 19. P1 / P2 预留

### P1

```text
Celery Beat 定时同步
SymbolRuleSnapshot 周期性低频更新策略
AccountChangeLog / PositionChangeLog（P0 不实现差异检测，只保存每次成功同步的全量快照）
partial_failed 细分状态
更完整 SyncHealth / 连续失败统计
估值字段 valuation_currency / valuation_value
历史快照归档策略
AlertEvent 路由、聚合、限频策略
```

### P2

```text
Redis latest_success 缓存、查询加速或同步锁
User Data Stream / WebSocket 账户推送
多账户管理
inactive market_type 只读监控
账户变更事件回调
独立审计导出
更复杂熔断恢复机制
```

---

## 20. 明确不做

P0 明确不做：

```text
自动划转
现货/资金账户转入合约账户
合约账户互转
自动补保证金
自动调杠杆
自动调保证金模式
交易下单
撤单
OrderPlan
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
ExecutionPreparation
ExecutionPreparation / Execution
ExchangeOrder
TradeFill
ExecutionResult
订单状态同步
成交状态同步
订单追踪
真实资金调度
PriceSnapshot WebSocket
真实 REST ticker 拉价
User Data Stream
Redis
UI
回测收益计算
```

---

## 21. P0 实施边界

Codex 实现时应严格限定在 Binance Account Sync 的只读账户事实同步边界内：

```text
1. 为 BinanceSyncRun / Account / Balance / Position / SymbolRule 快照补 account_domain。
2. 为 BinanceSyncRun / Account / Position 补 position_mode 标准化字段。
3. 将 PositionSnapshot.leverage 标准化为 observed_exchange_leverage。
4. 为 COIN-M SymbolRuleSnapshot 补 contract_size，并校验 Decimal > 0。
5. 补 margin_asset / settlement_asset 一致性校验。
6. 明确 mark_price 可供 PriceSnapshotService 派生 PriceSnapshot。
7. 增强 selector：get_balance_snapshot_for_asset、get_symbol_trading_context。
8. 更新 snapshot_hash / snapshot_set_hash 计算。
9. 补 fake fixture：U 本位 one-way、COIN-M one-way、hedge mode、缺 contractSize、缺 leverage、marginAsset 无余额。
10. 补测试覆盖 selector 与字段标准化。
```

不得借机实现：

```text
PriceSnapshot WebSocket
真实 REST ticker 拉价
OrderPlan
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
ExecutionPreparation
真实下单
撤单
订单查询
User Data Stream
自动调杠杆
自动调保证金模式
自动划转
```

---

## 22. 验收标准

P0 完成后必须满足：

```text
1. 可以通过配置选择 usds_m_futures 或 coin_m_futures。
2. 运行时只同步 active market_type / account_domain。
3. 成功同步会生成一个完整 succeeded BinanceSyncRun。
4. account / balance / position / symbol rule 快照都带 sync_run_id、market_type、account_domain。
5. OrderPlan / RiskCheck 可通过传入 sync_run_id 获取同一批次完整账户上下文。
6. 没有 succeeded sync_run 或 sync_run 过期时，后续链路必须 blocked。
7. 当正式交易链路传入的 binance_sync_run_id 不存在、market_type 不匹配、account_domain 不匹配、status 不是 succeeded、快照集合不完整或已过期时，下游必须 blocked，不得继续处理，不得自动回退到上一条 succeeded BinanceSyncRun，并必须写 AlertEvent。
8. API 失败、权限错误、余额不足、快照过旧会写 AlertEvent。
9. 不请求、不使用交易/划转/提现权限。
10. 不调用 transfer API。
11. 不生成 OrderPlan / CandidateOrderIntent / RiskCheckResult / ApprovedOrderIntent，不下单。
12. P0 不使用 Redis 或内存缓存作为账户上下文事实来源。
13. P0 具备简单 API 请求间隔 / 限频保护。
14. P0 不实现差异检测，只保存每次成功同步的全量快照。
15. 测试不访问真实 Binance API。
16. 007A OrderPlan 可以从 selector 获取 current_equity、position_mode、position_snapshot、symbol_rule、contract_size。
17. PriceSnapshotService 可以基于 BinancePositionSnapshot.mark_price 派生 PriceSnapshot。
18. 007B RiskCheck 可以从 selector 获取 available_balance、observed_exchange_leverage、margin_asset、contract_size。
19. COIN-M 缺 contract_size 不会被伪造，必须 fail-closed。
20. observed_exchange_leverage 不会被系统配置覆盖或伪造。
21. account_domain / market_type 不一致时 selector 不可消费。
22. 新增字段进入 hash，保证审计可追溯。
```

---

## 23. 最终边界一句话

`Binance Account Sync` 第一版是 **币安专属、只读、单 active account domain、数据库快照化的账户上下文模块**。它为 006C PriceSnapshot、007A OrderPlan 和 007B RiskCheck 提供标准化账户、持仓和交易规则输入，但不做风控批准、不生成订单、不调用交易执行、不进行资金划转，不负责订单状态同步，也不负责成交状态同步。
