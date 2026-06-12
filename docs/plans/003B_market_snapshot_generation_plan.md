# Plan 003B：MarketSnapshot 生成服务

## 0. 执行边界

本 plan 是 003A 之后的 MarketSnapshot 生成阶段。

003A 已完成：

```text
apps.market_snapshot
MarketSnapshot model
snapshot_key
DataQualityResult 绑定字段
4h / 1d 窗口字段
payload_schema_version
payload JSONField
created / blocked / failed 状态契约
```

本阶段负责真正生成 MarketSnapshot。

本阶段只允许：

```text
读取 DataQualityResult
MarketSnapshotService 只读取 / 校验已有 DataQualityResult
MarketSnapshotService 不触发 DataQuality，不调用 DataQualityService
读取 Kline4h / Kline1d
检查 DataQualityResult 是否 PASS 且覆盖 snapshot 窗口
检查 KlineWriteLock 是否占用
组装 MarketSnapshot payload
幂等写入 MarketSnapshot
blocked / failed 时写 AlertEvent
提供 management command
提供 tests
提供 implementation 文档
```

本阶段不负责 Feature、Signal、Strategy、Risk、Trading。

---

## 1. 阶段目标

本阶段完成后，项目应具备：

```text
MarketSnapshotService
MarketSnapshot payload builder
snapshot_key 生成逻辑
已有 DataQualityResult 读取 / 校验能力
DataQualityResult 覆盖窗口校验
Kline4h / Kline1d 窗口读取能力
KlineWriteLock 检查
created / blocked / failed 结果处理
AlertEvent 写入
build_market_snapshot management command
tests
implementation 文档
```

本阶段的核心目标是：

```text
把已经通过 data_quality 的 4h + 1d K 线窗口，固化成一个可复现、可审计、可被后续 Feature / Strategy 读取的 MarketSnapshot。
```

---

## 2. 依赖文档

Codex 开始前必须阅读：

```text
AGENTS.md
docs/rules/project_invariants.md
docs/requirements/01_project_scope.md
docs/requirements/02_system_capabilities.md
docs/requirements/market_snapshot.md
docs/requirements/data_quality.md
docs/architecture/data_flow.md
docs/implementation/002A_market_data_models.md
docs/implementation/002B_data_collection_gateway_and_service.md
docs/implementation/002C_data_quality.md
docs/implementation/002D_data_backfill.md
docs/implementation/003A_market_snapshot_model.md
docs/plans/003B_market_snapshot_generation_plan.md
```

如果本 plan 与 `AGENTS.md` 或 `docs/rules/project_invariants.md` 冲突，必须以后两者为准，并在总结中说明冲突。

如果本 plan 与 requirements 或 architecture 冲突，必须停止并提示用户确认。

---

## 3. 本阶段明确不做

本阶段禁止实现：

```text
Binance REST client
Binance serverTime 请求
Binance Kline 请求
WebSocket
data_collection service
data_backfill service
BackfillRequest 创建
BackfillRequest 执行
BackfillRun / BackfillResult 修改
FeatureLayer
AtomicSignal
DomainSignal
StrategySignal
MarketRegime
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
写 Kline4h / Kline1d
修改 Kline4h / Kline1d
删除 Kline4h / Kline1d
覆盖 OHLCV
修复 K 线
执行回补
保存完整 K 线数组到 snapshot payload
在 payload 中保存 OHLCV 明细数组
在 payload 中保存策略结论
在 payload 中保存交易建议
```

本阶段测试不得访问真实 Binance、真实 Hermes、生产 MySQL、生产 Redis。

---

## 4. 默认窗口要求

本阶段默认 MarketSnapshot 窗口为：

```text
MARKET_SNAPSHOT_4H_LOOKBACK_COUNT = 500
MARKET_SNAPSHOT_1D_LOOKBACK_COUNT = 365
```

含义：

```text
4h 500 根，约 83 天
1d 365 根，约 1 年
```

不得把这些数字写死在 service 逻辑中。

必须支持 settings 配置，并允许 management command 参数覆盖：

```bash
python manage.py build_market_snapshot --lookback-4h 500 --lookback-1d 365
```

建议新增 settings：

