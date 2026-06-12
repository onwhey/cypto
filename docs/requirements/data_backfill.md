# data_backfill.md

# 数据回补模块需求

## 1. 文档目的

本文档定义中低频趋势跟踪自动交易系统的数据回补模块需求。

数据回补模块负责在初始化、缺口、异常、人工指定复核和数据冲突场景下，通过 Binance REST 重新拉取官方已收盘 K 线，并以幂等、可追溯、不可静默覆盖的方式写入对应周期 K 线表。

本文档只定义数据回补模块需求，不定义：

```text
具体数据库字段
具体 Django model
具体 Django management command 名称
具体 Celery task 名称
具体 Binance endpoint 封装代码
具体质量检查算法
具体 Hermes 发送实现
具体策略逻辑
具体风控逻辑
具体交易执行逻辑
```

数据回补模块不判断回补后的数据是否最终可信。

BackfillRun 完成后，必须标记 / 要求 DataQuality 复检；实际复检触发由后续统一编排层、recovery scan 或人工命令负责。

data_backfill 不直接触发 DataQuality task，不同步调用 DataQuality，不等待 DataQuality 结果。

只有新的 DataQualityResult = PASS 且 allows_downstream=True，才允许继续 MarketSnapshot、FeatureLayer、策略链路、OrderPlan、RiskCheck 和 Execution 链路。

回补完成不能直接放行 MarketSnapshot。

---

## 2. 模块目标

数据回补模块的目标是：

```text
补齐项目初始化所需历史 K 线
补齐运行过程中发现的缺失 K 线
对人工指定时间范围重新拉取官方 K 线
对数据冲突重新拉取官方 K 线用于复核
保证回补数据只来自 Binance REST 官方已收盘 K 线
保证回补写入幂等
保证回补不静默覆盖已有 K 线
保证回补过程可审计、可复盘、可告警
```

第一阶段范围：

```text
交易所：Binance
市场类型：U 本位合约
交易品种：BTCUSDT
K 线周期：4h、1d
数据来源：Binance REST
时间体系：UTC
```

说明：

```text
4h K 线写入 4h 专用 K 线表。
1d K 线写入 1d 专用 K 线表。
正常采集和回补写入同一张对应周期 K 线表。
不得因为是回补数据而另建正式 K 线回补表。
```

---

## 3. 模块边界

### 3.1 数据回补模块负责

```text
接收明确的回补请求
校验回补参数
按 UTC 时间范围分批请求 Binance REST
过滤未收盘 K 线
解析 Binance K 线数据
按唯一业务键幂等写入对应周期 K 线表
发现冲突时记录冲突并阻断写入
记录 BackfillRun / BackfillEvent；BackfillResult 如保留，仅作为 BackfillRun 的结果摘要语义
记录 AlertEvent
回补完成后必须标记 / 要求 DataQuality 复检；实际复检触发由后续统一编排层、recovery scan 或人工命令负责。
支持 dry-run 回补检查
BackfillRequest 原子 claim / 防重复执行
missing_open_times 精确回补
分页上限与 limit_exceeded 处理
与 data_collection 共用 K 线写入锁
```

### 3.2 数据回补模块不负责

```text
定时增量采集
WebSocket 实时行情监测
数据质量最终放行
生成 MarketSnapshot
计算 FeatureValue
生成 AtomicSignal
生成 StrategySignal
生成 DecisionSnapshot
读取账户事实
读取仓位事实
调用 RiskCheck
生成 CandidateOrderIntent / ApprovedOrderIntent
调用 Execution
调用大模型
同步调用 Hermes Webhook
人工修改 K 线
自动修复 K 线
静默覆盖冲突 K 线
```

---

## 4. 回补和采集的关系

数据采集和数据回补是代码职责拆分，不是存储分表拆分。

```text
data_collection
= 正常采集 + 增量 lookback window 短期自动补漏

data_backfill
= 初始化历史补齐 + 明确缺口回补 + 人工指定区间回补 + 冲突复核
```

存储规则：

