# Plan 004A：FeatureLayer 基础特征框架

## 0. 执行边界

本 plan 是 003B MarketSnapshot 生成服务之后的 FeatureLayer 第一阶段。

本阶段目标是搭建可扩展的基础特征框架，而不是一次性实现大量特征。

本阶段只做：

```text
FeatureDefinition
FeatureSet
FeatureValue
CalculatorRegistry
少量基础 calculator
少量默认 FeatureDefinition
FeatureLayerService
seed_feature_definitions command
build_features command
tests
implementation 文档
```

本阶段不追求特征数量。第一版只实现基础数值特征，只需要足够验证后续策略模块可以稳定读取 FeatureSet / FeatureValue。

004A 不实现比较型 bool 条件。`*_gt_*`、`feature_compare`、comparison calculator 属于 004B AtomicSignal 或后续原子信号层，不属于 FeatureLayer。

本阶段不做 AtomicSignal、StrategySignal、Risk、Trading。

---

## 1. 阶段目标

本阶段完成后，系统应具备：

```text
apps.feature_layer app
FeatureDefinition model
FeatureSet model
FeatureValue model
FeatureCalculatorRegistry
按 algorithm_name + algorithm_version 注册 calculator
默认 FeatureDefinition seed 机制
基于 usable MarketSnapshot 生成 FeatureSet
FeatureValue 逐条绑定 FeatureDefinition
definition_set_hash 冻结本次特征定义集合
feature_set_key 幂等
Decimal 结果持久化
management command
tests
docs/implementation/004A_feature_layer.md
```

核心目标：

```text
让每个 usable MarketSnapshot 可以生成一批可追溯、可版本化、可复用的基础特征结果，供后续 AtomicSignal / StrategySignal 使用。
```

---

## 2. 依赖文档

Codex 开始前必须阅读：

```text
AGENTS.md
docs/rules/project_invariants.md
docs/requirements/01_project_scope.md
docs/requirements/02_system_capabilities.md
docs/requirements/feature_layer.md
docs/requirements/market_snapshot.md
docs/architecture/data_flow.md
docs/implementation/003A_market_snapshot_model.md
docs/implementation/003B_market_snapshot_generation.md
docs/plans/004A_feature_layer_plan.md
```

如果本 plan 与 `AGENTS.md` 或 `docs/rules/project_invariants.md` 冲突，必须以后两者为准，并在总结中说明冲突。

如果本 plan 与 `docs/requirements/feature_layer.md` 冲突，必须停止并提示用户确认。

---

## 3. 本阶段明确不做

本阶段禁止实现：

```text
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
回测系统
research/recompute 历史重算流程
特征重要性评分
特征自动筛选
后台 UI
大模型分析
WebSocket
Binance REST 请求
data_collection
data_quality
data_backfill
MarketSnapshot 生成服务修改
```

本阶段禁止：

```text
请求 Binance
写 Kline4h / Kline1d
修改 Kline4h / Kline1d
修改 MarketSnapshot
生成 long / short 信号
生成 entry / stop_loss / take_profit
生成 position_size / leverage
生成 should_trade
直接调用 Hermes webhook
```

---

## 4. 模块结构

新增：

```text
apps/feature_layer/
```

建议文件结构：

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
    ranges.py
    returns.py
    volume.py
  management/
    __init__.py
    commands/
      __init__.py
      seed_feature_definitions.py
      build_features.py
  migrations/
```

如果 Codex 判断第一版 calculator 文件过多，可以合并为少量文件，但不得把所有逻辑堆成一个巨大不可维护文件。

---

## 5. 核心模型

### 5.1 FeatureDefinition

FeatureDefinition 是长期特征定义。

必须表达：

```text
特征代码
算法名称
算法版本
算法参数
结果类型
生命周期状态
是否参与 production FeatureSet
```

建议字段：

```text
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

#### 约束

必须支持：

```text
同一 feature_code 可以存在不同 algorithm_version
同一 algorithm_name 可以存在不同 algorithm_version
同一 algorithm_version 可以被多个不同 params 的 FeatureDefinition 复用
```

建议唯一约束：

```text
feature_code + algorithm_name + algorithm_version + params_hash
```

或等价唯一业务约束。

#### 生命周期

FeatureDefinition 不得物理删除。

状态至少支持：

```text
draft
active
deprecated
retired
disabled
```

production FeatureSet 第一版只使用：

```text
status = active
```

#### 算法版本不可变

一旦某个 algorithm_name + algorithm_version 被正式 FeatureValue 使用，不得修改其历史算法行为。

