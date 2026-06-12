# data_quality.md

# 数据质量模块需求

## 1. 文档目的

本文档定义中低频趋势跟踪自动交易系统的数据质量模块需求。

数据质量模块是数据链路的前置哨兵，负责检查采集后的 K 线数据是否可以进入后续分析链路。

本文档只定义数据质量检查需求，不定义：

```text
具体数据库字段
具体 Django model
具体 Celery task 名称
具体质量检查代码实现
具体数据回补实现
具体 Hermes 发送实现
具体风控逻辑
具体策略逻辑
具体交易执行逻辑
```

数据质量模块不采集数据，不修复数据，不修改 K 线主表，不生成 MarketSnapshot。

---

## 2. 模块目标

数据质量模块的目标是：

```text
检查 K 线数据是否可信
发现任何 K 线问题立即阻断当前分析流程
记录数据质量结果
记录数据质量问题
写入告警事件
在可明确回补的场景下创建 BackfillRequest
防止坏数据进入 MarketSnapshot
防止坏数据进入策略链路
保证后续特征、信号、策略、回测和复盘建立在可信数据上
```

第一阶段采用严格阻断策略：

```text
只要发现 K 线存在任何质量问题，
无论问题大小，
当前分析流程立即阻断。
```

但阻断分析流程不等于系统停止处理问题。

如果质量问题属于可回补类型，DataQuality 只创建 BackfillRequest；后续由 data_backfill 扫描 / 领取 pending BackfillRequest，或由后续统一编排层触发处理。

---

## 3. 模块边界

### 3.1 数据质量模块负责

```text
读取待检查 K 线数据
检查 K 线时间是否正确
检查 K 线是否已收盘
检查 K 线是否连续
检查 K 线是否缺失
检查 K 线是否重复
检查 OHLCV 是否合理
检查数据来源是否合法
检查周期边界是否正确
检查同一唯一业务键是否存在冲突
写入 DataQualityResult
写入 DataQualityIssue
FAIL 或系统异常时必要写入 AlertEvent
在可明确回补的场景下创建 BackfillRequest
返回 PASS / FAIL 结论
对 FAIL 结果阻断当前分析流程
```

### 3.2 数据质量模块不负责

```text
采集 K 线
直接回补 K 线
修复 K 线
覆盖 K 线
删除 K 线
修改 K 线 OHLCV
生成 MarketSnapshot
计算 FeatureValue
生成 AtomicSignal
生成 StrategySignal
生成 DecisionSnapshot
生成 CandidateOrderIntent / ApprovedOrderIntent
调用 Execution
调用 RiskCheck
通知风控系统
同步调用 Hermes Webhook
调用大模型
```

DataQuality 写入边界：

```text
DataQuality 只写 DataQualityResult / DataQualityIssue。
FAIL、阻断、系统异常或需要通知的异常事件时，DataQuality 写入 AlertEvent。
需要回补时，DataQuality 创建 BackfillRequest。
BackfillRequest 属于 apps.data_backfill。
DataQuality 不直接执行回补。
DataQuality 不调用 BackfillService。
DataQuality 不启动 data_backfill 调度或回补执行逻辑。
BackfillRun 完成后，必须标记 / 要求 DataQuality 复检；实际复检由后续统一编排层、recovery scan 或人工命令重新执行。
只有新的 DataQualityResult = PASS 且 allows_downstream=True，才允许继续 MarketSnapshot。
回补完成不能直接放行 MarketSnapshot。
```

说明：

```text
data_quality 可以通过创建 BackfillRequest 发起回补请求，但不能自己执行回补逻辑。
实际回补必须由 data_backfill 模块完成。
```

---

## 4. 第一阶段范围

### 4.1 P0 必须实现

```text
4h K 线质量检查
1d K 线质量检查
UTC 时间检查
已收盘检查
周期边界检查
连续性检查
缺失检查
重复检查
OHLC 合法性检查
成交量合法性检查
数据来源检查
数据冲突检查
DataQualityResult 记录
DataQualityIssue 记录
AlertEvent 仅用于 FAIL、阻断、系统异常或需要通知的异常事件
check_scope / run_mode 记录
BackfillRequest 幂等创建或复用
FAIL 时阻断 MarketSnapshot
可回补问题创建 BackfillRequest
```

