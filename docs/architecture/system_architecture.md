# system_architecture.md

# 系统架构说明

## 1. 文档目的

本文档定义中低频趋势跟踪自动交易系统的工程架构约束。

本文档用于回答：

```text
系统整体如何分层？
Django app / service / domain / task / command 应该如何分工？
同步链路和异步链路如何区分？
MySQL、Redis、Celery、Binance gateway、PriceSnapshot、Hermes、大模型各自处于什么位置？
DecisionSnapshot、OrderPlan、RiskCheck、ExecutionPreparation、Execution、Tracking 如何分工？
trace_id、trigger_source、幂等键如何贯穿系统？
异常恢复如何分层？
回测和实盘如何复用核心逻辑？
```

本文档不定义：

```text
具体策略公式
具体特征算法
具体原子信号算法
具体数据库字段
具体 Django model 字段
具体 Celery task 名称
具体函数名
具体测试命令
```

模块职责、输入输出、允许调用和禁止调用，以：

```text
docs/architecture/01_module_map.md
```

为准。

能力优先级和 P0 / P1 / P2 范围，以：

```text
docs/requirements/02_system_capabilities.md
```

为准。

项目红线和不变量，以：

```text
docs/rules/project_invariants.md
```

为准。

---

## 2. 架构总原则

系统架构必须优先保证：

```text
数据可信
链路可追溯
风控不可绕过
订单生成与真实执行分离
交易执行受控
回测与实盘逻辑一致
异常可恢复
复盘可解释
```

核心原则：

```text
1. 上游模块不得反向依赖下游模块。
2. 策略模块不得直接访问交易所接口。
3. DecisionSnapshot 不得读取账户 / 持仓 / BinanceSyncRun，不得生成交易所订单动作。
4. OrderPlan 是唯一将目标仓位转换为 CandidateOrderIntent 的模块。
5. RiskCheck 只检查 CandidateOrderIntent，不直接检查 DecisionSnapshot 的动作字段。
6. ApprovedOrderIntent 是风控通过后的订单意图，不是真实交易所订单。
7. ExecutionPreparation 负责执行前最终检查和 price guard。
8. Execution 是唯一允许真实下单的入口。
9. Tracking 负责订单、成交和仓位追踪，不反向影响策略判断。
10. MySQL 是核心业务主存储。
11. Redis 只做缓存、锁、短期状态和 Celery 基础设施，不做核心数据唯一存储。
12. Celery task 只作为任务入口，不写复杂业务逻辑。
13. Management command 只作为人工入口，不写复杂业务逻辑。
14. 外部服务访问必须通过 gateway / client 层统一封装。
15. 所有关键链路必须有 trace_id、trigger_source、状态和错误记录。
16. 所有正式订单相关事件必须 AlertEvent。
```

---

## 3. 系统业务分层

系统按业务责任划分为以下层级：

```text
数据层
→ 特征层
→ 信号层
→ 策略层
→ 决策层
→ 账户事实 / 价格事实 / 订单规划 / 风控层
→ 执行准备 / 执行 / 跟踪层
→ 通知、复盘与辅助层
```

### 3.1 数据层

包括：

```text
数据采集
数据质量
数据回补
MarketSnapshot
```

职责：

```text
提供可信市场数据
阻断不合格数据
修复缺失数据
生成可复盘的市场快照
```

数据层不得生成交易信号，不得访问交易所交易接口。

### 3.2 特征层

包括：

```text
FeatureLayer
FeatureSet
FeatureValue
FeatureQualityCheck
```

职责：

```text
统一计算可复用特征
统一特征口径
提供原子信号证据
保证特征可追溯和可复算
```

特征层不得生成交易信号，不得判断开仓、平仓、加仓、减仓。

### 3.3 信号层

包括：

```text
AtomicSignal
AtomicSignalQualityCheck
DomainSignal
DomainQualityCheck
MarketRegime
```

职责：

```text
生成最小市场判断
聚合同类信号
识别市场环境
为策略层提供结构化证据
```

信号层不得直接下单，不得生成 CandidateOrderIntent、ApprovedOrderIntent 或交易所订单。

### 3.4 策略层

包括：

```text
StrategyRoute
StrategyParamSnapshot
StrategySignal
```

职责：

```text
选择策略
固定参数快照
生成策略倾向、强度、置信度和证据摘要
```

策略层不得直接访问交易所，不得绕过风控，不得生成交易所订单。

### 3.5 决策层

包括：

```text
DecisionSnapshot
DecisionPolicy
```

职责：

```text
将 StrategySignal 转换为目标仓位意图
记录每个分析周期的目标仓位决策快照
保留证据链、策略版本、参数版本和 trace_id
```

DecisionSnapshot 输出目标仓位语义，例如：

```text
target_intent
target_position_ratio
```


DecisionSnapshot 不读取账户、余额、持仓、BinanceSyncRun 或 PriceSnapshot。

### 3.6 账户事实 / 价格事实 / 订单规划 / 风控层

包括：

```text
Binance Account Sync
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheck
```

职责：

