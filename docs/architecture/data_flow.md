# 数据流与对象流转说明

本文档定义中低频趋势跟踪自动交易系统在一次运行过程中的数据流、对象流转、条件分支、异步辅助链路和阻断规则。

---

## 1. 文档目的

本文档定义中低频趋势跟踪自动交易系统在一次运行过程中的数据流、对象流转、条件分支、异步辅助链路和阻断规则。

本文档用于回答：

```text
一次策略周期从哪里开始？
每一步读取什么对象？
每一步生成什么对象？
哪些对象必须追溯到上游对象？
哪些环节是主链路必经步骤？
哪些环节是条件触发的补偿链路？
哪些环节可以异步处理？
哪些失败必须阻断下游？
trace_id、trigger_source 和业务幂等键如何传递？
回测与实盘的数据流哪里一致、哪里不同？
订单意图、风控审批、执行准备、真实执行的边界在哪里？
```

本文档不定义：

```text
具体数据库字段
具体 Django app 结构
具体 Celery task 名称
具体函数名
具体策略公式
具体特征算法
具体原子信号算法
具体订单状态机实现
```

模块职责、输入输出、允许调用和禁止调用，以：

```text
docs/architecture/01_module_map.md
```

为准。

系统工程架构、存储、gateway、Celery、同步异步边界，以：

```text
docs/architecture/system_architecture.md
```

为准。

系统红线和不变量，以：

```text
docs/rules/project_invariants.md
```

为准。

---

## 2. 数据流分类

系统数据流分为四类：

```text
正常主链路
条件补偿链路
异步辅助链路
阻断链路
```

### 2.1 正常主链路

正常主链路是一次策略周期在无异常情况下必须经过的对象流转。

核心主链路：

```text
MarketData / KlineRecord
→ DataQualityResult
→ MarketSnapshot
→ FeatureSet / FeatureValue
→ AtomicSignal
→ DomainSignal
→ MarketRegime
→ StrategyRoute
→ StrategyParamSnapshot
→ StrategySignal
→ StrategySignalQualityResult
→ DecisionSnapshot
→ BinanceSyncRun / Binance Account Sync 快照
→ PriceSnapshot
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheckResult
→ ApprovedOrderIntent
→ ExecutionPreparation
→ ExecutionPreparationResult / PreparedExecutionRequest
→ ExecutionResult
→ ExchangeOrder
→ TradeFill
→ PositionState
→ ReviewRecord
```

注意：

```text
DecisionSnapshot 不表达 ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE 订单动作。
DecisionSnapshot 只表达目标仓位意图，例如 target_position_ratio / target_intent。
不是每个 DecisionSnapshot 都会生成 CandidateOrderIntent。
不是每个 CandidateOrderIntent 都会通过 RiskCheck。
不是每个 ApprovedOrderIntent 都会真实下单。
不是每次运行都需要数据回补。
不是每个流程都会调用 Hermes 或大模型。
```

### 2.2 条件补偿链路

条件补偿链路只在特定条件触发时运行。

例如：

```text
数据质量失败 → 数据回补 → 数据质量复检
Account Sync 失败或过期 → 重新同步账户快照
PriceSnapshot 过期 → 刷新或重新派生价格快照
OrderPlan 检测 active_order_exists → 阻断并等待执行/追踪模块处理
RiskCheck 拒绝 primary 反手 → 检查 fallback_reduce_only_intent
ExecutionPreparation 价格偏离过大 → 要求重新 OrderPlan / RiskCheck 或人工处理
订单状态未知 → 订单状态补偿查询
仓位不一致 → 仓位强制同步 / 人工恢复
风控熔断 → 人工确认后恢复
```

条件补偿链路不得绕过主链路中的强制检查。

### 2.3 异步辅助链路

异步辅助链路可以后置处理，不应阻塞主交易判断。

例如：

```text
Hermes / Alert 通知投递
ReviewRecord 聚合
ModelReviewRecord 生成
策略表现统计
事件日历同步
非阻塞审计导出
```

异步辅助链路不得反向触发真实交易。

注意：

```text
订单相关正式事件必须写 AlertEvent。
通知投递失败可以异步重试，但不得改变订单事实或交易事实。
```

### 2.4 阻断链路

阻断链路表示某一步失败后，不允许继续进入下游关键环节。

例如：

```text
数据质量失败 → 阻断 MarketSnapshot
FeatureQualityCheck 失败 → 阻断对应 AtomicSignal
AtomicSignalQualityCheck 失败 → 阻断或降级 DomainSignal
DomainQualityCheck 失败 → 阻断或降级 MarketRegime / StrategyRoute
StrategySignalQualityResult FAIL → 阻断 DecisionSnapshot
DecisionSnapshot 不可用 / 过期 → 阻断 OrderPlan
BinanceSyncRun 不可消费 → 阻断 PriceSnapshot / OrderPlan / RiskCheck
PriceSnapshot 缺失或过期 → 阻断 OrderPlan / RiskCheck / ExecutionPreparation
OrderPlan blocked → 不生成 CandidateOrderIntent
RiskCheck DENY / BLOCKED / FAILED → 不生成 ApprovedOrderIntent
ExecutionPreparation price guard 失败 → 不提交真实订单
Execution 状态未知 → 进入异常同步 / 人工恢复
```

---

## 3. 正常策略周期主链路

一次正常策略周期应按以下顺序流转。