```text
正常采集得到的 4h K 线写入 4h K 线表。
回补得到的 4h K 线也写入 4h K 线表。
正常采集得到的 1d K 线写入 1d K 线表。
回补得到的 1d K 线也写入 1d K 线表。
```

不得创建：

```text
正常采集 K 线表
回补 K 线表
人工修复 K 线表
临时正式 K 线表
```

说明：

```text
data_collection 中的 historical collection 表示底层历史拉取能力。
data_backfill 中的 initial_historical_backfill 表示由回补模块编排的大范围初始化补齐场景。
两者可复用底层 Binance REST 拉取、解析和幂等写入能力，但回补流程必须有独立的 BackfillRun / BackfillEvent 审计记录；BackfillResult 如保留，仅作为 BackfillRun 的结果摘要语义。
```

---

## 5. 第一阶段范围

### 5.1 P0 必须实现

```text
4h K 线 initial_historical_backfill
1d K 线 initial_historical_backfill
4h K 线 gap_backfill
1d K 线 gap_backfill
manual_range_backfill
conflict_recheck
Binance REST 官方已收盘 K 线拉取
UTC 起止时间参数
周期边界校验
分批请求
未收盘过滤
幂等写入
冲突不覆盖
BackfillRun 记录
BackfillResult 结果摘要记录（如保留）
BackfillEvent 记录
AlertEvent 记录
回补完成后标记 / 要求 DataQuality 复检；实际复检由后续统一编排层、recovery scan 或人工命令重新执行。
与 data_collection 共用 K 线写入锁
dry-run 回补检查
```

### 5.2 P1 可预留

```text
回补任务重试
回补任务暂停 / 取消
大范围历史回补进度统计
回补后标记 / 要求 DataQuality 复检
回补任务查询命令
多次失败聚合告警
```

### 5.3 P2 后续再做

```text
多品种回补
多交易所回补
多数据源交叉回补
复杂冲突解决工作流
可视化回补任务管理
自动化长期历史完整性扫描
```

---

## 6. 回补触发场景

### 6.1 initial_historical_backfill

项目初始化时，用于补齐系统运行和回测所需历史数据。

示例：

```text
补齐过去两年的 BTCUSDT 4h K 线
补齐过去两年的 BTCUSDT 1d K 线
```

特点：

```text
通常由人工命令触发。
必须指定明确起止 UTC 时间。
不得自动猜测超大时间范围。
必须支持分批请求。
必须幂等。
```

### 6.2 gap_backfill

data_quality 发现缺失、不连续或断档后，针对缺失区间重新拉取。

示例：

```text
数据库已有 04:00、08:00、16:00
缺少 12:00
data_quality 发现 MISSING_KLINE
触发或人工执行 gap_backfill
```

规则：

```text
gap_backfill 只补明确缺失的时间范围。
gap_backfill 不得扩展为无限制历史重拉。
```

### 6.3 manual_range_backfill

用户人工指定某个 UTC 时间范围重新拉取。

用途：

```text
人工复核某段历史 K 线
补偿采集长期失败区间
重新拉取某段 K 线供 data_quality 复检
```

规则：

```text
必须明确 symbol、timeframe、start_time_utc、end_time_utc。
不得缺省起止时间后自动猜测。
不得使用本地时间。
```

### 6.4 conflict_recheck

当 data_collection 或 data_quality 发现同一唯一业务键下 OHLCV 不一致时，重新从 Binance REST 拉取官方 K 线进行复核。

规则：

```text
conflict_recheck 不得自动覆盖已有正式 K 线。
conflict_recheck 只记录新旧值对比、冲突状态和复核结果。
冲突解决是否需要人工确认，由后续数据质量 / 人工恢复流程定义。
```

### 6.5 failure_recovery_backfill

当采集任务连续失败或系统停机导致某段时间 K 线缺失时，用于补齐失败期间数据。

规则：

```text
必须指定明确 UTC 时间范围。
必须走 Binance REST。
必须复检 data_quality。
```

---

