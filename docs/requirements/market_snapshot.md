# market_snapshot.md

# 行情快照模块需求

## 1. 文档目的

本文档定义中低频趋势跟踪自动交易系统的 MarketSnapshot 行情快照模块需求。

MarketSnapshot 是策略分析前的统一市场事实输入，用于固定某一轮分析所依赖的 4h 与 1d 行情窗口、数据质量结果、追溯信息和生成状态。

本文档只定义行情快照模块需求，不定义：

```text
具体数据库字段
具体 Django model
具体 Django management command 名称
具体 Celery task 名称
具体特征计算算法
具体原子信号算法
具体策略逻辑
具体风控逻辑
具体交易执行逻辑
具体 Hermes 发送实现
具体回测实现
```

MarketSnapshot 不是策略信号，不是交易建议，不是订单意图。

---

## 2. 模块目标

MarketSnapshot 模块的目标是：

```text
固定一轮分析使用的市场事实输入
统一 4h 与 1d 行情窗口
保证下游 FeatureLayer、AtomicSignal、StrategySignal、DecisionSnapshot 使用同一份行情证据
保证快照可追溯到 K 线、DataQualityResult、trace_id 和 trigger_source
防止策略层直接散乱查询 K 线表
防止策略层懒加载生成快照
防止坏数据进入特征、信号、策略和后续交易链路
```

第一阶段范围：

```text
交易所：Binance
市场类型：U 本位合约
交易品种：BTCUSDT
主策略周期：4h
大周期环境周期：1d
时间体系：UTC
```

说明：

```text
4h = 主策略周期
1d = 大周期趋势 / 市场环境辅助周期
MarketSnapshot 必须同时固定 4h 与 1d 的行情窗口
```

---

## 3. 核心定位

MarketSnapshot 是行情事实快照。

它回答：

```text
本轮分析使用哪一批 4h K 线？
本轮分析使用哪一批 1d K 线？
最新已收盘 4h K 线是哪一根？
当前理论最新已收盘 1d K 线是哪一根？
这些 K 线是否通过 data_quality？
后续特征、信号、策略、决策是否基于同一份行情证据？
未来复盘能否还原当时看到的市场数据？
```

MarketSnapshot 不回答：

```text
是否做多
是否做空
是否开仓
是否平仓
止盈是多少
止损是多少
仓位是多少
是否下单
```

---

## 4. 模块边界

### 4.1 MarketSnapshot 模块负责

```text
读取已通过 data_quality 的 4h K 线窗口
读取已通过 data_quality 的 1d K 线窗口
确认 4h 与 1d 新鲜度
确认 4h 与 1d 窗口数量满足要求
确认快照窗口可完整回查
确认 DataQualityResult = PASS
生成 MarketSnapshot
记录 4h / 1d 窗口起止时间
记录 4h / 1d actual_count
记录 4h / 1d lookback_count
记录 trace_id
记录 trigger_source
记录 snapshot status
记录 blocked / failed 原因
写入 AlertEvent
供 FeatureLayer 和后续链路读取
```

### 4.2 MarketSnapshot 模块不负责

```text
采集 K 线
回补 K 线
修复 K 线
修改 K 线主表
删除 K 线
重新实现 data_quality
请求 Binance REST
请求 Binance WebSocket
计算 FeatureValue
生成 AtomicSignal
生成 DomainSignal
生成 MarketRegime
生成 StrategySignal
生成 DecisionSnapshot
读取账户状态
读取仓位状态
调用 OrderPlan
生成 CandidateOrderIntent
生成 ApprovedOrderIntent
调用 RiskCheck
调用 ExecutionPreparation
调用 Execution
调用大模型
同步调用 Hermes Webhook
```

---

## 5. 第一阶段范围

### 5.1 P0 必须实现

