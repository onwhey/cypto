# Plan 005B：StrategySignalQuality / StrategyValidation 策略信号质量验证层

> 当前阶段：005B StrategySignalQuality / StrategyValidation  
> 前置阶段：005A StrategySignal 已完成  
> 需求来源：`docs/requirements/strategy_signal_quality.md`  
> 核心目标：验证 005A 生成的 `StrategySignal` 是否可信、完整、可追溯、数值合法、证据充分、未过期，并决定是否允许后续 Decision / Risk 流程继续读取。  
> 核心原则：质量闸门 fail-closed；不修改 `StrategySignal` 原始结果；质量失败 / 阻断必须写 `AlertEvent`。

---

## 0. 执行边界

005B 是 `StrategySignal` 的质量验证层，不是策略生成层、风控层、执行层、收益回测层。

005B 的输入：

```text
StrategySignal
StrategyDefinition
StrategyRouteRule
AtomicSignalSet
AtomicSignalValue
AtomicSignalDefinition
```

005B 的输出：

```text
StrategySignalQualityResult
StrategySignalQualityIssue
AlertEvent（质量失败 / 阻断 / 连续失败时）
```

005B 只回答：

```text
这个 StrategySignal 是否质量合格，是否允许后续 Decision / Risk 继续读取？
```

005B 不回答：

```text
是否应该交易
是否开仓 / 平仓
下多少仓位
用多少杠杆
什么价格入场
怎么下单
策略是否赚钱
```

005B 必须保持：

```text
StrategySignal 保持原样
质量处理写入独立质量结果表
后续模块读取质量结果决定是否继续
```

---

## 1. 前置条件

开始前必须确认以下文件已存在并已进入项目：

```text
docs/requirements/strategy_signal_quality.md
```

开始前必须阅读：

```text
AGENTS.md
README.md
docs/rules/project_invariants.md
docs/requirements/01_project_scope.md
docs/requirements/02_system_capabilities.md
docs/requirements/atomic_signals.md
docs/requirements/strategy_signals.md
docs/requirements/strategy_signal_quality.md
docs/plans/005A_strategy_signal_plan.md
apps/atomic_signal/models.py
apps/strategy_signal/models.py
apps/strategy_signal/services.py
apps/strategy_signal/router.py
apps/strategy_signal/calculators/atomic_vote.py
apps/strategy_signal/default_strategy_definitions.py
apps/strategy_signal/default_route_rules.py
```

如本 plan 与 `AGENTS.md` 或 `docs/rules/project_invariants.md` 冲突，以后两者为准。

---

## 2. 本阶段明确不做

005B 禁止实现：

```text
重新生成 StrategySignal
重新执行 StrategyRouter
重新执行 atomic_vote:v1
重新聚合 AtomicSignal
重新计算 Feature
重新读取 Kline
读取交易所盘口 / 订单簿
读取账户余额
读取当前持仓
读取订单 / 成交
生成 entry_price
生成 stop_loss
生成 take_profit
生成 position_size
生成 leverage
生成订单动作枚举
生成 CandidateOrderIntent / ApprovedOrderIntent
下单
回测收益
Sharpe
最大回撤
参数扫描
模拟盘
UI / dashboard
自动修改 StrategySignal.direction
自动修改 StrategySignal.strength
自动修改 StrategySignal.confidence
```

005B 禁止调用：

```text
Binance REST
WebSocket
MarketSnapshotService
FeatureLayerService
AtomicSignalService
DataQualityService
BackfillService
StrategySignalService 的生成逻辑
Risk / Trading / Execution 相关服务
Hermes webhook
notifications service 直接发送通知
```

005B 可以写 `AlertEvent`，但必须沿用项目既有 `AlertEvent` 落库方式，不得直接调用 Hermes webhook。

---

## 3. App 与文件组织

005B 建议继续放在现有 app：

```text
apps.strategy_signal
```

原因：005B 是 `StrategySignal` 的质量验证子模块，不是独立业务链路。

建议新增 / 修改文件：

```text
apps/strategy_signal/models.py
apps/strategy_signal/constants.py
apps/strategy_signal/quality_hashes.py
apps/strategy_signal/quality_selectors.py
apps/strategy_signal/quality_checks.py
apps/strategy_signal/quality_services.py
apps/strategy_signal/management/commands/validate_strategy_signal.py
apps/strategy_signal/migrations/0002_strategy_signal_quality.py

tests/strategy_signal/test_strategy_signal_quality_models.py
tests/strategy_signal/test_strategy_signal_quality_checks.py
tests/strategy_signal/test_strategy_signal_quality_service.py
tests/strategy_signal/test_validate_strategy_signal_command.py
```

