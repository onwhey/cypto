# 01_module_map.md

# 模块地图与边界说明

## 1. 文档目的

本文档定义系统主要模块的职责边界、输入输出、允许调用关系、禁止调用关系和失败处理原则。

本文档用于约束后续模块级 requirements、architecture、plans 和 Codex 实现。

本文档不定义：

```text
具体数据库表结构
具体字段设计
具体 Celery task 名称
具体 Django app 目录
具体交易所 API endpoint
具体策略公式
具体回测撮合算法
具体订单状态机
具体代码实现
```

如果本文档与 `docs/rules/project_invariants.md` 冲突，以 `project_invariants.md` 为准。

---

## 2. 模块调用总原则

系统采用单向主链路。

正式主链路为：

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
→ StrategySignalQuality
→ DecisionSnapshot
→ Binance Account Sync
→ PriceSnapshot
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation
→ Execution
→ Tracking / ExchangeOrder / TradeFill / PositionState
→ AlertEvent / ReviewRecord
```

核心原则：

```text
上游模块不得反向依赖下游模块。
策略模块不得直接调用交易执行模块。
策略模块不得直接访问交易所交易接口。
FeatureLayer 不得调用 AtomicSignal。
AtomicSignal 不得调用其他 AtomicSignal。
AtomicSignal 不得直接调用 StrategySignal、DecisionSnapshot、OrderPlan、RiskCheck 或 Execution。
DomainSignal 不得直接生成订单。
Hermes / notifications 不得触发交易动作。
大模型不得进入实时交易链路。
```

交易链路必须遵守：

```text
DecisionSnapshot 只表达目标仓位意图，不表达具体订单动作。
OrderPlan 是唯一把目标仓位转换成 CandidateOrderIntent 的模块。
RiskCheck 只检查 CandidateOrderIntent / order_components，不直接检查 DecisionSnapshot。
ApprovedOrderIntent 是风控通过后的订单意图，不是真实交易所订单。
ExecutionPreparation 是执行前最终保护层。
Execution 是唯一允许真实下单的模块。
Tracking 负责订单、成交和仓位状态追踪。
```

辅助模块不一定出现在每次主链路中，但必须作为统一服务存在：

```text
数据回补模块：由 pending BackfillRequest 扫描 / 领取、后续统一编排层、recovery scan 或人工命令触发，用于补齐缺失数据。
Binance Account Sync：只读账户事实来源。
PriceSnapshot：价格事实层，为 OrderPlan / RiskCheck / ExecutionPreparation 提供价格快照。
配置与安全模块：统一管理真实交易、账户读取、通知、大模型等危险开关。
监控与恢复模块：区分基础设施级自动恢复和业务级人工恢复。
```

模块交互优先通过 service / domain 层，不得跨模块直接操作对方内部实现细节。

---

## 3. 模块边界总表

| 模块 | 主要职责 | 输入对象 | 输出对象 | 允许调用 | 禁止调用 | 失败处理 |
|---|---|---|---|---|---|---|
| 数据采集 | 获取 K 线和市场数据 | 配置、交易所行情数据源 | MarketData、KlineRecord | 交易所行情 REST、存储服务 | 策略、风控、交易执行、交易接口 | 记录失败、告警、不得生成不可信数据 |
| 数据质量 | 校验数据完整性与可用性 | MarketData、KlineRecord | DataQualityResult、DataQualityIssue、AlertEvent | 数据存储、AlertEvent 存储、BackfillRequest 存储 | 策略、交易执行 | 不合格数据必须阻断；可回补问题创建 BackfillRequest |
| 数据回补 | 补齐缺失或异常 K 线 | BackfillRequest、DataQualityIssue、缺失范围、人工指令 | BackfillRun、BackfillResult（结果摘要，如保留）、KlineRecord、复检标记 / recheck_required 状态 | 交易所行情 REST、数据存储 | 策略、交易执行、交易接口 | 回补失败必须留痕，不得伪造数据；DataQualityResult 由后续 DataQuality 复检产生，不属于 data_backfill 输出 |
| MarketSnapshot | 固定分析时刻市场状态 | KlineRecord、DataQualityResult | MarketSnapshot | 数据存储、数据质量结果 | 策略下游以外模块、交易接口 | 快照失败则阻断 FeatureLayer |
| FeatureLayer | 统一计算基础特征和中性复合特征 | MarketSnapshot | FeatureSet、FeatureValue、FeatureQualityCheck | MarketSnapshot 查询、特征计算器 | AtomicSignal 反向调用、策略、交易执行 | 特征质量失败则阻断对应 AtomicSignal |
| AtomicSignal | 生成最小市场事件判断 | FeatureValue | AtomicSignal、AtomicSignalQualityCheck | FeatureLayer 输出 | 其他 AtomicSignal、策略、交易执行 | 信号失败必须记录，不得静默进入领域层 |
| DomainSignal | 聚合同类 AtomicSignal | AtomicSignal | DomainSignal、DomainQualityCheck | AtomicSignal 查询 | 交易执行、交易所 | 领域质量失败则阻断或降级 |
| MarketRegime | 判断市场状态 | DomainSignal、MarketSnapshot | MarketRegime、RegimeQualityStatus | DomainSignal 查询 | 交易执行、交易所 | 不确定状态输出 UNKNOWN / NOT_TRADABLE |
| StrategyRoute | 选择适用策略 | MarketRegime、DomainSignal、配置 | StrategyRoute | 配置、市场环境结果 | 交易执行、交易所 | 无匹配策略输出 NO_ROUTE / fail-closed |
| StrategyParamSnapshot | 提供策略参数来源和参数快照 | 配置文件、环境配置、人工确认版本 | StrategyConfig、StrategyParamSnapshot | 配置系统、存储服务 | 自动调参直接写生产配置、交易执行 | 参数缺失或版本不明时阻断策略信号 |
| StrategySignal | 生成策略方向、强度、置信度和证据 | StrategyRoute、DomainSignal、AtomicSignal、参数快照 | StrategySignal | 上游信号与配置 | 交易执行、交易所、OrderPlan 直接生成 | 策略异常必须记录并阻断 DecisionSnapshot |
| StrategySignalQuality | 校验 StrategySignal 是否可被下游消费 | StrategySignal | StrategySignalQualityResult、Issue | StrategySignal 查询、AlertEvent | 交易执行、交易接口 | fail-closed；不合格不得进入 DecisionSnapshot |
| DecisionSnapshot | 记录每周期目标仓位意图和证据快照 | StrategySignal、StrategySignalQuality、上游证据链 | DecisionSnapshot | 上游证据、存储服务 | 账户/持仓读取、交易所、OrderPlan 内部逻辑、Execution | 生成失败不得进入 OrderPlan |
| Binance Account Sync | 只读同步账户、余额、持仓、symbol rule | Binance REST 只读接口、配置 | BinanceSyncRun、Account/Balance/Position/SymbolRuleSnapshot | Binance 只读账户接口、存储、AlertEvent | 下单、撤单、划转、调杠杆、调保证金模式、OrderPlan/RiskCheck 业务逻辑 | failed / stale 不得被正式交易链路消费 |
| PriceSnapshot | 固化某一时刻用于估值/风控/执行前检查的价格事实 | PositionSnapshot.mark_price、fixture、后续 REST/WS 来源 | PriceSnapshot | Account Sync 快照、价格源、存储 | 直接下单、策略决策、绕过 ExecutionPreparation | 价格缺失/过期时阻断 OrderPlan 或 RiskCheck |
| OrderPlan | 将目标仓位转换为候选订单意图 | DecisionSnapshot、BinanceSyncRun、PriceSnapshot、当前持仓、symbol rule | OrderPlan、CandidateOrderIntent、order_components、fallback_reduce_only_intent | Binance Account Sync selector、PriceSnapshot、存储、AlertEvent | RiskCheck 结果伪造、真实下单、撤单、调杠杆、调保证金模式 | 无法生成合法候选时 blocked / no_order_required / AlertEvent |
| CandidateOrderIntent | 表达待风控的候选订单意图 | OrderPlan | CandidateOrderIntent | OrderPlan 存储 | 交易所接口、直接执行、绕过 RiskCheck | 不可消费时 RiskCheck 必须 BLOCKED |
| RiskCheck | 风控审查 CandidateOrderIntent / order_components | CandidateOrderIntent、OrderPlan、BinanceSyncRun、PriceSnapshot、风险规则 | RiskCheckResult、RiskRuleResult、RiskCheckIssue | Account Sync selector、PriceSnapshot、规则插件、AlertEvent | 直接消费 DecisionSnapshot 的动作字段或 ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE 枚举、真实下单、撤单、任意 MODIFY、调杠杆 | ALLOW / DENY / BLOCKED / FAILED 均 AlertEvent |
| ApprovedOrderIntent | 表示风控通过的订单意图 | RiskCheckResult、selected CandidateOrderIntent | ApprovedOrderIntent | RiskCheck 结果、存储、AlertEvent | 真实下单、修改订单参数、绕过 ExecutionPreparation | 生成失败不得进入 ExecutionPreparation |
| ExecutionPreparation | 执行前最终保护与准备 | ApprovedOrderIntent、最新 PriceSnapshot、active order 状态、执行配置 | ExecutionPreparationResult、PreparedExecutionRequest / blocked result | PriceSnapshotService、配置、订单状态查询、AlertEvent | 策略、RiskCheck 业务判断、直接改仓位 | price guard 失败或快照过期必须阻断执行 |
| Execution | 唯一真实下单入口 | PreparedExecutionRequest / ApprovedOrderIntent、执行配置 | ExecutionResult、ExchangeOrder | execution gateway、订单存储、AlertEvent | 策略、原子信号、大模型、Hermes 触发交易、自动调杠杆 | 下单成功/失败/未知均记录并 AlertEvent |
| Tracking | 追踪订单、成交和仓位状态 | ExecutionResult、交易所订单/成交/仓位回报 | ExchangeOrder、TradeFill、PositionState、OrderStatusRecord | REST 查询、后续 User Data Stream、存储、AlertEvent | 策略决策、自动改策略、自动调杠杆 | 状态不明必须异常同步/人工恢复 |
| Hermes / notifications | 发送状态通知 | 业务事件、异常、复盘摘要 | HermesNotification / AlertEvent | notifications service | 交易执行、策略决策、订单生成 | 通知失败必须记录，不改变交易事实 |
| Review | 聚合证据链并归因 | trace_id、DecisionSnapshot ID、订单/成交/仓位记录 | ReviewRecord、ReviewFinding | 历史数据、审计查询、大模型复盘辅助 | 实时下单、自动改策略 | 复盘失败不得修改生产策略 |
| ModelReview | 事后解释与研究辅助 | ReviewRecord、审计摘要 | ModelReviewRecord | 模型服务、复盘查询 | 实时交易、RiskCheck、OrderPlan、Execution | 调用失败只影响解释，不影响交易事实 |
| 配置与安全 | 管理开关、密钥、危险配置 | env、settings、配置文件 | ConfigSnapshot、ConfigCheckResult | 配置读取、安全检查 | 业务硬编码密钥、自动开启真实交易 | 危险配置异常必须阻断真实交易 |
| 调度与幂等 | 定时触发与防重复 | 调度配置、trace_id、分析周期 | TaskRunRecord、IdempotencyRecord | Celery、Redis 锁、DB 幂等 | 复杂业务逻辑、绕过 service | 重复任务不得重复下单 |
| 监控与恢复 | 异常发现、告警、恢复入口 | 任务、订单、仓位、告警 | AlertRecord、RecoveryRecord | 通知、状态查询、人工确认入口 | 绕过风控恢复交易、自动业务级恢复 | 恢复操作必须留痕 |

---

## 4. 数据采集模块

### 4.1 职责

负责从可信数据源获取市场数据。

第一阶段重点：

```text
Binance BTCUSDT U 本位 K 线
历史 K 线采集
最新已收盘 K 线采集
```

WebSocket 不参与 K 线采集。WebSocket 只作为后续实时价格、执行保护、订单追踪等模块的价格 / 事件来源预留。

### 4.2 输入对象

```text
交易所配置
交易品种
时间周期
采集时间范围
trigger_source
trace_id
```

### 4.3 输出对象

```text
MarketData
KlineRecord
DataSourceRecord
TaskRunRecord
```

### 4.4 允许调用

```text
交易所行情 REST 接口
数据存储 service
日志 service
通知 service
```

### 4.5 禁止调用

```text
策略模块
风控模块
Execution
交易所下单接口
仓位修改接口
大模型
```

### 4.6 失败处理

```text
采集失败必须记录错误码和 trace_id。
数据不完整不得伪造。
数据缺失应交由数据质量和回补流程处理。
不得静默生成 MarketSnapshot。
```

---

## 5. 数据质量模块

### 5.1 职责

负责校验市场数据是否可进入下游链路。

### 5.2 输入对象

```text
MarketData
KlineRecord
DataSourceRecord
trace_id
```

### 5.3 输出对象

```text
DataQualityResult
DataQualityIssue
AlertEvent
```

### 5.4 允许调用

```text
数据存储 service
DataQualityResult 存储 service
DataQualityIssue 存储 service
AlertEvent 存储 service
BackfillRequest 存储 service
```

### 5.5 禁止调用

```text
RiskCheck
Execution
交易所交易接口
Hermes Webhook 直接发送
大模型
直接 Binance REST
KlineRecord 写入
```

### 5.6 失败处理

```text
发现任何 K 线质量问题，DataQualityResult 必须为 FAIL。
必须写 DataQualityIssue 和 AlertEvent。
可回补问题创建 BackfillRequest。
data_quality 不执行回补，不调用 BackfillService，不投递 data_backfill task。
必须阻断 MarketSnapshot。
第一阶段不使用 WARN 放行或轻微降级机制。
```

---

## 6. 数据回补模块

### 6.1 职责

负责扫描 / 领取 pending BackfillRequest，在 claim BackfillRequest 后创建 BackfillRun，并通过可信数据源重新拉取缺失区间。BackfillRun 完成后，data_backfill 只标记 / 暴露需要 DataQuality 复检的状态；实际复检由后续统一编排层、recovery scan 或人工命令重新执行。

### 6.2 输入对象

```text
BackfillRequest
DataQualityIssue
缺失时间范围
交易品种
时间周期
人工回补指令
trigger_source
trace_id
```

### 6.3 输出对象

```text
BackfillRun
BackfillResult（结果摘要，如保留）
KlineRecord
复检标记 / recheck_required 状态
TaskRunRecord
```

DataQualityResult 由后续 DataQuality 复检产生，不属于 data_backfill 输出。

### 6.4 允许调用

```text
交易所行情 REST 接口
数据存储 service
限频控制 service
通知 service
```

### 6.5 禁止调用

```text
策略模块
风控模块
Execution
交易所下单接口
仓位接口
```

### 6.6 失败处理

```text
回补失败必须记录缺失范围、错误码和 trace_id。
回补结果必须重新经过数据质量检查。
不得为了通过质量检查而伪造或手工改写 K 线。
回补任务必须具备幂等和限频边界。
```

---

## 7. MarketSnapshot 模块

### 7.1 职责

负责在策略分析周期固定市场状态，形成可复盘的 MarketSnapshot。

### 7.2 输入对象

```text
4h KlineRecord 窗口
1d KlineRecord 窗口
4h DataQualityResult = PASS
1d DataQualityResult = PASS
analysis_close_time_utc
lookback_4h_count
lookback_1d_count
trace_id
trigger_source
```

### 7.3 输出对象

```text
MarketSnapshot
SnapshotBuildRecord
```

### 7.4 允许调用

```text
数据存储 service
数据质量结果查询 service
```

### 7.5 禁止调用

```text
FeatureLayer 以外的下游业务逻辑
策略模块
风控模块
Execution
交易所交易接口
```

### 7.6 失败处理

```text
MarketSnapshot 生成失败时不得进入 FeatureLayer。
必须记录失败原因、输入范围和 trace_id。
```

---

## 8. FeatureLayer 模块

### 8.1 职责

负责基于 MarketSnapshot 统一计算特征。

### 8.2 输入对象

```text
MarketSnapshot
特征配置
特征参数版本
trace_id
```

### 8.3 输出对象

```text
FeatureSet
FeatureValue
FeatureQualityCheck
```

### 8.4 允许调用

```text
MarketSnapshot 查询 service
特征计算器
特征存储 service
短期内存缓存
可选 Redis 短期缓存
```

### 8.5 禁止调用

```text
AtomicSignal 反向调用
StrategySignal
DecisionSnapshot
OrderPlan
RiskCheck
Execution
交易所接口
Hermes 直接发送
大模型
```

### 8.6 失败处理

```text
FeatureQualityCheck 失败时，对应特征不得进入 AtomicSignal。
同一 MarketSnapshot 下同一特征应避免重复计算。
特征计算失败必须记录 feature_key、参数版本、trace_id。
```

---

## 9. AtomicSignal 模块

### 9.1 职责

负责基于 FeatureValue 生成最小市场事件判断。

原子信号可以复杂，但输出必须表达一个单一市场事件。

### 9.2 输入对象

```text
FeatureSet
FeatureValue
原子信号配置
参数版本
trace_id
```

### 9.3 输出对象

```text
AtomicSignal
AtomicSignalQualityCheck
```

### 9.4 允许调用

```text
FeatureLayer 输出查询 service
原子信号计算器
原子信号存储 service
```

### 9.5 禁止调用

```text
其他 AtomicSignal 的 result
DomainSignal
StrategySignal
DecisionSnapshot
OrderPlan
RiskCheck
Execution
交易所接口
大模型
```

### 9.6 失败处理

```text
AtomicSignalQualityCheck 失败时不得进入 DomainSignal。
单个原子信号失败不得静默忽略，必须记录状态。
如果关键原子信号缺失，下游必须阻断或降级。
```

### 9.7 原子信号独立性说明

```text
原子信号之间保持平权。
原子信号不得直接调用另一个原子信号。
原子信号不得覆盖、否定、修改另一个原子信号结果。
原子信号如需复杂事实，应读取 FeatureLayer 提供的 FeatureValue。
不同原子信号之间的冲突、确认、否定、降权，由领域模块或更高层处理。
```

---

## 10. DomainSignal 模块

### 10.1 职责

负责聚合同类 AtomicSignal，形成领域结论。

### 10.2 输入对象

```text
AtomicSignal
AtomicSignalQualityCheck
领域配置
trace_id
```

### 10.3 输出对象

```text
DomainSignal
DomainQualityCheck
```

### 10.4 允许调用

```text
AtomicSignal 查询 service
领域聚合 service
领域存储 service
```

### 10.5 禁止调用

```text
FeatureLayer 重新计算特征
StrategySignal 直接执行
OrderPlan
RiskCheck
Execution
交易所接口
```

### 10.6 失败处理

```text
DomainQualityCheck 失败时不得进入 MarketRegime 或 StrategyRoute。
证据不足时应输出不确定或不可用状态，不得伪造确定结论。
```

---

## 11. MarketRegime 模块

### 11.1 职责

负责根据 DomainSignal 判断当前市场状态。

### 11.2 输入对象

```text
DomainSignal
DomainQualityCheck
MarketSnapshot
trace_id
```

### 11.3 输出对象

```text
MarketRegime
RegimeQualityStatus
```

### 11.4 允许调用

```text
DomainSignal 查询 service
市场环境识别 service
```

### 11.5 禁止调用

```text
交易所接口
Execution
OrderPlan
CandidateOrderIntent
Hermes 直接触发交易
```

### 11.6 失败处理

```text
环境识别失败时应输出 UNKNOWN 或 NOT_TRADABLE。
不得在市场状态不明时默认允许交易。
```

---

## 12. StrategyRoute 模块

### 12.1 职责

负责根据 MarketRegime、DomainSignal 和配置选择适用策略。

第一阶段可以只有默认趋势跟踪路由。

### 12.2 输入对象

```text
MarketRegime
DomainSignal
策略配置
trace_id
```

### 12.3 输出对象

```text
StrategyRoute
```

### 12.4 允许调用

```text
市场环境查询 service
策略配置 service
策略路由 service
```

### 12.5 禁止调用

```text
Execution
交易所接口
RiskCheck
大模型实时决策
```

### 12.6 失败处理

```text
无匹配策略时输出 NO_ROUTE 或 NOT_TRADABLE 路由。
不得因为路由失败而默认执行某策略。
```

---

## 13. StrategyParamSnapshot 模块

### 13.1 职责

负责管理策略参数来源、参数版本和参数快照。

### 13.2 输入对象

```text
策略配置文件
环境配置
人工确认的参数版本
trace_id
```

### 13.3 输出对象

```text
StrategyConfig
StrategyParamSnapshot
```

### 13.4 允许调用

```text
配置 service
存储 service
```

### 13.5 禁止调用

```text
自动调参服务直接写生产配置
Execution
交易所接口
```

### 13.6 失败处理

```text
参数缺失或版本不明时不得生成 StrategySignal。
生产参数变更必须人工确认。
第一阶段可以采用参数快照嵌入方式，把本次策略使用的参数字典记录到 StrategySignal 或 DecisionSnapshot。
```

---

## 14. StrategySignal 模块

### 14.1 职责

负责基于 StrategyRoute 和上游证据生成 StrategySignal。

StrategySignal 可以表达：

```text
方向
强度
置信度
证据摘要
策略模式
使用的原子信号
```

StrategySignal 不是订单，也不是最终执行指令。

### 14.2 输入对象

```text
StrategyRoute
MarketRegime
DomainSignal
AtomicSignal
策略参数快照
trace_id
```

### 14.3 输出对象

```text
StrategySignal
```

### 14.4 允许调用

```text
策略配置 service
上游证据查询 service
策略信号 service
```

### 14.5 禁止调用

```text
Execution
交易所接口
CandidateOrderIntent 直接生成
修改仓位
修改杠杆
```

### 14.6 失败处理

```text
策略信号生成失败必须记录。
不得在策略信号不完整时进入 DecisionSnapshot。
```

---

## 15. StrategySignalQuality 模块

### 15.1 职责

负责校验 StrategySignal 是否可进入 DecisionSnapshot。

### 15.2 输入对象

```text
StrategySignal
MarketSnapshot
trace_id
trigger_source
```

### 15.3 输出对象

```text
StrategySignalQualityResult
StrategySignalQualityIssue
```

### 15.4 允许调用

```text
StrategySignal 查询 service
AlertEvent 存储 service
```

### 15.5 禁止调用

```text
DecisionSnapshot 生成逻辑
OrderPlan
RiskCheck
Execution
交易所接口
```

### 15.6 失败处理

```text
StrategySignalQualityResult 未通过时，不得生成可交易 DecisionSnapshot。
WARNING 在 P0 不允许向下游放行，除非对应 requirement 明确修改。
```

---

## 16. DecisionSnapshot 模块

### 16.1 职责

负责记录系统每个策略分析周期的目标仓位决策快照。

DecisionSnapshot 只表达：

```text
target_intent
target_position_ratio
decision_policy
evidence_summary
```

DecisionSnapshot 不表达具体交易所订单动作。

### 16.2 输入对象

```text
StrategySignal
StrategySignalQualityResult
MarketSnapshot
FeatureSet / FeatureValue 引用
AtomicSignal 引用
DomainSignal 引用
MarketRegime
StrategyRoute
StrategyParamSnapshot
trace_id
trigger_source
```

### 16.3 输出对象

```text
DecisionSnapshot
```

### 16.4 允许调用

```text
上游证据查询 service
DecisionPolicy
DecisionSnapshot 存储 service
AlertEvent 存储 service
```

### 16.5 禁止调用

```text
Binance Account Sync selector
账户余额 / 当前持仓读取
PriceSnapshot
OrderPlan 内部逻辑
RiskCheck
Execution
交易所接口
Hermes 触发交易
大模型实时判断
```

### 16.6 失败处理

```text
DecisionSnapshot 生成失败不得进入 OrderPlan。
DecisionSnapshot 过期或不可用时，OrderPlan 不得生成 CandidateOrderIntent。
target_position_ratio 超出合法范围必须 fail-closed。
```

---

## 17. Binance Account Sync 模块

### 17.1 职责

负责只读同步 Binance active account domain 的账户、余额、持仓和交易规则快照。

正式链路中，它服务：

```text
PriceSnapshot
OrderPlan
RiskCheck
ExecutionPreparation / Tracking 的账户事实核验
```

但它仍然只是账户事实来源，不是交易模块。

### 17.2 输入对象

```text
BINANCE_ACTIVE_MARKET_TYPE
Binance 只读账户接口
trigger_source
trace_id
```

### 17.3 输出对象

```text
BinanceSyncRun
BinanceAccountSnapshot
BinanceBalanceSnapshot
BinancePositionSnapshot
BinanceSymbolRuleSnapshot
BinanceSyncHealth
```

### 17.4 允许调用

```text
Binance 只读 REST account / balance / positionRisk / exchangeInfo
存储 service
AlertEvent 存储 service
```

### 17.5 禁止调用

```text
下单
撤单
资金划转
修改杠杆
修改 margin type
修改 position mode
OrderPlan 业务逻辑
RiskCheck 业务逻辑
PriceSnapshot 反向修改 BinancePositionSnapshot
WebSocket / User Data Stream
```

### 17.6 失败处理

```text
只有 BinanceSyncRun.status=succeeded 才能被正式交易链路消费。
failed / running / stale 的 sync_run 不得被 OrderPlan / RiskCheck 消费。
正式链路必须显式传入 binance_sync_run_id，不得 fallback latest succeeded。
account_domain / market_type / snapshot_set_hash 不一致必须不可消费。
```

---

## 18. PriceSnapshot 模块

### 18.1 职责

负责把某一时刻用于估值、风控、执行前判断的价格固化为可追溯价格事实。

PriceSnapshot 是价格事实层，不是行情分析层，不是最终成交价。

### 18.2 输入对象

```text
BinancePositionSnapshot.mark_price
symbol
market_type
account_domain
source
as_of_utc
trace_id
trigger_source
```

P0 可从 Binance Account Sync 的 positionRisk mark_price 派生。

### 18.3 输出对象

```text
PriceSnapshot
PriceSnapshotHealth / price freshness result
```

### 18.4 允许调用

```text
Binance Account Sync selector
PriceSnapshot 存储 service
```

后续 P1/P2 可接入：

```text
REST mark price
WebSocket mark price
best bid / best ask
```

但下游仍应通过 PriceSnapshot 消费价格事实。

### 18.5 禁止调用

```text
策略决策
OrderPlan 以外的订单生成逻辑
RiskCheck 放行逻辑
Execution 真实下单
交易所交易接口
```

### 18.6 失败处理

```text
价格缺失、不可解析、<=0 或超过 RiskCheck TTL 时，OrderPlan / RiskCheck 必须 fail-closed。
PriceSnapshot 不是 ExecutionPreparation 的最终价格保护替代品。
```

---

## 19. OrderPlan 模块

### 19.1 职责

负责把 DecisionSnapshot 的目标仓位转换成可风控的 CandidateOrderIntent。

OrderPlan 是唯一允许执行以下转换的模块：

```text
target_position_ratio
+ 当前账户权益
+ 当前持仓
+ PriceSnapshot
+ symbol rule
→ CandidateOrderIntent
```

### 19.2 输入对象

```text
DecisionSnapshot
BinanceSyncRun
Binance trading context bundle
PriceSnapshot
OrderPlanConfig
trace_id
trigger_source
```

### 19.3 输出对象

```text
OrderPlan
CandidateOrderIntent
order_components
primary_intent
fallback_reduce_only_intent
```

### 19.4 允许调用

```text
DecisionSnapshot 查询 service
Binance Account Sync selector
PriceSnapshot service
OrderPlan calculator
OrderPlan 存储 service
AlertEvent 存储 service
```

### 19.5 禁止调用

```text
RiskCheck 内部规则
Execution
交易所交易接口
撤单
查成交
修改杠杆
修改 margin type
修改 position mode
WebSocket 直接读取
大模型生成订单
Hermes 触发订单
```

### 19.6 失败处理

```text
无法取得可消费 BinanceSyncRun / PriceSnapshot 时必须 blocked。
position_mode != one_way 时必须 blocked。
active_order_exists 时必须 blocked。
理论调仓名义小于 min_rebalance_notional 时输出 no_order_required。
净额反手必须生成 primary_intent 与 fallback_reduce_only_intent。
```

---

## 20. CandidateOrderIntent 模块

### 20.1 职责

负责表达由 OrderPlan 生成、等待 RiskCheck 审批的候选订单意图。

CandidateOrderIntent 不是可执行订单。

### 20.2 输入对象

```text
OrderPlan
order_components
price_snapshot_id
binance_sync_run_id
trace_id
```

### 20.3 输出对象

```text
CandidateOrderIntent
```

### 20.4 允许调用

```text
OrderPlan 存储 service
审计 / hash service
```

### 20.5 禁止调用

```text
交易所接口
Execution
RiskCheck 放行逻辑
自行修改订单数量
```

### 20.6 失败处理

```text
hash 失败、结构不合法、已消费、已取消或过期时，RiskCheck 必须 BLOCKED。
```

---

## 21. RiskCheck 模块

### 21.1 职责

负责交易前强制风控审查。

正式 RiskCheck 只检查：

```text
CandidateOrderIntent
order_components
BinanceSyncRun / account context
PriceSnapshot
RiskRuleDefinition / risk_config
```

RiskCheck 不直接消费 DecisionSnapshot 的动作字段或 ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE 枚举。

### 21.2 输入对象

```text
CandidateOrderIntent
OrderPlan
BinanceSyncRun
Binance trading context bundle
PriceSnapshot
RiskRuleDefinition / risk_config
trace_id
trigger_source
```

### 21.3 输出对象

```text
RiskCheckResult
RiskRuleResult
RiskCheckIssue
selected_intent_type
```

状态：

```text
ALLOW
DENY
BLOCKED
FAILED
```

### 21.4 允许调用

```text
CandidateOrderIntent 查询 service
OrderPlan 查询 service
Binance Account Sync selector
PriceSnapshot 查询 service
RiskRuleRegistry / RuleEngine
AlertEvent 存储 service
```

### 21.5 禁止调用

```text
DecisionSnapshot.decision_action
ENTER_LONG / ENTER_SHORT / EXIT 禁止输入合同
Execution
交易所下单接口
撤单
任意 MODIFY
自动缩小订单
修改杠杆
修改 margin type
大模型判断放行
```

### 21.6 失败处理

```text
ALLOW / DENY / BLOCKED / FAILED 都必须 AlertEvent。
increase_risk 不通过但 fallback_reduce_only 通过时，可选择 fallback_reduce_only_intent。
fallback 失败时按原因 DENY / BLOCKED / FAILED。
P0 不做任意 MODIFY。
```

---

## 22. ApprovedOrderIntent 模块

### 22.1 职责

负责表达已经通过 RiskCheck 的订单意图。

ApprovedOrderIntent 是进入 ExecutionPreparation 的输入，不是真实交易所订单。

### 22.2 输入对象

```text
RiskCheckResult
selected CandidateOrderIntent
trace_id
trigger_source
```

### 22.3 输出对象

```text
ApprovedOrderIntent
```

### 22.4 允许调用

```text
RiskCheckResult 查询 service
CandidateOrderIntent 查询 service
ApprovedOrderIntent 存储 service
AlertEvent 存储 service
```

### 22.5 禁止调用

```text
交易所接口
真实下单
修改订单数量
绕过 ExecutionPreparation
```

### 22.6 失败处理

```text
RiskCheckResult 非 ALLOW 不得生成 ApprovedOrderIntent。
ApprovedOrderIntent 过期或已消费时不得进入 ExecutionPreparation。
```

---

## 23. ExecutionPreparation 模块

### 23.1 职责

负责真实执行前的最终准备和保护检查。

包括：

```text
ApprovedOrderIntent 有效性检查
active order 检查
执行模式开关检查
最新 PriceSnapshot 刷新 / 校验
price guard：latest_price_snapshot_age <= 30 秒
price_deviation <= 0.5%
clientOrderId / 幂等准备
```

### 23.2 输入对象

```text
ApprovedOrderIntent
latest PriceSnapshot
execution_mode
execution_config
trace_id
trigger_source
```

### 23.3 输出对象

```text
ExecutionPreparationResult
PreparedExecutionRequest
```

### 23.4 允许调用

```text
ApprovedOrderIntent 查询 service
PriceSnapshotService
active order selector
配置与安全 service
AlertEvent 存储 service
```

### 23.5 禁止调用

```text
策略信号
DecisionSnapshot 重新判断
RiskCheck 内部规则重新实现
真实下单
修改杠杆
修改 margin type
```

### 23.6 失败处理

```text
price guard 失败必须 blocked。
无法取得新鲜价格必须 blocked。
执行开关未开启时不得进入 Execution。
所有正式结果必须 AlertEvent。
```

---

## 24. Execution 模块

### 24.1 职责

负责统一执行 PreparedExecutionRequest / ApprovedOrderIntent。

支持模式：

```text
dry-run
paper trading
real trading
```

### 24.2 输入对象

```text
PreparedExecutionRequest
ApprovedOrderIntent
execution_mode
交易所配置
trace_id
trigger_source
```

### 24.3 输出对象

```text
ExecutionResult
ExchangeOrder
```

说明：

```text
ExecutionResult 记录执行尝试结果，不等同于交易所订单状态。
ExchangeOrder 记录交易所订单状态，不等同于成交记录。
```

### 24.4 允许调用

```text
execution gateway
订单存储 service
AlertEvent 存储 service
```

### 24.5 禁止调用

```text
策略模块反向修改
原子信号
大模型
Hermes 触发交易
修改杠杆接口
修改 margin type 接口
```

### 24.6 失败处理

```text
执行失败必须记录状态、错误码和 trace_id。
订单状态不明时必须进入 Tracking / 异常同步流程。
不得重复提交同一个 ApprovedOrderIntent。
真实下单成功、失败、未知都必须 AlertEvent。
```

---

## 25. Tracking / 订单成交仓位模块

### 25.1 职责

负责记录和追踪订单、成交、仓位变化。

P0 可以使用 REST 查询确认订单状态。P1/P2 可接入 User Data Stream / WebSocket 监听订单事件。

### 25.2 输入对象

```text
ExecutionResult
交易所订单回报
交易所成交回报
交易所仓位回报
execution_mode
trace_id
```

### 25.3 输出对象

```text
ExchangeOrder
TradeFill
OrderStatusRecord
PositionState
PositionSyncRecord
```

说明：

```text
TradeFill 记录真实成交，不等同于仓位状态。
PositionState 记录仓位事实。
```

### 25.4 允许调用

```text
execution gateway / order query
成交查询 service
仓位读取 service
AlertEvent 存储 service
```

### 25.5 禁止调用

```text
策略模块
原子信号
修改杠杆
自动改策略
自动恢复真实交易
```

### 25.6 失败处理

```text
订单状态不明必须告警。
部分成交必须记录。
成交记录不得覆盖 DecisionSnapshot。
仓位不同步必须告警，并阻断后续真实交易。
paper trading 与 real trading 必须隔离。
```

---

## 26. Hermes / notifications 模块

### 26.1 职责

负责发送系统状态、交易状态、异常和复盘通知。

Hermes 是通知出口，不是交易决策模块。

### 26.2 输入对象

```text
AlertEvent
业务事件
异常事件
OrderStatusRecord
PositionState
ReviewRecord
trace_id
```

### 26.3 输出对象

```text
HermesNotification
```

### 26.4 允许调用

```text
notifications service
Hermes client
```

### 26.5 禁止调用

```text
Execution
交易所接口
StrategySignal
CandidateOrderIntent / ApprovedOrderIntent 生成
大模型实时交易
```

### 26.6 失败处理

```text
通知失败必须记录。
通知失败不得改变交易事实。
通知内容不得表达为人工下单请求。
```

---

## 27. Review 模块

### 27.1 职责

负责根据 trace_id、DecisionSnapshot ID 或交易生命周期 ID 聚合交易前、交易中、交易后的完整证据链。

### 27.2 输入对象

```text
trace_id
DecisionSnapshot ID
OrderPlan ID
RiskCheckResult ID
ExecutionResult ID
ReviewScope
```

### 27.3 输出对象

```text
ReviewRecord
ReviewFinding
```

### 27.4 允许调用

```text
历史数据查询 service
审计查询 service
大模型复盘辅助模块
```

### 27.5 禁止调用

```text
Execution
交易所接口
自动修改生产策略
自动调参
```

### 27.6 失败处理

```text
复盘失败必须记录。
复盘失败不得影响已发生交易事实。
复盘结论不得自动进入生产策略。
```

---

## 28. ModelReview 模块

### 28.1 职责

负责基于 ReviewRecord 进行事后解释、总结和研究辅助。

### 28.2 输入对象

```text
ReviewRecord
审计摘要
异常日志摘要
trace_id
```

### 28.3 输出对象

```text
ModelReviewRecord
复盘解释
优化建议草案
```

### 28.4 允许调用

```text
模型服务
复盘查询 service
```

### 28.5 禁止调用

```text
实时交易链路
RiskCheck 放行逻辑
OrderPlan
CandidateOrderIntent
ApprovedOrderIntent
Execution
生产策略自动修改
```

### 28.6 失败处理

```text
模型调用失败只影响解释，不影响交易事实。
基础告警不得依赖大模型。
```

---

## 29. 配置与安全模块

### 29.1 职责

负责配置来源、危险开关、密钥边界和配置检查。

### 29.2 输入对象

```text
.env
Django settings
策略配置文件
风险配置
execution_config
trace_id
```

### 29.3 输出对象

```text
ConfigSnapshot
ConfigCheckResult
```

### 29.4 允许调用

```text
配置读取 service
安全检查 service
```

### 29.5 禁止调用

```text
业务代码硬编码密钥
自动开启真实交易
自动修改杠杆
```

### 29.6 失败处理

```text
危险配置异常必须阻断真实交易。
缺少必要配置时不得降级为不安全默认值。
dry-run、paper trading、real trading 必须显式配置。
从 paper trading 切换到 real trading 必须人工确认。
```

---

## 30. 调度与幂等模块

### 30.1 职责

负责周期任务调度、任务入口和幂等控制。

幂等控制必须重点覆盖：

```text
同一分析周期不得重复生成有效 DecisionSnapshot。
同一 DecisionSnapshot 不得重复生成多个有效 OrderPlan。
同一 OrderPlan 不得重复生成多个有效 CandidateOrderIntent。
同一 CandidateOrderIntent 不得重复生成多个有效 RiskCheckResult。
同一 ApprovedOrderIntent 不得重复提交交易所。
```

### 30.2 输入对象

```text
调度配置
trace_id
trigger_source
交易品种
时间周期
分析时刻
任务参数
```

### 30.3 输出对象

```text
TaskRunRecord
IdempotencyRecord
幂等键
```

### 30.4 允许调用

```text
Celery
Redis 锁
数据库幂等记录
各模块 service 入口
```

### 30.5 禁止调用

```text
直接写复杂业务逻辑
绕过 service 调用模块内部实现
绕过风控触发交易
```

### 30.6 失败处理

```text
失败任务不得无限重试。
重复任务不得导致重复 CandidateOrderIntent、重复 ApprovedOrderIntent 或重复下单。
```

推荐幂等键：

```text
分析类任务：(symbol, timeframe, analysis_close_time_utc)
DecisionSnapshot：(strategy_signal_id, decision_policy_version, analysis_close_time_utc)
BinanceSyncRun：(exchange, market_type, started_at_utc, trigger_source, trace_id)
PriceSnapshot：(symbol, market_type, account_domain, source, as_of_utc, mark_price)
OrderPlan：(decision_snapshot_id, binance_sync_run_id, price_snapshot_id, config_hash)
CandidateOrderIntent：(order_plan_id, intent_type, side, size, components_hash)
RiskCheckResult：(candidate_order_intent_id, binance_sync_run_id, price_snapshot_id, rule_set_hash, risk_config_hash)
ApprovedOrderIntent：(risk_check_result_id, selected_candidate_order_intent_id)
Execution 提交：(approved_order_intent_id, execution_mode, client_order_id)
人工触发任务：(manual_request_id 或 confirm_token)
```

`trace_id` 用于追踪链路，不应单独作为业务幂等键。

---

## 31. 监控与异常恢复模块

### 31.1 职责

负责发现异常、告警、基础设施级自动恢复和业务级人工恢复入口。

恢复分两类：

```text
基础设施级恢复：网络重连、WebSocket 重连、失败查询重试、任务有限重试。可以自动化。
业务级恢复：仓位不一致、订单状态未知、风控熔断解除、失败订单意图重放。必须人工确认。
```

### 31.2 输入对象

```text
TaskRunRecord
OrderStatusRecord
PositionSyncRecord
异常事件
AlertEvent
trace_id
```

### 31.3 输出对象

```text
AlertRecord
RecoveryRecord
```

### 31.4 允许调用

```text
通知 service
状态查询 service
人工确认入口
基础设施重试入口
```

### 31.5 禁止调用

```text
绕过风控恢复交易
自动修改杠杆
自动修改生产策略
业务级异常自动恢复交易
```

### 31.6 失败处理

```text
恢复操作必须记录 trace_id、触发来源、确认参数和执行结果。
人工恢复不得绕过审计。
```

典型人工恢复操作包括：

```text
强制同步订单状态
强制同步仓位状态
取消卡住的本地订单状态
重放失败但未执行的 ApprovedOrderIntent
清除人工确认后的风控熔断标记
标记某次异常为已处理
```

---

## 32. 第一阶段最小闭环模块组合

第一阶段最小闭环必须包含：

```text
数据采集
数据质量
数据回补边界
MarketSnapshot
FeatureLayer
AtomicSignal
DomainSignal
MarketRegime
StrategyRoute
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
Execution dry-run / paper trading / real trading 开关
Tracking
AlertEvent / notifications
Review
审计与 trace_id
调度与幂等
配置与安全
```

可以简化但不能删除：

```text
DomainSignal
MarketRegime
StrategyRoute
StrategyParamSnapshot
Binance Account Sync
PriceSnapshot
OrderPlan
RiskCheck
ExecutionPreparation
Tracking
Review
```

第一阶段不进入主链路或只做预留：

```text
ModelReview
事件日历
复杂策略评估
多策略组合
多交易所
多品种
WebSocket PriceFeed
User Data Stream
```

---

## 33. 后续文档承接

本文档之后，需要继续细化或保持同步：

```text
docs/architecture/data_flow.md
docs/architecture/system_architecture.md
docs/rules/project_invariants.md

docs/requirements/decision_snapshot.md
docs/plans/006A_decision_snapshot_plan.md

docs/requirements/binance_account_sync.md
docs/plans/006B_binance_account_sync_plan.md

docs/requirements/price_snapshot.md
docs/plans/006C_price_snapshot_plan.md

docs/requirements/order_plan.md
docs/plans/007A_order_plan_plan.md

docs/requirements/risk_check.md
docs/plans/007B_risk_check_plan.md

以下后续 requirements 尚未创建，属于后续阶段补齐，不阻塞当前 001/002 阶段。

后续：
docs/requirements/approved_order_intent.md
docs/requirements/execution_preparation.md
docs/requirements/execution.md
docs/requirements/tracking.md
docs/requirements/review_engine.md
```

后续任何模块级文档不得突破本文档定义的模块边界。
