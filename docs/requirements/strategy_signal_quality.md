# 005B StrategySignalQuality / StrategyValidation 需求文档

> 建议落地路径：`docs/requirements/strategy_signal_quality.md`  
> 上游：005A `StrategySignal`  
> 下游：后续 `DecisionSnapshot`  
> 定位：质量验证层，不是策略生成层、风控层、执行层、收益回测层。

---

## 1. 模块定位

005B 负责验证 005A 生成的 `StrategySignal` 是否：

```text
可信
完整
可追溯
数值合法
证据充分
未过期
适合进入后续 DecisionSnapshot 流程
```

005B 不重新生成策略信号，不改变 005A 的原始输出，不判断策略是否赚钱，也不生成交易计划。

核心链路：

```text
AtomicSignalSet / AtomicSignalValue
→ 005A StrategySignal
→ 005B StrategySignalQuality / StrategyValidation
→ DecisionSnapshot
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheck
```

StrategySignalQuality 只负责校验 StrategySignal 是否可进入 DecisionSnapshot。
StrategySignalQuality 不直接进入 RiskCheck。
RiskCheck 只能消费 CandidateOrderIntent。

005B 的核心问题是：

```text
这个 StrategySignal 是否满足质量闸门，是否允许后续模块继续读取？
```

---

## 2. 职责边界

### 2.1 005B 可以做

```text
结构完整性检查
数值合法性检查
可追溯性检查
证据完整性检查
快照一致性检查
数据新鲜度检查
质量结果记录
质量问题记录
质量阻断
AlertEvent 告警
management command
测试
```

### 2.2 005B 不得做

```text
重新聚合 AtomicSignal
重新选择 StrategyDefinition
重新执行 StrategyRouter
重新计算 Feature
重新读取 Kline
读取交易所盘口 / 订单簿
读取账户余额
读取当前持仓
生成 entry_price
生成 stop_loss
生成 take_profit
生成 position_size
生成 leverage
生成 NO_TRADE / HOLD
生成任何订单意图
下单
回测收益
Sharpe / 最大回撤
参数扫描
模拟盘
UI / dashboard
```

### 2.3 不能修改 StrategySignal 原始结果

005B 不允许把 005A 的结果原地改成：

```text
direction = neutral
strength = 0
confidence = 0
```

质量处理必须写入独立的质量结果表。正确方式是：

```text
StrategySignal 保持原样
StrategySignalQualityResult 记录 passed / warning / failed
StrategySignalQualityResult.allows_downstream 决定后续是否可继续
StrategySignalQualityIssue 记录具体问题
```

原因：如果 005B 修改 StrategySignal 原文，后续复盘会无法区分“005A 原始策略输出”和“005B 质量处理结果”。

---

## 3. 输入与读取边界

005B 允许读取：

```text
StrategySignal
StrategyDefinition
StrategyRouteRule
AtomicSignalSet
AtomicSignalValue
AtomicSignalDefinition
AlertEvent
```

005B 可以通过 `StrategySignal` 上的引用和快照追溯上游对象，但不得绕过 005A 重新进行策略选择或信号聚合。

005B 不得读取：

```text
Kline4h
Kline1d
交易所 API
WebSocket
账户
持仓
订单
成交
```

005B 不得调用：

```text
MarketSnapshotService
FeatureLayerService
AtomicSignalService
DataQualityService
BackfillService
StrategySignalService 的生成逻辑
ExecutionPreparation / Execution
```

---

## 4. 输出对象

### 4.1 StrategySignalQualityResult

建议新增模型：`StrategySignalQualityResult`。

核心字段建议：

```text
quality_result_key
strategy_signal
strategy_signal_key
strategy_code
strategy_version
strategy_definition
strategy_route_rule
quality_schema_version
rule_set_hash
validation_mode
validation_as_of_utc
reference_time_utc
quality_status
quality_score
is_usable
allows_downstream
issue_count
warning_count
failed_count
blocked_reason
error_code
error_message
summary_text_zh
check_summary
created_at
updated_at
trace_id
trigger_source
```

`quality_status` 建议：