```text
BTCUSDT 4h + 1d MarketSnapshot 生成
由编排层主动生成 MarketSnapshot
禁止策略层懒加载生成 MarketSnapshot
4h 新鲜度检查
1d 新鲜度检查
4h / 1d DataQualityResult = PASS 检查
4h / 1d DataQualityResult 覆盖窗口检查
4h / 1d lookback_count 配置
4h / 1d actual_count 记录
4h / 1d 窗口起止 open_time_utc 记录
MarketSnapshot 幂等生成或复用
CREATED / BLOCKED / FAILED 状态
blocked / failed 时写 AlertEvent
trace_id / trigger_source 记录
UTC 时间规则
```

### 5.2 P1 可预留

```text
人工命令生成或验证 MarketSnapshot
dry-run 快照检查
快照查询命令
快照复盘还原校验
快照生成失败聚合告警
快照生成运行统计
```

### 5.3 P2 后续再做

```text
多品种 MarketSnapshot
多交易所 MarketSnapshot
多周期扩展
执行级短周期快照
可视化快照查看
快照与回测批量生成
```

---

## 6. 快照生成方式

MarketSnapshot 不由策略模块懒加载生成。

禁止流程：

```text
StrategyService
→ 发现没有 MarketSnapshot
→ 临时生成 MarketSnapshot
→ 继续策略分析
```

正确流程：

```text
RunAnalysisCycleService 或等价编排服务
→ 确认 K 线采集完成
→ 确认 data_quality PASS
→ 生成或复用 MarketSnapshot
→ FeatureLayer
→ AtomicSignal
→ DomainSignal
→ MarketRegime
→ StrategySignal
→ DecisionSnapshot
```

规则：

```text
MarketSnapshot 是策略运行前的必备输入。
策略层只能读取已生成的 MarketSnapshot。
FeatureLayer、AtomicSignal、StrategySignal、DecisionSnapshot 不得自行创建 MarketSnapshot。
```

---

## 7. RunAnalysisCycleService 编排要求

RunAnalysisCycleService 是顺序编排服务，不是简单定时触发器。

无论是正常调度、手动触发、失败重试还是服务器重启恢复，都必须按以下顺序确认：

```text
1. 目标分析周期确定
2. K 线采集完成
3. data_quality PASS
4. MarketSnapshot 生成或复用
5. FeatureLayer
6. AtomicSignal
7. DomainSignal
8. MarketRegime
9. StrategySignal
10. DecisionSnapshot
11. OrderPlan / RiskCheck / ExecutionPreparation / Execution 链路
```

任何步骤缺失或失败，都不得跳过继续执行下游。

如果服务器宕机后重启，RunAnalysisCycleService 必须从缺失的最早安全步骤开始恢复。

示例：

```text
原计划 UTC 08:05 运行
服务器 10:30 才恢复
→ 检查 08:00 对应 4h 已收盘 K 线是否存在
→ 检查当前理论最新已收盘 1d K 线是否存在
→ 检查对应 data_quality 是否 PASS
→ 检查该周期 MarketSnapshot 是否已存在
→ 缺什么补什么
→ 不得直接跳过到策略层
```

---

## 8. 生成时机与新鲜度规则

### 8.1 4h 分析周期

MarketSnapshot 是 4h 分析周期的行情事实快照。

4h K 线收盘边界为 UTC：

```text
00:00
04:00
08:00
12:00
16:00
20:00
```

快照生成应发生在 4h K 线收盘后的受控分析窗口，例如：

```text
00:05
04:05
08:05
12:05
16:05
20:05
```

说明：

```text
具体延迟分钟数由 scheduler / 编排配置决定。
核心要求不是固定 :05，而是对应 4h K 线已经收盘、采集成功、data_quality PASS。
```

### 8.2 4h 新鲜度

每次生成 MarketSnapshot 前，必须确认：

```text
理论最新已收盘 4h K 线已经采集成功
理论最新已收盘 4h K 线 DataQualityResult = PASS
MarketSnapshot 使用的 latest_4h_open_time_utc 等于理论最新已收盘 4h open_time_utc
```

否则：

```text
MarketSnapshot = BLOCKED
写 AlertEvent
不进入 FeatureLayer
不进入策略链路
```

### 8.3 1d 新鲜度

1d 新鲜度必须按日线收盘边界单独判断。

不得用 4h 的收盘节奏推断 1d。

规则：

