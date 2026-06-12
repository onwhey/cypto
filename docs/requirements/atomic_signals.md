# AtomicSignal 原子信号需求说明

## 1. 模块定位

AtomicSignal 是当前项目中的原子市场判断单元，位于 FeatureLayer 之后、StrategySignal 之前。

它负责：

```text
基于 usable FeatureSet / FeatureValue
读取数据库中 enabled=True 的 AtomicSignalDefinition
执行单一、原子化的市场条件判断
生成 AtomicSignalSet / AtomicSignalValue
为后续 StrategySignal 提供可追溯、可审计、可复盘的中间信号
```

AtomicSignal 不是机器学习模型，不涉及模型训练、模型推理或模型部署。

当前项目中不得使用以下命名作为模块名、模型名、类名或表名：

```text
WeakModel
ModelResult
ModelSignal
```

避免误导为机器学习模型。

AtomicSignal 不是策略模块，不生成最终交易决策，不生成订单意图，不执行交易。

正确链路：

```text
MarketSnapshot
→ FeatureSet / FeatureValue
→ AtomicSignalSet / AtomicSignalValue
→ StrategySignal
→ StrategySignalQuality
→ DecisionSnapshot
→ Binance Account Sync / PriceSnapshot
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation
→ Execution
```

---

## 2. 模块目标

AtomicSignal 要解决的问题：

```text
把基础特征转换成单点条件判断
统一原子信号输入输出契约
让策略层组合原子信号，而不是重复写条件判断
让每个原子信号都可单独统计、复盘、回测
让新信号可以先观察、不参与真实策略
让信号失败可以局部降级，不因单个失败中断全流程
让大规模失败可以阻断下游，避免污染策略判断
```

AtomicSignal 不负责：

```text
信号聚合
权重分配
冲突解决
策略级判断
风控否决
最终仓位计算
订单生成
交易执行
```

这些必须上移到 StrategySignal 或更上层策略组合模块。

---

## 3. 核心原则

### 3.1 只读取 FeatureSet / FeatureValue

AtomicSignal 只能读取：

```text
FeatureSet
FeatureValue
AtomicSignalDefinition
```

禁止读取：

```text
Kline4h / Kline1d
Binance REST
WebSocket
MarketSnapshotService
FeatureLayerService
DataQualityService
BackfillService
```

AtomicSignal 不得重新计算 Feature，也不得重复计算指标。

### 3.2 原子信号之间平权

AtomicSignal 之间禁止互相调用。

每个 AtomicSignal 只做一个单一、原子化的市场事件判断。

禁止：

```text
AtomicSignal A 依赖 AtomicSignal B 的输出
AtomicSignal 内部聚合多个 AtomicSignal
AtomicSignal 内部分配权重
AtomicSignal 内部做策略路由
```

信号聚合、冲突解决、权重分配必须由 StrategySignal 或更上层策略组合模块完成。

### 3.3 统一输入输出契约

所有 AtomicSignalValue 必须遵守统一输出字段，不得每个信号自定义不兼容结构。

核心输出必须包括：

```text
signal_code
algorithm_version
role
direction
strength
confidence
evidence_items
evidence_text_zh
status
is_valid
used_feature_codes
used_feature_value_ids
```

### 3.4 不输出固定权重

AtomicSignal 禁止输出：

```text
weight
priority
fixed_weight
strategy_weight
final_score
strategy_score
```

权重由后续 StrategySignal 或更上层策略组合模块根据市场环境、历史表现、策略上下文动态分配。

AtomicSignal 的 role 只帮助上层理解用途，不代表权重、优先级或交易动作。

### 3.5 版本必须可追溯

AtomicSignalDefinition 必须包含：

```text
algorithm_name
algorithm_version
params
params_hash
definition_hash
```

算法逻辑升级必须新增 algorithm_version 和对应 calculator；已发布版本的计算行为不得被原地修改。

参数变化应新增 AtomicSignalDefinition，不得原地修改已用于 production 的定义。

### 3.6 失败不能伪装成正常 neutral

单个 AtomicSignal 失败时，可以将方向降级为：

```text
direction = neutral
strength = 0
confidence = 0
```

但必须同时：

```text
status = failed
is_valid = False
error_code 非空
error_message 非空
evidence_text_zh 说明失败原因
```

下游必须先看 `is_valid`，再看 `direction / strength / confidence`。

---

## 4. 核心模型

