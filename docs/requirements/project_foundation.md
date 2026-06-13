# 项目基础设施需求

## 1. 文档目的

本文档定义中低频趋势跟踪自动交易系统的项目基础设施需求。

项目基础设施阶段的目标不是实现交易策略，也不是实现数据采集，而是先搭建后续模块可以稳定依赖的开发框架、配置、存储、任务、日志、通知和测试底座。

本文档用于回答：

```text
项目基础框架应该具备哪些能力？
哪些能力必须优先使用框架内建能力？
MySQL 和 Redis 在第一阶段需要达到什么可用状态？
AlertEvent / Hermes 通知底座需要做到什么程度？
后续 data_collection、data_quality、data_backfill、MarketSnapshot 可以依赖哪些基础能力？
哪些能力第一阶段明确不做？
哪些行为 Codex 不得实现？
```

本文档不定义：

```text
K 线采集逻辑
数据质量检查规则
数据回补规则
MarketSnapshot 生成规则
FeatureLayer
AtomicSignal
StrategySignal
DecisionSnapshot
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
交易所账户 / 订单 / 仓位接口
自动交易逻辑
```

---

## 2. 模块目标

project_foundation 阶段的目标是：

```text
搭建 Django 项目基础结构
配置基础环境变量
确认 MySQL 可连接、可迁移、可测试
确认 Redis 可连接、可用于 Celery 和基础锁能力
确认 Celery / Celery Beat 基础配置可用
建立统一日志基础
建立 trace_id / trigger_source 基础规范
建立 AlertEvent 数据库事件表
建立 HermesNotification 发送结果记录
建立 notifications / Hermes 底层服务
建立基础测试框架
明确禁止自研框架能力
```

一句话：

```text
装好开发框架，调整好基础配置，确认 MySQL / Redis 可用，并搭建可被后续模块复用的 Hermes / notifications 底层服务。
```

---

## 3. 基础设施建设原则

### 3.1 框架优先原则

项目基础设施阶段必须优先使用成熟框架和库的内建能力。

第一阶段基于：

```text
Django
Django settings
Django ORM
Django migrations
Django management command
Python logging / Django logging
Celery
Celery Beat
Redis
pytest / pytest-django 或 Django test framework
```

禁止为框架已经提供的基础能力重复造轮子。

不得自研：

```text
ORM
migration 系统
配置加载框架
日志框架
任务队列框架
调度框架
测试框架
数据库连接池框架
后台命令框架
```

如确实需要封装，只能在框架能力之上做薄封装，不得替代框架机制。

### 3.2 业务约束补充原则

项目允许在框架基础上补充交易系统所需的业务约束，包括：

```text
trace_id
trigger_source
AlertEvent
HermesNotification
notifications service
Hermes client
审计记录
幂等边界
任务入口边界
```

这些是项目业务对象，不是自研框架。

### 3.3 入口层轻量原则

以下入口只负责解析参数、校验入口请求，并调用 application service：

```text
Django management command
Celery task
Celery Beat schedule
CLI / scripts（如保留）
```

入口层不得承载核心业务逻辑。

禁止：

```text
management command 直接写复杂业务流程
Celery task 直接串联完整业务链路
scripts 直接访问数据库完成业务写入
scripts 直接请求外部服务并落库
```

核心业务逻辑必须放在 service 层。

---

## 4. 第一阶段范围

### 4.1 P0 必须实现

P0-A：基础框架

```text
Django 项目基础结构
Django settings 基础配置
环境变量加载规则
MySQL 配置入口
Django migrations 基础可用
Redis 配置入口
Celery broker / result backend 基础配置
Celery Beat 基础配置
Python logging / Django logging 基础配置
trace_id 生成与传递规范
trigger_source 规范
基础异常类型
基础测试框架
```

P0-B：Hermes / notifications，必须在 P0-A 验收、MySQL / Redis 配置确认后执行

```text
AlertEvent model / migration
HermesNotification model / migration
notifications service
Hermes client
Hermes 发送结果记录
```

P0-A 和 P0-B 不得在同一次 Codex 执行中混合实现。

### 4.2 P1 可预留

```text
AlertEvent 消费重试策略
Hermes 发送失败重试
通知去重
通知级别路由
通知查询命令
Celery worker 健康检查
Redis 分布式锁工具封装
统一 service result 类型
结构化日志增强
```

