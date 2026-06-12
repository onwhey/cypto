# Plan 002B：Binance REST K 线采集网关与采集服务

> 状态：正式版 v2  
> 前置阶段：002A 行情 K 线基础模型已完成并验收  
> 本阶段核心：REST 已收盘 K 线采集、短时同步 retry、基础校验、代码层去重、幂等写入、lookback window 补漏、采集运行审计  
> 本阶段不做：WebSocket、完整 data_quality、data_backfill、MarketSnapshot、策略、风控、交易

---

## 0. 执行边界

本 plan 是 002A 之后的第一个行情采集开发计划。

本阶段只负责：

```text
通过 Binance USDⓈ-M Futures REST market data API
获取 BTCUSDT U 本位合约 4h / 1d 已收盘 K 线
经过基础校验、代码层去重和幂等写入
写入 002A 已建立的 Kline4h / Kline1d 主事实表
并记录一次采集运行 DataCollectionRun
```

本阶段必须保持以下边界：

```text
采集层负责“正常采集 + 短 lookback window 补漏”。
完整 data_quality 负责“采集完成后的质量判断与阻断”。
data_backfill 负责“明确缺口的历史回补 / 人工回补 / 冲突复核”。
WebSocket 只作为未来即时消息能力，不参与任何 K 线采集任务。
```

Codex 不得把本 plan 扩展成完整数据质量系统、完整回补系统、MarketSnapshot、策略或交易系统。

---

## 1. 阶段目标

本阶段目标是建立一个最小但可靠的行情采集闭环：

```text
Binance REST client
→ serverTime 获取
→ Kline REST 拉取
→ 已收盘过滤
→ open_time / close_time 归一化
→ 单根 K 线基础校验
→ 同批响应代码层去重
→ lookback window 缺口识别
→ 与数据库已有 K 线比对
→ 幂等写入 Kline4h / Kline1d
→ DataCollectionRun 审计记录
→ 失败 / 阻断时写 AlertEvent
```

本阶段完成后，项目应具备：

```text
apps/market_data/binance_client.py 或等价 REST client
apps/market_data/validators.py 或 domain_rules.py
apps/market_data/services.py 或 data_collection service
DataCollectionRun model / migration
KlineWriteLock model / migration
management command 入口
可选 Celery task 入口
基础单元测试 / service 测试
implementation 文档
```

本阶段只支持第一阶段交易范围：

```text
exchange = binance
market_type = usds_m_futures
symbol = BTCUSDT
timeframe = 4h / 1d
data_source = binance_rest
```

---

## 2. 前置条件

开始前必须确认 002A 已验收：

```text
market_data app 已存在
Kline4h model 已存在
Kline1d model 已存在
Kline4h / Kline1d 分表
无 is_closed
无 raw_payload
无 Binance 原始 closeTime 字段
DecimalField(max_digits=30, decimal_places=12)
唯一约束：exchange + market_type + symbol + open_time_utc
基础索引存在
OHLCV CheckConstraint 已声明
migrations 已存在并已执行
测试通过
```

如果 002A 未通过，不得开始 002B。

---

## 3. 依赖文档

Codex 开始前必须阅读：

```text
AGENTS.md
docs/rules/project_invariants.md
docs/requirements/01_project_scope.md
docs/requirements/02_system_capabilities.md
docs/requirements/data_collection.md
docs/requirements/data_quality.md
docs/requirements/data_backfill.md
docs/requirements/market_snapshot.md
docs/architecture/00_project_overview.md
docs/architecture/01_module_map.md
docs/architecture/system_architecture.md
docs/architecture/data_flow.md
docs/implementation/001A_project_foundation_framework.md
docs/implementation/001B_project_foundation_hermes.md
docs/implementation/002A_market_data_models.md
docs/plans/002B_data_collection_gateway_and_service_plan.md
```

