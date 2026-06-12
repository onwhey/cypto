# Plan 005A：StrategySignal 策略信号层

> 当前阶段：005A StrategySignal  
> 前置阶段：004B AtomicSignal 已完成  
> 需求来源：`docs/requirements/strategy_signals.md`  
> 核心目标：建立配置化策略注册与路由架构，基于可用 AtomicSignal 生成可审计、可追溯、默认阻断的 StrategySignal。  
> 核心原则：fail-closed（默认阻断）。

---

## 0. 执行边界

本阶段只实现 StrategySignal 策略信号层基础框架。

本阶段目标：

```text
基于 usable AtomicSignalSet / AtomicSignalValue
通过 StrategyRouteRule 选择 StrategyDefinition
执行配置化策略聚合算法 atomic_vote:v1
生成 StrategySignal
保存 routing_snapshot / input_snapshot / aggregation_snapshot / evidence_text_zh
为后续 DecisionSnapshot / RiskCheck 提供可追溯、可审计、默认阻断的策略级市场判断
```

本阶段不实现交易决策，不实现风控，不生成订单意图。

005A 的输出只回答：

```text
当前是否形成合格的策略级市场判断
```

005A 不回答：

```text
是否应该真实交易
下多少仓位
是否开仓 / 平仓
是否下单
```

---

## 1. 前置条件

开始前必须确认以下文件已存在并已进入项目：

```text
docs/requirements/strategy_signals.md
```

开始前必须阅读：

```text
AGENTS.md
docs/rules/project_invariants.md
docs/requirements/01_project_scope.md
docs/requirements/02_system_capabilities.md
docs/requirements/feature_layer.md
docs/requirements/atomic_signals.md
docs/requirements/strategy_signals.md
docs/architecture/data_flow.md
docs/implementation/004B_atomic_signal.md
apps/atomic_signal/models.py
apps/atomic_signal/services.py
apps/atomic_signal/default_atomic_signal_definitions.py
apps/atomic_signal/registry.py
apps/atomic_signal/calculators/feature_compare.py
```

如本 plan 与 `AGENTS.md` 或 `docs/rules/project_invariants.md` 冲突，以后两者为准。

---

## 2. 本阶段明确不做

005A 禁止实现：

```text
DecisionSnapshot
RiskCheck
CandidateOrderIntent / ApprovedOrderIntent
Execution
账户读取
当前持仓读取
订单读取
成交读取
仓位建议
杠杆
止损
止盈
entry_price
position_size
订单动作枚举
真实交易
回测收益评估
模拟盘
参数扫描
UI / 后台页面
动态权重
自动市场环境路由
多策略自动切换
fallback 策略执行
scaler 仓位缩放
```

005A 禁止调用：

```text
Binance REST
WebSocket
Kline4h / Kline1d 查询逻辑
MarketSnapshotService
FeatureLayerService
AtomicSignalService
DataQualityService
BackfillService
Risk / Trading / Execution 相关服务
Hermes webhook
notifications service 直接发送通知
```

005A 只能读取：

```text
AtomicSignalSet
AtomicSignalValue
AtomicSignalDefinition
StrategyDefinition
StrategyRouteRule
```

可以通过外键或冗余字段追溯：

```text
FeatureSet
MarketSnapshot
trace_id
market identity
```

但不得重新计算上游数据。

---

## 3. 新增 app

新增 Django app：

```text
apps.strategy_signal
```

并在 `config/settings.py` 注册：

```text
apps.strategy_signal
```

建议目录：

```text
apps/strategy_signal/
  __init__.py
  apps.py
  constants.py
  models.py
  hashes.py
  selectors.py
  services.py
  router.py
  default_strategy_definitions.py
  default_route_rules.py
  definition_seed.py
  route_rule_seed.py
  registry.py
  calculators/
    __init__.py
    base.py
    atomic_vote.py
  management/
    __init__.py
    commands/
      __init__.py
      seed_strategy_definitions.py
      seed_strategy_route_rules.py
      build_strategy_signal.py
  migrations/
```

命名说明：

```text
strategy_signal = app 名
StrategyDefinition = 策略注册表
StrategyRouteRule = 策略选择器 / 路由表
StrategySignal = 策略级信号结果
StrategyRouterService = 路由服务
StrategySignalService = 策略信号生成服务
```

---

## 4. 核心模型

### 4.1 StrategyDefinition

新增模型：

```text
StrategyDefinition
```

用途：

```text
注册系统中有哪些策略
定义策略使用哪个聚合算法版本
定义策略允许使用哪些 AtomicSignal
定义 required signals / gatekeepers
定义 static_weights / threshold / conflict_policy
定义该策略是否 active + enabled
```

建议字段：

```text
strategy_code
strategy_name
strategy_version
strategy_mode
algorithm_name
algorithm_version
params
params_hash
definition_hash
status
enabled
description
created_at_utc
updated_at_utc
```

字段语义：

```text
strategy_code = 策略业务编码，例如 trend_following_v1
strategy_version = 策略参数/定义版本，例如 v1
strategy_mode = 策略模式，例如 trend_following / mean_reversion / observe_only
algorithm_name = 聚合算法名，005A 第一版为 atomic_vote
algorithm_version = 聚合算法版本，005A 第一版为 v1
params = 策略参数 JSON
params_hash = canonical params hash
definition_hash = 完整定义 hash
status = draft / active / deprecated / retired / disabled
enabled = 是否允许被 StrategyRouteRule 选中并执行
```