```text
1d 必须是当前时刻理论上最新的已收盘 1d K 线。
1d 不要求每个 4h 周期都有新 K 线。
1d 必须已采集成功。
1d 必须 DataQualityResult = PASS。
```

示例：

```text
UTC 2026-06-05 04:05
4h 最新已收盘 = 2026-06-05 00:00 这根 4h
1d 最新已收盘 = 2026-06-04 00:00 这根 1d
```

示例：

```text
UTC 2026-06-06 00:05
4h 最新已收盘 = 2026-06-05 20:00 这根 4h
1d 最新已收盘 = 2026-06-05 00:00 这根 1d
```

如果 UTC 00:05 后新的 1d 理论上已经收盘，但 1d 采集或 data_quality 未完成：

```text
MarketSnapshot = BLOCKED
写 AlertEvent
不允许沿用过期 1d 假装新鲜
```

### 8.4 4h 与 1d 分别计算理论最新已收盘 K 线

必须分别计算：

```text
4h theoretical_latest_closed_open_time_utc
1d theoretical_latest_closed_open_time_utc
```

禁止：

```text
用 4h 节奏推断 1d 是否应该更新。
用当前机器本地时间判断最新 K 线。
使用本地时间或 PRC 时间参与业务判断。
```

---

## 9. 前置条件

生成 CREATED 状态 MarketSnapshot 前必须满足：

```text
4h K 线表已初始化
1d K 线表已初始化
理论最新已收盘 4h K 线存在
当前理论最新已收盘 1d K 线存在
4h 目标窗口 DataQualityResult = PASS
1d 目标窗口 DataQualityResult = PASS
4h lookback 窗口数量满足要求
1d lookback 窗口数量满足要求
4h 窗口可按 UTC open_time 连续回查
1d 窗口可按 UTC open_time 连续回查
读取到的 4h K 线全部已收盘
读取到的 1d K 线全部已收盘
```

如果任一条件不满足：

```text
不得生成 CREATED 快照。
必须返回 BLOCKED 或 FAILED。
必须写 blocked_reason 或 error_message。
必要时写 AlertEvent。
```

---

## 10. 快照状态

MarketSnapshot 第一阶段状态包括：

```text
CREATED
BLOCKED
FAILED
```

### 10.1 CREATED

表示快照成功生成，且满足后续 FeatureLayer、信号和策略读取条件。

CREATED 快照必须满足：

```text
4h DataQualityResult = PASS
1d DataQualityResult = PASS
4h / 1d 窗口数量满足要求
4h / 1d 窗口可完整回查
4h / 1d 新鲜度满足要求
```

### 10.2 BLOCKED

表示前置数据条件不满足，当前流程被阻断。

常见原因：

```text
4h 未初始化
1d 未初始化
4h 最新已收盘 K 线缺失
1d 当前理论最新已收盘 K 线缺失
4h DataQualityResult 非 PASS
1d DataQualityResult 非 PASS
4h K 线数量不足
1d K 线数量不足
4h 窗口无法完整回查
1d 窗口无法完整回查
读取到未收盘 K 线
```

BLOCKED 不代表自动修复，不代表自动回补，也不代表策略失败。

### 10.3 FAILED

表示快照生成过程发生程序错误或存储错误。

常见原因：

```text
数据库查询失败
写入 MarketSnapshot 失败
JSON 序列化失败
幂等冲突无法处理
未预期异常
```

FAILED 必须记录 trace_id、error_code、error_message，并写 AlertEvent。

---

## 11. 快照内容

MarketSnapshot 必须记录以下信息：

```text
snapshot_id
exchange
market_type
symbol
base_timeframe = 4h
higher_timeframe = 1d
analysis_close_time_utc
status
blocked_reason
error_message
latest_4h_open_time_utc
latest_1d_open_time_utc
lookback_4h_count
lookback_1d_count
actual_4h_count
actual_1d_count
start_4h_open_time_utc
end_4h_open_time_utc
start_1d_open_time_utc
end_1d_open_time_utc
data_quality_result_4h_id
data_quality_result_1d_id
trace_id
trigger_source
created_at_utc
```

