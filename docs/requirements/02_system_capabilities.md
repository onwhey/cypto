# 02_system_capabilities.md

# 系统能力总览

## 1. 文档目的

本文档定义本项目的系统能力地图。

本文档用于承接用户对完整系统的预期，回答：

```text
系统最终应该具备哪些能力？
第一阶段必须具备哪些能力？
哪些能力第一阶段只做简化或占位？
哪些能力后续再做？
哪些能力明确禁止？
各能力之间有什么关键依赖？
哪些能力需要预留扩展点，避免后续重构？
```

本文档不定义具体实现细节。

本文档不定义：

```text
数据库表结构
字段设计
索引设计
Django app 结构
Celery task 名称
交易所 API endpoint
WebSocket stream 名称
具体策略公式
具体回测撮合算法
具体订单状态机
具体代码实现
```

具体实现应在后续文档中定义：

```text
docs/architecture/system_architecture.md
docs/architecture/01_module_map.md
docs/architecture/data_flow.md
docs/requirements/*.md
docs/plans/*.md
```

如本文档与 `docs/rules/project_invariants.md` 冲突，以 `project_invariants.md` 为准。

---

## 2. 能力优先级定义

本文档使用以下优先级：

```text
P0 = 第一阶段必须具备的能力
P1 = 第一阶段需要保留边界，可简化实现或占位
P2 = 后续阶段再实现的能力
禁止 = 当前项目明确不允许实现的能力
```

优先级解释：

```text
P0
= 没有该能力，第一阶段闭环无法成立。

P1
= 第一阶段不需要复杂实现，但需要预留位置，避免后期返工。

P2
= 不是第一阶段目标，后续在策略、风控、执行和复盘稳定后再考虑。

禁止
= 与项目不变量冲突，不能实现。
```

### 2.1 P0 / P1 断层控制原则

部分能力在第一阶段可能只做简化实现，但不得只留下空概念，导致后续扩展时重构核心链路。

因此：

```text
P0 如果只做基础版本，也必须保留清晰输入、输出、状态、追溯关系和扩展边界。
P1 如果作为占位，必须说明后续接入点，不得影响 P0 主链路稳定运行。
P0 对象不得因为 P1 尚未实现而缺失关键追溯引用。
P0 设计不得封死后续领域模块、策略路由、复盘归因和多策略扩展。
```

---

## 3. 总体能力地图

系统能力分为以下几类：

```text
数据能力
数据回补能力
数据质量能力
市场快照能力
特征能力
原子信号能力
领域模块能力
市场环境识别能力
策略路由能力
策略信号能力
策略信号质量能力
策略参数管理能力
DecisionSnapshot 能力
回测能力
Binance Account Sync 能力
PriceSnapshot 能力
OrderPlan 能力
CandidateOrderIntent / ApprovedOrderIntent 能力
RiskCheck 能力
ExecutionPreparation 能力
Execution 能力
订单与成交能力
仓位同步与仓位记录能力
Hermes 通知能力
复盘能力
策略评估与进化能力
事件日历能力
大模型复盘辅助能力
审计与追踪能力
调度与幂等能力
监控与异常恢复能力
配置与安全能力
```

第一阶段目标不是一次性完成复杂量化平台，而是完成最小可信自动交易闭环。

第一阶段主交易链路为：

```text
数据采集
→ 数据质量
→ 必要时数据回补
→ MarketSnapshot
→ FeatureLayer
→ AtomicSignal
→ DomainSignal / MarketRegime
→ StrategyRoute / StrategyParamSnapshot
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
→ ExecutionResult
→ ExchangeOrder
→ TradeFill
→ PositionState
→ Hermes 通知
→ 复盘与归因
```

---

## 4. 数据能力

### 4.1 能力目标

数据能力负责为 MarketSnapshot、FeatureLayer、AtomicSignal、回测、风控和实盘交易提供可信市场数据。

第一阶段以 Binance BTCUSDT U 本位合约为主。

数据能力至少包括：

```text
通过 Binance REST 获取历史 K 线
通过 Binance REST 采集最新已收盘 K 线
支持 4h 和 1d 周期
支持后续扩展资金费率、订单簿摘要、成交数据、交易规则等数据
为后续流程提供统一、可追溯、可校验的数据输入
```

缺失 K 线的修补职责由“数据回补能力”承担。

### 4.2 第一阶段优先级

```text
P0：Binance BTCUSDT U 本位 K 线 REST 采集
P0：历史 K 线采集
P0：4h + 1d 双周期 K 线存储
P0：只采集已收盘 K 线
P0：K 线收盘数据用于策略周期分析
P0：第一阶段策略仅依赖已收盘 K 线生成交易决策
P1：WebSocket 实时行情监测，仅用于后续风险或实时价格观察，不参与 K 线采集
P1：资金费率数据读取
P2：多品种数据采集
P2：多交易所数据采集
P2：深度订单簿完整采集
```

