# Plan 002D：data_backfill 历史初始化与缺口回补

## 0. 执行边界

本 plan 是 002C 之后的数据回补阶段。

本阶段负责执行 K 线回补，包括：

```text
项目初期历史 K 线初始化拉取
手动指定时间区间回补
消费 data_quality 创建的 BackfillRequest
对明确缺失 open_time 的 K 线进行补齐
记录回补运行结果
BackfillRun 完成后，必须标记 / 要求 DataQuality 复检
只有新的 DataQualityResult = PASS 且 allows_downstream=True，才允许继续 MarketSnapshot
```

本阶段允许请求 Binance REST Kline，但必须复用 002B 已有的 Binance market data gateway。

本阶段允许写入 `Kline4h` / `Kline1d`，但只能以幂等方式插入缺失 K 线。

本阶段禁止静默覆盖、删除或修复已有 K 线。

---

## 1. 阶段目标

本阶段完成后，项目应具备：

```text
data_backfill app
BackfillRun 回补执行审计模型
BackfillService
BackfillRequest 消费能力
历史区间初始化回补能力
指定区间回补能力
明确缺口回补能力
Binance REST 分页拉取能力
K 线基础校验复用
K 线幂等写入复用
冲突检测但不覆盖
Celery task
Celery Beat 兜底扫描任务
management command
tests
implementation 文档
```

核心目标：

```text
可靠地把缺失的已收盘 K 线补入正式 Kline4h / Kline1d 主事实表
不破坏已有历史数据
不制造重复 K 线
不绕过质量检查链路
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
docs/implementation/002C_data_quality.md
docs/plans/002D_data_backfill_plan.md
```

如果本 plan 与 `AGENTS.md` 或 `docs/rules/project_invariants.md` 冲突，必须以后两者为准，并在总结中说明冲突。

如果本 plan 与 requirements 或 architecture 冲突，必须停止并提示用户确认。

---

## 3. 本阶段明确不做

本阶段禁止实现：

```text
WebSocket
MarketSnapshot
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

本阶段禁止：

```text
重新实现 Binance HTTP client
直接使用 requests.get / httpx 访问 Binance
调用 DataCollectionService.collect_lookback 作为回补逻辑
绕过 Kline4h / Kline1d model 写库
手写 SQL 建表
静默覆盖已有 K 线
删除已有 K 线
修改已有 OHLCV
把 WebSocket 用于 K 线回补
```

本阶段测试不得访问真实 Binance、真实 Hermes、生产 MySQL、生产 Redis。

---

## 4. 与 002B 的关系：复用底层能力，不调用业务 service

002B 已经实现：

```text
BinanceMarketDataClient
market_data validators
DataCollectionService
DataCollectionRun
```

002D 必须复用底层能力：

```text
apps/market_data/binance_client.py
apps/market_data/validators.py
```

002D 不得复用 `DataCollectionService.collect_lookback()` 作为回补实现。

原因：

```text
data_collection = 正常短 lookback 采集
data_backfill = 历史初始化 / 明确缺口 / 指定区间回补
```

两者可以共享底层工具，但不能共享业务 service 语义。

正确依赖：

```text
DataCollectionService ┐
                      ├→ BinanceMarketDataClient
BackfillService       ┘

DataCollectionService ┐
                      ├→ market_data.validators
BackfillService       ┘
```

错误依赖：

```text
BackfillService → DataCollectionService.collect_lookback()
```

---

## 5. Kline 幂等写入复用提示

002B 的 DataCollectionService 已经实现了 K 线写入规则：

```text
同批 K 线按 open_time 去重
已存在且一致则跳过
已存在但不一致则 conflict
不存在则插入
不覆盖、不更新、不删除已有 OHLCV
```

002D 需要同一套写入语义。

Codex 应优先考虑做最小抽象，避免在 data_backfill 中复制一份写入逻辑。

建议新增：

```text
apps/market_data/kline_writer.py
```

或等价模块，例如：

```text
apps/market_data/repositories.py
```

提供共享函数或类，例如：

```text
write_klines_idempotently(...)
compare_existing_kline(...)
```

该共享写入能力应负责：

```text
按 exchange + market_type + symbol + open_time_utc 查重
代码层去重
已存在一致跳过
已存在冲突记录 conflict
缺失插入
返回 inserted / skipped / conflict / duplicate 等统计
```

允许对 002B 的 `DataCollectionService` 做最小调整，使其复用该 writer。

限制：

```text
不得改变 002B 对外行为
不得大规模重写 002B
不得改变 Kline4h / Kline1d 表结构
不得为了抽象提前实现策略、风控、MarketSnapshot
```

如果 Codex 判断当前抽象成本过高，必须在总结中说明，但仍不得复制过多复杂逻辑。

---

## 6. 与 002C 的关系：消费 BackfillRequest

002C 已经创建或定义了 `apps.data_backfill.BackfillRequest`。

002D 必须复用现有 `apps.data_backfill.BackfillRequest`，不得重复创建第二套同义模型。

002D 可以在必要时扩展 BackfillRequest 的状态字段或增加执行相关模型，但必须通过 Django migration 完成。

正确链路：

```text
data_quality
→ 创建 BackfillRequest
→ 当前 analysis cycle blocked