如果本 plan 与 `AGENTS.md` 或 `docs/rules/project_invariants.md` 冲突，必须以后两者为准，并在总结中说明冲突。

如果本 plan 与 requirements 冲突，必须停止并提示用户确认。

---

## 4. 官方 API 范围

本阶段只允许使用 Binance USDⓈ-M Futures Market Data REST API。

允许接口：

```text
GET /fapi/v1/time
GET /fapi/v1/klines
```

用途：

```text
/fapi/v1/time
= 获取 Binance server time，用于判断已收盘边界，避免依赖本机时钟。

/fapi/v1/klines
= 获取 BTCUSDT U 本位合约 4h / 1d K 线。
```

注意：

```text
Kline 由 open time 唯一识别。
/fapi/v1/klines 的请求权重基于 LIMIT 参数变化。
limit 默认 500，最大 1500。
```

本阶段不使用账户、订单、持仓、交易相关接口。

---

## 5. 本阶段明确不做

本阶段禁止实现：

```text
WebSocket
用 WebSocket 拼 K 线
用 WebSocket 写 Kline4h / Kline1d
Spot API
COIN-M Futures API
账户接口
仓位接口
订单接口
下单接口
自动交易
Execution
CandidateOrderIntent / ApprovedOrderIntent
RiskCheck

StrategySignal
AtomicSignal
FeatureLayer
DecisionSnapshot
MarketSnapshot
data_quality app
DataQualityResult
DataQualityIssue
data_backfill app
BackfillRun
BackfillResult
BackfillRequest
历史大区间回补
人工指定区间回补
全库历史连续性扫描
跨周期一致性检查
回测
大模型分析
```

本阶段禁止私自扩展到多品种、多市场、多交易所。

---

## 6. WebSocket 绝对边界

WebSocket 不参与任何 K 线采集任务。

本阶段禁止：

```text
WebSocket client
WebSocket stream
通过 WebSocket 构造 4h / 1d K 线
通过 WebSocket 修复 K 线
通过 WebSocket 写入 Kline4h / Kline1d
通过 WebSocket 参与 lookback window 补漏
```

项目后续如需要即时消息能力，WebSocket 只能用于：

```text
最新价格
mark price
盘口
成交推送
订单状态
账户 / 仓位状态
极端行情风控触发
```

正式 K 线主事实来源只能是：

```text
Binance REST 已收盘 K 线
```

---

## 7. 建议新增 / 修改文件

建议新增：

```text
apps/market_data/binance_client.py
apps/market_data/validators.py
apps/market_data/services.py
apps/market_data/management/commands/collect_klines.py
apps/market_data/tasks.py                  # 可选，若接入 Celery
apps/market_data/migrations/0002_datacollectionrun.py
tests/market_data/test_binance_client.py
tests/market_data/test_kline_validators.py
tests/market_data/test_data_collection_service.py
tests/market_data/test_collect_klines_command.py
docs/implementation/002B_data_collection_gateway_and_service.md
```

允许修改：

```text
apps/market_data/models.py                 # 新增 DataCollectionRun / KlineWriteLock
apps/market_data/constants.py              # 新增 timeframe / collection mode / run status / error code 常量
config/settings.py                         # 新增 Binance market data 配置项
config/celery.py                           # 如需 task 注册，保持入口轻量
```

禁止新增：

```text
apps/data_quality/
apps/data_backfill/
apps/market_snapshot/
apps/strategy/
apps/risk/
apps/execution/
apps/realtime_market/
apps/market_stream/
```

---

## 8. DataCollectionRun 模型

### 8.1 模型定位

`DataCollectionRun` 用于记录一次采集运行。

它是批次级 / 运行级审计表，不是逐根 K 线事件表。

002B P0 只实现 `DataCollectionRun` 作为采集审计摘要。
`DataCollectionEvent` 不属于 002B P0；如未来需要逐请求 / 逐事件明细记录，作为 P1 扩展。

### 8.2 字段要求

