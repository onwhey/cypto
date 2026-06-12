# 项目系统总览

## 1. 文档目的

本文档描述中低频趋势跟踪自动交易系统的整体架构与模块总览，为后续模块级 requirements、数据流、实现和开发计划提供参考。

本文档只提供项目级总览，不定义具体数据库表、字段、Celery task、Django app 目录、策略公式或代码实现。

详细边界以以下文档为准：

```text
docs/rules/project_invariants.md
docs/architecture/01_module_map.md
docs/architecture/system_architecture.md
docs/architecture/data_flow.md
docs/requirements/*.md
docs/plans/*.md
```

如果本文档与 `project_invariants.md` 冲突，以 `project_invariants.md` 为准。

---

## 2. 项目定位

本项目是一个中低频趋势跟踪自动交易系统。

系统目标是构建一个从行情数据、特征、信号、决策、账户事实、价格事实、订单计划、风控、执行、追踪、复盘到审计的自动交易闭环。

本项目允许自动交易，但所有真实交易必须经过：

```text
结构化策略决策
DecisionSnapshot 目标仓位表达
Binance Account Sync 账户事实
PriceSnapshot 价格事实
OrderPlan 候选订单计划
RiskCheck 风控审批
ApprovedOrderIntent 风控通过意图
ExecutionPreparation 执行前最终检查
Execution 真实执行边界
Tracking 订单 / 成交 / 仓位追踪
审计日志
复盘追踪
```

本项目不是：

```text
人工喊单系统
大模型实时交易系统
单纯回测脚本
单纯交易所 API 下单脚本
前端展示平台
```

大模型不得参与实时交易决策。

---

## 3. 当前核心链路

核心主链路为:

```text
数据采集
→ DataQuality 可在可回补 FAIL 时创建 BackfillRequest
→ data_backfill 后续消费 pending BackfillRequest 并执行回补
→ 回补完成后标记 / 要求 DataQuality 复检
→ 新的 DataQualityResult = PASS 且 allows_downstream=True
→ MarketSnapshot
→ FeatureLayer
→ AtomicSignal
→ DomainSignal
→ MarketRegime
→ StrategyRoute
→ StrategyParamSnapshot
→ StrategySignal
→ DecisionSnapshot
→ Binance Account Sync
→ PriceSnapshot
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation
→ Execution
→ Tracking
→ Hermes / AlertEvent
→ ReviewRecord
```

核心语义：

```text
DecisionSnapshot 只表达目标仓位，不生成订单动作。
Binance Account Sync 是只读账户事实层。
PriceSnapshot 是价格事实层。
OrderPlan 负责把目标仓位转换为 CandidateOrderIntent。
CandidateOrderIntent 是待风控审批的候选订单意图。
RiskCheck 只审批 CandidateOrderIntent，不直接审批 DecisionSnapshot。
ApprovedOrderIntent 是风控通过后的订单意图，不等于交易所订单。
ExecutionPreparation 做执行前最终检查和 price guard。
Execution 是唯一真实下单入口。
Tracking 负责订单、成交、仓位追踪。
Hermes / AlertEvent 负责通知和事件审计，不触发交易。
ReviewRecord 负责事后复盘，不自动修改生产策略。
```

---

## 4. 系统模块总览

### 4.1 数据采集模块

职责：

```text
采集历史 K 线和最新已收盘 K 线
P0 使用 REST 采集 K 线
不负责修复数据缺口
不负责生成交易信号
不访问交易所交易接口
```

扩展点：

```text
资金费率
订单簿摘要
成交数据
WebSocket 实时行情监测边界预留
```

注意：WebSocket 不参与 K 线采集。后续 WebSocket 只能作为价格流或账户事件流的来源之一，不能绕过数据质量和价格事实层。

---

### 4.2 数据质量模块

职责：

```text
校验数据完整性、连续性、重复、缺失、异常和收盘状态
阻断不合格数据进入 MarketSnapshot
发现缺失或断档时，创建 BackfillRequest
PASS 只写 DataQualityResult；FAIL 写 DataQualityResult / DataQualityIssue / 必要 AlertEvent
```

数据质量未通过时，不得生成正式 MarketSnapshot。

---