### 4.2 P1 可预留

```text
质量检查批次统计
最近 K 线及时性检查
多次连续失败聚合告警
数据质量复检入口
历史区间质量扫描
数据质量报表
回补请求状态追踪
```

### 4.3 P2 后续再做

```text
多品种质量检查
多交易所质量检查
多数据源交叉验证
复杂异常行情检测
质量评分体系
可视化数据质量看板
```

---

## 5. 严格阻断策略

第一阶段采用严格阻断策略。

规则：

```text
只要 data_quality 检查到 K 线存在任何问题，
无论严重程度大小，
DataQualityResult 必须为 FAIL，
并立即阻断当前分析流程。
```

阻断范围包括：

```text
MarketSnapshot
FeatureLayer
AtomicSignal
DomainSignal
MarketRegime
StrategyRoute
StrategySignal
StrategySignalQuality
DecisionSnapshot
Binance Account Sync
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
```

说明：

```text
data_quality 不调用 RiskCheck。
data_quality 不通知风控系统。
data_quality 不写跨流程风险状态。
data_quality 直接在数据链路前端阻断当前分析流程。
```

最终处理流程：

```text
data_quality 发现 K 线问题
→ 写 DataQualityResult = FAIL
→ 写 DataQualityIssue
→ 写 AlertEvent
→ 判断是否属于可回补问题
    ├─ 是 → 创建 BackfillRequest
    └─ 否 → 等待人工处理
→ 阻断当前分析流程
→ 不生成 MarketSnapshot
→ notifications / Hermes 服务异步消费 AlertEvent
→ 发送 HermesNotification
→ 记录发送结果
```

---

## 6. 回补触发策略

### 6.1 data_quality 可以创建的 BackfillRequest

data_quality 可以在发现明确可回补问题时，创建 BackfillRequest。

可创建 BackfillRequest 的问题包括：

```text
MISSING_KLINE
NON_CONTINUOUS_KLINE
LATEST_KLINE_DELAYED
EMPTY_KLINE_SET
```

触发方式可以是：

```text
创建 BackfillRequest
```

触发后，当前分析流程仍然必须阻断。

### 6.2 data_quality 不应自动修复的问题

以下问题不应由 data_quality 自动修复，也不应静默自动处理：

```text
DUPLICATE_KLINE
DATA_CONFLICT
INVALID_OHLC
INVALID_VOLUME
INVALID_DATA_SOURCE
INVALID_TIME_BOUNDARY
UNCLOSED_KLINE
DATABASE_INCONSISTENCY
```

这些问题必须：

```text
写 DataQualityIssue
写 AlertEvent
阻断当前流程
交由 data_backfill conflict_recheck 或人工处理流程处理
```

说明：

```text
DATA_CONFLICT 可以触发 conflict_recheck 类型的回补复核，
但不得自动覆盖已有 K 线。
```

### 6.3 回补触发不得绕过 data_backfill

data_quality 不得直接：

```text
请求 Binance REST
写入 KlineRecord
修改 K 线主表
覆盖冲突数据
删除重复数据
```

所有实际回补动作必须由 data_backfill 执行。

---

## 7. 输入对象

数据质量模块输入至少包括：

```text
exchange
market_type
symbol
timeframe
check_start_time_utc
check_end_time_utc
KlineRecord 集合
data_source
collection_mode
trigger_source
trace_id
```

### 7.1 exchange

第一阶段：

```text
binance
```

### 7.2 market_type

第一阶段：

```text
usds_m_futures
```

### 7.3 symbol

第一阶段：

```text
BTCUSDT
```

### 7.4 timeframe

第一阶段允许：

```text
4h
1d
```

### 7.5 check_start_time_utc / check_end_time_utc

