# Plan 003A：MarketSnapshot 模型与快照契约

## 0. 执行边界

本 plan 是 MarketSnapshot 的第一阶段。

本阶段只负责建立 MarketSnapshot 的数据库模型、状态语义、字段契约、唯一约束、基础索引、测试和 implementation 文档。

本阶段不负责生成 MarketSnapshot，不读取 Kline4h / Kline1d 窗口，不调用 DataQualityService，不写 AlertEvent，不接入 scheduler，不实现 FeatureLayer，不实现策略、风控或交易。

本阶段的目标是先把“快照应该存什么、不应该存什么、如何追溯、如何幂等”定义清楚。

---

## 1. 阶段目标

本阶段完成后，项目应具备：

```text
market_snapshot app
MarketSnapshot model
MarketSnapshot 状态常量 / 枚举
payload schema version 常量
UTC 时间字段规范
DataQualityResult 关联字段
4h / 1d 窗口索引字段
market_as_of_utc 字段
snapshot_key 幂等键
created / blocked / failed 状态语义
is_usable 语义
Django migrations
model tests
implementation 文档
```

本阶段只定义事实快照表结构，不实现快照生成业务。

---

## 2. 依赖文档

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
docs/implementation/002A_market_data_models.md
docs/implementation/002B_data_collection_gateway_and_service.md
docs/implementation/002C_data_quality.md
docs/implementation/002D_data_backfill.md
docs/plans/003A_market_snapshot_model_plan.md
```

如果本 plan 与 `AGENTS.md` 或 `docs/rules/project_invariants.md` 冲突，必须以后两者为准，并在总结中说明冲突。

如果本 plan 与 requirements 或 architecture 冲突，必须停止并提示用户确认。

---

## 3. 本阶段明确不做

本阶段禁止实现：

```text
MarketSnapshot 生成 service
MarketSnapshot builder
读取 Kline4h / Kline1d 窗口
读取 DataQualityResult 选择逻辑
DataQualityResult PASS 校验逻辑
Kline 窗口覆盖校验
KlineWriteLock 读取逻辑
AlertEvent 写入
Hermes 通知
management command
Celery task
Celery Beat schedule
RunAnalysisCycleService
MarketSnapshot scheduler
FeatureLayer
AtomicSignal
DomainSignal
MarketRegime
StrategySignal
DecisionSnapshot
RiskCheck

