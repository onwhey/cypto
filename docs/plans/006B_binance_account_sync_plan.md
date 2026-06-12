# 006B Binance Account Sync / 币安账户信息同步模块实现计划

> 建议路径：`docs/plans/006B_binance_account_sync_plan.md`  
> 对应需求文件：`docs/requirements/binance_account_sync.md`  
> 当前定位：Binance Account Sync 是 `006C PriceSnapshot → 007A OrderPlan → 007B RiskCheck` 的只读账户事实来源。  
> 当前阶段：P0 只读同步。  

---

## 0. 模块定位

006B 只实现 Binance 账户事实同步。

正式链路中 006B 的定位：

```text
Binance Account Sync
= 只读账户事实来源
= PriceSnapshot 派生来源之一
= OrderPlan 当前权益 / 当前持仓 / position_mode / symbol rule / contract_size 来源
= RiskCheck 余额 / observed_exchange_leverage / margin_asset / contract_size / mark_price 来源
```

本阶段输出：

```text
BinanceSyncRun
BinanceAccountSnapshot
BinanceBalanceSnapshot
BinancePositionSnapshot
BinanceSymbolRuleSnapshot
selector / trading context bundle
health service
management command
fake client / fixture tests
```

006B 不生成：

```text
DecisionSnapshot
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
ExecutionOrder
```

---

## 1. 阶段目标

本阶段实现 / 补齐 `Binance Account Sync / 币安账户信息同步模块` 的 P0 能力。

目标是：

```text
从 Binance 官方 API 只读拉取 active market_type 的账户、余额、持仓和交易规则数据，
标准化后写入数据库快照，
形成可被 PriceSnapshot、007A OrderPlan、007B RiskCheck 显式引用的账户上下文批次。
```

本阶段输出：

```text
BinanceSyncRun
BinanceAccountSnapshot
BinanceBalanceSnapshot
BinancePositionSnapshot
BinanceSymbolRuleSnapshot
selector / trading context bundle
health service
management command
fake client / fixture tests
```

本阶段不输出：

```text
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
ExecutionReport
ExecutionOrder
```

---

## 2. 强制边界

### 2.1 只读

本模块只允许读取 Binance 数据，不允许执行任何会改变账户状态的操作。

禁止实现：

```text
下单
撤单
开仓
平仓
资金划转
自动补保证金
调整杠杆
调整保证金模式
提现
OrderPlan
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
Execution
007B 风控批准
仓位计算
```

### 2.2 Binance 专属

app、模型、配置项、服务命名必须显式包含 Binance 语义。

建议 app：

```text
apps/binance_account_sync/
```

不得把 P0 做成泛交易所模块。

### 2.3 单 active market_type / account_domain

运行时必须通过配置选择唯一 active market_type：

```text
BINANCE_ACTIVE_MARKET_TYPE=usds_m_futures
```

或：

```text
BINANCE_ACTIVE_MARKET_TYPE=coin_m_futures
```

P0 合法值仅允许：

```text
usds_m_futures
coin_m_futures
```

P0 中：

```text
account_domain == market_type
```

系统只初始化并同步 active market_type 对应 client。

非 active market_type / account_domain 不参与同步、不参与 OrderPlan、不参与 RiskCheck、不参与订单或交易链路。

---

## 3. 官方文档约束

所有 endpoint、字段映射、权限说明必须以 Binance 官方开发者文档为准。

禁止以第三方 SDK、博客、示例项目作为字段依据。

P0 要支持的官方接口：

```text
USDⓈ-M Futures:
GET /fapi/v3/account
GET /fapi/v3/balance
GET /fapi/v3/positionRisk
GET /fapi/v1/exchangeInfo

COIN-M Futures:
GET /dapi/v1/account
GET /dapi/v1/balance
GET /dapi/v1/positionRisk
GET /dapi/v1/exchangeInfo
```

实现时应在 Binance adapter / mapper 的注释中标明字段来源来自官方 endpoint 文档。

`PositionSnapshot` 必须以 `/positionRisk` 作为主要来源；`/account` 中的 positions 可以保存到 raw_payload 或用于辅助校验，但不得替代 `/positionRisk` 生成正式持仓快照。

---

## 4. 建议代码结构