```text
passed
warning
failed
```

语义：

```text
passed：质量检查通过，允许后续读取
warning：存在非阻断问题，是否允许后续由配置决定，默认可以 allows_downstream=True
failed：存在严重质量问题，必须 allows_downstream=False
```

### 4.2 StrategySignalQualityIssue

建议新增模型：`StrategySignalQualityIssue`。

核心字段建议：

```text
quality_result
issue_code
severity
check_name
field_name
message_zh
details
created_at
```

`severity` 建议：

```text
info
warning
error
critical
```

其中：

```text
error / critical 默认导致 quality_status=failed
warning 默认导致 quality_status=warning
```

---

## 5. 幂等要求

005B 需要支持幂等。

推荐幂等 key 基于：

```text
strategy_signal_id
quality_schema_version
rule_set_hash
validation_mode
reference_time_utc
```

如果同一 `StrategySignal` 在同一规则集、同一验证模式、同一 reference time 下重复验证，应返回已有 `StrategySignalQualityResult`，不得重复写入等价结果。

注意：

```text
live 模式下数据新鲜度会随时间变化。
因此 validation_as_of_utc / reference_time_utc 必须明确记录。
如果 reference_time_utc 不同，可以生成新的质量结果。
```

---

## 6. 质量检查项

### 6.1 P0：结构完整性检查

必须检查：

```text
StrategySignal 是否存在
StrategySignal.status 是否合法
StrategySignal.direction 是否合法
StrategySignal.strength 是否存在且类型正确
StrategySignal.confidence 是否存在且类型正确
StrategySignal.evidence_text_zh 是否存在
StrategySignal.evidence_items 是否存在且符合当前 StrategySignal schema；strategy_signal.v1 中必须为非空 dict
StrategySignal.routing_snapshot 是否存在
StrategySignal.input_snapshot 是否存在
StrategySignal.aggregation_snapshot 是否存在
StrategySignal.conflict_snapshot 是否存在
StrategySignal.used_atomic_signal_value_ids 是否存在且为 list
StrategySignal.strategy_definition 是否可追溯
StrategySignal.atomic_signal_set 是否可追溯
```

字段名必须按当前项目模型为准。
`StrategySignalQualityReport` 不定义 `evidence_summary` / `param_version` 字段。

### 6.2 P0：数值合法性检查

必须检查：

```text
strength 在 [0, 1]
confidence 在 [0, 1]
strength 不是 NaN
confidence 不是 NaN
strength 不是 inf
confidence 不是 inf
created 状态下 strength / confidence 语义与 aggregation_snapshot 一致
blocked / failed 状态下 allows_downstream=False
created 且质量通过时 allows_downstream=True
```

可选 warning：

```text
direction=neutral 但 strength 很高
confidence 长期恒定为 0 或 1
strength 长期恒定为 0 或 1
```

注意：`direction=neutral` 时 strength 是否必须低于某阈值，不作为第一版硬失败。因为 neutral 可能来自多空强度都高但相互抵消。

### 6.3 P0：可追溯性检查

必须检查：

```text
used_atomic_signal_value_ids 对应的 AtomicSignalValue 真实存在
used AtomicSignalValue 属于同一个 AtomicSignalSet
used AtomicSignalValue 所属 AtomicSignalSet 与 StrategySignal.atomic_signal_set 一致
used AtomicSignalValue.is_valid=True
used AtomicSignalValue.status 合法
used AtomicSignalValue.role 不为 scaler
used AtomicSignalValue.definition 真实存在
StrategyDefinition 真实存在
StrategyDefinition.strategy_code / strategy_version / definition_hash 与 StrategySignal 记录一致
StrategyRouteRule 可追溯
routing_snapshot 中的 selected_strategy_code 与 StrategyDefinition 一致
```

如果 StrategyDefinition 中声明了：

```text
allowed_signal_codes
required_signal_codes
```

005B 应检查：

```text
used signals 是否在 allowed_signal_codes 内
required_signal_codes 是否被使用或在快照中有明确检查结果
required signal 缺失时 StrategySignal 是否 blocked
```