CandidateOrderIntent / ApprovedOrderIntent
Execution
账户接口
仓位接口
订单接口
自动交易
回测
大模型分析
Binance REST 请求
Binance WebSocket
WebSocket 实时行情
```

本阶段禁止外部网络请求。

本阶段测试不得访问真实 Binance、真实 Hermes、生产 MySQL、生产 Redis。

---

## 4. 命名原则

当前项目统一使用：

```text
MarketSnapshot
```

不得使用容易混淆的名称：

```text
MarketContextSnapshot
```

除非用户明确要求全项目改名。

建议 app：

```text
apps/market_snapshot/
```

建议文件：

```text
apps/market_snapshot/__init__.py
apps/market_snapshot/apps.py
apps/market_snapshot/constants.py
apps/market_snapshot/models.py
apps/market_snapshot/migrations/
tests/market_snapshot/test_market_snapshot_model.py
docs/implementation/003A_market_snapshot_model.md
```

---

## 5. MarketSnapshot 的定位

MarketSnapshot 是行情事实快照，不是策略信号。

它的作用是：

```text
记录某一次后续分析所使用的 4h + 1d 市场事实窗口
绑定当时的数据质量检查结果
提供后续 FeatureLayer / Strategy / 复盘的统一事实输入边界
```

它不是第二份 K 线库。

它不得保存完整 K 线数组。

它不得包含策略结论。

---

## 6. 快照事实层边界

MarketSnapshot 允许描述：

```text
exchange
market_type
symbol
base_timeframe = 4h
higher_timeframe = 1d
4h 窗口起止 open_time_utc
1d 窗口起止 open_time_utc
4h / 1d latest open_time_utc
4h / 1d actual_count
4h / 1d lookback_count
关联的 DataQualityResult
market_as_of_utc
payload schema version
payload 元数据摘要
trace_id
trigger_source
```

MarketSnapshot 禁止描述：

```text
trend=bullish / bearish
signal=long / short
entry_price
stop_loss
take_profit
position_size
leverage
should_trade
stop_trading
strategy verdict
risk verdict
LLM conclusion
交易建议
```

复杂指标也不属于本阶段，例如：

```text
MA20
EMA
ATR
RSI
支撑压力
波动率 regime
趋势判断
```

这些属于后续 FeatureLayer / Signal 阶段。

---

## 7. 模型范围

本阶段只实现：

```text
MarketSnapshot
```

允许实现常量 / 枚举：

```text
MarketSnapshotStatus
MarketSnapshotSchemaVersion
MarketSnapshotTimeframeRole
```

本阶段不实现：

```text
MarketSnapshotService
MarketSnapshotBuilder
MarketSnapshotRepository
MarketSnapshotAlertService
MarketSnapshot task
MarketSnapshot command
```

如需要 repository / service，必须留到 003B。

---

## 8. MarketSnapshot 字段需求

### 8.1 身份与幂等字段

必须包含：

```text
snapshot_key
status
is_usable
```

`snapshot_key` 是幂等业务键，必须唯一。

建议 `snapshot_key` 语义包含：

```text
exchange
market_type
symbol
base_timeframe
higher_timeframe
latest_4h_open_time_utc
latest_1d_open_time_utc
lookback_4h_count
lookback_1d_count
quality_result_4h_id
quality_result_1d_id
```

003A 不实现生成逻辑，但必须为该 key 预留字段和唯一约束。

### 8.2 状态字段

状态只允许：

```text
created
blocked
failed
```

语义：

```text
created = 快照成功生成，可作为后续事实输入
blocked = 前置条件不满足，当前主链路阻断
failed  = 快照创建过程异常
```

`is_usable` 规则：

```text
status=created → is_usable=True
status=blocked → is_usable=False
status=failed  → is_usable=False
```

003A 应通过 model constraint 或 service-level 文档明确该规则。

由于当前本地 MySQL 8.0.12 不实际执行 CheckConstraint，测试可检查 model 声明和对象语义，但 implementation 文档必须记录正式环境需要 MySQL 8.0.16+ 才能执行数据库 CHECK。

### 8.3 市场上下文字段

必须包含：

```text
exchange
market_type
symbol
base_timeframe
higher_timeframe
```

第一阶段默认：

```text
exchange = binance
market_type = usds_m_futures
symbol = BTCUSDT
base_timeframe = 4h
higher_timeframe = 1d
```

不得用数据库取值约束锁死这些值。

可以使用 constants，避免字符串散落。

### 8.4 4h 窗口字段

必须包含：

```text
lookback_4h_count
actual_4h_count
start_4h_open_time_utc
end_4h_open_time_utc
latest_4h_open_time_utc
```

字段语义：

```text
lookback_4h_count = 计划窗口数量
actual_4h_count = 实际窗口数量
start_4h_open_time_utc = 4h 窗口第一根 K 线 open_time_utc
end_4h_open_time_utc = 4h 窗口最后一根 K 线 open_time_utc
latest_4h_open_time_utc = 当前快照使用的最新 4h open_time_utc
```

### 8.5 1d 窗口字段

必须包含：

```text
lookback_1d_count
actual_1d_count
start_1d_open_time_utc
end_1d_open_time_utc
latest_1d_open_time_utc
```

字段语义同 4h。

### 8.6 数据质量关联字段

必须关联：

```text
quality_result_4h
quality_result_1d
```

字段应指向：

```text
apps.data_quality.models.DataQualityResult
```

允许 `null=True`，用于 blocked / failed 情况。

但 `created` 快照原则上必须有 4h 与 1d 对应的 PASS DataQualityResult。003A 只定义字段和约束说明，003B 实现生成时强制校验。

建议冗余保存：

```text
quality_status_4h
quality_status_1d
quality_check_type_4h
quality_check_type_1d
```

但不得以这些冗余字段替代外键追溯。

### 8.7 市场截止时间字段

必须包含：

```text
market_as_of_utc
```

含义：

```text
该 snapshot 代表的市场事实截止时间
```

注意区分：

```text
created_at_utc = 系统创建 snapshot 的时间
market_as_of_utc = 这份市场事实的截止边界
```

建议 003B 中使用 latest 4h close boundary 或等价规则填充。

003A 只建立字段。

### 8.8 payload 字段

必须包含：

```text
payload_schema_version
payload
```

建议默认：

```text
payload_schema_version = market_snapshot.v1
```

`payload` 使用 JSONField。

payload 只能保存摘要和元数据，不得保存完整 K 线数组或 OHLCV 明细数组。

payload 必须支持 schema version。

### 8.9 blocked / failed 字段

必须包含：

```text
blocked_reason
error_code
error_message
```

要求：

```text
created 状态可以为空
blocked 状态应记录 blocked_reason
failed 状态应记录 error_code / error_message
```

003A 不实现 blocked / failed 生成逻辑，但字段必须存在。

### 8.10 审计字段

必须包含：

```text
trace_id
trigger_source
created_at_utc
updated_at_utc
```

UTC 语义必须清晰。

---

## 9. 不使用 open_time_ms 作为主业务字段

MarketSnapshot 不使用 open_time_ms 作为主业务字段。
当前项目 Kline 主字段是 open_time_utc / close_time_utc。

字段清单：

```text
open_time_utc
close_time_utc
```

MarketSnapshot 主字段必须使用 UTC datetime：

```text
start_4h_open_time_utc
end_4h_open_time_utc
latest_4h_open_time_utc
start_1d_open_time_utc
end_1d_open_time_utc
latest_1d_open_time_utc
```

不得以 `*_open_time_ms` 作为主字段。

如未来确实需要毫秒时间戳，可由服务层从 UTC datetime 派生，不在 003A 主模型中添加。

---

## 10. payload 契约

003A 只定义 payload 字段和文档契约，不实现 payload 构造。

payload v1 建议结构：

```json
{
  "schema_version": "market_snapshot.v1",
  "exchange": "binance",
  "market_type": "usds_m_futures",
  "symbol": "BTCUSDT",
  "base_timeframe": "4h",
  "higher_timeframe": "1d",
  "market_as_of_utc": "2026-06-06T20:00:00Z",
  "windows": {
    "4h": {
      "lookback_count": 500,
      "actual_count": 500,
      "start_open_time_utc": "...",
      "end_open_time_utc": "...",
      "latest_open_time_utc": "..."
    },
    "1d": {
      "lookback_count": 365,
      "actual_count": 365,
      "start_open_time_utc": "...",
      "end_open_time_utc": "...",
      "latest_open_time_utc": "..."
    }
  },
  "quality": {
    "4h_result_id": 1,
    "1d_result_id": 2
  },
  "boundary": {
    "fact_snapshot_only": true,
    "no_strategy_signal": true,
    "no_trading_advice": true,
    "no_full_kline_array": true
  }
}
```

禁止 payload 包含完整 K 线数组，例如：

```text
open
high
low
close
volume
quote_volume
trade_count
```

禁止 payload 包含策略字段，例如：

```text
trend
signal
entry_price
stop_loss
take_profit
position_size
leverage
should_trade
```

---

## 11. 默认配置

本阶段可以新增 settings 默认值，但不实现使用逻辑。

建议：

```text
MARKET_SNAPSHOT_4H_LOOKBACK_COUNT = 500
MARKET_SNAPSHOT_1D_LOOKBACK_COUNT = 365
MARKET_SNAPSHOT_PAYLOAD_SCHEMA_VERSION = market_snapshot.v1
```

这些值不得散落在业务代码中。

如新增 settings，也应更新 `.env.example`。

---

## 12. 唯一约束与索引

### 12.1 唯一约束

必须有：

```text
snapshot_key unique
```

### 12.2 基础索引

至少支持：

```text
exchange + market_type + symbol + base_timeframe + higher_timeframe + market_as_of_utc
status + is_usable + created_at_utc
trace_id
latest_4h_open_time_utc
latest_1d_open_time_utc
quality_result_4h
quality_result_1d
```

索引字段长度必须兼容 MySQL。

长字符串字段不得随意加索引。

`snapshot_key` 如果使用 hash，建议 max_length=64。

如果使用可读字符串，应确保唯一索引不会超过当前 MySQL key length 限制。

优先建议：

```text
snapshot_key = sha256 hex digest, max_length=64, unique=True
```

可读信息放入 payload 或其他字段。

---

## 13. 与 data_quality 的关系

MarketSnapshot 依赖 DataQualityResult，但 003A 不实现读取和判断逻辑。

003A 必须在模型中建立：

```text
quality_result_4h
quality_result_1d
```

003B 才实现：

```text
读取 DataQualityResult
确认 4h 和 1d 都 PASS
确认 quality result 覆盖 snapshot 需要的窗口
FAIL / 缺失 / 覆盖不足时 blocked
```

003A 不得重新实现 data_quality。

---

## 14. 与 data_backfill 的关系

003A 不读取 BackfillRequest / BackfillRun。

003B 生成快照时可以检查是否存在 active collection/backfill 或 KlineWriteLock。

本阶段只在文档中说明：

```text
MarketSnapshot 不执行回补
MarketSnapshot 不创建 BackfillRequest
MarketSnapshot 不修复 Kline
MarketSnapshot 不等待 backfill 完成
```

---

## 15. 与 AlertEvent / Hermes 的关系

003A 不写 AlertEvent。

003B 可以在 blocked / failed 场景写 AlertEvent。

任何阶段都不得直接调用 Hermes client。

正确链路：

```text
market_snapshot
→ 写 AlertEvent
→ notifications service 异步消费
→ Hermes client 发送
```

本阶段只定义模型，不实现通知。

---

## 16. Migration 要求

必须使用 Django migrations。

Codex 完成后必须运行：

```bash
python manage.py makemigrations
python manage.py makemigrations --check --dry-run
python manage.py migrate --plan
python manage.py migrate
python manage.py check
python manage.py test
pytest
```

如果 migration 失败，必须修复，不得 fake migration。

注意 MySQL key length 限制：

```text
unique / index 的 CharField 不得过长
hash key 建议 max_length=64
```

---

## 17. 测试要求

必须测试：

```text
MarketSnapshot 可创建
snapshot_key 唯一约束生效
status=created 时 is_usable=True
status=blocked 时 is_usable=False
status=failed 时 is_usable=False
可关联 4h DataQualityResult
可关联 1d DataQualityResult
payload_schema_version 默认值正确
payload 可保存摘要 JSON
payload 不应在测试样例中包含完整 K 线数组
4h / 1d UTC 窗口字段可保存
market_as_of_utc 可保存
trace_id / trigger_source 可保存
```

必须测试字段不存在或未实现：

```text
不得出现 strategy_signal 字段
不得出现 entry_price 字段
不得出现 stop_loss 字段
不得出现 take_profit 字段
不得出现 leverage 字段
不得出现 position_size 字段
```

测试禁止：

```text
请求 Binance
请求 Hermes
调用 DataQualityService
调用 BackfillService
生成 MarketSnapshot service
生成 FeatureLayer
进入策略 / 风控 / 交易
```

---

## 18. 越界检查

Codex 完成后建议运行：

```bash
grep -R "requests.get" apps/market_snapshot tests/market_snapshot
grep -R "httpx" apps/market_snapshot tests/market_snapshot
grep -R "BinanceMarketDataClient" apps/market_snapshot tests/market_snapshot
grep -R "DataQualityService" apps/market_snapshot tests/market_snapshot
grep -R "BackfillService" apps/market_snapshot tests/market_snapshot
grep -R "MarketSnapshotService" apps/market_snapshot tests/market_snapshot
grep -R "FeatureLayer" apps/market_snapshot tests/market_snapshot
grep -R "StrategySignal" apps/market_snapshot tests/market_snapshot
grep -R "CandidateOrderIntent / ApprovedOrderIntent" apps/market_snapshot tests/market_snapshot
grep -R "Execution" apps/market_snapshot tests/market_snapshot
```

如果命中，Codex 必须解释原因。

---

## 19. implementation 文档要求

完成后必须新增：

```text
docs/implementation/003A_market_snapshot_model.md
```

内容至少包括：

```text
本阶段新增了哪些 app / model
MarketSnapshot 定位
MarketSnapshot 字段说明
snapshot_key 幂等说明
created / blocked / failed 状态说明
is_usable 说明
DataQualityResult 关联说明
4h / 1d 窗口索引说明
market_as_of_utc 说明
payload_schema_version 说明
payload 禁止保存完整 K 线数组说明
唯一约束和索引说明
测试方式
本阶段明确未实现内容
```

必须明确写入：

```text
本阶段未实现 MarketSnapshot 生成 service。
本阶段未读取 Kline4h / Kline1d。
本阶段未调用 DataQualityService。
本阶段未写 AlertEvent。
本阶段未请求 Binance。
本阶段未实现 FeatureLayer。
本阶段未实现策略、风控、交易。
本阶段未实现 WebSocket。
```

---

## 20. 人工审查清单

用户合并前应检查：

```text
是否只新增 market_snapshot 模型层
是否没有 MarketSnapshotService
是否没有 builder / command / task
是否没有请求 Binance
是否没有读取 Kline 窗口
是否没有调用 DataQualityService
是否没有写 AlertEvent
是否没有 FeatureLayer / Strategy / Risk / Execution
是否 snapshot_key 使用安全长度
是否 payload 不保存完整 K 线数组
是否 status / is_usable 语义清晰
是否有 migrations
是否测试通过
```

---

## 21. Codex 输出总结要求

Codex 完成后必须输出：

```text
修改 / 新增文件清单
新增 app 名称
新增 model 名称
migration 文件路径
MarketSnapshot 字段说明
snapshot_key 说明
状态与 is_usable 说明
DataQualityResult 关联说明
payload schema version 说明
唯一约束和索引说明
测试命令和结果
无法运行的命令及原因
是否存在未完成项
```

并明确说明：

```text
本阶段未实现 MarketSnapshot 生成 service。
本阶段未读取 Kline4h / Kline1d。
本阶段未调用 DataQualityService。
本阶段未写 AlertEvent。
本阶段未请求 Binance。
本阶段未实现 FeatureLayer。
本阶段未实现策略、风控、交易。
本阶段未实现 WebSocket。
```

---

## 22. 最终验收标准

本阶段完成后，必须满足：

```text
market_snapshot app 可加载。
MarketSnapshot model 存在。
snapshot_key 唯一。
status 支持 created / blocked / failed。
is_usable 语义明确。
可关联 4h DataQualityResult。
可关联 1d DataQualityResult。
记录 4h / 1d 窗口索引。
记录 market_as_of_utc。
记录 payload_schema_version。
payload 不保存完整 K 线数组。
migrations 存在。
model tests 通过。
未实现 MarketSnapshot 生成 service。
未读取 Kline4h / Kline1d。
未调用 DataQualityService。
未写 AlertEvent。
未请求 Binance。
未实现 FeatureLayer。
未实现策略、风控、交易。
未实现 WebSocket。
```

---

## 23. 下一阶段

本阶段完成并验收后，下一阶段才开始：

```text
003B_market_snapshot_generation_plan.md
```

003B 才允许实现：

```text
MarketSnapshotService
读取 DataQualityResult
确认 4h + 1d PASS
确认 quality result 覆盖 snapshot 窗口
读取 Kline4h / Kline1d 窗口
检查 active KlineWriteLock / collection / backfill 状态
生成 payload
幂等写入 MarketSnapshot
blocked / failed 写 AlertEvent
management command
service tests
```