```text
apps/binance_account_sync/
  __init__.py
  apps.py
  constants.py
  models.py
  hashes.py
  clients.py
  adapters.py
  normalizers.py
  selectors.py
  health.py
  services.py
  management/
    commands/
      sync_binance_account.py
  migrations/
    0001_initial.py
    xxxx_account_domain_position_mode_contract_size.py

tests/binance_account_sync/
  test_models.py
  test_hashes.py
  test_normalizers.py
  test_services.py
  test_selectors.py
  test_health.py
  test_management_commands.py
  fixtures/
    usds_m_account.json
    usds_m_balance.json
    usds_m_position_risk.json
    usds_m_exchange_info.json
    coin_m_account.json
    coin_m_balance.json
    coin_m_position_risk.json
    coin_m_exchange_info.json
```

说明：

```text
clients.py / adapters.py 负责 Binance endpoint client 抽象和 fake client 注入。
normalizers.py 负责 raw payload 到标准字段的映射。
services.py 负责同步流程、事务、AlertEvent。
selectors.py 负责按 sync_run_id 提供快照查询和交易上下文 bundle。
health.py 负责同步健康状态查询。
```

---

## 5. 配置项计划

在 `config/settings.py` 或项目现有配置方式中增加 / 保持：

```text
BINANCE_ACTIVE_MARKET_TYPE
BINANCE_API_KEY
BINANCE_API_SECRET
BINANCE_RECV_WINDOW
BINANCE_API_REQUEST_INTERVAL_SECONDS
BINANCE_ACCOUNT_SYNC_MAX_AGE_SECONDS
BINANCE_SYMBOL_RULE_MAX_AGE_SECONDS
BINANCE_ACCOUNT_SYNC_FAILURE_ALERT_THRESHOLD
BINANCE_ACCOUNT_SYNC_TIMEOUT_SECONDS
```

P0 注意：

```text
API key / secret 不得写入代码、测试 fixture、日志、AlertEvent 或 raw_payload。
BINANCE_ACTIVE_MARKET_TYPE 必须启动时或同步时校验。
P0 不使用 Redis。
P0 不使用内存缓存作为账户上下文事实来源。
```

---

## 6. 数据模型实现计划

### 6.1 BinanceSyncRun

用于表示一次 Binance 账户上下文同步批次。

P0 状态：

```text
running
succeeded
failed
```

P0 不实现 `partial_failed` 主状态。部分 API 成功、部分 API 失败时，整体按 `failed` 处理。`partial_failed` 可作为 P1 预留，但不得被正式交易链路消费。

建议字段：

```text
sync_run_key
exchange
market_type
account_domain
active_market_type_at_runtime
status
position_mode
started_at_utc
completed_at_utc
as_of_utc
expires_at
source
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

关键规则：

```text
account_domain = active_market_type
只有 status=succeeded 才代表该批账户上下文完整可用。
running / failed 不得被正式交易链路使用。
```

---

### 6.2 BinanceAccountSnapshot

保存账户总体状态。

建议字段：

```text
sync_run
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
created_at
```

说明：

```text
current_equity 口径下游优先使用 margin_balance / totalMarginBalance，而不是 wallet_balance。
Account Sync 保存标准字段和 raw_payload，不在本模块做订单仓位计算。
```

---

### 6.3 BinanceBalanceSnapshot

保存资产余额明细。

建议字段：

```text
sync_run
exchange
market_type
account_domain
asset
native_balance
available_balance
cross_wallet_balance
cross_unrealized_pnl
withdraw_available
max_withdraw_amount
margin_balance
wallet_balance
update_time_utc
as_of_utc
endpoint
raw_payload
snapshot_hash
created_at
```

U 本位常见资产：

```text
USDT
USDC
```

COIN-M 常见结算资产：

```text
BTC
ETH
```

禁止跨 market_type / account_domain 使用余额兜底。

---

### 6.4 BinancePositionSnapshot

保存某个 symbol 的持仓状态。

建议字段：

```text
sync_run
exchange
market_type
account_domain
symbol
pair
margin_asset
settlement_asset
position_side
normalized_position_side
position_mode_observed
position_amount
entry_price
break_even_price
mark_price
mark_price_as_of_utc
unrealized_pnl
liquidation_price
leverage
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
created_at
```

标准化要求：

```text
One-way Mode 的 BOTH 应根据 position_amount 标准化为 long / short / none。
Hedge Mode 的 LONG / SHORT 应保留原始 position_side，并生成 normalized_position_side。
position_amount = 0 必须标准化为 none。
position_amount = 0 不等于没有 positionSide；仍应根据 positionSide 推断 position_mode。
```

`observed_exchange_leverage` 来源：

```text
/fapi/v3/positionRisk.leverage
/dapi/v1/positionRisk.leverage
```

解析规则：

```text
leverage 可解析为 Decimal 且 > 0
→ observed_exchange_leverage = parsed value