`DataCollectionRun` 至少覆盖：

```text
exchange
market_type
symbol
timeframe
data_source
collection_mode
status
requested_start_time_utc
requested_end_time_utc
server_time_utc
expected_count
fetched_count
closed_count
inserted_count
skipped_existing_count
duplicate_in_response_count
conflict_count
missing_count
missing_open_times
attempt_count
max_attempts
first_attempt_at_utc
last_attempt_at_utc
finished_at_utc
error_code
error_message
trace_id
trigger_source
created_at_utc
updated_at_utc
```

字段可根据 Django 实现做合理可空处理。

### 8.3 状态建议

```text
running
retrying
success
failed
blocked
skipped
```

语义：

```text
running  = 当前采集运行已开始
retrying = 当前采集运行正在进行短时同步 retry
success  = 请求成功、校验完成、写入完成、lookback window 无未解决缺口
failed   = 请求失败 / 响应非法 / 解析失败 / 校验失败 / retry 后仍失败
blocked  = 并发冲突、已有同 symbol/timeframe running/retrying run，或其他阻断条件
skipped  = 明确无需采集，例如窗口无新已收盘 K 线
```

### 8.4 collection_mode 建议

```text
scheduled_lookback
manual_lookback
```

本阶段不做历史大区间 backfill mode。

---

## 9. Binance REST client 要求

### 9.1 client 职责

REST client 只负责 HTTP 请求和响应基础解析。

允许：

```text
get_server_time()
get_klines(symbol, interval, start_time, end_time, limit)
```

禁止：

```text
写数据库
写 AlertEvent
写 DataCollectionRun
调用 data_quality
调用 data_backfill
生成 MarketSnapshot
调用策略 / 风控 / 交易
```

### 9.2 配置项

建议在 settings 中新增：

```text
BINANCE_FAPI_BASE_URL=https://fapi.binance.com
BINANCE_HTTP_TIMEOUT_SECONDS=10
BINANCE_KLINE_LIMIT_DEFAULT=20
BINANCE_COLLECTION_LOOKBACK_BARS_4H=5
BINANCE_COLLECTION_LOOKBACK_BARS_1D=5
BINANCE_HTTP_MAX_ATTEMPTS=3
BINANCE_HTTP_RETRY_BASE_DELAY_SECONDS=1
BINANCE_HTTP_RETRY_MAX_DELAY_SECONDS=5
BINANCE_HTTP_RETRY_TOTAL_BUDGET_SECONDS=15
```

不得把 URL、timeout、lookback、retry 参数硬编码散落在业务代码中。

---

## 10. 短时同步 retry 要求

### 10.1 retry 定位

本阶段允许 Binance REST client 做有限、短时、同步 retry。

retry 的目的只是处理短暂网络抖动、临时限流或服务端临时错误。

retry 不得变成隐藏 backfill，不得跨越当前采集运行，不得跨 analysis cycle。

### 10.2 retry 只允许发生在当前 DataCollectionRun 内

必须满足：

```text
retry 只能发生在当前 DataCollectionRun 生命周期内部。
retry 完成前，不允许 data_quality、MarketSnapshot、策略流程继续执行。
retry 不得使用长延迟异步 Celery retry。
retry 不得在当前 collection run 失败后 5 分钟 / 10 分钟再自动补写。
retry 不得跨 analysis cycle。
```

明确禁止这种情况：

```text
04:05 collection 失败
04:10 data_quality 已经开始并判定缺失
04:15 collection retry 又写入 K 线
```

这种设计会造成：

```text
data_quality 结果失真
data_backfill 重复触发
MarketSnapshot 基于半更新数据
策略信号不可复现
审计链混乱
```

### 10.3 允许 retry 的错误

只允许对临时性 HTTP 失败 retry：

```text
timeout
connection error
HTTP 429
HTTP 5xx
```

### 10.4 禁止 retry 的错误

以下错误不得 retry：

