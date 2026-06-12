# Plan 002A：行情数据基础模型与数据库结构

> 核心边界：只建立正式 K 线主事实表，不实现采集、质量检查、回补、快照、策略、风控、交易。

---

## 0. 执行边界

本 plan 是第一个业务开发计划。

本 plan 只负责建立行情数据层的最小数据库模型、常量、基础约束、索引、migration、测试和 implementation 文档。

本 plan 的目标不是实现完整数据层，而是先建立后续 `data_collection`、`data_quality`、`data_backfill`、`MarketSnapshot` 可以依赖的正式 K 线主事实表。

Codex 不得把本 plan 扩展成完整数据层实现。

---

## 1. 阶段目标

本阶段完成后，项目应具备：

```text
market_data Django app
Kline4h 正式 K 线主表
Kline1d 正式 K 线主表
可选 BaseKline abstract model
受控常量
UTC 时间字段规范
DecimalField 价格和成交量字段
复合唯一约束
基础索引
数据库基础合法性约束
Django migrations
基础 model tests
implementation 文档
```

本阶段不连接 Binance。

本阶段不写采集 service。

本阶段不写 data_quality。

本阶段不写 data_backfill。

本阶段不写 MarketSnapshot。

本阶段不写策略、风控、交易。

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
docs/implementation/001A_project_foundation_framework.md
docs/implementation/001B_project_foundation_hermes.md
docs/plans/002A_market_data_models_plan.md
```

如果本 plan 与 `AGENTS.md` 或 `docs/rules/project_invariants.md` 冲突，必须以后两者为准，并在总结中说明冲突。

如果本 plan 与 requirements 冲突，必须停止并提示用户确认。

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
历史回补
data_quality 检查逻辑
DataQualityResult
DataQualityIssue
BackfillRun
BackfillResult
DataConflict
MarketSnapshot
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
Hermes 调用
AlertEvent 写入
notifications service 调用
```

本阶段禁止外部网络请求。

本阶段测试不得访问真实 Binance、真实 Hermes、生产 MySQL、生产 Redis。

---

## 4. 框架使用要求

必须使用：

```text
Django app
Django model
Django ORM
Django migrations
Django TestCase / pytest-django
Django settings
Python logging / Django logging
```

禁止：

```text
自研 ORM
手写 CREATE TABLE 作为业务建表方式
用 pymysql.connect / mysql.connector 承载业务写入
用 scripts 直接创建表
用 Redis 存储 K 线主事实
绕过 migrations 修改业务表
```

---

## 5. 建议 app 结构

建议创建：

```text
apps/market_data/
```

如果仓库已有等价 app，应优先沿用现有结构，不得重复创建多个行情 app。

建议文件：

```text
apps/market_data/__init__.py
apps/market_data/apps.py
apps/market_data/models.py
apps/market_data/constants.py
apps/market_data/migrations/__init__.py
tests/market_data/__init__.py
tests/market_data/test_kline_models.py
docs/implementation/002A_market_data_models.md
```

不得创建：

```text
tests/market_data/test_collection_event_model.py
```

因为本阶段不实现采集事件表。

---

## 6. 模型范围

本阶段只实现以下模型：

```text
Kline4h
Kline1d
```

允许实现抽象基类：

```text
BaseKline
```

但必须满足：

```text
BaseKline 只能是 abstract model。
不得创建 base_kline 真实业务表。
```

本阶段不实现：

```text
DataCollectionEvent
DataCollectionRun
DataQualityResult
DataQualityIssue
BackfillRun
BackfillResult
DataConflict
MarketSnapshot
```

---

## 7. Kline4h / Kline1d 模型定位

`Kline4h` 和 `Kline1d` 是正式行情 K 线主事实表。

```text
Kline4h = BTCUSDT U 本位 4h 已收盘 K 线
Kline1d = BTCUSDT U 本位 1d 已收盘 K 线
```

第一阶段使用 4h / 1d 分表。

必须：

```text
4h 和 1d 分表存储。
不得将 4h 和 1d 放入同一张正式 K 线表。
不做 1m。
```

`Kline4h` / `Kline1d` 只保存已经确认收盘的正式 K 线。

未收盘 K 线不得进入正式表。

---

## 8. Kline 字段要求

`Kline4h` 和 `Kline1d` 必须覆盖以下字段语义：

```text
exchange
market_type
symbol
open_time_utc
close_time_utc
open_price
high_price
low_price
close_price
volume
quote_volume
trade_count
data_source
trace_id
trigger_source
created_at_utc
updated_at_utc
```

字段命名应优先使用上述名称。

### 8.1 不允许的字段

本阶段不得在正式 K 线主表中实现：

```text
is_closed
raw_payload
```

原因：