`params` 至少支持：

```text
allowed_signal_codes
required_signal_codes
required_gatekeeper_codes
optional_gatekeeper_codes
static_weights
min_voter_count
min_total_strength
direction_threshold
conflict_policy
neutral_policy
```

示例：

```json
{
  "allowed_signal_codes": [
    "sma_4h_20_above_sma_4h_60"
  ],
  "required_signal_codes": [],
  "required_gatekeeper_codes": [],
  "optional_gatekeeper_codes": [],
  "static_weights": {
    "sma_4h_20_above_sma_4h_60": 1.0
  },
  "min_voter_count": 1,
  "min_total_strength": 0.0,
  "direction_threshold": 0.2,
  "conflict_policy": "allow_neutral",
  "neutral_policy": "allow_neutral"
}
```

必须实现约束：

```text
status=active 且 enabled=True 才允许被生产路由选中
enabled=False 的 StrategyDefinition 不得生成 usable StrategySignal
retired / disabled / deprecated 不得被生产路由选中
params 必须是 dict
allowed_signal_codes 必须是 list[str]
required_signal_codes 必须是 allowed_signal_codes 的子集
required_gatekeeper_codes 必须是 allowed_signal_codes 的子集
optional_gatekeeper_codes 必须是 allowed_signal_codes 的子集
static_weights 的 key 必须属于 allowed_signal_codes
static_weights 的 value 必须 >= 0
min_voter_count 必须 >= 0
direction_threshold 必须 >= 0
```

建议唯一性：

```text
strategy_code + strategy_version + algorithm_name + algorithm_version + params_hash 唯一
definition_hash 唯一
```

生命周期规则：

```text
StrategyDefinition 不允许物理删除
删除/停用只能通过 retired / disabled / enabled=False 表达
已用于 production 的 strategy_code / strategy_version / algorithm_name / algorithm_version / params / params_hash / definition_hash 不得被 seed 覆盖
参数变化应新增 strategy_version 或新 definition
算法变化应新增 algorithm_version
```

可以通过 model `delete()` / queryset `delete()` 防止常规 ORM 删除。

---

### 4.2 StrategyRouteRule

新增模型：

```text
StrategyRouteRule
```

用途：

```text
作为策略选择器
描述当前该运行哪个 StrategyDefinition
避免 StrategyRouterService 硬编码 if-else 绑定具体策略
支持后续从 fixed_default 扩展到 regime_based / context_based
```

建议字段：

```text
route_rule_code
route_rule_version
router_mode
priority
match_conditions
selected_strategy_code
selected_strategy_version
fallback_strategy_code
fallback_strategy_version
fallback_policy
status
enabled
valid_from_utc
valid_to_utc
params
params_hash
definition_hash
created_at_utc
updated_at_utc
```

字段语义：

```text
route_rule_code = 路由规则业务编码，例如 default_route_v1
route_rule_version = 路由规则版本
router_mode = fixed_default / regime_based / context_based / multi_condition / soft_route
priority = 匹配优先级，数值越大优先级越高，或按项目约定固定方向
match_conditions = 路由匹配条件，005A fixed_default 可为空 dict
selected_strategy_code = 选中的 StrategyDefinition.strategy_code
selected_strategy_version = 选中的 StrategyDefinition.strategy_version
fallback_strategy_code = fallback 策略编码，005A 只预留不执行
fallback_policy = none / explicit_only
status = draft / active / deprecated / retired / disabled
enabled = 是否允许该路由规则参与生产路由
```

005A 第一版必须支持：

```text
router_mode = fixed_default
```

含义：

```text
不根据市场环境自动切换策略
通过 StrategyRouteRule 固定选择一个默认策略
```

示例：

```json
{
  "route_rule_code": "default_route_v1",
  "route_rule_version": "v1",
  "router_mode": "fixed_default",
  "priority": 100,
  "match_conditions": {},
  "selected_strategy_code": "trend_following_v1",
  "selected_strategy_version": "v1",
  "fallback_strategy_code": null,
  "fallback_strategy_version": null,
  "fallback_policy": "none",
  "status": "active",
  "enabled": true
}
```

必须实现约束：

```text
status=active 且 enabled=True 才参与生产路由
router_mode=fixed_default 时 match_conditions 必须为空 dict 或仅包含无副作用配置
selected_strategy_code / selected_strategy_version 必填
fallback_policy 默认为 none
005A 不执行 fallback
valid_from_utc / valid_to_utc 如果存在，当前时间必须落在窗口内
```

建议唯一性：

```text
route_rule_code + route_rule_version + params_hash 唯一
definition_hash 唯一
```

生命周期规则：

```text
StrategyRouteRule 不允许物理删除
路由规则变更应新增 route_rule_version 或新 route_rule_code
已用于 production 的 match_conditions / selected_strategy_code / selected_strategy_version / params_hash / definition_hash 不得被 seed 覆盖
停用只能通过 disabled / retired / enabled=False 表达
```

---

### 4.3 StrategySignal

新增模型：

```text
StrategySignal
```

用途：

```text
表示某个 AtomicSignalSet 经 StrategyRouteRule 选中 StrategyDefinition 后生成的策略级市场判断
```

建议字段：