```text
HTTP 4xx，429 除外
JSON 结构非法
Kline payload 字段数量不符合预期
Kline 字段解析失败
OHLCV 校验失败
同批 K 线冲突
数据库已有 K 线冲突
数据库写入失败
代码 bug / 未知异常
```

这些错误应直接标记当前 run 为 failed 或 blocked，并记录 error_code / error_message。

### 10.5 retry 参数

默认建议：

```text
max_attempts = 3
base_delay_seconds = 1
max_delay_seconds = 5
total_retry_budget_seconds <= 15
```

示例：

```text
第 1 次请求失败 → 等 1 秒
第 2 次请求失败 → 等 2 秒
第 3 次仍失败 → 当前 DataCollectionRun failed
```

### 10.6 retry 日志

每次 retry 必须使用 logging 记录：

```text
trace_id
trigger_source
endpoint
attempt
max_attempts
error_code
error_message
next_delay_seconds
```

retry 不得写 Kline 表。

真正写库必须在最终拿到有效响应、完成解析、校验、去重之后进行。

---

## 11. 下游流程阻断要求

002B 必须为后续 RunAnalysisCycleService 提供清晰结果。

如果 collection 成功：

```text
DataCollectionRun.status = success
允许后续 data_quality 执行
```

如果 collection retry 后仍失败：

```text
DataCollectionRun.status = failed
写 error_code / error_message
必要时写 AlertEvent
当前 analysis cycle 必须阻断
不生成 MarketSnapshot
不进入策略 / 风控 / 交易
```

如果发现并发冲突：

```text
DataCollectionRun.status = blocked 或 skipped
当前 run 不得继续写入 Kline
必要时写 AlertEvent
```

注意：

```text
data_quality 可以作为 K 线质量保底流程。
data_backfill 可以作为明确缺口补齐流程。
但它们不能与当前 collection retry 并发抢写同一窗口。
```

---

## 12. 并发控制要求

同一个：

```text
exchange + market_type + symbol + timeframe
```

在同一时间只能有一个 `running` 或 `retrying` 的 collection run。

本阶段至少要在代码层检查：

```text
如果已有同业务键 running / retrying DataCollectionRun：
- 新 run 不启动采集
- 标记为 blocked 或 skipped
- 记录 existing_run 信息
- 不请求 Binance
- 不写 Kline
```

如实现数据库层唯一约束或锁，也必须保持 Django ORM / transaction 方式，不得自研锁框架。

禁止：

```text
多个 collection run 并发写同一 symbol/timeframe
collection retry 与下一轮 collection 并发写同一窗口
collection retry 与 data_backfill 并发写同一 open_time
```

数据库唯一约束是最后防线，不是正常并发控制手段。

KlineWriteLock 属于 `apps.market_data`，由 002B 实现。
data_collection 和 data_backfill 必须共用 KlineWriteLock 保护 Kline4h / Kline1d 主事实表写入。
KlineWriteLock 的粒度为 exchange + market_type + symbol + timeframe。
MarketSnapshot 只检查 KlineWriteLock 是否活跃，不创建锁。

---

## 13. 时间边界与已收盘过滤

### 13.1 不信任本机时间

判断已收盘 K 线时必须使用 Binance server time。

禁止使用：

```text
datetime.now()
time.time()
机器本地时区
```

来决定当前 K 线是否已收盘。

### 13.2 open_time / close_time 规则

002A 已确定：

```text
close_time_utc = open_time_utc + timeframe
```

本阶段不保存 Binance 原始 closeTime。

解析 Binance Kline payload 时：

```text
使用 open time 作为唯一业务定位。
忽略 Binance 原始 close time，不保存，不作为业务边界。
```

### 13.3 已收盘判断

对每根 K 线：

```text
candidate_close_time_utc = open_time_utc + timeframe
```

只有满足：

```text
server_time_utc >= candidate_close_time_utc
```