### 6.6 missing_open_times 精确回补

当 BackfillRequest 指定 `missing_open_times` 时，data_backfill 必须按精确 open_time 集合处理。

规则：

```text
missing_open_times 非空时，只允许写入这些 open_time 对应的 K 线。
Binance REST 返回范围内额外 open_time 时，必须过滤掉。
不得因为请求范围覆盖了额外 K 线，就顺手写入未被请求的 open_time。
过滤掉的额外 K 线应计入审计摘要或 BackfillEvent。
```

说明：

```text
精确回补用于防止 data_quality 只请求补某几根缺失 K 线时，data_backfill 越界写入额外数据。
如果需要扩大回补范围，必须创建新的 BackfillRequest 或由人工明确指定 manual_range_backfill。
```

## 7. 回补数据来源

第一阶段正式 K 线回补只能使用：

```text
Binance REST 官方已收盘 K 线
```

允许调用：

```text
Binance server time
Binance Kline REST
```

禁止使用：

```text
WebSocket 拼接 K 线
REST 最新价格接口拼 K 线
第三方行情源
人工录入价格
人工录入成交量
推测值
manual_repair
system_repair
human_edit
manual_input
大模型生成数据
```

禁止访问：

```text
Binance order endpoint
Binance account endpoint
Binance position endpoint
Binance leverage endpoint
Binance margin endpoint
Binance listenKey
```

数据回补模块不得读取账户、订单、仓位、杠杆或保证金信息。

---

## 8. 时间规则

数据回补模块所有业务时间统一使用 UTC。

要求：

```text
start_time_utc 必须是 UTC。
end_time_utc 必须是 UTC。
回补范围必须按对应周期边界对齐。
K 线 open_time 必须按 Binance 返回时间戳解释为 UTC。
K 线 close_time 必须按 Binance 返回时间戳解释为 UTC。
如果系统保存 close_time_utc，只保存规范化周期结束时间，不保存 Binance raw closeTime 作为核心业务字段。
```

禁止：

```text
使用本地时间作为回补范围。
使用 PRC 时间作为回补范围。
根据运行机器时区推断时间。
根据请求 IP 推断时间。
请求 K 线时传 timeZone 参数。
```

周期边界：

```text
4h open_time_utc 必须落在 00:00、04:00、08:00、12:00、16:00、20:00。
1d open_time_utc 必须落在 UTC 00:00。
```

---

## 9. 输入对象

数据回补模块输入至少包括：

```text
exchange
market_type
symbol
timeframe
backfill_mode
start_time_utc
end_time_utc
limit_per_request
data_source
trigger_source
trace_id
dry_run
reason
missing_open_times（可选）
```

### 9.1 exchange

第一阶段：

```text
binance
```

### 9.2 market_type

第一阶段：

```text
usds_m_futures
```

### 9.3 symbol

第一阶段：

```text
BTCUSDT
```

### 9.4 timeframe

第一阶段允许：

```text
4h
1d
```

### 9.5 backfill_mode

建议取值：

```text
initial_historical_backfill
gap_backfill
manual_range_backfill
conflict_recheck
failure_recovery_backfill
```

### 9.6 data_source

正式回补数据来源只能是：

```text
binance_rest
```

### 9.7 trigger_source

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

说明：

```text
第一阶段可以优先支持 management_command / cli。
后续如允许系统自动触发回补，必须显式记录 trigger_source = celery_worker / system / retry。
trigger_source 不得为空。
trigger_source 不得由程序猜测。
Celery worker 执行任务时不得覆盖原始 trigger_source。
```

---

## 10. 输出对象

数据回补模块输出包括：

```text
BackfillRun
BackfillResult（如保留，仅作为 BackfillRun 的结果摘要语义）
BackfillEvent
DataConflict
AlertEvent
KlineRecord
```

### 10.1 BackfillRun

表示一次回补任务。

至少应表达：

```text
trace_id
trigger_source
backfill_mode
exchange
market_type
symbol
timeframe
requested_start_time_utc
requested_end_time_utc
status
created_at_utc
```

