# AGENTS.md

## 1. 项目定位

本项目是一个中低频趋势跟踪自动交易系统。

系统目标是构建一个从行情数据、特征、信号、目标仓位决策、订单计划、风控审批、执行、追踪、复盘到审计的自动交易闭环。

当前核心交易链路为：

```text
MarketSnapshot
→ FeatureLayer
→ AtomicSignal / DomainSignal / MarketRegime
→ StrategySignal
→ DecisionSnapshot
→ Binance Account Sync / PriceSnapshot
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation
→ Execution
→ Tracking / Review
```

本项目允许自动交易，但所有真实交易必须经过结构化策略决策、账户事实读取、价格事实确认、订单计划、风控审批、执行前最终校验、交易执行边界、配置开关、AlertEvent、日志审计和复盘追踪。

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

## 2. AGENTS.md 职责边界

本文档只定义 Codex 的工作纪律、文档优先级、开发安全规则和最高级禁止行为。

本文档不承载完整系统设计，不重复维护完整模块设计。

具体内容以以下文档为准：

```text
系统红线：docs/rules/project_invariants.md
项目范围：docs/requirements/01_project_scope.md
系统能力：docs/requirements/02_system_capabilities.md
系统架构：docs/architecture/*.md
模块需求：docs/requirements/*.md
开发计划：docs/plans/*.md
架构决策：docs/decisions/*.md
```

新增普通业务模块时，优先修改 requirements / architecture / plans。

只有当新增内容影响以下事项时，才修改 AGENTS.md：

```text
Codex 工作纪律
文档优先级
真实交易红线
最高禁止项
开发流程
回报要求
```

---

## 3. 技术底座

本项目技术底座：

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

不得随意升级或降级核心版本，除非先有架构决策文档并完成兼容性验证。

依赖版本应在 `pyproject.toml` 中使用兼容范围，锁文件固定实际安装版本。

---

## 4. 文档优先级

开发前必须先阅读并遵守：

```text
docs/rules/project_invariants.md
docs/requirements/*.md
docs/architecture/*.md
docs/decisions/*.md
docs/plans/*.md
```

文档优先级：

```text
project_invariants.md
> decisions
> requirements
> architecture
> plans
> implementation
> code
```

如果文档之间冲突，以更高优先级文档为准，并向用户说明冲突，不得自行猜测。

本项目以当前仓库文档为唯一实现依据。


如果 `docs/decisions` 或 `docs/plans` 目录尚未存在，表示该阶段尚未开始；Codex 不得因此自行猜测实现方案，也不得绕过已有 requirements / architecture 文档。

---

## 5. 最高交易红线

以下规则任何时候不得违反：

```text
策略模块不得直接下单。
原子信号不得直接下单。
特征层不得生成交易信号。
DecisionSnapshot 不得生成订单动作，不得直接读取账户、持仓或 BinanceSyncRun。
OrderPlan 是唯一把目标仓位转换为 CandidateOrderIntent 的模块。
RiskCheck 只审批 CandidateOrderIntent，不直接审批 DecisionSnapshot。
ApprovedOrderIntent 只是风控审批通过的订单意图，不等于交易所订单。
ExecutionPreparation 只做执行前最终检查和 price guard，不得真实下单。
Execution 是唯一允许提交真实订单的模块。
Hermes 不得触发交易。
大模型不得参与实时交易决策。
不得绕过 OrderPlan。
不得绕过 RiskCheck。
不得绕过 ApprovedOrderIntent。
不得绕过 ExecutionPreparation。
不得绕过 Execution。
真实交易默认关闭，必须显式配置开启。
系统不得自动调整杠杆。
paper trading 状态不得污染 real trading。
```

真实交易必须经过项目文档定义的完整交易链路。

如果文档没有明确允许真实交易，默认不得执行真实交易。

所有正式订单相关事件必须写 AlertEvent，包括但不限于：