### 4.1 AtomicSignalDefinition

AtomicSignalDefinition 是长期存在的原子信号定义。

它回答：

```text
系统里有哪些原子信号？
这个原子信号怎么判断？
依赖哪些 FeatureValue？
算法版本是什么？
参数是什么？
是否启用？
是否允许参与正式策略？
```

建议字段语义：

```text
id
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

#### signal_code

原子信号业务编码，稳定、可读、唯一。

示例：

```text
close_4h_above_sma_4h_20
close_4h_above_sma_4h_60
sma_4h_20_above_sma_4h_60
close_4h_breaks_range_4h_60_high
close_4h_below_range_4h_60_low
volume_4h_above_volume_sma_4h_20
close_1d_above_sma_1d_20
```

禁止使用具有交易动作含义的编码：

```text
should_long
should_short
buy_signal
sell_signal
entry_signal
stop_loss_signal
take_profit_signal
```

#### role

role 表示原子信号在后续策略组合中的用途。

第一版允许角色：

```text
voter
descriptor
gatekeeper
scaler
filter
```

含义：

```text
voter      = 可参与方向倾向投票
descriptor = 只描述市场状态
gatekeeper = 可被上层策略用作必要条件
scaler     = 可给上层提供缩放参考
filter     = 可被上层策略用作过滤条件
```

约束：

```text
role 不代表权重
role 不代表优先级
role 不改变 AtomicSignal 的平权地位
role 不代表交易动作
```

`gatekeeper` 只能表示“可被上层用作门控条件”。AtomicSignal 本身不能一票否决交易，是否采纳门控条件由 StrategySignal 决定；交易前最终阻断由后续 RiskCheck 对 CandidateOrderIntent 审批。

`scaler` 可以输出 `scale_factor`，但 `scale_factor` 只是上层 StrategySignal 的参考，不是最终仓位，不是 leverage，不是订单。RiskCheck 不得直接消费 AtomicSignalValue 的 scale_factor 作为订单调整依据。

#### default_direction

默认方向倾向。

建议取值：

```text
bullish
bearish
neutral
none
```

含义：

```text
bullish = 条件成立时通常偏多
bearish = 条件成立时通常偏空
neutral = 条件成立时表达中性状态
none    = 无方向语义
```

direction 只是方向倾向，不是交易指令。

#### status

生命周期状态：

```text
draft
active
deprecated
retired
disabled
```

production AtomicSignalSet 默认只使用：

```text
status = active
```

#### enabled

控制是否计算该原子信号。

规则：

```text
enabled=True  → AtomicSignalService 会计算并入库 AtomicSignalValue
enabled=False → 不计算，不生成新的 AtomicSignalValue
```

#### participates_in_strategy

控制是否允许后续 StrategySignal 使用该原子信号。

规则：

```text
enabled=True + participates_in_strategy=False
→ 观察模式
→ 计算并入库
→ 可用于统计、复盘、大模型辅助分析
→ 不得进入正式 StrategySignal

enabled=True + participates_in_strategy=True
→ production 模式
→ 计算并入库
→ 允许后续 StrategySignal 使用

enabled=False + participates_in_strategy=False
→ 不计算，不参与策略
```

禁止组合：

```text
enabled=False + participates_in_strategy=True
```

该组合语义冲突，应通过 model clean 或 service 校验拒绝。

下游 StrategySignal 只能读取：

```text
AtomicSignalDefinition.status = active
AtomicSignalDefinition.enabled = True
AtomicSignalDefinition.participates_in_strategy = True
AtomicSignalValue.is_valid = True
AtomicSignalSet.is_usable = True
AtomicSignalSet.allows_downstream = True
```

#### params

算法参数，使用 JSON 保存。

示例：

```json
{
  "left_feature_code": "sma_4h_20",
  "operator": "gt",
  "right_feature_code": "sma_4h_60"
}
```

params 是 AtomicSignalDefinition 完整身份的一部分。

---

### 4.2 AtomicSignalSet

AtomicSignalSet 是某个 FeatureSet 上生成的一批原子信号结果。

它回答：

```text
这个 FeatureSet 上，使用哪一批 AtomicSignalDefinition，生成了哪一批 AtomicSignalValue？
```

建议字段语义：

```text
id
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

#### 粒度

AtomicSignalSet 与 FeatureSet 对齐。

规则：