### 4.3 P2 后续再做

```text
多通知通道
通知看板
复杂通知订阅规则
Admin 后台管理
任务可视化看板
完整运维监控系统
多环境部署编排
```

### 4.4 第一阶段明确不做

```text
K 线采集
data_quality
data_backfill
MarketSnapshot
FeatureLayer
AtomicSignal
StrategySignal
DecisionSnapshot
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
Binance 下单接口
Binance 账户接口
Binance 仓位接口
Binance 杠杆接口
自动交易
回测系统
Admin 后台
WebSocket 实时行情
```

---

## 5. Django 项目框架要求

项目必须基于 Django 标准结构建设。

要求：

```text
使用 Django settings 管理配置。
使用 Django app 组织模块。
使用 Django ORM 管理业务表。
使用 Django migrations 管理表结构变化。
使用 Django management command 作为人工命令入口。
使用 Django / pytest-django 测试能力。
```

禁止：

```text
绕过 Django settings 自写配置系统。
绕过 Django ORM 自写通用 ORM。
绕过 Django migrations 手写业务建表系统。
在 scripts 中直接执行核心业务 SQL。
在入口文件中塞入复杂业务逻辑。
```

说明：

```text
如果某些底层能力需要薄封装，封装必须服务于业务边界和可测试性，不得替代 Django 框架能力。
```

---

## 6. 配置与环境变量要求

配置必须通过 Django settings 和环境变量管理。

P0 配置至少覆盖：

```text
Django secret key
debug 开关
allowed hosts
MySQL 连接参数
Redis 连接参数
Celery broker
Celery result backend
Hermes webhook / endpoint 配置
Hermes 发送开关
Hermes 真实发送默认关闭
日志级别
当前运行环境
```

要求：

```text
敏感配置不得提交到 Git。
.env 不得提交。
配置缺失时必须明确报错。
默认测试环境不得使用生产配置。
Hermes webhook、数据库密码、Redis 密码不得写入日志。
```

禁止：

```text
在业务代码中硬编码数据库连接信息。
在业务代码中硬编码 Hermes webhook。
在业务代码中硬编码生产密钥。
在测试中使用真实生产配置。
```

---

## 7. MySQL 基础能力要求

MySQL 是项目主业务数据存储。

P0 必须实现：

```text
Django 可以连接 MySQL。
Django migrations 可以正常执行。
业务表通过 Django model + migration 创建。
测试环境可以使用独立测试数据库或测试替身。
基础连接失败时有明确错误。
```

MySQL 第一阶段将承载：

```text
AlertEvent
HermesNotification
后续 KlineRecord
后续 DataQualityResult
后续 DataQualityIssue
后续 BackfillRun / BackfillResult（结果摘要，如保留）
后续 MarketSnapshot
```

禁止：

```text
使用 Redis 替代 MySQL 存储核心业务事实。
只写日志不入库。
在无 migration 的情况下手动创建业务表。
用裸 SQL 作为常规业务写入方式。
绕过 Django ORM 写核心业务对象。
```

说明：

```text
某些性能敏感查询后续可以使用受控 raw SQL，但第一阶段不得绕过 ORM 建立基础业务能力。
```

---

## 8. Redis 基础能力要求

Redis 是短期状态、Celery 基础设施和后续分布式锁能力的支撑。

P0 Redis 用途：

```text
Celery broker
Celery result backend
基础连接验证
后续分布式锁能力预留
短期状态能力预留
```

Redis 可以用于：

```text
Celery 任务队列
Celery 任务结果
短期锁
短期状态
限频状态
临时去重 key
```

Redis 禁止用于：

```text
核心 K 线主存储
DataQualityResult 主存储
BackfillRun 主存储；BackfillResult 如保留，仅作为 BackfillRun 的结果摘要语义。
MarketSnapshot 主存储
CandidateOrderIntent 主存储
ApprovedOrderIntent 主存储
ExchangeOrder 主存储
TradeFill 主存储
长期审计数据主存储
```

规则：

```text
Redis 数据必须可过期、可重建、可丢失后恢复。
Redis 不得替代 MySQL 作为核心事实存储。
```

---

## 9. Celery / Celery Beat 基础要求