```text
Binance Account Sync：只读同步账户、余额、持仓、symbol rule 和交易环境事实。
PriceSnapshot：固化某一时刻用于估值、风控和执行保护的价格事实。
OrderPlan：将 DecisionSnapshot 的目标仓位与当前账户 / 持仓 / 价格事实转换为 CandidateOrderIntent。
CandidateOrderIntent：表达待风控的候选订单意图。
RiskCheck：按 order_components 和账户事实检查候选订单是否允许进入执行准备。
```

OrderPlan 不做最终风控审批，不真实下单。

RiskCheck 不生成目标仓位，不真实下单，不调用 Binance 修改交易环境。

### 3.7 执行准备 / 执行 / 跟踪层

包括：

```text
ApprovedOrderIntent
ExecutionPreparation
Execution
ExchangeOrder
TradeFill
PositionState
Tracking
```

职责：

```text
ApprovedOrderIntent：风控通过后的订单意图。
ExecutionPreparation：真实下单前最终检查，包括 price guard、幂等、开关、过期校验。
Execution：唯一真实下单入口。
Tracking：订单、成交、部分成交、失败、拒单、仓位变化追踪。
```

ExecutionPreparation 不生成策略信号，不修改风控结论。

Execution 是唯一允许调用 trading gateway 发起真实交易类请求的模块。

Tracking 不得反向修改 DecisionSnapshot、OrderPlan 或 RiskCheck 事实。

### 3.8 通知、复盘与辅助层

包括：

```text
AlertEvent
HermesNotification
ReviewRecord
ModelReviewRecord
策略评估
事件日历
监控与恢复
```

职责：

```text
发送通知
聚合证据链
辅助复盘解释
长期统计策略表现
发现异常并支持恢复
```

通知、复盘和大模型不得反向触发实时交易。

---

## 4. 工程分层

系统代码应按工程职责分层。

推荐分层：

```text
entrypoint 层
application service 层
domain 层
repository / selector 层
gateway / client 层
model 层
```

### 4.1 entrypoint 层

包括：

```text
Celery task
Management command
Django admin action
API view（如后续存在）
```

职责：

```text
解析参数
生成或传递 trace_id
设置 trigger_source
调用 application service
输出执行结果
```

禁止：

```text
在 task 中写复杂业务逻辑
在 command 中写复杂业务逻辑
在 admin action 中绕过风控
在 view 中直接访问交易所或直接写核心业务状态
```

### 4.2 application service 层

职责：

```text
编排一个完整业务用例
组织多个 domain service
控制事务边界
调用 repository / gateway
处理幂等、状态流转和错误记录
```

示例：

```text
构建 MarketSnapshot
运行一次策略分析周期
生成 DecisionSnapshot
同步 Binance 账户事实
派生 PriceSnapshot
生成 OrderPlan
运行 RiskCheck
准备执行 ApprovedOrderIntent
执行一次真实 / 模拟订单
同步订单状态
聚合复盘证据
```

application service 可以调用多个 domain service，但不得把所有业务逻辑塞成一个超大 service。

### 4.2.1 RunAnalysisCycleService

RunAnalysisCycleService 是 4h 分析周期的主编排服务。

它负责按顺序确认并调用：

```text
data_collection
→ data_quality
→ 如发现可回补问题，则创建 BackfillRequest 并阻断当前分析流程
→ data_backfill 扫描 / 领取 pending BackfillRequest，claim 后创建 BackfillRun 并执行回补
→ BackfillRun 完成后，标记 / 要求 DataQuality 复检
→ 后续统一编排层、recovery scan 或人工命令重新执行 DataQuality
→ 新的 DataQualityResult = PASS 且 allows_downstream=True
→ MarketSnapshot
→ FeatureLayer
→ AtomicSignal
→ DomainSignal
→ MarketRegime
→ StrategySignal
→ DecisionSnapshot
```

RunAnalysisCycleService 的产物到 DecisionSnapshot 为止。

交易链路可以由同一上层 orchestration 继续触发，但必须显式进入：

```text
Binance Account Sync
→ PriceSnapshot
→ OrderPlan
→ RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation
→ Execution
→ Tracking
```

RunAnalysisCycleService 必须保证：

```text
不得跳过数据采集。
不得跳过 data_quality。
不得在 data_quality 非 PASS 时生成 MarketSnapshot。
不得由策略层懒加载生成 MarketSnapshot。
不得在 MarketSnapshot 不存在或非 CREATED 时进入 FeatureLayer。
不得在上游失败时继续执行策略、决策或交易链路。
```

正常调度、手动触发、失败重试、服务器重启恢复，都必须通过 RunAnalysisCycleService 或等价编排服务按顺序恢复。

如果服务器宕机后重启，RunAnalysisCycleService 必须从缺失的最早安全步骤开始恢复：

```text
缺 K 线 → 先采集或回补
data_quality 未 PASS → 先质量检查或复检
MarketSnapshot 不存在 → 先生成或复用 MarketSnapshot
MarketSnapshot 已存在但下游未完成 → 从快照后的下一步继续
```