### 10.2 BackfillRequest claim / 防重复执行

如果 data_backfill 消费 DataQuality 创建的 BackfillRequest，必须先原子 claim。

允许 claim 的状态：

```text
pending
failed（且允许 retry）
```

claim 时必须原子更新或等价保证：

```text
status = running
locked_by = trace_id 或任务唯一标识
locked_at_utc
attempt_count = attempt_count + 1
started_at_utc
```

如果 BackfillRequest 已经处于以下状态：

```text
running
success
conflict
blocked
cancelled
```

则本次执行不得请求 Binance，不得写 K 线表，只能返回 skipped / already_claimed / terminal_state 等结构化结果。

规则：

```text
BackfillRequest claim 是防止重复回补的第一道门。
K 线写入锁是防止同一 symbol/timeframe 并发写入的第二道门。
两者都不能省略。
```

### 10.3 BackfillResult

如保留 BackfillResult，只能作为 BackfillRun 的结果摘要或结果字段语义，不得与 BackfillRun 作为同义主对象混用。

状态建议：

```text
running
success
failed
blocked
skipped
dry_run_success
dry_run_failed
```

含义：

```text
running = 任务已开始
success = 回补成功完成
failed = 外部请求、解析、写库或未预期异常导致失败
blocked = 参数、质量、冲突或规则导致阻断
skipped = 因锁、重复任务或条件不满足跳过
dry_run_success = dry-run 检查完成且未发现阻断问题
dry_run_failed = dry-run 检查发现问题或执行失败
```

至少应表达：

```text
fetched_count
closed_count
inserted_count
skipped_count
conflict_count
filtered_unclosed_count
error_code
error_message
started_at_utc
finished_at_utc
```

### 10.4 BackfillEvent

表示回补过程中的事件。

事件示例：

```text
backfill_started
binance_request_started
binance_request_succeeded
binance_request_failed
unclosed_kline_filtered
kline_inserted
kline_skipped_existing
kline_conflict_detected
backfill_blocked
backfill_failed
backfill_succeeded
```

### 10.5 DataConflict

表示回补过程中发现的数据冲突。

触发条件：

```text
同一唯一业务键已存在，
但重新拉取的 OHLCV / quote_volume / trade_count 与数据库已有值不一致。
```

DataConflict 至少应能记录：

```text
exchange
market_type
symbol
timeframe
open_time_utc
old_value_summary
new_value_summary
data_source
trigger_source
backfill_mode
trace_id
created_at_utc
```

### 10.6 AlertEvent

回补失败、阻断、冲突或无法确认健康状态时，必须写入 AlertEvent。

AlertEvent 由 notifications / Hermes 服务异步消费。

data_backfill 不直接同步调用 Hermes。

---

## 11. 分批请求要求

Binance K 线接口有单次请求 limit 限制，回补必须支持分批请求。

要求：

```text
根据 start_time_utc / end_time_utc 拆分请求范围。
每批请求 limit 不得超过 Binance K 线接口最大限制。
每批请求范围必须按 timeframe 推进。
不得无限循环。
必须记录每批请求范围和结果摘要。
```

失败规则：

```text
任一批次请求失败，本次回补必须 failed 或 blocked。
不得静默跳过失败批次继续写入后续批次。
第一阶段不支持 partial_success。
```

说明：

```text
第一阶段优先采用 all-or-nothing 语义。
如果实现上无法保证完全事务回滚，必须通过 BackfillRun 结果字段或 BackfillResult 结果摘要明确记录可能的部分影响。
```

---

### 11.1 分页上限与 limit_exceeded

回补必须配置分页和写入规模上限。

建议配置项：

```text
DATA_BACKFILL_KLINE_PAGE_LIMIT
DATA_BACKFILL_MAX_PAGES_PER_RUN
DATA_BACKFILL_MAX_BARS_PER_RUN
```

规则：