Celery 是项目异步任务和后续调度任务基础。

P0 要求：

```text
Celery app 可初始化。
Celery 使用 Redis 作为 broker。
Celery result backend 可配置。
Celery Beat 基础配置可用。
可以定义一个最小测试任务验证 worker 可运行。
```

Celery task 只允许作为入口层。

禁止：

```text
Celery task 中直接写完整业务流程。
Celery task 中直接访问 Binance 并写业务表。
Celery task 中绕过 application service。
Celery task 中吞掉异常不记录。
Celery task 中绕过 trace_id。
```

后续业务任务必须调用 application service，例如：

```text
RunAnalysisCycleService
DataCollectionService
DataBackfillService
NotificationDispatchService
```

---

## 10. 日志基础要求

项目必须使用 Python logging / Django logging。

P0 要求：

```text
统一日志格式。
日志包含时间、级别、模块、message。
重要业务日志应包含 trace_id。
异常日志应包含错误类型和必要上下文。
```

禁止记录：

```text
数据库密码
Redis 密码
Hermes webhook
token
Authorization header
cookie
.env 完整内容
账户信息
仓位信息
订单敏感信息
```

禁止：

```text
用 print 作为业务日志。
每个模块自定义一套日志系统。
日志中输出完整敏感配置。
```

---

## 11. trace_id 要求

trace_id 用于追踪一次业务运行或一次事件链路。

P0 要求：

```text
提供 trace_id 生成工具或规范。
入口层如未传入 trace_id，应生成 trace_id。
下游 service 必须显式传递 trace_id。
AlertEvent 必须记录 trace_id。
HermesNotification 必须记录 trace_id。
```

规则：

```text
trace_id 用于追踪，不作为业务幂等键。
trace_id 不得替代唯一业务键。
```

后续模块必须可复用 trace_id：

```text
data_collection
data_quality
data_backfill
MarketSnapshot
FeatureLayer
StrategySignal
DecisionSnapshot
OrderPlan
CandidateOrderIntent
RiskCheckResult
ApprovedOrderIntent
ExecutionPreparation
ExecutionResult
ExchangeOrder
TradeFill
PositionState
ReviewRecord
```

---

## 12. trigger_source 要求

trigger_source 用于记录任务触发来源。

P0 建议取值：

```text
cli
management_command
celery_beat
celery_worker
manual_recovery
admin
retry
system
```

规则：

```text
trigger_source 必须显式设置。
trigger_source 不得由程序随意猜测。
Celery worker 不得覆盖原始 trigger_source。
如果需要记录执行者，应另设 executor_source。
```

AlertEvent、HermesNotification 和后续业务记录必须能保存 trigger_source。

---

## 13. Hermes / notifications 底层服务

### 13.0 Hermes 定义与资料来源

本项目中的 Hermes 指 Nous Research 的 Hermes Agent / Hermes Gateway 相关能力。

Hermes Agent 是 Nous Research 提供的 agent 系统。其官方文档说明 Hermes 具备 messaging gateway 能力，可以连接 Telegram、Discord、Slack、WhatsApp、Signal、Email、DingTalk、Feishu/Lark、WeCom、Weixin、QQ、Microsoft Teams 等多个消息平台。

官方文档还提供面向 LLM / coding agents 的机器可读入口，包括：

```text
https://hermes-agent.nousresearch.com/docs/llms.txt
https://hermes-agent.nousresearch.com/docs/llms-full.txt
```

本项目只把 Hermes 作为通知投递相关的外部依赖 / 底层通知通道使用。

foundation 阶段只允许实现：

```text
AlertEvent 入库
notifications service 消费 AlertEvent
Hermes client 调用 Hermes 相关通知接口
HermesNotification 记录发送结果
```

foundation 阶段不使用 Hermes 的以下能力：

```text
agent 自主执行任务
skills
memory
terminal control
background sessions
cron automations
LLM 分析
策略建议生成
交易决策
自动修复数据
自动执行交易
```

Codex 查阅 Hermes 资料时，只允许使用官方资料：

```text
https://hermes-agent.nousresearch.com/docs
https://hermes-agent.nousresearch.com/docs/llms.txt
https://hermes-agent.nousresearch.com/docs/llms-full.txt
```