```text
strategy_signal_key
atomic_signal_set
feature_set
market_snapshot
strategy_definition
strategy_route_rule
exchange
market_type
symbol
base_timeframe
higher_timeframe
strategy_code
strategy_version
strategy_mode
algorithm_name
algorithm_version
params_hash
definition_hash
route_rule_code
route_rule_version
router_mode
route_params_hash
route_definition_hash
direction
strength
confidence
status
is_usable
allows_downstream
blocked_reason
error_code
error_message
used_atomic_signal_value_ids
input_snapshot
routing_snapshot
role_group_snapshot
aggregation_snapshot
conflict_snapshot
evidence_items
evidence_text_zh
strategy_signal_schema_version
trace_id
trigger_source
latency_ms
created_at_utc
updated_at_utc
```

必须冗余保存：

```text
strategy_code
strategy_version
strategy_mode
algorithm_name
algorithm_version
params_hash
definition_hash
route_rule_code
route_rule_version
router_mode
route_params_hash
route_definition_hash
market identity
trace_id
```

原因：

```text
即使 StrategyDefinition / StrategyRouteRule 后续状态变化，历史 StrategySignal 仍可读、可追溯、可审计。
```

`direction` 允许值：

```text
bullish
bearish
neutral
none
```

语义：

```text
bullish = 策略级偏多市场判断
bearish = 策略级偏空市场判断
neutral = 有效中性判断
none = 无有效策略方向，通常用于 blocked / failed
```

`status` 允许值：

```text
created
blocked
failed
```

语义：

```text
created → 计算成功，可根据 is_usable / allows_downstream 进入下游
blocked → 输入或配置不满足策略生成条件，不允许下游
failed → 服务异常或结果校验失败，不允许下游
```

下游只允许消费：

```text
status = created
is_usable = True
allows_downstream = True
```

必须实现约束：

```text
strength 范围 0~1
confidence 范围 0~1
created 时 direction 不得为 none
blocked / failed 时 direction 应为 none
blocked / failed 时 is_usable=False 且 allows_downstream=False
created + allows_downstream=True 时 evidence_items 必填
evidence_text_zh 必填
used_atomic_signal_value_ids 必填或明确为空原因
input_snapshot / routing_snapshot / aggregation_snapshot 必填
```

建议唯一性：

```text
strategy_signal_key 唯一
atomic_signal_set + strategy_definition + strategy_route_rule + strategy_signal_schema_version 唯一
```

`strategy_signal_key` 建议基于：

```text
atomic_signal_set_id
atomic_signal_set_key
strategy_definition_id
strategy_code
strategy_version
algorithm_name
algorithm_version
params_hash
strategy_route_rule_id
route_rule_code
route_rule_version
route_params_hash
strategy_signal_schema_version
```

生成 SHA-256。

---

## 5. constants

新增常量文件：

```text
apps/strategy_signal/constants.py
```

至少定义：

```text
StrategyDefinitionStatus
StrategyRouteRuleStatus
StrategySignalStatus
StrategySignalDirection
StrategySignalRole
StrategyRouterMode
StrategyFallbackPolicy
StrategyConflictPolicy
StrategyNeutralPolicy
StrategyErrorCode
StrategyAlertType
DEFAULT_STRATEGY_SIGNAL_SCHEMA_VERSION
DEFAULT_STRATEGY_ROUTER_MODE
DEFAULT_STRATEGY_DIRECTION_THRESHOLD
DEFAULT_STRATEGY_MIN_VOTER_COUNT
```

建议默认：

```text
DEFAULT_STRATEGY_SIGNAL_SCHEMA_VERSION = "strategy_signal.v1"
DEFAULT_STRATEGY_ROUTER_MODE = "fixed_default"
DEFAULT_STRATEGY_DIRECTION_THRESHOLD = 0.2
DEFAULT_STRATEGY_MIN_VOTER_COUNT = 1
```

错误码至少包含：

```text
atomic_signal_set_unusable
no_active_route_rule
no_matching_route_rule
multiple_route_rules_conflict
route_rule_invalid
selected_strategy_not_found
selected_strategy_disabled
selected_strategy_not_active
selected_strategy_retired
selected_strategy_unavailable
strategy_definition_invalid
no_participating_atomic_signals
no_valid_voter
required_signal_missing
required_gatekeeper_missing
required_gatekeeper_invalid
gatekeeper_blocked
direction_conflict_blocked
strategy_signal_validation_failed
strategy_signal_calculation_failed
```

Alert 类型至少包含：

```text
strategy_signal_blocked
strategy_signal_failed
```

---

## 6. settings / .env.example

在 `config/settings.py` 增加配置入口：

```text
STRATEGY_SIGNAL_SCHEMA_VERSION
STRATEGY_DEFAULT_ROUTER_MODE
STRATEGY_DEFAULT_DIRECTION_THRESHOLD
STRATEGY_DEFAULT_MIN_VOTER_COUNT
```

`.env.example` 增加示例：

```text
STRATEGY_SIGNAL_SCHEMA_VERSION=strategy_signal.v1
STRATEGY_DEFAULT_ROUTER_MODE=fixed_default
STRATEGY_DEFAULT_DIRECTION_THRESHOLD=0.2
STRATEGY_DEFAULT_MIN_VOTER_COUNT=1
```

注意：

```text
具体默认策略不能写死在 settings.py
默认策略必须来自 StrategyRouteRule seed / 数据库配置
```

