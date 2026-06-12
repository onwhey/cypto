# data_collection.md

# 数据采集模块需求

## 1. 文档目的

本文档定义中低频趋势跟踪自动交易系统的数据采集模块需求。

数据采集模块是系统的数据入口，负责从可信数据源获取市场原始数据，并将其以可追溯、可幂等、可审计的方式写入系统存储，为后续数据质量检查、MarketSnapshot、FeatureLayer、回测、风控和实盘交易提供基础数据。

本文档只定义数据采集模块需求，不定义：

```text
具体数据库字段
具体 Django model
具体 Celery task 名称
具体 Binance endpoint 封装方式
具体数据质量算法
具体数据回补算法
具体 Hermes 发送实现
具体风控逻辑
具体策略逻辑
```

数据是否合格，由数据质量模块判断。

缺口回补、指定区间回补和数据冲突处理，由数据回补模块和数据质量模块共同处理。

Hermes 通知发送由 notifications / Hermes 服务处理。002B P0 只写 DataCollectionRun 采集审计摘要；异常或告警写 AlertEvent。逐请求 / 逐事件明细记录如未来需要，作为 P1 DataCollectionEvent 扩展。

---

## 2. 模块目标

数据采集模块的目标是：

```text
可信获取行情数据
只采集标准已收盘 K 线作为主策略数据
统一使用 UTC 时间
幂等写入 MySQL
记录采集任务审计摘要；逐请求 / 逐事件明细记录作为 P1 扩展
为数据质量模块提供输入
避免数据采集失败静默发生
避免采集模块越权进入策略、风控或交易执行
```

第一阶段重点：

```text
交易所：Binance
市场类型：U 本位合约
交易品种：BTCUSDT
K 线周期：4h、1d
采集方式：Binance REST
数据用途：策略分析、回测、MarketSnapshot、复盘
```

说明：

```text
4h K 线 = 主策略周期数据。
1d K 线 = 大周期趋势 / 市场环境辅助数据。
4h 与 1d 采集代码可以复用。
4h 与 1d 存储必须分表，不得混入同一张 K 线业务表。
```

---

## 3. 模块边界

### 3.1 数据采集模块负责

```text
通过 Binance REST 获取 4h / 1d 历史已收盘 K 线。
通过 Binance REST 获取 4h / 1d 最新已收盘 K 线。
增量采集时默认使用 lookback window 多采最近若干根已收盘 K 线。
按唯一业务键幂等写入对应周期 K 线表。
记录采集任务状态。
记录 DataCollectionRun 采集审计摘要。
记录异常事件或告警事件。
记录 trace_id、trigger_source、data_source、collection_mode。
为数据质量模块提供待检查数据范围。
保留 WebSocket 实时行情监测边界。
```

### 3.2 数据采集模块不负责

```text
判断数据是否最终可进入策略链路。
生成 MarketSnapshot。
计算 FeatureValue。
生成 AtomicSignal。
生成 StrategySignal。
生成 DecisionSnapshot。
读取账户状态。
读取仓位状态。
调用 OrderPlan。
生成 CandidateOrderIntent。
生成 ApprovedOrderIntent。
调用 RiskCheck。
调用 ExecutionPreparation。
调用 Execution。
访问交易所下单接口。
调用大模型。
同步等待 Hermes 发送成功。
人工修复 K 线。
手工改写 K 线。
```

---

## 4. 第一阶段范围

### 4.1 P0 必须实现

```text
Binance BTCUSDT U 本位 4h 已收盘 K 线 REST 采集。
Binance BTCUSDT U 本位 1d 已收盘 K 线 REST 采集。
历史区间采集。
最新已收盘 K 线增量采集。
增量采集默认使用 lookback window。
lookback window 内短期漏采自动幂等补齐。
4h / 1d K 线分表存储。
K 线唯一业务键约束。
K 线幂等写入。
采集任务状态记录。
DataCollectionRun 采集审计摘要记录。
受控 retry 策略。
采集异常事件入库。
采集成功后进入或等待数据质量检查。
所有时间统一 UTC。
```

### 4.2 P1 可预留