```text
OrderPlan no_order_required / blocked / failed
CandidateOrderIntent generated / skipped / blocked
RiskCheck ALLOW / DENY / BLOCKED / FAILED
fallback_reduce_only selected
ApprovedOrderIntent generated / expired / canceled
ExecutionPreparation passed / blocked / failed
Execution submitted / failed / canceled / rejected
ExchangeOrder accepted / partially_filled / filled / expired / unknown
TradeFill recorded
PositionState changed
```

---

## 6. 核心对象边界

Codex 必须严格区分以下对象：

```text
DecisionSnapshot 不等于 CandidateOrderIntent。
DecisionSnapshot 不等于 ApprovedOrderIntent。
DecisionSnapshot 不等于 ExchangeOrder。
OrderPlan 不等于 RiskCheck。
CandidateOrderIntent 不等于 ApprovedOrderIntent。
ApprovedOrderIntent 不等于 ExchangeOrder。
ExecutionPreparation 不等于 Execution。
ExchangeOrder 不等于 TradeFill。
TradeFill 不等于 PositionState。
Binance Account Sync 不等于交易执行。
PriceSnapshot 不等于 WebSocket。
RiskCheck 不等于交易执行。
```

对象语义：

```text
DecisionSnapshot = 每个策略周期的目标仓位决策快照。
Binance Account Sync = 只读账户事实来源。
PriceSnapshot = 价格事实层。
OrderPlan = 目标仓位转换为候选订单意图。
CandidateOrderIntent = 待风控审批的候选订单意图。
RiskCheck = 对 CandidateOrderIntent 做风控审批。
ApprovedOrderIntent = 风控通过后的订单意图。
ExecutionPreparation = 执行前最终价格和时效检查。
Execution = 真实或模拟执行入口。
ExchangeOrder = 交易所订单状态记录。
TradeFill = 成交记录。
PositionState = 仓位状态记录。
```

不得把策略判断、目标仓位、候选订单意图、风控审批、执行前检查、交易所订单、真实成交和仓位状态混成一个对象。

完整核心对象清单以 requirements 和 architecture 文档为准，AGENTS.md 只保留红线级边界。

---

## 7. 正式交易链路工作纪律

Codex 实现交易链路相关模块时必须遵守：

```text
DecisionSnapshot 只表达 target_intent / target_position_ratio 等目标仓位语义。
DecisionSnapshot 不输出 ENTER_LONG / ENTER_SHORT / EXIT / HOLD / NO_TRADE 等订单动作。
OrderPlan 基于 DecisionSnapshot、Binance Account Sync、PriceSnapshot 生成 CandidateOrderIntent。
OrderPlan 不访问 Binance REST / WebSocket，不真实下单，不做最终风控审批。
RiskCheck 只消费 CandidateOrderIntent / OrderPlan，不消费 DecisionSnapshot。
RiskCheck 不生成新的 CandidateOrderIntent，不重新设计订单内容。
RiskCheck 在 ALLOW 时生成或确认 ApprovedOrderIntent；在 DENY / BLOCKED / FAILED 时不得生成 ApprovedOrderIntent。
RiskCheck 不真实下单，不撤单，不修改杠杆，不修改保证金模式。
RiskCheck P0 不做任意 MODIFY；只允许选择 OrderPlan 预生成的 fallback_reduce_only。
ApprovedOrderIntent 只能由 RiskCheck ALLOW 后生成或等价确认。
ExecutionPreparation 必须基于最新 PriceSnapshot 做 price guard。
Execution 是唯一允许调用 trading gateway 提交真实订单的模块。
Tracking 负责订单、成交、仓位追踪，不得反向修改策略判断。
```

价格相关规则：

```text
PriceSnapshot 是价格事实层。
WebSocket 后续只能作为 PriceFeed / PriceSnapshot 来源之一。
OrderPlan、RiskCheck、ExecutionPreparation 不得直接依赖 WebSocket 内部连接状态。
下游应读取 PriceSnapshot，而不是直接调用 WebSocket。
```