用于定义本次质量检查覆盖的 UTC 时间范围。

要求：

```text
必须使用 UTC。
不得使用本地时间。
不得使用 PRC 时间。
不得使用运行机器时区。
```

---

### 7.6 check_scope / run_mode

数据质量检查必须记录本次检查的运行场景。

建议字段或等价语义：

```text
check_scope
run_mode
```

P0 允许场景：

```text
analysis_cycle
daily
manual
```

含义：

```text
analysis_cycle = 正常 4h 分析链路前置质量检查。
daily = 日常巡检或定期质量复核。
manual = 人工指定范围质量检查。
```

规则：

```text
check_scope 不得影响质量判断标准。
无论 check_scope 是什么，只要发现 issue，DataQualityResult 都必须为 FAIL。
check_scope 只影响审计、AlertEvent 类型和调度语义。
```

## 8. 输出对象

数据质量模块输出包括：

```text
DataQualityResult
DataQualityIssue
AlertEvent
BackfillRequest（可回补问题时）
```

### 8.1 DataQualityResult

DataQualityResult 表示一次质量检查的总体结论。

第一阶段只需要两类结论：

```text
PASS
FAIL
```

含义：

```text
PASS = 本次检查范围内未发现 K 线质量问题，允许进入 MarketSnapshot。
FAIL = 本次检查范围内发现至少一个 K 线质量问题，必须阻断当前流程。
```

说明：

```text
第一阶段不使用 WARN 放行机制。
只要发现问题，无论大小，都视为 FAIL。
```

DataQualityResult 至少应表达：

```text
trace_id
trigger_source
exchange
market_type
symbol
timeframe
check_start_time_utc
check_end_time_utc
status
issue_count
created_at_utc
```

字段细节由后续数据库设计确定。

### 8.2 DataQualityIssue

DataQualityIssue 表示具体质量问题。

可能的问题类型包括：

```text
MISSING_KLINE
DUPLICATE_KLINE
NON_CONTINUOUS_KLINE
UNCLOSED_KLINE
INVALID_TIME_BOUNDARY
INVALID_OPEN_CLOSE_TIME
INVALID_OHLC
INVALID_VOLUME
INVALID_DATA_SOURCE
DATA_CONFLICT
EMPTY_KLINE_SET
LATEST_KLINE_DELAYED
UNEXPECTED_SYMBOL
UNEXPECTED_TIMEFRAME
DATABASE_INCONSISTENCY
```

DataQualityIssue 至少应表达：

```text
trace_id
related_quality_result
exchange
market_type
symbol
timeframe
issue_type
affected_start_time_utc
affected_end_time_utc
affected_open_time_utc
severity
message
created_at_utc
```

说明：

```text
severity 可用于通知展示和后续统计。
但第一阶段无论 severity 高低，只要存在 issue，DataQualityResult 都是 FAIL。
```

### 8.3 AlertEvent

AlertEvent 表示需要通知系统处理的告警事件。

DataQuality PASS 不写 AlertEvent。

AlertEvent 仅用于 FAIL、阻断、系统异常或需要通知的异常事件。

AlertEvent 由 notifications / Hermes 服务异步消费。

数据质量模块不直接发送 Hermes。

### 8.4 BackfillRequest

当 data_quality 发现明确可回补问题时，可以创建 BackfillRequest。

BackfillRequest 至少应表达：

```text
trace_id
source_quality_result_id
source_quality_issue_id
exchange
market_type
symbol
timeframe
backfill_mode
requested_start_time_utc
requested_end_time_utc
trigger_source
reason
status
created_at_utc
```

说明：

```text
BackfillRequest 只是回补请求记录。
实际回补动作必须由 data_backfill 执行。
```

---

### 8.5 BackfillRequest 幂等规则

当 data_quality 发现可明确回补的问题时，可以创建 BackfillRequest。

BackfillRequest 的业务幂等键至少应包含：

```text
exchange
market_type
symbol
timeframe
reason_code
requested_start_time_utc
requested_end_time_utc
missing_open_times
```