---

## 7. Hash 规则

新增：

```text
apps/strategy_signal/hashes.py
```

至少实现：

```text
canonical_json
compute_params_hash(params)
compute_strategy_definition_hash(definition_fields)
compute_route_rule_params_hash(params)
compute_route_rule_definition_hash(route_rule_fields)
compute_strategy_signal_key(...)
```

### 7.1 params_hash

建议：

```text
sha256(canonical_json(params))
```

### 7.2 StrategyDefinition definition_hash

建议包含：

```text
strategy_code
strategy_version
strategy_mode
algorithm_name
algorithm_version
params_hash
status-independent identity fields
```

不要包含：

```text
description
created_at_utc
updated_at_utc
enabled
status
```

原因：

```text
status / enabled 是运行状态，不应改变策略定义身份。
```

### 7.3 StrategyRouteRule definition_hash

建议包含：

```text
route_rule_code
route_rule_version
router_mode
priority
match_conditions
selected_strategy_code
selected_strategy_version
fallback_strategy_code
fallback_strategy_version
fallback_policy
params_hash
valid_from_utc
valid_to_utc
```

不要包含：

```text
description
created_at_utc
updated_at_utc
enabled
status
```

### 7.4 StrategySignal key

建议基于：

```text
atomic_signal_set_id
atomic_signal_set_key
strategy_definition_id
strategy_route_rule_id
strategy_code
strategy_version
algorithm_name
algorithm_version
params_hash
route_rule_code
route_rule_version
route_params_hash
strategy_signal_schema_version
```

生成 SHA-256。

---

## 8. 默认定义与 seed

### 8.1 default_strategy_definitions.py

新增：

```text
apps/strategy_signal/default_strategy_definitions.py
```

职责：

```text
保存默认 StrategyDefinition 模板
不读数据库
不写数据库
不参与正式计算
```

正式计算只能读取数据库中的 `StrategyDefinition`。

005A 第一版默认 seed 一个策略：

```text
strategy_code = trend_following_v1
strategy_name = 趋势跟踪默认策略
strategy_version = v1
strategy_mode = trend_following
algorithm_name = atomic_vote
algorithm_version = v1
status = active
enabled = True
```

params 建议：

```json
{
  "allowed_signal_codes": [
    "sma_4h_20_above_sma_4h_60"
  ],
  "required_signal_codes": [],
  "required_gatekeeper_codes": [],
  "optional_gatekeeper_codes": [],
  "static_weights": {
    "sma_4h_20_above_sma_4h_60": 1.0
  },
  "min_voter_count": 1,
  "min_total_strength": 0.0,
  "direction_threshold": 0.2,
  "conflict_policy": "allow_neutral",
  "neutral_policy": "allow_neutral"
}
```

注意：

```text
该策略 active + enabled 不代表一定能生成 created StrategySignal。
如果当前 AtomicSignalDefinition 仍 participates_in_strategy=False，StrategySignal 必须 blocked。
blocked_reason = no_participating_atomic_signals 或 no_valid_voter
```

005A 不强制 seed B/C 空策略。

后续新增 B/C/D 策略时：

```text
新增 StrategyDefinition seed 或数据库配置
新增或修改 StrategyRouteRule
不改 StrategyRouterService 主逻辑
```

### 8.2 default_route_rules.py

新增：

```text
apps/strategy_signal/default_route_rules.py
```

职责：

```text
保存默认 StrategyRouteRule 模板
不读数据库
不写数据库
不参与正式计算
```

005A 第一版默认 seed 一条 fixed_default 路由：

```text
route_rule_code = default_route_v1
route_rule_version = v1
router_mode = fixed_default
priority = 100
match_conditions = {}
selected_strategy_code = trend_following_v1
selected_strategy_version = v1
fallback_strategy_code = null
fallback_strategy_version = null
fallback_policy = none
status = active
enabled = True
```

### 8.3 seed_strategy_definitions

新增 command：

```bash
python manage.py seed_strategy_definitions
```

职责：

```text
读取 default_strategy_definitions.py
计算 params_hash / definition_hash
按 strategy_code + strategy_version + algorithm_name + algorithm_version + params_hash 幂等写入 StrategyDefinition 表
```

规则：

```text
已存在相同定义不得重复插入
已存在时只能更新 strategy_name / description 等非身份字段
不得覆盖 algorithm_name / algorithm_version / params / params_hash / definition_hash
不得把 retired / disabled 的 StrategyDefinition 自动恢复为 active
不得修改 enabled 的人工配置，除非显式 force
```

seed 命令不生成：

```text
StrategySignal
AtomicSignalSet
AtomicSignalValue
```

seed 命令不调用：

```text
StrategySignalService
StrategyRouterService
AtomicSignalService
FeatureLayerService
Binance
Kline
```

### 8.4 seed_strategy_route_rules

新增 command：

```bash
python manage.py seed_strategy_route_rules
```

职责：

```text
读取 default_route_rules.py
计算 params_hash / definition_hash
按 route_rule_code + route_rule_version + params_hash 幂等写入 StrategyRouteRule 表
```

规则：

```text
已存在相同定义不得重复插入
已存在时只能更新 description 等非身份字段
不得覆盖 match_conditions / selected_strategy_code / selected_strategy_version / params_hash / definition_hash
不得把 retired / disabled 的 StrategyRouteRule 自动恢复为 active
不得修改 enabled 的人工配置，除非显式 force
```