```text
调度触发
→ 数据准备
→ 数据质量检查
→ MarketSnapshot
→ FeatureLayer
→ AtomicSignal
→ DomainSignal
→ MarketRegime
→ StrategyRoute
→ StrategyParamSnapshot
→ StrategySignal
→ StrategySignalQualityResult
→ DecisionSnapshot
→ 账户事实与价格事实准备
→ OrderPlan
→ RiskCheck
→ ExecutionPreparation
→ Execution
→ 通知与复盘链路
```

### 3.1 调度触发

输入：

```text
symbol
timeframe
analysis_close_time_utc
strategy_id 或 strategy_route
execution_mode
trigger_source
trace_id
```

输出：

```text
TaskRunRecord
业务幂等键
```

推荐分析周期幂等键：

```text
(symbol, timeframe, analysis_close_time_utc, strategy_id, execution_mode)
```

规则：

```text
trace_id 用于追踪一次运行。
业务幂等键用于防止同一业务周期重复执行。
trace_id 不得单独作为业务幂等键。
重复触发时，应复用已有结果或明确跳过，不得重复下单。
```

---

## 4. 数据准备流

### 4.1 正常数据采集流

正常情况下：

```text
调度触发
→ 数据采集
→ MarketData / KlineRecord
→ 数据质量检查
```

输入：

```text
symbol
timeframe
采集时间范围
数据源配置
trace_id
trigger_source
```

输出：

```text
MarketData
KlineRecord
DataSourceRecord
TaskRunRecord
```

规则：

```text
数据采集只负责获取可信数据源中的市场数据。
数据采集不得生成交易信号。
数据采集不得访问交易所交易接口。
数据采集失败不得伪造数据。
WebSocket 不参与 K 线采集。
```

### 4.2 数据质量检查流

```text
MarketData / KlineRecord
→ DataQualityResult
```

检查内容包括：

```text
K 线缺失
K 线重复
时间连续性
收盘状态
价格异常
成交量异常
数据源可追溯性
```

输出：

```text
DataQualityResult
DataQualityIssue
AlertEvent
```

分支：

```text
DataQualityResult = PASS
→ 允许进入 MarketSnapshot

DataQualityResult = FAIL
→ 写 DataQualityIssue
→ 写 AlertEvent
→ 如属于可回补问题，则创建 BackfillRequest
→ 阻断当前分析流程
→ 不生成 MarketSnapshot
```

---

## 5. 数据回补条件分支

数据回补不是正常主链路的必经步骤。

只有当数据质量检查发现缺失、断档、异常或采集失败时，才创建 BackfillRequest，并由 data_backfill 或后续统一编排层处理。

### 5.1 正常无回补路径

```text
数据采集
→ 数据质量检查
→ PASS
→ MarketSnapshot
```

### 5.2 数据回补路径

```text
数据采集
→ 数据质量检查
→ DataQualityIssue / AlertEvent
→ BackfillRequest
→ data_backfill
→ BackfillRun 完成后，标记 / 要求 DataQuality 复检
→ 后续统一编排层、recovery scan 或人工命令重新执行 DataQuality
→ 新的 DataQualityResult = PASS 且 allows_downstream=True
→ 继续 MarketSnapshot
```

输入：

```text
DataQualityIssue
缺失时间范围
symbol
timeframe
trigger_source
trace_id
```

输出：

```text
BackfillResult
KlineRecord
复检标记 / recheck_required 状态
TaskRunRecord
```

DataQualityResult 由后续 DataQuality 复检产生，不属于 data_backfill 输出。

规则：

```text
数据回补不得伪造 K 线。
数据回补不得使用推测值补 K 线。
数据回补不得绕过数据质量复检。
BackfillRun 完成后，data_backfill 只标记 / 暴露需要 DataQuality 复检的状态。
data_backfill 不直接投递 DataQuality task，不同步调用 DataQuality，不等待 DataQuality 结果。
实际复检由后续统一编排层、recovery scan 或人工命令重新执行。
只有新的 DataQualityResult = PASS 且 allows_downstream=True，才允许继续 MarketSnapshot。
回补完成不能直接放行 MarketSnapshot。
回补失败必须记录错误码、错误消息、缺失范围和 trace_id。
回补后仍未通过质量检查时，必须阻断 MarketSnapshot。
KlineWriteLock 属于 apps.market_data，由 002B 实现。
data_collection 和 data_backfill 必须共用 KlineWriteLock 保护 Kline4h / Kline1d 主事实表写入。
MarketSnapshot 只检查 KlineWriteLock 是否活跃，不创建锁。
```

---

## 6. MarketSnapshot 流

MarketSnapshot 固定某一策略分析时刻的市场状态。

```text
4h DataQualityResult = PASS
+ 1d DataQualityResult = PASS
+ latest_4h_open_time_utc 符合理论最新已收盘 4h
+ latest_1d_open_time_utc 符合当前理论最新已收盘 1d
→ MarketSnapshot
```

输入：

```text
已通过质量检查的 MarketData / KlineRecord
DataQualityResult
symbol
timeframe
analysis_close_time_utc
trace_id
trigger_source
```

输出：

```text
MarketSnapshot
SnapshotBuildRecord
```

规则：

```text
MarketSnapshot 必须追溯到原始 K 线和数据质量结果。
MarketSnapshot 是 FeatureLayer、AtomicSignal、StrategySignal、DecisionSnapshot 的共同证据基础。
MarketSnapshot 生成失败时，不得进入 FeatureLayer。
MarketSnapshot 不等于 PriceSnapshot。
MarketSnapshot 使用分析周期数据，不用于最终执行价格保护。
```

---

## 7. FeatureLayer 流