```text
每页请求 limit 不得超过 Binance 官方 K 线接口限制。
如果预计或实际分页超过 DATA_BACKFILL_MAX_PAGES_PER_RUN，必须停止本次回补。
如果预计或实际 K 线数量超过 DATA_BACKFILL_MAX_BARS_PER_RUN，必须停止本次回补。
超过限制时 BackfillRun.status = failed 或 blocked；如保留 BackfillResult，同步记录结果摘要。
error_code = limit_exceeded。
必须写 BackfillEvent 和 AlertEvent。
不得写入已经拉取到的部分 KlineRecord。
```

说明：

```text
P0 优先 all-or-nothing。
如果实现无法事务性回滚已经写入的内容，必须在 BackfillRun 结果字段或 BackfillResult 结果摘要中明确记录可能的部分影响，并且不得允许下游继续。
```

## 12. 未收盘过滤要求

回补只能写入已收盘 K 线。

判断依据：

```text
Binance serverTime
K 线 close_time
```

规则：

```text
如果 close_time >= Binance serverTime，视为未收盘。
未收盘 K 线不得写入 4h / 1d 主 K 线表。
未收盘 K 线必须计入 filtered_unclosed_count。
如果请求结果全部为未收盘 K 线，本次回补必须 blocked 或 failed，并写入 AlertEvent。
```

禁止：

```text
使用本机时间作为唯一判断依据。
未获取 Binance serverTime 仍继续写入。
把未收盘 K 线写入正式表。
后续再覆盖未收盘 K 线。
```

---

## 13. 幂等写入规则

回补必须支持重复执行。

同一根 K 线唯一业务键至少包括：

```text
exchange
market_type
symbol
open_time_utc
```

说明：

```text
第一阶段 4h 和 1d 分表存储，单表唯一键中不强制包含 timeframe。
如果未来多周期共表，唯一键必须包含 timeframe。
```

写入规则：

```text
同一唯一键不存在 → 插入。
同一唯一键存在且 OHLCV 一致 → 跳过或计入 skipped_count。
同一唯一键存在但 OHLCV 不一致 → 记录 DataConflict，阻断本次回补，不得覆盖。
```

禁止：

```text
静默覆盖已有 K 线。
删除已有 K 线。
人工修复已有 K 线。
自动选择冲突数据中的一条作为正确数据。
```

---

## 14. K 线写入锁

data_backfill 会写入正式 K 线表，必须与 data_collection 共用 KlineWriteLock。

KlineWriteLock 属于 `apps.market_data`，由 002B 实现。
data_collection 和 data_backfill 必须共用 KlineWriteLock 保护 Kline4h / Kline1d 主事实表写入。
MarketSnapshot 只检查 KlineWriteLock 是否活跃，不创建锁。

锁粒度：

```text
exchange + market_type + symbol + timeframe
```

推荐锁 key 语义：

```text
kline_write:{exchange}:{market_type}:{symbol}:{timeframe}
```

要求：

```text
获取锁必须具备原子性。
锁必须设置 TTL。
锁 owner 应为 trace_id 或等价唯一任务标识。
如果锁已存在，本次回补必须 skipped 或拒绝执行。
锁存在时不得继续请求 Binance。
锁存在时不得继续写正式 K 线表。
释放锁时必须校验 owner。
只能释放当前任务自己持有的锁。
```

目的：

```text
防止 data_collection 与 data_backfill 同时写同一 symbol/timeframe。
防止两个回补任务并发写同一 symbol/timeframe。
防止 lookback 自动补漏与人工回补互相冲突。
```

---

## 15. dry-run 要求

data_backfill 应支持 dry-run。

dry-run 允许：

```text
校验参数
请求 Binance REST
过滤未收盘 K 线
解析 K 线
比对数据库已有 K 线
生成回补报告
写 BackfillRun 结果字段 / BackfillEvent；BackfillResult 如保留，仅作为结果摘要
返回潜在阻断、冲突或异常摘要
```

说明：

