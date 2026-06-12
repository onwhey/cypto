# FeatureLayer 需求说明

## 1. 模块定位

FeatureLayer 是 MarketSnapshot 之后、AtomicSignal / StrategySignal 之前的基础特征层。

它负责：

```text
基于 usable MarketSnapshot
回查该 snapshot 对应的 4h / 1d K 线窗口
使用当前 active FeatureDefinition 集合
计算基础市场特征
生成 FeatureSet 和 FeatureValue
为后续 AtomicSignal 提供稳定、可追溯、可复用的输入，并通过追溯链路支持 StrategySignal、DecisionSnapshot、OrderPlan、RiskCheck 和复盘
```

FeatureLayer 不是策略模块，不生成交易信号，不判断多空，不做风控，不生成订单意图，不调用交易所，不调用大模型。

正确链路：

```text
Kline4h / Kline1d
→ DataQualityResult
→ MarketSnapshot
→ FeatureLayer
→ AtomicSignal
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

## 2. 解决的问题

如果后续每个原子信号、策略、复盘模块都自行计算均线、ATR、收益率、区间高低点等基础特征，会产生以下问题：

```text
不同模块对同一特征的算法实现不一致
回测和实盘使用的特征定义不一致
特征算法升级后污染历史结果
无法追溯某个信号当时使用的是哪个特征版本
重复计算，增加系统复杂度
策略层混入数据处理逻辑
```

FeatureLayer 要解决的是：

```text
统一特征定义
统一算法版本
统一参数管理
统一计算结果存储
统一追溯链路
```

---

## 3. 核心原则

### 3.1 基于 MarketSnapshot

FeatureLayer 的正式输入必须是 `MarketSnapshot`。

不得绕过 MarketSnapshot 直接基于“最新 K 线”生成特征。

只允许基于：

```text
MarketSnapshot.status = created
MarketSnapshot.is_usable = True
```

生成 production FeatureSet。

如果 MarketSnapshot 是：

```text
blocked
failed
is_usable=False
```

FeatureLayer 必须拒绝生成 usable FeatureSet。

### 3.2 FeatureSet 与 MarketSnapshot 对齐

FeatureSet 不是每日粒度记录。

正常生产流程中：

```text
一个 usable MarketSnapshot
→ 一个 production FeatureSet
```

由于 MarketSnapshot 通常跟随 4h 收盘周期生成，因此一天内可能生成多个 FeatureSet。

FeatureSet 的粒度应由以下信息决定：

```text
market_snapshot_id
snapshot_key
feature_schema_version
definition_set_hash
```

### 3.3 特征定义与特征值分离

FeatureLayer 必须区分：

```text
FeatureDefinition = 特征定义
FeatureSet        = 某个 MarketSnapshot 上的一批特征计算结果
FeatureValue      = 某个 FeatureSet 内某一个特征的具体值
Calculator        = 算法代码
```

不得只用一个 JSON payload 作为全部特征管理方式。

### 3.4 算法版本不可变

一旦某个 `algorithm_name + algorithm_version` 已经用于正式 FeatureValue，该算法版本的历史行为不得修改。

如果算法逻辑升级，必须新增 algorithm_version 和对应 calculator 实现。

例如：

```text
AtrWilderV1Calculator 必须保留
AtrWilderV2Calculator 必须新增
```

不得直接修改 v1 算法使历史结果不可复现。

### 3.5 参数属于 FeatureDefinition

参数变化不一定是算法版本变化。

应区分：

```text
algorithm_name + algorithm_version = 用哪段算法代码
params                           = 这段代码的参数
FeatureDefinition                = 特征完整身份
```

例如：

```text
sma_4h_20
sma_4h_60
sma_1d_200
```

可以共用：

```text
algorithm_name = simple_moving_average
algorithm_version = v1
```

但 params 不同：

```json
{"timeframe": "4h", "source": "close", "window": 20}
{"timeframe": "4h", "source": "close", "window": 60}
{"timeframe": "1d", "source": "close", "window": 200}
```

如果参数变化导致业务语义变化，应新增 FeatureDefinition，不得原地修改已经用于生产的定义。

### 3.6 FeatureDefinition 不物理删除

后台所谓“删除无效特征”不得物理删除 FeatureDefinition。

只能通过状态表达生命周期：

```text
draft
active
deprecated
retired
disabled
```

历史 FeatureValue 必须永远可以追溯到当时使用的 FeatureDefinition。

### 3.7 不产生策略语义

FeatureLayer 只能计算中性基础数值特征。

允许：

```text
latest_close_4h
sma_4h_20
sma_4h_60
atr_4h_14
range_1d_365_high
close_4h_vs_sma_4h_200_pct
```

比较型 bool 条件不属于 FeatureLayer。`*_gt_*`、`feature_compare`、comparison calculator 属于后续 AtomicSignal 阶段，不在 004A FeatureLayer 中注册或默认启用。

004A FeatureLayer 不注册 `feature_compare:v1`，默认特征不包含 `sma_4h_20_gt_sma_4h_60`。

禁止：

```text
bullish
bearish
long
short
entry
stop_loss
take_profit
position_size
leverage
should_trade
risk_signal
trend_signal
```

FeatureLayer 不得把条件判断、方向判断、风控判断或交易执行判断伪装成基础特征。

---

## 4. 模型需求

### 4.1 FeatureDefinition

FeatureDefinition 是长期存在的特征定义。

它回答：

```text
系统里有哪些特征？
这个特征怎么算？
用哪个算法版本？
参数是什么？
当前是否启用？
这个特征结果是什么类型？
```

建议字段语义：

```text
id
feature_code
display_name
description
category
timeframe
value_type
algorithm_name
algorithm_version
params
params_hash
definition_hash
status
is_required
warmup_bars
output_unit
depends_on
created_at_utc
updated_at_utc
```

#### feature_code

特征业务代码，必须稳定、可读、唯一。

示例：

```text
sma_4h_20
sma_4h_60
sma_4h_200
atr_4h_14
return_4h_6
range_1d_365_high
close_4h_vs_sma_4h_200_pct
```

`feature_code` 应尽量表达关键参数，避免隐藏重要语义。

#### algorithm_name

算法族名称。

示例：

```text
simple_moving_average
exponential_moving_average
atr_wilder
rolling_high
rolling_low
return_pct
volume_sma
feature_ratio
```

#### algorithm_version

算法实现版本。

示例：

```text
v1
v2
wilder_v1
wilder_v2
```

同一 algorithm_name 可以有多个 algorithm_version。

算法逻辑升级必须新增版本，不得覆盖旧版本。

#### params

算法参数，使用 JSON 保存。

示例：

```json
{
  "timeframe": "4h",
  "source": "close",
  "window": 20
}
```

params 是 FeatureDefinition 完整身份的一部分。

#### params_hash

对规范化后的 params 计算 hash，用于追溯和防止参数漂移。

建议：

```text
sha256(canonical_json(params))
```

#### definition_hash

对完整定义计算 hash。

建议包含：

```text
feature_code
algorithm_name
algorithm_version
params_hash
value_type
timeframe
```

作用：

```text
证明某个 FeatureDefinition 的完整版本身份
```

#### status

生命周期状态：

```text
draft       = 定义中，不参与正式计算
active      = 当前启用，参与 production FeatureSet
deprecated  = 不推荐新用，但保留历史兼容
retired     = 不再参与新计算，只保留历史追溯
disabled    = 临时禁用
```

production FeatureSet 默认只使用 `active` FeatureDefinition。

#### is_required

是否为必需特征。

第一版可以将 active 特征全部视为 required，但字段应保留。

建议规则：

```text
required 特征失败 → FeatureSet failed / is_usable=False
optional 特征失败 → FeatureValue.is_valid=False，可继续
```

如第一版不区分 optional，则任一 active 特征失败即 FeatureSet failed。

#### warmup_bars

计算该特征需要的最少 K 线根数。

示例：

```text
sma_200 → 200
atr_14 → 15 或更多
range_365_high → 365
```

FeatureLayer 必须在计算前确认 MarketSnapshot 提供的窗口足够。

#### depends_on

依赖其他特征代码。

例如：

```text
close_4h_vs_sma_4h_200_pct
```

可以依赖：

```text
latest_close_4h
sma_4h_200
```

第一版可以不实现复杂依赖调度，但字段可以保留，供后续复合特征使用。

---

### 4.2 FeatureSet

FeatureSet 代表某个 MarketSnapshot 上的一批特征计算结果。

它回答：

```text
这个 MarketSnapshot 上，使用哪一批 FeatureDefinition，生成了哪一批 FeatureValue？
```

建议字段语义：

```text
id
feature_set_key
market_snapshot
snapshot_key
feature_schema_version
definition_set_hash
status
is_usable
active_definition_count
computed_count
valid_count
invalid_count
required_failed_count
optional_failed_count
payload_summary
error_code
error_message
trace_id
trigger_source
created_at_utc
updated_at_utc
```

#### feature_set_key

FeatureSet 的业务唯一键。

建议基于：

```text
market_snapshot_id
snapshot_key
feature_schema_version
definition_set_hash
```

生成 SHA-256。

规则：

```text
同一 MarketSnapshot + 同一 feature_schema_version + 同一 definition_set_hash
→ 只能生成一个 production FeatureSet
```

重复执行应返回已有 FeatureSet，不得无限插入。

#### definition_set_hash

本次 FeatureSet 使用的 FeatureDefinition 集合指纹。

建议基于所有参与计算的 active FeatureDefinition 的：

```text
feature_code
algorithm_name
algorithm_version
params_hash
definition_hash
```

按稳定顺序拼接后 hash。

作用：

```text
冻结本次计算使用的是哪一批特征定义
```

注意：

```text
definition_set_hash 只是集合指纹
不能替代 FeatureValue 对 FeatureDefinition 的逐条绑定
```

#### status

建议支持：

```text
created
blocked
failed
```

含义：

```text
created = 计算完成且可用
blocked = 前置条件不满足，例如 MarketSnapshot 不可用
failed  = 计算异常或 required 特征失败
```

#### is_usable

规则：

```text
created → True
blocked / failed → False
```

---

### 4.3 FeatureValue

FeatureValue 是某个 FeatureSet 内某一个 FeatureDefinition 的计算结果。

它回答：

```text
某个 MarketSnapshot 对应的 FeatureSet 里，某个特征的实际值是多少？
它使用哪个 FeatureDefinition？
使用哪个算法版本和参数？
是否有效？
```

建议字段语义：

```text
id
feature_set
feature_definition
feature_code
algorithm_name
algorithm_version
params_hash
definition_hash
value_type
value_decimal
value_integer
value_bool
value_text
value_json
is_valid
error_code
error_message
created_at_utc
updated_at_utc
```

#### 绑定要求

每条 FeatureValue 必须绑定具体 FeatureDefinition。

不得只靠 `definition_set_hash` 追溯版本。

必须冗余保存：

```text
feature_code
algorithm_name
algorithm_version
params_hash
definition_hash
```

原因：

```text
即使 FeatureDefinition 后续状态或描述变化，历史 FeatureValue 仍然易查和可读。
```

#### value 字段

根据 value_type 使用对应字段：

```text
decimal → value_decimal
integer → value_integer
bool    → value_bool
text    → value_text
json    → value_json
```

004A 默认 FeatureDefinition 只使用基础数值结果，不使用比较型 bool FeatureValue。

价格、均线、ATR、收益率等数值不得用 float 保存。

建议：

```text
数据库使用 DecimalField
JSON payload 中 Decimal 转 string
```

---

## 5. 代码架构

建议新增 app：

```text
apps/feature_layer/
```

建议结构：

```text
apps/feature_layer/
  __init__.py
  apps.py
  constants.py
  models.py
  default_definitions.py
  registry.py
  selectors.py
  services.py
  calculators/
    __init__.py
    latest.py
    moving_average.py
    volatility.py
    returns.py
    ranges.py
    volume.py
  management/
    commands/
      seed_feature_definitions.py
      build_features.py
  migrations/