FeatureLayer 基于 MarketSnapshot 统一计算特征。

```text
MarketSnapshot
→ FeatureLayer
→ FeatureSet / FeatureValue / FeatureQualityCheck
```

输入：

```text
MarketSnapshot
特征配置
特征参数版本
trace_id
```

输出：

```text
FeatureSet
FeatureValue
FeatureQualityCheck
```

规则：

```text
同一 MarketSnapshot 下，同一 feature_key、timeframe、params、version 的特征结果只计算一次。
同一策略周期内，FeatureLayer 应提供内存级复用能力。
影响原子信号和交易决策的关键特征必须可追溯。
FeatureQualityCheck 未通过时，对应特征不得进入 AtomicSignal。
```

阻断：

```text
关键 FeatureQualityCheck 失败
→ 阻断对应 AtomicSignal

非关键 FeatureQualityCheck 失败
→ 记录降级原因，下游可按配置降级
```

---

## 8. AtomicSignal 流

AtomicSignal 基于 FeatureValue 生成最小市场判断。

```text
FeatureSet / FeatureValue
→ AtomicSignal
→ AtomicSignalQualityCheck
```

输入：

```text
FeatureSet
FeatureValue
原子信号配置
参数版本
trace_id
```

输出：

```text
AtomicSignal
AtomicSignalQualityCheck
```

规则：

```text
AtomicSignal 必须引用 FeatureSet / FeatureValue。
AtomicSignal 不得绕过 FeatureLayer 重新计算基础指标。
AtomicSignal 不得调用其他 AtomicSignal。
AtomicSignal 不得生成 StrategySignal、DecisionSnapshot、OrderPlan、CandidateOrderIntent 或交易动作。
```

阻断：

```text
AtomicSignalQualityCheck 失败
→ 不得进入 DomainSignal

关键 AtomicSignal 缺失
→ 下游必须阻断或输出不可交易 / 不确定状态
```

---

## 9. DomainSignal 与 MarketRegime 流

### 9.1 DomainSignal 流

```text
AtomicSignal
→ DomainSignal
→ DomainQualityCheck
```

输入：

```text
AtomicSignal
AtomicSignalQualityCheck
领域配置
trace_id
```

输出：

```text
DomainSignal
DomainQualityCheck
```

规则：

```text
DomainSignal 聚合同类 AtomicSignal。
DomainSignal 可解释信号之间的确认、冲突、互斥、否定和降权关系。
DomainSignal 不得直接生成 OrderPlan / CandidateOrderIntent。
DomainSignal 不得直接访问交易所。
```

### 9.2 MarketRegime 流

```text
DomainSignal
→ MarketRegime
```

输入：

```text
DomainSignal
DomainQualityCheck
MarketSnapshot
trace_id
```

输出：

```text
MarketRegime
RegimeQualityStatus
```

规则：

```text
MarketRegime 用于辅助 StrategyRoute 和 StrategySignal。
MarketRegime 不直接下单。
环境识别失败时，应输出 UNKNOWN 或 NOT_TRADABLE。
市场状态不明时不得默认允许交易。
```

---

## 10. 策略流

### 10.1 StrategyRoute 流

```text
MarketRegime + DomainSignal + 配置
→ StrategyRoute
```

输入：

```text
MarketRegime
DomainSignal
策略配置
trace_id
```

输出：

```text
StrategyRoute
```

规则：

```text
第一阶段可以只有默认趋势跟踪路由。
无匹配策略时输出 NO_ROUTE 或 NO_TRADE 路由。
不得因为路由失败而默认执行某策略。
```

### 10.2 StrategyParamSnapshot 流

```text
策略配置 / 参数来源
→ StrategyParamSnapshot
```

输入：

```text
策略配置文件
环境配置
人工确认参数版本
trace_id
```

输出：

```text
StrategyParamSnapshot
```

规则：

```text
策略参数必须有明确来源。
StrategySignal 和 DecisionSnapshot 必须能追溯到当时使用的参数。
参数缺失或版本不明时不得生成 StrategySignal。
```

### 10.3 StrategySignal 流

```text
StrategyRoute + MarketRegime + DomainSignal + AtomicSignal + StrategyParamSnapshot
→ StrategySignal
→ StrategySignalQualityResult
```

输入：

```text
StrategyRoute
MarketRegime
DomainSignal
AtomicSignal
StrategyParamSnapshot
trace_id
```

输出：

```text
StrategySignal
StrategySignalQualityResult
```

规则：

```text
StrategySignal 可以表达交易倾向、强度、信心和证据摘要。
StrategySignal 不是订单。
StrategySignal 不得直接进入 Execution。
StrategySignal 必须先经过 StrategySignalQualityResult。
StrategySignalQualityResult 不通过时，不得生成可用 DecisionSnapshot。
```

---

## 11. DecisionSnapshot 流

DecisionSnapshot 是系统每个策略分析周期的目标仓位决策快照。

```text
StrategySignal + StrategySignalQualityResult + 上游证据链 + DecisionPolicy
→ DecisionSnapshot
```

输入：

```text
MarketSnapshot
FeatureSet
AtomicSignal
DomainSignal
MarketRegime
StrategyRoute
StrategyParamSnapshot
StrategySignal
StrategySignalQualityResult
DecisionPolicy / decision_config
trace_id
trigger_source
```

输出：

```text
DecisionSnapshot
```

规则：