如果当前实现已有更合适的命名风格，可保持一致，但必须避免把质量验证逻辑混入 005A 的 `StrategySignalService` 生成逻辑。

---

## 4. 新增模型

### 4.1 StrategySignalQualityResult

新增模型：

```text
StrategySignalQualityResult
```

定位：单次 `StrategySignal` 质量验证结果。

建议字段：

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
trace_id
trigger_source
created_at
updated_at
```

字段语义：

```text
quality_result_key：幂等 key
strategy_signal：被验证的 StrategySignal
strategy_signal_key：冗余保存，便于审计
strategy_code / strategy_version：冗余保存策略身份
strategy_definition：验证时追溯到的 StrategyDefinition
strategy_route_rule：验证时追溯到的 StrategyRouteRule，如无法追溯可为空但必须记录 issue
quality_schema_version：质量结果 schema 版本
rule_set_hash：本次质量规则集 hash
validation_mode：live / replay / backfill / manual
validation_as_of_utc：实际验证发生时间
reference_time_utc：用于新鲜度检查的参考时间
quality_status：passed / warning / failed
quality_score：预留评分字段，第一版不作为主要硬闸门
is_usable：质量结果本身是否可用
allows_downstream：是否允许后续 Decision / Risk 读取
issue_count / warning_count / failed_count：问题统计
blocked_reason：质量阻断原因
error_code / error_message：质量验证异常信息
summary_text_zh：中文质量摘要
check_summary：结构化检查摘要
trace_id / trigger_source：链路追踪
```

`quality_status` 枚举：

```text
passed
warning
failed
```

语义：

```text
passed：质量检查通过，allows_downstream=True
warning：存在非阻断问题，默认 allows_downstream=True，可配置
failed：存在严重质量问题，allows_downstream=False
```

### 4.2 StrategySignalQualityIssue

新增模型：

```text
StrategySignalQualityIssue
```

定位：单条质量问题。

建议字段：

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

`severity` 枚举：

```text
info
warning
error
critical
```

默认处理：

```text
info：不影响 quality_status
warning：quality_status 至少为 warning
error：quality_status=failed
critical：quality_status=failed
```

---

## 5. 模型约束与校验

### 5.1 StrategySignalQualityResult.clean()

必须校验：

```text
quality_status 合法
validation_mode 合法
issue_count / warning_count / failed_count >= 0
quality_score 为空或在 [0, 1]
quality_status=passed 时 allows_downstream=True
quality_status=failed 时 allows_downstream=False
quality_status=failed 时 blocked_reason 或 error_code 非空
summary_text_zh 非空
check_summary 非空 dict
strategy_signal 不为空
quality_result_key 非空
quality_schema_version 非空
rule_set_hash 非空
reference_time_utc 非空
validation_as_of_utc 非空
```

### 5.2 StrategySignalQualityIssue.clean()

必须校验：

```text
issue_code 非空
severity 合法
check_name 非空
message_zh 非空
message_zh 为中文说明或至少包含中文解释
```

### 5.3 数据库约束

建议唯一约束：

```text
strategy_signal
quality_schema_version
rule_set_hash
validation_mode
reference_time_utc
```

或使用唯一 `quality_result_key`。

如果 MySQL 对部分 CheckConstraint 支持有限，可沿用项目现有做法，在 `clean()` 和 service 写入前显式校验，数据库约束作为辅助。

---

## 6. 配置项

在 `config/settings.py` 中增加默认配置，不要求必须通过环境变量提供。

建议配置：

```text
STRATEGY_SIGNAL_QUALITY_SCHEMA_VERSION = "strategy_signal_quality.v1"
STRATEGY_SIGNAL_QUALITY_RULE_SET_VERSION = "p0.v1"
STRATEGY_SIGNAL_MAX_STALENESS_SECONDS = 4 * 60 * 60
STRATEGY_SIGNAL_QUALITY_FAIL_ALERT_ENABLED = True
STRATEGY_SIGNAL_QUALITY_WARNING_ALERT_ENABLED = False
STRATEGY_SIGNAL_QUALITY_CONSECUTIVE_FAILURE_THRESHOLD = 3
STRATEGY_SIGNAL_QUALITY_WARNING_ALLOWS_DOWNSTREAM = True
```

配置原则：

```text
测试环境不依赖真实环境变量
live / replay / backfill / manual 都必须可配置
warning 是否阻断由配置决定，默认不阻断
failed 必须阻断
```

---

## 7. 幂等设计

005B 必须幂等。

推荐 `quality_result_key` 基于：

```text
strategy_signal_id
strategy_signal_key
quality_schema_version
rule_set_hash
validation_mode
reference_time_utc
```

重复执行同一 `StrategySignal`、同一规则集、同一验证模式、同一 reference time：

```text
返回已有 StrategySignalQualityResult
不得重复写 StrategySignalQualityIssue
不得重复写等价 AlertEvent
```

注意：

```text
live 模式下 reference_time_utc 不同，可以产生新的质量结果
replay / backfill 模式必须由调用方传入 reference_time_utc
```

---

## 8. 质量验证服务

新增服务：

```text
StrategySignalQualityService
```

建议入口：

```python
validate(
    strategy_signal_id: int,
    *,
    validation_mode: str = "live",
    reference_time_utc: datetime | None = None,
    dry_run: bool = False,
    trace_id: str | None = None,
    trigger_source: str = "manual",
) -> StrategySignalQualitySummary
```

### 8.1 主流程

```text
1. 读取 StrategySignal
2. 准备 reference_time_utc
3. 生成 rule_set_hash
4. 生成 quality_result_key
5. 非 dry-run 下检查幂等，命中则返回已有结果
6. 执行 P0 质量检查项
7. 聚合 issues
8. 根据 severity 计算 quality_status / allows_downstream
9. 生成 summary_text_zh / check_summary
10. dry-run：只返回摘要，不写库、不写 AlertEvent
11. 非 dry-run：transaction.atomic 写 StrategySignalQualityResult / Issue
12. quality failed 或质量阻断：写 AlertEvent
13. 检查连续失败阈值，必要时写 AlertEvent
14. 返回结果摘要
```

### 8.2 异常处理

服务异常时：

```text
非 dry-run：写 failed StrategySignalQualityResult
非 dry-run：写 AlertEvent
 dry-run：不写库、不写 AlertEvent，只返回 failed 摘要
