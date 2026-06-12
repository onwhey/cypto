# Plan 002C：K 线数据质量检查与 BackfillRequest 创建边界

## 0. 执行边界

本 plan 是 002B 之后的数据质量阶段。

本阶段只负责检查已经入库的 `Kline4h` / `Kline1d` 是否满足进入后续分析链路的最低质量要求，并记录检查结果、问题明细、告警事件和可回补问题的回补请求。

本阶段不负责采集 K 线，不负责请求 Binance，不负责修复 K 线，不负责执行回补，不负责生成 MarketSnapshot，不负责策略、风控或交易。

本阶段必须同时支持：

```text
正常分析 / 策略流程前的数据质量检查
每日定时数据质量报告
手动指定参数的数据质量复核
```

---

## 1. 阶段目标

本阶段完成后，项目应具备：

```text
data_quality app
DataQualityResult model
DataQualityIssue model
apps.data_backfill.BackfillRequest 最小请求模型
DataQualityService
无副作用的 K 线质量规则复用
每日定时质量报告 task
手动 run_data_quality management command
AlertEvent 写入
Django migrations
tests
implementation 文档
```

本阶段的核心目标是：

```text
发现 K 线质量问题
记录 PASS / FAIL 结果
记录问题明细
明确是否允许下游继续
对可回补问题生成回补请求
FAIL、阻断、系统异常或需要通知的异常事件通过 AlertEvent 报告
```

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
docs/plans/002C_data_quality_plan.md
```

如果本 plan 与 `AGENTS.md` 或 `docs/rules/project_invariants.md` 冲突，必须以后两者为准，并在总结中说明冲突。

如果本 plan 与 `docs/requirements/data_quality.md` 或 `docs/architecture/data_flow.md` 冲突，必须停止并提示用户确认。

---

## 3. 本阶段明确不做

本阶段禁止实现：

```text
Binance REST client
Binance serverTime 请求
Binance Kline 请求
WebSocket
data_collection service
lookback window 采集
历史数据采集
直接写 Kline4h / Kline1d
修改 Kline4h / Kline1d
删除 Kline4h / Kline1d
覆盖 OHLCV
修复 K 线
data_backfill 执行 service
BackfillResult
BackfillEvent
实际回补写入
MarketSnapshot model
MarketSnapshot service
FeatureLayer
AtomicSignal
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
直接调用 Hermes webhook
```

本阶段禁止外部网络请求。

本阶段测试不得访问真实 Binance、真实 Hermes、生产 MySQL、生产 Redis。

---

## 4. 与 002B 的关系：必须复用 K 线基础规则

002B 已经在 `apps/market_data/validators.py` 或等价位置实现了无副作用的 K 线基础规则。

002C 必须复用这些规则，不得重新复制一套独立校验逻辑。

允许在 `apps/market_data/validators.py` 中补充新的无副作用函数，例如：

```text
build_expected_open_times
find_missing_open_times
validate_time_boundary
validate_ohlcv_values
detect_non_continuous_open_times
compare_kline_values
```

这些函数必须保持无副作用：

```text
不请求 Binance
不写数据库
不写 AlertEvent
不创建 DataQualityResult
不创建 DataQualityIssue
不创建 BackfillRequest
不调用 data_backfill
不生成 MarketSnapshot
不调用策略 / 风控 / 交易
```

002C 的 `DataQualityService` 可以调用这些函数完成质量检查。

---

## 5. data_quality 的使用场景

本阶段的数据质量能力有三个使用场景。

### 5.1 正常分析 / 策略流程前检查

后续完整链路应为：

```text
data_collection
→ data_quality
→ MarketSnapshot
→ FeatureLayer / Signal
→ Strategy
→ Risk
→ Execution
```

本阶段只实现 `DataQualityService` 和质量结果，不实现完整 `RunAnalysisCycleService` / `MarketSnapshot`。

规则：

```text
DataQualityResult = PASS
→ allows_downstream = True
→ 后续编排层可以继续 MarketSnapshot