字段命名和具体表结构由后续 plans 与数据库设计确定，但上述语义必须可保存、可查询、可追溯。

---

## 12. Payload 规则

MarketSnapshot payload 只保存摘要、窗口索引和元数据。

允许包含：

```text
snapshot_id
symbol
base_timeframe
higher_timeframe
analysis_close_time_utc
latest_4h_open_time_utc
latest_1d_open_time_utc
lookback_4h_count
lookback_1d_count
actual_4h_count
actual_1d_count
start_4h_open_time_utc
end_4h_open_time_utc
start_1d_open_time_utc
end_1d_open_time_utc
data_quality_result ids
source table names
trace_id
trigger_source
```

禁止在 payload 中保存完整 K 线数组。

禁止在 payload 中保存逐根 OHLCV 明细：

```text
open
high
low
close
volume
quote_volume
trade_count
```

后续模块如需完整 K 线，应通过 MarketSnapshot 中记录的：

```text
exchange
market_type
symbol
timeframe
start_open_time_utc
end_open_time_utc
actual_count
```

回查正式 K 线表。

MarketSnapshot 不是第二份 K 线库。

---

## 13. 禁止包含策略内容

MarketSnapshot 禁止包含：

```text
trend = bullish / bearish
signal = long / short
entry_price
stop_loss
take_profit
position_size
leverage
开仓建议
平仓建议
止盈建议
止损建议
停止交易建议
策略评分
大模型解释
```

这些内容属于 FeatureLayer、AtomicSignal、StrategySignal、RiskCheck 或 Review，不属于 MarketSnapshot。

---

## 14. 幂等要求

同一分析周期应只有一个有效 CREATED MarketSnapshot。

推荐业务幂等键：

```text
exchange
market_type
symbol
base_timeframe
higher_timeframe
analysis_close_time_utc
```

规则：

```text
同一分析周期重复运行时，如果已存在 CREATED 快照，应复用。
不得重复生成多个有效 CREATED 快照。
如果已存在 BLOCKED / FAILED 快照，允许根据配置重新尝试。
重新尝试必须保留原失败记录，不能静默覆盖审计记录。
```

snapshot_id 应具备可读性和唯一性。

推荐语义：

```text
MS-BTCUSDT-4H-1D-YYYYMMDDTHHMMSSZ
```

具体格式可由后续实现确定。

---

## 15. 输入对象

MarketSnapshot 模块输入至少包括：

```text
exchange
market_type
symbol
base_timeframe
higher_timeframe
analysis_close_time_utc
lookback_4h_count
lookback_1d_count
trigger_source
trace_id
```

第一阶段固定：

```text
exchange = binance
market_type = usds_m_futures
symbol = BTCUSDT
base_timeframe = 4h
higher_timeframe = 1d
```

---

## 16. 输出对象

MarketSnapshot 模块输出：

```text
MarketSnapshot
AlertEvent
```

MarketSnapshot 可以是：

```text
CREATED
BLOCKED
FAILED
```

AlertEvent 只在 BLOCKED / FAILED 或无法确认状态时写入。

---

## 17. lookback_count 要求

lookback_count 必须可配置，不得硬编码死在业务逻辑中。

建议默认值：

```text
4h lookback_count = 500
1d lookback_count = 365
```

含义：

```text
4h 500 根约 83 天
1d 365 根约 1 年
```

规则：

```text
MarketSnapshot 必须记录实际使用的 lookback_count。
MarketSnapshot 必须记录实际读取到的 actual_count。
配置变化不得影响已生成快照的复盘解释。
```

如果实际数量不足：

```text
MarketSnapshot = BLOCKED
写 blocked_reason
写 AlertEvent
```

---

## 18. 成功流程

```text
RunAnalysisCycleService 触发
→ 计算 analysis_close_time_utc
→ 分别计算理论最新已收盘 4h / 1d open_time_utc
→ 确认 4h K 线存在
→ 确认 1d K 线存在
→ 确认 4h DataQualityResult = PASS
→ 确认 1d DataQualityResult = PASS
→ 读取 4h lookback 窗口
→ 读取 1d lookback 窗口
→ 确认 actual_count 满足要求
→ 确认窗口可完整回查
→ 生成 MarketSnapshot = CREATED
→ 返回 MarketSnapshot
→ FeatureLayer 读取该 MarketSnapshot
```