```text
dry-run 默认不写正式 K 线表。
dry-run 是否写 BackfillRun 结果字段 / BackfillEvent / BackfillResult 结果摘要由实现配置决定，但必须在结果中明确标记为 dry_run。
dry-run 默认不写正式告警 AlertEvent；如项目选择记录严重 dry-run 异常，也必须明确标记为 dry_run，不得触发正式交易链路。
```

dry-run 禁止：

```text
写入 4h / 1d 主 K 线表
修改已有 K 线
删除已有 K 线
覆盖冲突数据
生成 MarketSnapshot
进入策略链路
调用 Execution
```

dry-run 目的：

```text
人工确认回补范围和潜在冲突。
在正式写入前预估 fetched、closed、inserted、skipped、conflict 数量。
```

---

## 16. 成功流程

### 16.1 标准回补流程

```text
接收回补请求
→ 校验参数
→ 如果来自 BackfillRequest，先原子 claim 为 running
→ 校验 missing_open_times 精确范围（如存在）
→ 获取 K 线写入锁
→ 写 BackfillRun.status = running
→ 获取 Binance serverTime
→ 按时间范围分批请求 Binance REST K 线
→ 过滤未收盘 K 线
→ 如果 missing_open_times 非空，过滤不在集合内的额外 K 线
→ 转换为内部 KlineRecord
→ 与数据库已有 K 线按唯一业务键比对
→ 不存在则插入
→ 已存在且一致则跳过
→ 发现冲突则阻断
→ 写 BackfillRun 结果字段 / BackfillEvent；BackfillResult 如保留，仅作为结果摘要
→ 释放 K 线写入锁
→ 标记 / 要求 DataQuality 复检
```

### 16.2 回补成功后的强制复检

回补成功不等于数据可信。

BackfillRun 完成后，必须标记 / 要求 DataQuality 复检；实际复检触发由后续统一编排层、recovery scan 或人工命令负责。

data_backfill 不直接触发 DataQuality task，不同步调用 DataQuality，不等待 DataQuality 结果。

```text
BackfillRun.status = success；BackfillResult 如保留，仅作为结果摘要
→ 标记 / 要求 DataQuality 复检
→ 后续统一编排层、recovery scan 或人工命令重新执行 DataQuality
→ 新的 DataQualityResult = PASS
→ allows_downstream=True
→ 允许继续 MarketSnapshot
```

如果 data_quality 复检 FAIL：

```text
不得生成 MarketSnapshot。
不得进入 FeatureLayer。
不得进入策略链路。
不得进入 OrderPlan / RiskCheck / Execution 链路。
```

---

## 17. 失败与阻断流程

### 17.1 请求失败

```text
Binance REST 请求失败
→ 写 BackfillRun.status = failed；BackfillResult 如保留，仅作为结果摘要
→ 写 BackfillEvent
→ 写 AlertEvent
→ 不写入正式 K 线表
→ 释放锁
```

### 17.2 参数失败

```text
参数缺失 / 时间非法 / 周期不对齐
→ 写 BackfillRun.status = blocked 或 failed；BackfillResult 如保留，仅作为结果摘要
→ 写 BackfillEvent
→ 写 AlertEvent
→ 不请求 Binance
→ 不写入正式 K 线表
```

### 17.3 数据冲突

```text
发现同一唯一键 OHLCV 不一致
→ 写 DataConflict
→ 写 BackfillRun.status = blocked；BackfillResult 如保留，仅作为结果摘要
→ 写 AlertEvent
→ 不覆盖已有 K 线
→ 不继续写冲突后的数据
→ 释放锁
```

### 17.4 写库失败

```text
写入 K 线表失败
→ 回滚当前事务中未提交写入
→ 写 BackfillRun.status = failed；BackfillResult 如保留，仅作为结果摘要
→ 写 AlertEvent
→ 释放锁
```

### 17.5 无法确认健康状态

如果发生异常导致系统无法确认回补是否成功、是否部分写入、是否释放锁，必须：

```text
写 AlertEvent
记录 BackfillRun.status = failed 或 unknown；BackfillResult 如保留，仅作为结果摘要
要求人工检查
不得继续下游流程
```