```text
WebSocket 实时行情监测。
RealtimePriceState。
PriceEvent。
MarketShockEvent。
WebSocketConnectionState。
资金费率采集。
交易规则读取边界。
采集限频控制。
采集任务重试策略。
DataCollectionEvent 逐请求 / 逐事件明细记录。
```

### 4.3 P2 后续再做

```text
多品种采集。
多交易所采集。
订单簿深度完整采集。
逐笔成交采集。
多数据源交叉采集。
WebSocket 事件进入自动风控策略。
```

### 4.4 第一阶段明确不做

```text
1m K 线采集。
WebSocket 拼接 4h / 1d K 线。
WebSocket 写入 4h / 1d 主 K 线表。
WebSocket 作为第一阶段策略信号输入。
WebSocket 直接触发交易。
数据采集模块直接发送 Hermes。
数据采集模块调用大模型。
数据采集模块访问交易所交易接口。
```

---

## 5. REST 与 WebSocket 边界

### 5.1 REST K 线采集

Binance REST 是第一阶段标准 K 线来源。

REST 负责：

```text
历史 4h K 线采集。
历史 1d K 线采集。
最新已收盘 4h K 线采集。
最新已收盘 1d K 线采集。
增量 lookback window 采集。
短期漏采自动补齐。
```

REST K 线用于：

```text
数据质量检查
MarketSnapshot
FeatureLayer
AtomicSignal
StrategySignal
DecisionSnapshot
回测
复盘
```

### 5.2 WebSocket 实时行情监测

WebSocket 也是数据采集的一种方式，但不属于第一阶段主 K 线采集链路。

WebSocket 用于：

```text
实时价格状态
短期价格变化监测
连接状态监测
异常波动事件
闪崩 / 急拉风险事件边界预留
```

WebSocket 可以输出：

```text
RealtimePriceState
PriceEvent
MarketShockEvent
WebSocketConnectionState
```

WebSocket 禁止：

```text
自行拼接 4h K 线。
自行拼接 1d K 线。
写入 4h / 1d 主 K 线表。
作为第一阶段策略主周期数据来源。
直接生成 MarketSnapshot。
直接生成 FeatureValue。
直接生成 AtomicSignal。
直接生成 StrategySignal。
直接生成 DecisionSnapshot。
直接调用 OrderPlan。
直接调用 RiskCheck。
直接调用 ExecutionPreparation。
直接调用 Execution。
直接触发真实交易。
```

WebSocket 的风险用途应由后续监控、异常恢复或风控模块消费，不由数据采集模块直接处理。

---

## 6. 时间规则

系统所有核心业务时间统一使用 UTC。

要求：

```text
K 线 open_time 使用 UTC。
K 线 close_time 使用 UTC。
K 线排序使用 UTC。
K 线连续性判断使用 UTC。
K 线收盘判断使用 UTC。
策略周期判断使用 UTC。
回测判断使用 UTC。
订单追踪使用 UTC。
成交追踪使用 UTC。
仓位追踪使用 UTC。
复盘追踪使用 UTC。
```

禁止：

```text
使用本地时间参与业务判断。
使用 PRC 时间参与业务判断。
使用运行机器时区参与业务判断。
设计 local_time / prc_time 作为核心业务字段。
根据请求 IP 推断时间。
```

Binance K 线请求要求：

```text
请求 K 线时不传 timeZone 参数。
全部使用 Binance 默认 UTC 口径。
K 线 open_time / close_time 使用 Binance 返回的时间戳，并按 UTC 解释。
```

K 线是否已收盘，优先依据：

```text
Binance serverTime
K 线 close_time
```

不得单纯依赖本机当前时间判断 K 线是否已收盘。

---

## 7. K 线周期与存储范围

第一阶段只采集：

```text
4h
1d
```

4h 用途：

```text
主趋势判断
主策略周期
策略信号生成
回测
复盘
```

1d 用途：

```text
大周期趋势判断
市场环境辅助
策略过滤辅助
复盘辅助
```

存储要求：

```text
4h K 线必须存入 4h 专用 K 线表。
1d K 线必须存入 1d 专用 K 线表。
4h 与 1d 不得混入同一张 K 线业务表。
```