```

### 5.1 calculators

算法代码按算法族组织，不按特征数量组织。

不要：

```text
一个特征一个文件
一个巨大 calculators.py
```

推荐：

```text
latest.py
moving_average.py
volatility.py
returns.py
ranges.py
volume.py
```

示例：

```text
SimpleMovingAverageV1Calculator
AtrWilderV1Calculator
AtrWilderV2Calculator
RollingHighV1Calculator
ReturnPctV1Calculator
VolumeSmaV1Calculator
```

004A FeatureLayer 不包含 comparison calculator。比较型 bool 条件应在后续 AtomicSignal 阶段重新定义，不应放入 `apps/feature_layer/calculators/`。

### 5.2 registry

CalculatorRegistry 根据：

```text
algorithm_name + algorithm_version
```

找到对应 calculator。

例如：

```text
("simple_moving_average", "v1") → SimpleMovingAverageV1Calculator
("atr_wilder", "v1") → AtrWilderV1Calculator
("atr_wilder", "v2") → AtrWilderV2Calculator
```

禁止在 service 中硬编码大量 if/else 分发。

### 5.3 selectors

selectors 负责：

```text
读取 usable MarketSnapshot
根据 MarketSnapshot 窗口字段回查 Kline4h / Kline1d
读取 active FeatureDefinition
```

selectors 只读数据库，不写入数据。

### 5.4 services

FeatureLayerService 负责主流程：

```text
接收 market_snapshot_id 或 snapshot_key
接收 trigger_source / trace_id
确认 MarketSnapshot usable
读取 4h / 1d K 线窗口
读取 active FeatureDefinition
冻结本次 active FeatureDefinition 集合
计算 definition_set_hash
检查是否已有 FeatureSet
调用 calculators 计算 FeatureValue
写 FeatureSet / FeatureValue
返回结构化结果
```

services 不请求 Binance，不修改 Kline，不生成信号。

FeatureLayerService 不直接读取 `default_definitions.py`，不调用 MarketSnapshotService，不修改 MarketSnapshot。

计算过程中不得再次重新读取 active FeatureDefinition，避免同一个 FeatureSet 的定义集合发生漂移。

### 5.5 build_features command

`build_features` 是 FeatureLayer 手动生成入口。

必须支持：

```text
--market-snapshot-id
--snapshot-key
--trigger-source
--trace-id
```

command 只解析参数并调用 FeatureLayerService，不承载核心计算逻辑。

command 输出至少包括：

```text
FeatureSet id
feature_set_key
status
computed_count
valid_count
invalid_count
```

### 5.6 calculator 中文计算逻辑文档

FeatureLayer implementation 必须维护：

```text
docs/implementation/feature_layer/
```

每个默认 calculator 文档应说明：

```text
基本信息
中文名
中文解释
计算步骤
计算公式
关键失败条件
交易语义边界
```

当前 004A calculator 文档包括：

```text
base.md
latest.md
moving_average.md
volatility.md
ranges.md
returns.md
volume.md
```

`comparisons.md` 不属于 004A FeatureLayer。

`volatility.md` 必须明确当前 `atr_wilder:v1` 是 True Range 简单平均，不是完整递推 Wilder smoothing。

如果未来实现真正 Wilder 递推，必须新增 algorithm_version，例如 `v2`，不得修改 `v1` 历史行为。

---

## 6. 默认特征定义

### 6.1 default_definitions.py 与 FeatureDefinition 表

`apps/feature_layer/default_definitions.py` 是默认特征模板 / 默认特征清单。

`default_definitions.py` 不是数据库表。

`default_definitions.py` 不是正式运行时特征字典。

`FeatureDefinition` 表才是正式运行时特征字典。

FeatureLayerService 正式计算时只读取数据库中：

```text
status = active
```

的 FeatureDefinition。

FeatureLayerService 不得直接读取 `default_definitions.py`。

`default_definitions.py` 与 `FeatureDefinition` 表不是二选一。

`default_definitions.py` 与 `FeatureDefinition` 表不是求合集。

如果数据库有 20 个 active FeatureDefinition，而 `default_definitions.py` 有 14 个，正式计算使用数据库中的 20 个。

如果数据库中某默认特征已 retired / disabled，即使 `default_definitions.py` 里仍存在，也不得参与 production FeatureSet 计算。

### 6.2 seed_feature_definitions 硬规则

命令：

```bash
python manage.py seed_feature_definitions
```

命令行为：

```text
只负责把 default_definitions.py 中的默认模板写入 / 同步到 FeatureDefinition 表
按 feature_code + algorithm_name + algorithm_version + params_hash upsert
不计算 FeatureSet
不生成 FeatureValue
不调用 FeatureLayerService
不请求 Binance
不修改 Kline
必须幂等
不得重复插入已有定义
已存在时只能更新 display_name / description / category / output_unit 等非身份字段
不得覆盖 algorithm_name / algorithm_version / params / params_hash / definition_hash
不得把 retired / disabled 的 FeatureDefinition 自动恢复为 active
不得覆盖人工生命周期配置
```

不建议在 migration 中插入大量特征定义数据。

---

## 7. 第一版默认特征范围

第一版只做基础、可复用、中性数值特征。

当前 004A 第一版实际默认 FeatureDefinition 数量为 14 个。

### 7.1 当前 4h 默认特征

```text
latest_close_4h
sma_4h_20
sma_4h_60
atr_4h_14
range_4h_60_high
range_4h_60_low
return_4h_1
volume_sma_4h_20
```

### 7.2 当前 1d 默认特征

```text
latest_close_1d
sma_1d_20
atr_1d_14
range_1d_60_high
range_1d_60_low
return_1d_1
```

### 7.3 未来候选特征

未来可以新增基础数值特征，例如：

```text
latest_high_4h
latest_low_4h
latest_volume_4h
sma_4h_120
sma_4h_200
range_4h_20_high
range_4h_20_low
return_4h_3
return_4h_6
close_4h_vs_sma_4h_20_pct
close_4h_vs_sma_4h_60_pct
close_4h_vs_sma_4h_200_pct
latest_high_1d
latest_low_1d
latest_volume_1d
sma_1d_60
sma_1d_120
sma_1d_200
range_1d_90_high
range_1d_90_low
range_1d_365_high
range_1d_365_low
return_1d_7
return_1d_30
volume_sma_1d_20
close_1d_vs_sma_1d_200_pct
```

未来候选特征不得包含比较型 bool 条件。

### 7.4 禁止特征

禁止定义带有策略含义的特征代码：

```text
bullish_trend
bearish_trend
long_signal
short_signal
entry_signal
stop_loss_price
take_profit_price
position_size
leverage
should_trade
risk_block
```

也禁止在 FeatureLayer 默认特征或 calculator 中实现比较型 bool 条件。此类条件属于后续 AtomicSignal。

---

## 8. 计算规则

### 8.1 数据来源

FeatureLayer 不请求 Binance。

FeatureLayer 必须通过 MarketSnapshot 窗口回查：

```text
Kline4h
Kline1d
```

不得随便读取最新 K 线。

### 8.2 只基于 usable MarketSnapshot

如果 MarketSnapshot 不可用：

```text
FeatureSet.status = blocked
FeatureSet.is_usable = False
```

不得生成 usable FeatureSet。

### 8.3 不重新做 data_quality

FeatureLayer 不调用 DataQualityService。

MarketSnapshot 已经绑定 DataQualityResult 并通过质量检查。

FeatureLayer 只消费 MarketSnapshot。

### 8.4 窗口数量检查

FeatureLayer 应确认回查到的 K 线数量与 MarketSnapshot 记录的 actual_count 一致。

数量不一致：

```text
FeatureSet.status = failed
FeatureSet.is_usable = False
```

不得继续生成 usable FeatureSet。

### 8.5 精度

涉及价格、成交量、均线、ATR、收益率的计算应使用 Decimal 或可控精度。

不得使用不可控 float 作为最终持久化值。

### 8.6 写库、事务与批量写入

FeatureLayer 可以先在内存中组织计算结果。

最终结果必须写入数据库，后续 AtomicSignal / StrategySignal 不得依赖内存临时特征。

写 FeatureSet / FeatureValue 必须使用 `transaction.atomic()`。

FeatureValue 应使用 `bulk_create` 或等价批量写入。

不得出现 FeatureSet 写入成功但 FeatureValue 只写一半的状态。

failed FeatureSet 不得给后续 AtomicSignal 使用，也不得通过追溯链路进入 StrategySignal。

---

## 9. 生命周期示例

假设：

```text
第 1 轮：active 特征 10 个，全部 v1
第 2 轮：新增 10 个 active 特征，全部 v1
第 3 轮：其中 5 个特征新增 v2，旧 v1 retired
```

则：

```text
FeatureSet_1 使用 definition_set_hash(A_v1...J_v1)，FeatureValue 10 条
FeatureSet_2 使用 definition_set_hash(A_v1...T_v1)，FeatureValue 20 条
FeatureSet_3 使用 definition_set_hash(A_v2...E_v2 + F_v1...T_v1)，FeatureValue 20 条
```

历史 FeatureSet 不自动重算，不覆盖。

如果以后需要用新特征定义重算历史，必须作为 research/backtest/recompute 单独流程，不能覆盖 production FeatureSet。

---

## 10. 与后续模块关系

后续 AtomicSignal 应读取 FeatureSet / FeatureValue，而不是自行重复计算基础特征。StrategySignal 应通过 AtomicSignalSet / AtomicSignalValue 取得策略输入，并通过追溯链路关联 FeatureSet / FeatureValue。

FeatureLayer 不负责：

```text
原子信号
领域信号
策略信号
策略路由
市场环境判断
风控检查
交易决策
订单执行
```

---

## 11. AlertEvent

FeatureLayer 第一版可以不写 AlertEvent，除非 FeatureSet failed / blocked 需要系统告警。

如果写 AlertEvent，必须遵守：

```text
FeatureLayer 只写 AlertEvent
不得直接调用 Hermes client
不得直接调用 notifications service
```

created 默认不应写 AlertEvent，避免通知噪音。

---

## 12. 测试要求

至少覆盖：

```text
FeatureDefinition 可创建
FeatureDefinition 生命周期通过 status 表达
同一 feature_code 可存在不同 algorithm_version
params_hash / definition_hash 稳定
FeatureSet 绑定 MarketSnapshot
FeatureSet 记录 definition_set_hash
FeatureValue 绑定 FeatureDefinition
FeatureValue 冗余 feature_code / algorithm_version / params_hash
usable MarketSnapshot 可以生成 FeatureSet
blocked / failed MarketSnapshot 不能生成 usable FeatureSet
同一 snapshot + definition_set_hash 重复执行返回已有 FeatureSet
active FeatureDefinition 参与计算
retired / disabled FeatureDefinition 不参与新计算
algorithm_name + algorithm_version 可通过 registry 找到 calculator
registry 不支持 feature_compare:v1
算法版本升级不覆盖旧 calculator
Decimal 结果不使用 float 持久化
payload / value_json 中 Decimal 转 string
default definitions 不包含 sma_4h_20_gt_sma_4h_60
build_features 不生成比较型 bool FeatureValue
seed 不恢复 retired / disabled
seed 不调用 FeatureLayerService
FeatureValue 使用 bulk_create 或等价批量写入
写入在 transaction.atomic 中完成
build_features command 可运行
FeatureLayer 不请求 Binance
FeatureLayer 不修改 Kline4h / Kline1d
FeatureLayer 不调用 data_quality
FeatureLayer 不调用 data_backfill
FeatureLayer 不生成 AtomicSignal / StrategySignal
FeatureLayer 不生成交易建议
```

---

## 13. 明确不做

FeatureLayer 第一版不做：

```text
后台 UI
自动特征优选
特征重要性评分
特征低质量自动淘汰
research/backtest 重算流程
AtomicSignal
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
大模型分析
WebSocket
Binance REST 请求
```

---

## 14. 验收标准

FeatureLayer 需求完成后，后续 plan / 实现必须满足：

```text
FeatureDefinition 管理特征定义、算法版本、参数、状态
FeatureSet 绑定 MarketSnapshot
FeatureValue 绑定 FeatureSet 和 FeatureDefinition
FeatureSet 记录 definition_set_hash
FeatureValue 逐条记录 feature_definition / feature_code / algorithm_version / params_hash
Calculator 按 algorithm_name + algorithm_version 注册
参数变化通过 FeatureDefinition 表达
算法变化通过新增 algorithm_version 和 calculator 表达
旧 FeatureDefinition 不物理删除
旧 calculator 行为不被修改
FeatureSet 粒度与 MarketSnapshot 对齐，不是每日一个
default_definitions.py 不是正式运行时特征字典
FeatureDefinition 表是正式运行时特征字典
FeatureLayerService 只读取 status=active 的 FeatureDefinition
FeatureLayerService 不直接读取 default_definitions.py
seed_feature_definitions 幂等且不恢复 retired / disabled
当前 004A 默认 FeatureDefinition 为 14 个基础数值特征
FeatureLayer registry 不注册 feature_compare:v1
FeatureLayer 默认特征不包含 sma_4h_20_gt_sma_4h_60
FeatureLayer 不生成比较型 bool FeatureValue
不产生策略信号
不请求 Binance
不修改 Kline
不生成交易意图，不执行交易
```
