# 006A DecisionSnapshot 实现计划

> 建议存放路径：`docs/plans/006A_decision_snapshot_plan.md`  
> 对齐需求：`docs/requirements/decision_snapshot.md`  
> 当前阶段：006A DecisionSnapshot  
> 上游：005A StrategySignal / 005B StrategySignalQuality  
> 下游：007A OrderPlan  
> 关键原则：006A 只冻结策略侧目标仓位意图，不读取当前持仓，不生成订单动作，不做风控，不下单。

---

## 1. 本计划的目标

006A 的目标是实现 DecisionSnapshot 模块。

本阶段只做一件事：

```text
把通过质量验证的 StrategySignal，经过版本化 DecisionPolicy，冻结成一个可审计、可追溯、可幂等的目标仓位意图快照。
```

006A 的正式合同为：

```text
StrategySignal
→ StrategySignalQualityResult
→ DecisionPolicy / DecisionRuleDefinition
→ DecisionSnapshot(target_intent, target_position_ratio)
→ OrderPlan
```

DecisionSnapshot 回答：

```text
在当前策略信号和决策规则下，系统理想上希望处于什么目标仓位？
```

DecisionSnapshot 不回答：

```text
当前有没有仓位
应该买还是卖
应该开仓还是平仓
应该反手还是减仓
订单数量是多少
保证金是否足够
是否通过风控
是否可以真实下单
```

这些问题属于后续模块：

```text
OrderPlan
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
```

---

## 2. 严格边界

### 2.1 006A 必须实现

```text
DecisionRuleDefinition / DecisionPolicyDefinition
DecisionPolicy registry
默认 P0 决策策略
DecisionSnapshot target-position 字段
DecisionSnapshotService
selectors
hash / idempotency
input_snapshot
evidence_summary
lineage_snapshot
expires_at
trace_id
AlertEvent
dry-run
management command
tests
```

### 2.2 006A 不得实现

```text
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
PriceSnapshot
Binance Account Sync
真实 Binance REST / WebSocket 调用
账户余额读取
交易所持仓读取
订单数量计算
订单 side 计算
reduceOnly / closePosition 计算
保证金估算
symbol rule 校验
价格保护
成交追踪
撤单
查询订单
回测收益
UI / dashboard
```

### 2.3 006A 不得读取的对象

```text
BinanceSyncRun
BinanceAccountSnapshot
BinanceBalanceSnapshot
BinancePositionSnapshot
BinanceSymbolRuleSnapshot
PriceSnapshot
CandidateOrderIntent
ApprovedOrderIntent
ExchangeOrder
TradeFill
```

006A 只能通过 `StrategySignal` 的外键链路保留市场快照、策略定义、质量结果等 lineage 摘要。

---

## 3. 输出合同

### 3.1 target_intent

`DecisionSnapshot` 必须输出：

```text
target_intent
```

P0 允许值：

```text
TARGET_POSITION
NO_TARGET_CHANGE
NO_TRADE
```

含义：

```text
TARGET_POSITION
= 本周期明确提出目标仓位比例，下游 OrderPlan 可以消费。

NO_TARGET_CHANGE
= 本周期不提出新的目标仓位，不代表目标空仓。

NO_TRADE
= 本周期信号不足、冲突或不可交易，不进入订单链路。
NO_TRADE 只能作为 DecisionSnapshot 的 target_intent / 目标仓位意图语义，表示不产生目标仓位变化或不进入订单计划。
NO_TRADE 不得作为下游订单动作输入 RiskCheck / Execution。
```

### 3.2 target_position_ratio

`target_position_ratio` 仅在以下情况必填：

```text
target_intent = TARGET_POSITION
```

范围：

```text
-1.0 <= target_position_ratio <= +1.0
```

含义：

```text
+1.0 = 决策策略预算内最大多头目标
+0.5 = 半多头目标
 0.0 = 目标空仓
-0.5 = 半空头目标
-1.0 = 决策策略预算内最大空头目标
```

必须明确区分：

```text
target_position_ratio = 0.0
```

和：

```text
target_intent = NO_TARGET_CHANGE
```