说明：

```text
采集代码可以复用。
存储、唯一约束、连续性检查、质量状态应按周期独立处理。
```

第一阶段不采集 1m K 线。

---

## 8. 输入对象

数据采集模块输入至少包括：

```text
exchange
market_type
symbol
timeframe
start_time_utc
end_time_utc
collection_mode
data_source
trigger_source
trace_id
lookback_count
```

### 8.1 exchange

第一阶段：

```text
binance
```

### 8.2 market_type

第一阶段：

```text
usds_m_futures
```

表示 Binance U 本位合约。

### 8.3 symbol

第一阶段：

```text
BTCUSDT
```

### 8.4 timeframe

第一阶段允许：

```text
4h
1d
```

### 8.5 collection_mode

collection_mode 表示采集模式。

建议取值：

```text
historical
latest_closed
incremental
manual_backfill
```

说明：

```text
historical = 初始化或指定历史区间采集。
latest_closed = 获取最新已收盘 K 线。
incremental = 周期性增量采集。
manual_backfill = 人工指定区间回补触发的采集请求。
```

注意：

```text
manual_backfill 仍然通过 Binance REST 获取官方已收盘 K 线。
manual_backfill 不等于人工修改 K 线。
```

### 8.6 data_source

data_source 表示行情数据来源。

第一阶段允许：

```text
binance_rest
binance_websocket
```

其中：

```text
binance_rest = REST K 线数据来源。
binance_websocket = WebSocket 实时行情状态来源，不得写入 4h / 1d 主 K 线表。
```

### 8.7 trigger_source

trigger_source 表示任务触发来源。

建议取值：

```text
cli
management_command
celery_beat
celery_worker
manual_recovery
retry
system
admin
```

规则：

```text
trigger_source 必须显式传入或由明确入口设置。
trigger_source 不得为空。
trigger_source 不得由程序猜测。
Celery worker 执行任务时不得覆盖原始 trigger_source。
```

---

## 9. 输出对象

数据采集模块输出包括：

```text
KlineRecord
TaskRunRecord
AlertEvent
DataSourceRecord
```

### 9.1 KlineRecord

KlineRecord 表示一根标准 K 线。

第一阶段必须表达以下信息：

```text
exchange
market_type
symbol
timeframe
open_time_utc
close_time_utc
open_price
high_price
low_price
close_price
volume
quote_volume
trade_count（如 Binance 返回）
data_source
collection_mode
trace_id
created_at_utc
```

字段命名和精度由后续数据库设计确定，但上述语义必须可保存、可查询、可追溯。

说明：

```text
主 K 线表不得保存未收盘 K 线。
因此不需要以 is_closed 作为业务状态字段来区分已收盘 / 未收盘。
如保存 close_time_utc，只能保存规范化周期结束时间，不得保存 Binance raw closeTime 作为独立业务事实。
```

### 9.2 DataCollectionEvent（P1 扩展，不属于 002B P0）

002B P0 不实现 DataCollectionEvent；如未来需要逐请求 / 逐事件明细记录，可作为 P1 扩展。

未来扩展时可用于记录：

```text
采集开始
采集完成
REST 请求失败
REST 响应异常
数据解析失败
lookback 自动补漏
发现数据冲突
写库成功
写库失败
WebSocket 连接成功
WebSocket 断开
WebSocket 重连
```

### 9.3 TaskRunRecord

TaskRunRecord 表示一次采集任务运行记录。

至少应能记录：

```text
trace_id
trigger_source
collection_mode
exchange
market_type
symbol
timeframe
start_time_utc
end_time_utc
任务状态
错误码
错误消息
开始时间
结束时间
```

### 9.4 DataCollectionRun / TaskRunRecord 审计摘要

DataCollectionRun 或等价 TaskRunRecord 必须能表达一次采集生命周期的审计摘要。

除基础状态字段外，建议记录：