### 4.3 数据回补模块

职责：

```text
扫描 / 领取 pending BackfillRequest，claim 后创建 BackfillRun 并执行回补
回补完成后只标记 / 要求 DataQuality 复检，不直接触发 DataQuality task
实际复检由后续统一编排层、recovery scan 或人工命令重新执行
只有新的 DataQualityResult = PASS 且 allows_downstream=True，才允许继续 MarketSnapshot
不允许人工伪造、手工改写或推测补齐 K 线
```

数据回补是条件补偿链路，不是每次主链路必经步骤。
DataQuality 不直接触发 data_backfill，不执行回补。
data_backfill 不同步调用 DataQuality，不等待 DataQuality 结果。
MarketSnapshotService 只读取 / 校验已有 DataQualityResult；回补完成不能直接放行 MarketSnapshot。

---

### 4.4 MarketSnapshot 模块

职责：

```text
在策略分析周期固定市场状态
输出 MarketSnapshot
作为 FeatureLayer、AtomicSignal、StrategySignal、DecisionSnapshot 和复盘的共同证据基础
```

MarketSnapshot 必须追溯到：

```text
KlineRecord
DataQualityResult
lookback 配置
analysis_close_time_utc
trace_id
trigger_source
```

MarketSnapshot 生成失败时，不得进入 FeatureLayer。

---

### 4.5 FeatureLayer 模块

职责：

```text
统一计算基础特征，例如 RSI、EMA、ATR、ADX 等
支持中性复合特征，例如结构点、突破区间、回撤幅度
输出 FeatureSet、FeatureValue、FeatureQualityCheck
单周期内复用特征，避免重复查询和重复计算
```

FeatureLayer 不生成交易信号，不表达交易观点。

---

### 4.6 原子信号模块

职责：

```text
基于 FeatureSet / FeatureValue 生成最小市场事件判断
输出 AtomicSignal、AtomicSignalQualityCheck
支持单信号独立拆分和质量检查
```

原子信号之间不得互相调用、互相覆盖或互相否定。

原子信号不得生成 StrategySignal、DecisionSnapshot、CandidateOrderIntent 或交易动作。

---

### 4.7 领域模块

职责：

```text
聚合同类 AtomicSignal
输出 DomainSignal，例如趋势、动量、波动、突破验证等
解释同类信号之间的确认、冲突、否定和降权关系
```

领域模块不直接生成订单，不访问交易所。

---

### 4.8 市场环境识别模块

职责：

```text
根据 DomainSignal 判断 MarketRegime
输出趋势、震荡、高波动、低波动、不适合交易等状态
```

市场状态不明时，不得默认允许交易。

---

### 4.9 策略路由模块

职责：

```text
根据 MarketRegime、DomainSignal 和配置选择策略
输出 StrategyRoute
第一阶段可默认单策略，但必须保留路由边界
```

无匹配策略时，应输出不可交易 / 无路由状态，不得默认执行错误策略。

---

### 4.10 策略参数管理模块

职责：

```text
集中管理策略参数来源、参数版本和 StrategyParamSnapshot
保证 StrategySignal、DecisionSnapshot 和复盘可追溯到当时使用的参数
```

第一阶段可采用参数快照嵌入方式，避免过早实现复杂参数版本系统。

---

### 4.11 策略信号模块

职责：

```text
输出 StrategySignal
表达方向、强度、置信度、证据摘要、策略模式和参数版本
```

StrategySignal 不是订单，不得直接进入交易所。

StrategySignal 必须进入 DecisionSnapshot。

---

### 4.12 DecisionSnapshot 模块

职责：

```text
记录系统每个策略分析周期的决策快照
基于 StrategySignal 和上游证据生成目标仓位表达
输出 target_intent / target_position_ratio 等目标仓位字段
```

DecisionSnapshot 必须追溯：

```text
MarketSnapshot
FeatureSet
AtomicSignal
DomainSignal
MarketRegime
StrategyRoute
StrategySignal
StrategyParamSnapshot
trace_id
trigger_source
```

重要边界：