---

## 9. Algorithm Registry

新增：

```text
apps/strategy_signal/registry.py
```

`StrategyAlgorithmRegistry` 根据：

```text
algorithm_name + algorithm_version
```

找到对应 calculator。

禁止在 `StrategySignalService` 中写大量 if/else 分发策略算法。

005A 第一版注册：

```text
atomic_vote:v1
```

---

## 10. Calculator 架构

### 10.1 Base

新增：

```text
apps/strategy_signal/calculators/base.py
```

建议结构：

```text
StrategyCalculationContext
CalculatedStrategySignal
StrategyCalculationError
```

calculator 只能接收：

```text
StrategyDefinition
StrategyRouteRule
AtomicSignalSet
已过滤 / 分组后的 AtomicSignalValue 列表
```

calculator 不得：

```text
读数据库
写数据库
请求外部服务
读取 Kline
调用 FeatureLayerService
调用 AtomicSignalService
生成 DecisionSnapshot
调用 RiskCheck
生成订单
```

---

### 10.2 atomic_vote:v1

新增：

```text
apps/strategy_signal/calculators/atomic_vote.py
```

实现：

```text
AtomicVoteV1Calculator
algorithm_name = atomic_vote
algorithm_version = v1
```

输入：

```text
role=voter 的 AtomicSignalValue
StrategyDefinition.params.static_weights
StrategyDefinition.params.direction_threshold
StrategyDefinition.params.min_voter_count
StrategyDefinition.params.min_total_strength
StrategyDefinition.params.conflict_policy
StrategyDefinition.params.neutral_policy
```

计算：

```text
effective_strength = strength × confidence × static_weight
bullish_total = sum(effective_strength where direction = bullish)
bearish_total = sum(effective_strength where direction = bearish)
neutral_count = count(direction = neutral)
```

方向判断：

```text
if bullish_total - bearish_total > direction_threshold:
    direction = bullish
elif bearish_total - bullish_total > direction_threshold:
    direction = bearish
else:
    direction = neutral
```

confidence：

```text
confidence = abs(bullish_total - bearish_total) / (bullish_total + bearish_total + epsilon)
```

strength：

```text
strength = min(max(bullish_total, bearish_total), 1)
```

结果必须包含：

```text
direction
strength
confidence
aggregation_snapshot
conflict_snapshot
evidence_items
evidence_text_zh
```

禁止输出：

```text
position_size
leverage
entry_price
stop_loss
take_profit
order_intent
订单动作枚举
```

---

## 11. Selectors

新增：

```text
apps/strategy_signal/selectors.py
```

至少包含：

```text
get_usable_atomic_signal_set(...)
get_atomic_signal_values_for_set(atomic_signal_set)
get_active_route_rules(now_utc)
get_strategy_definition_by_code_version(strategy_code, strategy_version)
get_existing_strategy_signal(strategy_signal_key)
```

规则：

```text
get_active_route_rules 只读取数据库中 status=active 且 enabled=True 的 StrategyRouteRule
get_strategy_definition_by_code_version 必须校验 status=active 且 enabled=True，或由 RouterService 显式校验
不得读取 default_strategy_definitions.py
不得读取 default_route_rules.py
```

---

## 12. StrategyRouterService

新增：

```text
apps/strategy_signal/router.py
```

主入口建议：

```text
select_strategy(
    atomic_signal_set,
    trace_id=None,
    trigger_source="manual",
    now_utc=None,
)
```

返回结构建议：

```text
StrategyRouteResult
```

包含：

```text
is_routed
status
error_code
error_message
strategy_definition
route_rule
routing_snapshot
```

### 12.1 路由流程

```text
1. 接收 AtomicSignalSet
2. 读取 active + enabled StrategyRouteRule
3. 过滤 valid_from_utc / valid_to_utc
4. 按 priority 排序
5. 匹配 match_conditions
6. 005A 第一版仅支持 fixed_default
7. 如果没有匹配规则 → blocked
8. 如果匹配多条且优先级冲突 → blocked
9. 根据 selected_strategy_code / selected_strategy_version 查找 StrategyDefinition
10. 校验 StrategyDefinition active + enabled
11. 返回 StrategyDefinition + StrategyRouteRule + routing_snapshot
```

### 12.2 fixed_default 匹配规则

```text
router_mode = fixed_default
match_conditions = {}
直接匹配
```

禁止：

```text
硬编码 selected_strategy_code
硬编码趋势市 → 趋势策略
硬编码震荡市 → 震荡策略
自动选择其他 enabled 策略
```

### 12.3 策略不可用处理

如果 RouteRule 选中的 StrategyDefinition：

```text
不存在
disabled
not active
retired
```

必须返回 blocked：

```text
blocked_reason = selected_strategy_unavailable
allows_downstream = False
```

不得自动寻找其他策略。

### 12.4 fallback

005A 第一版不执行 fallback。

即使字段存在：

```text
fallback_strategy_code
fallback_strategy_version
fallback_policy
```

也只记录，不运行。

如 RouteRule 选中策略不可用：

```text
直接 blocked
```

---

## 13. StrategySignalService

新增：

```text
apps/strategy_signal/services.py
```

主入口建议：