Celery task、management command 和 scheduler 只能作为入口调用 RunAnalysisCycleService，不得在入口层直接串联完整业务逻辑。

### 4.2.2 TradeOrchestrationService

TradeOrchestrationService 是 DecisionSnapshot 之后的受控交易链路编排服务。

它负责按顺序确认并调用：

```text
DecisionSnapshot
→ BinanceSyncRun 显式选择 / 校验
→ PriceSnapshot 生成 / 校验
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation
→ Execution
→ Tracking
```

TradeOrchestrationService 必须保证：

```text
不得让 RiskCheck 直接消费 DecisionSnapshot 的动作字段或 ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE 枚举。
不得绕过 OrderPlan 生成 CandidateOrderIntent。
不得绕过 RiskCheck 生成 ApprovedOrderIntent。
不得绕过 ExecutionPreparation 直接下单。
不得在 PriceSnapshot 过期时继续估值 / 风控 / 执行。
不得在 active order 存在时重复生成有效交易链路。
```

### 4.3 domain 层

职责：

```text
承载纯业务判断
封装策略、风控、特征、信号、订单规划、执行准备等核心规则
尽量不依赖 Django ORM 细节
尽量不直接访问外部服务
```

domain 层适合放：

```text
特征计算规则
原子信号判断规则
领域聚合规则
市场环境识别规则
DecisionPolicy
OrderPlan 计算规则
RiskCheck rule plugin
ExecutionPreparation price guard 规则
订单状态归一化规则
```

### 4.4 repository / selector 层

职责：

```text
封装数据库读写
提供稳定查询接口
隔离 ORM 细节
避免业务层散落复杂 query
```

规则：

```text
repository 负责写入
selector 负责查询
核心业务对象的创建、状态更新必须通过明确 service / repository
不得在任意模块随意操作其他模块内部数据
```

示例 selector：

```text
Binance Account Sync selector
PriceSnapshot selector
OrderPlan selector
RiskCheck selector
Execution selector
Tracking selector
```

### 4.5 gateway / client 层

职责：

```text
封装外部服务访问
统一超时、重试、错误码、限频、日志和密钥脱敏
```

包括：

```text
Binance market data gateway
Binance account read-only gateway
Binance trading gateway
Binance user data stream client（P1/P2）
PriceFeed client（P1/P2）
Hermes client
LLM client
```

业务模块不得直接拼接外部 API 请求。

### 4.6 model 层

Django model 只负责：

```text
数据结构
字段约束
索引约束
最小数据校验
中文说明
```

禁止在 model 中堆叠：

```text
策略逻辑
交易逻辑
风控逻辑
外部请求
Hermes 发送
大模型调用
复杂状态机
```

---

## 5. Django app 划分原则

Django app 应按业务边界划分，而不是按技术层随意拆分。

建议 app 方向：

```text
market_data
features
signals
strategy
decision_snapshot
binance_account_sync
price_snapshot
order_plan
risk_check
approved_order_intent（后续可确定命名）
execution_preparation
execution
tracking
notifications
review
scheduler 或 operations
common
```

说明：

```text
market_data：K 线、数据源、数据质量、回补、MarketSnapshot
features：FeatureSet、FeatureValue、FeatureQualityCheck
signals：AtomicSignal、DomainSignal、MarketRegime
strategy：StrategyRoute、StrategyParamSnapshot、StrategySignal
decision_snapshot：DecisionPolicy、DecisionSnapshot
binance_account_sync：BinanceSyncRun、账户、余额、持仓、symbol rule 只读快照
price_snapshot：PriceSnapshot、价格事实派生、TTL、hash
order_plan：OrderPlan、CandidateOrderIntent、order_components、fallback_reduce_only_intent
risk_check：RiskRuleDefinition、RiskCheckResult、RiskRuleResult、RiskCheckIssue
approved_order_intent：风控通过后的订单意图，具体编号和 app 命名后续确定
execution_preparation：执行前 price guard、幂等、开关和过期校验
execution：真实 / paper / dry-run 执行，ExchangeOrder 初始写入
tracking：订单状态、TradeFill、PositionState、User Data Stream / REST 查询补偿
notifications：AlertEvent、HermesNotification、通知发送记录
review：ReviewRecord、ModelReviewRecord、策略评估
scheduler / operations：TaskRunRecord、IdempotencyRecord、RecoveryRecord
common：trace_id、错误码、枚举、时间工具、Decimal 格式化等
```

实际 app 名称可在后续 plans 中确定，但必须遵守：

```text
不得让一个 app 承担多个架构层职责。
不得为了方便把所有 model 塞进一个 app。
不得让 strategy app 直接依赖 execution gateway。
不得让 notifications app 反向触发交易。
OrderPlan / RiskCheck 不得直接依赖 WebSocket。
Execution 是唯一真实下单入口。
```

---

## 6. 主业务链路架构

第一阶段主业务链路：