---

## 18. AlertEvent 规则

以下情况必须写入 AlertEvent：

```text
回补参数非法
Binance serverTime 获取失败
Binance REST 请求失败
请求结果为空
全部 K 线未收盘
回补结果被 blocked
发现 DataConflict
写库失败
任务异常
无法确认回补健康状态
释放锁失败
data_quality 复检 FAIL
```

AlertEvent 必须能表达：

```text
event_type
exchange
market_type
symbol
timeframe
backfill_mode
trigger_source
data_source
requested_start_time_utc
requested_end_time_utc
error_code
error_message
trace_id
created_at_utc
```

说明：

```text
data_backfill 只写 AlertEvent。
notifications / Hermes 服务异步消费 AlertEvent。
data_backfill 不直接同步调用 Hermes Webhook。
Hermes 发送失败不得改变 BackfillRun 状态；BackfillResult 如保留，也不得因 Hermes 发送失败改变结果摘要。
```

---

## 19. 与 data_quality 的关系

data_backfill 不代替 data_quality。

data_backfill 只负责补齐或重新拉取。

data_quality 负责判断回补后的数据是否能进入下游。

规则：

```text
回补前可根据 DataQualityIssue 确定回补范围。
BackfillRun 完成后，必须标记 / 要求 DataQuality 复检；实际复检触发由后续统一编排层、recovery scan 或人工命令负责。
只有新的 DataQualityResult = PASS 且 allows_downstream=True，才允许继续 MarketSnapshot。
data_backfill 不得绕过 data_quality。
```

data_backfill 不负责：

```text
最终判断数据可信
生成 DataQualityResult = PASS
强行放行 MarketSnapshot
```

---

## 20. 与 data_collection 的关系

data_collection 负责：

```text
正常采集
增量 lookback window 自动补齐短期漏采
```

data_backfill 负责：

```text
初始化历史补齐
明确缺口回补
人工指定范围回补
冲突复核
采集长期失败后的补偿
```

两者共同遵守：

```text
只写入对应周期 K 线表
不静默覆盖
不人工修复
不写未收盘 K 线
共用 K 线写入锁
统一 UTC
统一唯一业务键
```

---

## 21. 与 MarketSnapshot 的关系

MarketSnapshot 不直接读取 BackfillResult 作为可信依据。

MarketSnapshot 只能读取通过 data_quality PASS 的 K 线。

BackfillRun 完成后，必须标记 / 要求 DataQuality 复检；实际复检触发由后续统一编排层、recovery scan 或人工命令负责。只有新的 DataQualityResult = PASS 且 allows_downstream=True，才允许继续 MarketSnapshot。

如果回补成功但 data_quality 尚未复检：

```text
不得生成 MarketSnapshot。
```

如果回补成功但 data_quality 复检 FAIL：

```text
不得生成 MarketSnapshot。
```

---

## 22. 与 notifications / Hermes 的关系

data_backfill 只写 AlertEvent。

notifications / Hermes 服务负责：

```text
消费 AlertEvent
发送 HermesNotification
记录发送结果
处理发送失败
```

data_backfill 不负责：

```text
同步调用 Hermes Webhook
等待 Hermes 发送成功
处理 Hermes 重试
根据 Hermes 结果改变回补状态
```

---

## 23. 与交易链路的关系

data_backfill 不参与交易链路。

data_backfill 不调用：

```text
OrderPlan
RiskCheck
ExecutionPreparation
Execution
Binance Account Sync
账户接口
仓位接口
订单接口
杠杆接口
保证金接口
```

data_backfill 不生成：

```text
CandidateOrderIntent
ApprovedOrderIntent
交易所订单 payload
执行结果
```

如果回补失败或质量复检失败，本轮数据链路已经阻断，不会进入 MarketSnapshot、策略链路或交易链路。

---

## 24. 禁止事项

数据回补模块禁止：

