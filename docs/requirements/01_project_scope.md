# 01_project_scope.md

# 项目范围说明

## 1. 文档目的

本文档定义本项目第一阶段的项目目标、系统边界、交付范围和不做事项。

本文档属于需求范围文档，用于回答：

```text
这个项目要做什么？
第一阶段做什么？
第一阶段不做什么？
什么能力只是预留？
什么结果算阶段性交付完成？
```

本文档不定义具体数据库表、字段、Django app 结构、Celery 任务名称、交易所接口细节、策略公式、回测撮合细节和代码实现方案。

具体实现由后续文档补充：

```text
docs/requirements/02_system_capabilities.md
docs/architecture/system_architecture.md
docs/architecture/01_module_map.md
docs/architecture/data_flow.md
docs/requirements/*.md
docs/plans/*.md
```

如本文档与 `docs/rules/project_invariants.md` 冲突，以 `project_invariants.md` 为准。

---

## 2. 项目定位

本项目是一个中低频趋势跟踪自动交易系统。

系统目标是从行情数据开始，经过数据校验、必要时数据回补、特征计算、原子信号、领域判断、策略信号、系统决策、账户与仓位事实同步、价格事实快照、订单计划、风控审查、执行前准备、交易执行、订单 / 成交 / 仓位记录、Hermes 通知和复盘归因，形成一个可回测、可模拟、可实盘、可追溯、可审计、可复盘的自动交易闭环。

本项目不是：

```text
人工喊单系统
人工 Advice 建议系统
大模型实时交易系统
大模型喊单系统
纯回测研究脚本
单纯交易所 API 下单脚本
前端展示平台
```

本项目允许自动交易，但自动交易必须受到结构化风控、交易执行边界、真实交易开关、审计日志和复盘系统约束。

---

## 3. 第一阶段范围

第一阶段只围绕最小可验证自动交易闭环建设。

### 3.1 第一阶段交易范围

```text
交易所：Binance
市场类型：U 本位合约
交易品种：BTCUSDT
交易风格：中低频趋势跟踪
主要驱动：K 线收盘后的周期性分析
```

第一阶段只允许一个运行时账户域参与主链路：

```text
market_type = usds_m_futures
symbol = BTCUSDT
exchange = binance
```

范围澄清：

数据分析域（DataCollection / DataQuality / Backfill / MarketSnapshot / FeatureLayer / AtomicSignal / StrategySignal / DecisionSnapshot）P0 只使用 usds_m_futures / BTCUSDT。

交易执行域（Binance Account Sync / PriceSnapshot / OrderPlan / RiskCheck）P0 需要支持 usds_m_futures 与 coin_m_futures 的 market_type 隔离和计算差异。

运行时仍只能单选一个 active domain。非 active domain 只能作为独立监控对象，不得参与当前主链路决策、风控或交易。

### 3.2 第一阶段系统范围

第一阶段包含以下能力或边界：

```text
数据采集
数据校验
必要时数据回补
MarketSnapshot
FeatureLayer
AtomicSignal
DomainSignal
MarketRegime
StrategyRoute 占位
StrategyParamSnapshot
StrategySignal
StrategySignalQuality
DecisionSnapshot
Binance Account Sync
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
ExchangeOrder
TradeFill
PositionState / PositionSnapshot
Hermes 通知
基础复盘
调度与幂等
监控与人工恢复边界
```

说明：

```text
StrategyRoute 第一阶段可以是轻量占位或默认趋势跟踪路由。
DomainSignal / MarketRegime 第一阶段可以保留清晰边界，先实现最小可用版本。
Binance Account Sync 负责读取账户、余额、仓位和交易规则事实，不做交易决策。
PriceSnapshot 负责提供风控和订单计划使用的价格事实。
DecisionSnapshot 只输出目标仓位意图，不生成订单动作。
OrderPlan 是唯一把目标仓位转换为候选订单意图的模块。
RiskCheck 是交易前强制闸门，只审批 CandidateOrderIntent。
ExecutionPreparation 负责真实提交前的最终准备和校验，不真实下单。
Execution 是唯一允许提交真实订单的模块。
```

