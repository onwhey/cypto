# Plan 004B：AtomicSignal 原子信号层

## 0. 执行边界

本阶段只实现 AtomicSignal 原子信号层基础框架。

本阶段目标：

```text
基于 usable FeatureSet / FeatureValue
读取数据库中 enabled=True 的 AtomicSignalDefinition
执行单一、原子化的市场条件判断
生成 AtomicSignalSet / AtomicSignalValue
为后续 StrategySignal 提供可追溯、可审计、可复盘的中间信号
```

本阶段不实现 StrategySignal，不实现 Risk / Trading，不生成交易决策。

---

## 1. 前置条件

开始前必须确认以下文件已存在并已进入项目：

```text
docs/requirements/atomic_signals.md
```

开始前必须阅读：

```text
AGENTS.md
docs/rules/project_invariants.md
docs/requirements/01_project_scope.md
docs/requirements/02_system_capabilities.md
docs/requirements/feature_layer.md
docs/requirements/atomic_signals.md
docs/architecture/data_flow.md
docs/implementation/004A_feature_layer.md
docs/implementation/feature_layer/README.md
apps/feature_layer/models.py
apps/feature_layer/services.py
apps/feature_layer/default_definitions.py
apps/feature_layer/registry.py
```

如本 plan 与 `AGENTS.md` 或 `docs/rules/project_invariants.md` 冲突，以后两者为准。

---

## 2. 本阶段明确不做

004B 禁止实现：

```text
StrategySignal
MarketRegime
DecisionSnapshot
RiskCheck
CandidateOrderIntent / ApprovedOrderIntent
Execution
账户
仓位
订单
自动交易
权重分配
冲突解决
信号聚合
最终仓位缩放
Kline 读取
Feature 重新计算
Binance REST
WebSocket
大模型分析
后台 UI
回测系统
```

004B 禁止输出：

```text
weight
priority
fixed_weight
strategy_weight
final_score
strategy_score
should_long
should_short
entry_price
stop_loss
take_profit
position_size
leverage
order_intent
```

004B 禁止调用：

```text
Binance client
Kline4h / Kline1d 读取逻辑
FeatureLayerService
MarketSnapshotService
DataQualityService
BackfillService
Risk / Trading / Execution 相关服务
Hermes webhook
notifications service
```

AtomicSignal 只能读取：

```text
FeatureSet
FeatureValue
AtomicSignalDefinition
```

---

## 3. 新增 app

新增 Django app：

```text
apps.atomic_signal
```

并在 `config/settings.py` 注册：

```text
apps.atomic_signal
```

建议目录：

```text
apps/atomic_signal/
  __init__.py
  apps.py
  constants.py
  models.py
  hashes.py
  selectors.py
  services.py
  default_atomic_signal_definitions.py
  definition_seed.py
  registry.py
  calculators/
    __init__.py
    base.py
    feature_compare.py
  management/
    __init__.py
    commands/
      __init__.py
      seed_atomic_signal_definitions.py
      build_atomic_signals.py
  migrations/
```

---

## 4. 核心模型

### 4.1 AtomicSignalDefinition

新增模型：

```text
AtomicSignalDefinition
```

用途：

```text
定义系统中有哪些原子信号
定义每个原子信号使用哪个算法版本
定义依赖哪些 FeatureValue
定义是否启用
定义是否允许进入 StrategySignal
```

建议字段：

```text
signal_code
display_name
description
category
role
default_direction
algorithm_name
algorithm_version
params
params_hash
definition_hash
status
enabled
participates_in_strategy
is_required
depends_on_feature_codes
output_type
created_at_utc
updated_at_utc
```

字段语义：

```text
signal_code = 原子信号业务编码
role = voter / descriptor / gatekeeper / scaler / filter
default_direction = bullish / bearish / neutral / none
status = draft / active / deprecated / retired / disabled
enabled = 是否计算该原子信号
participates_in_strategy = 是否允许后续 StrategySignal 使用
params_hash = canonical params hash
definition_hash = 完整定义 hash
depends_on_feature_codes = 依赖的 feature_code 列表
```

必须实现约束：

```text
enabled=False + participates_in_strategy=True 必须被拒绝
status=active 且 enabled=True 才参与 AtomicSignalService 计算
status=active 且 enabled=True 且 participates_in_strategy=True 才允许后续 StrategySignal 使用
```

建议唯一性：

```text
signal_code + algorithm_name + algorithm_version + params_hash 唯一
```

生命周期规则：