杠杆相关规则：

```text
max_target_notional_to_equity_ratio 属于 OrderPlan 目标仓位换算参数。
observed_exchange_leverage 属于 Binance Account Sync 观测到的交易所杠杆设置。
configured_exchange_leverage 属于系统配置 / 环境校验。
observed_exchange_leverage 不得参与 OrderPlan 目标仓位计算。
系统不得调用交易所修改杠杆接口。
```

---

## 8. 业务逻辑组织规则

业务逻辑必须放在：

```text
service 层
domain 层
```

不得把复杂业务逻辑堆进：

```text
Django model
Celery task
management command
view
serializer
```

Django model 只定义数据结构和最小约束。

Celery task 只作为任务入口。

Management command 只作为人工命令入口。

跨模块编排应放在 orchestration service 或明确的 service 层中。

---

## 9. 数据与存储红线

MySQL 是核心业务主存储。

Redis 只能用于：

```text
缓存
分布式锁
Celery broker
短期幂等控制
短期任务状态
限流计数
短期特征序列缓存
Celery result backend（如启用）
```

禁止把 Redis 作为核心业务数据唯一存储。

禁止在单个字段中保存：

```text
大批量 K 线
完整历史窗口
完整历史指标数组
不可控长文本
逃避表结构设计的大 JSON
```

交易、风控、订单、成交、仓位、复盘相关数据必须可追溯、可审计、可复盘。

---

## 10. 配置与密钥规则

所有环境配置必须进入 `.env.example`，并带中文注释。

禁止硬编码：

```text
数据库密码
Redis 密码
Webhook Secret
模型 API Key
交易所 API Key
真实交易开关
真实发送开关
生产环境开关
杠杆配置
```

禁止提交真实 `.env`、真实 API Key、真实 Token、真实 Webhook Secret。

日志、异常信息、Hermes 通知不得暴露密钥。

---

## 11. 文件修改安全规则

修改已有文件前必须先读取原文件内容。

禁止：

```text
清空重写已有核心文件
删除整个 docs/、apps/、config/、tests/ 目录
用脚手架命令覆盖已有项目结构
大范围格式化与当前任务无关的文件
为了通过测试而删除测试
为了通过测试而降低测试强度
绕过业务规则
新增重复模块
新增重复工具类
新增重复封装
新增重复配置
```

新增文件必须符合当前目录结构和文档约定。

---

## 12. 开发纪律

每次修改必须围绕用户本次任务。

禁止：

```text
顺手重构无关模块
顺手改命名
顺手优化相邻代码
顺手新增计划外功能
顺手调整架构
提前实现超出当前 plans 的能力
```

如果发现当前任务需要突破已有文档边界，必须停止并向用户说明原因。

如果存在不确定字段、表名、配置名、流程边界、交易行为或真实交易风险，必须停止并向用户确认。

不得自行猜测业务规则。

---

## 13. 函数与职责规则

```text
不按文件总行数机械限制拆分。
单个函数 / 方法原则上不超过 120 行。
超过 180 行需要在回报中说明原因。
超过 250 行原则上应拆分，除非用户明确允许。
一个函数只做一个清晰职责。
一个 class / service 不得承担多个架构层职责。
禁止把完整主链路塞进一个 service、task 或 management command。
不得为了满足行数规则进行无意义拆分。
```

优先保证职责清晰、调用链清楚、测试容易。

---

## 14. Management command 与 Celery task 规则

management command 只作为人工命令入口。

Celery task 只作为异步任务入口。

二者只能：

```text
解析参数
生成或传递 trace_id
设置 trigger_source
调用 service
输出结果
```

禁止在 command 或 task 中写复杂业务逻辑。

如果 command / task 涉及真实交易，必须明确：