```text
数据采集
→ data_quality
→ 如发现可回补问题，则创建 BackfillRequest 并阻断当前分析流程
→ data_backfill 扫描 / 领取 pending BackfillRequest，claim 后创建 BackfillRun 并执行回补
→ BackfillRun 完成后，标记 / 要求 DataQuality 复检
→ 后续统一编排层、recovery scan 或人工命令重新执行 DataQuality
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
→ ExchangeOrder / TradeFill / PositionState / Tracking
→ AlertEvent / Hermes 通知
→ 复盘与归因
```

链路语义：

```text
DecisionSnapshot 是目标仓位快照，不是订单动作。
Binance Account Sync 是只读账户事实来源。
PriceSnapshot 是价格事实来源，不是最终成交价。
OrderPlan 把目标仓位转换成 CandidateOrderIntent。
CandidateOrderIntent 是待风控订单意图，不可执行。
RiskCheck 是 CandidateOrderIntent 进入执行准备前的强制闸门。
ApprovedOrderIntent 是风控通过订单意图，不是交易所订单。
ExecutionPreparation 是真实下单前最终检查层。
Execution 是唯一真实下单入口。
ExchangeOrder 是交易所订单状态记录。
TradeFill 是成交记录。
PositionState 是仓位状态。
Tracking 负责后续订单、成交、仓位状态同步。
ReviewRecord 是复盘记录。
```

该主链路由 RunAnalysisCycleService、TradeOrchestrationService 或等价 application service 编排。

Celery task、management command、scheduler 只负责触发 service，不得直接承载完整主链路逻辑。

策略层不得懒加载生成 MarketSnapshot；MarketSnapshot 必须在 data_quality PASS 后由编排层主动生成或复用。

任何模块不得跳过中间强边界直接调用下游模块。

---

## 7. 同步链路与异步链路

### 7.1 必须同步完成的链路

以下链路必须同步完成，不能通过异步任务绕过：

```text
MarketSnapshot 生成前的数据质量确认
FeatureLayer 对当前 MarketSnapshot 的关键特征计算
AtomicSignal 对当前 FeatureValue 的判断
StrategySignal 生成
DecisionSnapshot 生成
BinanceSyncRun 显式选择 / 可消费校验
PriceSnapshot 生成 / TTL 校验
OrderPlan 生成
CandidateOrderIntent 生成
RiskCheck
ApprovedOrderIntent 生成或确认
ExecutionPreparation 真实下单前最终校验
Execution 真实下单请求
```

原因：

```text
这些步骤决定是否产生订单意图或真实交易。
如果异步乱序或延迟，可能导致基于过期数据、过期账户、过期价格、过期风控或错误仓位交易。
```

### 7.2 可以异步执行的链路

以下链路可以异步处理：

```text
Hermes 通知发送
ReviewRecord 聚合
ModelReviewRecord 生成
策略表现统计
事件日历同步
订单状态补偿查询
仓位心跳同步
数据缺口扫描
非阻塞告警
User Data Stream 消息消费（Tracking 层）
WebSocket PriceFeed 价格写入 PriceSnapshot（P1/P2）
```

异步任务必须记录：

```text
trace_id
trigger_source
executor_source（如适用）
任务状态
错误码
错误消息
开始时间
结束时间
```

### 7.3 不得异步绕过的边界

禁止通过异步任务绕过：

```text
DataQuality
MarketSnapshot
DecisionSnapshot
OrderPlan
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
真实交易开关
账户事实检查
仓位冲突检查
杠杆观测值检查
PriceSnapshot TTL 检查
```

---

## 8. 存储架构

### 8.1 MySQL

MySQL 是核心业务主存储。

以下数据必须以 MySQL 或等价可靠主存储为准：

```text
市场数据
数据质量结果
MarketSnapshot
FeatureSet
FeatureValue
AtomicSignal
DomainSignal
MarketRegime
StrategyRoute
StrategyParamSnapshot
StrategySignal
DecisionSnapshot
BinanceSyncRun
BinanceAccountSnapshot
BinanceBalanceSnapshot
BinancePositionSnapshot
BinanceSymbolRuleSnapshot
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheckResult
RiskRuleResult / RiskCheckIssue
ApprovedOrderIntent
ExecutionPreparationResult
ExecutionResult
ExchangeOrder
TradeFill
PositionState
TrackingRecord
TradeLifecycle
ReviewRecord
AlertEvent
HermesNotification
TaskRunRecord
IdempotencyRecord
RecoveryRecord
```

原则：

```text
核心结果必须结构化存储。
核心结果必须可追溯。
核心结果必须可审计。
核心结果不得只存在 Redis。
```

### 8.2 Redis

Redis 只用于：

```text
Celery broker
Celery result backend（如启用）
分布式锁
短期幂等控制
短期任务状态
限流计数
短期特征序列缓存
最近窗口缓存
WebSocket 最新价短期缓存（P1/P2，可丢失）
```

禁止：

```text
Redis 作为核心业务数据唯一存储。
Redis 长期保存订单、成交、仓位、复盘等核心数据。
Redis 中的数据丢失后导致无法复盘。
Redis 最新价缓存替代 PriceSnapshot 事实表。
```

Redis 数据必须满足：