leverage 缺失 / 空字符串 / 非法 / <= 0
→ observed_exchange_leverage = null
→ 记录 normalization warning / evidence
```

禁止：

```text
不得用系统配置值替代 observed_exchange_leverage。
不得伪造默认杠杆。
不得用 max_target_notional_to_equity_ratio 填充 observed_exchange_leverage。
```

---

### 6.5 BinanceSymbolRuleSnapshot

保存 symbol 交易规则。

建议字段：

```text
sync_run
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
max_leverage
raw_filters
as_of_utc
endpoint
raw_payload
snapshot_hash
created_at
```

P0 必须保存 `raw_filters`，避免 Binance filters 字段变化时丢失信息。

COIN-M：

```text
contract_size 来源于 /dapi/v1/exchangeInfo symbols[].contractSize。
market_type = coin_m_futures 时 contract_size 必须存在、可解析为 Decimal、且 > 0。
不得硬编码 contract_size=100。
不得用 mark_price 替代 contract_size。
```

U 本位：

```text
contract_size 可以为空。
不得使用 contract_size 计算 U 本位目标数量。
```

`settlement_asset`：

```text
P0 中 settlement_asset = margin_asset。
如果 Binance raw payload 没有 settlementAsset 字段，可以由 marginAsset 标准化得到。
```

---

## 7. normalizer / mapper 改造计划

### 7.1 active market_type 到 account_domain

在 normalizer 输入中显式传入：

```text
exchange = binance
market_type = active_market_type
account_domain = active_market_type
```

所有快照统一写入该三元身份：

```text
exchange
market_type
account_domain
```

### 7.2 position_mode 推断

新增 `infer_position_mode(position_rows)` 工具函数。

输入：

```text
positionRisk rows
或 account positions rows
```

输出：

```text
one_way
hedge
unknown
```

推断规则：

```text
如果目标 market_type 的 positionRisk/account positions 仅出现 BOTH：
  position_mode = one_way

如果出现 LONG 或 SHORT：
  position_mode = hedge

如果没有足够 positionSide 信息：
  position_mode = unknown
```

优先使用 `/positionRisk` 的 `positionSide` 字段。

### 7.3 leverage 标准化

新增 `parse_observed_exchange_leverage(raw_value)`。

要求：

```text
合法 Decimal > 0 → 返回 Decimal
非法 / 缺失 / <=0 → 返回 None + warning evidence
```

不得抛出未捕获异常导致整个 normalizer 崩溃；但如果关键结构缺失导致整批不可用，应由 service 层将 sync run 标记为 failed。

### 7.4 contract_size 标准化

新增 `parse_contract_size(raw_value, market_type)`。

规则：

```text
coin_m_futures：缺失 / 非法 / <=0 → 标记 symbol rule 不可用于下游
usds_m_futures：允许为空
```

Account Sync 可以成功保存原始 symbol rule；selector 在目标 symbol 被请求时必须返回明确不可消费原因。

### 7.5 mark_price 标准化

确保 `mark_price` 和 `mark_price_as_of_utc` 可供 PriceSnapshotService 派生价格快照。

字段来源：

```text
/fapi/v3/positionRisk.markPrice
/dapi/v1/positionRisk.markPrice
```

语义：

```text
U 本位：mark_price 是 symbol 的 quote 价格，例如 BTCUSDT 的 USDT 标记价格。
COIN-M：mark_price 是 symbol / pair 的 USD 标记价格，例如 BTCUSD_PERP 的 BTC/USD 标记价格。
COIN-M 中 mark_price != contract_size。
```

### 7.6 balance 口径

U 本位：

```text
current_equity 口径优先使用 totalMarginBalance 或 asset.marginBalance。
available_balance 使用 availableBalance 或 asset.availableBalance。
```

COIN-M：

```text
current_equity_native = assets[marginAsset].marginBalance
available_balance_native = assets[marginAsset].availableBalance
```

Account Sync 不强制做 COIN-M USD 估值；USD 估值由 OrderPlan / RiskCheck 结合 PriceSnapshot 计算。

禁止：

```text
用 walletBalance 替代 marginBalance 作为 current_equity 默认口径。
U 本位和 COIN-M 余额互相兜底。
```

---

## 8. 同步服务实现计划

### 8.1 主流程

`BinanceAccountSyncService.sync_once(...)` 应完成单次同步。

P0 流程：

```text
1. 读取 BINANCE_ACTIVE_MARKET_TYPE
2. 校验 active market_type 合法且唯一
3. 设置 account_domain = active_market_type
4. 创建 BinanceSyncRun(status=running)
5. 只初始化 active market_type 对应 client
6. 在数据库事务外调用 Binance API：
   - account
   - balance
   - positionRisk
   - exchangeInfo