```text
未收盘 K 线从采集入口就应该被拒绝，不应该进入正式 K 线主表。
正式 K 线主表只保存结构化 OHLCV，不保存 Binance 原始 closeTime 或 raw payload。
```

---

## 9. 时间字段规范

字段必须使用 UTC 语义：

```text
open_time_utc
close_time_utc
created_at_utc
updated_at_utc
```

要求：

```text
open_time_utc / close_time_utc 必须是 timezone-aware datetime 或项目统一 UTC datetime。
不得使用本地时间。
不得使用 PRC 时间。
不得根据运行机器时区推断。
```

### 9.1 close_time_utc 业务语义

`close_time_utc` 统一表示 K 线右开区间结束边界。

必须满足：

```text
Kline4h.close_time_utc = open_time_utc + 4 hours
Kline1d.close_time_utc = open_time_utc + 1 day
```

不得保存 Binance 原始 closeTime。

不得因为 Binance 原始 closeTime 是区间末尾毫秒而污染系统内部 K 线边界。

后续采集阶段如需要处理 Binance 原始 closeTime，应在采集解析层完成转换，不进入正式 K 线主表。

---

## 10. Decimal 精度要求

价格和成交量必须使用 `DecimalField`。

不得使用：

```text
FloatField
```

存储任何价格或成交量。

必须显式设置精度，不得让 Codex 自行猜测。

要求：

```text
open_price:
DecimalField(max_digits=30, decimal_places=12)

high_price:
DecimalField(max_digits=30, decimal_places=12)

low_price:
DecimalField(max_digits=30, decimal_places=12)

close_price:
DecimalField(max_digits=30, decimal_places=12)

volume:
DecimalField(max_digits=30, decimal_places=12)

quote_volume:
DecimalField(max_digits=30, decimal_places=12)
```

`trade_count` 应使用整数类型，不得使用 DecimalField 或 FloatField。

---

## 11. 常量要求

建议在 `apps/market_data/constants.py` 中定义常量，避免取值散落在业务代码中。

第一阶段至少包括：

```text
Exchange.BINANCE = binance
MarketType.USDS_M_FUTURES = usds_m_futures
MarketType.COIN_M_FUTURES = coin_m_futures
Symbol.BTCUSDT = BTCUSDT
Timeframe.FOUR_HOURS = 4h
Timeframe.ONE_DAY = 1d
DataSource.BINANCE_REST = binance_rest
```

本阶段不强制：

```text
Django choices
数据库取值 CheckConstraint
```

原因：

```text
当前业务默认只使用 Binance / U 本位 / BTCUSDT / binance_rest。
但不在数据库层锁死 exchange / market_type / symbol / data_source，避免后续扩展新 symbol 或新数据源时频繁改表结构。
```

---

## 12. 复合唯一约束

`Kline4h` 和 `Kline1d` 必须有复合唯一约束。

由于 4h / 1d 分表，单表唯一业务键至少包括：

```text
exchange
market_type
symbol
open_time_utc
```

目的：

```text
防止同一交易所、同一市场、同一品种、同一开盘时间的 K 线重复入库。
```

如果实现者选择保留 `timeframe` 字段，则唯一约束必须包括：

```text
exchange
market_type
symbol
timeframe
open_time_utc
```

但当前建议分表，不强制保存 `timeframe` 字段。

---

## 13. 基础索引

`Kline4h` 和 `Kline1d` 都应有基础索引，至少支持：

```text
按 exchange + market_type + symbol + open_time_utc 查询
按 symbol + open_time_utc 查询
按 open_time_utc 范围查询
```

目的：

```text
后续 data_quality 要按时间范围检查连续性。
后续 MarketSnapshot 要按窗口读取 4h / 1d。
没有索引会导致后续扫描性能过差。
```

---

## 14. 数据库基础合法性约束

本阶段不实现完整 data_quality。

但正式 K 线主事实表必须具备最低数据库防线，防止明显非法 OHLCV 数据入库。

`Kline4h` 和 `Kline1d` 必须增加 DB-level CheckConstraint，至少覆盖：

```text
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
close_time_utc > open_time_utc
```

这些约束不是完整的数据质量检查。

完整的数据缺失、重复、连续性、时效性、异常波动检查应留到后续 data_quality 阶段。

---

## 15. 与 DataCollectionEvent / DataCollectionRun 的关系

本阶段不实现：

```text
DataCollectionEvent
DataCollectionRun
CollectionEventType
CollectionEventStatus
CollectionMode
```

原因：

```text
002A 不做采集 service。
没有采集 service 时，采集事件表没有生产者和完整业务闭环。
```

后续 002B 实现 data_collection 时，再根据真实采集流程设计采集运行记录。

后续更推荐设计为批次 / 运行级记录，例如：