```text
可过期
可重建
可从 MySQL 或外部可信数据源恢复
```

### 8.3 Artifact / 原始内容存储

以下内容不得直接污染主业务表：

```text
大模型完整原始返回
超长异常堆栈
长文本解释
大批量窗口计算明细
无法结构化的大型原始内容
```

应采用：

```text
隔离表
artifact 存储
归档存储
storage_ref
hash
摘要字段
```

主业务表只保存状态、摘要、关键证据、错误码、追溯 ID 和 artifact 引用。

---

## 9. 外部网关架构

### 9.1 Binance gateway

交易所访问必须通过 gateway / client 层。

推荐拆分：

```text
market_data_gateway：行情数据、K 线、资金费率、订单簿摘要
account_read_gateway：账户权益、余额、杠杆观测值、仓位读取
rules_gateway：交易规则、精度、步长、最小名义价值、contractSize
price_gateway：REST mark price / ticker 价格读取（P1）
trading_gateway：真实下单、撤单、订单查询、成交查询
user_data_stream_client：订单 / 成交 / 账户事件推送（P1/P2）
price_feed_client：WebSocket 价格 feed（P1/P2）
```

调用限制：

```text
market_data 模块只能调用行情类 gateway。
Binance Account Sync 只能调用 account_read_gateway / rules_gateway。
PriceSnapshotService P0 不直接调用真实 Binance REST，可从 Account Sync position mark_price 派生；P1 可调用 price_gateway。
OrderPlan 不直接调用 Binance gateway。
RiskCheck 不直接调用 Binance gateway。
ExecutionPreparation 可以通过 PriceSnapshotService 获取最新价格事实，但不直接连 WebSocket。
Execution 是唯一允许调用 trading_gateway 发起真实交易类请求的模块。
Tracking 可以调用订单查询 / 成交查询 / User Data Stream client。
策略、特征、原子信号、领域模块、大模型、Hermes 不得直接调用 trading_gateway。
```

所有 gateway 调用必须具备：

```text
超时
错误码
错误消息
trace_id
限频边界
密钥脱敏
真实请求开关（交易类请求必须更严格）
```

### 9.2 PriceFeed 与 PriceSnapshot

PriceFeed 是价格来源，PriceSnapshot 是价格事实。

P0：

```text
PriceSnapshot 可从 BinancePositionSnapshot.mark_price 派生。
不接 WebSocket。
不做真实 REST ticker 拉价。
不做盘口深度。
```

P1/P2：

```text
WebSocket / REST ticker / mark price 可以作为 PriceFeed 来源。
PriceFeed 负责获取最新价格。
PriceSnapshotService 负责固化价格事实、生成 hash、提供 TTL 校验。
```

禁止：

```text
OrderPlan 直接读 WebSocket。
RiskCheck 直接读 WebSocket。
ExecutionPreparation 直接读 WebSocket 内存值作为唯一事实。
WebSocket 最新价绕过 PriceSnapshot 进入风控或执行。
```

正确方向：

```text
WebSocket / REST / Account Sync
→ PriceSnapshotService
→ PriceSnapshot
→ OrderPlan / RiskCheck / ExecutionPreparation
```

### 9.3 Hermes client

Hermes 访问必须通过 notifications service / Hermes client。

业务模块不得各自封装 Hermes 请求。

Hermes client 负责：

```text
统一 URL / token 管理
真实发送开关
dry-run 记录
超时
错误记录
trace_id 传递
消息模板渲染
```

Hermes client 不得反向调用交易模块。

### 9.4 LLM client

大模型访问必须通过 review / model_review 相关 service。

LLM client 只允许服务：

```text
事后复盘
异常解释
策略研究辅助
```

禁止服务：

```text
实时交易判断
RiskCheck 放行
CandidateOrderIntent 生成
ApprovedOrderIntent 生成
Execution 执行
生产策略自动修改
```

大模型调用必须记录模型、版本、输入摘要、输出摘要、调用状态和 trace_id。

---

## 10. 交易执行架构

### 10.1 交易前置链路

任何可能进入交易执行的链路必须满足：

```text
DecisionSnapshot 已生成且可用
BinanceSyncRun 已显式传入且可消费
PriceSnapshot 已生成且未过期
OrderPlan 已生成
CandidateOrderIntent 已生成
RiskCheck 已通过
ApprovedOrderIntent 已生成或确认
ExecutionPreparation 已通过
执行模式已明确
真实交易开关已确认
```

缺任一条件，不得提交真实订单。

### 10.2 DecisionSnapshot

DecisionSnapshot 只表达目标仓位意图。

DecisionSnapshot 不得：

```text
读取账户 / 持仓
输出交易所订单动作
直接进入 RiskCheck
直接生成 CandidateOrderIntent
直接生成 ApprovedOrderIntent
直接触发 Execution
```

### 10.3 OrderPlan / CandidateOrderIntent

OrderPlan 负责：

```text
读取 DecisionSnapshot 目标仓位
读取 Binance Account Sync 账户事实
读取 PriceSnapshot 价格事实
计算当前仓位与目标仓位差值
生成 CandidateOrderIntent
生成 order_components
必要时生成 fallback_reduce_only_intent
```