```text
AtomicSignalDefinition 不允许物理删除
删除/停用只能通过 retired / disabled / enabled=False 表达
已用于 production 的 algorithm_name / algorithm_version / params / params_hash / definition_hash 不得被 seed 覆盖
```

可以通过 model `delete()` / queryset `delete()` 防止常规 ORM 删除。

---

### 4.2 AtomicSignalSet

新增模型：

```text
AtomicSignalSet
```

用途：

```text
表示某个 FeatureSet 上生成的一批 AtomicSignalValue
```

建议字段：

```text
atomic_signal_set_key
feature_set
feature_set_key
market_snapshot
snapshot_key
signal_schema_version
definition_set_hash
status
is_usable
allows_downstream
active_definition_count
computed_count
valid_count
invalid_count
failed_count
required_failed_count
failure_ratio
error_code
error_message
details
trace_id
trigger_source
created_at_utc
updated_at_utc
```

关系：

```text
AtomicSignalSet 必须绑定 FeatureSet
AtomicSignalSet 应冗余 market_snapshot / snapshot_key / feature_set_key，便于追溯
```

状态：

```text
created
blocked
failed
```

语义：

```text
created → is_usable=True, allows_downstream=True
blocked / failed → is_usable=False, allows_downstream=False
```

幂等规则：

```text
同一个 FeatureSet
+ 同一个 signal_schema_version
+ 同一个 definition_set_hash
→ 只能生成一个 AtomicSignalSet
```

`atomic_signal_set_key` 建议基于：

```text
feature_set_id
feature_set_key
signal_schema_version
definition_set_hash
```

生成 SHA-256。

---

### 4.3 AtomicSignalValue

新增模型：

```text
AtomicSignalValue
```

用途：

```text
表示某个 AtomicSignalSet 中某一个 AtomicSignalDefinition 的计算结果
```

建议字段：

```text
atomic_signal_set
atomic_signal_definition
signal_code
role
direction
strength
confidence
status
is_valid
is_active
participates_in_strategy
algorithm_name
algorithm_version
params_hash
definition_hash
value_bool
value_decimal
value_text
value_json
evidence_items
evidence_text_zh
used_feature_codes
used_feature_value_ids
role_specific_payload
scale_factor
error_code
error_message
timestamp
latency_ms
created_at_utc
updated_at_utc
```

字段要求：

```text
direction = bullish / bearish / neutral / none
strength 范围 0~1
confidence 范围 0~1
evidence_items 必填
evidence_text_zh 必填
used_feature_codes 必填
used_feature_value_ids 必填
```

必须冗余保存：

```text
signal_code
role
algorithm_name
algorithm_version
params_hash
definition_hash
participates_in_strategy
```

原因：

```text
即使 AtomicSignalDefinition 后续状态变化，历史 AtomicSignalValue 仍可读、可追溯。
```

`evidence_text_zh` 必须是中文自然语言说明，供用户审阅、复盘和大模型辅助分析。