### 4.3 能力边界

数据能力不负责：

```text
修补缺失数据
生成交易信号
判断开仓或平仓
生成交易意图
执行订单
复盘归因
```

第一阶段策略决策只基于 K 线收盘后的数据。WebSocket 实时行情监测不作为第一阶段策略信号的必要输入。

---

## 5. 数据回补能力

### 5.1 能力目标

数据回补能力负责在发现 K 线缺失、断档或采集失败后，重新从可信数据源拉取缺失时间范围的数据，保证下游使用的数据完整可信。

数据回补能力不是人工修复数据，也不是手工改库。

### 5.2 第一阶段优先级

```text
P0：根据数据质量检查结果回补缺失 K 线
P0：支持按交易品种、时间周期、缺失起止时间回补
P0：回补任务必须幂等，重复执行不得产生重复 K 线
P0：回补结果必须重新经过数据质量检查
P0：可由数据质量结果生成 BackfillRequest
P1：人工触发指定区间回补
P1：周期性扫描历史缺口并回补
P2：多数据源交叉回补
```

### 5.3 能力边界

数据回补能力只负责补齐可信数据源中的缺失数据。

禁止：

```text
人工伪造 K 线
手工改写 K 线
用推测值补 K 线
绕过数据质量检查直接进入 MarketSnapshot
```

---

## 6. 数据质量能力

### 6.1 能力目标

数据质量能力负责判断数据是否可信，阻止不合格数据静默进入策略链路。

### 6.2 第一阶段优先级

```text
P0：K 线缺失检查
P0：K 线重复检查
P0：K 线时间连续性检查
P0：K 线收盘状态检查
P0：异常数据阻断并写入 DataQualityIssue / AlertEvent
P0：可回补数据质量问题创建 BackfillRequest
P0：MarketSnapshot 必须绑定允许下游继续的 DataQualityResult
P1：REST K 线与 WebSocket 实时行情状态的异常差异监测，不作为主 K 线质量放行依据
P2：多数据源交叉验证
```

### 6.3 能力边界

数据质量能力可以阻断下游流程。

数据质量能力不负责修正策略逻辑，不得人工伪造、人工修复或手工改写 K 线。

---

## 7. 市场快照能力

### 7.1 能力目标

MarketSnapshot 用于固定某一策略分析时刻的市场状态，并作为后续特征、原子信号、策略信号和 DecisionSnapshot 的共同证据基础。

### 7.2 第一阶段优先级

```text
P0：按策略周期生成 MarketSnapshot
P0：MarketSnapshot 绑定数据源、交易品种、时间周期和 trace_id
P0：MarketSnapshot 绑定 4h 与 1d 数据质量结果
P0：MarketSnapshot 可追溯到原始 K 线
P0：MarketSnapshot 不包含订单、账户、风控、执行字段
P1：快照包含资金费率、交易规则等扩展数据引用
P2：多品种组合快照
```

### 7.3 能力边界

MarketSnapshot 不负责计算策略信号，不负责读取账户，不负责下单。

---

## 8. 特征能力

### 8.1 能力目标

特征能力负责基于 MarketSnapshot 统一计算可复用的基础数值、指标、结构点和中间结果。

核心目标：

```text
防止同一指标在不同原子信号中重复计算或口径不一致
保证原子信号、回测和实盘使用同一套特征定义
为复盘提供可追溯的特征证据
```

示例：RSI、EMA、ATR、ADX、价格结构点、成交量均值、波动率、突破区间等。

### 8.2 第一阶段优先级

```text
P0：FeatureLayer 统一计算入口
P0：FeatureSet
P0：FeatureValue
P0：FeatureQualityCheck
P0：关键特征可追溯到 MarketSnapshot
P0：影响原子信号和交易决策的关键特征持久化到 MySQL
P0：同一策略周期内，同一特征不得重复计算
P0：FeatureLayer 支持单周期内存复用
P1：短期特征序列缓存到 Redis
P1：结构点识别
P1：feature registry（特征注册表）
P2：复杂形态特征
P2：多周期特征融合
```

### 8.3 能力边界

特征能力不负责：

```text
生成交易信号
判断开仓
判断平仓
生成交易意图
访问交易所交易接口
```

具体特征定义、参数、版本、缓存 key、批量计算方式，由后续 `docs/requirements/feature_layer.md` 和 architecture 文档定义。

---

## 9. 原子信号能力

### 9.1 能力目标