才允许进入写入候选集。

未收盘 K 线必须过滤，不写入 Kline4h / Kline1d。

---

## 14. lookback window 补漏要求

### 14.1 定位

002B 必须支持短 lookback window 补漏。

这是 data_collection 的职责，不是完整 data_quality。

目的：

```text
如果上一轮采集失败，本轮采集能通过最近 N 根已收盘 K 线补上短期缺口。
```

### 14.2 默认窗口

建议默认：

```text
4h lookback = 最近 5 根已收盘 K 线
1d lookback = 最近 5 根已收盘 K 线
```

可通过 settings 配置。

### 14.3 expected open_time 生成

基于 Binance server time 和 timeframe 计算最近 N 根已收盘 K 线的 `open_time_utc`。

例如 4h，在 server_time 为 16:05 UTC 时，最近已收盘边界为 16:00，lookback=5，应检查：

```text
00:00
04:00
08:00
12:00
16:00
```

### 14.4 请求范围

应根据 expected open_time 生成 REST 请求范围：

```text
start_time = 最早 expected open_time
end_time   = 最后 expected open_time + timeframe
limit      = 至少 expected_count，且保持小 limit，默认 20 以内
```

本阶段不得使用大 limit 做历史回补。

### 14.5 请求后完整性检查

本次请求、过滤、去重、写入后，必须再次查询数据库确认 lookback window 内 expected open_time 是否齐全。

如果仍缺失：

```text
missing_count > 0
missing_open_times 记录缺失 open_time
DataCollectionRun.status = failed 或 blocked
必要时写 AlertEvent
```

但本阶段不创建 DataQualityResult，不创建 DataQualityIssue，不创建 BackfillRequest，不触发回补执行。

---

## 15. K 线基础规则复用原则

本阶段允许在 `apps/market_data/validators.py` 或 `apps/market_data/domain_rules.py` 中实现无副作用基础规则函数。

允许函数包括：

```text
parse_binance_kline_payload(...)
normalize_open_time_utc(...)
calculate_close_time_utc(open_time_utc, timeframe)
validate_kline_ohlcv(...)
build_expected_open_times(...)
filter_closed_klines(...)
dedupe_kline_items_by_open_time(...)
compare_kline_values(...)
find_missing_open_times(...)
```

这些函数只能负责基础事实规则。

不得：

```text
请求 Binance
写数据库
写 AlertEvent
创建 DataCollectionRun
创建 DataQualityResult
创建 DataQualityIssue
触发 data_backfill
生成 MarketSnapshot
调用策略 / 风控 / 交易模块
```

后续 data_quality 可以复用这些基础规则，但 data_quality workflow 不得在 002B 中提前实现。

---

## 16. 入库前基础校验

002B 写入 Kline4h / Kline1d 前必须检查：

```text
Binance 返回字段数量符合预期
open_time 能解析为 UTC timezone-aware datetime
close_time_utc = open_time_utc + timeframe
只写已收盘 K 线
OHLCV 能转换为 Decimal
open_price > 0
high_price > 0
low_price > 0
close_price > 0
high_price >= low_price
high_price >= open_price
high_price >= close_price
low_price <= open_price
low_price <= close_price
volume >= 0
quote_volume >= 0
trade_count >= 0
symbol / interval 与请求一致
```

校验失败的 K 线不得写入正式表。

本阶段不把这些校验命名为 data_quality，也不得创建 data_quality app。

---

## 17. 代码层去重要求

002B 写入 Kline4h / Kline1d 前，必须先在代码层按业务键去重：

```text
exchange + market_type + symbol + open_time_utc
```

不得依赖数据库唯一约束作为正常去重流程。

### 17.1 同批 Binance 返回去重

若同批返回存在重复 open_time_utc：

```text
内容一致：只保留一条，duplicate_in_response_count +1
内容不一致：记录 conflict，不写入该 open_time_utc
```