禁止 Codex 使用第三方博客、非官方教程、论坛帖子来实现 Hermes 集成。

如果官方 Hermes 文档与本项目需求冲突，以本项目需求为准：

```text
业务模块只写 AlertEvent，不得直接调用 Hermes。
Hermes 发送失败不得改变原业务事实。
Hermes 不参与策略建议、交易决策、数据修复或自动交易。
```

---

### 13.1 模块定位

Hermes / notifications 是本项目对 Hermes Agent / Hermes Gateway 通知能力的受控封装，不属于 Django 框架能力。

它负责：

```text
记录系统告警事件
异步消费 AlertEvent
调用 Hermes client 发送通知
记录 HermesNotification 发送结果
保证通知失败不阻塞主业务事实记录
```

它不负责：

```text
生成交易建议
生成策略判断
修复数据
回补数据
调用风控
调用交易执行
修改业务事实
```


### 13.2 AlertEvent

AlertEvent 是系统内部告警事件。

后续模块只写 AlertEvent，不直接同步调用 Hermes Webhook。

P0 AlertEvent 至少应表达：

```text
event_type
severity
status
source_module
message
trace_id
trigger_source
related_object_type
related_object_id
error_code
error_message
created_at_utc
```

字段细节由后续数据库设计确定，但必须满足：

```text
可记录来源模块
可记录严重程度
可记录是否已被处理
可关联业务对象
可追踪 trace_id
```

AlertEvent 示例来源：

```text
data_collection 采集失败
data_quality 发现 K 线问题
data_backfill 回补失败
MarketSnapshot BLOCKED / FAILED
后续 Execution 执行异常
```

### 13.3 HermesNotification

HermesNotification 用于记录一次 Hermes 发送尝试和结果。

P0 至少应表达：

```text
alert_event_id
trace_id
status
request_summary
response_summary
error_code
error_message
attempt_count
sent_at_utc
created_at_utc
```

发送状态建议：

```text
pending
sending
sent
failed
skipped
```

说明：

```text
HermesNotification 是发送记录，不是业务事实。
Hermes 发送失败不得改变原始业务事件事实。
```

### 13.4 notifications service

notifications service 负责消费 AlertEvent 并发送 Hermes 通知。

P0 要求：

```text
可以查询待发送 AlertEvent。
可以调用 Hermes client。
可以写 HermesNotification。
可以记录发送成功或失败。
发送失败不得影响原业务模块已写入的 AlertEvent。
```

禁止：

```text
notifications service 修改 KlineRecord。
notifications service 修改 DataQualityResult。
notifications service 修改 BackfillRun / BackfillResult。
notifications service 修改 MarketSnapshot。
notifications service 触发交易。
notifications service 触发风控。
notifications service 触发回补。
```

### 13.5 Hermes client

Hermes client 只负责与 Hermes 外部接口交互。

要求：

```text
Hermes endpoint 从配置读取。
发送超时必须可配置。
发送失败必须返回明确错误。
不得在日志中输出 webhook secret。
不得由业务模块直接调用 Hermes webhook。
```

禁止：

```text
在 data_collection 中直接调用 Hermes webhook。
在 data_quality 中直接调用 Hermes webhook。
在 data_backfill 中直接调用 Hermes webhook。
在 MarketSnapshot 中直接调用 Hermes webhook。
在 repository 中直接调用 Hermes webhook。
```

正确链路：

```text
业务模块
→ 写 AlertEvent
→ notifications service 异步消费 AlertEvent
→ Hermes client 发送
→ HermesNotification 记录发送结果
```

---

## 14. 基础异常类型要求

P0 应提供基础异常规范或基础异常类型。

至少区分：

```text
配置错误
数据库连接错误
Redis 连接错误
外部服务调用错误
通知发送错误
参数错误
系统未预期错误
```

要求：

```text
异常消息不得包含敏感信息。
异常应保留 trace_id。
异常应可被记录到日志或 AlertEvent。
```

不要求在 foundation 阶段为每个后续业务模块定义专属异常。

---

## 15. 测试基础要求

项目必须具备可运行的基础测试能力。

P0 要求：

```text
可以运行默认测试命令。
默认测试不访问真实 Binance。
默认测试不发送真实 Hermes。
默认测试不访问生产 MySQL。
默认测试不访问生产 Redis。
MySQL / Redis / Hermes 相关测试应使用测试配置、mock 或显式集成测试开关。
```