```text
使用本地时间作为业务时间
请求 K 线时传 timeZone 参数
写入未收盘 K 线
使用 WebSocket 拼接 K 线
使用第三方行情源
人工录入 K 线
人工修改 OHLCV
manual_repair
system_repair
human_edit
manual_input
静默覆盖冲突 K 线
删除已有 K 线
自动选择冲突数据中的一条作为正确数据
跳过 data_quality 复检
生成 MarketSnapshot
计算 FeatureValue
生成 AtomicSignal
生成 StrategySignal
生成 DecisionSnapshot
读取账户事实
读取仓位事实
调用 RiskCheck
生成 CandidateOrderIntent / ApprovedOrderIntent
调用 Execution
调用大模型
同步调用 Hermes Webhook
访问 Binance 交易类接口
```

---

## 25. 测试要求

默认测试不得依赖真实外部服务。

测试应覆盖：

```text
参数缺失时拒绝
start_time_utc >= end_time_utc 时拒绝
时间范围不按周期边界对齐时拒绝
不允许本地时间参数
不允许非法 timeframe
Binance REST 请求失败时记录 failed
serverTime 获取失败时记录 failed
未收盘 K 线被过滤
全部未收盘时阻断
已存在且一致时跳过
不存在时插入
已存在但 OHLCV 不一致时记录 DataConflict 且不覆盖
dry-run 不写 K 线表
BackfillRequest 通过原子 claim 防止重复执行
missing_open_times 非空时只回补指定 open_time
超过分页或 bars 上限时 limit_exceeded，且不写部分 K 线
回补成功后要求 data_quality 复检
回补失败时写 AlertEvent
不调用真实 Binance
不调用真实 Hermes
不调用大模型
不访问账户 / 仓位 / 订单 / 杠杆接口
不实现 WebSocket
不实现交易执行
```

如需真实集成测试，必须使用显式环境开关，并且默认关闭。

---

## 26. 验收标准

第一阶段完成后，数据回补模块必须满足：

```text
支持 BTCUSDT U 本位 4h 回补。
支持 BTCUSDT U 本位 1d 回补。
支持 initial_historical_backfill。
支持 gap_backfill。
支持 manual_range_backfill。
支持 conflict_recheck。
只使用 Binance REST 官方已收盘 K 线。
所有时间统一 UTC。
请求 K 线时不传 timeZone。
支持分批请求。
支持未收盘过滤。
支持幂等写入。
支持冲突不覆盖。
支持 dry-run。
支持 BackfillRequest 原子 claim。
支持 missing_open_times 精确回补。
支持分页和最大 bars 上限，超过限制不得部分写入。
回补与采集共用 K 线写入锁。
4h 回补写入 4h K 线表。
1d 回补写入 1d K 线表。
不创建单独正式回补 K 线表。
回补失败写 BackfillRun 状态；BackfillResult 如保留，仅作为结果摘要。
回补失败写 AlertEvent。
冲突写 DataConflict。
回补完成后标记 / 要求 DataQuality 复检；实际复检由后续统一编排层、recovery scan 或人工命令重新执行。
data_quality PASS 前不得进入 MarketSnapshot。
不调用 RiskCheck。
不调用 Execution。
不调用大模型。
不同步调用 Hermes。
不访问 Binance 交易类接口。
```

---

## 27. 总结

数据回补模块负责非正常实时采集路径下的补齐与复核。

第一阶段核心口径：

```text
回补只从 Binance REST 拉官方已收盘 K 线。
回补支持 4h 和 1d。
4h / 1d 分表写入。
回补不等于人工改数。
回补不得静默覆盖冲突 K 线。
回补必须幂等。
回补必须记录 BackfillRun / BackfillEvent；BackfillResult 如保留，仅作为 BackfillRun 的结果摘要语义。
回补失败、冲突、阻断必须写 AlertEvent。
回补和采集必须共用 K 线写入锁。
回补完成后标记 / 要求 DataQuality 复检；实际复检由后续统一编排层、recovery scan 或人工命令重新执行。
只有 data_quality PASS，后续链路才能继续。
```