禁止在 `evidence_text_zh` 中写：

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
```

`scale_factor` 只允许 `role=scaler` 使用：

```text
scale_factor 只是 StrategySignal / RiskCheck 的参考
scale_factor 不是 position_size
scale_factor 不是 leverage
scale_factor 不是最终仓位
```

---

## 5. constants

新增常量文件：

```text
apps/atomic_signal/constants.py
```

至少定义：

```text
AtomicSignalDefinitionStatus
AtomicSignalSetStatus
AtomicSignalValueStatus
AtomicSignalRole
AtomicSignalDirection
AtomicSignalOutputType
AtomicSignalErrorCode
AtomicSignalAlertType
DEFAULT_ATOMIC_SIGNAL_SCHEMA_VERSION
DEFAULT_ATOMIC_SIGNAL_FAILURE_BLOCK_RATIO
```

建议默认：

```text
DEFAULT_ATOMIC_SIGNAL_SCHEMA_VERSION = "atomic_signal.v1"
DEFAULT_ATOMIC_SIGNAL_FAILURE_BLOCK_RATIO = 0.3
```

失败阈值必须从 settings 读取，不能在业务逻辑中写死。

---

## 6. settings / .env.example

在 `config/settings.py` 增加配置入口：

```text
ATOMIC_SIGNAL_SCHEMA_VERSION
ATOMIC_SIGNAL_FAILURE_BLOCK_RATIO
```

`.env.example` 增加示例：

```text
ATOMIC_SIGNAL_SCHEMA_VERSION=atomic_signal.v1
ATOMIC_SIGNAL_FAILURE_BLOCK_RATIO=0.3
```

---

## 7. 默认定义与 seed

### 7.1 default_atomic_signal_definitions.py

新增：

```text
apps/atomic_signal/default_atomic_signal_definitions.py
```

职责：

```text
保存默认 AtomicSignalDefinition 模板
不读数据库
不写数据库
不参与正式计算
```

正式计算只能读数据库中的 `AtomicSignalDefinition`。

第一版默认信号不要贪多，建议先实现以下 7 个候选中的一部分或全部：

```text
close_4h_above_sma_4h_20
close_4h_above_sma_4h_60
sma_4h_20_above_sma_4h_60
close_4h_breaks_range_4h_60_high
close_4h_below_range_4h_60_low
volume_4h_above_volume_sma_4h_20
close_1d_above_sma_1d_20
```

004B 第一版至少应包含：

```text
sma_4h_20_above_sma_4h_60
```

作为从 FeatureLayer 移出的比较型 bool 条件。

默认信号全部应使用：

```text
enabled=True
participates_in_strategy=False
```

作为观察模式起步，除非 plan / requirements 明确指定某个信号进入 production。

原因：

```text
新增原子信号先收集表现数据，不直接影响正式策略。
```

### 7.2 seed_atomic_signal_definitions

新增 command：

```bash
python manage.py seed_atomic_signal_definitions
```

职责：

```text
读取 default_atomic_signal_definitions.py
计算 params_hash / definition_hash
按 signal_code + algorithm_name + algorithm_version + params_hash 幂等写入 AtomicSignalDefinition 表
```

规则：

```text
已存在相同定义不得重复插入
已存在时只能更新 display_name / description / category 等非身份字段
不得覆盖 algorithm_name / algorithm_version / params / params_hash / definition_hash
不得把 retired / disabled 的 AtomicSignalDefinition 自动恢复为 active
不得修改 enabled 的人工配置，除非显式 force
不得修改 participates_in_strategy 的人工配置，除非显式 force
```

seed 命令不生成：

```text
AtomicSignalSet
AtomicSignalValue
FeatureSet
FeatureValue
```

seed 命令不调用：

```text
AtomicSignalService
FeatureLayerService
Binance
Kline
```

---

## 8. Hash 规则

新增：

```text
apps/atomic_signal/hashes.py
```

至少实现：

```text
canonical_json
compute_params_hash(params)
compute_definition_hash(definition_fields)
compute_definition_set_hash(definitions)
compute_atomic_signal_set_key(feature_set, signal_schema_version, definition_set_hash)
```

### 8.1 params_hash

建议：

```text
sha256(canonical_json(params))
```

### 8.2 definition_hash

建议包含：

```text
signal_code
algorithm_name
algorithm_version
params_hash
role
default_direction
output_type
depends_on_feature_codes
```

### 8.3 definition_set_hash

建议基于参与计算的 definitions：

```text
signal_code
algorithm_name
algorithm_version
params_hash
definition_hash
enabled
participates_in_strategy
role
default_direction
```

稳定排序后计算 SHA-256。

注意：

```text
definition_set_hash 只是集合指纹
不能替代 AtomicSignalValue 对 AtomicSignalDefinition 的逐条绑定
```

---

## 9. Calculator 架构

### 9.1 Registry

新增：

```text
apps/atomic_signal/registry.py
```

`CalculatorRegistry` 根据：

```text
algorithm_name + algorithm_version
```

找到对应 calculator。

禁止在 service 中写大量 if/else 分发。

### 9.2 Base

新增：

```text
apps/atomic_signal/calculators/base.py
```

建议结构：

```text
AtomicSignalCalculationContext
CalculatedAtomicSignal
AtomicSignalCalculationError
```

calculator 只能接收：

```text
AtomicSignalDefinition
FeatureValue map
```

calculator 不得：

```text
读数据库
写数据库
请求外部服务
读取 Kline
调用 FeatureLayerService
调用其他 AtomicSignal
生成 StrategySignal
```

### 9.3 feature_compare:v1

新增：

```text
apps/atomic_signal/calculators/feature_compare.py
```

实现：

```text
FeatureCompareV1Calculator
algorithm_name = feature_compare
algorithm_version = v1
```

params 示例：

```json
{
  "left_feature_code": "sma_4h_20",
  "operator": "gt",
  "right_feature_code": "sma_4h_60"
}
```

支持 operator：

```text
gt
gte
lt
lte
eq
```

输出：

```text
value_bool
direction
strength
confidence
evidence_items
evidence_text_zh
used_feature_codes
used_feature_value_ids
```

第一版简单规则：

```text
条件成立 → value_bool=True, strength=1
条件不成立 → value_bool=False, strength=0
计算成功 → confidence=1
计算失败 → status=failed, is_valid=False, direction=neutral, strength=0, confidence=0
```

direction：

```text
条件成立时使用 definition.default_direction
条件不成立时可使用 neutral 或 none，具体由 calculator 说明
计算失败时 direction=neutral 且 is_valid=False
```

证据示例：

```text
evidence_items = {
  "left_feature_code": "sma_4h_20",
  "left_value": "102500.25",
  "operator": "gt",
  "right_feature_code": "sma_4h_60",
  "right_value": "101800.10",
  "result": true
}