规则：

```text
同一质量问题重复检查时，应复用已有 pending / running / retryable failed BackfillRequest。
不得因为重复运行 data_quality 而无限创建等价 BackfillRequest。
missing_open_times 必须作为幂等键的一部分。
如果 missing_open_times 不同，应视为不同回补请求。
BackfillRequest 只是请求记录，实际回补动作仍由 data_backfill 执行。
```

## 9. 检查内容

### 9.1 时间字段检查

必须检查：

```text
open_time_utc 是否存在
close_time_utc 是否存在
open_time_utc 是否早于 close_time_utc
open_time_utc 是否为 UTC
close_time_utc 是否为 UTC
K 线是否已收盘
```

禁止：

```text
使用本地时间判断
使用 PRC 时间判断
使用运行机器时区判断
根据请求 IP 推断时间
```

### 9.2 周期边界检查

4h K 线 open_time_utc 必须落在 UTC 4 小时边界：

```text
00:00
04:00
08:00
12:00
16:00
20:00
```

1d K 线 open_time_utc 必须落在 UTC 日线边界：

```text
00:00
```

如果 open_time_utc 不在对应周期边界，必须记录：

```text
DataQualityIssue = INVALID_TIME_BOUNDARY
DataQualityResult = FAIL
```

### 9.3 已收盘检查

主 K 线表中只允许已收盘 K 线。

检查规则：

```text
K 线必须满足已收盘条件。
close_time_utc 必须早于当前可信交易所 serverTime。
主 K 线表不得依赖 is_closed 字段区分未收盘数据；未收盘 K 线从源头不得写入主 K 线表。
```

如发现未收盘 K 线进入主 K 线表，必须记录：

```text
DataQualityIssue = UNCLOSED_KLINE
DataQualityResult = FAIL
```

### 9.4 连续性检查

必须按行情时间检查连续性。

4h K 线要求：

```text
下一根 open_time_utc - 当前 open_time_utc = 4 小时
```

1d K 线要求：

```text
下一根 open_time_utc - 当前 open_time_utc = 1 天
```

禁止使用数据库自增 id 判断顺序、连续性或缺口。

如发现不连续，必须记录：

```text
DataQualityIssue = NON_CONTINUOUS_KLINE
DataQualityResult = FAIL
```

并可设置 / 记录：

```text
backfill_mode = gap_backfill
```

### 9.5 缺失检查

如果检查范围内缺少应存在的 K 线，必须记录：

```text
DataQualityIssue = MISSING_KLINE
DataQualityResult = FAIL
```

DataQualityIssue 应记录：

```text
缺失的 open_time_utc
缺失数量
影响起止时间
```

并可设置 / 记录：

```text
backfill_mode = gap_backfill
```

### 9.6 最新 K 线延迟检查

如果理论最新已收盘 K 线应存在，但数据库中不存在，应记录：

```text
DataQualityIssue = LATEST_KLINE_DELAYED
DataQualityResult = FAIL
```

并可设置 / 记录：

```text
backfill_mode = failure_recovery_backfill 或 gap_backfill
```

### 9.7 重复检查

同一唯一业务键不得存在多条有效 K 线。

唯一业务键至少包括：

```text
exchange
market_type
symbol
timeframe
open_time_utc
```

如果同一唯一业务键存在多条有效记录，必须记录：

```text
DataQualityIssue = DUPLICATE_KLINE
DataQualityResult = FAIL
```

说明：

```text
数据库应通过复合唯一索引尽量防止重复。
但 data_quality 仍需检查重复问题，防止历史脏数据、人工导入、旧代码或软删除状态造成多条有效记录。
```

重复问题不应由 data_quality 自动修复。

### 9.8 OHLC 合法性检查

必须检查：

```text
open_price 是否存在
high_price 是否存在
low_price 是否存在
close_price 是否存在
open_price >= 0
high_price >= 0
low_price >= 0
close_price >= 0
high_price >= low_price
high_price >= open_price
high_price >= close_price
low_price <= open_price
low_price <= close_price
```

