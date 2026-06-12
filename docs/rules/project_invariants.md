# project_invariants.md

# 项目不变量

## 1. 目的

本文档定义本项目所有阶段必须遵守的最高级别边界。

本文档优先级高于普通需求文档、架构文档、开发计划、Codex 指令和具体代码实现。后续任何设计、实现、测试、部署、调度、交易、通知和复盘，都不得违反本文档。

如果代码、计划、需求、架构文档与本文档冲突，以本文档为准。

本文档不定义具体策略公式、具体数据库字段、具体 Django app 结构、具体 Celery task 名称、具体交易所 API endpoint、具体回测撮合细节和具体代码实现。

具体实现由后续 requirements、architecture、plans 和 implementation 文档补充，但不得违反本文档。

当前正式交易链路为：

```text
StrategySignal
→ DecisionSnapshot
→ PriceSnapshot / Binance Account Sync
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation
→ Execution
→ Tracking / Review
```

本文档中的交易对象边界、风控边界、执行边界和告警边界均以该正式链路为准。

---

## 2. 项目定位不变量

本项目是一个中低频趋势跟踪自动交易系统。

系统第一阶段以 Binance BTCUSDT U 本位合约为主，同时文档和模型设计必须为 COIN-M / 币本位合约保留明确隔离边界。

数据分析域 P0 只使用 usds_m_futures / BTCUSDT；交易执行域的 Binance Account Sync / PriceSnapshot / OrderPlan / RiskCheck P0 必须支持 usds_m_futures 与 coin_m_futures 的 market_type 隔离和计算差异。

系统目标是构建一个可回测、可模拟、可实盘、可追溯、可审计、可复盘的自动交易闭环。

本项目允许自动交易，但所有真实交易必须受到以下约束：