前者表示目标空仓，后续 OrderPlan 可能生成平仓候选订单；后者表示本周期没有新目标，不应生成订单计划。

### 3.3 target_confidence

建议字段：

```text
target_confidence
```

范围：

```text
0.0 <= target_confidence <= 1.0
```

### 3.4 target_reason_code / target_reason_summary

必须保存：

```text
target_reason_code
target_reason_summary
```

用于审计、复盘和人工排查。

### 3.5 target_schema_version

P0 固定：

```text
target_schema_version = v2
```

用于标识目标仓位合同。

---

## 4. 状态与下游消费语义

建议状态：

```text
created
blocked
failed
```

下游消费规则：

```text
TARGET_POSITION + status=created + is_usable=True + allows_downstream=True
→ OrderPlan 可以消费。

NO_TARGET_CHANGE + status=created + is_usable=True + allows_downstream=False
→ 正常终止，不生成 OrderPlan。

NO_TRADE + status=created + is_usable=True + allows_downstream=False
→ 正常终止，不生成 OrderPlan。

blocked / failed
→ 不允许下游消费，必须 fail-closed。
```

注意：

```text
NO_TARGET_CHANGE / NO_TRADE 是合格 DecisionSnapshot 的正常终态，不能被当作系统异常。
```

但如果 NO_TRADE 是因为上游缺失、质量失败或策略信号不可用导致，则应使用 `blocked`，而不是正常 `created + NO_TRADE`。

---

## 5. 推荐代码结构

建议结构：

```text
apps/decision_snapshot/
  __init__.py
  apps.py
  constants.py
  models.py
  selectors.py
  hashes.py
  services.py
  default_decision_rule_definitions.py
  definition_seed.py
  policies/
    __init__.py
    base.py
    registry.py
    target_position_basic.py
  management/
    __init__.py
    commands/
      __init__.py
      seed_decision_rule_definitions.py
      build_decision_snapshot.py
  migrations/
    __init__.py
```

测试目录：

```text
tests/decision_snapshot/
  test_decision_snapshot_models.py
  test_decision_snapshot_hashes.py
  test_decision_rule_seed.py
  test_decision_policy_target_position_basic.py
  test_decision_snapshot_service.py
  test_build_decision_snapshot_command.py
  test_decision_snapshot_boundaries.py
```

---

## 6. 模型实现计划

### 6.1 DecisionRuleDefinition

`DecisionRuleDefinition` 用于保存决策策略定义。

建议字段：

```text
decision_rule_code
decision_rule_name
decision_rule_version
algorithm_name
algorithm_version
params
params_hash
definition_hash
status
enabled
description
created_at
updated_at
```

生产可用规则必须满足：

```text
status = active
enabled = True
```

规则身份字段不得原地修改。参数变化必须新增版本。

### 6.2 DecisionSnapshot

`DecisionSnapshot` 必须支持以下逻辑字段：

```text
decision_snapshot_key
decision_schema_version
target_schema_version
strategy_signal
strategy_signal_quality_result
decision_rule_definition
decision_rule_code
decision_rule_version
decision_rule_hash
status
is_usable
allows_downstream
blocked_reason
failed_reason
error_code
error_message
target_intent
target_position_ratio
target_confidence
target_reason_code
target_reason_summary
signal_direction
signal_strength
signal_confidence
market_as_of_utc
cycle_identity
input_snapshot
evidence_summary
lineage_snapshot
expires_at
trace_id
trigger_source
created_at
updated_at
```

### 6.3 字段约束

必须在 model clean / service validation / tests 中保证：

```text
target_intent in [TARGET_POSITION, NO_TARGET_CHANGE, NO_TRADE]

TARGET_POSITION:
  target_position_ratio 必填
  -1.0 <= target_position_ratio <= +1.0

NO_TARGET_CHANGE / NO_TRADE:
  target_position_ratio 必须为 null

target_confidence 如存在：
  0.0 <= target_confidence <= 1.0
```

MySQL 低版本可能不强制 CHECK，测试不能只依赖数据库 CHECK 约束，service / model 层也要校验。

---

## 7. DecisionPolicy 实现计划

### 7.1 Policy registry

实现策略注册表：

```text
DecisionPolicyRegistry
```