如发现异常，必须记录：

```text
DataQualityIssue = INVALID_OHLC
DataQualityResult = FAIL
```

### 9.9 成交量合法性检查

必须检查：

```text
volume 是否存在
quote_volume 是否存在
volume >= 0
quote_volume >= 0
trade_count >= 0（如字段存在）
```

对 BTCUSDT U 本位 4h / 1d，volume 或 quote_volume 为 0 时，也视为质量问题。

如发现异常，必须记录：

```text
DataQualityIssue = INVALID_VOLUME
DataQualityResult = FAIL
```

### 9.10 数据来源检查

第一阶段 4h / 1d 主 K 线合法数据来源只能是：

```text
binance_rest
```

禁止以下来源进入主 K 线表：

```text
binance_websocket
manual_repair
system_repair
human_edit
manual_input
```

如发现非法来源，必须记录：

```text
DataQualityIssue = INVALID_DATA_SOURCE
DataQualityResult = FAIL
```

### 9.11 数据冲突检查

数据冲突指同一唯一业务键的 K 线存在数值不一致问题。

例如同一根 K 线的以下字段不一致：

```text
open_price
high_price
low_price
close_price
volume
quote_volume
trade_count
```

如发现冲突，必须记录：

```text
DataQualityIssue = DATA_CONFLICT
DataQualityResult = FAIL
```

说明：

```text
data_quality 不负责判断应该保留哪一条。
data_quality 不覆盖旧数据。
data_quality 不删除冲突数据。
data_quality 只记录冲突并阻断当前流程。
```

DATA_CONFLICT 可触发：

```text
backfill_mode = conflict_recheck
```

但 conflict_recheck 不得自动覆盖已有 K 线。

---

## 10. K 线写入入口收窄

K 线主表的写入入口必须收窄。

允许写入 K 线主表的模块只有：

```text
DataCollectionService
DataBackfillService
```

禁止以下模块写入、更新或删除 K 线主表：

```text
data_quality
MarketSnapshot
FeatureLayer
AtomicSignal
DomainSignal
MarketRegime
StrategySignal
DecisionSnapshot
Binance Account Sync
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
Hermes
大模型
人工后台直接改表
```

data_quality 只能写：

```text
DataQualityResult
DataQualityIssue
AlertEvent
BackfillRequest
```

data_quality 禁止写：

```text
KlineRecord
open_price
high_price
low_price
close_price
volume
quote_volume
trade_count
```

---

## 11. 成功流程

### 11.1 PASS 流程

```text
读取待检查 KlineRecord
→ 执行时间检查
→ 执行周期边界检查
→ 执行已收盘检查
→ 执行连续性检查
→ 执行缺失检查
→ 执行重复检查
→ 执行 OHLCV 检查
→ 执行数据来源检查
→ 执行数据冲突检查
→ 未发现任何问题
→ 写 DataQualityResult = PASS
→ 返回 PASS
→ 允许进入 MarketSnapshot
```

规则：

```text
只有 DataQualityResult = PASS 时，才允许生成 MarketSnapshot。
```

---

## 12. 失败流程

### 12.1 FAIL 流程

```text
读取待检查 KlineRecord
→ 执行质量检查
→ 发现任意问题
→ 写 DataQualityResult = FAIL
→ 写 DataQualityIssue
→ 写 AlertEvent
→ 如果问题可回补，创建 BackfillRequest
→ 返回 FAIL
→ 阻断当前分析流程
```

阻断后不得生成：

```text
MarketSnapshot
FeatureSet
FeatureValue
AtomicSignal
DomainSignal
MarketRegime
StrategyRoute
StrategySignal
StrategySignalQuality
DecisionSnapshot
Binance Account Sync
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
ExecutionRecord
ExchangeOrder
TradeFill
PositionSnapshot
```

### 12.2 AlertEvent 流程

```text
data_quality 发现问题
→ 写 AlertEvent
→ notifications / Hermes 服务异步消费 AlertEvent
→ 发送 HermesNotification
→ 记录发送结果
```