evidence_text_zh = "4h SMA20 为 102500.25，高于 4h SMA60 的 101800.10，因此该均线比较条件成立。"
```

---

## 10. Selectors

新增：

```text
apps/atomic_signal/selectors.py
```

至少包含：

```text
get_usable_feature_set(...)
get_feature_values_for_set(feature_set)
get_enabled_atomic_signal_definitions()
```

规则：

```text
get_enabled_atomic_signal_definitions 只读取数据库中 status=active 且 enabled=True 的 AtomicSignalDefinition
不得读取 default_atomic_signal_definitions.py
```

---

## 11. AtomicSignalService

新增：

```text
apps/atomic_signal/services.py
```

主入口建议：

```text
build_for_feature_set(feature_set_id=None, feature_set_key=None, trace_id=None, trigger_source="manual", dry_run=False)
```

流程：

```text
1. 接收 feature_set_id 或 feature_set_key
2. 读取 FeatureSet
3. 确认 FeatureSet.status=created 且 is_usable=True
4. 加载 FeatureValue 成 feature_code → FeatureValue 映射
5. 读取数据库中 status=active 且 enabled=True 的 AtomicSignalDefinition
6. 冻结本次 AtomicSignalDefinition 集合
7. 计算 definition_set_hash
8. 生成 atomic_signal_set_key
9. 如果 AtomicSignalSet 已存在，返回已有结果
10. 逐个调用 calculator
11. 单个失败时生成 failed AtomicSignalValue 结果，并继续计算其他 signal
12. 统计 computed_count / valid_count / invalid_count / failed_count / failure_ratio
13. 判断是否达到大规模失败阈值
14. transaction.atomic 写 AtomicSignalSet / bulk_create AtomicSignalValue
15. 单个失败写 AlertEvent
16. 大规模失败写 AlertEvent
17. 返回结构化结果
```

### 11.1 FeatureSet 校验

只允许使用：

```text
FeatureSet.status = created
FeatureSet.is_usable = True
```

如果 FeatureSet 不可用：

```text
AtomicSignalSet.status = blocked
is_usable=False
allows_downstream=False
blocked_reason / error_code 记录原因
```

### 11.2 单个失败继续

单个 AtomicSignal 失败时：

```text
AtomicSignalValue.status = failed
AtomicSignalValue.is_valid = False
AtomicSignalValue.direction = neutral
AtomicSignalValue.strength = 0
AtomicSignalValue.confidence = 0
AtomicSignalValue.error_code 非空
AtomicSignalValue.error_message 非空
AtomicSignalValue.evidence_text_zh 非空
```

继续计算其他 AtomicSignal。

### 11.3 大规模失败阻断

读取配置：

```text
ATOMIC_SIGNAL_FAILURE_BLOCK_RATIO
```

默认：

```text
0.3
```

如果：

```text
invalid_count / active_definition_count >= ATOMIC_SIGNAL_FAILURE_BLOCK_RATIO
```

则：

```text
AtomicSignalSet.status = failed
AtomicSignalSet.is_usable = False
AtomicSignalSet.allows_downstream = False
```

并写 AlertEvent。

### 11.4 正常 neutral 与失败 neutral

必须可区分：

```text
正常 neutral:
status=created
is_valid=True
direction=neutral