OrderPlan 不负责：

```text
最终风控放行
真实下单
撤单
查订单
查成交
修改杠杆
修改保证金模式
执行前 price guard
```

CandidateOrderIntent 不是可执行订单，只能进入 RiskCheck。

### 10.4 RiskCheck / ApprovedOrderIntent

RiskCheck 负责：

```text
检查 CandidateOrderIntent
检查 order_components
检查账户、余额、持仓、symbol rule、observed_exchange_leverage
按 market_type 分别检查 U 本位 / COIN-M
决定 ALLOW / DENY / BLOCKED / FAILED
```

RiskCheck 不负责：

```text
生成目标仓位
重新生成 CandidateOrderIntent
真实下单
撤单
查订单
修改杠杆
修改保证金模式
```

ApprovedOrderIntent 表示：

```text
CandidateOrderIntent 已通过 RiskCheck
可以进入 ExecutionPreparation
```

ApprovedOrderIntent 不表示：

```text
已经下单
已经成交
一定会成交
```

### 10.5 ExecutionPreparation

ExecutionPreparation 负责真实下单前最终检查：

```text
ApprovedOrderIntent 未过期
RiskCheck 未过期
PriceSnapshot 最新性满足执行要求
price_deviation <= 配置阈值
active order 未冲突
真实交易开关开启
订单提交开关开启
execution_mode 明确
client_order_id 幂等检查
```

P0 建议 price guard：

```text
latest_price_snapshot_age <= 30 秒
price_deviation <= 0.5%
```

ExecutionPreparation 不负责：

```text
策略判断
目标仓位计算
风控规则重新审批
真实下单
```

### 10.6 Execution

Execution 负责：

```text
接收通过 ExecutionPreparation 的 ApprovedOrderIntent
确认执行模式
执行最终防重复校验
调用 trading_gateway 或模拟执行器
生成 ExecutionResult
记录 ExchangeOrder
触发订单与仓位 Tracking
触发 AlertEvent / 通知和复盘后续链路
```

Execution 不负责：

```text
生成策略信号
生成 DecisionSnapshot
计算目标仓位
决定风控是否放行
修改杠杆
调用大模型
让 Hermes 触发交易
```

### 10.7 真实交易提交边界

真实交易提交前必须再次确认：

```text
真实交易开关开启
订单提交开关开启
RiskCheck 未过期
ApprovedOrderIntent 未过期
ExecutionPreparation 已通过
价格快照满足执行要求
交易规则仍有效
账户和仓位状态未过期
ApprovedOrderIntent 未被执行过
execution_mode = real trading
```

### 10.8 Tracking

Tracking 负责：

```text
订单状态查询 / 接收
部分成交记录
完全成交记录
拒单 / 过期 / 撤单记录
TradeFill 生成
PositionState 更新
异常订单补偿查询
User Data Stream 消息消费（P1/P2）
```

Tracking 不负责：

```text
策略判断
风控放行
重新生成 ApprovedOrderIntent
自动重放交易
自动修改杠杆
```

订单状态不明时必须进入异常同步或人工恢复流程，不得静默假设成功或失败。

---

## 11. 执行模式隔离架构

系统必须支持并隔离：

```text
dry-run
paper trading
real trading
```

### 11.1 dry-run

dry-run 规则：

```text
不调用交易所交易接口
不生成真实 ExchangeOrder
不生成真实 TradeFill
不更新真实 PositionState
只记录预期行为和校验结果
```

### 11.2 paper trading

paper trading 规则：

```text
可以生成模拟 ExchangeOrder
可以生成模拟 TradeFill
可以生成模拟 PositionState
必须明确 execution_mode = paper
不得参与 real trading 的资金管理、风控和仓位判断
```

### 11.3 real trading

real trading 规则：

```text
只能基于真实交易所订单、成交、仓位更新
必须调用 trading_gateway
必须记录真实 exchange_order_id 或等价交易所标识
必须严格通过真实交易开关、RiskCheck、ApprovedOrderIntent、ExecutionPreparation 和 Execution
```

### 11.4 状态隔离方式

所有订单、成交、仓位、执行记录必须保存：

```text
execution_mode
trace_id
source
是否真实交易
```

paper trading 与 real trading 可通过以下方式隔离：

```text
独立表
独立字段
独立状态前缀
独立 service 查询过滤
```

无论采用哪种实现，必须保证：

```text
real trading 查询仓位时不会读到 paper trading 仓位。
real trading 风控不会使用 paper trading 订单和成交。
从 paper trading 切换到 real trading 必须人工确认。
```

---

## 12. trace_id、trigger_source 与幂等架构

### 12.1 trace_id

trace_id 用于追踪一次系统运行。

所有关键对象必须保存或传递 trace_id。

trace_id 用于回答：

```text
这次运行从哪里开始？
经过哪些模块？
每个模块输入输出是什么？
哪里失败？
是否生成目标仓位决策？
是否生成候选订单意图？
是否通过风控？
是否形成 ApprovedOrderIntent？
是否进入执行准备？
是否执行交易？
是否通知和复盘？
```