```text
DecisionSnapshot 不输出 ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE 作为订单动作。
DecisionSnapshot 不读取账户、余额、持仓或 BinanceSyncRun。
DecisionSnapshot 不生成 CandidateOrderIntent。
DecisionSnapshot 不调用交易所。
DecisionSnapshot 不等于订单计划。
```

---

### 4.13 Binance Account Sync 模块

职责：

```text
只读同步 Binance active account domain 的账户、余额、持仓和 symbol rule
生成 BinanceSyncRun、Account / Balance / Position / SymbolRule 快照
为 PriceSnapshot、OrderPlan、RiskCheck 提供账户事实
```

P0 特征：

```text
Binance 专属
只读
单 active market_type / account_domain
usds_m_futures 与 coin_m_futures 严格隔离
不下单、不撤单、不划转、不调杠杆、不调保证金模式
```

关键字段：

```text
account_domain
position_mode
observed_exchange_leverage
contract_size（COIN-M）
margin_asset / settlement_asset
mark_price
current_equity / available_balance
snapshot_hash / snapshot_set_hash
```

注意：`observed_exchange_leverage` 是交易所实际观测杠杆，不参与 OrderPlan 目标仓位计算。

---

### 4.14 PriceSnapshot 模块

职责：

```text
维护价格事实快照
为 OrderPlan、RiskCheck、ExecutionPreparation 提供可追溯价格输入
```

P0 可从 BinancePositionSnapshot.mark_price 派生。

后续 WebSocket / REST ticker / mark price / best bid ask 都只能作为 PriceSnapshot 的来源之一。下游不得直接依赖 WebSocket。

PriceSnapshot 必须具备：

```text
symbol
market_type
account_domain
price_type
mark_price 或等价价格字段
source
as_of_utc
price_snapshot_hash
TTL / freshness check
```

---

### 4.15 OrderPlan 模块

职责：

```text
把 DecisionSnapshot 的目标仓位转换为 CandidateOrderIntent
计算目标名义仓位与目标数量
处理当前仓位与目标仓位之间的差值
记录 order_components
```

OrderPlan 输入：

```text
DecisionSnapshot.target_position_ratio
BinanceSyncRun / trading context
PriceSnapshot
当前持仓
当前权益
symbol rule
配置
```

OrderPlan 输出：

```text
OrderPlan
CandidateOrderIntent
order_components
必要时 fallback_reduce_only_intent
```

重要边界：

```text
OrderPlan 不访问 Binance API。
OrderPlan 不真实下单。
OrderPlan 不做最终风控审批。
OrderPlan 不修改杠杆。
OrderPlan 不修改保证金模式。
OrderPlan 不连接 WebSocket。
```

P0 只支持 Binance One-Way Mode。Hedge Mode 必须 blocked。

---

### 4.16 CandidateOrderIntent 模块

职责：

```text
表达待 RiskCheck 审批的候选订单意图
保存 OrderPlan 生成的订单方向、数量、价格参考、reduceOnly、positionSide 和组件拆解
```

CandidateOrderIntent 不是可执行订单。

只有 RiskCheck 通过后，才能生成 ApprovedOrderIntent。

---

### 4.17 RiskCheck 模块

职责：

```text
审批 CandidateOrderIntent
检查账户事实、价格事实、当前持仓、symbol rule、余额、保证金、杠杆观测值、有效杠杆和风控规则
输出 RiskCheckResult
```

RiskCheck 输入：

```text
CandidateOrderIntent
OrderPlan
BinanceSyncRun / trading context
PriceSnapshot
当前账户快照
当前持仓快照
symbol rule
风险配置
```

RiskCheck 输出：

```text
RiskCheckResult
ALLOW / DENY / BLOCKED / FAILED
selected CandidateOrderIntent
```

重要边界：

```text
RiskCheck 不直接审批 DecisionSnapshot。
RiskCheck 不生成任意新订单。
RiskCheck 不真实下单。
RiskCheck 不修改杠杆。
RiskCheck 不修改保证金模式。
RiskCheck 不连接 WebSocket。
```

P0 不做任意 MODIFY / 自动缩仓。只允许选择 OrderPlan 预生成的 fallback_reduce_only_intent。

所有正式 RiskCheck 结果，包括 ALLOW、DENY、BLOCKED、FAILED，都必须写 AlertEvent。

---