原子信号能力负责基于 FeatureValue 生成最小市场判断单元。

原子信号必须足够小、单一、可复盘、可验证。

示例：趋势、动量、波动、突破、背离、假突破等原子信号。

### 9.2 第一阶段优先级

```text
P0：原子信号统一结构
P0：原子信号引用 FeatureValue
P0：AtomicSignalQualityCheck
P0：原子信号可追溯到 MarketSnapshot 和 FeatureSet
P0：原子信号支持按单个信号独立拆分
P0：原子信号不得互相调用
P0：原子信号冲突、互斥、确认、否定关系由后续领域模块或策略层解释
P1：趋势类原子信号
P1：动量类原子信号
P1：波动类原子信号
P1：结构类原子信号
P2：复杂形态类原子信号
P2：跨周期原子信号
```

### 9.3 能力边界

原子信号不负责：

```text
生成最终策略信号
生成交易意图
执行交易
调用大模型
处理其他原子信号之间的关系
```

第一阶段不引入可被其他原子信号直接引用的中间事件信号。具体信号定义、输入特征、参数、算法版本、质量检查和复盘证据，由后续 `docs/requirements/atomic_signals.md` 定义。

---

## 10. 领域模块能力

### 10.1 能力目标

领域模块负责聚合同类原子信号，形成更高层级的领域判断。

示例领域：趋势、动量、波动、支撑压力、突破验证、假突破识别、资金费率、流动性等。

### 10.2 第一阶段优先级

```text
P0：保留领域模块层
P0：DomainQualityCheck
P0：保留 DomainSignal 与 DecisionSnapshot 的追溯边界
P1：趋势领域模块
P1：波动领域模块
P1：动量领域模块
P2：支撑压力领域模块
P2：假突破识别模块
P2：资金费率领域模块
```

### 10.3 能力边界

领域模块不负责直接生成交易意图、直接生成 DecisionSnapshot 或直接访问交易所。

领域模块输出必须经过质量检查后才能进入市场环境识别和策略路由。

---

## 11. 市场环境识别能力

### 11.1 能力目标

市场环境识别能力负责根据领域模块结果判断当前市场状态。

示例状态：趋势行情、震荡行情、高波动行情、低波动行情、突破环境、假突破风险环境、不适合交易环境。

### 11.2 第一阶段优先级

```text
P0：保留 MarketRegime 对象和边界
P1：简单趋势 / 震荡 / 高波动识别
P2：复杂市场环境分类
P2：多周期市场环境融合
```

### 11.3 能力边界

市场环境识别不直接下单，用于辅助策略路由和策略信号生成。

---

## 12. 策略路由能力

### 12.1 能力目标

策略路由负责根据市场环境、领域模块结果和配置选择适用策略。

第一阶段保留策略路由层，但不实现复杂多策略系统。

### 12.2 第一阶段优先级

```text
P0：保留 StrategyRoute 对象和接口边界
P0：默认趋势跟踪策略路由
P0：StrategyRoute 结果可追溯到 DecisionSnapshot
P0：路由必须支持配置化策略注册，不得用硬编码 if-else 固化策略映射
P1：简单 demo 路由
P2：多策略路由
P2：策略权重分配
P2：策略自动筛选
```

### 12.3 能力边界

策略路由不负责提交订单、访问交易所、绕过风控或自动优化策略。

---

## 13. 策略信号能力

### 13.1 能力目标

策略信号能力负责基于市场环境、策略路由、策略定义和可用原子信号，生成策略级市场判断。

在 005A 阶段，StrategySignal 只表达：

```text
direction
strength
confidence
evidence_text_zh
evidence_items
input_snapshot
routing_snapshot
aggregation_snapshot
conflict_snapshot
blocked / failed 状态
allows_downstream
```

StrategySignal 不表达：

```text
entry_price
stop_loss
take_profit
position_size
leverage
NO_TRADE
HOLD
订单 side
订单 quantity
reduce_only
```

入场价格、退出价格、止损、止盈、仓位、杠杆和订单参数，分别由后续 DecisionSnapshot、PriceSnapshot、Binance Account Sync、OrderPlan、RiskCheck、ExecutionPreparation 和 Execution 处理。

### 13.2 第一阶段优先级

```text
P0：StrategySignal 结构
P0：中低频趋势跟踪策略信号
P0：StrategySignal 可追溯到上游证据
P0：无有效参与信号时必须 fail-closed
P1：简单策略配置
P1：策略失效条件
P2：多策略信号合成
P2：策略自动优化
```

### 13.3 能力边界

StrategySignal 不是订单，不是仓位计划，不是风控结果。