```text
requested_start_time_utc
requested_end_time_utc
binance_server_time_utc
attempt_count
max_attempts
retryable_error_count
fetched_count
closed_count
inserted_count
skipped_existing_count
duplicate_in_response_count
missing_in_lookback_count
conflict_count
filtered_unclosed_count
error_code
error_message
```

说明：

```text
DataCollectionRun 用于审计一次采集任务。
它不替代 KlineRecord。
它不作为 K 线唯一业务事实来源。
trace_id 只用于追踪，不作为采集幂等键。
```

### 9.5 AlertEvent

AlertEvent 表示需要通知系统处理的告警事件。

数据采集模块只负责写入 AlertEvent，不负责同步发送 Hermes。

---

## 10. 唯一业务键与幂等要求

### 10.1 K 线唯一业务键

同一根 K 线的唯一业务键至少包括：

```text
exchange
market_type
symbol
timeframe
open_time_utc
```

或等价 UTC 时间戳字段。

禁止使用数据库自增 id 判断行情顺序、唯一性、连续性或缺口。

### 10.1.1 复合唯一约束

4h K 线表和 1d K 线表都必须建立复合唯一约束，用于防止同一根 K 线被重复写入。

唯一约束字段至少包括：

```text
exchange
market_type
symbol
open_time_utc
```

### 10.2 幂等写入规则

重复采集同一时间范围时：

```text
不得插入重复 K 线。
不得因重复运行产生重复业务记录。
不得静默覆盖已存在 K 线核心数值。
必须记录本次采集任务执行结果。
```

写入规则：

```text
同一唯一键不存在 → 插入。
同一唯一键存在且 OHLCV 一致 → 跳过或更新非核心审计字段。
同一唯一键存在但 OHLCV 不一致 → 记录 DataConflict / DataQualityIssue，不得静默覆盖。
```

### 10.3 业务幂等与 trace_id

trace_id 用于追踪一次运行，不作为唯一业务幂等键。

采集业务幂等键应至少包含：

```text
exchange
market_type
symbol
timeframe
collection_mode
start_time_utc
end_time_utc
```

对于增量任务，可结合：

```text
analysis_close_time_utc
lookback_count
```

---

## 11. 增量采集与 lookback window

### 11.1 默认行为

增量采集默认不得只采最新一根 K 线。

每次增量采集必须使用 lookback window，向前多请求若干根已收盘 K 线。

目的：

```text
检查最近 K 线连续性。
自动补齐 lookback 范围内的短期漏采。
发现重新拉取后的数据冲突。
降低单次任务失败造成的数据缺口风险。
```

### 11.2 示例

如果数据库已有：

```text
04:00
08:00
```

当前已收盘到：

```text
16:00
```

增量采集不应只请求：

```text
16:00
```

而应请求最近多根已收盘 K 线，例如：

```text
04:00
08:00
12:00
16:00
```

处理方式：

```text
04:00 已存在且一致 → 跳过
08:00 已存在且一致 → 跳过
12:00 不存在 → 插入
16:00 不存在 → 插入
```

### 11.3 lookback 范围内缺口

lookback window 内发现数据库缺少某根 K 线时，数据采集模块允许直接幂等写入补齐。

这属于增量采集的短期自愈能力。

### 11.4 lookback 范围外缺口

以下情况应交由数据质量 / 数据回补模块处理：

```text
lookback window 外缺口。
连续多根 K 线缺失。
历史大范围缺口。
数据冲突。
交易所返回异常。
数据库中已存在 K 线与重新采集结果不一致。
```

### 11.5 lookback_count 配置

lookback_count 应可配置。

第一阶段建议默认：

```text
4h：最近 10 根已收盘 K 线
1d：最近 5 根已收盘 K 线
```

具体数值可在配置中调整。

---

## 12. 已收盘 K 线规则

系统只能把已收盘 K 线作为标准行情写入主 K 线表。

要求：

```text
未收盘 K 线不得写入 4h / 1d 主 K 线表。
未收盘 K 线不得用于第一阶段策略分析。
未收盘 K 线不得生成 MarketSnapshot。
未收盘 K 线不得进入 FeatureLayer。
```

已收盘判断：

```text
Binance serverTime > K 线 close_time
```