```text
MARKET_SNAPSHOT_4H_LOOKBACK_COUNT = 500
MARKET_SNAPSHOT_1D_LOOKBACK_COUNT = 365
MARKET_SNAPSHOT_MAX_LOOKBACK_COUNT = 5000
MARKET_SNAPSHOT_PAYLOAD_SCHEMA_VERSION = market_snapshot.v1
```

规则：

```text
lookback_4h_count 必须为正整数
lookback_1d_count 必须为正整数
lookback 不得超过 MARKET_SNAPSHOT_MAX_LOOKBACK_COUNT
```

---

## 5. data_quality 前置规则

MarketSnapshot 必须绑定授权本次快照生成的 DataQualityResult。

正常 analysis cycle 中，应先运行：

```text
data_quality latest_n 4h limit=500
data_quality latest_n 1d limit=365
```

然后 MarketSnapshot 绑定这两条结果：

```text
quality_result_4h = 本轮 pre-snapshot 4h DataQualityResult
quality_result_1d = 本轮 pre-snapshot 1d DataQualityResult
```

本阶段允许两种输入方式。

### 5.1 显式传入 DataQualityResult id

Service / command 可以接收：

```text
quality_result_4h_id
quality_result_1d_id
```

这种方式优先。

### 5.2 缺少可用 DataQualityResult 时 blocked

如果没有显式传入 quality result id，或传入的 DataQualityResult 不可用 / 不覆盖目标窗口，MarketSnapshotService 不得调用 DataQualityService 生成检查结果。

MarketSnapshotService 必须生成 blocked 结果，并提示需要由后续统一编排层先运行 DataQuality：

```text
missing_quality_result
quality_result_not_cover_window
requires_data_quality_check
```

注意：

```text
MarketSnapshotService 只读取 / 校验已有 DataQualityResult
MarketSnapshotService 不触发 DataQuality
MarketSnapshotService 不调用 DataQualityService
但不得复制 data_quality 检查逻辑
不得自己创建 DataQualityResult / DataQualityIssue
不得自己创建 BackfillRequest
```

---

## 6. DataQualityResult 覆盖校验

MarketSnapshot 不能随便使用最近一条 PASS。

必须校验 DataQualityResult：

```text
status = PASS
allows_downstream = True
timeframe 匹配
symbol 匹配
exchange / market_type 匹配
lookback_limit >= snapshot lookback_count
actual_count >= snapshot lookback_count
actual_start_time_utc <= snapshot_start_time_utc
actual_end_time_utc >= snapshot_end_time_utc
```

如果任一不满足：

```text
MarketSnapshot.status = blocked
MarketSnapshot.is_usable = False
blocked_reason = quality_result_not_covering_snapshot_window 或 quality_result_failed
不生成 usable snapshot
```

每日默认 50 根 data_quality 结果不得授权 500 / 365 的 MarketSnapshot，除非它的实际覆盖范围满足上述要求。

---

## 7. KlineWriteLock 检查

正常流程中，MarketSnapshot 应只在 data_collection 完成且 data_quality PASS 后生成。

如果此前 DataQuality 创建了 BackfillRequest，且该请求已由 data_backfill claim 并执行回补，则必须等待 BackfillRun 完成；随后由后续统一编排层重新执行 DataQuality。只有新的 DataQualityResult = PASS 且 allows_downstream=True，才允许生成 MarketSnapshot。

KlineWriteLock 检查不是正常流程依赖，而是防御性保护。

如果生成快照时发现同一 symbol/timeframe 仍有 active 且未过期的 KlineWriteLock，说明当前存在异常并发、手动误触发或编排错误。此时 MarketSnapshot 必须 blocked，并写 AlertEvent。

生成 snapshot 前必须检查对应 K 线写入锁。

需要检查：

```text
KlineWriteLock for binance:usds_m_futures:BTCUSDT:4h
KlineWriteLock for binance:usds_m_futures:BTCUSDT:1d
```

如果任一锁 active 且未过期：

```text
MarketSnapshot.status = blocked
MarketSnapshot.is_usable = False
blocked_reason = kline_write_lock_active
不读取半更新窗口作为 usable snapshot
```

不得绕过该检查。

dry-run 默认不写 AlertEvent，除非显式要求通知。

## 8. Kline 窗口读取

本阶段允许读取：

```text
Kline4h
Kline1d
```

读取规则：

```text
按 exchange + market_type + symbol 过滤
按 open_time_utc 倒序取最近 N 根
再按 open_time_utc 升序用于窗口摘要
```