StrategySignal 必须经过 StrategySignalQuality 后，才允许进入 DecisionSnapshot。

---

## 14. 策略信号质量能力

### 14.1 能力目标

策略信号质量能力负责检查 StrategySignal 是否结构完整、数值有效、证据可追溯、快照一致、时效可用，并阻止不合格策略信号进入 DecisionSnapshot。

### 14.2 第一阶段优先级

```text
P0：StrategySignalQualityResult
P0：StrategySignalQualityIssue
P0：质量结果必须绑定 StrategySignal
P0：blocked / failed / warning 均不得默认放行下游
P0：质量失败必须写入 AlertEvent
P1：更细粒度质量规则
P2：按策略类型配置不同质量规则
```

### 14.3 能力边界

策略信号质量能力不生成策略信号，不修改策略方向，不调整仓位，不生成订单参数。

---

## 15. 策略参数管理能力

### 15.1 能力目标

策略参数管理能力负责集中管理策略参数、参数版本和回测 / 实盘使用的参数快照。

示例参数：均线周期、RSI 阈值、ATR 止损倍数、突破窗口、仓位上限、策略启用状态等。

### 15.2 第一阶段优先级

```text
P0：策略参数必须有明确来源，不得散落硬编码在多个模块中
P0：DecisionSnapshot 和回测结果必须能追溯到策略参数版本或参数快照
P1：配置文件形式的策略参数管理
P1：参数版本标识
P2：数据库化策略参数管理
P2：参数回滚
P2：参数热更新
```

### 15.3 能力边界

策略参数管理不等于自动调参。

策略参数变更不得自动进入生产策略。生产策略参数变更必须经过人工确认、回测验证和文档记录。

---

## 16. DecisionSnapshot 能力

### 16.1 能力目标

DecisionSnapshot 记录系统每次策略分析周期的操盘判断快照，用于回答系统当时看到了什么、判断了什么、目标仓位是什么、为什么允许或不允许继续下游、后续是否正确。

DecisionSnapshot 是目标仓位判断，不是订单意图，不是风控结果，不是执行请求。

### 16.2 第一阶段优先级

```text
P0：每个策略分析周期生成 DecisionSnapshot
P0：DecisionSnapshot 追溯 MarketSnapshot、FeatureSet、AtomicSignal、StrategySignal 和 StrategySignalQualityResult
P0：DecisionSnapshot 输出 target_intent
P0：DecisionSnapshot 输出 target_position_ratio
P0：DecisionSnapshot 输出 target_confidence
P0：DecisionSnapshot 输出 target_reason_code
P0：DecisionSnapshot 输出 target_schema_version
P0：DecisionSnapshot 进入复盘系统
P0：DecisionSnapshot 保留领域模块、市场环境、策略路由的扩展追溯边界
P1：DecisionSnapshot 与 TradeLifecycle 关联
P2：更复杂的目标仓位表达
```

### 16.3 能力边界

DecisionSnapshot 不直接调用交易所，不读取账户，不读取仓位，不读取实时价格，不生成订单 side、quantity、reduce_only。

DecisionSnapshot 之后的交易链路必须经过：

```text
DecisionSnapshot
→ Binance Account Sync / PriceSnapshot
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation
→ Execution
```

风控或执行准备拒绝时，必须记录拒绝结果，不得静默丢弃。

---

## 17. 回测能力

### 17.1 能力目标

回测能力用于验证策略是否具有真实市场逻辑，以及回测结果是否可信。

回测必须尽量复用实盘链路中的核心逻辑。

### 17.2 第一阶段优先级

```text
P0：基础历史回测
P0：手续费
P0：slippage（滑点）
P0：K 线收盘后生成信号
P0：避免 look-ahead bias（前视偏差 / 未来函数）
P0：回测结果可追溯到 DecisionSnapshot
P0：第一阶段回测不依赖 WebSocket 实时流
P0：第一阶段如不建模冲击成本，必须明确假设无显著冲击成本
P1：资金费率
P1：成交约束
P1：最小下单量
P1：杠杆和保证金风险
P2：冲击成本模型
P2：参数网格测试
P2：walk-forward analysis（滚动样本外测试）
P2：多策略组合回测
```

### 17.3 能力边界

回测不得使用实盘不可获得的数据，不得为了提高收益率而引入不可执行假设。

如果后续实盘策略依赖实时流数据，则必须扩展回测模拟能力后才能进入真实交易。

---

## 18. Binance Account Sync 能力

### 18.1 能力目标

Binance Account Sync 能力负责在交易前统一读取和记录交易所账户、余额、仓位和交易规则事实，为 PriceSnapshot、OrderPlan、RiskCheck 和复盘提供可信输入。