失败降级 neutral:
status=failed
is_valid=False
direction=neutral
```

下游必须先看 `is_valid`，再看 `direction`。

### 11.5 dry-run

`dry_run=True` 时：

```text
执行读取和计算
不写 AtomicSignalSet
不写 AtomicSignalValue
不写 AlertEvent
返回将要生成的摘要
```

---

## 12. AlertEvent

AtomicSignal 层只允许写 AlertEvent 表。

不得直接调用：

```text
Hermes webhook
notifications service
```

建议事件类型：

```text
atomic_signal_failed
atomic_signal_set_failed
atomic_signal_set_blocked
```

规则：

```text
单个 signal 失败 → warning AlertEvent
大规模失败 → error AlertEvent
FeatureSet blocked / unusable → error 或 warning AlertEvent
created 成功默认不写 AlertEvent
dry-run 不写 AlertEvent
```

AlertEvent 应包含：

```text
feature_set_id
atomic_signal_set_key
signal_code
error_code
error_message
trace_id
trigger_source
market context
```

---

## 13. Management command

新增：

```bash
python manage.py build_atomic_signals
```

支持参数：

```text
--feature-set-id
--feature-set-key
--trace-id
--trigger-source
--dry-run
```

规则：

```text
feature-set-id 与 feature-set-key 至少传一个
command 只解析参数并调用 AtomicSignalService
command 不承载核心业务逻辑
```

输出：

```text
AtomicSignalSet id
atomic_signal_set_key
status
is_usable
allows_downstream
computed_count
valid_count
invalid_count
failed_count
failure_ratio
```

dry-run 输出：

```text
would_create
would_return_existing
computed_count
valid_count
invalid_count
failed_count
```

---

## 14. 下游使用规则

虽然本阶段不实现 StrategySignal，但必须在模型和文档中明确：

StrategySignal 只能使用：

```text
AtomicSignalSet.status = created
AtomicSignalSet.is_usable = True
AtomicSignalSet.allows_downstream = True
AtomicSignalValue.is_valid = True
AtomicSignalDefinition.participates_in_strategy = True
```

观察模式信号：

```text
enabled=True
participates_in_strategy=False
```

可以用于：

```text
统计
观察
复盘
大模型辅助分析
离线评估
```

不得用于正式策略组合。

---

## 15. implementation 文档

完成后新增：

```text
docs/implementation/004B_atomic_signal.md
```

必须说明：

```text
新增 app / model
migration
AtomicSignalDefinition 生命周期
enabled / participates_in_strategy 语义
默认 definitions 数量
默认 signal_code 清单
calculator 列表
feature_compare:v1 计算逻辑
definition_set_hash 规则
atomic_signal_set_key 规则
AtomicSignalService 流程
失败处理
AlertEvent 规则
build_atomic_signals command
seed_atomic_signal_definitions command
明确未实现 StrategySignal / Risk / Trading
```

---

## 16. 测试要求

新增测试目录：

```text
tests/atomic_signal/
```

至少覆盖：

### 16.1 Model

```text
AtomicSignalDefinition 可创建
同一 signal_code 可存在不同 algorithm_version
enabled 控制是否计算
participates_in_strategy 控制是否允许下游使用
enabled=False + participates_in_strategy=True 被拒绝
status=retired / disabled 不参与 enabled definitions
AtomicSignalSet 绑定 FeatureSet
AtomicSignalValue 绑定 AtomicSignalDefinition
AtomicSignalValue 冗余 signal_code / role / algorithm_version / params_hash / definition_hash
evidence_items 必填
evidence_text_zh 必填
scale_factor 只允许 scaler 使用或非 scaler 时必须为空
```

### 16.2 Hash

```text
params_hash 稳定
definition_hash 稳定
definition_set_hash 稳定
atomic_signal_set_key 稳定
definition_set_hash 不能替代 AtomicSignalValue 逐条绑定 definition
```

### 16.3 Seed

```text
seed_atomic_signal_definitions 可运行
seed 幂等
seed 不重复插入已有定义
seed 不恢复 retired / disabled
seed 不覆盖人工 enabled / participates_in_strategy
seed 不调用 AtomicSignalService
```

### 16.4 Registry / Calculator

```text
CalculatorRegistry 能按 feature_compare:v1 找到 calculator
unsupported calculator 会失败
feature_compare 支持 gt / gte / lt / lte / eq
feature_compare 缺少 left_feature_code 时失败
feature_compare 缺少 right_feature_code 时失败
feature_compare 依赖 FeatureValue 不存在时失败
feature_compare 成功时写 evidence_items / evidence_text_zh
feature_compare 不输出交易建议
```

### 16.5 Service

```text
usable FeatureSet 可以生成 AtomicSignalSet
blocked / failed / unusable FeatureSet 不能生成 usable AtomicSignalSet
AtomicSignalService 只读取数据库 AtomicSignalDefinition
AtomicSignalService 不读取 default_atomic_signal_definitions.py
enabled=False 的 definition 不计算
enabled=True + participates_in_strategy=False 生成观察信号
相同 FeatureSet + definition_set_hash 重复执行返回已有 AtomicSignalSet
单个 AtomicSignal 失败不中断整体计算
单个失败写 AtomicSignalValue failed
单个失败写 AlertEvent
大规模失败阻断下游
大规模失败写 AlertEvent
失败 neutral 与正常 neutral 可区分
dry-run 不写 AtomicSignalSet / AtomicSignalValue / AlertEvent
transaction.atomic 写入
bulk_create 写 AtomicSignalValue
```

### 16.6 越界

```text
AtomicSignal 不读取 Kline4h / Kline1d
AtomicSignal 不请求 Binance
AtomicSignal 不重新计算 Feature
AtomicSignal 不调用 FeatureLayerService
AtomicSignal 不调用 DataQualityService
AtomicSignal 不调用 BackfillService
AtomicSignal 不生成 StrategySignal
AtomicSignal 不生成 RiskCheckResult / CandidateOrderIntent / ApprovedOrderIntent / ExecutionPreparation / Execution
AtomicSignal 不输出 weight / priority
AtomicSignal 不输出 entry / stop_loss / take_profit / position_size / leverage
```

---

## 17. 运行命令

完成后必须运行：

```bash
python manage.py makemigrations
python manage.py makemigrations --check --dry-run
python manage.py migrate --plan
python manage.py migrate
python manage.py check
python manage.py test
pytest
```

如果任何命令失败，必须修复，不得绕过，不得 fake migration。

---

## 18. 越界搜索

完成后运行：

```powershell
Select-String -Path .\apps\atomic_signal\**\*.py,.\tests\atomic_signal\**\*.py,.\config\**\*.py -Pattern "Kline4h","Kline1d","Binance","requests.get","httpx","WebSocket","websocket","FeatureLayerService","DataQualityService","BackfillService","MarketSnapshotService","StrategySignal","MarketRegime","RiskCheck","CandidateOrderIntent / ApprovedOrderIntent","Execution","create_order","place_order","position_size","leverage","entry_price","stop_loss","take_profit","should_long","should_short","weight","priority"
```

命中必须解释：

```text
负向测试中的命中可以接受
常量禁止词中的命中可以接受
业务逻辑越界调用不可接受
```

---

## 19. Codex 完成后输出要求

Codex 完成后必须输出：

```text
1. 修改 / 新增文件清单
2. 新增 app / model 名称
3. migration 文件路径
4. 默认 AtomicSignalDefinition 数量
5. 默认 signal_code 清单
6. 已实现 calculator 列表
7. AtomicSignalDefinition 生命周期规则
8. enabled / participates_in_strategy 规则
9. definition_set_hash 规则
10. atomic_signal_set_key 幂等规则
11. AtomicSignalService 流程
12. seed_atomic_signal_definitions command 说明
13. build_atomic_signals command 说明
14. AlertEvent 规则
15. 测试命令和结果
16. 越界搜索结果
17. 明确说明本阶段未实现 StrategySignal / Risk / Trading，未请求 Binance，未读取 Kline，未重新计算 Feature，未生成交易建议。
```

---

## 20. 验收标准

004B 完成后必须满足：

```text
apps.atomic_signal 已注册
AtomicSignalDefinition / AtomicSignalSet / AtomicSignalValue 已建模
默认 AtomicSignalDefinition 可 seed 入库
AtomicSignalService 正式计算只读数据库 AtomicSignalDefinition
enabled 控制是否计算
participates_in_strategy 控制是否允许 StrategySignal 使用
enabled=False + participates_in_strategy=True 被拒绝
feature_compare:v1 可用
至少 sma_4h_20_above_sma_4h_60 可生成 AtomicSignalValue
AtomicSignalSet 与 FeatureSet 对齐
AtomicSignalValue 可追溯到 FeatureValue / FeatureDefinition / FeatureSet / MarketSnapshot
单个失败不阻断整体
大规模失败阻断下游
evidence_items 和 evidence_text_zh 必填
AtomicSignal 不输出 weight / priority
AtomicSignal 不输出交易字段
AtomicSignal 不读取 Kline
AtomicSignal 不请求 Binance
AtomicSignal 不重新计算 Feature
AtomicSignal 不生成 StrategySignal / Risk / Trading
测试通过