算法升级必须新增 algorithm_version 和 calculator 实现。

---

### 5.2 FeatureSet

FeatureSet 是某个 MarketSnapshot 上的一批特征计算结果。

必须绑定：

```text
market_snapshot
snapshot_key
```

建议字段：

```text
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

#### 粒度

FeatureSet 不是每日一个。

规则：

```text
一个 usable MarketSnapshot + 一个 feature_schema_version + 一个 definition_set_hash
→ 最多一个 production FeatureSet
```

由于 MarketSnapshot 通常按 4h 周期生成，一天内可能生成多个 FeatureSet。

#### 幂等

必须生成稳定 `feature_set_key`。

建议基于：

```text
market_snapshot_id
snapshot_key
feature_schema_version
definition_set_hash
```

生成 SHA-256。

重复执行相同输入时：

```text
返回已有 FeatureSet
不得重复写 FeatureValue
```

---

### 5.3 FeatureValue

FeatureValue 是一个 FeatureSet 内某个 FeatureDefinition 的计算结果。

必须绑定：

```text
feature_set
feature_definition
```

并冗余记录：

```text
feature_code
algorithm_name
algorithm_version
params_hash
definition_hash
value_type
```

建议字段：

```text
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

#### 结果类型

至少支持：

```text
decimal
integer
bool
text
json
```

根据 value_type 写入对应字段。

数值类特征不得用 float 持久化。

---

## 6. definition_set_hash

本阶段必须实现 `definition_set_hash`。

含义：

```text
本次 FeatureSet 使用的 FeatureDefinition 集合指纹
```

生成规则建议：

```text
读取所有 active FeatureDefinition
按 feature_code + algorithm_name + algorithm_version + params_hash 稳定排序
拼接 feature_code / algorithm_name / algorithm_version / params_hash / definition_hash
计算 SHA-256
```

注意：

```text
definition_set_hash 只是集合指纹
不能替代 FeatureValue 对 FeatureDefinition 的逐条绑定
```

---

## 7. CalculatorRegistry

必须实现 calculator registry。

规则：

```text
algorithm_name + algorithm_version → calculator class
```

示例：

```text
("latest_value", "v1") → LatestValueV1Calculator
("simple_moving_average", "v1") → SimpleMovingAverageV1Calculator
("atr_wilder", "v1") → AtrWilderV1Calculator
("rolling_high", "v1") → RollingHighV1Calculator
("rolling_low", "v1") → RollingLowV1Calculator
("return_pct", "v1") → ReturnPctV1Calculator
("volume_sma", "v1") → VolumeSmaV1Calculator
```

`feature_compare:v1` 不属于 FeatureLayer registry，由 AtomicSignal 层实现。FeatureLayer 不包含 `calculators/comparisons.py`，也不注册 comparison calculator。

禁止在 FeatureLayerService 中写大量 if/else 判断算法。

calculator 必须是纯计算逻辑：

```text
不读数据库
不写数据库
不请求 Binance
不写 AlertEvent
不生成信号
不调用策略
```

---

## 8. 第一版默认特征

本阶段只需要实现少量基础数值特征，验证框架可用。

不要一次性实现 100 个特征。

第一版默认 FeatureDefinition 固定为当前 14 个基础数值特征：

### 8.1 4h

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

### 8.2 1d

```text
latest_close_1d
sma_1d_20
atr_1d_14
range_1d_60_high
range_1d_60_low
return_1d_1
```

第一版默认特征不得包含比较型 bool 条件，例如 `sma_4h_20_gt_sma_4h_60`。这类 `*_gt_*` 条件属于 004B AtomicSignal 或后续原子信号层。

当前默认特征必须覆盖：

```text
latest value
SMA
ATR
rolling high/low
return pct
```

禁止定义策略语义特征，例如：