Binance Account Sync 不负责订单状态同步，不负责成交状态同步，不生成或处理 ExchangeOrder / TradeFill / ExecutionResult。订单状态与成交状态属于后续 Execution / ExchangeOrder / TradeFill / Tracking 边界。

第一阶段只允许选择一个 active account domain / market_type。第一阶段默认是 Binance BTCUSDT U 本位合约。

### 18.2 第一阶段优先级

```text
P0：读取所选 account domain 的账户事实
P0：读取余额事实
P0：读取真实仓位事实
P0：读取 BTCUSDT U 本位交易规则事实
P0：记录 trace_id、读取时间、交易所来源、market_type、symbol 和错误状态
P0：交易所实际杠杆必须提供给 RiskCheck 二次核验
P0：账户同步失败时不得默认放行真实交易
P1：订单执行后刷新账户事实
P1：账户状态心跳检查
P2：账户权益曲线统计
P2：多账户管理
P2：U 本位与币本位同时监控但隔离消费
```

### 18.3 能力边界

Binance Account Sync 只提供事实输入，不生成策略信号，不生成目标仓位，不生成 CandidateOrderIntent，不提交订单。

未被选中的 account domain 不得参与 OrderPlan、RiskCheck 或 Execution。

---

## 19. PriceSnapshot 能力

### 19.1 能力目标

PriceSnapshot 能力负责在交易前固定下游 OrderPlan / RiskCheck / ExecutionPreparation 使用的价格事实，避免不同模块各自读取价格导致口径不一致。

PriceSnapshot 是交易前价格事实，不是策略信号，不是 MarketSnapshot 的替代品。

### 19.2 第一阶段优先级

```text
P0：为 OrderPlan / RiskCheck 提供统一价格事实
P0：绑定 exchange、market_type、symbol、price_source、as_of_utc 和 trace_id
P0：价格过期或缺失时阻断下游
P0：价格异常必须写 AlertEvent
P1：WebSocket 最新价参与风险保护
P1：价格偏离检查
P2：多价格源交叉验证
```

### 19.3 能力边界

PriceSnapshot 不生成策略信号，不修改 DecisionSnapshot，不生成订单，不提交订单。

MarketSnapshot 用于策略分析周期的市场证据；PriceSnapshot 用于交易前价格事实，两者不得混用。

---

## 20. OrderPlan 能力

### 20.1 能力目标

OrderPlan 负责把 DecisionSnapshot 的目标仓位判断，结合 Binance Account Sync 和 PriceSnapshot 的事实输入，转换为可供风控审批的 CandidateOrderIntent。

OrderPlan 是目标仓位到候选订单意图的唯一转换模块。

### 20.2 第一阶段优先级

```text
P0：读取有效 DecisionSnapshot
P0：读取同一 trace_id 或同一交易周期内有效的 Account / Balance / Position / SymbolRule 事实
P0：读取有效 PriceSnapshot
P0：计算 current_position 与 target_position 的差值
P0：生成 CandidateOrderIntent
P0：区分 reduce_risk 与 increase_risk 组件
P0：净额反手场景支持 primary_intent 与 fallback_reduce_only_intent
P0：OrderPlan 结果可追溯
P0：OrderPlan 不直接放行交易
P1：更多订单类型规划
P2：多策略合并订单计划
```

### 20.3 能力边界

OrderPlan 不访问交易所，不提交订单，不绕过 RiskCheck，不自行判断风控是否放行。

OrderPlan 可以生成候选意图，但只有 RiskCheck 放行后才可能形成 ApprovedOrderIntent。

---

## 21. CandidateOrderIntent / ApprovedOrderIntent 能力

### 21.1 能力目标

CandidateOrderIntent 记录 OrderPlan 生成的候选订单意图。

ApprovedOrderIntent 记录 RiskCheck 放行后的可执行订单意图。

两者必须分离，避免未通过风控的候选订单被误执行。

### 21.2 第一阶段优先级

```text
P0：CandidateOrderIntent 可追溯到 DecisionSnapshot、OrderPlan、Account Sync 和 PriceSnapshot
P0：CandidateOrderIntent 记录 side、quantity、reduce_only、order_type、time_in_force 等订单意图字段
P0：CandidateOrderIntent 记录 order_components
P0：ApprovedOrderIntent 只能由 RiskCheck ALLOW 产生
P0：ApprovedOrderIntent 可追溯到 RiskCheckResult
P0：未批准的 CandidateOrderIntent 不得进入 ExecutionPreparation
P1：多候选意图排序
P2：复杂订单意图组合
```

### 21.3 能力边界

CandidateOrderIntent 不是交易所订单，不得直接执行。