### 4.18 ApprovedOrderIntent 模块

职责：

```text
表达 RiskCheck 审批通过后的订单意图
作为 ExecutionPreparation 的输入
```

ApprovedOrderIntent 不等于交易所订单。

ApprovedOrderIntent 不代表已提交、不代表已成交。

---

### 4.19 ExecutionPreparation 模块

职责：

```text
在真实执行前执行最终校验
刷新或读取最新 PriceSnapshot
执行 price guard
确认 ApprovedOrderIntent 未过期、未执行、未被撤销
确认真实交易开关和执行模式
```

P0 / 后续执行前 price guard 语义：

```text
最新价格必须足够新鲜
价格偏离不得超过允许阈值
超过阈值则阻断执行或要求重新风控
```

ExecutionPreparation 不生成策略信号，不生成 CandidateOrderIntent，不审批风控，不真实下单。

---

### 4.20 Execution 模块

职责：

```text
唯一真实下单入口
接收通过 ExecutionPreparation 的 ApprovedOrderIntent
根据 execution_mode 执行 dry-run / paper trading / real trading
生成 ExecutionOrder / ExchangeOrder / ExecutionReport 等执行记录
```

Execution 可以调用 trading gateway。

除 Execution 外，其他模块不得调用真实下单、撤单、订单查询和成交查询等交易类接口。

Execution 不得绕过 RiskCheck，也不得自动修改杠杆或保证金模式。

---

### 4.21 Tracking 模块

职责：

```text
追踪订单状态
追踪成交
同步仓位
处理部分成交、状态未知、订单超时和仓位不一致
```

输出对象包括：

```text
ExchangeOrder
TradeFill
PositionState
OrderStatusRecord
PositionSyncRecord
```

订单状态未知或仓位不一致时，必须告警，并阻断后续真实交易，等待受控恢复。

---

### 4.22 Hermes / AlertEvent 通知模块

职责：

```text
发送系统状态、交易状态、异常和复盘通知
记录 AlertEvent / HermesNotification
```

所有正式订单相关事件必须写 AlertEvent。

包括但不限于：

```text
OrderPlan blocked / generated
CandidateOrderIntent generated
RiskCheck ALLOW / DENY / BLOCKED / FAILED
ApprovedOrderIntent generated
ExecutionPreparation pass / blocked / failed
Execution submitted / failed
ExchangeOrder state change
TradeFill generated
PositionState changed
Tracking anomaly
```

Hermes 只通知，不参与决策，不触发下单，不生成订单意图。

---

### 4.23 复盘模块

职责：

```text
根据 trace_id、DecisionSnapshot ID 或交易生命周期 ID 聚合全链路证据
输出 ReviewRecord
```

复盘证据包括：

```text
MarketSnapshot
FeatureSet
AtomicSignal
DomainSignal
MarketRegime
StrategyRoute
StrategySignal
DecisionSnapshot
BinanceSyncRun
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
ExecutionPreparation
Execution
ExchangeOrder
TradeFill
PositionState
AlertEvent / HermesNotification
```

复盘结论不得自动修改生产策略。

---

### 4.24 策略评估与进化能力

职责：

```text
长期统计策略、原子信号、领域模块和市场环境下的表现
输出人工确认的优化、降权或禁用建议
```

不得自动修改生产策略。

---

### 4.25 事件日历能力

职责：

```text
配置重要市场事件
辅助风险控制和策略限制
```

第一阶段不进入主交易链路。

后续如影响交易，必须通过 RiskCheck 或明确的策略规则显式处理。

---

### 4.26 大模型复盘辅助模块

职责：

```text
用于事后复盘、异常解释和策略研究
输出 ModelReviewRecord
```

大模型不参与实时交易链路，不影响 RiskCheck，不生成 CandidateOrderIntent 或 ApprovedOrderIntent。

---

### 4.27 调度与幂等模块

职责：

```text
负责周期调度、任务入口和防重复执行
业务幂等键不只依赖 trace_id
```

幂等重点：

```text
同一分析周期不得重复生成有效 DecisionSnapshot。
同一 DecisionSnapshot 不得重复生成冲突 OrderPlan。
同一 OrderPlan 不得重复生成多个有效 CandidateOrderIntent。
同一 CandidateOrderIntent 不得重复生成多个有效 RiskCheckResult。
同一 ApprovedOrderIntent 不得重复提交执行。
```