---

## 19. 阻断流程

```text
RunAnalysisCycleService 触发
→ MarketSnapshot 前置检查失败
→ 写 MarketSnapshot = BLOCKED 或返回 blocked 结果
→ 写 AlertEvent
→ 阻断当前流程
→ 不进入 FeatureLayer
→ 不进入 AtomicSignal
→ 不进入 StrategySignal
→ 不进入 DecisionSnapshot
→ 不进入 OrderPlan / RiskCheck / ExecutionPreparation / Execution 链路
```

阻断后不得自动：

```text
请求 Binance
触发 data_backfill
修复 K 线
生成策略信号
生成交易意图
```

---

## 20. 失败流程

```text
MarketSnapshot 生成过程异常
→ 写 MarketSnapshot = FAILED 或失败记录
→ 写 AlertEvent
→ 阻断当前流程
→ 不进入下游
```

失败必须记录：

```text
trace_id
trigger_source
error_code
error_message
analysis_close_time_utc
```

---

## 21. AlertEvent 规则

MarketSnapshot 在以下情况必须写入 AlertEvent：

```text
4h 数据未初始化
1d 数据未初始化
4h 最新已收盘 K 线缺失
1d 当前理论最新已收盘 K 线缺失
4h DataQualityResult 非 PASS
1d DataQualityResult 非 PASS
4h K 线数量不足
1d K 线数量不足
窗口无法完整回查
快照写入失败
快照生成异常
无法确认快照状态
```

AlertEvent 由 notifications / Hermes 服务异步消费。

MarketSnapshot 模块不直接同步调用 Hermes。

Hermes 发送失败不得改变 MarketSnapshot 状态。

---

## 22. 与 data_collection 的关系

data_collection 负责采集 4h / 1d K 线。

MarketSnapshot 不触发采集，不请求 Binance。

如果 K 线未采集完成：

```text
MarketSnapshot = BLOCKED
写 AlertEvent
等待 data_collection 完成，或等待 data_backfill 完成 BackfillRun 并标记 / 要求 DataQuality 复检
后续统一编排层、recovery scan 或人工命令重新执行 DataQuality
新的 DataQualityResult = PASS 且 allows_downstream=True 后，才允许继续 MarketSnapshot
```

---

## 23. 与 data_quality 的关系

MarketSnapshot 只能基于 DataQualityResult = PASS 的 K 线生成。

MarketSnapshotService 只读取 / 校验已有 DataQualityResult。

如果缺少可用 DataQualityResult，或结果不覆盖目标窗口，则 MarketSnapshot 必须 BLOCKED，由后续统一编排层先运行 DataQuality。

规则：

```text
4h DataQualityResult 必须 PASS。
1d DataQualityResult 必须 PASS。
4h 与 1d 任一周期非 PASS，MarketSnapshot 必须 BLOCKED。
```

MarketSnapshot 不重新实现完整数据质量检查。

MarketSnapshot 只做前置确认：

```text
确认 PASS 记录存在
确认 PASS 记录覆盖目标窗口
确认窗口数量满足要求
确认窗口可完整回查
```

---

### 23.1 DataQualityResult 覆盖校验细则

MarketSnapshot 绑定的 4h / 1d DataQualityResult 不能只满足 `PASS`，还必须覆盖本次快照目标窗口。

每个 timeframe 对应的 DataQualityResult 必须满足：

```text
status = PASS
allows_downstream = True
exchange / market_type / symbol / timeframe 与快照目标一致
lookback_limit >= MarketSnapshot 对应 lookback_count
actual_count >= MarketSnapshot 对应 lookback_count
actual_start_time_utc <= MarketSnapshot 对应窗口 start_open_time_utc
actual_end_time_utc >= MarketSnapshot 对应窗口 end close boundary
```

规则：