### 17.2 与数据库已有记录比对

若数据库已存在同业务键：

```text
OHLCV / close_time_utc / trade_count 一致：跳过，skipped_existing_count +1
OHLCV / close_time_utc / trade_count 不一致：记录 conflict，不覆盖、不更新
```

若数据库不存在：

```text
通过基础校验后插入
```

### 17.3 冲突处理

遇到 conflict：

```text
conflict_count +1
记录 conflict open_time
不覆盖已有数据
不更新已有数据
不删除已有数据
DataCollectionRun.status = failed 或 blocked
必要时写 AlertEvent
```

本阶段不做冲突复核，不做自动修复。

冲突复核属于后续 data_backfill / data_quality 相关阶段。

---

## 18. 幂等写入要求

同一 collection run 或重复执行相同窗口时，必须满足：

```text
不会重复插入同一根 K 线
已存在且一致的数据会被跳过
已存在但不一致的数据不会被覆盖
重复执行不会改变历史 OHLCV
```

实现上应使用 Django ORM 和 transaction。

禁止：

```text
bulk blind insert 直接依赖唯一键报错
update_or_create 覆盖历史 K 线
无条件 delete + insert
ON DUPLICATE KEY UPDATE 覆盖 OHLCV
原生 SQL 绕过 ORM
```

---

## 19. AlertEvent 边界

本阶段允许在采集失败、retry 后失败、冲突、窗口仍缺失、并发阻断时写 AlertEvent。

但必须遵守：

```text
业务模块只写 AlertEvent。
不得直接调用 Hermes client。
不得直接请求 Hermes webhook。
通知发送仍由 notifications service / Celery 异步处理。
```

AlertEvent 只用于告警，不参与采集修复，不参与策略决策。

---

## 20. Management command 入口

必须提供轻量入口，例如：

```bash
python manage.py collect_klines --symbol BTCUSDT --timeframe 4h
python manage.py collect_klines --symbol BTCUSDT --timeframe 1d
python manage.py collect_klines --symbol BTCUSDT --timeframe all
```

入口只负责：

```text
解析参数
设置 trace_id / trigger_source
调用 application service
输出简要结果
```

禁止在 command 中承载核心采集逻辑。

---

## 21. Celery task 入口

本阶段可以实现 Celery task，但不是必须。

如果实现，必须保持入口轻量：

```text
设置 trace_id / trigger_source
调用 collection service
返回 DataCollectionRun 结果摘要
```

禁止使用 Celery long-delayed retry 来补采当前失败 run。

禁止：

```text
self.retry(countdown=300)
失败 5 分钟后自动重试写 K 线
跨 analysis cycle 异步补写
```

调度 cadence 可以后续再由 Celery Beat plan 明确。

---

## 22. RunAnalysisCycleService 接入预留

本阶段可以不实现 `RunAnalysisCycleService`。

但 002B 的 service 返回结果必须能支持后续编排：

```text
collection success → 允许 data_quality
collection failed / blocked → 阻断当前 analysis cycle
```

不得设计成：

```text
data_quality 固定按时间启动，与 collection 状态无关
MarketSnapshot 固定按时间启动，与 collection 状态无关
策略固定按时间启动，与 data_quality 状态无关
```

后续整体流程应是：

```text
RunAnalysisCycleService
→ data_collection
→ data_quality
→ MarketSnapshot
→ strategy
```

而不是多个任务按时间并行抢跑。

---

## 23. 测试要求

必须测试：

### 23.1 REST client

```text
get_server_time 成功解析 serverTime
get_klines 成功解析 payload
HTTP timeout 会进行短时同步 retry
HTTP 5xx 会进行短时同步 retry
HTTP 429 会进行短时同步 retry
HTTP 400 不 retry
超过 max_attempts 后抛出受控异常
retry 日志包含 trace_id / endpoint / attempt
```

测试不得请求真实 Binance。

必须使用 mock / fake HTTP client。