DataQualityResult = FAIL
→ allows_downstream = False
→ 写 DataQualityIssue
→ 写 AlertEvent
→ 如属于可回补问题，创建 BackfillRequest
→ 阻断当前 analysis cycle
→ 不生成 MarketSnapshot
→ 不进入策略 / 风控 / 交易
```

注意：

```text
阻断当前 analysis cycle 不等于禁止补偿链路。
FAIL 且属于可回补问题时，可以创建 BackfillRequest；当前主链路必须停止。
回补后必须重新运行 data_quality，复检 PASS 后才允许进入 MarketSnapshot。
```

### 5.2 每日定时质量报告

每日任务用于日常健康巡检。

默认检查：

```text
Kline4h 最近 N 根
Kline1d 最近 N 根
```

其中 N 默认值来自 settings，默认建议为：

```text
DATA_QUALITY_DAILY_LOOKBACK_LIMIT = 50
```

DataQuality PASS 只写 DataQualityResult = PASS 且 allows_downstream=True，不写 AlertEvent，不创建 BackfillRequest。

DataQualityResult 本身就是 PASS 的审计记录。

### 5.3 手动质量复核

必须提供 management command，例如：

```bash
python manage.py run_data_quality --timeframe 4h --limit 50
python manage.py run_data_quality --timeframe 4h --limit 100
python manage.py run_data_quality --timeframe 4h --limit 1000
python manage.py run_data_quality --timeframe 1d --limit 1000
python manage.py run_data_quality --timeframe all --limit 1000
```

手动复核用于检查更长历史窗口。

本阶段至少必须支持 `latest_n` 模式，即按最近 N 根 K 线进行检查。

如实现成本可控，允许额外支持指定时间区间：

```bash
python manage.py run_data_quality --timeframe 4h --start-time 2025-01-01T00:00:00Z --end-time 2026-01-01T00:00:00Z
```

但不得为了区间模式实现采集、回补或外部请求。

---

## 6. 参数化要求

本阶段不得把 `50` 写死在 service 逻辑中。

必须支持：

```text
lookback_limit 参数
settings 默认值
settings 最大上限
management command --limit
Celery task 使用 settings 默认值
```

建议 settings：

```text
DATA_QUALITY_DAILY_LOOKBACK_LIMIT = 50
DATA_QUALITY_DEFAULT_LOOKBACK_LIMIT = 50
DATA_QUALITY_MAX_LOOKBACK_LIMIT = 5000
```

规则：

```text
limit 必须为正整数
limit 默认 50
limit 不得超过 DATA_QUALITY_MAX_LOOKBACK_LIMIT
超过上限必须拒绝并返回受控错误
```

`limit=1000` 只表示检查数据库中的最近 1000 根 K 线，不表示请求 Binance，也不表示执行回补。

---

## 7. 建议 app 与文件结构

建议新增：

```text
apps/data_quality/
apps/data_quality/__init__.py
apps/data_quality/apps.py
apps/data_quality/constants.py
apps/data_quality/models.py
apps/data_quality/services.py
apps/data_quality/tasks.py
apps/data_quality/management/commands/run_data_quality.py
apps/data_quality/migrations/
tests/data_quality/test_data_quality_models.py
tests/data_quality/test_data_quality_service.py
tests/data_quality/test_data_quality_tasks.py
tests/data_quality/test_run_data_quality_command.py
docs/implementation/002C_data_quality.md
```

如果为了满足可回补问题的请求记录，需要提前新增最小回补请求模型，只允许创建 `apps.data_backfill.BackfillRequest`。

### 方案：创建最小 `apps/data_backfill` app

```text
apps/data_backfill/
apps/data_backfill/__init__.py
apps/data_backfill/apps.py
apps/data_backfill/constants.py
apps/data_backfill/models.py
apps/data_backfill/migrations/
```

本阶段只允许实现 `BackfillRequest` 模型和常量。

禁止实现 data_backfill 执行 service / task / client。

不得在 `apps.data_quality` 中创建第二套 BackfillRequest 或等价回补请求模型。

`apps.data_backfill` 是 BackfillRequest 的唯一归属 app，但本阶段必须严格限制其范围。

---

## 8. 模型范围

本阶段必须实现：

```text
DataQualityResult
DataQualityIssue
apps.data_backfill.BackfillRequest
```

允许实现抽象基类或枚举常量。

本阶段不实现：

```text
BackfillRun executor
BackfillResult
BackfillEvent
BackfillService
MarketSnapshot
```

---

## 9. DataQualityResult 需求

### 9.1 模型定位

`DataQualityResult` 记录一次质量检查运行的汇总结果。

建议一条 result 对应：

```text
一个 exchange
一个 market_type
一个 symbol
一个 timeframe
一个 check_scope
一次 trigger_source
```

例如：

```text
BTCUSDT 4h latest_n scheduled
BTCUSDT 1d latest_n scheduled
BTCUSDT 4h latest_n analysis_cycle
```

### 9.2 必须字段语义

至少覆盖：

```text
check_type
check_scope
status
allows_downstream
exchange
market_type
symbol
timeframe
lookback_limit
requested_start_time_utc
requested_end_time_utc
actual_start_time_utc
actual_end_time_utc
reference_time_utc
expected_count
actual_count
missing_count
invalid_count
duplicate_count
issue_count
backfill_request_count
summary
details
trace_id
trigger_source
created_at_utc
updated_at_utc
```

字段可按 Django 风格实现，但语义必须完整。

### 9.3 status

第一阶段只允许：

```text
PASS
FAIL
```

不要引入：

```text
MISSING
INCOMPLETE
FATAL
WARNING
PARTIAL
```

细节写入 `DataQualityIssue`。

### 9.4 allows_downstream

规则必须固定：

```text
PASS → allows_downstream=True
FAIL → allows_downstream=False
```

不得出现：

```text
FAIL 但 allows_downstream=True
PASS 但 allows_downstream=False
```

除非未来有正式 plan 修改该规则。

---

## 10. DataQualityIssue 需求

### 10.1 模型定位

`DataQualityIssue` 记录一次质量检查发现的具体问题。

一个 `DataQualityResult` 可以关联多个 `DataQualityIssue`。

### 10.2 issue_type

至少支持：

```text
EMPTY_KLINE_SET
INSUFFICIENT_KLINE_COUNT
MISSING_KLINE
NON_CONTINUOUS_KLINE
LATEST_KLINE_DELAYED
DUPLICATE_KLINE
INVALID_OHLC
INVALID_VOLUME
INVALID_TIME_BOUNDARY
DATA_CONFLICT
DATABASE_INCONSISTENCY
```

### 10.3 可回补问题

以下问题属于可明确回补问题：

```text
MISSING_KLINE
NON_CONTINUOUS_KLINE
LATEST_KLINE_DELAYED
EMPTY_KLINE_SET
INSUFFICIENT_KLINE_COUNT
```

这些 issue 必须：

```text
is_backfillable=True
backfill_status=pending 或 requested
保存 missing_open_times / window_start / window_end / timeframe / symbol 等上下文
创建 BackfillRequest
```

### 10.4 不应自动回补的问题

以下问题不得自动触发回补执行：

```text
DUPLICATE_KLINE
DATA_CONFLICT
INVALID_OHLC
INVALID_VOLUME
INVALID_TIME_BOUNDARY
DATABASE_INCONSISTENCY
```

这些 issue 可以：

```text
is_backfillable=False
写 AlertEvent
阻断当前分析流程
等待人工复核或后续 conflict_recheck 设计
```

### 10.5 必须字段语义

至少覆盖：

```text
result
issue_type
severity
exchange
market_type
symbol
timeframe
open_time_utc
window_start_time_utc
window_end_time_utc
is_backfillable
backfill_status
backfill_request
details
trace_id
trigger_source
created_at_utc
updated_at_utc
```

---

## 11. BackfillRequest 最小需求

### 11.1 模型定位

`BackfillRequest` 是回补请求记录，不是回补执行器。

它表示：

```text
data_quality 已经发现明确可回补问题
后续 data_backfill 阶段可以读取该请求并执行回补
```

本阶段只创建请求，不执行请求。

### 11.2 必须字段语义

至少覆盖：

```text
source
source_result
source_issue
status
exchange
market_type
symbol
timeframe
requested_start_time_utc
requested_end_time_utc
missing_open_times
reason_code
idempotency_key
trace_id
trigger_source
created_at_utc
updated_at_utc
```

### 11.3 status

本阶段至少支持：

```text
pending
cancelled
```

如需更多状态，最多允许：

```text
pending
cancelled
```

不要提前实现：

```text
running
success
failed
retrying
```

这些属于后续 data_backfill 执行阶段。

### 11.4 幂等要求

创建 BackfillRequest 必须幂等。

建议使用唯一：

```text
idempotency_key
```

同一类质量问题重复检查时，不得无限创建重复 BackfillRequest。

建议 key 语义包含：

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

如果重复触发，应复用已有 pending BackfillRequest，或者记录已存在，不创建重复请求。

---

## 12. 检查规则

本阶段至少检查：

```text
是否有 K 线
是否满足 requested lookback_limit
open_time_utc 是否连续
是否缺失 expected open_time
close_time_utc 是否等于 open_time_utc + timeframe
OHLCV 是否合法
volume / quote_volume / trade_count 是否合法
是否存在重复 open_time 语义问题
最新 K 线是否延迟
```

### 12.1 latest_n 检查

`latest_n` 模式应：

```text
读取最近 N 根 Kline4h 或 Kline1d
按 open_time_utc 升序检查
根据 timeframe 生成 expected open_time 序列
识别缺失和不连续
识别最新 K 线是否延迟
```

如果数据库不足 N 根，应写 `INSUFFICIENT_KLINE_COUNT`。

如果数据库为空，应写 `EMPTY_KLINE_SET`。

### 12.2 reference_time_utc

data_quality 不请求 Binance。

`reference_time_utc` 的来源规则：

```text
analysis_cycle 调用时，可以由上游传入 DataCollectionRun.server_time_utc 或等价可信时间
scheduled / manual 调用时，可以使用 Django timezone.now() 的 UTC aware datetime
```

不得在 data_quality 内部调用 Binance server time。

### 12.3 freshness / latest delayed

根据 `reference_time_utc` 和 timeframe 计算理论上应该存在的最新已收盘 K 线。

如果数据库最新 open_time 早于应存在 open_time，应产生：

```text
LATEST_KLINE_DELAYED
```

该问题属于可回补问题。

---

## 13. AlertEvent 要求

本阶段可以写 `AlertEvent`，但不得直接调用 Hermes client，不得直接调用 notifications service。

正确链路：

```text
data_quality
→ 写 AlertEvent
→ notifications service 异步消费
→ Hermes client 发送
```

### 13.1 analysis_cycle 场景

正常分析流程前检查：

```text
PASS：不写 AlertEvent
FAIL：必须写 AlertEvent
```

### 13.2 daily scheduled 场景

每日定时质量报告：

```text
PASS：不写 AlertEvent
FAIL：必须写 AlertEvent
```

每日任务 FAIL 时建议写聚合 AlertEvent，避免 4h / 1d 每个 issue 都产生大量通知。

### 13.3 manual 场景

手动命令 PASS 不写 AlertEvent，FAIL 或系统异常时写 AlertEvent。

如果实现 `--no-alert`，必须在 plan 总结中说明；默认不得静默。

### 13.4 event_type 建议

建议区分：

```text
data_quality_analysis_failed
data_quality_daily_fail
data_quality_manual_fail
```

具体字段必须适配现有 `AlertEvent` model。

---

## 14. Celery task / Celery Beat 要求

本阶段必须新增每日定时任务。

建议任务：

```text
run_daily_data_quality_report
```

默认每日执行时间建议：

```text
00:20 UTC
```

任务行为：

```text
读取 settings.DATA_QUALITY_DAILY_LOOKBACK_LIMIT
检查 Kline4h
检查 Kline1d
PASS 不写 AlertEvent；FAIL 或系统异常时写 AlertEvent
不得请求 Binance
不得写 Kline
不得执行回补
```

如果当前项目尚未配置 Celery Beat schedule，可先在 settings 中添加可读配置或 task 函数，并在 implementation 文档中说明如何接入。

不得自研调度器。

禁止：

```text
schedule.every
threading.Timer
while True sleep
系统后台常驻线程
```

---

## 15. Management command 要求

新增：

```text
python manage.py run_data_quality
```

至少支持参数：

```text
--timeframe 4h|1d|all
--limit <positive_int>
--trigger-source manual|scheduled|analysis_cycle
```

可选支持：

```text
--start-time
--end-time
--no-alert
```

要求：

```text
command 只解析参数和调用 service
不承载核心业务逻辑
输出 PASS / FAIL 摘要
输出 result id
输出 issue count
输出 backfill request count
```

---

## 16. 数据库与 migration 要求

必须使用 Django migrations。

必须新增 migration。

如果新增 `apps/data_quality`，必须在 settings 注册 app。

如果新增最小 `apps/data_backfill`，必须在 settings 注册 app。

新增 model 后必须生成 migration 文件。

Codex 完成后必须运行：

```bash
python manage.py makemigrations
python manage.py makemigrations --check --dry-run
python manage.py migrate --plan
python manage.py check
python manage.py test
pytest
```

如果新增 model 但没有 migration，必须停止并修复。

---

## 17. 测试要求

必须测试：

```text
DataQualityResult 可创建
DataQualityIssue 可创建
BackfillRequest 可创建
PASS 时 allows_downstream=True
FAIL 时 allows_downstream=False
latest_n 支持 limit=50
latest_n 支持 limit=100
latest_n 支持 limit=1000 或不超过 settings 最大值的较大 limit
limit 超过最大值会被拒绝
空 K 线表产生 EMPTY_KLINE_SET
不足 N 根产生 INSUFFICIENT_KLINE_COUNT
缺失 open_time 产生 MISSING_KLINE 或 NON_CONTINUOUS_KLINE
close_time_utc != open_time_utc + timeframe 产生 INVALID_TIME_BOUNDARY
非法 OHLCV 产生 INVALID_OHLC / INVALID_VOLUME
最新 K 线延迟产生 LATEST_KLINE_DELAYED
可回补问题创建 BackfillRequest
重复可回补问题不会无限创建重复 BackfillRequest
不可回补问题不创建 BackfillRequest
analysis_cycle FAIL 写 AlertEvent
daily FAIL 写 AlertEvent
manual command 可运行
daily task 可运行
002C 复用 market_data validators
```

测试禁止：

```text
真实请求 Binance
真实请求 Hermes
连接生产 MySQL
连接生产 Redis
执行 data_backfill
生成 MarketSnapshot
进入策略 / 风控 / 交易
```

---

## 18. 越界检查

Codex 完成后建议运行：

```bash
grep -R "requests.get" apps/data_quality tests/data_quality
grep -R "httpx" apps/data_quality tests/data_quality
grep -R "Binance" apps/data_quality tests/data_quality
grep -R "Kline4h.objects.create" apps/data_quality tests/data_quality
grep -R "Kline1d.objects.create" apps/data_quality tests/data_quality
grep -R "MarketSnapshot" apps/data_quality tests/data_quality
grep -R "CandidateOrderIntent / ApprovedOrderIntent" apps/data_quality tests/data_quality
grep -R "Execution" apps/data_quality tests/data_quality
grep -R "WebSocket" apps/data_quality tests/data_quality
```

如果命中，Codex 必须解释原因。

注意：测试可以创建 Kline4h / Kline1d 作为测试数据，但 production data_quality service 不得写 Kline 表。

---

## 19. implementation 文档要求

完成后必须新增：

```text
docs/implementation/002C_data_quality.md
```

内容至少包括：

```text
本阶段新增了哪些 app / model
DataQualityResult 字段说明
DataQualityIssue 字段说明
BackfillRequest 字段说明
检查规则说明
latest_n 参数说明
daily scheduled report 说明
manual command 说明
AlertEvent 写入规则
BackfillRequest 创建规则
PASS / FAIL 与 allows_downstream 规则
与 002B validators 的复用关系
测试方式
本阶段明确未实现内容
```

必须明确写入：

```text
本阶段未实现 Binance REST client。
本阶段未实现 data_collection service。
本阶段未执行 data_backfill。
本阶段未修复 Kline4h / Kline1d。
本阶段未实现 MarketSnapshot。
本阶段未实现 WebSocket。
本阶段未实现策略、风控、交易。
```

---

## 20. 人工审查清单

用户合并前应检查：

```text
是否新增 data_quality app
是否新增 DataQualityResult / DataQualityIssue
是否新增 apps.data_backfill.BackfillRequest
是否没有请求 Binance
是否没有写 Kline4h / Kline1d 的生产逻辑
是否没有执行 data_backfill
是否没有 MarketSnapshot
是否没有 WebSocket
是否没有策略 / 风控 / 交易
是否复用 002B validators
是否 latest_n 的 limit 参数化
是否 DataQuality PASS 不写 AlertEvent
是否 analysis_cycle FAIL 会 allows_downstream=False
是否可回补问题会创建 BackfillRequest
是否 BackfillRequest 幂等
是否测试通过
```

---

## 21. Codex 输出总结要求

Codex 完成后必须输出：

```text
修改 / 新增文件清单
新增 app 名称
新增 model 名称
是否新增 migrations
DataQualityResult / DataQualityIssue 字段说明
BackfillRequest 字段说明
检查规则说明
AlertEvent 写入规则
BackfillRequest 幂等规则
测试命令和结果
无法运行的命令及原因
是否存在未完成项
```

并明确说明：

```text
本阶段未请求 Binance。
本阶段未执行 data_backfill。
本阶段未修改 Kline4h / Kline1d。
本阶段未生成 MarketSnapshot。
本阶段未实现 WebSocket。
本阶段未实现策略、风控、交易。
```

---

## 22. 最终验收标准

本阶段完成后，必须满足：

```text
data_quality app 可加载。
DataQualityResult model 存在。
DataQualityIssue model 存在。
apps.data_backfill.BackfillRequest 存在。
latest_n 支持参数化 limit。
每日任务默认检查最近 50 根，但 50 不写死。
手动命令可查 50 / 100 / 1000。
PASS / FAIL 结果清晰。
FAIL 时 allows_downstream=False。
DataQuality PASS 不写 AlertEvent。
AlertEvent 仅用于 FAIL、阻断、系统异常或需要通知的异常事件。
analysis_cycle FAIL 写 AlertEvent。
可回补问题创建 BackfillRequest。
BackfillRequest 创建幂等。
不请求 Binance。
不写 Kline4h / Kline1d 生产逻辑。
不执行 data_backfill。
不生成 MarketSnapshot。
不实现 WebSocket。
不实现策略、风控、交易。
测试通过。
```

---

## 23. 下一阶段

本阶段完成并验收后，下一阶段才开始：

```text
002D_data_backfill_plan.md
```

002D 才允许实现：

```text
读取 BackfillRequest
Binance REST 回补
明确缺口补齐
冲突复核
BackfillRun / BackfillResult
回补后标记 / 要求 DataQuality 复检，实际复检由后续统一编排层、recovery scan 或人工命令重新执行
```