```text
一个 usable FeatureSet
+ 一个 signal_schema_version
+ 一个 AtomicSignalDefinition 集合
→ 最多一个 production AtomicSignalSet
```

#### 幂等

必须生成稳定 `atomic_signal_set_key`。

建议基于：

```text
feature_set_id
feature_set_key
signal_schema_version
definition_set_hash
```

生成 SHA-256。

重复执行相同输入时：

```text
返回已有 AtomicSignalSet
不得重复写 AtomicSignalValue
```

#### definition_set_hash

本次 AtomicSignalSet 使用的 AtomicSignalDefinition 集合指纹。

建议基于所有参与计算的 definitions：

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

### 4.3 AtomicSignalValue

AtomicSignalValue 是某个 AtomicSignalSet 内某一个 AtomicSignalDefinition 的计算结果。

建议字段语义：

```text
id
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

#### direction

建议取值：

```text
bullish
bearish
neutral
none
```

direction 表示方向倾向，不是交易指令。

禁止将 direction 解释为：

```text
开多
开空
买入
卖出
下单
```

#### strength

信号强度，范围：

```text
0 <= strength <= 1
```

第一版可以简单实现：

```text
条件成立 → strength = 1
条件不成立 → strength = 0
计算失败 → strength = 0
```

后续可扩展归一化强度。

#### confidence

信号可信度，范围：

```text
0 <= confidence <= 1
```

第一版可以简单实现：

```text
输入特征完整且计算成功 → confidence = 1
计算失败或输入缺失 → confidence = 0
```

后续可根据特征质量、历史稳定性、延迟等扩展。

#### evidence_items

结构化证据，JSONField，必须保存。

用于系统、回测、复盘程序读取。

示例：

```json
{
  "left_feature_code": "sma_4h_20",
  "left_value": "102500.25",
  "operator": "gt",
  "right_feature_code": "sma_4h_60",
  "right_value": "101800.10",
  "result": true
}
```

#### evidence_text_zh

中文证据说明，TextField，必须保存。

用于：

```text
用户审阅
复盘
告警摘要
大模型辅助分析
```

示例：

```text
4h SMA20 为 102500.25，高于 4h SMA60 的 101800.10，因此该均线比较条件成立。
```

失败时也必须写中文说明：

```text
该原子信号计算失败：缺少 sma_4h_60 特征值，因此无法判断 4h SMA20 是否高于 SMA60。
```

evidence_text_zh 禁止包含交易建议：

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

#### used_feature_codes / used_feature_value_ids

必须记录本次原子信号使用了哪些 FeatureValue。

用于追溯：

```text
AtomicSignalValue
→ FeatureValue
→ FeatureDefinition
→ FeatureSet
→ MarketSnapshot
```

#### role_specific_payload

role_specific_payload 用于保存特定 role 的扩展内容。

约束：

```text
不得保存 weight / priority
不得保存 entry / stop_loss / take_profit
不得保存 position_size / leverage
不得保存 candidate_order_intent / approved_order_intent / order payload
不得表达买入、卖出、开多、开空
```

#### scale_factor

scale_factor 只允许 role=scaler 的 AtomicSignalValue 使用。

约束：

```text
scale_factor 只是给上层 StrategySignal 的缩放参考
scale_factor 不是 position_size
scale_factor 不是 leverage
scale_factor 不是最终仓位
AtomicSignal 不决定是否采用 scale_factor
```

---

## 5. 默认定义与运行时来源

AtomicSignal 应与 FeatureLayer 一样，区分：

```text
default_atomic_signal_definitions.py = 默认原子信号模板
AtomicSignalDefinition 表 = 正式运行时原子信号字典
```

流程：

```text
default_atomic_signal_definitions.py
→ seed_atomic_signal_definitions
→ AtomicSignalDefinition 表
→ AtomicSignalService 读取数据库中 status=active 且 enabled=True 的定义
→ AtomicSignalSet / AtomicSignalValue
```

正式计算时：

```text
AtomicSignalService 只读取数据库 AtomicSignalDefinition 表
不得直接读取 default_atomic_signal_definitions.py
不得与 default_atomic_signal_definitions.py 求合集
```

`default_atomic_signal_definitions.py` 只负责默认 seed 模板。

---

## 6. seed 命令

必须提供命令：

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
已存在时只能更新 display_name / description 等非身份字段
不得覆盖 algorithm_name / algorithm_version / params / params_hash / definition_hash
不得把 retired / disabled 的 AtomicSignalDefinition 自动恢复为 active
不得修改 participates_in_strategy 的人工配置，除非明确指定强制参数
不得修改 enabled 的人工配置，除非明确指定强制参数
```