如果无法确认 K 线已收盘：

```text
不得写入主 K 线表。
必须记录 DataCollectionRun 采集审计摘要。
可以在后续任务中重新拉取。
```

---

## 13. 采集成功流程

### 13.1 历史采集流程

```text
接收历史采集请求
→ 校验 exchange / market_type / symbol / timeframe / 时间范围
→ 调用 Binance REST
→ 过滤未收盘 K 线
→ 转换为内部 KlineRecord
→ 按唯一业务键幂等写入对应周期 K 线表
→ 记录 DataCollectionRun 审计摘要
→ 记录 TaskRunRecord
→ 输出采集结果摘要
→ 进入或等待数据质量检查
```

### 13.2 增量采集流程

```text
接收增量采集请求
→ 根据 timeframe 和 lookback_count 计算请求范围
→ 调用 Binance REST 获取最近多根已收盘 K 线
→ 过滤未收盘 K 线
→ 与数据库已有 K 线按唯一业务键比对
→ 插入缺失 K 线
→ 发现冲突则记录异常，不静默覆盖
→ 记录 DataCollectionRun 审计摘要
→ 记录 TaskRunRecord
→ 进入或等待数据质量检查
```

---

## 14. 失败流程

采集失败必须记录，不得静默成功。

必须记录：

```text
trace_id
trigger_source
collection_mode
exchange
market_type
symbol
timeframe
start_time_utc
end_time_utc
错误类型
错误码
错误消息
是否可重试
```

典型失败：

```text
Binance REST 请求超时
Binance REST 限频
Binance REST 返回异常
K 线数据为空
时间范围非法
返回数据格式异常
K 线未收盘
写入数据库失败
唯一键冲突
同一唯一键数据不一致
```

失败后禁止：

```text
伪造 K 线。
手工补值。
静默成功。
生成 MarketSnapshot。
触发策略分析。
直接调用 Hermes 阻塞流程。
直接调用 OrderPlan、RiskCheck、ExecutionPreparation 或 Execution。
```

---

### 14.1 Retry 规则

数据采集模块允许在同一次 DataCollectionRun 生命周期内执行受控同步 retry。

允许 retry 的场景仅包括：

```text
Binance REST 请求 timeout
网络 connection error
HTTP 429 / 限频
HTTP 5xx
```

禁止 retry 的场景包括：

```text
HTTP 4xx（429 除外）
JSON 解析失败
Binance payload 结构校验失败
K 线周期边界校验失败
未收盘 K 线进入待写入集合
同一唯一键数据冲突
数据库写入错误
唯一键冲突无法归类为幂等跳过
业务参数非法
```

规则：

```text
retry 不得跨 DataCollectionRun 异步漂移。
retry 不得在下游 MarketSnapshot / FeatureLayer 已经开始后继续补写本轮 K 线。
retry 后仍失败时，DataCollectionRun 必须 failed 或 blocked。
失败、冲突或无法确认写入状态时，allows_downstream 必须为 False 或等价不可下游消费状态。
```

## 15. 数据冲突处理

数据冲突指：

```text
同一唯一业务键的 K 线重新采集后，核心行情字段与数据库已有记录不一致。
```

核心行情字段包括：

```text
open_price
high_price
low_price
close_price
volume
quote_volume
trade_count
```

处理规则：

```text
不得静默覆盖。
不得人工修改。
不得以 manual_repair / system_repair / human_edit 作为数据来源。
必须记录 DataConflict 或 DataQualityIssue。
必须记录旧值摘要和新值摘要。
必须记录 trace_id、data_source、trigger_source、collection_mode。
是否覆盖或修复由后续数据质量 / 数据回补规则决定。
```

如果已有 StrategySignal、DecisionSnapshot 或回测结果引用过该 K 线，后续处理必须保证复盘可解释。

---

## 16. 数据来源、触发来源与采集模式

本项目拆分三个概念：

```text
data_source = 数据来源
trigger_source = 触发来源
collection_mode = 采集模式
```

### 16.1 data_source

表示行情数据从哪里来。

允许：

```text
binance_rest
binance_websocket
```