建议覆盖：

```text
Django settings 可加载
MySQL 配置可读取
Redis 配置可读取
AlertEvent model 可创建
HermesNotification model 可创建
notifications service 可处理 mock AlertEvent
Hermes client 可被 mock
trace_id 可生成
trigger_source 可传递
```

如需真实集成测试，必须使用显式环境变量开启，并默认关闭。

---

## 16. 与后续数据层的关系

project_foundation 完成后，后续模块可以依赖：

```text
Django app 结构
Django settings
Django ORM
Django migrations
MySQL 连接
Redis 连接
Celery 基础配置
日志基础
trace_id 规范
trigger_source 规范
AlertEvent
HermesNotification
notifications service
Hermes client
基础测试框架
```

后续数据层包括：

```text
data_collection
data_quality
data_backfill
MarketSnapshot
```

这些模块不得重新实现 foundation 已提供的基础能力。

---

## 17. 禁止事项

project_foundation 阶段禁止：

```text
实现 K 线采集
实现 data_quality
实现 data_backfill
实现 MarketSnapshot
实现 FeatureLayer
实现 AtomicSignal
实现 StrategySignal
实现 DecisionSnapshot
实现 OrderPlan
实现 CandidateOrderIntent
实现 RiskCheck
实现 ApprovedOrderIntent
实现 ExecutionPreparation
实现 Execution
接入 Binance 下单接口
接入 Binance 账户接口
接入 Binance 仓位接口
接入 Binance 杠杆接口
实现 WebSocket 行情
实现自动交易
实现策略逻辑
实现回测系统
自研 ORM
自研 migration
自研配置框架
自研日志框架
自研任务队列
自研调度系统
自研测试框架
用 scripts 承载核心业务逻辑
业务模块直接调用 Hermes webhook
```

---

## 18. 验收标准

第一阶段完成后，必须满足：

```text
Django 项目可以启动。
Django settings 可以加载基础配置。
MySQL 配置可用。
Django migrations 可运行。
Redis 配置可用。
Celery app 可初始化。
Celery Beat 基础配置存在。
日志系统可用。
trace_id 可生成和传递。
trigger_source 有基础规范。
AlertEvent 可以通过 Django ORM 写入。
HermesNotification 可以通过 Django ORM 写入。
notifications service 可以消费 mock AlertEvent。
Hermes client 可以通过 mock 验证发送流程。
Hermes 发送失败可以记录失败结果。
业务模块不需要直接调用 Hermes webhook。
默认测试可以运行。
默认测试不访问真实外部服务。
未实现 K 线采集。
未实现策略。
未实现风控。
未实现交易。
未自研框架已有基础能力。
```

---

## 19. 后续计划关系

本文档对应后续开发计划：

```text
docs/plans/001A_project_foundation_framework_plan.md
docs/plans/001B_project_foundation_hermes_plan.md
```
必须先完成 001A，经用户配置并确认 MySQL / Redis 后，才能执行 001B。

该 plan 完成并验收后，才能进入：

```text
docs/plans/002A_market_data_models_plan.md
docs/plans/002B_data_collection_gateway_and_service_plan.md
docs/plans/002C_data_quality_plan.md
docs/plans/002D_data_backfill_plan.md
docs/plans/003A_market_snapshot_model_plan.md
docs/plans/003B_market_snapshot_generation_plan.md
```

数据层和 MarketSnapshot plans 才开始实现：

```text
data_collection
data_quality
data_backfill
MarketSnapshot
```

---

## 20. 总结

project_foundation 是项目基础设施阶段。

核心口径：

```text
优先使用 Django / Celery / Redis / logging / pytest 等框架能力。
不要自研框架已经提供的基础能力。
MySQL 是核心业务事实存储。
Redis 是短期状态和 Celery 基础设施。
Celery 是异步任务和调度基础。
AlertEvent 是业务告警事件。
HermesNotification 是发送结果记录。
业务模块只写 AlertEvent，不直接调用 Hermes webhook。
notifications service 异步消费 AlertEvent 并调用 Hermes client。
Hermes 失败不得改变业务事实。
foundation 阶段不实现数据采集、策略、风控或交易。
```