重复调度不得导致重复下单。

---

### 4.28 监控与异常恢复模块

职责：

```text
异常发现
告警
基础设施级自动恢复
业务级人工恢复入口
```

业务级恢复必须人工确认，不得绕过 RiskCheck、ExecutionPreparation 或 Execution 边界。

---

## 5. 模块间总体数据流

```text
数据采集
→ DataQuality 可在可回补 FAIL 时创建 BackfillRequest
→ data_backfill 后续消费 pending BackfillRequest 并执行回补
→ 回补完成后标记 / 要求 DataQuality 复检
→ 新的 DataQualityResult = PASS 且 allows_downstream=True
→ MarketSnapshot
→ FeatureLayer
→ AtomicSignal
→ DomainSignal
→ MarketRegime
→ StrategyRoute
→ StrategyParamSnapshot
→ StrategySignal
→ DecisionSnapshot
→ Binance Account Sync / PriceSnapshot
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation
→ Execution
→ Tracking
→ Hermes / AlertEvent
→ ReviewRecord
```

说明：

```text
数据采集负责正常获取数据，数据回补负责发现缺口后补齐数据链路。
MarketSnapshot 是每次策略分析的共同证据基础。
FeatureLayer 提供所有原子信号共享特征。
原子信号互不调用，只记录自身判断和证据。
领域模块负责解释同类信号之间的确认、冲突、否定和降权关系。
DecisionSnapshot 是系统每周期目标仓位决策的关键记录。
Binance Account Sync 是账户事实层，不是交易执行层。
PriceSnapshot 是价格事实层，不是 WebSocket 本身。
OrderPlan 是目标仓位到候选订单意图的转换层。
CandidateOrderIntent 是待风控审批的候选订单意图。
RiskCheck 是 CandidateOrderIntent 的风控审批层。
ApprovedOrderIntent 是风控通过后的订单意图，不是交易所订单。
ExecutionPreparation 是执行前最终检查和 price guard。
Execution 是唯一真实下单入口。
Tracking 记录订单、成交和仓位状态。
Hermes / AlertEvent 只通知和审计，不参与决策或下单。
ReviewRecord 通过 trace_id、DecisionSnapshot ID 或交易生命周期 ID 聚合证据链。
大模型只做事后复盘辅助，不进入实时交易链路。
```

---

## 6. 当前 P0 边界摘要

P0 必须坚持：

```text
Binance 专属
单 active account_domain / market_type
UTC-only
REST K 线采集
K 线质量检查 fail-closed
MarketSnapshot 作为市场证据基础
FeatureLayer / AtomicSignal / StrategySignal / DecisionSnapshot 可追溯
DecisionSnapshot 只表达 target_position_ratio
Account Sync 只读
PriceSnapshot 独立价格事实层
OrderPlan 生成 CandidateOrderIntent
RiskCheck 审批 CandidateOrderIntent
ExecutionPreparation 执行最终检查
Execution 才允许真实下单
所有订单相关正式事件 AlertEvent
大模型不进入实时交易链路
Hermes 不触发交易
```

P0 不做：

```text
WebSocket 参与 K 线采集
自动划转
自动补保证金
自动调杠杆
自动调保证金模式
大模型实时交易
自动参数优化
自动策略上线
未经验证的实盘执行
任意 MODIFY / 自动缩仓
绕过 RiskCheck 的真实下单
```

---

## 7. 与其他文档关系

```text
docs/rules/project_invariants.md
= 系统最高红线和不变量

docs/architecture/01_module_map.md
= 模块职责、输入输出、允许调用、禁止调用、失败处理

docs/architecture/system_architecture.md
= 工程分层、存储、调度、gateway、同步异步、模式隔离、trace、幂等和恢复架构

docs/architecture/data_flow.md
= 一次运行中的数据流、对象流、条件分支和阻断规则

docs/requirements/*.md
= 模块级详细需求

docs/plans/*.md
= 开发计划和实施步骤
```

本文档只作为总览，不替代上述详细文档。