ApprovedOrderIntent 仍不是交易所订单，必须经过 ExecutionPreparation 后才能进入 Execution。

---

## 22. RiskCheck 能力

### 22.1 能力目标

RiskCheck 是交易执行前的强制闸门。

RiskCheck 只审批 CandidateOrderIntent，不直接消费 DecisionSnapshot 生成订单，不自行造单，不自行缩单。

### 22.2 第一阶段优先级

```text
P0：真实交易开关检查
P0：账户余额检查
P0：仓位冲突检查
P0：单笔风险检查
P0：交易品种检查
P0：杠杆配置二次核验
P0：DecisionSnapshot / MarketSnapshot / FeatureSet / StrategySignal / StrategySignalQualityResult 可追溯性校验
P0：CandidateOrderIntent 结构校验
P0：reduce_risk / increase_risk 组件校验
P0：RiskCheckResult 记录
P0：ALLOW / DENY / BLOCKED / FAILED 状态
P0：风控拒绝原因可复盘
P1：当日最大亏损限制
P1：连续亏损限制
P1：冷却期
P1：订单价格偏离检查
P2：组合级风险控制
P2：动态风险预算
```

### 22.3 能力边界

风控不负责生成策略信号，不负责提交订单。

RiskCheck P0 不做任意 MODIFY，不临时缩单，不拆单，不自行生成 fallback。

净额反手场景中，RiskCheck 只允许在 OrderPlan 预生成的 primary_intent 和 fallback_reduce_only_intent 之间做审批选择。

任何时候都不允许系统自动调整杠杆。

---

## 23. ExecutionPreparation 能力

### 23.1 能力目标

ExecutionPreparation 负责在真实提交订单前，对 ApprovedOrderIntent 做最终执行准备和交易规则校验，生成执行请求所需的安全参数。

ExecutionPreparation 是执行前准备层，不是真实下单层。

### 23.2 第一阶段优先级

```text
P0：只接收 ApprovedOrderIntent
P0：校验交易品种、market_type、side、quantity、reduce_only、order_type
P0：校验最小下单量、价格精度、数量精度
P0：校验执行模式 dry-run / paper / real
P0：生成 ExecutionPreparationResult / PreparedExecutionRequest
P0：交易规则读取失败时不得提交真实订单
P1：订单参数格式化能力
P1：交易规则缓存与定期刷新
P2：复杂订单类型执行准备
```

### 23.3 能力边界

ExecutionPreparation 不生成 CandidateOrderIntent，不绕过 RiskCheck，不真实下单，不撤单。

---

## 24. Execution 能力

### 24.1 能力目标

Execution 负责通过统一执行模块处理订单相关行为。

Execution 是唯一允许提交真实订单、撤单、查询真实订单状态的模块。

### 24.2 第一阶段优先级

```text
P0：dry-run 执行
P0：paper trading 或模拟执行
P0：dry-run / paper trading / real trading 三种模式状态隔离
P0：只处理通过 ExecutionPreparation 的执行请求
P0：ExecutionResult 记录
P0：执行后触发订单状态和仓位状态同步
P1：真实交易接口边界
P1：真实下单开关
P1：真实撤单开关
P1：订单状态查询
P2：复杂订单类型
P2：高级执行算法
```

### 24.3 能力边界

Execution 不负责生成策略信号，不得绕过 RiskCheck，不得绕过 ExecutionPreparation，不得自动调整杠杆。

paper trading 状态不得参与 real trading 的仓位规划、风控或仓位判断。执行模式切换必须人工确认。

---

## 25. 订单与成交能力

### 25.1 能力目标

订单与成交能力负责记录订单生命周期和真实成交结果。

核心对象：

```text
CandidateOrderIntent
ApprovedOrderIntent
ExecutionPreparationResult
PreparedExecutionRequest
ExecutionResult
ExchangeOrder
TradeFill
```

### 25.2 第一阶段优先级

```text
P0：CandidateOrderIntent 记录
P0：ApprovedOrderIntent 记录
P0：ExecutionResult 记录
P0：ExchangeOrder 记录
P0：TradeFill 记录
P0：订单、成交可追溯到 DecisionSnapshot
P0：执行后主动查询订单状态
P1：部分成交处理
P1：撤单记录
P1：订单状态同步
P2：复杂订单状态机
```

### 25.3 能力边界

订单记录不得替代 DecisionSnapshot，成交记录不得替代仓位同步结果。

---

## 26. 仓位同步与仓位记录能力

### 26.1 能力目标

仓位同步与仓位记录能力负责记录和同步当前仓位状态。

仓位状态用于 OrderPlan、RiskCheck、复盘和后续交易判断。

### 26.2 第一阶段优先级