禁止作为正式 4h / 1d K 线来源：

```text
manual_repair
system_repair
human_edit
manual_input
```

### 16.2 trigger_source

表示谁触发了采集任务。

允许：

```text
cli
management_command
celery_beat
celery_worker
manual_recovery
retry
system
admin
```

### 16.3 collection_mode

表示本次采集目的。

允许：

```text
historical
latest_closed
incremental
manual_backfill
```

说明：

```text
collection_mode 不得替代 data_source。
manual_backfill 不得成为人工改写 K 线的理由。
```

---

## 17. 存储要求

### 17.1 MySQL

K 线必须结构化存入 MySQL。

要求：

```text
一根 K 线一行。
4h 与 1d 分表存储。
不得把一批 K 线塞进 JSON 字段。
不得只写日志。
不得只存在 Redis。
```

第一阶段至少需要支持：

```text
4h K 线表
1d K 线表
采集运行审计摘要记录
采集任务记录
异常 / 告警事件记录
```

具体表名和字段由后续 plans 和数据库设计确定。

### 17.1.1 索引要求

4h K 线表和 1d K 线表必须支持按交易品种和时间范围高效查询。

每张 K 线表至少需要建立复合唯一约束：

```text
(exchange, market_type, symbol, open_time_utc)
```

该复合唯一约束用于防止同一根 K 线被重复写入。

适用场景包括：

```text
历史采集
增量 lookback window 采集
短期漏采自动补齐
人工指定区间回补
重复任务重跑
```

每张 K 线表还必须支持以下查询场景：

```text
按 exchange + market_type + symbol 查询
按 open_time_utc 时间范围查询
按时间正序读取连续 K 线
```

如果数据库实现中，复合唯一约束已经覆盖以下查询路径：

```text
(exchange, market_type, symbol, open_time_utc)
```

则可以不重复建立相同字段顺序的普通索引。

如果后续某张 K 线表支持多个 timeframe，则唯一约束必须改为：

```text
(exchange, market_type, symbol, timeframe, open_time_utc)
```

第一阶段由于 4h K 线和 1d K 线分表存储，单表唯一约束中不强制包含 `timeframe`。


### 17.2 Redis

Redis 可用于：

```text
短期限频
短期任务状态
分布式锁
WebSocket 最新价格状态
WebSocket 连接状态
去重 key
```

Redis 禁止：

```text
作为 4h / 1d K 线主存储。
长期保存核心 K 线。
替代 MySQL。
```

Redis 数据必须可过期、可重建、可丢失后恢复。

---

## 18. Hermes 与告警事件

数据采集模块可以产生异常事件或告警事件，但不直接同步调用 Hermes Webhook。

正确模式：

```text
数据采集模块
→ 写入 DataCollectionRun / AlertEvent
→ notifications / Hermes 服务异步消费
→ 发送 HermesNotification
```

要求：

```text
采集异常必须入库。
采集异常不得依赖 Hermes 发送成功才算记录完成。
Hermes 发送失败不得改变数据采集事实。
Hermes 不得反向触发采集补写、策略分析或交易执行。
```

数据采集模块只负责写库，不负责搭建 Hermes 底层消费服务。

---

## 19. 与数据质量和数据回补的关系

### 19.1 与数据质量模块

数据采集成功不等于数据可信。

采集完成后，必须进入或等待数据质量检查。

数据质量模块负责：

```text
连续性检查
重复检查
缺失检查
OHLC 合法性检查
已收盘检查
周期边界检查
数据源检查
```

数据采集模块可以发现明显异常并记录事件，但最终是否允许进入 MarketSnapshot，由数据质量模块决定。

### 19.2 与数据回补模块

数据采集和数据回补是代码职责拆分，不是存储分表拆分。

```text
DataCollectionService = 正常采集、历史初始化采集、增量 lookback 采集。
DataBackfillService = 缺口回补、人工指定区间回补、较大范围补偿、冲突复核。
```

存储规则：

```text
正常采集来的 4h K 线写入 4h K 线表。
回补采集来的 4h K 线也写入 4h K 线表。
正常采集来的 1d K 线写入 1d K 线表。
回补采集来的 1d K 线也写入 1d K 线表。
```