规则：

```text
data_quality 不直接调用 Hermes。
Hermes 发送失败不得改变 DataQualityResult。
Hermes 发送失败不得让 data_quality 重试质量检查。
AlertEvent 写入失败时，本次质量检查必须视为失败，并记录系统异常。
```

### 12.3 AlertEvent 与 check_scope

DataQualityResult 本身就是 PASS 的审计记录。

DataQuality PASS 只写 DataQualityResult = PASS 且 allows_downstream=True，不写 AlertEvent，不创建 BackfillRequest。

data_quality 的 AlertEvent 行为必须按 check_scope 明确记录。

P0 建议事件语义：

```text
analysis_cycle + FAIL：写 data_quality_analysis_failed。
daily + FAIL：写 data_quality_daily_fail。
manual + FAIL：写 data_quality_manual_fail。
```

规则：

```text
FAIL 必须写 AlertEvent。
PASS 不写 AlertEvent。
AlertEvent 仅用于 FAIL、阻断、系统异常或需要通知的异常事件。
AlertEvent 写入失败时，本次质量检查必须视为系统异常，不得静默通过。
```

### 12.4 Backfill 触发流程

```text
data_quality 发现可回补问题
→ 写 DataQualityIssue
→ 写 AlertEvent
→ 创建 BackfillRequest
→ 当前分析流程阻断
→ data_backfill 执行回补
→ BackfillRun 完成后，标记 / 要求 DataQuality 复检
→ 后续统一编排层、recovery scan 或人工命令重新执行 DataQuality
→ 只有新的 DataQualityResult = PASS 且 allows_downstream=True，才允许继续 MarketSnapshot
```

规则：

```text
data_quality 创建 BackfillRequest 不等于 data_quality 执行回补。
回补成功不等于数据可信。
回补后必须重新 data_quality。
```

---

## 13. 与数据采集模块关系

数据采集模块负责：

```text
采集 4h / 1d 已收盘 K 线
lookback window 自动补齐短期漏采
幂等写入 K 线主表
记录采集事件
```

数据质量模块负责：

```text
检查采集结果是否可进入下游
发现任何问题即阻断
写 DataQualityResult
写 DataQualityIssue
写 AlertEvent
可回补问题创建 BackfillRequest
```

重要边界：

```text
采集成功不等于数据可信。
只有质量检查 PASS，数据才可进入 MarketSnapshot。
```

---

## 14. 与数据回补模块关系

数据回补模块负责：

```text
缺口回补
人工指定区间回补
较大范围补偿
冲突复核
```

数据质量模块负责：

```text
发现缺口
发现冲突
记录问题
阻断流程
创建 BackfillRequest
```

data_quality 不直接执行回补。

当数据采集或数据回补补齐数据后，必须重新运行 data_quality。

只有复检 PASS，后续流程才允许继续。

---

## 15. 与 MarketSnapshot 的关系

MarketSnapshot 只能基于 DataQualityResult = PASS 的 K 线生成。

如果 DataQualityResult = FAIL：

```text
不得生成 MarketSnapshot。
不得使用部分可用 K 线绕过失败结果生成 MarketSnapshot。
不得由 MarketSnapshot 模块自行忽略数据质量问题。
```

MarketSnapshot 必须可追溯到对应 DataQualityResult。

---

## 16. 与 notifications / Hermes 的关系

data_quality 只写 AlertEvent。

notifications / Hermes 服务负责：

```text
消费 AlertEvent
发送 HermesNotification
记录发送结果
处理发送失败
```

data_quality 不负责：

```text
同步调用 Hermes Webhook
等待 Hermes 发送成功
处理 Hermes 重试
根据 Hermes 结果改变质量检查状态
```

设计目的：

```text
数据质量检查不被外部通知服务阻塞。
数据质量问题即使通知失败也不会丢失。
```

---

## 17. 与风控模块关系

第一阶段 data_quality 不通知风控系统。

data_quality 不调用 RiskCheck。