```text
每个策略分析周期必须生成 DecisionSnapshot。
DecisionSnapshot 只表达目标仓位意图，例如 target_intent / target_position_ratio。
DecisionSnapshot 不读取当前账户余额。
DecisionSnapshot 不读取当前持仓。
DecisionSnapshot 不消费 BinanceSyncRun。
DecisionSnapshot 不生成具体订单动作。
DecisionSnapshot 不输出 ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE 作为下游订单动作。
DecisionSnapshot 不等于 CandidateOrderIntent。
DecisionSnapshot 不得直接调用交易所。
DecisionSnapshot 必须用于后续复盘。
```

分支：

```text
DecisionSnapshot 可用，target_position_ratio 有效
→ 进入 OrderPlan 链路

DecisionSnapshot target_position_ratio = 0
→ 仍可进入 OrderPlan，由 OrderPlan 判断是否需要平仓、减仓或 no_order_required

DecisionSnapshot BLOCKED / 不可用 / 过期
→ 不进入 OrderPlan
→ 记录阻断原因并进入复盘链路
```

---

## 12. 账户事实与价格事实准备

交易前置链路开始前，需要准备账户事实和价格事实。

### 12.1 Binance Account Sync 流

```text
Binance REST 只读接口
→ BinanceSyncRun
→ BinanceAccountSnapshot / BinanceBalanceSnapshot / BinancePositionSnapshot / BinanceSymbolRuleSnapshot
```

输入：

```text
BINANCE_ACTIVE_MARKET_TYPE
account_domain
symbol
trigger_source
trace_id
```

输出：

```text
BinanceSyncRun
BinanceAccountSnapshot
BinanceBalanceSnapshot
BinancePositionSnapshot
BinanceSymbolRuleSnapshot
BinanceTradingContextBundle / selector result
```

规则：

```text
Binance Account Sync 是只读账户事实来源。
只同步 active market_type / account_domain。
不得下单、撤单、划转、调杠杆、调保证金模式。
只有 status=succeeded 的 BinanceSyncRun 可被正式交易链路消费。
正式交易链路必须显式传入 binance_sync_run_id。
不得自动 fallback 到 latest succeeded。
account_domain、market_type、symbol 必须一致。
COIN-M 必须保留 contract_size、margin_asset、mark_price、observed_exchange_leverage 等字段。
```

阻断：

```text
BinanceSyncRun 缺失 / 失败 / 过期 / market_type 不一致 / account_domain 不一致 / 快照不完整
→ 阻断 OrderPlan / RiskCheck
→ 写 AlertEvent
```

### 12.2 PriceSnapshot 流

```text
BinancePositionSnapshot.mark_price / REST mark price / fixture
→ PriceSnapshotService
→ PriceSnapshot
```

输入：

```text
symbol
market_type
account_domain
mark_price
price_source
as_of_utc
binance_sync_run_id 或 source object id
trace_id
```

输出：

```text
PriceSnapshot
price_snapshot_hash
```

规则：

```text
PriceSnapshot 是价格事实层，不是 K 线，不是成交价。
PriceSnapshot 为 OrderPlan、RiskCheck、ExecutionPreparation 提供可追溯价格依据。
OrderPlan / RiskCheck / ExecutionPreparation 不应直接依赖 WebSocket 内存价格。
WebSocket 未来只是 PriceSnapshot 的数据来源之一。
P0 可以从 BinancePositionSnapshot.mark_price 派生 PriceSnapshot。
PriceSnapshot 缺失或过期时，不得用 4h / 1d K 线收盘价替代。
```

PriceSnapshot P0 不实现：

```text
WebSocket
盘口深度
滑点模型
真实成交价
最终执行 price guard
```

---

## 13. OrderPlan 流

OrderPlan 是把 DecisionSnapshot 的目标仓位转换为候选订单意图的模块。

```text
DecisionSnapshot
+ BinanceSyncRun / BinanceTradingContextBundle
+ PriceSnapshot
→ OrderPlan
→ CandidateOrderIntent
```

输入：

```text
DecisionSnapshot
binance_sync_run_id
BinanceTradingContextBundle
PriceSnapshot
symbol
market_type
account_domain
max_target_notional_to_equity_ratio
min_rebalance_notional
trace_id
trigger_source
```

输出：

```text
OrderPlan
CandidateOrderIntent(primary_intent)
CandidateOrderIntent(fallback_reduce_only_intent，可选)
order_components
```

规则：

```text
OrderPlan 负责读取当前账户、当前持仓、symbol rule 和 PriceSnapshot。
OrderPlan 负责把 target_position_ratio 转换为目标数量 / 合约张数。
OrderPlan 负责计算当前仓位与目标仓位的差值。
OrderPlan 负责生成 CandidateOrderIntent 和 order_components。
OrderPlan 不做最终风控批准。
OrderPlan 不真实下单。
OrderPlan 不调用 Binance API 修改账户环境。
OrderPlan 不使用 observed_exchange_leverage 计算目标仓位。
OrderPlan 使用 max_target_notional_to_equity_ratio 定义系统内部目标名义仓位。
```

### 13.1 One-Way Mode

P0 只支持：

```text
position_mode = one_way
positionSide = BOTH
```

如果检测到 Hedge Mode：

```text
OrderPlan.status = blocked
blocked_reason = hedge_mode_not_supported
allows_downstream = false
写 AlertEvent
```

### 13.2 净额反手与 fallback

One-Way Mode 下允许净额反手。

示例：

```text
当前 long +0.8
目标 short -0.5
```

OrderPlan 可以生成：