7. 标准化字段并构建待写入快照对象
8. 推断 position_mode
9. 校验本批次基础数据完整性
10. 在 transaction.atomic 中一次性写入所有快照，并将 BinanceSyncRun 更新为 succeeded
11. 任一步失败，BinanceSyncRun 标记为 failed，写 AlertEvent
```

### 8.2 事务边界

外部 Binance API 请求不得放入 `transaction.atomic`。

必须在 `transaction.atomic` 中一次性完成：

```text
BinanceAccountSnapshot 写入
BinanceBalanceSnapshot 写入
BinancePositionSnapshot 写入
BinanceSymbolRuleSnapshot 写入
snapshot_set_hash 计算
BinanceSyncRun(status=succeeded) 更新
```

如果任一必需快照写入失败：

```text
整个批次回滚
不得产生 status=succeeded 的 BinanceSyncRun
事务回滚后，单独将 BinanceSyncRun 标记为 failed
写 AlertEvent
```

### 8.3 succeeded 发布规则

`BinanceSyncRun(status=succeeded)` 是账户上下文批次可被后续交易链路消费的唯一发布标记。

只有 account / balance / positionRisk / symbol rule 快照全部写入成功后，才允许将 sync run 标记为 `succeeded`。

字段级 warning 不一定导致 BinanceSyncRun failed。

建议策略：

```text
API 请求或批次结构失败 → sync_run failed
单个 symbol 的 contract_size 缺失 → sync_run 可以 succeeded，但 selector 对该 symbol 不可消费
目标交易 symbol 的关键字段缺失 → 下游 blocked
```

---

## 9. Binance adapter / fake client 计划

P0 应实现可注入 client，避免测试访问真实 Binance API。

建议接口：

```text
fetch_account()
fetch_balance()
fetch_position_risk()
fetch_exchange_info()
```

active market_type 映射：

```text
usds_m_futures -> fapi adapter
coin_m_futures -> dapi adapter
```

测试中必须使用 fake client / fixture。

---

## 10. API 限频保护

P0 必须实现简单请求间隔控制。

配置项：

```text
BINANCE_API_REQUEST_INTERVAL_SECONDS
```

同步服务在连续请求 Binance REST API 时，应至少间隔该配置值。

如收到 HTTP 429 或 Binance 限频相关错误：

```text
停止本次同步
BinanceSyncRun.status = failed
写 AlertEvent
由后续调度按重试策略处理
```

P0 不实现复杂动态限频器。P1/P2 可基于 Binance 官方 request weight、响应 header 和官方文档增强。

---

## 11. hash 与不可变性

### 11.1 快照 hash

每个快照必须有稳定 `snapshot_hash`。

hash 应基于：

```text
标准化核心字段
raw_payload 摘要
exchange
market_type
account_domain
endpoint
as_of_utc
```

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
mark_price_as_of_utc
```

### 11.2 批次 hash

`BinanceSyncRun` 必须有 `snapshot_set_hash`。

它应基于本批次所有快照 hash 稳定生成。

要求：

```text
字段变化 → snapshot_hash 变化
任一快照 hash 变化 → snapshot_set_hash 变化
```

### 11.3 不可变性

`succeeded` 后的快照核心字段不得被业务流程原地修改。

如果 Binance 后续数据变化，应生成新的 `BinanceSyncRun` 和新快照。

---

## 12. selector / context bundle 实现计划

正式交易链路必须以 `binance_sync_run_id` 为主输入。