### 6.4 P0：证据充分性检查

必须检查：

```text
evidence_text_zh 非空
evidence_text_zh 为中文自然语言或至少包含中文说明
evidence_items 非空
evidence_items 必须符合当前 StrategySignal schema；strategy_signal.v1 中必须为非空 dict
created 信号必须包含策略生成依据
blocked 信号必须包含 blocked_reason 或阻断依据
failed 信号必须包含 error_code / error_message 或失败依据
```

第一版只做结构化检查，不使用大模型判断“文字质量”。

未来如果允许 `evidence_items` 使用 list，必须通过新的 `schema_version` 区分；005B 不应在当前 `strategy_signal.v1` schema 下要求 `evidence_items` 为 list。

### 6.5 P0：快照一致性检查

必须检查：

```text
routing_snapshot 与 StrategyDefinition / StrategyRouteRule 一致
input_snapshot 与 AtomicSignalSet / used ids 一致
role_group_snapshot 中的 voter / gatekeeper / filter / descriptor 数量与 used ids 能对应
aggregation_snapshot 中的 direction / strength / confidence 与 StrategySignal 主字段一致
conflict_snapshot 中的冲突结果与 direction / status 一致
gatekeeper_blocked 时 StrategySignal.status 必须为 blocked 且 allows_downstream=False
无 voter 时 StrategySignal.status 必须为 blocked 或 failed，不得 created
```

自洽性检查必须优先依赖结构化 snapshot，不解析 `evidence_text_zh` 的自然语言内容。

### 6.6 P0/P1：数据新鲜度检查

005B 应支持数据新鲜度检查，但必须区分运行模式。

运行模式：

```text
live
replay
backfill
manual
```

live 模式：

```text
使用 reference_time_utc 或 validation_as_of_utc 检查 StrategySignal 所依赖的 market_as_of_utc 是否过期
超过配置阈值则 failed 或 warning
```

replay / backfill 模式：

```text
不得用真实 now() 判断历史信号过期
必须使用传入 reference_time_utc
```

如果无法从 StrategySignal 或其上游引用取得 `market_as_of_utc`，应记录 issue：

```text
missing_market_as_of_utc
```

严重程度可配置。第一版建议作为 warning，除非生产 live 模式明确要求必须存在。

---

## 7. AlertEvent 要求

005B 必须在质量阻断或严重质量失败时写入 `AlertEvent`。

### 7.1 必须写 AlertEvent 的场景

```text
quality_status=failed
allows_downstream=False 且原因来自质量检查
StrategySignal 缺失关键字段
StrategySignal 可追溯性失败
used_atomic_signal_value_ids 丢失或无法追溯
StrategyDefinition / StrategyRouteRule 无法追溯
created StrategySignal 证据缺失
created StrategySignal 快照不一致
live 模式下数据严重过期
连续多次 StrategySignalQuality failed
```

### 7.2 默认不写 AlertEvent 的场景

```text
quality_status=passed
普通 warning，且未超过告警阈值
dry-run
```

### 7.3 连续失败告警

005B 应预留连续失败告警逻辑：

```text
同一 strategy_code / strategy_version 连续 N 次 quality_status=failed
→ 写 AlertEvent
```

第一版可以通过配置关闭或只实现简单查询。

### 7.4 AlertEvent 内容

AlertEvent 至少应包含：

```text
strategy_signal_id
quality_result_id
strategy_code
strategy_version
quality_status
failed_issue_codes
trace_id
trigger_source
summary_text_zh
```

---

## 8. dry-run 要求

005B 必须支持 dry-run。

dry-run 时：

```text
不写 StrategySignalQualityResult
不写 StrategySignalQualityIssue
不写 AlertEvent
不修改数据库
返回完整检查摘要
```

---

## 9. Management command 要求

建议新增 management command，例如：

```text
python manage.py validate_strategy_signal --strategy-signal-id <id>
```

建议参数：

```text
--strategy-signal-id
--dry-run
--validation-mode live|replay|backfill|manual
--reference-time-utc
--trace-id
```

---

## 10. 配置要求