```text
build_for_atomic_signal_set(
    atomic_signal_set_id=None,
    atomic_signal_set_key=None,
    trace_id=None,
    trigger_source="manual",
    dry_run=False,
)
```

### 13.1 主流程

```text
1. 接收 atomic_signal_set_id 或 atomic_signal_set_key
2. 读取 AtomicSignalSet
3. 校验 AtomicSignalSet.status=created 且 is_usable=True 且 allows_downstream=True
4. 调用 StrategyRouterService.select_strategy
5. 如果路由 blocked → 生成 blocked StrategySignal 或 dry-run 摘要
6. 读取 AtomicSignalValue
7. 过滤：status=created, is_valid=True, participates_in_strategy=True
8. 应用 StrategyDefinition.params.allowed_signal_codes
9. 校验 required_signal_codes
10. 按 role 分组：voter / gatekeeper / filter / descriptor / scaler
11. 校验 role 与 AtomicSignalDefinition / AtomicSignalValue 冗余字段一致
12. 执行 required_gatekeeper 检查
13. optional_gatekeeper 第一版只记录
14. filter 第一版只记录
15. descriptor 第一版只记录
16. scaler 第一版只记录为 ignored_or_reserved，不参与计算
17. 校验 voter 数量和强度
18. 计算 strategy_signal_key
19. 幂等检查已有 StrategySignal
20. 调用 StrategyAlgorithmRegistry 找到 atomic_vote:v1 calculator
21. 执行聚合算法
22. 校验 direction / strength / confidence / evidence
23. 非 dry-run 使用 transaction.atomic 写入 StrategySignal
24. blocked / failed 写 AlertEvent
25. 返回 StrategySignal 或 dry-run 摘要
```

### 13.2 输入过滤规则

只使用：

```text
AtomicSignalSet.status = created
AtomicSignalSet.is_usable = True
AtomicSignalSet.allows_downstream = True
AtomicSignalValue.status = created
AtomicSignalValue.is_valid = True
AtomicSignalValue.participates_in_strategy = True
AtomicSignalValue.signal_code 在 allowed_signal_codes 中
```

无效信号处理：

```text
is_valid=False → 忽略，并记录 ignored_signals
participates_in_strategy=False → 忽略，并记录 observer / ignored 原因
不在 allowed_signal_codes → 忽略，并记录 ignored_by_strategy_whitelist
```

如果忽略后没有可用 voter：

```text
status = blocked
blocked_reason = no_valid_voter
allows_downstream = False
```

如果没有任何 participating atomic signals：

```text
status = blocked
blocked_reason = no_participating_atomic_signals
allows_downstream = False
```

### 13.3 role 使用规则

005A 不重新定义 role，只消费 AtomicSignal 层产出的 role。

```text
voter → 参与方向聚合
gatekeeper → 策略信号前置阻断
filter → 第一版只记录，不过滤分支
descriptor → 第一版只记录上下文
scaler → 第一版只记录 ignored_or_reserved，不参与仓位/强度计算
```

role 不匹配场景：

```text
StrategyDefinition.params 把某 signal_code 配进 required_gatekeeper_codes
但 AtomicSignalValue.role 不是 gatekeeper
→ blocked / failed
```

建议第一版处理为：

```text
blocked_reason = role_mismatch
```

### 13.4 gatekeeper 规则

```text
required_gatekeeper 缺失 → blocked
required_gatekeeper is_valid=False → blocked
required_gatekeeper status!=created → blocked
required_gatekeeper direction / value 表示阻断 → blocked
optional_gatekeeper 第一版只记录，不影响结果
```

gatekeeper 只能输出 StrategySignal blocked。

禁止输出：

```text
订单动作枚举
平仓
开仓
仓位
订单
```

### 13.5 scaler 规则

005A 不使用 scaler。

如果出现 role=scaler：

```text
记录到 role_group_snapshot.scaler
标记 ignored_or_reserved
不得读取 scale_factor 参与 position_size / leverage / order sizing
不得影响 StrategySignal.strength
```

---

## 14. 快照与证据

StrategySignal 必须保存：

```text
input_snapshot
routing_snapshot
role_group_snapshot
aggregation_snapshot
conflict_snapshot
evidence_items
evidence_text_zh
used_atomic_signal_value_ids
```

### 14.1 input_snapshot

至少包含：

```text
atomic_signal_set_id
atomic_signal_set_key
feature_set_id
feature_set_key
market_snapshot_id
snapshot_key
input_value_count
participating_value_count
ignored_value_count
ignored_reasons
```

### 14.2 routing_snapshot

至少包含：

```text
route_rule_id
route_rule_code
route_rule_version
router_mode
matched_conditions
selected_strategy_id
selected_strategy_code
selected_strategy_version
fallback_used
fallback_reason
route_reason
```

### 14.3 role_group_snapshot

至少包含：

```text
voter_signal_codes
gatekeeper_signal_codes
filter_signal_codes
descriptor_signal_codes
scaler_signal_codes
ignored_signal_codes
role_mismatch_signal_codes
```

### 14.4 aggregation_snapshot

至少包含：

```text
algorithm_name
algorithm_version
static_weights
effective_strength_by_signal
bullish_total
bearish_total
neutral_count
bullish_count
bearish_count
voter_count
min_voter_count
min_total_strength
direction_threshold
```

### 14.5 conflict_snapshot

至少包含：

```text
conflict_detected
conflict_reason
conflict_policy
bullish_total
bearish_total
score_difference
is_severe_conflict
```