```text
只检查最近 50 根的 DataQualityResult 不得授权需要 500 根 4h 的 MarketSnapshot。
只覆盖短窗口的 DataQualityResult 不得授权更长 lookback 的 MarketSnapshot。
4h 和 1d 必须分别满足覆盖校验。
任一周期覆盖不足时，MarketSnapshot 必须 BLOCKED，并写 AlertEvent。
```

说明：

```text
DataQualityResult 是 MarketSnapshot 的质量授权边界。
MarketSnapshot 不重新执行完整 data_quality，但必须确认质量结果覆盖自己实际使用的窗口。
```

## 24. 与 data_backfill 的关系

MarketSnapshot 不触发 data_backfill。

如果发现缺失、滞后、数量不足或质量未 PASS：

```text
MarketSnapshot = BLOCKED
写 AlertEvent
由人工或后续编排决定是否触发 data_backfill
```

MarketSnapshot 不得自动补数据。

---

## 25. 与 FeatureLayer 的关系

FeatureLayer 必须读取 MarketSnapshot。

FeatureLayer 不得直接绕过 MarketSnapshot 去散乱查询 K 线表。

FeatureLayer 不得自行生成 MarketSnapshot。

FeatureLayer 计算结果必须能追溯到 MarketSnapshot。

---

## 26. 与策略链路的关系

AtomicSignal、DomainSignal、MarketRegime、StrategySignal 和 DecisionSnapshot 必须基于同一个 MarketSnapshot。

策略层不得懒加载生成 MarketSnapshot。

策略层不得直接查询 K 线表来替代 MarketSnapshot。

如果 MarketSnapshot 不存在或状态非 CREATED：

```text
策略链路不得运行。
```

---

## 27. 与交易和风控的关系

MarketSnapshot 不参与交易和风控。

MarketSnapshot 不调用：

```text
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
账户接口
仓位接口
订单接口
杠杆接口
保证金接口
```

如果 MarketSnapshot BLOCKED / FAILED，本轮流程不会进入策略和交易链路。

---

## 28. 与回测的关系

回测应尽量复用 MarketSnapshot 的语义。

回测中的 MarketSnapshot 也应固定：

```text
4h 窗口
1d 窗口
analysis_close_time_utc
lookback_count
actual_count
参数版本或快照配置
```

回测不得使用实盘当时不可获得的数据。

回测不得使用未收盘 K 线生成快照。

回测不得绕过 MarketSnapshot 直接让策略散乱查询未来数据。

---

## 29. 重启恢复要求

服务器宕机或任务失败后，系统恢复时不得跳过 MarketSnapshot 前置链路。

重启后 RunAnalysisCycleService 必须检查：

```text
目标 analysis_close_time_utc 是否已存在有效 MarketSnapshot
目标 4h K 线是否已采集
当前理论最新 1d K 线是否已采集
对应 data_quality 是否 PASS
后续 FeatureLayer / StrategySignal / DecisionSnapshot 是否已完成
```

恢复规则：

```text
已有 CREATED MarketSnapshot → 复用，不重复生成新的有效快照。
缺少 K 线 → 先采集或回补，再 data_quality，再 MarketSnapshot。
data_quality FAIL → 阻断，不生成 MarketSnapshot。
MarketSnapshot 已生成但下游未完成 → 从 MarketSnapshot 后的下一步恢复。
```

禁止：

```text
因为当前时间已经晚于计划时间，就跳过采集和 data_quality。
因为服务器重启，就直接生成策略信号。
因为已有旧快照，就忽略当前 analysis_close_time_utc。
```

---

## 30. 禁止事项

MarketSnapshot 模块禁止：

```text
策略懒加载生成快照
请求 Binance REST
请求 Binance WebSocket
写入 KlineRecord
修改 K 线主表
删除 K 线
触发 data_backfill
重新实现 data_quality
基于非 PASS 数据生成 CREATED 快照
只用 4h 生成快照并假装 1d 已满足
用过期 1d 假装新鲜
保存完整 K 线数组到 payload
保存逐根 OHLCV 明细到 payload
生成 FeatureValue
生成 AtomicSignal
生成 StrategySignal
生成 DecisionSnapshot
生成交易建议
生成 CandidateOrderIntent
生成 ApprovedOrderIntent
调用 OrderPlan
调用 RiskCheck
调用 ExecutionPreparation
调用 Execution
调用大模型
同步调用 Hermes Webhook
使用本地时间参与业务判断
请求 K 线时传 timeZone 参数
```