### 23.2 validators / domain_rules

```text
open_time 毫秒时间戳解析为 UTC datetime
close_time_utc = open_time_utc + timeframe
未收盘 K 线被过滤
OHLCV 合法数据通过
OHLCV 非法数据失败
expected open_time 列表生成正确
lookback window 缺口识别正确
同批重复 open_time 内容一致时去重
同批重复 open_time 内容不一致时标记 conflict
K 线字段比较一致 / 不一致能正确识别
```

### 23.3 collection service

```text
4h K 线成功写入 Kline4h
1d K 线成功写入 Kline1d
已存在且一致时跳过，不重复插入
已存在但不一致时 conflict，不覆盖
同批重复返回不会重复插入
lookback window 可补上上一轮漏掉的 K 线
本次请求后仍缺失时 DataCollectionRun failed / blocked
HTTP retry 成功后只写入一次 K 线
HTTP retry 失败后 DataCollectionRun failed
retry 完成前不会启动下游流程
已有 running / retrying run 时新 run blocked / skipped
失败 / blocked 时必要情况下写 AlertEvent
不直接调用 Hermes client
```

### 23.4 management command

```text
collect_klines --timeframe 4h 调用 service
collect_klines --timeframe 1d 调用 service
collect_klines --timeframe all 分别处理 4h / 1d
参数非法时失败且不请求 Binance
```

---

## 24. 测试禁止项

测试禁止：

```text
请求真实 Binance
请求真实 Hermes
连接生产 MySQL
连接生产 Redis
依赖机器本地时间判断已收盘
使用 WebSocket
实现 data_quality workflow
实现 data_backfill workflow
生成 MarketSnapshot
调用策略 / 风控 / 交易
```

---

## 25. migration 要求

如新增 `DataCollectionRun` / `KlineWriteLock` model，必须创建 migration：

```text
apps/market_data/migrations/0002_datacollectionrun.py
```

Codex 完成后必须运行：

```bash
python manage.py makemigrations market_data
python manage.py makemigrations --check --dry-run
python manage.py migrate --plan
python manage.py check
python manage.py test
pytest
```

如果新增 model 但没有 migration，必须停止并修复。

---

## 26. implementation 文档要求

完成后必须新增：

```text
docs/implementation/002B_data_collection_gateway_and_service.md
```

内容至少包括：

```text
本阶段新增 / 修改文件清单
Binance REST client 职责
使用的官方 REST endpoint
serverTime 用途
Kline open_time / close_time 归一化规则
已收盘过滤规则
lookback window 补漏规则
短时同步 retry 规则
retry 不得跨 analysis cycle 的说明
下游流程阻断规则
并发控制规则
代码层去重规则
幂等写入规则
冲突处理规则
DataCollectionRun 字段说明
AlertEvent 使用边界
测试方式
本阶段明确未实现内容
```

必须明确写入：

```text
本阶段未实现 WebSocket。
本阶段未实现完整 data_quality。
本阶段未实现 DataQualityResult / DataQualityIssue。
本阶段未实现 data_backfill。
本阶段未实现 MarketSnapshot。
本阶段未实现策略、风控、交易。
本阶段未使用 WebSocket 参与 K 线采集。
```

---

## 27. 人工审查清单

合并前必须检查：

```text
是否只新增 REST 采集相关代码
是否未实现 WebSocket
是否未用 WebSocket 写 Kline
是否未实现 data_quality app
是否未实现 data_backfill app
是否未实现 MarketSnapshot
是否未实现策略 / 风控 / 交易
是否没有账户 / 仓位 / 订单接口
是否使用 Binance server time 判断已收盘
是否没有保存 Binance 原始 closeTime
是否 close_time_utc = open_time_utc + timeframe
是否实现短时同步 retry
是否禁止长延迟异步 retry
是否 retry 完成前阻断下游流程
是否实现 DataCollectionRun
是否实现同业务键 running / retrying 并发控制
是否实现 lookback window 补漏
是否实现代码层去重
是否没有依赖数据库唯一约束作为正常去重
是否已存在且一致则跳过
是否已存在但不一致则 conflict 且不覆盖
是否失败 / blocked 时按规则写 AlertEvent
是否不直接调用 Hermes client
是否测试不请求真实 Binance
```