### 14.6 evidence_text_zh

必须是中文自然语言。

允许：

```text
本次策略信号使用 1 个有效 voter 原子信号，多头有效强度 0.80，空头有效强度 0.00，未检测到方向冲突，策略级方向判断为偏多。
```

禁止出现：

```text
开多
开空
买入
卖出
止损
止盈
仓位
杠杆
下单
订单动作枚举
```

---

## 15. 幂等规则

StrategySignal 必须幂等。

流程：

```text
1. 在写库前计算 strategy_signal_key
2. 查询是否已存在相同 strategy_signal_key
3. 如果存在，默认返回已有 StrategySignal
4. 不重复写 StrategySignal
5. 不重复写 AlertEvent
```

第一版不实现：

```text
--force 重算
覆盖已有结果
删除重建
```

后续如需要重算，应单独设计 P2 / P3。

---

## 16. dry-run

`StrategySignalService.build_for_atomic_signal_set(..., dry_run=True)` 必须支持 dry-run。

dry-run 时：

```text
不写 StrategySignal
不写 AlertEvent
不修改数据库
返回 routing_snapshot
返回 input_snapshot
返回 aggregation_snapshot
返回 conflict_snapshot
返回 evidence_text_zh
返回 blocked / failed 摘要
```

dry-run 仍然应该执行：

```text
输入校验
路由选择
策略定义校验
AtomicSignalValue 过滤
role 分组
gatekeeper 检查
voter 聚合
结果校验
```

---

## 17. AlertEvent

blocked / failed 场景写 AlertEvent。

需要写 AlertEvent 的场景：

```text
strategy_signal_blocked
strategy_signal_failed
no_active_route_rule
no_matching_route_rule
selected_strategy_unavailable
strategy_definition_invalid
no_participating_atomic_signals
no_valid_voter
required_signal_missing
required_gatekeeper_missing
required_gatekeeper_invalid
gatekeeper_blocked
direction_conflict_blocked
strategy_signal_validation_failed
```

不写 AlertEvent 的场景：

```text
正常 created
dry-run
重复幂等返回已有记录
```

规则：

```text
业务模块只写 AlertEvent
不得直接调用 Hermes webhook
不得直接调用 notifications service 发送通知
```

AlertEvent payload 至少包含：

```text
strategy_signal_key
atomic_signal_set_id
strategy_code
strategy_version
route_rule_code
route_rule_version
error_code
blocked_reason
trace_id
trigger_source
```

---

## 18. Management commands

### 18.1 seed_strategy_definitions

```bash
python manage.py seed_strategy_definitions
```

参数建议：

```text
--dry-run
--force
```

第一版 `--force` 如实现，必须非常保守：

```text
不得覆盖身份字段
不得恢复 retired / disabled
不得修改人工 enabled 配置，除非明确 requirements 允许
```

### 18.2 seed_strategy_route_rules

```bash
python manage.py seed_strategy_route_rules
```

参数建议：

```text
--dry-run
--force
```

规则同 StrategyDefinition seed。

### 18.3 build_strategy_signal

```bash
python manage.py build_strategy_signal --atomic-signal-set-id <id>
```

支持参数：

```text
--atomic-signal-set-id
--atomic-signal-set-key
--dry-run
--trace-id
--trigger-source
```

第一版不建议支持：

```text
--force
```

可后续预留但不实现：

```text
--route-rule-code
--strategy-code
```

原因：005A 第一版应通过 StrategyRouteRule 统一路由，不建议命令绕过路由直接指定策略。

---

## 19. Migrations

至少生成 Django migration：

```text
apps/strategy_signal/migrations/0001_initial.py
```

包含：

```text
StrategyDefinition
StrategyRouteRule
StrategySignal
唯一约束
索引
字段 choices
```

建议索引：

```text
StrategyDefinition(strategy_code, strategy_version)
StrategyDefinition(status, enabled)
StrategyDefinition(definition_hash)
StrategyRouteRule(route_rule_code, route_rule_version)
StrategyRouteRule(status, enabled, priority)
StrategyRouteRule(router_mode)
StrategySignal(strategy_signal_key)
StrategySignal(status, is_usable, allows_downstream)
StrategySignal(exchange, market_type, symbol, base_timeframe)
StrategySignal(trace_id)
StrategySignal(created_at_utc)
```

---

## 20. 测试计划

新增测试目录建议：

```text
tests/strategy_signal/
  test_models.py
  test_hashes.py
  test_seed_strategy_definitions.py
  test_seed_strategy_route_rules.py
  test_router_service.py
  test_atomic_vote_calculator.py
  test_strategy_signal_service.py
  test_management_commands.py
```

至少覆盖：