### 12.1 基础 selector

保留 / 实现：

```text
get_sync_run(sync_run_id) -> BinanceSyncRun
get_account_snapshot(sync_run_id) -> BinanceAccountSnapshot
get_balance_snapshots(sync_run_id) -> list[BinanceBalanceSnapshot]
get_position_snapshot(sync_run_id, symbol, position_side=None) -> BinancePositionSnapshot
get_symbol_rule_snapshot(sync_run_id, symbol) -> BinanceSymbolRuleSnapshot
get_sync_context_bundle(sync_run_id, symbol) -> BinanceAccountContextBundle
```

### 12.2 新增 selector

新增或增强：

```text
get_balance_snapshot_for_asset(sync_run_id, asset)
get_symbol_trading_context(sync_run_id, symbol, account_domain=None, market_type=None)
```

### 12.3 get_symbol_trading_context 返回结构

建议返回 `BinanceTradingContextBundle` 或等价 dataclass：

```text
sync_run
account_snapshot
balance_snapshot_for_margin_asset
position_snapshot
symbol_rule_snapshot
position_mode
account_domain
market_type
symbol
margin_asset
contract_size
observed_exchange_leverage
mark_price
snapshot_set_hash
```

### 12.4 selector 必须校验

```text
sync_run.status == succeeded
sync_run 未过期
sync_run.market_type == requested_market_type
sync_run.account_domain == requested_account_domain
account snapshot 存在
position snapshot 存在
symbol rule snapshot 存在
balance snapshot 存在
snapshot_set_hash 可追溯
```

COIN-M 额外校验：

```text
symbol_rule.contract_size 存在且 > 0
symbol_rule.margin_asset 存在
margin_asset 能匹配到 balance snapshot
position_snapshot.mark_price 存在且 > 0
```

如果不满足，selector 返回明确不可消费错误，例如：

```text
sync_run_not_consumable
market_type_mismatch
account_domain_mismatch
position_mode_unknown
symbol_rule_missing
contract_size_missing
margin_asset_balance_missing
mark_price_missing
snapshot_set_hash_mismatch
```

下游 OrderPlan / RiskCheck 将这些错误转换为 blocked。

### 12.5 latest succeeded selector 限制

可提供：

```text
get_latest_succeeded_sync_run(market_type)
```

但只能用于：

```text
开发调试
手动查询
监控展示
非交易流程
```

正式交易链路不得自动 fallback 到 latest succeeded。

P0 不要求内存缓存，正式链路直接通过 sync_run_id 查询数据库快照，性能足够。

Redis / 内存缓存只作为 P2 性能优化项，不作为事实来源。

---

## 13. PriceSnapshot 派生边界

### 13.1 Account Sync 的责任

Account Sync 只负责保存：

```text
BinancePositionSnapshot.mark_price
BinancePositionSnapshot.mark_price_as_of_utc
```

### 13.2 PriceSnapshotService 的责任

PriceSnapshot P0 由独立模块负责：

```text
从 BinancePositionSnapshot 派生 PriceSnapshot
生成 price_snapshot_hash
执行 TTL 校验
提供给 OrderPlan / RiskCheck 使用
```

### 13.3 不建议在 006B 中直接写 PriceSnapshot

P0 推荐：

```text
Account Sync 不直接写 PriceSnapshot 表。
PriceSnapshotService 通过 sync_run_id + symbol 显式派生。
```

如果实现时选择在 Account Sync 成功后自动派生，也必须保证：

```text
PriceSnapshot 是独立模型
PriceSnapshot 有独立 hash
PriceSnapshot 不能反向修改 BinancePositionSnapshot
失败不得影响 BinanceSyncRun succeeded 发布，除非明确配置为强依赖
```

本计划默认不要求 006B 自动派生 PriceSnapshot。

---

## 14. 健康状态实现计划

本模块应提供健康状态 service / selector。

建议方法：

```text
get_binance_sync_health(market_type: str) -> dict
get_binance_sync_run_health(sync_run_id: int) -> dict
```

健康状态用于：

```text
编排层判断本轮同步是否成功
OrderPlan / RiskCheck 校验传入 sync_run_id 是否可用
Monitoring / AlertReview 监控同步是否异常
```

健康结果字段至少包含：