职责：

```text
根据 DecisionRuleDefinition.algorithm_name / algorithm_version 找到对应 policy executor。
```

如果找不到 policy：

```text
DecisionSnapshot.status = blocked
blocked_reason = decision_policy_plugin_missing
allows_downstream = False
写 AlertEvent
```

### 7.2 Policy executor 接口

建议接口：

```python
class DecisionPolicy:
    def evaluate(self, *, strategy_signal, quality_result, definition, reference_time_utc) -> DecisionPolicyOutput:
        ...
```

输出结构建议：

```text
target_intent
target_position_ratio
target_confidence
target_reason_code
target_reason_summary
evidence
```

Policy 不得读取当前持仓、账户余额、Binance 快照或 PriceSnapshot。

### 7.3 默认 P0 policy

新增默认策略：

```text
target_position_basic:v1
```

默认参数示例：

```json
{
  "min_strength_for_target": "0.50",
  "min_confidence_for_target": "0.50",
  "bullish_target_ratio": "1.0",
  "bearish_target_ratio": "-1.0",
  "neutral_intent": "NO_TRADE",
  "weak_signal_intent": "NO_TARGET_CHANGE"
}
```

默认行为：

```text
StrategySignal.direction = bullish
且 strength >= min_strength_for_target
且 confidence >= min_confidence_for_target
→ TARGET_POSITION, target_position_ratio = bullish_target_ratio

StrategySignal.direction = bearish
且 strength >= min_strength_for_target
且 confidence >= min_confidence_for_target
→ TARGET_POSITION, target_position_ratio = bearish_target_ratio

StrategySignal.direction = neutral / none
→ neutral_intent

strength 或 confidence 不达标
→ weak_signal_intent
```

以上映射必须来自 `DecisionRuleDefinition.params`，不得散落硬编码在 service 主流程中。

### 7.4 target flat

如果策略需要表达风险关闭 / 目标空仓，policy 应通过参数或明确 reason 生成：

```text
TARGET_POSITION, target_position_ratio = 0.0
```

不得用 `NO_TRADE` 表示目标空仓。

---

## 8. Seed 计划

实现：

```text
seed_decision_rule_definitions
```

默认 seed：

```text
decision_rule_code = target_position_basic
decision_rule_version = v1
algorithm_name = target_position_basic
algorithm_version = v1
status = active
enabled = True
```

幂等规则：

```text
以 decision_rule_code + decision_rule_version 查询 existing。

如果 params_hash 相同：
  跳过或返回 existing。

如果 params_hash 不同：
  明确报错，要求新增 decision_rule_version，不得原地修改。
```

不得以 `params_hash` 作为查找 existing 的条件。

---

## 9. Selector 计划

实现 selectors：

```text
get_usable_strategy_signal(strategy_signal_id)
get_quality_result_for_strategy_signal(strategy_signal_id, quality_result_id=None)
get_active_decision_rule_definition(rule_code=None, rule_version=None)
get_existing_decision_snapshot_by_key(snapshot_key)
```

上游准入：

```text
StrategySignal.status / is_usable / allows_downstream 必须满足下游消费条件。
StrategySignalQualityResult 必须存在且 allows_downstream=True。
QualityResult 必须和 StrategySignal 匹配。
DecisionRuleDefinition 必须 active + enabled。
DecisionRuleDefinition.definition_hash / params_hash 必须可校验。
```

不满足时：

```text
DecisionSnapshot.status = blocked
is_usable = False
allows_downstream = False
写 AlertEvent
```

---

## 10. Hash / 幂等计划

新增：

```text
build_decision_snapshot_key
build_decision_input_hash
build_decision_rule_hash
```

幂等 key 至少包含：

```text
strategy_signal_id
strategy_signal_quality_result_id
decision_rule_definition_id
decision_rule_definition_hash
target_schema_version
target_intent
target_position_ratio
target_confidence
target_reason_code
market_as_of_utc / cycle_identity
input lineage hashes
```

相同输入、相同规则、相同 policy 输出：

```text
返回已有 DecisionSnapshot
不得重复写等价结果
不得重复发送等价正式 AlertEvent
```