```text
StrategyDefinition params_hash / definition_hash 稳定
StrategyDefinition 生命周期校验
StrategyDefinition required_signal_codes 必须属于 allowed_signal_codes
StrategyDefinition static_weights key 必须属于 allowed_signal_codes
StrategyRouteRule params_hash / definition_hash 稳定
StrategyRouteRule fixed_default 匹配
StrategyRouteRule 选中策略不存在 → blocked
StrategyRouteRule 选中策略 disabled → blocked
StrategyRouteRule 选中策略 retired → blocked
不自动 fallback 到其他策略
AtomicSignalSet 不可用 → blocked
没有 active route rule → blocked
没有 participating atomic signals → blocked
没有有效 voter → blocked
invalid AtomicSignalValue 被忽略
participates_in_strategy=False 被忽略
不在 allowed_signal_codes 的信号被忽略
required signal 缺失 → blocked
required gatekeeper 缺失 → blocked
required gatekeeper invalid → blocked
required gatekeeper 触发阻断 → blocked
role 不匹配 → blocked / failed
有效 bullish voter → created bullish
有效 bearish voter → created bearish
bullish / bearish 差值不足 threshold → created neutral
严重冲突且配置 block → blocked
strength / confidence 越界 → failed
重复执行幂等返回已有记录
dry-run 不写 StrategySignal
dry-run 不写 AlertEvent
blocked / failed 写 AlertEvent
created 不写 AlertEvent
management seed 幂等
management build_strategy_signal 可 dry-run
```

测试必须确认：

```text
005A 不读取 Kline4h / Kline1d
005A 不调用 FeatureLayerService / AtomicSignalService
005A 不调用 Binance / WebSocket
005A 不生成 DecisionSnapshot / OrderPlan / CandidateOrderIntent / ApprovedOrderIntent / RiskCheckResult
005A 不输出 position_size / leverage / stop_loss / take_profit
```

---

## 21. 文档更新

完成实现后需要新增或更新：

```text
docs/implementation/005A_strategy_signal.md
docs/implementation/strategy_signal/README.md
docs/implementation/strategy_signal/atomic_vote.md
docs/implementation/strategy_signal/default_route_v1.md
docs/implementation/strategy_signal/trend_following_v1.md
```

实现文档必须说明：

```text
StrategyDefinition 字段与生命周期
StrategyRouteRule 字段与生命周期
fixed_default 路由行为
atomic_vote:v1 聚合规则
fail-closed 阻断规则
AlertEvent 场景
dry-run 行为
幂等 key 组成
P2 未实现项
```

---

## 22. P2 记录但不实现

以下内容必须在 requirements / implementation 中记录为 P2，但 005A 不实现：

```text
dynamic_weight 动态权重
regime_based 市场环境自动路由
context_based 多条件路由
multi_condition 路由
soft_route 多策略加权路由
fallback 策略执行
filter 复杂分支过滤
descriptor 影响路由或权重
market_regime_descriptor 驱动策略切换
多策略模式自动切换
参数热更新
回测收益评估
paper trading
参数扫描
策略表现统计
scaler 仓位缩放
持仓状态融合
DecisionSnapshot
RiskCheck
CandidateOrderIntent / ApprovedOrderIntent
Execution
UI / 后台页面
```

特别说明：

```text
动态权重必须等回测、样本外验证、成本模型、look-ahead bias 检查完成后再启用。
scaler 不属于 005A，放到后续 OrderPlan / RiskCheck。
```

---

## 23. 推荐实现顺序

建议 Codex 按以下顺序实现，不要跳步：

```text
1. 新增 apps.strategy_signal 并注册 settings
2. 新增 constants.py
3. 新增 models.py：StrategyDefinition / StrategyRouteRule / StrategySignal
4. 生成并检查 migration
5. 新增 hashes.py
6. 新增 default_strategy_definitions.py / default_route_rules.py
7. 新增 seed commands
8. 新增 selectors.py
9. 新增 registry.py 与 calculators/base.py
10. 实现 calculators/atomic_vote.py
11. 实现 router.py：StrategyRouterService
12. 实现 services.py：StrategySignalService
13. 新增 build_strategy_signal management command
14. 补全 tests
15. 更新 docs/implementation
16. 运行 manage.py check
17. 运行 makemigrations --check --dry-run
18. 运行 pytest / manage.py test
```

禁止先做：

```text
UI
回测
风控
订单
交易执行
动态权重
市场环境自动路由
```

---

## 24. 验收标准

005A 完成后必须满足：

```text
StrategyDefinition 可注册策略
StrategyRouteRule 可配置默认策略选择
StrategyRouterService 不硬编码具体策略
StrategySignalService 可执行被路由选中的策略定义
默认 fixed_default 路由可选择 trend_following_v1
RouteRule 选中策略不可用时 blocked，不自动找其他策略
没有可参与 AtomicSignalValue 时 blocked
voter 聚合可生成 bullish / bearish / neutral
required gatekeeper 可阻断
scaler 不参与 005A 计算
动态权重未实现但记录为 P2
StrategySignal 可追溯到 AtomicSignalSet / FeatureSet / MarketSnapshot
routing_snapshot / input_snapshot / aggregation_snapshot / evidence_text_zh 完整
幂等写入
失败不半写
blocked / failed 写 AlertEvent
created / dry-run / 幂等返回不写 AlertEvent
测试覆盖主要成功路径和失败路径
```

验收命令：

```bash
python manage.py check
python manage.py makemigrations --check --dry-run
python manage.py test
pytest
```

---

## 25. 最终边界声明

005A 的最终目标是：

```text
建立配置化策略注册与路由架构，
基于可用 AtomicSignal 生成可审计、可追溯、默认阻断的 StrategySignal。
```

005A 不是：

```text
风控模块
交易决策模块
仓位管理模块
订单模块
回测收益模块
策略优化模块
UI 模块
```

一句话：

```text
StrategySignal 只生成合格的策略级市场判断，不生成交易决策。
```