```text
active_market_type
account_domain
market_type
position_mode
is_healthy
latest_sync_status
latest_succeeded_sync_run_id
latest_succeeded_as_of_utc
latest_completed_at_utc
age_seconds
is_fresh
is_stale
missing_snapshot_types
missing_contract_size_symbols
missing_mark_price_symbols
missing_leverage_symbols
missing_margin_asset_balance_symbols
consecutive_failure_count
last_error_code
last_error_message
checked_at_utc
```

不健康情况：

```text
没有 succeeded sync run
sync_run_id 不存在
sync_run_id 状态不是 succeeded
sync_run.market_type 与 active market_type 不一致
sync_run.account_domain 与 requested account_domain 不一致
快照集合不完整
snapshot_set_hash 不可追溯
快照过期
连续同步失败超过阈值
```

P0 不实现复杂熔断状态机。

---

## 15. AlertEvent 计划

以下场景必须写 AlertEvent：

```text
active market_type 配置非法
API 权限不足
签名失败
timestamp / recvWindow 错误
HTTP 429 限频
网络超时
响应字段缺失
字段类型异常
同步失败
事务写入失败
快照过旧
余额不足
可用保证金不足
无 active 账户快照
sync_run failed
连续失败超过阈值
```

本次新增或增强以下 warning / evidence：

```text
observed_exchange_leverage 缺失 / 非法 / <=0
COIN-M contract_size 缺失 / 非法 / <=0
position_mode = unknown
margin_asset 找不到余额
mark_price 缺失 / 非法 / <=0
account_domain / market_type 不一致
```

AlertEvent 不得包含：

```text
API secret
签名
敏感 header
完整密钥
```

---

## 16. management command 计划

命令：

```text
python manage.py sync_binance_account
```

建议参数：

```text
--market-type
--trigger-source
--trace-id
--fixture-dir
--dry-run
```

规则：

```text
如果未传 market-type，使用 BINANCE_ACTIVE_MARKET_TYPE。
传入 market-type 时必须与合法值匹配。
dry-run 不写快照、不写 BinanceSyncRun succeeded、不写 AlertEvent。
fixture-dir 用于 fake client / 本地样本数据。
```

补充输出摘要：

```text
account_domain
position_mode
position_count
symbol_rule_count
missing_contract_size_count
missing_mark_price_count
missing_leverage_count
snapshot_set_hash
```

dry-run 输出：

```text
normalized preview
新增字段预览
```

---

## 17. 测试计划

### 17.1 模型 / migration / hash

测试：

```text
BinanceSyncRun 状态枚举
BinanceSyncRun 写入 account_domain
Account / Balance / Position / SymbolRule 快照写入 account_domain
PositionSnapshot 写入 observed_exchange_leverage
SymbolRuleSnapshot 写入 contract_size
snapshot_hash 稳定
snapshot_set_hash 稳定
新增字段进入 snapshot_hash
字段变化导致 snapshot_hash 变化
snapshot_hash 变化导致 snapshot_set_hash 变化
succeeded 后核心快照字段不可变约束
market_type / account_domain 强隔离
```

### 17.2 active market_type

测试：

```text
usds_m_futures 只使用 fapi adapter
coin_m_futures 只使用 dapi adapter
非法 active_market_type -> failed + AlertEvent
非 active market_type 不初始化、不同步
```

### 17.3 U 本位 fixture

新增或修改 fixture，覆盖：

```text
usds_m_futures one-way positionSide=BOTH
leverage 正常解析为 observed_exchange_leverage
markPrice 正常解析为 mark_price
USDT / USDC balance 可被 selector 找到
account_domain=usds_m_futures
contract_size 允许为空
```

### 17.4 COIN-M fixture

新增或修改 fixture，覆盖：

```text
coin_m_futures one-way positionSide=BOTH
exchangeInfo.contractSize 正常写入 contract_size
contract_size > 0
marginAsset 写入 margin_asset / settlement_asset
marginAsset 能匹配到 balance snapshot
markPrice 正常解析
observed_exchange_leverage 正常解析
account_domain=coin_m_futures
```

### 17.5 hedge mode / unknown mode

测试：

```text
positionSide 仅 BOTH → position_mode=one_way
positionSide 出现 LONG / SHORT → position_mode=hedge
没有足够 positionSide 信息 → position_mode=unknown
```

### 17.6 同步成功

测试：