```text
primary_intent: SELL 1.3, reduceOnly=false, positionSide=BOTH
components: close_long 0.8 + open_short 0.5

fallback_reduce_only_intent: SELL 0.8, reduceOnly=true, positionSide=BOTH
components: close_long 0.8
```

规则：

```text
fallback_reduce_only 只由 OrderPlan 预生成。
RiskCheck 可以选择 fallback，但不得任意修改订单数量。
```

---

## 14. RiskCheck 流

RiskCheck 是候选订单进入执行准备前的强制风控闸门。

```text
OrderPlan
+ CandidateOrderIntent
+ BinanceSyncRun / BinanceTradingContextBundle
+ PriceSnapshot
+ RiskRuleDefinition
→ RiskCheckResult
```

输入：

```text
OrderPlan
CandidateOrderIntent
primary_candidate_order_intent
fallback_candidate_order_intent（可选）
order_components
binance_sync_run_id
PriceSnapshot
RiskRuleDefinition / risk_config
trace_id
trigger_source
```

输出：

```text
RiskCheckResult
RiskRuleResult / RiskCheckIssue
AlertEvent
```

状态：

```text
ALLOW
DENY
BLOCKED
FAILED
```

规则：

```text
RiskCheck 只检查 CandidateOrderIntent，不直接消费 DecisionSnapshot。
RiskCheck 不使用 ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE 作为交易动作输入。
RiskCheck 不生成目标仓位。
RiskCheck 不做任意 MODIFY，不自动缩小订单。
RiskCheck 不真实下单。
RiskCheck 不调用 Binance API 修改账户环境。
RiskCheck 使用 observed_exchange_leverage 仅估算 increase_risk component 的新增保证金。
reduce_risk component 不因 observed_exchange_leverage 缺失而阻断。
RiskCheck 使用 PriceSnapshot 做 notional / margin_required 估算。
RiskCheck 的估值 PriceSnapshot 必须满足 RiskCheck TTL。
```

### 14.1 RiskCheck 分支

```text
primary_intent 全部通过
→ RiskCheckResult = ALLOW primary_intent
→ 可进入 ApprovedOrderIntent

primary_intent 的 increase_risk component 不通过，fallback_reduce_only_intent 通过
→ RiskCheckResult = ALLOW fallback_reduce_only_intent
→ 可进入 ApprovedOrderIntent

输入完整但风险约束不允许
→ RiskCheckResult = DENY
→ 不生成 ApprovedOrderIntent

前置条件不满足 / 数据缺失 / 快照不一致
→ RiskCheckResult = BLOCKED
→ 不生成 ApprovedOrderIntent

系统异常
→ RiskCheckResult = FAILED
→ 不生成 ApprovedOrderIntent
```

正式 RiskCheck 结果必须写 AlertEvent。

---

## 15. ApprovedOrderIntent 流

ApprovedOrderIntent 是通过 RiskCheck 的订单意图。

```text
RiskCheckResult(ALLOW)
→ ApprovedOrderIntent
```

输入：

```text
RiskCheckResult
selected_candidate_order_intent_id
OrderPlan
PriceSnapshot
BinanceSyncRun
trace_id
trigger_source
```

输出：

```text
ApprovedOrderIntent
```

规则：

```text
ApprovedOrderIntent 表示订单意图已通过风控，可以进入 ExecutionPreparation。
ApprovedOrderIntent 不表示已经真实下单。
ApprovedOrderIntent 不表示已经成交。
ApprovedOrderIntent 不保证一定能成交。
ApprovedOrderIntent 的具体模块编号后续单独确定。
```

---

## 16. ExecutionPreparation 流

ExecutionPreparation 是真实下单前的最终准备与价格保护层。

```text
ApprovedOrderIntent
+ 最新 PriceSnapshot
→ ExecutionPreparationResult
```

输入：

```text
ApprovedOrderIntent
selected CandidateOrderIntent
RiskCheckResult
latest PriceSnapshot
execution_mode
trace_id
trigger_source
```

输出：

```text
ExecutionPreparationResult
PreparedExecutionRequest
```

规则：

```text
ExecutionPreparation 不重新判断策略方向。
ExecutionPreparation 不重新制定目标仓位。
ExecutionPreparation 必须检查 ApprovedOrderIntent 未过期。
ExecutionPreparation 必须刷新或读取最新 PriceSnapshot。
ExecutionPreparation 执行最终 price guard。
```

最终 price guard：

```text
latest_price_snapshot_age <= 30 秒
price_deviation <= 0.5%
```

说明：

```text
30 秒不是指 OrderPlan / RiskCheck 后必须 30 秒内下单。
30 秒是执行准备真正准备发单时，最新价格快照的最大年龄。
```

分支：

```text
price guard 通过
→ 进入 Execution

PriceSnapshot 刷新失败 / 过期
→ BLOCKED
→ 写 AlertEvent

价格偏离超过 0.5%
→ BLOCKED
→ 要求重新 OrderPlan / RiskCheck 或人工处理
→ 写 AlertEvent
```

---

## 17. 执行链路

### 17.1 Execution 流

```text
ExecutionPreparationResult
→ Execution
→ ExecutionResult
→ ExchangeOrder
```

输入：

```text
ExecutionPreparationResult
PreparedExecutionRequest
ApprovedOrderIntent
execution_mode
交易所配置
trace_id
```

输出：

```text
ExecutionResult
ExchangeOrder
Exchange response summary
```

规则：