```

不能因为某个检查异常导致半写。

---

## 9. 质量检查项实现

建议将检查逻辑拆成独立函数或类：

```text
StructureIntegrityCheck
NumericValidityCheck
TraceabilityCheck
EvidenceCompletenessCheck
SnapshotConsistencyCheck
FreshnessCheck
```

每个 check 返回：

```text
check_name
passed
issues
summary
```

每个 issue 包含：

```text
issue_code
severity
field_name
message_zh
details
```

### 9.1 P0：结构完整性检查

必须检查：

```text
StrategySignal 存在
status 合法
direction 合法
strength 存在且类型正确
confidence 存在且类型正确
evidence_text_zh 非空
evidence_items 存在且为 list
routing_snapshot 存在且为 dict
input_snapshot 存在且为 dict
aggregation_snapshot 存在且为 dict
conflict_snapshot 存在且为 dict
used_atomic_signal_value_ids 存在且为 list
strategy_definition 可追溯
atomic_signal_set 可追溯
```

严重程度建议：

```text
关键字段缺失：error / critical
非关键快照字段缺失：error
字段类型错误：error
```

### 9.2 P0：数值合法性检查

必须检查：

```text
strength 在 [0, 1]
confidence 在 [0, 1]
strength 不是 NaN
confidence 不是 NaN
strength 不是 inf
confidence 不是 inf
created 状态下 strength / confidence 与 aggregation_snapshot 一致
blocked / failed 状态下 allows_downstream=False
created 且质量通过时 StrategySignal.allows_downstream=True
```

注意：

```text
direction=neutral 且 strength 较高，第一版只作为 warning，不作为硬失败
```

### 9.3 P0：可追溯性检查

必须检查：

```text
used_atomic_signal_value_ids 对应 AtomicSignalValue 真实存在
used AtomicSignalValue 属于同一个 AtomicSignalSet
used AtomicSignalValue.atomic_signal_set == StrategySignal.atomic_signal_set
used AtomicSignalValue.is_valid=True
used AtomicSignalValue.status 合法
used AtomicSignalValue.role != scaler
used AtomicSignalValue.definition 存在
StrategyDefinition 存在
StrategyDefinition.strategy_code / strategy_version / definition_hash 与 StrategySignal 记录一致
StrategyRouteRule 可追溯，或缺失时记录 issue
routing_snapshot.selected_strategy_code 与 StrategyDefinition 一致
```

如果 `StrategyDefinition.params` 中有：

```text
allowed_signal_codes
required_signal_codes
```

则必须检查：

```text
used signal codes 都在 allowed_signal_codes 内
required_signal_codes 被使用，或在快照中有明确检查结果
required 缺失时 StrategySignal 不得 created
```

### 9.4 P0：证据充分性检查

必须检查：

```text
evidence_text_zh 非空
evidence_text_zh 是中文说明或至少包含中文解释
evidence_items 非空
evidence_items 是结构化 list
created 信号包含策略生成依据
blocked 信号包含 blocked_reason 或阻断依据
failed 信号包含 error_code / error_message 或失败依据
```

第一版禁止使用大模型或自然语言解析来判断证据内容是否“语义正确”。

### 9.5 P0：快照一致性检查

必须检查：

```text
routing_snapshot 与 StrategyDefinition / StrategyRouteRule 一致
input_snapshot 与 AtomicSignalSet / used ids 一致
role_group_snapshot 中 voter / gatekeeper / filter / descriptor 数量与 used ids 可对应
aggregation_snapshot.direction 与 StrategySignal.direction 一致
aggregation_snapshot.strength 与 StrategySignal.strength 一致
aggregation_snapshot.confidence 与 StrategySignal.confidence 一致
conflict_snapshot 与 direction / status 一致
gatekeeper_blocked 时 StrategySignal.status=blocked 且 allows_downstream=False
无 voter 时 StrategySignal.status=blocked 或 failed，不得 created
```

自洽性检查必须依赖结构化 snapshot，不解析 `evidence_text_zh` 自然语言。

### 9.6 P0/P1：数据新鲜度检查

支持模式：

```text
live
replay
backfill
manual
```

live 模式：

```text
使用 reference_time_utc 或 validation_as_of_utc 检查 market_as_of_utc 是否过期
超过 STRATEGY_SIGNAL_MAX_STALENESS_SECONDS 则 warning 或 failed
```

replay / backfill 模式：

```text
不得用真实 now() 判断历史信号过期
必须使用传入 reference_time_utc
```

如果无法取得 `market_as_of_utc`：

```text
issue_code = missing_market_as_of_utc
第一版建议 warning
生产 live 模式可配置为 failed
```

`market_as_of_utc` 来源优先级建议：

```text
StrategySignal 字段
StrategySignal.input_snapshot
StrategySignal.atomic_signal_set
AtomicSignalSet.feature_set.market_snapshot
```

如果当前模型没有直接字段，不要新增上游调用逻辑，基于现有引用和快照追溯。

---

## 10. 质量状态计算

根据 issue severity 计算：

```text
存在 critical / error → quality_status=failed, allows_downstream=False
存在 warning 且无 error / critical → quality_status=warning
无 warning / error / critical → quality_status=passed
```

warning 默认：

```text
quality_status=warning
allows_downstream=True
```

除非配置：

```text
STRATEGY_SIGNAL_QUALITY_WARNING_ALLOWS_DOWNSTREAM=False
```

`quality_score` 第一版可以简单计算或保留默认：

```text
passed = 1.0
warning = 0.7
failed = 0.0
```

注意：第一版不应使用复杂质量评分作为硬阻断依据；硬阻断由明确 issue severity 决定。

---

## 11. AlertEvent 规则

005B 必须在质量失败或质量阻断时写 `AlertEvent`。

必须写 AlertEvent：

```text
quality_status=failed
allows_downstream=False 且原因来自质量检查
关键字段缺失
可追溯性失败
used_atomic_signal_value_ids 无法追溯
StrategyDefinition / StrategyRouteRule 无法追溯
created StrategySignal 证据缺失
created StrategySignal 快照不一致
live 模式下数据严重过期
连续多次 quality_status=failed
```

默认不写 AlertEvent：

```text
quality_status=passed
普通 warning 且未超过告警阈值
dry-run
```

AlertEvent 内容至少包含：

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

连续失败告警：

```text
同一 strategy_code / strategy_version 连续 N 次 quality_status=failed
→ 写 AlertEvent
```

第一版可以实现简单查询；如果实现复杂度过高，可在代码中预留但必须不影响单次 failed AlertEvent。

---

## 12. dry-run

005B 必须支持 dry-run。

dry-run 时：

```text
不写 StrategySignalQualityResult
不写 StrategySignalQualityIssue
不写 AlertEvent
不修改数据库
返回完整检查摘要
```

返回摘要应包含：

```text
quality_status
allows_downstream
issue_count
warning_count
failed_count
issues
check_summary
summary_text_zh
would_alert
```

---

## 13. Management command

新增 command：

```text
python manage.py validate_strategy_signal --strategy-signal-id <id>
```

参数：

```text
--strategy-signal-id
--dry-run
--validation-mode live|replay|backfill|manual
--reference-time-utc
--trace-id
--trigger-source
```

行为：

```text
调用 StrategySignalQualityService.validate
输出 quality_status / allows_downstream / issue_count
failed 时 command 可返回非 0 exit code，除非提供允许失败的参数
```

如果项目现有 management command 风格不使用非 0 exit code，可保持一致，但必须在输出中清楚显示 failed。

---

## 14. 测试计划

### 14.1 Model 测试

覆盖：

```text
StrategySignalQualityResult full_clean 正常
quality_status 非法 → validation error
quality_score 越界 → validation error
failed 但 allows_downstream=True → validation error
passed 但 allows_downstream=False → validation error
summary_text_zh 为空 → validation error
check_summary 为空 → validation error
StrategySignalQualityIssue severity 非法 → validation error
StrategySignalQualityIssue message_zh 为空 → validation error
```

### 14.2 Hash / 幂等测试

覆盖：

```text
相同 strategy_signal + schema + rule_set + mode + reference_time → 同一 quality_result_key
不同 reference_time → 不同 quality_result_key
重复 validate 返回已有 StrategySignalQualityResult
重复 validate 不重复写 Issue
重复 validate 不重复写 AlertEvent
```

### 14.3 P0 检查项测试

覆盖：

```text
正常 created StrategySignal → quality passed
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
无 voter 但 StrategySignal created → failed
live 模式数据过期 → failed 或 warning
replay 模式不使用真实 now()
```

### 14.4 AlertEvent 测试

覆盖：

```text
quality failed 写 AlertEvent
quality passed 不写 AlertEvent
普通 warning 默认不写 AlertEvent
dry-run 不写 AlertEvent
连续失败达到阈值写 AlertEvent
AlertEvent payload 包含 strategy_signal_id / quality_result_id / issue_codes / trace_id
```

### 14.5 dry-run 测试

覆盖：

```text
dry-run 不写 StrategySignalQualityResult
dry-run 不写 StrategySignalQualityIssue
dry-run 不写 AlertEvent
dry-run 返回完整 issue 摘要
```

### 14.6 Management command 测试

覆盖：

```text
validate_strategy_signal 正常输出 passed
validate_strategy_signal --dry-run 不写库
validation-mode 参数合法
reference-time-utc 参数合法
缺失 strategy-signal-id 报错
不存在 strategy-signal-id 报错或返回 failed 摘要
```

---

## 15. 实施顺序

建议按以下顺序实现：

```text
1. 增加 constants / settings 默认配置
2. 增加 StrategySignalQualityResult / StrategySignalQualityIssue 模型
3. 生成 migration
4. 实现 quality_result_key / rule_set_hash
5. 实现 quality checks 基础结构
6. 实现 P0 检查项
7. 实现 StrategySignalQualityService.validate
8. 实现 dry-run
9. 实现 AlertEvent 写入
10. 实现连续失败告警的最小逻辑
11. 实现 management command
12. 补齐 model / service / command tests
13. 运行 makemigrations --check --dry-run
14. 运行 manage.py test
15. 运行 pytest
```

---

## 16. 验收标准

005B 完成后必须满足：

```text
StrategySignalQualityResult 可落库
StrategySignalQualityIssue 可落库
单个 StrategySignal 可被独立验证
严重质量问题会 failed
failed 必须 allows_downstream=False
严重质量问题会写 AlertEvent
dry-run 不写库、不告警
幂等可用
P0 检查项测试覆盖
不修改 StrategySignal 原始结果
不重新生成 StrategySignal
不实现风控、订单、仓位、回测、UI
```

必须通过：

```text
python manage.py makemigrations --check --dry-run
python manage.py test
pytest
```

---

## 17. P1 / P2 预留，不在本阶段实现

P1 可预留：

```text
有限信号自洽性增强
时间序列稳定性检查
warning 级别告警阈值
更细的 freshness policy
```

P2 可预留：

```text
策略风格一致性检查
历史分布偏离检查
复杂质量评分
多周期质量趋势分析
```

预留项只写文档或常量注释，不实现业务逻辑，不影响 005B P0 验收。

---

## 18. 最终边界说明

005B 的定位是：

```text
验证 StrategySignal 质量
保护后续 Decision / Risk 不消费坏信号
记录质量问题
必要时告警
```

005B 不是：

```text
策略生成器
策略优化器
风控模块
订单模块
执行模块
回测模块
UI 模块
```

如果实现过程中发现需要读取账户、持仓、订单簿、交易所规则或生成仓位 / 价格 / 止损止盈，应立即停止并上移到后续风控 / 订单 / 执行前检查模块，不能塞进 005B。