```text
bullish_trend
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

---

## 9. default_definitions 与 seed 命令

必须新增：

```text
apps/feature_layer/default_definitions.py
```

保存默认 FeatureDefinition 列表。

必须新增命令：

```bash
python manage.py seed_feature_definitions
```

命令行为：

```text
读取 default_definitions
计算 params_hash / definition_hash
按 feature_code + algorithm_name + algorithm_version + params_hash upsert
已存在则跳过或仅更新 display_name / description 等非关键字段
不得覆盖 algorithm_name / algorithm_version / params
不得物理删除已存在 FeatureDefinition
```

seed 命令只负责定义，不计算 FeatureSet。

---

## 10. FeatureLayerService

必须实现：

```text
FeatureLayerService
```

主方法建议：

```text
build_features(market_snapshot_id 或 snapshot_key, trigger_source, trace_id)
```

流程：

```text
1. 读取 MarketSnapshot
2. 确认 status=created 且 is_usable=True
3. 根据 MarketSnapshot 窗口字段回查 Kline4h / Kline1d
4. 确认 Kline 查询数量等于 MarketSnapshot actual_count
5. 读取 active FeatureDefinition
6. 计算 definition_set_hash
7. 生成 feature_set_key
8. 如果 FeatureSet 已存在，返回已有 FeatureSet
9. 冻结本次 active FeatureDefinition 集合
10. 在内存中逐个计算 FeatureValue
11. 使用 transaction.atomic 写 FeatureSet 和 FeatureValue
12. 返回结构化结果
```

### 10.1 写入要求

计算结果可以先在内存中组织，但最终必须写库。

写库必须事务化：

```text
transaction.atomic()
→ 创建 FeatureSet
→ bulk_create FeatureValue
→ 提交
```

不得出现 FeatureSet 写入成功但 FeatureValue 只写一半的状态。

FeatureValue 应使用 `bulk_create`，不要逐条 save。

### 10.2 失败处理

第一版规则：

```text
active FeatureDefinition 全部视为 required
任一 required 特征计算失败
→ FeatureSet.status = failed
→ FeatureSet.is_usable = False
```

可以保存已计算成功和失败的 FeatureValue 作为排查依据，但后续策略不得使用 failed FeatureSet。

如果实现复杂，可以只写 failed FeatureSet 和错误摘要，不写部分 FeatureValue；但 implementation 文档必须说明。

---

## 11. selectors

selectors 负责只读查询：

```text
读取 usable MarketSnapshot
根据 MarketSnapshot 回查 Kline4h / Kline1d
读取 active FeatureDefinition
```

FeatureLayer 不得随便读取最新 K 线。

必须使用 MarketSnapshot 中记录的窗口：

```text
start_4h_open_time_utc
end_4h_open_time_utc
actual_4h_count
start_1d_open_time_utc
end_1d_open_time_utc
actual_1d_count
```

如果查询数量与 MarketSnapshot 不一致：

```text
FeatureSet.status = failed
is_usable = False
error_code = kline_window_mismatch
```

---

## 12. build_features 命令

必须新增：

```bash
python manage.py build_features
```

至少支持：

```text
--market-snapshot-id
--snapshot-key
--trigger-source manual|analysis_cycle|scheduled
--trace-id
```

规则：

```text
command 只解析参数并调用 FeatureLayerService
不承载核心计算逻辑
输出 FeatureSet id / feature_set_key / status / computed_count / valid_count / invalid_count
```

---

## 13. 不调用其他业务链路

本阶段不得调用：

```text
DataCollectionService
DataQualityService
BackfillService
MarketSnapshotService
BinanceMarketDataClient
Hermes client
Strategy service
Risk service
Trading service
```

FeatureLayer 只消费已经存在的 usable MarketSnapshot。

---

## 14. AlertEvent

第一版可以不写 AlertEvent。

如果 Codex 实现 failed / blocked AlertEvent，必须遵守：

```text
只写 AlertEvent
不得直接调用 Hermes client
不得直接调用 notifications service
created 默认不写 AlertEvent
```

为了控制范围，建议第一版暂不写 AlertEvent。

---

## 15. 数据库与 migration

必须使用 Django migrations。

必须注册 app：

```text
apps.feature_layer
```

必须新增 migration。

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

## 16. 测试要求

必须测试：

```text
FeatureDefinition 可创建
同一 feature_code 可存在不同 algorithm_version
params_hash 稳定
definition_hash 稳定
retired / disabled FeatureDefinition 不参与 active 集合
FeatureSet 可创建并绑定 MarketSnapshot
FeatureValue 可创建并绑定 FeatureDefinition
FeatureValue 冗余 feature_code / algorithm_version / params_hash / definition_hash
definition_set_hash 稳定
CalculatorRegistry 能按 algorithm_name + algorithm_version 找到 calculator
unsupported calculator 会失败
usable MarketSnapshot 可以生成 FeatureSet
blocked / failed MarketSnapshot 不能生成 usable FeatureSet
Kline 查询数量与 MarketSnapshot 不一致会 failed
相同 snapshot + definition_set_hash 重复执行返回已有 FeatureSet
FeatureValue 使用 bulk_create 或等价批量写入
写入在 transaction.atomic 中完成
Decimal 特征不以 float 持久化
default definitions 不包含 sma_4h_20_gt_sma_4h_60
registry 不支持 feature_compare:v1
build_features 不生成比较型 bool FeatureValue
seed_feature_definitions command 可运行
build_features command 可运行
FeatureLayer 不请求 Binance
FeatureLayer 不写 Kline4h / Kline1d
FeatureLayer 不调用 data_quality
FeatureLayer 不调用 data_backfill
FeatureLayer 不生成 AtomicSignal / StrategySignal
FeatureLayer 不生成交易建议
```

---

## 17. 越界检查

Codex 完成后建议运行：

```bash
grep -R "requests.get" apps/feature_layer tests/feature_layer
grep -R "httpx" apps/feature_layer tests/feature_layer
grep -R "Binance" apps/feature_layer tests/feature_layer
grep -R "DataQualityService" apps/feature_layer tests/feature_layer
grep -R "BackfillService" apps/feature_layer tests/feature_layer
grep -R "MarketSnapshotService" apps/feature_layer tests/feature_layer
grep -R "AtomicSignal" apps/feature_layer tests/feature_layer
grep -R "StrategySignal" apps/feature_layer tests/feature_layer
grep -R "RiskCheck" apps/feature_layer tests/feature_layer
grep -R "CandidateOrderIntent / ApprovedOrderIntent" apps/feature_layer tests/feature_layer
grep -R "Execution" apps/feature_layer tests/feature_layer
grep -R "create_order" apps/feature_layer tests/feature_layer
grep -R "place_order" apps/feature_layer tests/feature_layer
```

如果命中，Codex 必须解释原因。

---

## 18. implementation 文档要求

完成后必须新增：

```text
docs/implementation/004A_feature_layer.md
```

内容至少包括：

```text
本阶段新增 app / model
FeatureDefinition 字段说明
FeatureSet 字段说明
FeatureValue 字段说明
FeatureSet 与 MarketSnapshot 的关系
definition_set_hash 生成规则
feature_set_key 幂等规则
CalculatorRegistry 说明
默认 FeatureDefinition 说明
seed_feature_definitions 命令说明
build_features 命令说明
FeatureLayerService 流程
失败处理规则
测试方式
本阶段明确未实现内容
```

必须明确写入：

```text
本阶段未实现 AtomicSignal。
本阶段未实现 StrategySignal。
本阶段未实现 MarketRegime。
本阶段未实现 Risk / Trading。
本阶段未请求 Binance。
本阶段未修改 Kline4h / Kline1d。
本阶段未调用 data_quality / data_backfill。
本阶段未生成交易建议。
```

---

## 19. Codex 输出总结要求

Codex 完成后必须输出：

```text
修改 / 新增文件清单
新增 app / model 名称
migration 文件路径
默认 FeatureDefinition 数量
已实现 calculator 列表
FeatureDefinition 生命周期规则
definition_set_hash 规则
feature_set_key 幂等规则
FeatureLayerService 流程
seed_feature_definitions command 说明
build_features command 说明
测试命令和结果
无法运行的命令及原因
是否存在未完成项
```

并明确说明：

```text
本阶段未实现 AtomicSignal / StrategySignal。
本阶段未实现 Risk / Trading。
本阶段未请求 Binance。
本阶段未修改 Kline。
本阶段未调用 data_quality / data_backfill。
本阶段未生成交易建议。
```

---

## 20. 最终验收标准

本阶段完成后，必须满足：

```text
apps.feature_layer 可加载
FeatureDefinition 存在
FeatureSet 存在
FeatureValue 存在
FeatureDefinition 支持算法版本和 params
FeatureDefinition 不物理删除，通过 status 管理生命周期
FeatureSet 绑定 MarketSnapshot
FeatureValue 绑定 FeatureDefinition
definition_set_hash 可生成且稳定
feature_set_key 幂等
CalculatorRegistry 可用
至少有少量基础 calculator
seed_feature_definitions 可运行
build_features 可运行
usable MarketSnapshot 可生成 FeatureSet
不可用 MarketSnapshot 不生成 usable FeatureSet
同一 snapshot + definition_set_hash 不重复生成
不请求 Binance
不写 Kline
不调用 data_quality / data_backfill
不生成信号 / 策略 / 风控 / 交易
测试通过
```

---

## 21. 下一阶段

本阶段完成并验收后，下一阶段才开始：

```text
004B_atomic_signal_plan.md
```

004B 才允许实现：

```text
基于 FeatureSet / FeatureValue 生成中性原子信号
不直接生成最终策略决策
```