```text
Execution 是唯一真实交易提交入口。
Execution 不负责生成策略信号。
Execution 不得绕过 RiskCheck。
Execution 不得绕过 ExecutionPreparation。
Execution 不得自动调整杠杆。
Execution 不得让 Hermes 或大模型触发交易。
下单成功或失败都必须写 AlertEvent。
ExecutionResult 记录执行尝试结果，不等同于交易所订单状态。
ExchangeOrder 记录交易所订单状态，不等同于成交记录。
```

### 17.2 execution_mode 分支

#### dry-run

```text
ApprovedOrderIntent
→ ExecutionPreparation(dry-run)
→ Execution(dry-run)
→ ExecutionResult(dry-run)
```

规则：

```text
不调用交易所交易接口。
不生成真实 ExchangeOrder。
不生成真实 TradeFill。
不更新真实 PositionState。
只记录预期行为和校验结果。
```

#### paper trading

```text
ApprovedOrderIntent
→ ExecutionPreparation(paper)
→ ExecutionResult(paper)
→ 模拟 ExchangeOrder
→ 模拟 TradeFill
→ 模拟 PositionState
```

规则：

```text
必须明确 execution_mode = paper。
paper 订单、成交、仓位不得参与 real trading 风控。
paper 状态必须与 real 状态隔离。
```

#### real trading

```text
ApprovedOrderIntent
→ ExecutionPreparation(real)
→ trading_gateway / Binance REST New Order
→ ExecutionResult
→ ExchangeOrder
→ TradeFill
→ PositionState
```

规则：

```text
必须真实交易开关开启。
必须订单提交开关开启。
必须 RiskCheckResult = ALLOW。
必须 ExecutionPreparation 通过。
必须 ApprovedOrderIntent 未执行过。
必须 execution_mode = real。
```

---

## 18. 订单、成交与仓位流

### 18.1 ExchangeOrder 流

```text
ExecutionResult / 交易所订单回报
→ ExchangeOrder
```

规则：

```text
ExecutionResult 记录执行尝试结果，不等同于交易所订单状态。
ExchangeOrder 是交易所订单状态记录。
ExchangeOrder 不等同于成交记录。
订单状态必须追溯到 ApprovedOrderIntent / RiskCheckResult / CandidateOrderIntent / OrderPlan。
订单状态未知时必须进入异常同步流程。
订单相关成功、失败、拒单、撤单、超时都必须 AlertEvent。
```

### 18.2 TradeFill 流

```text
交易所成交回报 / REST 查询 / 模拟成交
→ TradeFill
```

规则：

```text
TradeFill 是成交记录。
TradeFill 必须追溯到 ExchangeOrder。
TradeFill 不等同于仓位状态。
部分成交必须记录。
成交记录不得覆盖 DecisionSnapshot。
```

### 18.3 PositionState 流

```text
TradeFill / 交易所仓位同步
→ PositionState
```

规则：

```text
PositionState 是仓位状态记录。
PositionState 必须追溯到 TradeFill 或交易所同步记录。
PositionState 记录仓位事实。
paper trading 仓位与 real trading 仓位必须隔离。
仓位不一致时必须告警，并阻断后续真实交易。
```

---

## 19. Alert / Hermes 通知流

Alert / Hermes 通知属于异步辅助投递链路，但 AlertEvent 本身是主链路事实记录。

触发来源：

```text
系统分析结果
数据质量失败
Account Sync 失败 / 过期
PriceSnapshot 过期 / 缺失
OrderPlan blocked / no_order_required / CandidateOrderIntent 生成
RiskCheck ALLOW / DENY / BLOCKED / FAILED
fallback_reduce_only 被选中
ApprovedOrderIntent 生成
ExecutionPreparation BLOCKED / PASSED
订单提交成功 / 失败
订单成交 / 部分成交
仓位变化
系统异常
数据异常
交易异常
复盘结果
```

流转：

```text
业务事件
→ AlertEvent
→ notifications service
→ Hermes client
→ HermesNotification
```

规则：

```text
所有订单相关正式事件，不管成功还是失败，都必须写 AlertEvent。
Hermes 不得参与交易决策。
Hermes 不得生成 CandidateOrderIntent / ApprovedOrderIntent。
Hermes 不得触发真实下单。
通知失败不得改变交易事实。
通知必须记录 trace_id 和 related object。
```

---

## 20. 复盘流

复盘属于异步辅助链路，但复盘所需证据必须在主链路中持续记录。

```text
trace_id / DecisionSnapshot ID / TradeLifecycle ID
→ 查询证据链
→ ReviewRecord
```

复盘聚合对象包括但不限于：

```text
MarketSnapshot
FeatureSet
FeatureValue
AtomicSignal
DomainSignal
MarketRegime
StrategyRoute
StrategyParamSnapshot
StrategySignal
StrategySignalQualityResult
DecisionSnapshot
BinanceSyncRun
BinanceAccountSnapshot
BinanceBalanceSnapshot
BinancePositionSnapshot
BinanceSymbolRuleSnapshot
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
ExecutionPreparationResult
ExecutionResult
ExchangeOrder
TradeFill
PositionState
AlertEvent
HermesNotification
```

规则：

```text
复盘模块负责查询并组装证据链。
不要求调用方一次性传入所有对象。
复盘失败不得改变交易事实。
复盘结论不得自动修改生产策略。
```

---

## 21. 大模型复盘辅助流

大模型复盘辅助只能在事后触发。

```text
ReviewRecord / 审计摘要
→ model_review service
→ LLM client
→ ModelReviewRecord
```

规则：

```text
大模型不得进入实时交易链路。
大模型不得影响 RiskCheck。
大模型不得生成或修改 CandidateOrderIntent / ApprovedOrderIntent。
大模型不得自动修改生产策略。
模型调用失败只影响解释，不影响交易事实。
```