### 12.2 trigger_source

trigger_source 用于记录运行来源。

示例：

```text
cli
celery_beat
celery_worker
admin
system
retry
manual_recovery
```

Celery worker 执行任务时不得覆盖原始 trigger_source。如需记录执行者，应另设 executor_source。

### 12.3 业务幂等键

trace_id 不得单独作为业务幂等键。

原因：

```text
重试可能产生新的 trace_id，但仍属于同一业务事件。
同一分析周期可能因失败重跑产生多次运行。
仅靠 trace_id 无法防止重复生成 DecisionSnapshot、OrderPlan、CandidateOrderIntent、RiskCheckResult、ApprovedOrderIntent 或重复下单。
```

推荐业务幂等键方向：

```text
分析任务：(symbol, timeframe, analysis_close_time_utc, strategy_id, execution_mode)
DecisionSnapshot：(symbol, timeframe, analysis_close_time_utc, strategy_route, execution_mode)
PriceSnapshot：(symbol, market_type, account_domain, source, as_of_utc, mark_price)
OrderPlan：(decision_snapshot_id, binance_sync_run_id, price_snapshot_id, target_position_ratio, current_position_snapshot_id)
CandidateOrderIntent：(order_plan_id, intent_type, side, requested_size, requested_notional)
RiskCheckResult：(candidate_order_intent_id, order_plan_id, binance_sync_run_id, price_snapshot_id, rule_set_hash, risk_config_hash)
ApprovedOrderIntent：(risk_check_result_id, selected_candidate_order_intent_id)
交易所提交：(approved_order_intent_id, execution_mode, client_order_id)
人工恢复：(manual_request_id 或 confirm_token)
```

幂等策略：

```text
重复运行应复用已有结果，或明确跳过。
重复运行不得重复下单。
同一个 ApprovedOrderIntent 不得提交为多个有效交易所订单。
幂等冲突必须记录并可复盘。
```

---

## 13. Celery 与调度架构

Celery Beat 负责周期性派发任务。

Celery Worker 负责执行异步任务入口。

规则：

```text
Celery Beat 正式环境只能有一个有效调度实例。
Celery task 只调用 application service。
Celery task 不写复杂业务逻辑。
所有任务必须记录 TaskRunRecord 或等价运行记录。
任务必须有最大重试次数。
任务失败必须记录错误码、错误消息和 trace_id。
```

主策略周期任务应由调度入口调用 application service，不能在 task 中直接串完整业务逻辑。

调度必须配合业务幂等键，防止重复任务导致重复交易。

---

## 14. 异常恢复架构

异常分为两类：

```text
基础设施级异常
业务级异常
```

### 14.1 基础设施级异常

包括：

```text
网络超时
WebSocket 断线
REST 请求临时失败
Celery task 临时失败
订单状态查询失败
价格 feed 中断
```

处理方式：

```text
可以自动重试
必须有最大重试次数
必须记录失败状态
必要时告警
不得无限重试
```

### 14.2 业务级异常

包括：

```text
仓位不一致
订单状态长期未知
RiskCheck 状态异常
ApprovedOrderIntent 状态异常
ExecutionPreparation 状态异常
真实交易开关异常
交易所实际杠杆观测值缺失或不可信
重复下单风险
风控熔断解除
失败 ApprovedOrderIntent 是否可重放
```

处理方式：

```text
必须人工确认
不得自动绕过风控
不得自动修改杠杆
不得自动修改生产策略
不得静默恢复真实交易
```

### 14.3 人工恢复入口

人工恢复可以包括：

```text
强制同步订单状态
强制同步仓位状态
取消卡住的本地订单状态
重放失败但未提交交易所的 ApprovedOrderIntent
清除人工确认后的风控熔断标记
标记异常 TradeLifecycle 进入人工复盘
```

所有人工恢复必须记录：

```text
trace_id
trigger_source
operator 或人工确认来源
确认参数
恢复前状态
恢复后状态
执行结果
```

---

## 15. 回测与实盘复用架构

回测和实盘必须尽量复用同一套核心 domain / service 逻辑。

应复用：

```text
MarketSnapshot 构建规则
FeatureLayer
AtomicSignal
DomainSignal
MarketRegime
StrategyRoute
StrategyParamSnapshot
StrategySignal
DecisionSnapshot
OrderPlan 等价逻辑
RiskCheck 等价逻辑
ExecutionPreparation 等价检查
```

可以替换：

```text
数据来源
撮合器
执行器
手续费 / 滑点模型
账户事实模拟器
价格快照模拟器
仓位状态模拟器
```

回测不得：

```text
使用实盘不可获得的数据
绕过 K 线收盘后信号生成规则
假设当前 K 线收盘价完美成交
绕过 OrderPlan
绕过风控逻辑
绕过执行前价格保护逻辑
```

回测输出也必须能追溯到 DecisionSnapshot、OrderPlan、RiskCheck 和参数快照。

---

## 16. 通知与复盘架构