```text
成功同步生成 succeeded BinanceSyncRun
account / balance / position / symbol rule 快照都带同一 sync_run_id
positionRisk 生成 BinancePositionSnapshot
exchangeInfo 生成 BinanceSymbolRuleSnapshot
raw_payload 保存
secret 不进入 raw_payload / AlertEvent
```

### 17.7 事务失败

测试：

```text
任一快照写入失败 -> transaction 回滚
不得生成 succeeded sync run
BinanceSyncRun 标记 failed
写 AlertEvent
selector 不可消费该批次
```

### 17.8 selector / health

测试：

```text
sync_run_id selector 只能读取同一批次快照
sync_run_id 非 succeeded -> unhealthy / unavailable
sync_run 过期 -> unhealthy / unavailable
快照缺失 -> unhealthy / unavailable
COIN-M contractSize 缺失 → selector 对目标 symbol 返回 contract_size_missing
COIN-M contractSize <=0 → selector 对目标 symbol 返回 contract_size_missing
marginAsset 找不到余额 → selector 返回 margin_asset_balance_missing
markPrice 缺失 / <=0 → selector 返回 mark_price_missing
leverage 缺失 / <=0 → observed_exchange_leverage=null，不伪造
account_domain 不一致 → selector 返回 account_domain_mismatch
market_type 不一致 → selector 返回 market_type_mismatch
get_latest_succeeded_sync_run 不得用于正式交易 fallback
```

### 17.9 PriceSnapshot 派生接口支持

测试：

```text
BinancePositionSnapshot.mark_price 可被 PriceSnapshotService 使用
mark_price_as_of_utc 可用于 PriceSnapshot.as_of_utc
COIN-M mark_price 不等于 contract_size
```

该测试可以放在 PriceSnapshot 模块，也可以在 Account Sync selector 测试中验证字段可取。

### 17.10 限频 / API 错误

测试：

```text
连续 API 请求遵守 BINANCE_API_REQUEST_INTERVAL_SECONDS
HTTP 429 -> failed + AlertEvent
权限错误 -> failed + AlertEvent
网络超时 -> failed + AlertEvent
字段缺失 -> failed + AlertEvent
```

### 17.11 资金不足

测试：

```text
available_balance <= 0 仍保存快照
写资金不足 AlertEvent
不调用 transfer API
不调用 trade API
不读取非 active market_type 兜底
```

---

## 18. 建议文件修改清单

预计修改：

```text
apps/binance_account_sync/models.py
apps/binance_account_sync/constants.py
apps/binance_account_sync/normalizers.py
apps/binance_account_sync/hashes.py
apps/binance_account_sync/selectors.py
apps/binance_account_sync/health.py
apps/binance_account_sync/services.py
apps/binance_account_sync/management/commands/sync_binance_account.py
apps/binance_account_sync/migrations/xxxx_account_domain_position_mode_contract_size.py

tests/binance_account_sync/test_models.py
tests/binance_account_sync/test_hashes.py
tests/binance_account_sync/test_normalizers.py
tests/binance_account_sync/test_selectors.py
tests/binance_account_sync/test_health.py
tests/binance_account_sync/test_management_commands.py
tests/binance_account_sync/fixtures/*.json
```

不得修改：

```text
apps/order_plan/
apps/risk_check/
apps/execution/
```

除非只是为了修正 import / type reference，且不得实现下游业务逻辑。

---

## 19. 实施顺序

建议 Codex 按以下顺序提交：

```text
1. 阅读 AGENTS.md、README.md、docs/rules/project_invariants.md、binance_account_sync requirement / plan、price_snapshot requirement / plan、order_plan requirement、risk_check requirement。
2. 增加 constants 与枚举：account_domain、position_mode。
3. 修改 models 与 migrations。
4. 修改 normalizers：account_domain、position_mode、observed_exchange_leverage、contract_size、mark_price。
5. 修改 hashes，确保新增字段进入 hash。
6. 修改 selectors，新增 get_balance_snapshot_for_asset / get_symbol_trading_context。
7. 修改 health / command 输出摘要。
8. 补 fixture。
9. 补测试。
10. 运行 makemigrations check、Django test、pytest。
```

---

## 20. 验收标准

P0 与本次正式链路接入 完成后必须满足：