```text
P0：PositionState 或等价仓位状态记录
P0：仓位可追溯到 TradeFill 或交易所同步记录
P0：支持读取真实仓位
P0：读取真实仓位必须通过 Binance Account Sync 和配置开关
P0：每次订单执行后主动查询仓位状态
P0：real trading 仓位不得与 paper trading 仓位混用
P1：仓位心跳检查定时任务
P1：仓位与 TradeLifecycle 关联
P1：仓位异常告警
P1：交易所仓位与系统仓位不一致时阻断后续真实交易并告警
P2：组合仓位管理
```

### 26.3 能力边界

仓位模块不负责生成策略信号，不得自动调整杠杆。

---

## 27. Hermes 通知能力

### 27.1 能力目标

Hermes 是系统通知出口，用于向用户微信发送系统状态、交易状态、异常和复盘结果。

### 27.2 第一阶段优先级

```text
P0：统一 notifications 模块
P0：Hermes dry-run / 真实发送开关
P0：系统异常通知
P0：数据异常通知
P0：风控拒绝通知
P0：订单状态通知
P0：成交通知
P0：仓位变化通知
P1：复盘摘要通知
P2：复杂日报 / 周报
```

### 27.3 能力边界

Hermes 不参与交易决策，不触发下单，不生成 CandidateOrderIntent 或 ApprovedOrderIntent。

---

## 28. 复盘能力

### 28.1 能力目标

复盘能力负责根据 trace_id、DecisionSnapshot ID 或 TradeLifecycle ID 聚合全链路证据，解释交易或不交易的原因、风控结果、执行结果、仓位变化和后续表现。

### 28.2 第一阶段优先级

```text
P0：DecisionSnapshot 基础复盘
P0：TradeLifecycle 基础复盘
P0：候选意图 / 批准意图 / 订单 / 成交 / 仓位追溯
P0：风控拒绝复盘
P1：亏损归因
P1：错过行情归因
P1：执行异常归因
P2：自动生成复盘报告
P2：策略表现统计看板
```

### 28.3 能力边界

复盘系统不自动修改生产策略。复盘结论只能作为后续人工研究和改进依据。

---

## 29. 策略评估与进化能力

### 29.1 能力目标

策略评估与进化能力负责长期跟踪策略、原子信号、领域模块和策略路由的表现，为人工优化提供依据。

### 29.2 第一阶段优先级

```text
P1：记录策略表现所需的追溯数据
P1：按 TradeLifecycle 统计基础表现
P2：按策略统计胜率、盈亏比、最大回撤、profit factor（盈亏因子）
P2：按原子信号统计后验表现
P2：按市场环境统计策略表现
P2：生成降权、禁用或优化建议
```

### 29.3 能力边界

策略评估不得自动修改生产策略、自动禁用策略或自动调整参数。

所有优化建议必须经过人工确认、回测验证和文档记录。

---

## 30. 事件日历能力

### 30.1 能力目标

事件日历能力用于记录可能影响市场波动和流动性的外部事件。

示例：CPI、非农就业、FOMC 利率决议、重大交易所维护、重大政策事件、自定义风险事件。

### 30.2 第一阶段优先级

```text
P2：事件日历配置
P2：事件前后风险提示
P2：事件窗口内自动收紧风控或暂停交易的能力边界
```

### 30.3 能力边界

事件日历第一阶段不进入交易主链路。

如后续事件日历影响风控，必须通过 RiskCheck 显式处理。

---

## 31. 大模型复盘辅助能力

### 31.1 能力目标

大模型仅用于事后复盘、解释、归因和策略研究辅助。

### 31.2 第一阶段优先级

```text
P1：交易结束后复盘总结
P1：异常日志解释
P1：亏损原因归因辅助
P2：周期性策略表现总结
P2：优化建议草案
```

### 31.3 禁止能力

```text
禁止大模型实时判断开仓
禁止大模型实时判断平仓
禁止大模型决定仓位
禁止大模型修改止损止盈
禁止大模型生成 CandidateOrderIntent
禁止大模型生成 ApprovedOrderIntent
禁止大模型影响 RiskCheck
禁止大模型自动修改生产策略
```

---

## 32. 审计与追踪能力

### 32.1 能力目标

审计与追踪能力负责保证每次系统行为可追溯、可解释、可排查。

### 32.2 第一阶段优先级

```text
P0：trace_id
P0：trigger_source
P0：运行状态记录
P0：错误码和错误消息
P0：核心对象之间的追溯关系
P1：审计查询工具
P2：可视化审计链路
```

### 32.3 能力边界

任何关键链路不得静默失败，任何交易相关行为必须可追溯。

---