不同 target intent / ratio / rule version / input lineage 应生成不同 key。

---

## 11. Service 流程计划

建议入口：

```python
DecisionSnapshotService.run(
    *,
    strategy_signal_id,
    strategy_signal_quality_result_id=None,
    decision_rule_code=None,
    decision_rule_version=None,
    reference_time_utc=None,
    trace_id=None,
    trigger_source="manual",
    dry_run=False,
)
```

流程：

```text
1. 解析 reference_time_utc，统一 UTC。
2. 读取 StrategySignal。
3. 读取并校验 StrategySignalQualityResult。
4. 读取 active DecisionRuleDefinition。
5. 校验 definition hash / params hash。
6. 从 registry 获取 DecisionPolicy executor。
7. 执行 policy，得到 target intent output。
8. 校验 target_intent / target_position_ratio / target_confidence。
9. 构造 input_snapshot / evidence_summary / lineage_snapshot。
10. 计算 decision_snapshot_key。
11. dry-run：不写库、不写 AlertEvent，返回摘要。
12. 非 dry-run：幂等 get-or-create。
13. 对 blocked / failed 写 AlertEvent。
14. 对正常 NO_TARGET_CHANGE / NO_TRADE 不写高严重级别 AlertEvent，除非由异常输入导致。
15. 返回 DecisionSnapshot 或结构化摘要。
```

异常处理：

```text
可预期输入问题 → blocked
不可预期代码/数据库异常 → failed
```

---

## 12. AlertEvent 计划

006A 必须在以下场景写 AlertEvent：

```text
missing_strategy_signal
strategy_signal_not_usable
missing_strategy_signal_quality_result
quality_result_not_allowing_downstream
quality_result_signal_mismatch
decision_rule_missing
decision_rule_disabled_or_not_active
decision_rule_hash_mismatch
decision_policy_plugin_missing
decision_policy_calculation_failed
invalid_target_intent
invalid_target_position_ratio
invalid_target_confidence
decision_snapshot_generation_failed
```

006A 不应对正常终态写高严重级别 AlertEvent：

```text
NO_TARGET_CHANGE
NO_TRADE
```

但可以写审计日志或低级别记录，具体依项目现有 AlertEvent 语义决定。

dry-run 不写 AlertEvent。

---

## 13. Management command 计划

新增命令：

```text
build_decision_snapshot
```

推荐参数：

```text
--strategy-signal-id
--strategy-signal-quality-result-id
--decision-rule-code
--decision-rule-version
--reference-time-utc
--trace-id
--trigger-source
--dry-run
```

命令输出必须展示：

```text
target_intent
target_position_ratio
target_confidence
target_reason_code
decision_snapshot_key
status
allows_downstream
```

命令不得接收账户、持仓、OrderPlan、RiskCheck 或 Execution 相关参数。

---

## 14. 测试计划

### 14.1 模型测试

覆盖：

```text
TARGET_POSITION 必须有 target_position_ratio
NO_TARGET_CHANGE / NO_TRADE 必须 target_position_ratio=null
target_position_ratio 边界 [-1, 1]
target_confidence 边界 [0, 1]
```

### 14.2 policy 测试

覆盖默认 `target_position_basic:v1`：

```text
bullish + strength/confidence 达标 → TARGET_POSITION + 正 ratio
bearish + strength/confidence 达标 → TARGET_POSITION + 负 ratio
neutral → NO_TRADE 或 params 指定 intent
weak strength → NO_TARGET_CHANGE 或 params 指定 intent
weak confidence → NO_TARGET_CHANGE 或 params 指定 intent
policy 输出 ratio 越界 → blocked
policy 输出非法 intent → blocked
```

### 14.3 service 测试

覆盖：

```text
上游 StrategySignal 不存在 → blocked + AlertEvent
StrategySignal 不可用 → blocked + AlertEvent
QualityResult 缺失 → blocked + AlertEvent
QualityResult 不允许下游 → blocked + AlertEvent
DecisionRuleDefinition 缺失/禁用/hash mismatch → blocked + AlertEvent
正常 TARGET_POSITION → created + allows_downstream=True
正常 NO_TARGET_CHANGE → created + allows_downstream=False，不进入订单链路
正常 NO_TRADE → created + allows_downstream=False，不进入订单链路
idempotency：相同输入返回已有 snapshot
不同 rule version 生成不同 snapshot
不同 target_position_ratio 生成不同 snapshot
failed 异常路径 → failed + AlertEvent
```