### 3.3 第一阶段策略范围

```text
只验证中低频趋势跟踪方向
允许先实现一个默认策略路由
允许策略路由作为轻量占位或 demo
不追求多策略组合
不追求自动策略筛选
不追求自动参数优化
```

策略层输出不得直接表达订单 side、quantity、reduce_only、杠杆或交易所下单参数。

策略判断必须先进入 `DecisionSnapshot`，再由后续账户事实、价格事实、订单计划和风控链路逐步转换为可执行对象。

### 3.4 第一阶段交易执行范围

```text
支持 dry-run
支持 paper trading 或模拟执行
架构允许真实交易
真实交易必须默认关闭
真实交易必须经过配置开关、Binance Account Sync、PriceSnapshot、OrderPlan、RiskCheck、ExecutionPreparation 和 Execution
dry-run、paper trading、real trading 的订单、成交、仓位状态必须隔离
paper trading 的虚拟仓位不得参与 real trading 的账户同步、风控判断或仓位判断
```

任何可真实下单的路径都必须满足：

```text
DecisionSnapshot 可用且未过期
账户事实快照可用且属于 active domain
PriceSnapshot 可用且未过期
OrderPlan 已生成 CandidateOrderIntent
RiskCheck 已生成 ALLOW 的 RiskCheckResult
ApprovedOrderIntent 已生成
ExecutionPreparation 已通过
真实交易开关已显式开启
```

### 3.5 第一阶段允许读取

第一阶段允许读取：

```text
交易所账户
账户权益
可用余额
冻结余额
手续费率
交易所实际杠杆
真实仓位
保证金余额
订单状态
成交状态
交易规则
行情价格事实
```

但读取必须通过明确的边界模块：

```text
账户、余额、仓位、交易规则事实：Binance Account Sync
风控与订单计划使用的价格事实：PriceSnapshot
真实订单提交与订单状态同步：Execution / exchange_gateway
```

读取行为必须受配置开关、trace_id、超时、错误码、失败记录、AlertEvent 和审计日志约束。

`DecisionSnapshot`、`StrategySignal`、`AtomicSignal`、`FeatureLayer` 不得直接读取交易所账户、仓位、余额、交易规则或实时价格。

---

## 4. 第一阶段不做事项

第一阶段不做：

```text
多交易所
多品种组合
复杂投资组合管理
复杂策略路由
复杂多策略权重分配
机器学习交易模型
大模型实时交易决策
大模型生成订单
自动参数优化
自动上线策略
自动禁用策略
复杂前端后台
复杂报表系统
自动调整杠杆
RiskCheck 任意修改订单数量
RiskCheck 自行生成新订单
DecisionSnapshot 直接生成订单动作
```

任何时候都不允许系统自动调整杠杆。

杠杆只能来自 env 或明确配置，且风控系统必须在交易前二次核验。

---

## 5. 第一阶段主链路概览

第一阶段遵循以下主链路：

```text
数据采集
→ 数据校验
→ 必要时数据回补
→ MarketSnapshot
→ FeatureLayer
→ AtomicSignal
→ DomainSignal
→ MarketRegime
→ StrategyRoute
→ StrategyParamSnapshot
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
→ ExchangeOrder / TradeFill / PositionState
→ Hermes 通知
→ 复盘与归因
```

说明：

```text
Binance Account Sync 是账户、余额、仓位和交易规则事实输入，不是交易决策模块。
PriceSnapshot 是订单计划和风控使用的价格事实输入，不是策略信号。
DecisionSnapshot 是系统操盘判断，只表达目标仓位意图，不表达订单动作。
OrderPlan 把目标仓位意图、账户事实、仓位事实、价格事实和交易规则转换为 CandidateOrderIntent。
CandidateOrderIntent 是候选订单意图，不是已批准订单，也不是交易所订单。
RiskCheck 是交易前强制闸门，输出 ALLOW / DENY / BLOCKED / FAILED。
ApprovedOrderIntent 只能由通过 RiskCheck 的 CandidateOrderIntent 生成。
ExecutionPreparation 是真实提交前的执行准备，不得真实下单。
Execution 是唯一允许提交真实订单的模块。
```