必须确认：

```text
actual_4h_count == lookback_4h_count
actual_1d_count == lookback_1d_count
```

如果数量不足：

```text
MarketSnapshot.status = blocked
blocked_reason = insufficient_kline_count
```

003B 不重新实现完整 data_quality，但可以做轻量一致性断言：

```text
查询数量是否满足要求
窗口 start / end 是否存在
quality result 是否覆盖该窗口
```

不得重新实现：

```text
缺失检查
连续性检查
OHLCV 检查
BackfillRequest 创建
```

这些属于 002C data_quality。

---

## 9. snapshot_key 生成与幂等

MarketSnapshot 必须使用 snapshot_key 作为业务唯一 ID。

snapshot_key 不等于数据库自增 id。

后续 Feature / Signal / Strategy / Decision 应通过 ForeignKey 绑定 MarketSnapshot，并可冗余记录 snapshot_key 用于日志和审计。

003B 必须生成稳定 snapshot_key。

建议基于以下内容生成：

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
payload_schema_version
```

建议使用 SHA-256 hex digest，并保存为 64 字符。

规则：

```text
同一批事实输入重复生成时 snapshot_key 相同
如果 snapshot_key 已存在，返回已有 snapshot
不得无限插入重复 MarketSnapshot
```

如果已有 snapshot 是 blocked / failed，是否复用也必须确定。

第一阶段建议：

```text
created snapshot 使用确定性 snapshot_key 并复用。
blocked / failed snapshot 可以记录审计，但不得被下游使用；如使用 snapshot_key，也必须确保不会覆盖 created snapshot。
```

---

## 10. market_as_of_utc

MarketSnapshot 必须区分：

```text
created_at_utc
market_as_of_utc
latest_4h_open_time_utc
latest_1d_open_time_utc
```

含义：

```text
created_at_utc = 系统创建 snapshot 的时间
market_as_of_utc = snapshot 代表的市场事实截止时间
```

第一阶段采用：

```text
market_as_of_utc = latest_4h.close_time_utc
```

原因：

```text
4h 是主分析周期，snapshot 通常由 4h 新收盘驱动。
```

implementation 文档必须明确写入该规则。

---

## 11. payload 要求

payload 必须使用 `payload_schema_version`：

```text
market_snapshot.v1
```

payload 允许保存：

```text
schema_version
snapshot_key
exchange
market_type
symbol
base_timeframe
higher_timeframe
lookback_4h_count
lookback_1d_count
actual_4h_count
actual_1d_count
start_4h_open_time_utc
end_4h_open_time_utc
latest_4h_open_time_utc
start_1d_open_time_utc
end_1d_open_time_utc
latest_1d_open_time_utc
quality_result_4h_id
quality_result_1d_id
source_tables
boundary flags
```

payload 禁止包含：

```text
完整 Kline4h 数组
完整 Kline1d 数组
OHLCV 明细数组
open/high/low/close/volume/trade_count 序列
趋势判断
做多 / 做空
entry_price
stop_loss
take_profit
position_size
leverage
stop_trading
RiskCheck
CandidateOrderIntent / ApprovedOrderIntent
Execution
大模型分析结果
```

MarketSnapshot 不是第二份 K 线库。

后续 Feature / Strategy 如需完整 K 线，应基于 snapshot 记录的窗口范围回查 Kline4h / Kline1d。

---

## 12. 状态语义

MarketSnapshot status 至少支持：

```text
created
blocked
failed
```

### 12.1 created

条件：

```text
4h quality PASS
1d quality PASS
quality 覆盖 snapshot 窗口
KlineWriteLock 未占用
4h Kline 数量满足 lookback
1d Kline 数量满足 lookback
snapshot 写入成功
```

要求：

```text
is_usable = True
```

### 12.2 blocked

表示前置条件不满足，例如：

```text
quality_result_failed
quality_result_missing
quality_result_not_covering_snapshot_window
kline_write_lock_active
insufficient_kline_count
invalid_lookback
```

要求：

```text
is_usable = False
blocked_reason 必须非空
```

### 12.3 failed

表示程序异常或存储异常，例如：

```text
database_error
payload_serialization_error
unexpected_exception
```

要求：

```text
is_usable = False
error_code 或 error_message 必须非空
```

---

## 13. AlertEvent 要求

本阶段可以写 AlertEvent，但不得直接调用 Hermes client，不得直接调用 notifications service。

正确链路：

```text
market_snapshot
→ 写 AlertEvent
→ notifications service 异步消费
→ Hermes client 发送
```

必须写 AlertEvent 的场景：

```text
MarketSnapshot blocked
MarketSnapshot failed
```

created 默认不写 AlertEvent，避免通知噪音。

AlertEvent 内容必须明确：

```text
symbol
timeframes
status
blocked_reason 或 error_code
trace_id
本事件不是交易建议
```

不得输出完整 payload 或 K 线数组。

---

## 14. Service 结构建议

建议新增：

```text
apps/market_snapshot/services.py
apps/market_snapshot/builders.py
apps/market_snapshot/selectors.py
```

职责：

### services.py

```text
MarketSnapshotService 主入口
协调 quality result
检查 lock
读取 Kline
调用 builder
写入 MarketSnapshot
写 AlertEvent
```

### builders.py

```text
组装 snapshot_key
组装 payload
计算 market_as_of_utc
```

### selectors.py

```text
读取 Kline4h / Kline1d 窗口
读取 / 校验 DataQualityResult
检查 KlineWriteLock
```

如果 Codex 判断拆分文件过多，也可以先放在 services.py，但必须保持入口层轻量，核心逻辑清晰可测试。

---

## 15. Management command 要求

新增：

```bash
python manage.py build_market_snapshot
```

至少支持参数：

```text
--lookback-4h
--lookback-1d
--quality-result-4h-id
--quality-result-1d-id
--trigger-source manual|analysis_cycle|scheduled
--trace-id
--dry-run
```

规则：

```text
command 只解析参数并调用 MarketSnapshotService
不承载核心业务逻辑
dry-run 不写 MarketSnapshot
dry-run 不写 AlertEvent，除非显式指定 notify 参数
```

输出摘要：

```text
status
snapshot_id
snapshot_key
is_usable
lookback_4h_count
lookback_1d_count
actual_4h_count
actual_1d_count
latest_4h_open_time_utc
latest_1d_open_time_utc
blocked_reason
error_code
trace_id
```

---

## 16. Dry-run 要求

dry-run 必须支持：

```text
执行 quality result 校验
执行 KlineWriteLock 检查
读取 Kline 窗口
组装 payload 摘要
生成 snapshot_key
不写 MarketSnapshot
不写 AlertEvent
不修改任何 Kline
```

dry-run 应返回结构化结果。

---

## 17. 数据库与 migration 要求

本阶段原则上不需要新增 MarketSnapshot 字段。

如果确实需要调整 003A 模型，必须：

```text
说明原因
使用 Django migration
不得破坏已有测试
不得大规模修改 003A
```

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

## 18. 测试要求

必须测试：

```text
quality_result_4h / quality_result_1d 均 PASS 且覆盖窗口时生成 created snapshot
created snapshot is_usable=True
quality_result_4h FAIL 时 blocked
quality_result_1d FAIL 时 blocked
quality result lookback_limit 不足时 blocked
quality result actual window 不覆盖 snapshot window 时 blocked
KlineWriteLock 4h active 时 blocked
KlineWriteLock 1d active 时 blocked
4h Kline 数量不足时 blocked
1d Kline 数量不足时 blocked
snapshot_key 幂等，相同输入返回已有 snapshot
payload_schema_version = market_snapshot.v1
payload 不包含完整 Kline 数组
payload 不包含 OHLCV 明细数组
payload 不包含策略 / 风控 / 交易字段
blocked 写 AlertEvent
failed 写 AlertEvent
created 默认不写 AlertEvent
dry-run 不写 MarketSnapshot
dry-run 不写 AlertEvent
management command 可运行
不会请求 Binance
不会写 Kline4h / Kline1d
不会调用 data_backfill
不会实现 Feature / Strategy / Risk / Trading
```

测试中允许创建 Kline4h / Kline1d 测试数据。

生产 service 不得写 Kline4h / Kline1d。

---

## 19. 越界检查

Codex 完成后建议运行：

```bash
grep -R "requests.get" apps/market_snapshot tests/market_snapshot
grep -R "httpx" apps/market_snapshot tests/market_snapshot
grep -R "Binance" apps/market_snapshot tests/market_snapshot
grep -R "DataCollectionService" apps/market_snapshot tests/market_snapshot
grep -R "BackfillService" apps/market_snapshot tests/market_snapshot
grep -R "MarketRegime" apps/market_snapshot tests/market_snapshot
grep -R "StrategySignal" apps/market_snapshot tests/market_snapshot
grep -R "RiskCheck" apps/market_snapshot tests/market_snapshot
grep -R "CandidateOrderIntent / ApprovedOrderIntent" apps/market_snapshot tests/market_snapshot
grep -R "Execution" apps/market_snapshot tests/market_snapshot
grep -R "create_order" apps/market_snapshot tests/market_snapshot
grep -R "place_order" apps/market_snapshot tests/market_snapshot
```

如果命中，Codex 必须解释原因。

---

## 20. implementation 文档要求

完成后必须新增：

```text
docs/implementation/003B_market_snapshot_generation.md
```

内容至少包括：

```text
本阶段新增了哪些 service / command
MarketSnapshotService 流程
DataQualityResult 绑定规则
快照前 DataQualityResult 绑定规则
quality 覆盖窗口校验规则
KlineWriteLock 检查规则
Kline 窗口读取规则
snapshot_key 生成规则
market_as_of_utc 规则
payload v1 schema
created / blocked / failed 规则
AlertEvent 写入规则
dry-run 行为
测试方式
本阶段明确未实现内容
```

必须明确写入：

```text
本阶段未请求 Binance。
本阶段未修改 Kline4h / Kline1d。
本阶段未执行 data_backfill。
本阶段未实现 FeatureLayer。
本阶段未实现策略、风控、交易。
本阶段未直接调用 Hermes webhook。
```

---

## 21. 人工审查清单

用户合并前应检查：

```text
是否只实现 MarketSnapshot 生成服务
是否没有请求 Binance
是否没有写 Kline4h / Kline1d
是否没有调用 data_backfill 执行
是否没有 Feature / Strategy / Risk / Trading
是否默认 4h lookback=500
是否默认 1d lookback=365
是否快照前 data_quality 覆盖快照窗口
是否绑定 quality_result_4h / quality_result_1d
是否检查 KlineWriteLock
是否 snapshot_key 幂等
是否 blocked / failed 写 AlertEvent
是否 payload 不保存完整 K 线数组
是否测试通过
```

---

## 22. Codex 输出总结要求

Codex 完成后必须输出：

```text
修改 / 新增文件清单
新增 service / command 名称
是否新增 migration
DataQualityResult 绑定规则
快照前 DataQualityResult 绑定规则
quality 覆盖窗口校验规则
KlineWriteLock 检查规则
snapshot_key 生成规则
market_as_of_utc 规则
payload schema 说明
AlertEvent 写入规则
dry-run 行为
测试命令和结果
无法运行的命令及原因
是否存在未完成项
```

并明确说明：

```text
本阶段未请求 Binance。
本阶段未修改 Kline4h / Kline1d。
本阶段未执行 data_backfill。
本阶段未实现 Feature / Strategy / Risk / Trading。
本阶段未直接调用 Hermes webhook。
```

---

## 23. 最终验收标准

本阶段完成后，必须满足：

```text
MarketSnapshotService 存在。
build_market_snapshot command 存在。
默认 4h lookback=500。
默认 1d lookback=365。
快照前 DataQualityResult 覆盖 snapshot 窗口。
MarketSnapshot 绑定 quality_result_4h / quality_result_1d。
DataQualityResult FAIL 会 blocked。
DataQualityResult 覆盖不足会 blocked。
KlineWriteLock active 会 blocked。
Kline 数量不足会 blocked。
created snapshot is_usable=True。
blocked / failed snapshot is_usable=False。
snapshot_key 幂等。
payload_schema_version = market_snapshot.v1。
payload 不含完整 K 线数组。
blocked / failed 写 AlertEvent。
created 默认不写 AlertEvent。
dry-run 不写库。
不请求 Binance。
不写 Kline4h / Kline1d。
不执行 data_backfill。
不实现 Feature / Strategy / Risk / Trading。
测试通过。
```

---

## 24. 下一阶段

本阶段完成并验收后，下一阶段才开始：

```text
004A_feature_layer_plan.md
```

004A 才允许实现：

```text
基于 MarketSnapshot 读取 Kline 窗口
计算基础 feature
为 AtomicSignal / StrategySignal 提供输入
```