seed 命令不生成 AtomicSignalSet，不生成 AtomicSignalValue。

---

## 7. Calculator 架构

AtomicSignal 必须有独立 calculator registry。

```text
algorithm_name + algorithm_version → calculator class
```

候选 calculator：

```text
feature_compare:v1
threshold_check:v1
range_breakout_check:v1
feature_ratio_check:v1
```

004B 第一版至少应实现：

```text
feature_compare:v1
```

其他 calculator 可按 plan 收敛，不要求第一版全部实现。

calculator 只负责纯判断逻辑：

```text
不读数据库
不写数据库
不请求 Binance
不读取 Kline
不调用 FeatureLayerService
不调用其他 AtomicSignal
不生成 StrategySignal
不调用 StrategySignal / DecisionSnapshot / OrderPlan / RiskCheck / ExecutionPreparation / Execution
```

### 7.1 中文算法说明文档要求

AtomicSignal implementation 必须维护：

```text
docs/implementation/atomic_signal/
```

每个默认 AtomicSignal calculator / signal rule 都必须有对应中文说明文档。

中文说明至少包括：

```text
基本信息
中文名
用途
输入 FeatureValue
依赖的 feature_code
计算逻辑
输出字段
关键失败条件
交易语义边界
```

规则：

```text
中文说明文档只解释原子信号算法和边界。
不得写交易建议。
不得写开仓、平仓、止损、止盈、仓位、杠杆或下单建议。
不得把 AtomicSignal 解释成 StrategySignal、DecisionSnapshot 或订单意图。
```

如果新增默认 AtomicSignalDefinition 或新增 calculator，必须同步新增或更新对应中文说明文档。

---

## 8. 第一版建议原子信号

第一版不要贪多，只实现少量框架验证信号。

基于当前 FeatureLayer 已有特征，可考虑：

```text
close_4h_above_sma_4h_20
close_4h_above_sma_4h_60
sma_4h_20_above_sma_4h_60
close_4h_breaks_range_4h_60_high
close_4h_below_range_4h_60_low
volume_4h_above_volume_sma_4h_20
close_1d_above_sma_1d_20
```

这些都只是条件判断，不是策略信号。

---

## 9. 失败处理

### 9.1 单个 AtomicSignalValue 失败

单个 AtomicSignal 失败时，不得中断整个 AtomicSignalSet 计算。

必须写入：

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

注意：

```text
这是失败降级 neutral，不是正常 neutral。
下游必须先看 is_valid，再看 direction。
```

并写 AlertEvent：

```text
alert_type = atomic_signal_failed
severity = warning
```

然后继续计算其他 AtomicSignal。

### 9.2 大规模失败

如果失败数量或失败比例超过配置阈值，则 AtomicSignalSet 必须阻断下游。

建议默认配置：

```text
ATOMIC_SIGNAL_FAILURE_BLOCK_RATIO = 0.3
```

该值必须可配置，不得在业务代码中写死。

规则：

```text
invalid_count / active_definition_count >= ATOMIC_SIGNAL_FAILURE_BLOCK_RATIO
→ AtomicSignalSet.status = failed
→ AtomicSignalSet.is_usable = False
→ AtomicSignalSet.allows_downstream = False
→ 写 AlertEvent
→ 下游 StrategySignal 不得使用
```

如果后续引入 required 信号，也可支持：

```text
required_failed_count > 0
→ AtomicSignalSet failed
```

### 9.3 AlertEvent

AtomicSignal 层只允许写 AlertEvent，不得直接调用 Hermes webhook，不得直接调用 notifications service。

建议事件：

```text
atomic_signal_failed
atomic_signal_set_failed
```

created / success 默认不写 AlertEvent，避免通知噪音。

---

## 10. AtomicSignalService 流程

主流程：