大模型不进入实时交易主链路。

大模型只允许在交易结束后、周期结束后或人工触发时，用于复盘总结、异常解释、亏损归因和策略研究辅助。

---

## 6. 第一阶段核心边界概览

第一阶段必须遵守以下边界。详细规则由 `project_invariants.md`、`02_system_capabilities.md`、`01_module_map.md` 和模块级 requirements 定义。

```text
FeatureLayer 只计算特征，不生成交易信号。
AtomicSignal 只记录最小市场判断，不生成交易建议或订单意图。
StrategySignal 不是订单，必须进入 StrategySignalQuality 和 DecisionSnapshot。
StrategyParamSnapshot 用于保证策略参数可追溯。
DecisionSnapshot 不等于订单意图，不得生成 side / quantity / reduce_only。
DecisionSnapshot 不得读取账户、余额、仓位、交易规则或实时价格。
Binance Account Sync 是账户事实输入，不是交易决策模块。
PriceSnapshot 是价格事实输入，不是交易信号。
OrderPlan 是唯一把目标仓位转换为 CandidateOrderIntent 的模块。
CandidateOrderIntent 是候选订单意图，不是已批准订单。
RiskCheck 是交易前强制闸门，只审批 CandidateOrderIntent。
RiskCheck 不得任意修改订单数量、拆单或自行生成新订单。
净额反手场景下，RiskCheck 只能选择 OrderPlan 预生成的 fallback_reduce_only_intent。
ApprovedOrderIntent 是通过风控后的订单意图，不是交易所订单。
ExecutionPreparation 不得真实下单。
Execution 是唯一允许提交真实订单的模块。
Hermes 只负责通知，不触发交易。
大模型只允许用于事后复盘辅助，不进入实时交易链路。
复盘通过 trace_id、DecisionSnapshot ID、OrderPlan ID、RiskCheckResult ID 或 TradeLifecycle ID 聚合证据链。
```

本文档只定义第一阶段范围，不展开各模块的详细输入输出、字段、状态机、算法、数据库结构或调用实现。

---

## 7. 第一阶段核心对象范围

第一阶段涉及的核心对象包括：

```text
MarketData
Kline4h
Kline1d
DataQualityResult
DataQualityIssue
BackfillRequest
BackfillRun
BackfillResult（如保留，仅作为 BackfillRun 的结果摘要语义）
MarketSnapshot
FeatureSet
FeatureValue
FeatureQualityCheck
AtomicSignalDefinition
AtomicSignalValue
AtomicSignalQualityCheck
DomainSignal
DomainQualityCheck
MarketRegime
StrategyRoute
StrategyParamSnapshot
StrategySignal
StrategySignalQualityResult
DecisionSnapshot
BinanceSyncRun
AccountSnapshot
BalanceSnapshot
PositionSnapshot
SymbolRuleSnapshot
PriceSnapshot
OrderPlan
OrderPlanComponent
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
ExecutionPreparation
ExecutionResult
ExchangeOrder
TradeFill
PositionState
TradeLifecycle
ReviewRecord
HermesNotification
AlertEvent
ModelReviewRecord
```

本文档只定义这些对象在项目范围中的存在意义，不定义字段结构。

字段结构、关系设计、索引设计和存储细节，应在后续 architecture 和 module requirements 中定义。

---

## 8. 第一阶段交付目标

第一阶段交付目标不是完整量化平台，而是一个最小可验证自动交易闭环。

第一阶段应至少达到：