### 16.1 AlertEvent / Hermes 通知

订单相关正式事件必须写 AlertEvent。

包括：

```text
OrderPlan 生成 / blocked / no_order_required
CandidateOrderIntent 生成
RiskCheck ALLOW / DENY / BLOCKED / FAILED
fallback_reduce_only 被选择
ApprovedOrderIntent 生成
ExecutionPreparation 通过 / 阻断 / 失败
Execution 下单成功 / 失败
订单部分成交 / 完全成交 / 拒单 / 过期 / 撤单
仓位变化
```

Hermes 通知属于异步辅助链路。

通知失败不得改变交易事实。

Hermes 不得触发交易。

通知内容必须区分：

```text
系统分析
策略信号
目标仓位决策
候选订单意图
风控结果
ApprovedOrderIntent
执行准备结果
交易所订单
真实成交
仓位变化
复盘结论
```

### 16.2 复盘

复盘通过以下标识聚合证据：

```text
trace_id
DecisionSnapshot ID
OrderPlan ID
CandidateOrderIntent ID
RiskCheckResult ID
ApprovedOrderIntent ID
ExchangeOrder ID
TradeLifecycle ID
```

复盘模块负责查询并组装证据链，而不是要求上游一次性传入所有对象。

复盘不得自动修改生产策略。

### 16.3 大模型复盘辅助

大模型只允许基于 ReviewRecord 或审计摘要做事后解释。

大模型输出必须保存为 ModelReviewRecord 或等价记录。

大模型不得影响实时交易链路。

---

## 17. 第一阶段架构边界

第一阶段目标是最小可信自动交易闭环，不是完整量化平台。

第一阶段必须实现或保留边界：

```text
数据采集
数据质量
数据回补边界
MarketSnapshot
FeatureLayer
AtomicSignal
DomainSignal 边界
MarketRegime 边界
StrategyRoute 默认实现
StrategyParamSnapshot
StrategySignal
DecisionSnapshot
Binance Account Sync
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent 边界
ExecutionPreparation 边界
Execution dry-run / paper trading / real trading 边界
ExchangeOrder / TradeFill / PositionState / Tracking
AlertEvent / Hermes 通知
ReviewRecord
trace_id / trigger_source
调度与幂等
配置与安全
```

第一阶段可以简化：

```text
领域模块
市场环境识别
策略路由
策略参数管理
ApprovedOrderIntent 独立模块编号
执行层状态机复杂度
复盘归因
```

第一阶段不得提前实现：

```text
复杂多策略组合
自动参数优化
自动策略上线
机器学习交易模型
大模型实时交易判断
复杂 UI
未经验证的真实交易自动化
绕过 PriceSnapshot 的 WebSocket 直连交易
```

---

## 18. 与其他文档的关系

本文档只定义系统工程架构和运行边界。

关系如下：

```text
AGENTS.md
= Codex 工作纪律和最高级开发规则

docs/rules/project_invariants.md
= 系统红线和不变量

docs/requirements/01_project_scope.md
= 第一阶段范围

docs/requirements/02_system_capabilities.md
= 系统能力地图和 P0/P1/P2

docs/architecture/00_project_overview.md
= 系统模块总览

docs/architecture/01_module_map.md
= 模块职责、输入输出、允许调用、禁止调用、失败处理

docs/architecture/system_architecture.md
= 工程分层、存储、调度、gateway、同步异步、模式隔离、trace、幂等和恢复架构

docs/architecture/data_flow.md
= 主链路和数据对象流转说明

docs/requirements/*.md
= 模块级详细需求

docs/plans/*.md
= 开发计划和实施步骤
```

如果本文档与 `project_invariants.md` 冲突，以 `project_invariants.md` 为准。

如果本文档与 `01_module_map.md` 在模块边界上冲突，以 `01_module_map.md` 为准；本文档不重复维护模块边界表。

---

## 19. 总结

本文档定义系统工程架构的约束重点：

```text
工程分层
Django app 划分原则
service / domain / task / command 边界
同步链路与异步链路
MySQL / Redis / artifact 存储边界
Binance / PriceFeed / Hermes / LLM gateway 边界
OrderPlan / RiskCheck / ExecutionPreparation / Execution 边界
dry-run / paper trading / real trading 模式隔离
trace_id / trigger_source / 幂等架构
Celery 与调度架构
异常恢复架构
回测与实盘复用架构
通知与复盘架构
第一阶段架构边界
```

它的目标是防止 Codex 或开发者把系统写成：

```text
策略直接下单
DecisionSnapshot 直接生成订单动作
RiskCheck 直接消费 DecisionSnapshot 的动作字段或 ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE 枚举
OrderPlan / RiskCheck 直接连 WebSocket
task 里堆满业务逻辑
model 里写风控和交易逻辑
Redis 充当主数据库
paper 仓位污染 real 交易
大模型进入实时交易链路
Hermes 反向触发交易
回测和实盘链路完全割裂
```

任何实现都必须优先保证架构边界清晰、交易链路受控、数据可追溯、异常可恢复和复盘可解释。