```text
1. 接收 feature_set_id 或 feature_set_key
2. 读取 FeatureSet
3. 确认 FeatureSet.status=created 且 is_usable=True
4. 加载 FeatureValue 成 feature_code → value 映射
5. 读取数据库中 status=active 且 enabled=True 的 AtomicSignalDefinition
6. 计算 definition_set_hash
7. 生成 atomic_signal_set_key
8. 如果 AtomicSignalSet 已存在，返回已有结果
9. 冻结本次 AtomicSignalDefinition 集合
10. 逐个调用 calculator
11. 单个失败时写 failed AtomicSignalValue，并继续
12. 统计 valid_count / invalid_count / failure_ratio
13. 判断是否超过失败阈值
14. transaction.atomic 写 AtomicSignalSet / bulk_create AtomicSignalValue
15. 对单个失败和大规模失败写 AlertEvent
16. 返回结构化结果
```

---

## 11. 下游使用规则

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

## 12. 独立评估

每个 AtomicSignal 应能独立回测和统计。

至少应支持后续统计：

```text
命中率
稳定性
有效性
与策略收益的关系
与其他原子信号的相关性
观察模式表现
```

004B 第一版不要求实现复杂评估系统，但模型和字段必须支持后续独立评估。

---

## 13. 特征与周期绑定

AtomicSignal 依赖的 Feature 必须明确周期。

规则：

```text
sma_4h_20 和 sma_1d_20 是两个不同特征
不同周期不得混用
AtomicSignalDefinition.depends_on_feature_codes 必须列明具体 feature_code
不得模糊依赖“某个 SMA”
```

特征算法版本由 FeatureDefinition 元数据管理，不依赖文件组织。

AtomicSignal 必须通过 used_feature_codes / used_feature_value_ids 记录实际使用的特征结果。

---

## 14. 明确禁止

AtomicSignal 第一版不做：

```text
StrategySignal
MarketRegime
DecisionSnapshot
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
账户 / 仓位 / 订单
自动交易
权重分配
冲突解决
信号聚合
最终仓位缩放
特征重新计算
Kline 读取
Binance 请求
WebSocket
大模型分析
```

AtomicSignal 禁止输出：

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
candidate_order_intent / approved_order_intent
```

---

## 15. 测试要求

至少覆盖：

```text
AtomicSignalDefinition 可创建
同一 signal_code 可存在不同 algorithm_version
enabled 控制是否计算
participates_in_strategy 控制是否允许下游使用
enabled=False + participates_in_strategy=True 被拒绝
enabled=True + participates_in_strategy=False 会生成观察信号
AtomicSignalService 只读取数据库 AtomicSignalDefinition，不读取 default 文件
seed_atomic_signal_definitions 幂等
seed 不重复插入已有定义
seed 不恢复 retired / disabled
seed 不覆盖人工 enabled / participates_in_strategy
AtomicSignalSet 绑定 FeatureSet
AtomicSignalValue 绑定 AtomicSignalDefinition
definition_set_hash 稳定
atomic_signal_set_key 幂等
CalculatorRegistry 可按 algorithm_name + algorithm_version 找 calculator
AtomicSignal 不读取 Kline
AtomicSignal 不重新计算 Feature
AtomicSignal 之间不互相调用
单个信号失败不中断整体计算
单个信号失败写 AlertEvent
大规模失败阻断下游
大规模失败写 AlertEvent
失败信号 direction=neutral 但 is_valid=False
正常 neutral 与失败 neutral 可区分
evidence_items 必填
evidence_text_zh 必填
evidence_text_zh 不包含交易建议
观察模式信号不参与 StrategySignal
role 不代表 weight / priority
scaler 的 scale_factor 不等于 position_size / leverage
不输出 weight / priority
不生成交易建议
```

---

## 16. 验收标准

AtomicSignal 需求完成后，后续 plan / 实现必须满足：

```text
AtomicSignalDefinition 管理原子信号定义、算法版本、参数、开关和观察模式
AtomicSignalSet 绑定 FeatureSet
AtomicSignalValue 绑定 AtomicSignalDefinition
AtomicSignalService 只读取数据库 AtomicSignalDefinition
default_atomic_signal_definitions.py 只用于 seed
enabled 控制是否计算
participates_in_strategy 控制是否允许策略使用
enabled=False + participates_in_strategy=True 被拒绝
单个失败可降级继续
大规模失败阻断下游
evidence_items 和 evidence_text_zh 都必须保留
role 可包含 voter / descriptor / gatekeeper / scaler / filter
role 不代表权重或交易动作
scale_factor 只是上层参考，不是仓位或杠杆
AtomicSignal 不输出 weight / priority
AtomicSignal 不生成交易决策
AtomicSignal 不读取 Kline
AtomicSignal 不重新计算 Feature
AtomicSignal 不执行交易
```