```text
结构化策略决策
账户事实读取
价格事实快照
OrderPlan 订单规划
CandidateOrderIntent 候选订单意图
RiskCheck 风控审查
ApprovedOrderIntent 风控通过订单意图
ExecutionPreparation 执行前准备
Execution 交易执行边界
真实交易开关
AlertEvent / 审计日志
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

## 3. 文档优先级不变量

文档优先级如下：

```text
project_invariants.md
> decisions
> requirements
> architecture
> plans
> implementation
> code
```

如果文档之间冲突，以更高优先级文档为准。

如果后续确实需要修改本文档，必须作为单独架构决策处理，不能在普通功能开发中顺手修改。

---

## 4. 技术版本不变量

本项目技术底座必须与 AGENTS.md 声明保持一致。

当前约束：

```text
Python 3.12.x
Django 5.2.x LTS
MySQL
Redis
Celery
Celery Beat
```

版本约束：

```toml
requires-python = ">=3.12,<3.13"
```

```text
Django>=5.2,<5.3
celery>=5.6,<5.7
```

不得随意升级或降级 Python、Django、Celery 等核心版本。

修改核心版本前，必须先有架构决策文档，并完成兼容性验证。

依赖版本应在 `pyproject.toml` 中使用兼容范围，锁文件固定实际安装版本。

---

## 5. 最高交易红线

以下规则任何时候不得违反：

```text
策略模块不得直接下单。
原子信号不得直接下单。
特征层不得生成交易信号。
市场分析模块不得直接下单。
Hermes 不得触发交易。
大模型不得参与实时交易决策。
DecisionSnapshot 不得直接生成交易所订单动作。
DecisionSnapshot 不得读取账户、余额、持仓或 BinanceSyncRun。
不得绕过 OrderPlan。
不得绕过 CandidateOrderIntent。
不得绕过 RiskCheck。
不得绕过 ApprovedOrderIntent。
不得绕过 ExecutionPreparation。
不得绕过 Execution。
OrderPlan 不得真实下单。
RiskCheck 不得真实下单。
ExecutionPreparation 不得真实下单。
真实交易默认关闭，必须显式配置开启。
系统不得自动调整杠杆。
系统不得自动调整保证金模式。
paper trading 状态不得污染 real trading。
所有正式订单相关事件必须写 AlertEvent。
```

真实交易必须经过项目文档定义的完整交易链路。

如果文档没有明确允许真实交易，默认不得执行真实交易。

禁止将以下表达作为正式设计、实现、测试或 Codex 指令依据：

```text
DecisionSnapshot → RiskCheck → OrderIntent
DecisionSnapshot.decision_action = ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE
PositionSizingResult 直接进入 RiskCheck 并生成 OrderIntent
```

任何实现、测试、文档和 Codex 指令均不得恢复上述合同。

---

## 6. 交易对象边界不变量

系统必须严格区分以下对象：

```text
DecisionSnapshot 不等于 CandidateOrderIntent。
DecisionSnapshot 不等于 ApprovedOrderIntent。
DecisionSnapshot 不等于 ExchangeOrder。
OrderPlan 不等于 RiskCheck。
OrderPlan 不等于 Execution。
CandidateOrderIntent 不等于 ApprovedOrderIntent。
ApprovedOrderIntent 不等于 ExchangeOrder。
ExchangeOrder 不等于 TradeFill。
TradeFill 不等于 PositionState。
RiskCheck 不等于 CandidateOrderIntent。
RiskCheck 不等于 ApprovedOrderIntent。
RiskCheck 不等于 ExecutionPreparation。
Binance Account Sync 不等于交易决策。
PriceSnapshot 不等于成交价。
ExecutionPreparation 不等于真实下单。
```

不得把策略判断、目标仓位、候选订单、风控结果、风控通过订单意图、交易所订单、真实成交和仓位状态混为一个对象。

### 6.1 DecisionSnapshot 边界

DecisionSnapshot 只表达策略周期的目标仓位意图。

DecisionSnapshot 不得：

```text
读取当前账户余额。
读取当前交易所持仓。
读取 BinanceSyncRun。
读取 PriceSnapshot。
生成 BUY / SELL 真实订单参数。
生成 ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE 作为订单动作。
直接进入 RiskCheck。
直接进入 Execution。
```

DecisionSnapshot 可以输出：

```text
target_intent
target_position_ratio
policy_version
evidence_summary
```

### 6.2 OrderPlan 边界

OrderPlan 是唯一允许把 DecisionSnapshot 的目标仓位转换为 CandidateOrderIntent 的模块。

OrderPlan 可以读取：

```text
DecisionSnapshot
Binance Account Sync 快照
PriceSnapshot
当前持仓快照
symbol rule
```

OrderPlan 必须输出可审计的：

```text
OrderPlan
CandidateOrderIntent
order_components
fallback_reduce_only_intent（仅在净额反手需要时）
```

OrderPlan 不得：

```text
真实下单。
撤单。
查询订单成交。
修改杠杆。
修改保证金模式。
最终风控放行。
```

### 6.3 RiskCheck 边界

RiskCheck 只检查 CandidateOrderIntent，不直接检查 DecisionSnapshot。

RiskCheck 必须基于：

```text
CandidateOrderIntent
order_components
OrderPlan
Binance Account Sync 快照
PriceSnapshot
RiskRuleDefinition / risk_config
```

RiskCheck 输出：

```text
ALLOW
DENY
BLOCKED
FAILED
```

RiskCheck 不得任意 MODIFY 候选订单。P0 只允许选择 OrderPlan 预生成的 `fallback_reduce_only_intent`。

### 6.4 价格事实边界

PriceSnapshot 是价格事实层。

OrderPlan、RiskCheck、ExecutionPreparation 不得直接依赖 WebSocket 内存态价格。

价格来源可以来自：

```text
Account Sync mark_price
REST mark price
WebSocket price feed
manual fixture
```

但下游必须通过 PriceSnapshot 或 PriceSnapshotService 消费价格事实。

---

## 7. 数据可信度不变量

市场数据必须来自可信数据源或明确记录的数据源。

第一阶段以 Binance BTCUSDT U 本位合约数据为主。

禁止：

```text
人工伪造 K 线。
人工修复 K 线。
手工改写 K 线。
将不合格数据静默送入策略链路。
数据质量检查失败后继续生成正式交易决策。
```

如果发现 K 线缺失、重复、异常或断档，应通过数据质量检查、重新采集、数据回补、事件日志和告警处理。

数据回补不得伪造 K 线。

回补后的数据必须重新进入数据质量检查。

所有业务判断、K 线连续性、策略运行、DecisionSnapshot、TradeLifecycle、订单状态、仓位状态、复盘与归因，统一使用 UTC 时间。


---

## 8. 质量检查不变量

关键质量检查不得被绕过。

至少包括：

```text
数据质量检查
特征质量检查
原子信号质量检查
领域模块质量检查
策略信号质量检查
DecisionSnapshot 可用性检查
PriceSnapshot 新鲜度 / hash 检查
OrderPlan 输入一致性检查
CandidateOrderIntent 结构检查
风控审查
ExecutionPreparation 执行前检查
```

禁止：

```text
不合格数据进入正式策略链路。
FeatureQualityCheck 未通过的特征进入原子信号。
AtomicSignal 未通过质量检查进入领域模块。
DomainSignal 未通过质量检查进入市场环境识别、策略路由或策略信号层。
StrategySignalQuality 未通过却生成可交易 DecisionSnapshot。
DecisionSnapshot 过期或不可用仍进入 OrderPlan。
PriceSnapshot 缺失或明显过期仍用于 OrderPlan / RiskCheck。
OrderPlan 输入不一致仍生成 CandidateOrderIntent。
CandidateOrderIntent 结构不合法仍进入 RiskCheck。
RiskCheck 未通过仍生成 ApprovedOrderIntent。
ExecutionPreparation 未通过仍提交真实订单。
```

严重质量问题必须记录状态、错误码、错误消息、trace_id 和 trigger_source。

---

## 9. 账户、价格、订单规划与风控不变量

Binance Account Sync 只表示账户事实，不是交易决策。

PriceSnapshot 只表示某一时刻的价格事实，不是成交价，也不是策略判断。

OrderPlan 负责把目标仓位转换为 CandidateOrderIntent，但不负责最终风控放行。

RiskCheck 是 CandidateOrderIntent 进入 ApprovedOrderIntent 前的强制闸门。

真实交易前必须满足：

```text
DecisionSnapshot 可用且未过期。
BinanceSyncRun 显式传入且可消费。
账户、余额、持仓、symbol rule 快照可追溯。
PriceSnapshot 可用且满足对应阶段的新鲜度要求。
OrderPlan 已生成 CandidateOrderIntent。
CandidateOrderIntent 结构合法且可追溯。
RiskCheck 通过。
ApprovedOrderIntent 可追溯且未过期。
ExecutionPreparation 通过。
真实交易开关已显式开启。
Execution 可审计。
```

RiskCheck 失败、拒绝或阻断时，不得生成 ApprovedOrderIntent。

任何 ApprovedOrderIntent 必须能追溯到：

```text
RiskCheckResult
CandidateOrderIntent
OrderPlan
DecisionSnapshot
BinanceSyncRun
PriceSnapshot
```

任何 RiskCheck 必须能追溯到 CandidateOrderIntent，不得只追溯到 DecisionSnapshot。

### 9.1 account_domain / market_type 隔离

账户事实必须按以下维度严格隔离：

```text
exchange
market_type
account_domain
symbol
```

禁止：

```text
U 本位余额不足时读取 COIN-M 余额兜底。
COIN-M 缺 margin_asset 余额时读取 U 本位 USDT 兜底。
跨 market_type 使用持仓快照。
跨 account_domain 使用 symbol rule。
```

### 9.2 observed_exchange_leverage 边界

`observed_exchange_leverage` 是 Binance 当前 symbol 实际观测到的交易所杠杆设置。

它不得用于计算：

```text
系统全仓。
系统半仓。
target_notional。
target_quantity。
target_contracts。
```

系统目标仓位只能由 OrderPlan 按文档定义的目标仓位公式计算。

RiskCheck 可以使用 `observed_exchange_leverage` 估算 increase_risk 组件的新增保证金。

`observed_exchange_leverage` 缺失时不得伪造，不得用系统配置值或 `max_target_notional_to_equity_ratio` 填充。

reduce_risk 组件不得因为 `observed_exchange_leverage` 缺失而被直接阻断。

---

## 10. 杠杆不变量

任何时候都不允许系统自动调整杠杆。

规则：

```text
系统不得调用交易所修改杠杆接口。
系统不得根据行情、策略、回测或复盘结果自动改变杠杆。
系统不得在 OrderPlan / RiskCheck / ExecutionPreparation 中修改杠杆。
如果代码中新增修改杠杆能力，视为违反本文档。
```

杠杆相关字段必须区分：

```text
max_target_notional_to_equity_ratio
= 系统内部目标仓位名义 / 当前权益比例。
= 属于 OrderPlan 目标仓位计算参数。
= 不是 Binance 交易所杠杆设置。