```text
能够稳定采集 Binance BTCUSDT U 本位 K 线数据
能够校验数据完整性和连续性
能够在发现缺失 K 线时创建 BackfillRequest，并在 data_backfill 回补后重新质检
能够生成 MarketSnapshot
能够生成 FeatureSet / FeatureValue
能够生成 AtomicSignal
能够生成 StrategySignal
能够生成 StrategySignalQualityResult
能够生成 DecisionSnapshot
能够同步 Binance 账户、余额、仓位和交易规则事实
能够生成 PriceSnapshot
能够通过 OrderPlan 生成 CandidateOrderIntent
能够通过 RiskCheck 阻断或放行 CandidateOrderIntent
能够在风控通过后生成 ApprovedOrderIntent
能够通过 ExecutionPreparation 完成真实提交前准备
能够通过 Execution 执行 dry-run / paper trading
能够隔离 dry-run / paper trading / real trading 的订单、成交、仓位状态
能够记录 ExchangeOrder / TradeFill / PositionState
能够通过 Hermes 发送关键状态通知
能够根据 trace_id / DecisionSnapshot ID / OrderPlan ID / RiskCheckResult ID / TradeLifecycle ID 做基础复盘
能够保留 DomainSignal、MarketRegime、StrategyRoute、StrategyParamSnapshot 的基础边界或最小记录
```

第一阶段完成后，系统应能回答：

```text
当时使用了哪些行情数据？
当时数据质量是否通过？
当时特征值是多少？
哪些原子信号成立？
策略为什么做出这个判断？
StrategySignalQuality 是否允许下游继续？
DecisionSnapshot 给出的目标仓位意图是什么？
账户、余额、仓位和交易规则事实是什么？
PriceSnapshot 使用的价格事实是什么？
OrderPlan 为什么生成这些候选订单意图？
CandidateOrderIntent 的 close / open / reduce-only 组成是什么？
RiskCheck 为什么放行、拒绝、阻断或失败？
是否生成 ApprovedOrderIntent？
ExecutionPreparation 是否通过？
订单是否提交？
订单是否成交？
仓位如何变化？
这次交易或不交易后续表现如何？
```

---

## 9. 第一阶段验收方向

第一阶段验收不以收益率为唯一标准。

第一阶段更重要的验收方向是：

```text
数据是否可信
链路是否完整
决策是否可追溯
风控是否有效
执行是否受控
回测和实盘逻辑是否一致
每次系统行为是否可复盘
```

基础验收标准：

```text
数据采集任务可重复运行且具备幂等控制
缺失或异常 K 线不会静默进入策略链路
缺失 K 线可以通过数据回补流程补齐并重新质检
特征计算结果可追溯到 MarketSnapshot
原子信号可追溯到 FeatureValue
StrategySignal 可追溯到 AtomicSignal 和 StrategyParamSnapshot
DecisionSnapshot 可追溯到 StrategySignal、StrategySignalQualityResult 和证据
Binance Account Sync 结果可追溯到交易所请求、响应摘要和 active domain
PriceSnapshot 可追溯到价格来源和时间戳
OrderPlan 可追溯到 DecisionSnapshot、账户事实、仓位事实、价格事实和交易规则
CandidateOrderIntent 可追溯到 OrderPlan
RiskCheckResult 可追溯到 CandidateOrderIntent 和风控规则
ApprovedOrderIntent 可追溯到 RiskCheckResult
ExecutionPreparation 可追溯到 ApprovedOrderIntent
ExchangeOrder 可追溯到 ExecutionPreparation / ApprovedOrderIntent
TradeFill 可追溯到 ExchangeOrder
PositionState 可追溯到成交或交易所同步
Hermes 通知可追溯到相关业务对象
复盘可以串起数据、特征、信号、决策、账户事实、价格事实、订单计划、风控、执行准备、订单、成交和仓位
paper trading 状态不得污染 real trading 风控判断
```

---

## 10. 风险声明

本项目的核心风险不是代码复杂度，而是：

```text
策略没有真实市场优势
回测存在 overfitting（过拟合）
回测存在 look-ahead bias（前视偏差 / 未来函数）
回测没有考虑手续费、slippage（滑点）和成交约束
数据质量不可信
数据缺失无法回补或回补后未重新质检
回测逻辑和实盘逻辑不一致
账户事实、仓位事实、价格事实或交易规则不可信
风控边界不清
paper trading 与 real trading 状态混用
交易执行不可审计
复盘无法解释系统行为
```

因此第一阶段优先级是：

```text
第一，真实策略有效性。
第二，数据、回测、风控、实盘一致性。
第三，策略组合、监控、复盘。
第四，后台、界面、交互体验。
```

如果某项功能不能直接服务策略验证、风控、执行可靠性或复盘可信度，应延后。