---

## 31. 测试要求

默认测试不得依赖真实外部服务。

测试应覆盖：

```text
4h + 1d 均 PASS 时生成 CREATED 快照
4h 缺失时 BLOCKED
1d 缺失时 BLOCKED
4h DataQualityResult 非 PASS 时 BLOCKED
1d DataQualityResult 非 PASS 时 BLOCKED
4h DataQualityResult 覆盖窗口不足时 BLOCKED
1d DataQualityResult 覆盖窗口不足时 BLOCKED
4h 数量不足时 BLOCKED
1d 数量不足时 BLOCKED
4h 窗口无法完整回查时 BLOCKED
1d 窗口无法完整回查时 BLOCKED
非 4h 收盘后受控窗口不生成可用快照
1d 新鲜度按日线边界单独判断
同一 analysis_close_time_utc 重复运行复用已有 CREATED 快照
BLOCKED / FAILED 写 AlertEvent
payload 不包含完整 K 线数组
payload 不包含 OHLCV 明细数组
不请求 Binance
不写 K 线表
不调用 RiskCheck
不调用 ExecutionPreparation / Execution
不调用大模型
服务器重启恢复时按顺序检查前置条件
```

---

## 32. 验收标准

第一阶段完成后，MarketSnapshot 模块必须满足：

```text
可以为 BTCUSDT 生成 4h + 1d MarketSnapshot。
MarketSnapshot 由编排层主动生成，不由策略层懒加载。
MarketSnapshot 必须基于 4h / 1d DataQualityResult = PASS。
MarketSnapshot 必须确认 4h / 1d DataQualityResult 覆盖目标 lookback 窗口。
短窗口 DataQualityResult 不得授权长窗口 MarketSnapshot。
MarketSnapshot 必须同时包含 4h 和 1d 窗口索引。
MarketSnapshot 能记录 latest_4h_open_time_utc。
MarketSnapshot 能记录 latest_1d_open_time_utc。
MarketSnapshot 能记录 lookback_count 和 actual_count。
MarketSnapshot 能追溯 trace_id 和 trigger_source。
MarketSnapshot payload 不保存完整 K 线数组。
MarketSnapshot 不请求 Binance。
MarketSnapshot 不写 K 线表。
MarketSnapshot 不触发回补。
MarketSnapshot 不生成策略信号。
MarketSnapshot 不生成交易意图。
MarketSnapshot 不调用 RiskCheck。
MarketSnapshot 不调用 ExecutionPreparation / Execution。
MarketSnapshot BLOCKED / FAILED 时写 AlertEvent。
MarketSnapshot CREATED 后才允许 FeatureLayer 运行。
服务器重启后仍严格按采集 → data_quality → MarketSnapshot 顺序恢复。
```

---

## 33. 总结

MarketSnapshot 是策略分析前的统一行情证据包。

第一阶段核心口径：

```text
MarketSnapshot 固定 4h + 1d 行情事实。
MarketSnapshot 不由策略懒加载生成。
MarketSnapshot 由 RunAnalysisCycleService 或等价编排层在 data_quality PASS 后生成。
4h 必须是最新已收盘主周期 K 线。
1d 必须是当前理论最新已收盘日线。
4h 和 1d 新鲜度分别判断。
任一周期不满足要求，MarketSnapshot 必须 BLOCKED。
MarketSnapshot 不请求 Binance、不回补、不写 K 线表。
MarketSnapshot payload 只保存窗口索引和摘要，不保存完整 K 线数组。
FeatureLayer、信号、策略、DecisionSnapshot 必须基于同一个 MarketSnapshot。
服务器重启后必须按顺序恢复，不能跳过采集、质量检查和快照。
```