```text
DataCollectionRun
```

而不是逐根 K 线事件日志。

---

## 16. 与 AlertEvent / Hermes 的关系

本阶段不写 AlertEvent。

本阶段不调用 notifications service。

本阶段不调用 Hermes client。

K 线模型不得依赖 Hermes。

后续 data_collection / data_quality 发现问题时，才允许按规则写 AlertEvent。

---

## 17. 与 data_quality 的关系

本阶段只提供 data_quality 未来要检查的数据表。

本阶段不实现：

```text
缺失检查
重复检查
连续性检查
时效性检查
异常波动检查
完整 OHLCV 质量检查
DataQualityResult
DataQualityIssue
AlertEvent 写入
BackfillRequest
```

但模型约束必须支持后续检查，例如：

```text
open_time_utc 范围查询
复合唯一约束
基础索引
数据库最低合法性约束
data_source 字段
trace_id 字段
trigger_source 字段
```

---

## 18. 与 data_backfill 的关系

本阶段只提供回补未来写入的正式 K 线表。

本阶段不实现：

```text
BackfillRun
BackfillResult
BackfillEvent
DataConflict
回补逻辑
回补写入 service
```

后续回补和正常采集都应写入同一套正式 K 线主事实表。

---

## 19. 与 MarketSnapshot 的关系

本阶段只提供 MarketSnapshot 未来读取的正式 K 线表。

本阶段不实现：

```text
MarketSnapshot model
MarketSnapshot service
4h / 1d 新鲜度判断
lookback 窗口读取 service
FeatureLayer
AtomicSignal
StrategySignal
```

后续 MarketSnapshot 必须在 data_quality PASS 之后，由编排层主动生成，不得由策略层懒加载。

---

## 20. 测试要求

必须测试：

```text
market_data app 可加载
Kline4h 可创建
Kline1d 可创建
Kline4h 使用 DecimalField 保存价格和成交量
Kline1d 使用 DecimalField 保存价格和成交量
Kline4h 不使用 FloatField 保存价格或成交量
Kline1d 不使用 FloatField 保存价格或成交量
Kline4h 复合唯一约束生效
Kline1d 复合唯一约束生效
Kline4h 可保存 UTC open_time_utc / close_time_utc
Kline1d 可保存 UTC open_time_utc / close_time_utc
Kline4h 的 close_time_utc 可按 4h 边界保存
Kline1d 的 close_time_utc 可按 1d 边界保存
Kline4h OHLCV 数据库合法约束生效
Kline1d OHLCV 数据库合法约束生效
Kline4h 可保存 trace_id / trigger_source
Kline1d 可保存 trace_id / trigger_source
```

测试禁止：

```text
请求 Binance
请求 Hermes
连接生产 MySQL
连接生产 Redis
实现采集逻辑
实现 data_quality
实现 data_backfill
实现 MarketSnapshot
实现策略、风控、交易
```

不得测试：

```text
DataCollectionEvent 创建能力
DataCollectionRun 创建能力
is_closed 字段
raw_payload 字段
```

因为这些内容不属于 002A。

---

## 21. migration 要求

必须使用 Django migrations。

新增 model 后，必须创建 migration 文件：

```text
apps/market_data/migrations/0001_initial.py
```

Codex 完成后必须尝试运行：

```bash
python manage.py makemigrations market_data
python manage.py makemigrations --check --dry-run
python manage.py migrate --plan
python manage.py check
python manage.py test
```

如果项目使用 pytest，也必须运行：

```bash
pytest
```

说明：

```text
makemigrations market_data = 根据 models.py 创建 migration 文件。
makemigrations --check --dry-run = 检查 models.py 与 migrations 是否一致，不创建文件。
migrate --plan = 查看将要执行的 migration，不真正修改数据库。
```

如果新增 model 但没有 migration，必须停止并修复。

如果 `makemigrations --check --dry-run` 显示仍有未生成的 migration，必须停止并修复。

---

## 22. implementation 文档要求

完成后必须新增：

```text
docs/implementation/002A_market_data_models.md
```

内容至少包括：

```text
本阶段新增了哪些 app / model
Kline4h 字段说明
Kline1d 字段说明
BaseKline 是否存在，是否为 abstract model
唯一约束说明
索引说明
UTC 时间说明
close_time_utc 边界语义说明
Decimal 精度说明
数据库基础合法性约束说明
测试方式
本阶段明确未实现内容
```

必须明确写入：

```text
本阶段未实现 Binance REST client。
本阶段未实现 Binance serverTime 请求。
本阶段未实现 Binance Kline 请求。
本阶段未实现 WebSocket。
本阶段未实现 data_collection service。
本阶段未实现 DataCollectionEvent / DataCollectionRun。
本阶段未实现 data_quality。
本阶段未实现 data_backfill。
本阶段未实现 MarketSnapshot。
本阶段未实现 FeatureLayer / AtomicSignal / StrategySignal。
本阶段未实现策略、风控、交易。
本阶段未调用 AlertEvent / Hermes / notifications service。
```