```text
真实交易开关
OrderPlan / CandidateOrderIntent 来源
RiskCheck 入口
ApprovedOrderIntent 来源
ExecutionPreparation 入口
Execution 入口
dry-run / paper / real 模式
```

---

## 15. 新增核心 Python 文件顶部说明

新增核心 Python 文件时，文件顶部必须用简短注释说明：

```text
属于哪个模块
负责什么
不负责什么
是否读写数据库
是否访问 Redis
是否访问外部服务
是否发送 Hermes
是否调用大模型
是否涉及交易执行
是否允许真实交易
```

如果不涉及某项，应明确写“不涉及”。

---

## 16. 测试与验收规则

每次开发必须提供可执行的验收方式。

至少说明：

```text
应运行哪些测试
应执行哪些 management command
应检查哪些数据库记录
什么结果算通过
什么结果算失败
```

如果测试无法运行，必须说明原因。

不得用“看起来没问题”代替验收。

交易相关功能必须额外说明：

```text
是否真实交易关闭
是否使用 dry-run
是否产生 CandidateOrderIntent
是否产生 ApprovedOrderIntent
是否提交 ExchangeOrder
是否写入 TradeFill
是否更新 PositionState
是否写 AlertEvent
是否发送 Hermes
```

---

## 17. 阶段交付回报规则

Codex 回报必须说明：

```text
本阶段实现了什么
修改和新增了哪些文件
主要调用链路是什么
是否写库
是否访问 Redis
是否发送 Hermes
是否调用大模型
是否涉及交易执行
是否涉及真实交易
是否涉及 FeatureLayer
是否涉及 AtomicSignal
是否涉及 DecisionSnapshot
是否涉及 Binance Account Sync
是否涉及 PriceSnapshot
是否涉及 OrderPlan / CandidateOrderIntent
是否涉及 RiskCheck / ApprovedOrderIntent
是否涉及 ExecutionPreparation / Execution
是否涉及 Tracking / TradeLifecycle
是否写 AlertEvent
dry-run / confirm-write 行为，如本阶段涉及
异常处理方式
测试命令和结果
本阶段明确不负责什么
是否违反 project_invariants.md
```

默认不强制创建 implementation 文档。

implementation 文档只用于记录复杂内部逻辑，例如：

```text
特征计算
原子信号
策略规则
风控规则
订单计划规则
回测撮合
执行状态机
复盘归因逻辑
```

---

## 18. 当前阶段禁止提前实现

除非用户明确要求，否则当前阶段不得提前实现：

```text
复杂 UI
多策略组合管理
机器学习模型
实时大模型交易判断
复杂报表系统
自动参数优化
自动策略上线
未经验证的实盘执行
```

项目优先级：

```text
第一，真实策略有效性。
第二，数据、回测、风控、实盘一致性。
第三，策略组合、监控、复盘。
第四，后台、界面、交互体验。
```

如果任务过早投入外围功能，Codex 应提醒用户该功能是否直接服务策略验证、风控或实盘可靠性。

---

## 19. 语言与通知

面向用户的通知、日志摘要、Hermes 消息优先使用中文。

专业术语可保留英文，但应附中文解释。

交易相关通知必须明确区分：

```text
系统分析
策略信号
目标仓位决策
订单计划
候选订单意图
风控结果
审批通过订单意图
执行前检查
交易所订单
真实成交
仓位变化
复盘结论
```

不得把通知写成模糊喊单。

---

## 20. 时间规则

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

## 21. 框架优先原则

Codex 必须优先使用项目已选定框架的内建能力。

本项目使用 Django、Django ORM、Django migrations、Django settings、Celery、Celery Beat、Redis、Python logging 和 pytest / Django test framework。

Codex 不得自研 ORM、migration、配置系统、日志系统、任务队列、调度系统、测试框架或数据库连接池。

scripts、management command、Celery task 只能作为入口调用 application service，不得承载核心业务逻辑。