建议搜索：

```bash
grep -R "websocket\|WebSocket" apps tests
grep -R "DataQualityResult\|DataQualityIssue" apps tests
grep -R "Backfill\|BackfillRun\|BackfillResult" apps tests
grep -R "MarketSnapshot" apps tests
grep -R "StrategySignal\|DecisionSnapshot\|CandidateOrderIntent / ApprovedOrderIntent\|Execution\|RiskCheck" apps tests
grep -R "account\|position\|leverage\|order" apps tests
grep -R "HermesClient\|send_webhook" apps tests
grep -R "update_or_create" apps/market_data tests/market_data
grep -R "ON DUPLICATE\|CREATE TABLE\|pymysql.connect\|mysql.connector" apps tests
```

命中不一定都是错误，但 Codex 必须解释原因。

---

## 28. Codex 输出总结要求

Codex 完成后必须输出：

```text
修改 / 新增文件清单
新增 model 名称
新增 migration 路径
REST client 文件路径
validators / domain_rules 文件路径
collection service 文件路径
management command 文件路径
是否实现 Celery task
DataCollectionRun 字段说明
retry 策略说明
并发控制说明
lookback window 补漏说明
代码层去重说明
幂等写入说明
冲突处理说明
AlertEvent 使用说明
测试命令和结果
无法运行的命令及原因
是否存在未完成项
```

并明确说明：

```text
本阶段未实现 WebSocket。
本阶段未实现完整 data_quality。
本阶段未实现 data_backfill。
本阶段未实现 MarketSnapshot。
本阶段未实现策略、风控、交易。
本阶段未使用 WebSocket 参与 K 线采集。
```

---

## 29. 最终验收标准

本阶段完成后，必须满足：

```text
Binance REST client 存在。
get_server_time 存在。
get_klines 存在。
DataCollectionRun model 存在。
KlineWriteLock model 存在。
DataCollectionRun / KlineWriteLock migration 存在。
collection service 存在。
management command 存在。
基础 validators / domain_rules 存在。
使用 Binance server time 判断已收盘。
close_time_utc = open_time_utc + timeframe。
不保存 Binance 原始 closeTime。
lookback window 可补漏。
代码层按业务键去重。
已存在且一致则跳过。
已存在但不一致则 conflict，不覆盖。
短时同步 retry 存在。
retry 不跨 analysis cycle。
retry 完成前下游流程不得继续。
同 symbol/timeframe running/retrying 并发被阻断。
KlineWriteLock 保护 Kline4h / Kline1d 主事实表写入。
失败 / blocked 时可写 AlertEvent。
不直接调用 Hermes client。
测试通过。
默认测试不访问真实外部服务。
未实现 WebSocket。
未实现完整 data_quality。
未实现 data_backfill。
未实现 MarketSnapshot。
未实现策略、风控、交易。
```

---

## 30. 下一阶段

本阶段完成并验收后，下一阶段才开始：

```text
002C_data_quality_plan.md
```

002C 才允许实现：

```text
DataQualityResult
DataQualityIssue
Kline4h / Kline1d 连续性检查
更大窗口缺失检查
OHLCV 质量检查
新鲜度检查
质量 PASS / FAIL
FAIL 写 AlertEvent
可回补问题创建 BackfillRequest
阻断 MarketSnapshot / 策略 / 交易
```

002C 不应重新复制基础 K 线规则，应优先复用 002B 中沉淀在 `apps/market_data/validators.py` 或 `domain_rules.py` 的无副作用基础规则函数。