不得创建“正常采集 K 线表”和“回补 K 线表”来拆分同一周期 K 线。

---

## 20. 禁止事项

数据采集模块禁止：

```text
使用本地时间作为业务判断时间。
请求 K 线时传 timeZone 参数。
采集未收盘 K 线进入主 K 线表。
将 WebSocket 数据写入 4h / 1d 主 K 线表。
把 4h 和 1d K 线混入同一张业务表。
使用数据库自增 id 判断 K 线顺序。
使用数据库自增 id 判断 K 线连续性。
静默覆盖同一唯一键下不同 OHLCV 的 K 线。
人工修改正式 K 线 OHLCV。
使用 manual_repair / system_repair / human_edit 作为正式 K 线来源。
生成 MarketSnapshot。
计算 FeatureValue。
生成 AtomicSignal。
生成 StrategySignal。
生成 DecisionSnapshot。
读取账户或仓位。
调用 OrderPlan。
生成 CandidateOrderIntent。
生成 ApprovedOrderIntent。
调用 RiskCheck。
调用 ExecutionPreparation。
调用 Execution。
调用大模型。
同步等待 Hermes 发送成功。
```

---

## 21. 验收标准

第一阶段完成后，数据采集模块必须满足：

```text
可以通过 Binance REST 获取 BTCUSDT U 本位 4h 已收盘 K 线。
可以通过 Binance REST 获取 BTCUSDT U 本位 1d 已收盘 K 线。
4h 和 1d K 线分别写入独立业务表。
采集代码可复用，但存储按周期隔离。
所有核心时间统一 UTC。
请求 K 线时不传 timeZone。
不会把未收盘 K 线写入主 K 线表。
增量采集默认使用 lookback window。
lookback 范围内短期漏采可以自动补齐。
重复采集不会产生重复 K 线。
同一唯一键数据不一致时不会静默覆盖。
采集失败会记录 DataCollectionRun 或等价 TaskRunRecord。
DataCollectionRun 会记录 fetched / inserted / skipped / duplicate / conflict / retry 等审计摘要。
retry 只用于 timeout / connection error / HTTP 429 / HTTP 5xx。
非 retryable 错误不会被无限重试。
采集异常会写入 AlertEvent 或等价事件。
Hermes 不会阻塞数据采集流程。
K 线排序、连续性、缺口判断不依赖数据库自增 id。
WebSocket 不写入 4h / 1d 主 K 线表。
WebSocket 不作为第一阶段策略主周期输入。
采集成功后进入或等待数据质量检查。
数据采集模块不会生成任何交易相关对象。
```

---

## 22. 后续待拆分文档

以下内容不在本文档展开，应由后续模块文档定义：

```text
data_quality.md
= 连续性、重复、缺失、脏数据、冲突检查、是否允许进入 MarketSnapshot。

data_backfill.md
= 缺口回补、人工指定区间回补、较大范围补偿、冲突复核。

market_snapshot.md
= 如何从通过质量检查的数据生成 MarketSnapshot。

notifications.md / monitoring_recovery.md 为后续补充文档，当前阶段不作为 001/002 阶段阻断依赖。

notifications.md
= 如何消费 AlertEvent 并发送 HermesNotification。

monitoring_recovery.md
= WebSocket 风险事件、闪崩事件、异常恢复和人工处理。
```

---

## 23. 总结

数据采集模块只负责可信地获取市场原始数据，并以可追溯、可幂等、可审计的方式写入系统。

第一阶段核心口径：

```text
Binance REST 是 4h / 1d 主 K 线标准来源。
WebSocket 是实时行情监测来源，不是主 K 线来源。
所有时间统一 UTC。
4h 和 1d 分表存储。
增量采集默认使用 lookback window。
lookback 范围内短期漏采由采集模块自动补齐。
超出 lookback 的缺口和数据冲突交给数据质量 / 数据回补模块。
采集异常只写运行审计和 AlertEvent，不同步阻塞 Hermes。
采集成功不等于数据可信，必须进入数据质量检查。
```