---

## 23. 人工审查清单

用户合并前应检查：

```text
是否只新增 market_data 相关基础模型
是否只实现 Kline4h / Kline1d
是否没有 DataCollectionEvent / DataCollectionRun
是否没有 is_closed 字段
是否没有 raw_payload 字段
是否没有 Binance 原始 closeTime 字段
是否没有 Binance REST client
是否没有 Binance serverTime 请求
是否没有 Binance Kline 请求
是否没有 WebSocket
是否没有数据采集 service
是否没有 data_quality
是否没有 data_backfill
是否没有 MarketSnapshot
是否没有策略 / 风控 / 交易
是否 Kline4h / Kline1d 分表
是否 DecimalField 用于价格和成交量
是否没有 FloatField 用于价格和成交量
是否存在复合唯一约束
是否存在基础索引
是否存在数据库基础合法性 CheckConstraint
是否存在 migrations
是否测试通过
```

建议搜索：

```bash
grep -R "requests.get" apps tests
grep -R "httpx" apps tests
grep -R "binance" apps tests
grep -R "Hermes" apps tests
grep -R "AlertEvent" apps tests
grep -R "DataCollectionEvent" apps tests
grep -R "DataCollectionRun" apps tests
grep -R "DataQuality" apps tests
grep -R "Backfill" apps tests
grep -R "MarketSnapshot" apps tests
grep -R "StrategySignal" apps tests
grep -R "CandidateOrderIntent / ApprovedOrderIntent" apps tests
grep -R "Execution" apps tests
grep -R "pymysql.connect" apps tests
grep -R "CREATE TABLE" apps tests
grep -R "FloatField" apps tests
```

如果命中，Codex 必须解释原因。

---

## 24. Codex 输出总结要求

Codex 完成后必须输出：

```text
修改 / 新增文件清单
新增 app 名称
新增 model 名称
是否新增 migrations
Kline4h / Kline1d 是否分表
Kline4h / Kline1d 字段清单
DecimalField 精度说明
复合唯一约束说明
基础索引说明
数据库基础合法性约束说明
测试命令和结果
无法运行的命令及原因
是否存在未完成项
```

并明确说明：

```text
本阶段未实现 Binance REST client。
本阶段未实现 Binance serverTime 请求。
本阶段未实现 Binance Kline 请求。
本阶段未实现 WebSocket。
本阶段未实现 data_collection service。
本阶段未实现 DataCollectionEvent / DataCollectionRun。
本阶段未实现 data_quality。
本阶段未实现 data_backfill。
本阶段未实现 MarketSnapshot。
本阶段未实现策略、风控、交易。
本阶段未调用 AlertEvent / Hermes / notifications service。
```

---

## 25. 最终验收标准

本阶段完成后，必须满足：

```text
market_data app 可加载。
Kline4h model 存在。
Kline1d model 存在。
Kline4h / Kline1d 使用分表。
BaseKline 如存在，必须是 abstract model。
Kline4h / Kline1d 不包含 is_closed。
Kline4h / Kline1d 不包含 raw_payload。
Kline4h / Kline1d 不保存 Binance 原始 closeTime。
open_time_utc / close_time_utc 使用 UTC 语义。
Kline4h.close_time_utc 表示 open_time_utc + 4 hours。
Kline1d.close_time_utc 表示 open_time_utc + 1 day。
价格和成交量使用 DecimalField(max_digits=30, decimal_places=12)。
价格和成交量不使用 FloatField。
Kline4h / Kline1d 有复合唯一约束。
Kline4h / Kline1d 有基础索引。
Kline4h / Kline1d 有数据库基础合法性 CheckConstraint。
migrations 存在。
model tests 通过。
默认测试不访问真实外部服务。
未实现 Binance REST client。
未实现采集逻辑。
未实现 DataCollectionEvent / DataCollectionRun。
未实现 data_quality。
未实现 data_backfill。
未实现 MarketSnapshot。
未实现策略、风控、交易。
```

---

## 26. 下一阶段

本阶段完成并验收后，下一阶段才开始：

```text
002B_data_collection_gateway_and_service_plan.md
```

002B 才允许讨论和实现：

```text
Binance REST market data client
serverTime 获取
Kline REST 拉取
已收盘过滤
lookback window 采集
幂等写入 service
采集运行记录
AlertEvent 写入规则
```

002B 是否实现 `DataCollectionRun`，应在 002B plan 中重新审查，不从 002A 预先创建。