---

## 22. 异常与人工恢复流

### 22.1 基础设施级恢复

示例：

```text
网络超时
WebSocket 断线
REST 请求临时失败
Celery task 临时失败
订单状态查询失败
PriceSnapshot 刷新失败
```

流转：

```text
异常事件
→ 有限重试
→ 成功：记录恢复结果
→ 失败：告警 / 人工处理
```

规则：

```text
可以自动重试。
必须有最大重试次数。
必须记录错误码、错误消息和 trace_id。
不得无限重试。
不得因为重试绕过 RiskCheck / ExecutionPreparation。
```

### 22.2 业务级人工恢复

示例：

```text
仓位不一致
订单状态长期未知
交易所实际杠杆与快照不一致
重复下单风险
风控熔断解除
失败 ApprovedOrderIntent 是否可重放
```

流转：

```text
业务异常
→ AlertEvent
→ 人工确认
→ RecoveryRecord
→ 必要时重新进入受控链路
```

规则：

```text
必须人工确认。
不得自动绕过 RiskCheck。
不得自动修改杠杆。
不得自动恢复真实交易。
不得自动修改生产策略。
```

---

## 23. 回测数据流

回测应尽量复用实盘核心对象流。

```text
历史 KlineRecord
→ DataQualityResult
→ MarketSnapshot
→ FeatureLayer
→ AtomicSignal
→ DomainSignal
→ MarketRegime
→ StrategyRoute
→ StrategyParamSnapshot
→ StrategySignal
→ StrategySignalQualityResult
→ DecisionSnapshot
→ 模拟 BinanceSyncRun / Account Snapshot
→ 模拟 PriceSnapshot
→ 等价 OrderPlan
→ 等价 CandidateOrderIntent
→ 等价 RiskCheck
→ 模拟 ApprovedOrderIntent
→ 模拟 ExecutionPreparation
→ 回测撮合 / 模拟 Execution
→ 模拟 ExecutionResult / ExchangeOrder / TradeFill / PositionState
→ ReviewRecord
```

回测可以替换：

```text
数据来源
账户状态模拟器
价格快照来源
撮合器
手续费模型
滑点模型
成交约束模型
资金费率模型
```

回测不得替换：

```text
K 线收盘后生成信号规则
FeatureLayer 定义
AtomicSignal 定义
StrategySignal 定义
DecisionSnapshot 追溯关系
OrderPlan 的目标仓位转换语义
RiskCheck 等价逻辑
ExecutionPreparation 等价价格保护逻辑
```

回测必须避免：

```text
look-ahead bias（前视偏差 / 未来函数）
survivorship bias（幸存者偏差）
overfitting（过拟合）
当前 K 线收盘价完美成交假设
实盘不可获得数据
绕过风控
绕过交易规则
```

---

## 24. trace_id、trigger_source 与幂等流转

### 24.1 trace_id

trace_id 必须从入口生成或传入，并贯穿：

```text
TaskRunRecord
MarketData
DataQualityResult
BackfillResult
MarketSnapshot
FeatureSet
AtomicSignal
DomainSignal
StrategySignal
StrategySignalQualityResult
DecisionSnapshot
BinanceSyncRun
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
ExecutionPreparationResult
ExecutionResult
ExchangeOrder
TradeFill
PositionState
AlertEvent
HermesNotification
ReviewRecord
ModelReviewRecord
```

trace_id 负责追踪，不负责业务去重。

### 24.2 trigger_source

trigger_source 必须记录运行来源。

示例：

```text
cli
celery_beat
celery_worker
admin
system
retry
manual_recovery
```

规则：

```text
下游链路必须继承或显式传递上游 trigger_source。
Celery worker 不得覆盖原始 trigger_source。
如需记录执行者，应使用 executor_source。
```

### 24.3 业务幂等键

推荐幂等键：

```text
分析任务：(symbol, timeframe, analysis_close_time_utc, strategy_id, execution_mode)
DecisionSnapshot：(strategy_signal_id, strategy_signal_quality_result_id, decision_policy_hash, execution_mode)
BinanceSyncRun：(exchange, market_type, account_domain, started_at_utc, trigger_source, trace_id)
PriceSnapshot：(symbol, market_type, account_domain, source, source_object_id, mark_price, as_of_utc)
OrderPlan：(decision_snapshot_id, binance_sync_run_id, price_snapshot_id, target_position_ratio, risk_config_hash)
CandidateOrderIntent：(order_plan_id, intent_type, side, requested_size, reduceOnly, components_hash)
RiskCheckResult：(candidate_order_intent_id, order_plan_id, binance_sync_run_id, price_snapshot_id, rule_set_hash, risk_config_hash)
ApprovedOrderIntent：(risk_check_result_id, selected_candidate_order_intent_id)
交易所提交：(approved_order_intent_id, execution_mode, client_order_id)
人工恢复：(manual_request_id 或 confirm_token)
```

规则：

```text
重复运行不得重复下单。
同一 OrderPlan 不得重复生成多个等价 active CandidateOrderIntent。
同一 CandidateOrderIntent 在相同风控输入下不得重复生成多个等价 RiskCheckResult。
同一 ApprovedOrderIntent 不得重复提交为多个有效交易所订单。
幂等冲突必须记录并可复盘。
```

---

## 25. 主链路阻断规则总表