```text
1. 可以通过配置选择 usds_m_futures 或 coin_m_futures。
2. 运行时只同步 active market_type。
3. 成功同步会生成一个完整 succeeded BinanceSyncRun。
4. account / balance / position / symbol rule 快照都带 sync_run_id。
5. BinanceSyncRun(status=succeeded) 只在全部必需快照写入成功后生成。
6. 事务写入失败时不会留下可消费的 succeeded 批次。
7. PriceSnapshotService 可以基于 BinancePositionSnapshot.mark_price 派生 PriceSnapshot。
8. 007A OrderPlan 可以通过 selector 获取 current_equity、position_mode、position_snapshot、symbol_rule、contract_size。
9. 007B RiskCheck 可以通过 selector 获取 available_balance、observed_exchange_leverage、margin_asset、contract_size。
10. account_domain 与 market_type 均进入快照与 selector 校验。
11. COIN-M contract_size 缺失 / 无效不会被伪造，selector 必须 fail-closed。
12. observed_exchange_leverage 缺失 / 无效不会被系统配置覆盖或伪造。
13. position_mode 能区分 one_way / hedge / unknown。
14. U 本位与 COIN-M 不会互相使用余额、持仓、symbol rule 或 contract_size。
15. 新增字段进入 snapshot_hash 与 snapshot_set_hash。
16. 正式交易 selector 仍然不 fallback 到 latest succeeded。
17. 传入的 sync_run_id 不存在、不是 succeeded、market_type 不匹配、account_domain 不匹配、快照不完整或已过期时，下游必须 blocked，不得继续处理。
18. API 失败、权限错误、余额不足、快照过旧会写 AlertEvent。
19. 不请求、不使用交易/划转/提现权限。
20. 不调用 transfer API。
21. 不生成 OrderPlan / CandidateOrderIntent / RiskCheckResult / ApprovedOrderIntent，不下单。
22. P0 不使用 Redis 或内存缓存作为账户上下文事实来源。
23. P0 不实现差异检测，只保存每次成功同步的全量快照。
24. P0 实现简单 API 请求间隔保护。
25. 测试不访问真实 Binance API。
```

---

## 21. P1 / P2 预留

### P1

```text
Celery Beat 定时同步
SymbolRuleSnapshot 周期性低频更新策略
AccountChangeLog / PositionChangeLog
partial_failed 细分状态
更完整 SyncHealth / 连续失败统计
估值字段 valuation_currency / valuation_value
历史快照归档策略
```

### P2

```text
Redis latest_success 缓存或同步锁
User Data Stream / WebSocket 账户推送
多账户管理
inactive market_type 只读监控
账户变更事件回调
独立审计导出
更复杂熔断恢复机制
动态 request weight 限频器
```

---

## 22. 明确不做

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
Execution
007B 风控批准
真实资金调度
WebSocket
Redis
内存缓存事实来源
UI
回测收益计算
```

---

## 23. Codex 实现注意事项

实现时必须：

```text
严格读取 AGENTS.md、README.md、docs/rules/project_invariants.md
严格读取 docs/requirements/binance_account_sync.md
严格读取 docs/plans/006B_binance_account_sync_plan.md
不修改 requirements / plans 文档，除非发现明确冲突并先汇报
不访问真实 Binance API 运行测试
不引入第三方接口字段口径
不把 API key / secret 写进日志、AlertEvent、raw_payload
不实现 P1 / P2 功能
不实现交易、划转、下单、风控批准
```

完成后必须运行：

```powershell
.\.venv\Scripts\python.exe manage.py makemigrations --check --dry-run
.\.venv\Scripts\python.exe manage.py test
.\.venv\Scripts\pytest.exe
```

---

## 24. 完成后汇总

完成后请汇总：

```text
1. 修改了哪些模型字段。
2. 新增 / 修改了哪些 migration。
3. account_domain 如何写入和校验。
4. position_mode 如何推断。
5. observed_exchange_leverage 如何标准化，缺失时如何处理。
6. COIN-M contract_size 如何写入和校验。
7. margin_asset / settlement_asset 如何匹配余额。
8. PriceSnapshot 派生需要的 mark_price 是否已暴露。
9. snapshot_hash / snapshot_set_hash 是否包含新增字段。
10. 新增 / 修改了哪些 selector。
11. 新增 / 修改了哪些 fixture 和测试。
12. 测试结果。
13. 是否发现需求 / plan 与当前实现冲突。
14. 是否有任何越界实现。
```