005B 应支持基础配置：

```text
STRATEGY_SIGNAL_QUALITY_SCHEMA_VERSION
STRATEGY_SIGNAL_QUALITY_RULE_SET_VERSION
STRATEGY_SIGNAL_MAX_STALENESS_SECONDS
STRATEGY_SIGNAL_QUALITY_FAIL_ALERT_ENABLED
STRATEGY_SIGNAL_QUALITY_WARNING_ALERT_ENABLED
STRATEGY_SIGNAL_QUALITY_CONSECUTIVE_FAILURE_THRESHOLD
```

配置应有默认值，不得依赖环境变量才能运行测试。

---

## 11. P1 / P2 预留项

### 11.1 P1：有限信号自洽性增强

可基于结构化字段继续增强：

```text
aggregation_snapshot 与 evidence_items 的结构化 reason_code 一致
gatekeeper issue 与 blocked_reason 一致
conflict_snapshot 与 confidence 一致
```

不解析自然语言。

### 11.2 P1：时间序列稳定性检查

后续可检查最近 N 个 StrategySignal：

```text
direction flip 频率
strength 突变
confidence 突变
blocked 比例
neutral 比例
```

第一版不作为硬阻断。

### 11.3 P2：策略风格一致性检查

后续在有足够历史样本后，可检查：

```text
当前 strength / confidence 是否偏离同策略历史分布
当前方向分布是否异常
不同市场环境下策略输出是否异常
```

必须有样本量门槛和样本外验证，不得主观拍阈值。

### 11.4 P2：质量评分

可以预留 `quality_score`，但第一版不建议用复杂评分作为硬阻断依据。

质量闸门应优先由明确 issue 的 severity 决定。

---

## 12. 明确排除项

以下内容不属于 005B：

```text
市场微观结构合理性检查
盘口深度检查
买卖价差检查
滑点预测
订单可成交性检查
入场方式检查
限价单 / 市价单适配
仓位规模检查
杠杆检查
止损止盈合理性检查
当前持仓冲突检查
账户余额检查
交易频率限制
冷却期
连续亏损暂停
收益回测
Sharpe
最大回撤
参数扫描
模拟盘
自动修改 StrategySignal
```

这些应由后续风控、订单意图、执行前检查、回测研究模块处理。

---

## 13. 测试要求

至少覆盖：

```text
StrategySignal 正常 created → quality passed
缺失 evidence_text_zh → failed
缺失 evidence_items → failed
strength 越界 → failed
confidence 越界 → failed
NaN / inf → failed
used_atomic_signal_value_ids 不存在 → failed
used AtomicSignalValue 不属于同一 AtomicSignalSet → failed
used scaler signal → failed
StrategyDefinition 不可追溯 → failed
StrategyRouteRule 不可追溯 → warning 或 failed
allowed_signal_codes 不匹配 → failed
required_signal 缺失但 StrategySignal created → failed
aggregation_snapshot 与主字段不一致 → failed
gatekeeper_blocked 但 status 不是 blocked → failed
live 模式数据过期 → failed 或 warning
replay 模式不使用真实 now()
dry-run 不写库、不写 AlertEvent
quality failed 写 AlertEvent
quality passed 不写 AlertEvent
幂等重复执行返回已有结果
```

---

## 14. 验收标准

005B 完成后应满足：

```text
StrategySignalQualityResult / Issue 可落库
单个 StrategySignal 可被独立验证
严重质量问题会阻断 downstream
严重质量问题会写 AlertEvent
passed / warning / failed 语义清晰
dry-run 可用
幂等可用
测试覆盖 P0 检查项
不修改 StrategySignal 原始结果
不实现风控、订单、仓位、回测、UI
```

---

## 15. 最终边界说明

005B 的定位是：

```text
验证 005A 产物质量
保护后续 DecisionSnapshot 及后续交易链路不消费坏信号
记录质量问题
触发必要告警
```

不是：

```text
生成新策略
优化策略参数
判断交易是否赚钱
生成交易计划
执行风控
下单
```

因此，005B 应保持为清晰的质量闸门层。