observed_exchange_leverage
= Binance 当前 symbol 实际观测到的交易所杠杆设置。
= 来自 Binance Account Sync / positionRisk。
= 只能用于保证金估算、环境审计和 evidence。
= 不得参与目标仓位计算。

configured_exchange_leverage
= 如项目后续定义该配置，只能作为交易环境审计或一致性检查字段。
= 不得用于放大系统目标仓位。
= 不得触发自动修改 Binance 杠杆。
```

如果后续需求要求校验交易所实际杠杆与配置一致，校验失败时只能：

```text
阻断真实交易。
写 AlertEvent。
等待人工处理。
```

不得由系统自动调用 Binance 修改杠杆。

---

## 11. 交易执行不变量

Execution 是唯一允许提交真实订单的模块。

Execution / Tracking 是唯一允许处理真实订单提交、撤单、订单状态查询、成交同步和订单追踪的交易执行相关模块。

交易所交易类 API 访问必须通过统一 exchange_gateway / execution 相关模块。

策略模块、特征层、原子信号、领域模块、市场环境识别、策略路由、DecisionSnapshot、OrderPlan、RiskCheck、ExecutionPreparation、Hermes、大模型，均不得直接访问交易所交易接口。

CandidateOrderIntent、ApprovedOrderIntent、ExchangeOrder、TradeFill、PositionState 必须保持边界清晰：

```text
CandidateOrderIntent = 候选订单意图，尚未通过风控。
ApprovedOrderIntent = 已通过风控的订单意图，尚未真实下单。
ExchangeOrder = 交易所订单状态记录。
TradeFill = 真实成交记录。
PositionState = 仓位状态记录。
```

所有 ApprovedOrderIntent 必须追溯到 RiskCheckResult。

所有 ExchangeOrder 必须追溯到 ApprovedOrderIntent。

所有 PositionState 变更必须追溯到 TradeFill 或交易所同步记录。

同一个 ApprovedOrderIntent 不得重复提交为多个有效交易所订单。

订单状态不明时必须进入异常同步或人工恢复流程，不得静默假设成功或失败。

### 11.1 ExecutionPreparation 边界

ExecutionPreparation 只做执行前准备和最终保护。

它可以检查：

```text
ApprovedOrderIntent 是否可执行。
最新 PriceSnapshot 是否足够新。
价格偏离是否在允许范围内。
真实交易开关是否开启。
active order 是否冲突。
```

ExecutionPreparation 不得真实下单。

---

## 12. 执行模式隔离不变量

系统执行模式至少包括：

```text
dry-run
paper trading
real trading
```

必须遵守：

```text
dry-run 不得生成真实订单、真实成交和真实仓位。
paper trading 的订单、成交、仓位必须与 real trading 隔离。
real trading 只能使用交易所真实订单、成交和仓位。
paper trading 的虚拟仓位不得参与 real trading 的资金管理、风控和仓位判断。
执行模式切换必须人工确认。
从 paper trading 切换到 real trading 前，必须清理或隔离 paper 状态。
```

任何执行模式都必须记录 trace_id、trigger_source、execution_mode 和结果状态。

---

## 13. 回测与实盘一致性不变量

回测和实盘必须尽量共享同一套核心逻辑。

回测不得使用实盘不可获得的数据。

必须避免：

```text
look-ahead bias（前视偏差 / 未来函数）
survivorship bias（幸存者偏差）
overfitting（过拟合）
未考虑手续费
未考虑 slippage（滑点）
未考虑成交约束
未考虑资金费率
未考虑最小下单量
未考虑交易规则
未考虑杠杆和保证金风险
未考虑 PriceSnapshot / 执行前价格偏离保护
```

原则：

```text
如果实盘只能在 K 线收盘后得到信号，回测也必须在 K 线收盘后生成信号。
如果实盘下一根 K 线才能执行，回测也不得假设当前 K 线收盘价完美成交。
如果实盘需要经过 OrderPlan、RiskCheck、ExecutionPreparation，回测也必须经过等价链路或明确标记的等价模拟逻辑。
如果实盘使用 FeatureLayer，回测也必须使用同一套特征定义。
```

不得为了得到更好回测结果而改变实盘不可实现的假设。

---

## 14. 复盘与优化不变量

复盘模块通过 trace_id、DecisionSnapshot ID 或 TradeLifecycle ID 聚合证据链。

复盘必须服务审计、归因和策略研究。

复盘不得自动修改生产策略。

策略评估不得自动修改生产策略。

任何策略优化建议必须经过：

```text
人工确认
回测验证
文档更新
风险复核
```

才能进入生产策略。

禁止系统根据复盘结果自动降权、自动禁用、自动调参或自动上线策略。

---

## 15. 大模型不变量

大模型仅允许用于事后复盘、解释、归因和策略研究辅助。

大模型禁止参与：

```text
实时判断是否开仓。
实时判断是否平仓。
实时决定仓位大小。
实时修改止损止盈。
实时生成 CandidateOrderIntent。
实时生成 ApprovedOrderIntent。
实时修改 CandidateOrderIntent。
实时修改 ApprovedOrderIntent。
影响 RiskCheck 是否放行。
绕过结构化风控。
根据自然语言直接生成订单。
自动修改生产策略。
```

交易前和交易中由规则系统负责。

交易后可以由大模型辅助解释。

基础告警不得调用大模型生成，必须由固定模板和代码逻辑生成。

所有大模型调用必须可追踪。

---

## 16. Hermes 通知不变量

Hermes 是通知出口 / 人机交互出口，不是交易决策模块。

Hermes 不得：

```text
参与交易决策。
生成 CandidateOrderIntent。
生成 ApprovedOrderIntent。
触发真实下单。
触发撤单。
触发平仓。
触发加仓。
触发减仓。
```

Hermes 通知必须通过统一 notifications 模块发送。

禁止在业务模块中反复封装 Hermes 请求。

Hermes 通知必须明确表达这是系统状态通知，不是人工下单请求。

Hermes 相关配置、超时、重试、真实发送开关、dry-run 行为、错误记录和 trace_id 传递，必须集中管理。

---

## 17. MySQL 主存储不变量

MySQL 是核心业务主存储。

以下数据必须以 MySQL 或等价可靠主存储为准：

```text
市场数据
价格快照 PriceSnapshot
特征结果
原子信号结果
领域模块结果
策略信号
DecisionSnapshot
BinanceSyncRun / 账户事实快照
OrderPlan
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
ExecutionPreparation 记录
ExchangeOrder
TradeFill
PositionState
TradeLifecycle
ReviewRecord
AlertEvent
审计记录
```

Redis 不得作为上述核心业务数据的唯一存储。

核心业务数据必须可追溯、可审计、可复盘。

---

## 18. 时间序列与结构化存储不变量

时间序列数据必须结构化存储。

包括但不限于：

```text
K 线
价格
指标序列
FeatureValue
AtomicSignal
DomainSignal
DecisionSnapshot
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
订单状态
成交记录
仓位状态
复盘记录
```

禁止将以下内容整体塞进单个字段：

```text
几百根 K 线
完整 K 线窗口
完整指标序列
完整策略运行窗口
完整历史数组
完整订单状态历史
完整仓位变化历史
```

正确方向：

```text
一根 K 线一行
一个 FeatureValue 一行
一次 AtomicSignal 结果一行
一次 DomainSignal 结果一行
一次 StrategySignal 一行
一次 DecisionSnapshot 一行
一次 PriceSnapshot 一行
一次 OrderPlan 一行
一次 CandidateOrderIntent 一行
一次 RiskCheckResult 一行
一次 ApprovedOrderIntent 一行
一次 ExecutionPreparation 记录一行
一次 ExchangeOrder 状态记录一行
一次 TradeFill 一行
一次 PositionState 一行
一次 ReviewRecord 一行
```

如果某项数据无法结构化保存，必须在 architecture 或 plans 中说明原因、边界和替代审计方式。

---

## 19. JSON 字段不变量

JSON 字段只能用于保存摘要、配置快照、少量结构化补充信息或私有小 payload。

允许保存：

```text
摘要
配置快照
少量 evidence_items
少量 display_items
少量 fingerprint_items
错误详情
追溯信息
少量 payload_summary
```

禁止使用 JSON 字段保存：

```text
大批量 K 线
大批量特征值
大批量原子信号明细
完整模型原文
完整长窗口计算过程
无法拆分的核心业务数据
```

JSON 字段不得变成逃避表结构设计的垃圾桶。

下游主链路不得硬编码依赖某个私有 payload_json 的内部字段。

---

## 20. 大字段与长文本不变量

普通业务字段必须控制大小。

任何普通业务字段原则上不得超过：

```text
32KB
或 10000 字符
```

超过该范围必须人工审核，并满足以下条件之一：

```text
拆表
拆行
摘要化
隔离存储
保存 artifact 引用
保存 hash / path / storage_ref
```

禁止为了省事直接改成 LONGTEXT、大 JSON 或无限制 TextField。

如确实需要 LONGTEXT、MEDIUMTEXT、超大 JSONField 或无限制 TextField，必须在对应 plan 或 architecture decision 中说明：

```text
为什么不能拆表
最大预期长度
是否进入主链路
是否只用于审计 / 归档
是否有 hash
是否有隔离存储引用
是否有清理策略
```

大模型原始返回、复杂计算过程、长文本解释、长数组明细，不得直接污染主业务表。

主业务表只保存：

```text
状态
摘要
结论
关键证据
错误码
追溯 ID
hash
storage_ref
```

完整原始内容应进入隔离表、artifact 存储、归档存储或专用 raw 表。

---

## 21. 字段语义与命名不变量

数据库字段不得使用模糊命名作为核心业务字段，例如：

```text
data
info
content
extra
payload
result
```

除非它们是明确限定用途的补充字段，并在字段注释或文档中说明内容边界。

核心业务字段必须优先使用明确命名。

所有字符串字段应明确：

```text
最大长度
是否 nullable
是否唯一
是否索引
是否参与追溯
```

能使用 `CharField(max_length=...)` 的字段，不得随意使用 `TextField`。

---

## 22. Decimal 数值字段不变量

禁止无差别使用超大精度 Decimal 字段，例如：

```text
DECIMAL(38,18)
DECIMAL(30,18)
```

所有 Decimal 字段必须按业务含义选择 `max_digits` / `decimal_places`。

价格、数量、USDT 金额、比率、权重、分数、置信度、手续费、滑点、资金费率、保证金比例，应分别设计精度，不能全部使用同一套精度。

数据库保存应保证必要计算精度，但 CLI、日志、Hermes 通知、后台展示不得输出大量无意义尾零。

禁止为了省事把需要计算的数值保存为字符串。

JSON 字段不得保存大量无意义小数尾零。

---

## 23. 表和字段中文说明不变量

所有核心表和核心字段必须有中文注释或等价中文说明。

要求：

```text
表名、字段名使用英文。
注释使用中文。
注释说明业务含义、取值范围、是否参与追溯、是否可为空。
```

Django model 中应尽量使用 `verbose_name`、字段注释或 migration 注释机制表达中文含义。

如果数据库层无法完整保存字段注释，也必须在对应模型、文档或 migration 说明中保留中文解释。

---

## 24. Django Migration 不变量

数据库结构变更必须通过 Django migration。

禁止：

```text
手工改表后不生成 migration。
生产环境直接手工修改核心表结构。
migration 写入真实密钥。
migration 写入真实 Token。
migration 写入真实 Webhook Secret。
migration 写入真实交易所 API Key。
同时维护 Alembic 作为主迁移体系。
```

migration 文件必须可审查、可回滚、可解释。

删除表、删除字段、重命名核心字段，必须单独说明影响范围。

migration 涉及大字段、JSON 字段、索引、唯一约束时，必须说明设计原因。

---

## 25. Redis 不变量

Redis 只允许用于：

```text
缓存
分布式锁
Celery broker
Celery result backend
短期幂等控制
短期任务状态
限流计数
短期特征序列缓存
最近窗口缓存
```

Redis 禁止用于：

```text
长期保存核心市场数据
长期保存核心特征结果
长期保存核心策略结果
长期保存核心订单记录
长期保存核心仓位记录
长期保存复盘数据
替代 MySQL 作为主存储
```

Redis 中的数据必须可以过期、可以重建、可以丢失后由 MySQL 或外部可信数据源恢复。

---

## 26. 配置与密钥不变量

运行配置必须来自明确的配置文件、环境变量或 Django settings。

禁止在业务代码中硬编码：

```text
数据库密码
Redis 密码
Webhook Secret
模型 API Key
交易所 API Key
生产环境开关
真实发送开关
真实模型调用开关
真实交易开关
杠杆配置
```

`.env.example` 中所有配置项必须有中文注释。

真实交易、真实 Hermes、真实大模型、高成本模型、账户读取、仓位读取、订单读取、真实下单、真实撤单、真实交易所请求等危险开关，必须默认关闭，并且生产开启前必须人工确认。

禁止提交真实 `.env`、真实 API Key、真实 Token、真实 Webhook Secret 和真实交易所 API Key。

日志、异常信息和通知不得暴露密钥。

---

## 27. 外部请求不变量

所有外部请求必须有超时设置。

所有外部请求失败必须记录：

```text
错误码
错误消息
trace_id
trigger_source
```

所有外部请求不得无限重试。

所有外部请求不得泄露密钥。

所有外部请求的真实发送开关必须可配置。

交易所请求必须通过统一模块封装。

交易所请求必须区分读取类请求和交易类请求。

交易类请求必须有更严格的显式开关、幂等和审计记录。

---

## 28. Celery 与调度不变量

Celery task 只作为任务入口，复杂业务必须放在 service / domain 层。

Celery Beat 只能有一个正式调度实例，避免重复派发任务。

所有定时任务必须有幂等控制。

所有关键任务必须记录：

```text
运行状态
开始时间
结束时间
错误码
错误消息
trace_id
trigger_source
```

所有关键链路必须有 Redis 锁或数据库幂等兜底。

失败任务不得无限重试，必须有最大重试次数和失败记录。

调度任务不得绕过质量检查层。

调度任务不得绕过风控审查层。

交易执行相关 task 必须具备幂等键。

同一个 DecisionSnapshot 在相同账户事实、价格事实和配置下不得重复生成多个等价有效 OrderPlan。

同一个 OrderPlan 不得重复生成多个等价有效 CandidateOrderIntent。

同一个 CandidateOrderIntent 不得重复生成多个等价 RiskCheckResult / ApprovedOrderIntent。

同一个 ApprovedOrderIntent 不得重复提交为多个有效交易所订单。

重复调度不得导致重复下单。

---

## 29. 审计与 trace_id 不变量

所有关键链路必须保留 trace_id。

关键链路包括但不限于：

```text
数据采集
数据校验
数据回补
快照生成
特征计算
特征质量检查
原子信号运行
原子信号质量检查
领域模块运行
领域模块质量检查
市场环境识别
策略路由
策略信号生成
DecisionSnapshot 生成
Binance Account Sync
PriceSnapshot 生成
OrderPlan 生成
CandidateOrderIntent 生成
风控审查 RiskCheck
ApprovedOrderIntent 生成
ExecutionPreparation 执行前准备
Execution 执行
ExchangeOrder 同步
TradeFill 同步
PositionState 更新
TradeLifecycle 更新
AlertEvent
Hermes 通知
复盘与归因
大模型复盘辅助
后续优化建议
```

每次运行必须能追踪：

```text
输入是什么
输出是什么
使用哪个市场快照
使用哪个账户快照
使用哪个价格快照
调用哪些模块
是否通过质量检查
是否生成 DecisionSnapshot
是否生成 OrderPlan
是否生成 CandidateOrderIntent
是否触发 RiskCheck
是否生成 ApprovedOrderIntent
是否通过 ExecutionPreparation
是否提交交易所订单
是否产生真实成交
是否更新仓位
是否关联 TradeLifecycle
是否写 AlertEvent / Hermes 通知
是否调用大模型复盘
```

不得在未记录 trace_id、状态、错误码和来源链路的情况下静默失败。

所有正式订单相关事件不管成功还是失败，都必须写 AlertEvent。

---

## 30. trigger_source 不变量

所有关键任务和核心链路运行记录必须保存 trigger_source。

trigger_source 用于区分本次运行来源，例如：

```text
cli
celery_beat
celery_worker
admin
system
retry
manual_recovery
```

禁止在无法识别触发来源的情况下写入核心运行记录。

Celery worker 执行异步任务时，不得覆盖原始 trigger_source。

如果需要记录实际执行者，应另设字段，例如 executor_source。

所有下游链路必须继承或显式传递上游 trigger_source，不得在中途丢失。

涉及写库、真实 Hermes 发送、真实大模型调用、账户读取、仓位读取、订单读取、真实下单、真实撤单、外部高成本请求的人工触发，必须有显式确认参数，并在运行记录中保存确认状态。

---

## 31. 代码结构不变量

Django model 只定义数据结构，不堆业务逻辑。

service / domain 层承载业务核心逻辑。

Celery task 只调用 service，不写复杂业务。

management command 只作为人工命令入口，不写复杂业务。

admin 只做查看、辅助管理和低风险操作，不承载策略逻辑。

模块之间不得直接操作对方内部细节，优先通过 service 接口交互。

交易所 API 必须通过统一 exchange_gateway / execution 模块封装。

OrderPlan / RiskCheck / ExecutionPreparation 不得直接读取 WebSocket 内存态价格，必须通过 PriceSnapshot / PriceSnapshotService / PriceFeed 抽象消费价格事实。

Hermes 必须通过统一 notifications 模块封装。

大模型必须通过统一 model_review / review 相关模块封装。

新功能必须先明确文档、边界、状态、错误码和验收标准，再写代码。

---

## 32. 测试不变量

核心链路必须有自动化测试。

禁止只靠手工测试验证核心策略链路。

质量检查、阻断、幂等、重复运行、配置关闭、外部请求失败必须有测试。

涉及真实外部请求的测试必须默认关闭。

测试不得依赖真实密钥。

测试不得自动调用真实大模型。

测试不得自动发送真实 Hermes。

测试不得自动提交真实交易所订单。

测试不得自动撤销真实交易所订单。

测试不得自动修改杠杆。

交易相关测试默认使用 mock、dry-run 或 paper trading。

回测测试必须覆盖手续费、滑点、成交约束和 K 线收盘后信号生成规则。

风控测试必须覆盖 observed_exchange_leverage 缺失、increase_risk / reduce_risk 分支、COIN-M contract_size 缺失、PriceSnapshot 过期和 fallback_reduce_only。

OrderPlan 测试必须覆盖 U 本位、COIN-M、净额反手、order_components、fallback_reduce_only 和 active_order_exists。

ExecutionPreparation 测试必须覆盖最新价格 TTL、价格偏离、真实交易开关和 ApprovedOrderIntent 幂等。

---

## 33. Codex 实现不变量

Codex 在本项目中必须遵守：

```text
不创建 Git 分支。
不切换 Git 分支。
不提交代码。
不写入真实密钥。
不绕过本文档。
不把临时兼容逻辑伪装成正式实现。
不为了通过测试而删除关键业务校验。
不擅自改变项目核心架构链路。
不擅自删除质量检查层。
不擅自让大模型进入实时交易链路。
不擅自让 Hermes 触发交易。
不擅自让策略模块直接调用交易所接口。
不擅自新增自动调整杠杆能力。
不擅自引入 DecisionSnapshot → RiskCheck → OrderIntent 链路。
不擅自把 DecisionSnapshot 写成 ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE 订单动作。
不擅自让 OrderPlan / RiskCheck / ExecutionPreparation 直接下单。
不擅自让 OrderPlan / RiskCheck 直接依赖 WebSocket 内存态价格。
```

完成任务后必须说明是否违反本文档。

---

## 34. 时间不变量

Binance 返回的时间戳按 UTC 解释。
系统内部所有时间统一 UTC。
不得根据服务器本地时区、用户 IP、运行机器时区转换业务时间。
K 线 open_time / close_time 必须使用 Binance 返回的时间戳，并按 UTC 存储和判断。
请求 K 线时不传 timeZone，默认使用 UTC。

所有核心业务时间、行情时间、K 线排序、连续性判断、回测判断、策略周期判断、订单追踪、成交追踪、仓位追踪、复盘追踪，统一使用 UTC 时间。

系统不设计本地时间字段。

通知、日志、复盘、后台展示如需人类可读时间，也应默认展示 UTC，并明确标注 UTC。

不得使用本地时间、PRC 时间或运行机器时区参与任何业务判断。

---

## 35. 不做事项

本文档不定义具体数据表、字段、索引、任务名称、Celery 队列、Django app 结构和具体代码实现。

具体实现由后续 requirements、architecture、plans 和 implementation 文档补充，但不得违反本文档。