## 33. 调度与幂等能力

### 33.1 能力目标

调度与幂等能力负责定时运行系统主链路，并避免重复执行造成重复数据、重复信号或重复下单。

### 33.2 第一阶段优先级

```text
P0：Celery Beat 周期调度
P0：Celery task 任务入口
P0：Redis 锁或数据库幂等兜底
P0：定义业务幂等键，避免同一分析周期重复生成 DecisionSnapshot / CandidateOrderIntent / ApprovedOrderIntent
P0：重复运行不重复下单
P0：任务失败记录
P1：失败重试限制
P1：人工恢复入口
P2：复杂任务编排
```

### 33.3 能力边界

Celery task 只作为任务入口，不写复杂业务逻辑。

trace_id 用于追踪一次运行，不适合作为唯一业务幂等键。

---

## 34. 监控与异常恢复能力

### 34.1 能力目标

监控与异常恢复能力负责发现系统异常、数据异常、交易异常，并支持恢复流程。

### 34.2 第一阶段优先级

```text
P0：任务失败告警
P0：数据异常告警
P0：风控异常告警
P0：交易执行异常告警
P1：订单状态异常恢复
P1：仓位不同步告警
P1：人工恢复命令入口
P1：强制同步订单状态
P1：强制同步仓位状态
P1：重放失败任务
P1：解除人工确认后的熔断或阻断状态
P2：自动恢复策略
P2：可视化监控面板
```

### 34.3 能力边界

异常恢复分为基础设施级恢复和业务级恢复。

基础设施级恢复可以自动化，但必须有最大重试次数、错误记录和告警。

业务级恢复必须人工确认，不得绕过风控、不得自动修改杠杆、不得自动修改生产策略。

---

## 35. 配置与安全能力

### 35.1 能力目标

配置与安全能力负责管理环境变量、危险开关、密钥和生产风险边界。

### 35.2 第一阶段优先级

```text
P0：.env.example 中文注释
P0：真实交易默认关闭
P0：真实 Hermes 发送默认关闭
P0：真实大模型调用默认关闭
P0：账户读取开关
P0：仓位读取开关
P0：订单读取开关
P0：真实下单开关
P0：真实撤单开关
P0：杠杆来自 env 或明确配置
P1：敏感信息脱敏
P1：配置检查命令
P2：配置管理后台
```

### 35.3 禁止能力

```text
禁止提交真实 .env
禁止提交真实 API Key
禁止日志打印密钥
禁止异常信息暴露密钥
禁止系统自动调整杠杆
```

---

## 36. 关键依赖摘要

系统能力必须按依赖顺序推进。

关键依赖：

```text
没有可信数据，就不能生成 MarketSnapshot。
没有 DataQuality PASS，就不能生成可用 MarketSnapshot。
没有 MarketSnapshot，就不能生成 FeatureSet。
没有 FeatureSet，就不能生成可信 AtomicSignal。
没有 AtomicSignal，就不能形成可信 StrategySignal。
没有 StrategySignalQuality PASS，就不能生成可用 DecisionSnapshot。
没有 DecisionSnapshot，就不能进入 OrderPlan。
没有 Binance Account Sync 和 PriceSnapshot，就不能生成 CandidateOrderIntent。
没有 CandidateOrderIntent，就不能进入 RiskCheck。
没有 RiskCheck ALLOW，就不能生成 ApprovedOrderIntent。
没有 ApprovedOrderIntent，就不能进入 ExecutionPreparation。
没有 ExecutionPreparation 通过，就不能进入 Execution。
没有 ExchangeOrder / TradeFill，就不能更新或校验仓位事实。
没有 DecisionSnapshot / TradeLifecycle，就不能做可信复盘。
```

完整数据流由 `docs/architecture/data_flow.md` 定义。

---

## 37. 明确禁止能力总表

以下能力在当前项目中明确禁止：

```text
大模型实时交易决策
大模型生成 CandidateOrderIntent
大模型生成 ApprovedOrderIntent
Hermes 触发交易
策略模块直接下单
原子信号直接下单
特征层生成交易信号
DecisionSnapshot 生成订单 side / quantity / reduce_only
绕过 RiskCheck 下单
绕过 ExecutionPreparation 下单
绕过 Execution 下单
RiskCheck 任意 MODIFY 候选订单
RiskCheck 临时缩单、拆单或自行生成 fallback
系统自动调整杠杆
复盘结果自动修改生产策略
策略评估自动禁用生产策略
策略参数自动热更新到生产环境
测试自动提交真实订单
测试自动撤销真实订单
未开启开关访问真实交易接口
paper trading 仓位参与 real trading 风控
未通过交易规则校验提交真实订单
```