### 14.4 hash 测试

必须证明：

```text
hash 包含 target_intent / target_position_ratio / rule hash / lineage
hash 不包含 BinanceSyncRun / position snapshot
hash 不包含任何订单动作输入
```

### 14.5 command 测试

覆盖：

```text
build_decision_snapshot 正常生成 target intent
--dry-run 不写库、不告警
命令输出不包含订单动作
命令不接收账户、持仓、订单、风控或执行参数
```

### 14.6 边界测试

至少增加断言或代码结构检查：

```text
006A service 不导入 apps.binance_account_sync
006A service 不导入 apps.price_snapshot
006A service 不导入 apps.order_plan
006A service 不导入 apps.risk_check
006A service 不导入 apps.execution
006A command 不调用 Binance client
```

如项目已有架构测试，可加入架构测试。

---

## 15. 与下游的接口约定

006A 完成后，007A OrderPlan 只能消费：

```text
DecisionSnapshot.status = created
DecisionSnapshot.is_usable = True
DecisionSnapshot.allows_downstream = True
DecisionSnapshot.target_intent = TARGET_POSITION
DecisionSnapshot.target_position_ratio is not null
DecisionSnapshot.expires_at 未过期
DecisionSnapshot.target_schema_version = v2
```

007A OrderPlan 不得消费任何订单动作字段或持仓上下文字段。

RiskCheck 不直接消费 DecisionSnapshot。

---

## 16. 需要同步检查的文档

实现本计划时，需要同步检查：

```text
docs/requirements/decision_snapshot.md
docs/architecture/data_flow.md
docs/rules/project_invariants.md
AGENTS.md
```

这些文档必须保持一致：

```text
DecisionSnapshot
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheck
```

---

## 17. 非目标

本阶段不实现：

```text
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
PriceSnapshot
Binance Account Sync
active_order_guard
订单告警链路
真实下单
撤单
查订单
查成交
```

这些由后续阶段处理。

---

## 18. 验收标准

实现完成后至少满足：

```text
1. 006A 不读取账户、持仓、PriceSnapshot 或 Binance 同步结果。
2. 006A 不输出买卖、开平仓、加减仓、反手等订单动作。
3. 006A 输出 target_intent。
4. TARGET_POSITION 时输出合法 target_position_ratio。
5. NO_TARGET_CHANGE / NO_TRADE 时 target_position_ratio 为 null。
6. target_position_ratio = 0.0 表示目标空仓，不等于 NO_TARGET_CHANGE。
7. DecisionPolicy 版本化、可 seed、可 hash、可审计。
8. 默认 policy 行为由 params 控制，不在 service 主流程硬编码。
9. DecisionSnapshot hash / idempotency 不包含 BinanceSyncRun、账户、持仓或订单动作输入。
10. build_decision_snapshot command 不接收账户、持仓、订单、风控或执行参数。
11. dry-run 不写库、不写 AlertEvent。
12. blocked / failed 写 AlertEvent。
13. 正常 NO_TARGET_CHANGE / NO_TRADE 不进入 OrderPlan。
14. 006A 不导入 Binance Account Sync / PriceSnapshot / OrderPlan / RiskCheck / Execution 业务逻辑。
15. `python manage.py makemigrations --check --dry-run` 通过。
16. `python manage.py test` 通过。
17. `pytest` 通过。
```

---

## 19. 完成后汇总要求

Codex 完成实现后需要汇总：

```text
1. 修改了哪些文件。
2. 是否新增或修改 migration。
3. 新 target_intent / target_position_ratio 如何实现。
4. 默认 DecisionPolicy 参数是什么。
5. hash / idempotency 包含哪些字段，明确不包含哪些越界输入。
6. management command 如何设计。
7. 新增 / 修改了哪些测试。
8. 测试结果。
9. 是否发现需求与实现边界冲突。
10. 是否有任何越界实现。
```