| 环节 | 失败对象 | 是否阻断下游 | 后续处理 |
|---|---|---|---|
| 数据采集 | MarketData 缺失 | 是 | 记录失败，等待重试或人工处理 |
| 数据质量 | DataQualityResult FAIL | 是 | 阻断当前分析流程；可回补问题创建 BackfillRequest |
| 数据回补 | BackfillResult FAIL | 是 | 告警或人工处理 |
| MarketSnapshot | 构建失败 | 是 | 阻断 FeatureLayer |
| FeatureLayer | FeatureQualityCheck FAIL | 部分或全部 | 阻断对应 AtomicSignal |
| AtomicSignal | AtomicSignalQualityCheck FAIL | 部分或全部 | 阻断或降级 DomainSignal |
| DomainSignal | DomainQualityCheck FAIL | 部分或全部 | 输出不确定 / 不可交易 |
| MarketRegime | UNKNOWN / NOT_TRADABLE | 可能 | 可生成不可交易 StrategySignal / DecisionSnapshot |
| StrategyRoute | NO_ROUTE | 是 | 生成不可交易 StrategySignal 或阻断 |
| StrategyParamSnapshot | 参数缺失 | 是 | 阻断 StrategySignal |
| StrategySignal | 生成失败 | 是 | 不生成可用 DecisionSnapshot |
| StrategySignalQualityResult | FAIL / BLOCKED | 是 | 不生成可用 DecisionSnapshot |
| DecisionSnapshot | 生成失败 / 不可用 / 过期 | 是 | 不进入 OrderPlan |
| BinanceSyncRun | 不存在 / failed / 过期 / 快照不完整 | 是 | 阻断 PriceSnapshot / OrderPlan / RiskCheck，写 AlertEvent |
| PriceSnapshot | 缺失 / 过期 / hash 失败 | 是 | 阻断 OrderPlan / RiskCheck / ExecutionPreparation，写 AlertEvent |
| OrderPlan | blocked / failed | 是 | 不生成 CandidateOrderIntent，写 AlertEvent |
| CandidateOrderIntent | 无效 / 过期 / hash 失败 | 是 | 不进入 RiskCheck，写 AlertEvent |
| RiskCheck | DENY / BLOCKED / FAILED | 是 | 不生成 ApprovedOrderIntent，写 AlertEvent |
| ApprovedOrderIntent | 生成失败 / 过期 | 是 | 不进入 ExecutionPreparation，写 AlertEvent |
| ExecutionPreparation | price guard FAIL / BLOCKED | 是 | 不提交真实订单，写 AlertEvent |
| Execution | 执行失败 / 状态未知 | 视情况 | 记录状态，必要时异常同步 / 人工处理，写 AlertEvent |
| ExchangeOrder | 状态未知 | 是 | 异常同步 / 人工恢复 |
| PositionState | 仓位不一致 | 是 | 阻断后续真实交易并告警 |
| Hermes | 通知失败 | 否 | 记录失败，不改变交易事实 |
| ReviewRecord | 复盘失败 | 否 | 记录失败，不改变交易事实 |
| ModelReviewRecord | 模型失败 | 否 | 只影响解释，不影响交易事实 |

---

## 26. 与其他文档的关系

本文档只描述数据和对象如何流转。

关系如下：

```text
docs/architecture/00_project_overview.md
= 系统模块总览

docs/architecture/01_module_map.md
= 模块职责、输入输出、允许调用、禁止调用、失败处理

docs/architecture/system_architecture.md
= 工程分层、存储、调度、gateway、同步异步、模式隔离、trace、幂等和恢复架构

docs/architecture/data_flow.md
= 一次运行中的数据流、对象流、条件分支和阻断规则

docs/requirements/decision_snapshot.md
= 006A DecisionSnapshot 正式合同

docs/requirements/price_snapshot.md
= PriceSnapshot 价格事实层

docs/requirements/order_plan.md
= 007A OrderPlan / CandidateOrderIntent

docs/requirements/binance_account_sync.md
= 006B Binance Account Sync 账户事实层

docs/requirements/risk_check.md
= 007B RiskCheck
```

如果本文档与 `docs/rules/project_invariants.md` 冲突，以 `project_invariants.md` 为准。

如果本文档与 `docs/architecture/01_module_map.md` 在模块边界上冲突，以 `01_module_map.md` 为准。

如果本文档与 `docs/architecture/system_architecture.md` 在工程实现边界上冲突，以 `system_architecture.md` 为准。

---

## 27. 总结

本文档的核心价值是把系统从“模块列表”转化为“对象流转链路”。

它明确：

```text
数据回补是条件补偿链路，不是正常主链路必经步骤。
DecisionSnapshot 只表达目标仓位，不直接生成订单动作。
PriceSnapshot 是长期价格事实层，WebSocket 只是未来数据来源之一。
OrderPlan 才负责把目标仓位转换为 CandidateOrderIntent。
RiskCheck 只检查 CandidateOrderIntent，不直接消费 DecisionSnapshot 的动作字段。
ApprovedOrderIntent 是风控通过的订单意图，不代表已经下单或成交。
ExecutionPreparation 是真实下单前的最终价格保护层。
Hermes、复盘、大模型是异步辅助链路，不得反向触发实时交易。
trace_id 用于追踪，业务幂等键用于防重复。
回测和实盘必须尽量复用同一套核心对象流。
失败必须明确阻断、降级、告警或进入人工恢复。
所有订单相关正式事件，不管成功还是失败，都必须 AlertEvent。
```

任何实现都必须保证对象流转清晰、阻断规则明确、交易链路受控、状态可审计、复盘可解释。