data_backfill
→ claim BackfillRequest
→ 执行回补
→ 写 BackfillRun，并记录 BackfillResult 语义
→ 更新 BackfillRequest 状态
→ BackfillRun 完成后，标记 / 要求 DataQuality 复检
→ 只有新的 DataQualityResult = PASS 且 allows_downstream=True，才允许继续 MarketSnapshot
```

data_quality 不应同步执行回补。

---

## 7. data_backfill 调度模型

本阶段 data_backfill 应具备三个入口。

### 7.1 pending BackfillRequest 扫描 / 领取

BackfillRequest 创建后，不要求 data_quality 投递回补 task。

data_backfill 必须提供 pending BackfillRequest 的扫描 / 领取能力。

该能力可以由 Celery Beat、management command、recovery scan 或后续统一编排层触发。

data_backfill 领取 BackfillRequest 后，创建 BackfillRun 并执行回补。

注意：

```text
DataQuality 只创建 BackfillRequest。
DataQuality 不投递 data_backfill task。
DataQuality 不调用 BackfillService。
DataQuality 不启动 data_backfill 调度或回补执行逻辑。
data_backfill claim BackfillRequest 后创建 BackfillRun 并执行回补。
```

### 7.2 Celery Beat 兜底扫描

必须实现周期性 recovery scan：

```text
扫描 pending BackfillRequest
扫描 stale running BackfillRequest
扫描可重试 failed BackfillRequest
```

该任务用于兜底：

```text
Celery 投递失败
worker 当时不可用
进程崩溃
request 长时间 pending
running 卡住
```

建议默认频率：

```text
每 1 分钟或 2 分钟
```

不得自研调度器。

禁止：

```text
schedule.every
threading.Timer
while True sleep
系统后台常驻线程
```

### 7.3 management command 人工触发

必须提供人工入口，用于：

```text
项目初期历史初始化
指定区间回补
处理 pending BackfillRequest
重跑 failed request
```

建议命令：

```bash
python manage.py run_backfill --timeframe 4h --start-time 2024-01-01T00:00:00Z --end-time 2026-01-01T00:00:00Z
python manage.py run_backfill --timeframe 1d --start-time 2024-01-01T00:00:00Z --end-time 2026-01-01T00:00:00Z
python manage.py run_backfill_requests --limit 10
python manage.py run_backfill_request --request-id 123
```

command 只解析参数并调用 service，不承载核心业务逻辑。

---

## 8. 并发与锁

002D 必须防止多个 worker 同时处理同一个 BackfillRequest。

必须实现原子 claim 机制：

```text
pending
→ claim 成 running
→ 执行回补
→ success / failed / conflict / cancelled
```

如果另一个 worker 抢同一条 request：

```text
发现已经 running 或已终态
→ 跳过
```

claim 必须使用数据库事务或等价原子更新。

禁止：

```text
先查 pending
再无锁更新 running
```

如有必要，BackfillRequest 或 BackfillRun 可增加：

```text
locked_by
locked_at_utc
attempt_count
max_attempts
last_error_code
last_error_message
```

stale running 的判断必须有明确超时配置，例如：

```text
DATA_BACKFILL_STALE_RUNNING_SECONDS = 900
```

---

## 9. BackfillRequest 状态

002C 可能只实现了最小状态。

002D 可以扩展 BackfillRequest status 至：

```text
pending
running
success
failed
conflict
cancelled
```

含义：

```text
pending   = 等待执行
running   = 已被 worker claim，正在执行
success   = 回补执行完成且没有阻断性问题
failed    = 请求失败或仍缺失
conflict  = 发现已有 K 线与 Binance 返回不一致，不覆盖
cancelled = 人工取消或不再处理
```

状态变化必须可审计。

---

## 10. BackfillRun 模型需求

本阶段应新增执行审计模型：

```text
BackfillRun
```

BackfillResult 如保留，只能作为 BackfillRun 的结果摘要或结果字段语义，不得与 BackfillRun 作为同义主对象混用。

必须记录一次回补执行的结果。

字段至少覆盖：

```text
request
status
exchange
market_type
symbol
timeframe
mode
requested_start_time_utc
requested_end_time_utc
server_time_utc
fetched_count
closed_count
expected_count
inserted_count
skipped_existing_count
duplicate_in_response_count
conflict_count
missing_count
missing_open_times
conflict_open_times
page_count
attempt_count
error_code
error_message
allows_quality_recheck
trace_id
trigger_source
started_at_utc
finished_at_utc
created_at_utc
updated_at_utc
```

status 至少支持：

```text
success
failed
conflict
cancelled
```

---

## 11. 回补模式

本阶段至少支持三种模式。

### 11.1 request 模式

消费 002C 创建的 BackfillRequest。

输入：

```text
BackfillRequest.id
```

执行：

```text
读取 requested_start_time_utc / requested_end_time_utc / missing_open_times
请求 Binance REST
校验
幂等写入
更新 BackfillRequest
写 BackfillRun
```

### 11.2 range 模式

人工指定时间范围回补。

输入：

```text
exchange
market_type
symbol
timeframe
start_time
end_time
```

用途：

```text
项目初期历史 K 线初始化
人工指定区间补齐
```

range 模式可以先创建 BackfillRequest，再执行 BackfillService，避免绕过审计。

### 11.3 missing_open_times 模式

针对明确缺失的 open_time 列表进行回补。

输入：

```text
missing_open_times
```

用途：

```text
data_quality 明确发现缺失 K 线
```

要求：

```text
只检查和写入指定缺失 open_time
Binance 返回额外 open_time 时必须谨慎处理
不得越界写入与请求无关的大量历史数据
```

---

## 12. Binance REST 请求要求

002D 允许请求 Binance REST Kline。

必须使用：

```text
apps.market_data.binance_client.BinanceMarketDataClient
```

不得新写：

```text
requests.get
httpx
urllib 直接调用 Binance
自研第二套 Binance client
```

允许对 `BinanceMarketDataClient` 做最小增强，例如：

```text
支持明确 start_time / end_time / limit
支持分页所需的请求参数
返回标准化结果
```

但不得加入：

```text
账户接口
订单接口
仓位接口
签名交易接口
WebSocket
```

---

## 13. 分页与 limit

Binance Kline REST 单次请求有 limit 上限。

002D 历史回补必须分页，不得一次性请求超大范围。

要求：

```text
单次请求 limit 不超过 Binance 限制
分页基于 open_time_utc 向前推进
每页结果必须校验
分页必须有最大页数或安全上限
分页过程必须记录 page_count
```

建议 settings：

```text
DATA_BACKFILL_KLINE_PAGE_LIMIT = 1000
DATA_BACKFILL_MAX_PAGES_PER_RUN = 100
DATA_BACKFILL_MAX_BARS_PER_RUN = 100000
```

如果超过上限：

```text
BackfillRun = failed
error_code = limit_exceeded
不继续请求
写 AlertEvent
```

---

## 14. 已收盘过滤与时间边界

回补只能写入已收盘 K 线。

必须基于 Binance server time 或可信 reference time 判断。

建议：

```text
BackfillService 开始时调用 get_server_time()
server_time_utc 记录到 BackfillRun
只写 close_time_utc <= server_time_utc 的 K 线
```

K 线时间边界仍然使用：

```text
close_time_utc = open_time_utc + timeframe
```

不得保存 Binance 原始 closeTime。

不得保存 raw payload 为正式事实。

---

## 15. 校验要求

002D 必须复用 `apps/market_data/validators.py`。

写入前必须校验：

```text
Binance payload 结构
open_time_utc 可解析
close_time_utc = open_time_utc + timeframe
已收盘
OHLCV 合法
volume / quote_volume / trade_count 合法
同批 open_time 去重
expected open_time 是否覆盖请求缺口
```

如果校验失败：

```text
不写入该 K 线
记录 BackfillRun failed 或 conflict
写 AlertEvent
```

---

## 16. 写入规则

002D 写入 Kline4h / Kline1d 必须遵守：

```text
不存在 → 插入
已存在且一致 → 跳过
已存在但不一致 → conflict，不覆盖、不更新、不删除
同批重复且一致 → 去重
同批重复但不一致 → conflict
```

数据库唯一约束只是最后防线。

代码层必须先去重和查重。

不得依赖数据库 IntegrityError 作为正常去重流程。

---

## 17. 与 data_quality 复检的关系

BackfillRun 完成后，data_backfill 只标记 / 暴露需要 DataQuality 复检的状态。

只有新的 DataQualityResult = PASS 且 allows_downstream=True，才允许继续 MarketSnapshot。

回补完成不能直接放行 MarketSnapshot。

data_backfill 不直接投递 DataQuality task。

data_backfill 不同步调用 DataQuality。

data_backfill 不等待 DataQuality 结果。

DataQuality 复检由后续统一编排层、recovery scan 或人工命令触发。

```text
BackfillRun.allows_quality_recheck = True
BackfillRequest.status = success
AlertEvent 提示需要复检
```

原则：

```text
BackfillRun 完成后，必须标记 / 要求 DataQuality 复检
只有新的 DataQualityResult = PASS 且 allows_downstream=True，才允许继续 MarketSnapshot / 策略
```

---

## 18. AlertEvent 要求

本阶段可以写 AlertEvent，但不得直接调用 Hermes client，不得直接调用 notifications service。

正确链路：

```text
data_backfill
→ 写 AlertEvent
→ notifications service 异步消费
→ Hermes client 发送
```

必须写 AlertEvent 的场景：

```text
BackfillRun failed
BackfillRun conflict
BackfillRequest claim 失败异常
超过分页 / bars 上限
回补完成但仍缺失
```

建议写 AlertEvent 的场景：

```text
历史初始化成功
重要 BackfillRequest success
标记 / 要求 DataQuality 复检
```

---

## 19. Settings 要求

建议新增：

```text
DATA_BACKFILL_KLINE_PAGE_LIMIT = 1000
DATA_BACKFILL_MAX_PAGES_PER_RUN = 100
DATA_BACKFILL_MAX_BARS_PER_RUN = 100000
DATA_BACKFILL_STALE_RUNNING_SECONDS = 900
DATA_BACKFILL_REQUEST_SCAN_LIMIT = 20
DATA_BACKFILL_ENABLE_IMMEDIATE_TASK = True
```

这些配置必须写入 `.env.example` 或 settings 默认值说明。

不得把关键数字散落在业务代码中。

---

## 20. Celery task 要求

建议新增：

```text
process_backfill_request(backfill_request_id)
scan_pending_backfill_requests()
run_backfill_range(...)
```

task 只负责调用 service，不承载核心业务逻辑。

task 必须处理：

```text
request 不存在
request 已终态
request 正在 running
service 抛出受控异常
```

不得无限 retry。

如使用 Celery retry，必须有明确 max_retries 和短时同步/受控策略，不得产生跨周期隐式写入竞态。

---

## 21. Management command 要求

至少新增一个命令：

```text
python manage.py run_backfill
```

支持：

```text
--timeframe 4h|1d
--start-time
--end-time
--limit-pages
--trigger-source manual
```

可选新增：

```text
python manage.py run_backfill_requests --limit 10
python manage.py run_backfill_request --request-id 123
```

command 要求：

```text
只解析参数
调用 BackfillService
输出 BackfillRun id
输出 status
输出 inserted / skipped / conflict / missing 统计
```

---

## 22. 数据库与 migration 要求

必须使用 Django migrations。

如果新增 `apps/data_backfill`，必须在 settings 注册 app。

如果修改 `BackfillRequest`，必须新增 migration。

如果新增 `BackfillRun` / `BackfillResult`，必须新增 migration。

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

---

## 23. 测试要求

必须测试：

```text
BackfillRun 可创建
BackfillRequest pending 可被 claim 成 running
同一 request 并发 claim 只有一个成功
已终态 request 不会重复执行
request 模式能读取 BackfillRequest 并执行
range 模式能创建请求并执行
missing_open_times 模式只处理指定缺口
使用 BinanceMarketDataClient，不重新 HTTP 请求
测试中 Binance 请求必须 mock
分页请求能推进 open_time
超过最大页数 / bars 上限会失败
未收盘 K 线被过滤
非法 OHLCV 不写入
同批重复一致会去重
同批重复冲突会 conflict
已存在一致会跳过
已存在冲突不覆盖
缺失 K 线会插入
BackfillRun 记录 inserted / skipped / conflict / missing
BackfillRequest 状态正确更新
BackfillRun failed 写 AlertEvent
BackfillRun conflict 写 AlertEvent
回补成功后标记 / 要求 DataQuality 复检
management command 可运行
Celery task 可运行
```

测试禁止：

```text
真实请求 Binance
真实请求 Hermes
连接生产 MySQL
连接生产 Redis
生成 MarketSnapshot
进入策略 / 风控 / 交易
使用 WebSocket
```

---

## 24. 越界检查

Codex 完成后建议运行：

```bash
grep -R "requests.get" apps/data_backfill tests/data_backfill
grep -R "httpx" apps/data_backfill tests/data_backfill
grep -R "WebSocket" apps/data_backfill tests/data_backfill
grep -R "MarketSnapshot" apps/data_backfill tests/data_backfill
grep -R "CandidateOrderIntent / ApprovedOrderIntent" apps/data_backfill tests/data_backfill
grep -R "Execution" apps/data_backfill tests/data_backfill
grep -R "create_order" apps/data_backfill tests/data_backfill
grep -R "place_order" apps/data_backfill tests/data_backfill
```

如果命中，Codex 必须解释原因。

---

## 25. implementation 文档要求

完成后必须新增：

```text
docs/implementation/002D_data_backfill.md
```

内容至少包括：

```text
本阶段新增了哪些 app / model
BackfillRequest 如何被消费
BackfillRun 字段说明
BinanceMarketDataClient 复用说明
market_data validators 复用说明
Kline 幂等写入复用说明
历史初始化流程
指定区间回补流程
BackfillRequest 回补流程
分页规则
并发 claim / lock 规则
冲突处理规则
AlertEvent 写入规则
data_quality 复检规则
测试方式
本阶段明确未实现内容
```

必须明确写入：

```text
本阶段未实现 WebSocket。
本阶段未实现 MarketSnapshot。
本阶段未实现策略、风控、交易。
本阶段未覆盖或删除已有 Kline。
本阶段未直接调用 Hermes webhook。
```

---

## 26. 人工审查清单

用户合并前应检查：

```text
是否复用 BinanceMarketDataClient
是否没有重新写 Binance HTTP client
是否没有调用 DataCollectionService.collect_lookback
是否复用 market_data validators
是否抽出了或复用了 Kline 幂等写入逻辑
是否没有复制大量 002B 写入逻辑
是否 BackfillRequest claim 是原子的
是否防止重复处理同一 request
是否分页有上限
是否不写未收盘 K 线
是否不覆盖已有冲突 K 线
是否回补成功后标记 / 要求 DataQuality 复检
是否没有 MarketSnapshot / WebSocket / 策略 / 风控 / 交易
是否测试通过
```

---

## 27. Codex 输出总结要求

Codex 完成后必须输出：

```text
修改 / 新增文件清单
新增 app 名称
新增 model 名称
migration 文件路径
是否复用 BinanceMarketDataClient
是否复用 market_data validators
是否新增 Kline writer / repository
是否调整 DataCollectionService，是否保持对外行为不变
BackfillRequest 状态说明
BackfillRun 字段说明
分页规则说明
并发 claim / lock 说明
冲突处理说明
AlertEvent 写入规则
data_quality 复检规则
测试命令和结果
无法运行的命令及原因
是否存在未完成项
```

并明确说明：

```text
本阶段未实现 WebSocket。
本阶段未实现 MarketSnapshot。
本阶段未实现策略、风控、交易。
本阶段未覆盖或删除已有 Kline。
本阶段未直接调用 Hermes webhook。
```

---

## 28. 最终验收标准

本阶段完成后，必须满足：

```text
data_backfill app 可加载。
BackfillRun 执行审计模型存在。
BackfillRequest 可被消费。
BackfillRequest claim 原子且防并发。
支持历史区间初始化回补。
支持指定区间回补。
支持 data_quality BackfillRequest 回补。
使用 BinanceMarketDataClient。
复用 market_data validators。
Kline 写入幂等。
已有一致跳过。
已有冲突不覆盖。
同批重复先去重。
分页有上限。
未收盘 K 线不写入。
失败 / 冲突写 AlertEvent。
回补完成后标记 / 要求 DataQuality 复检。
不实现 WebSocket。
不实现 MarketSnapshot。
不实现策略、风控、交易。
测试通过。
```

---

## 29. 下一阶段

本阶段完成并验收后，下一阶段才开始：

```text
003A_market_snapshot_model_plan.md
```

003A 才允许实现：

```text
基于 DataQualityResult PASS 生成 MarketSnapshot
同时读取 4h + 1d K 线窗口
记录 snapshot payload
为 FeatureLayer / Strategy 提供稳定输入
```