data_quality 不写跨流程风险状态。

原因：

```text
数据质量检查发生在 MarketSnapshot 之前。
如果数据质量失败，本轮流程已经被阻断，不会进入策略和交易链路。
RiskCheck 是交易前风控，不承担数据质量前置闸门职责。
```

后续如果引入持仓保护、异常恢复或跨流程风险状态，再由监控与异常恢复模块单独设计。

---

## 18. 禁止事项

数据质量模块禁止：

```text
修改 KlineRecord。
覆盖 K 线 OHLCV。
删除重复 K 线。
自动选择冲突数据中的一条作为正确数据。
直接请求 Binance REST。
直接写入 K 线主表。
直接执行回补逻辑。
生成 MarketSnapshot。
计算 FeatureValue。
生成 AtomicSignal。
生成 StrategySignal。
生成 DecisionSnapshot。
读取账户状态。
读取仓位状态。
调用 RiskCheck。
生成 CandidateOrderIntent / ApprovedOrderIntent。
调用 Execution。
调用大模型。
同步调用 Hermes Webhook。
写跨流程风险状态。
通知风控系统。
在发现问题后继续放行当前分析流程。
```

---

## 19. 验收标准

第一阶段完成后，数据质量模块必须满足：

```text
可以检查 BTCUSDT U 本位 4h K 线质量。
可以检查 BTCUSDT U 本位 1d K 线质量。
所有时间检查使用 UTC。
可以识别未收盘 K 线。
可以识别周期边界错误。
可以识别 K 线不连续。
可以识别 K 线缺失。
可以识别最新 K 线延迟。
可以识别重复 K 线。
可以识别 OHLC 不合法。
可以识别 volume / quote_volume 不合法。
可以识别非法数据来源。
可以识别同一唯一业务键的数据冲突。
发现任何问题都会写 DataQualityResult = FAIL。
发现任何问题都会写 DataQualityIssue。
发现任何问题都会写 AlertEvent。
发现任何问题都会阻断 MarketSnapshot。
可回补问题会创建 BackfillRequest。
等价可回补问题会复用 BackfillRequest，不会无限创建重复请求。
DataQualityResult 会记录 check_scope / run_mode。
FAIL、阻断、系统异常或需要通知的异常事件的 AlertEvent 语义明确。
PASS 时才允许进入 MarketSnapshot。
DataQuality PASS 不写 AlertEvent。
data_quality 不修改 K 线主表。
data_quality 不直接请求 Binance。
data_quality 不直接执行回补。
data_quality 不调用 RiskCheck。
data_quality 不写跨流程风险状态。
data_quality 不同步调用 Hermes。
AlertEvent 可被 notifications / Hermes 服务异步消费。
```

---

## 20. 后续待拆分文档

以下内容不在本文档展开：

```text
data_backfill.md
= 缺口回补、指定区间回补、冲突复核和回补后的复检流程。

market_snapshot.md
= 如何基于 PASS 的 DataQualityResult 生成 MarketSnapshot。

notifications.md / monitoring_recovery.md 为后续补充文档，当前阶段不作为 001/002 阶段阻断依赖。

notifications.md
= 如何消费 AlertEvent、发送 HermesNotification、记录发送结果。

monitoring_recovery.md
= 持仓保护、异常恢复、人工处理和跨流程风险状态。
```

---

## 21. 总结

数据质量模块是数据链路哨兵。

第一阶段核心口径：

```text
data_quality 只检查、记录、告警、创建 BackfillRequest 和阻断当前分析流程。
data_quality 不修数据、不直接回补、不交易、不通知风控。
只要发现任何 K 线问题，立即 DataQualityResult = FAIL。
FAIL 时必须写 DataQualityIssue 和 AlertEvent。
可回补问题可以创建 BackfillRequest。
FAIL 时必须阻断 MarketSnapshot、策略和交易链路。
K 线主表只能由 DataCollectionService 和 DataBackfillService 写入。
Hermes 由 notifications 服务异步消费 AlertEvent 后发送。
```
