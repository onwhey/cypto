# 001B Hermes / notifications 通知底座实现说明

## 阶段边界

本阶段只实现 `docs/plans/001B_project_foundation_hermes_plan.md` 中的 Hermes / notifications 基础能力。

本阶段没有实现：

```text
K 线采集
data_quality
data_backfill
MarketSnapshot
FeatureLayer
AtomicSignal
DomainSignal
MarketRegime
StrategySignal
DecisionSnapshot
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
Tracking
Binance 账户 / 仓位 / 下单 / 杠杆接口
WebSocket 行情
自动交易
回测系统
策略逻辑
风控逻辑
大模型分析
```

## 配置

001B 新增 Hermes 配置，全部来自 `.env` / Django settings：

```text
HERMES_ENDPOINT
HERMES_TOKEN
HERMES_SECRET
HERMES_ENABLED
HERMES_TIMEOUT_SECONDS
HERMES_MAX_ATTEMPTS
```

`HERMES_ENABLED` 默认关闭。默认测试不真实访问 Hermes endpoint。

## AlertEvent

`apps.notifications.models.AlertEvent` 是系统内部告警事件。

它负责记录：

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
updated_at_utc
```

AlertEvent 使用 Django ORM 和 Django migrations 管理，存储在 MySQL 或 Django 测试数据库中。

AlertEvent 不调用 Hermes，不承载业务流程，不触发交易。

## HermesNotification

`apps.notifications.models.HermesNotification` 记录 Hermes 发送尝试和结果。

它负责记录：

```text
alert_event
trace_id
status
request_summary
response_summary
error_code
error_message
attempt_count
sent_at_utc
created_at_utc
updated_at_utc
```

HermesNotification 是发送记录，不是业务事实，不替代 AlertEvent。

## Hermes client

`apps.notifications.hermes_client.HermesClient` 是受控外部请求薄封装。

职责：

```text
从 settings 读取 Hermes endpoint / token / secret / enabled / timeout
当 HERMES_ENABLED=false 时返回 skipped，不访问外部服务
当 HERMES_ENABLED=true 时使用配置 endpoint 发送 JSON 请求
返回明确 sent / failed / skipped 结果
不写数据库
不记录密钥
不调用大模型
不触发交易
```

## notifications service 调用链

调用链：

```text
AlertEvent(status=pending)
→ NotificationDispatchService.get_pending_alert_events()
→ HermesClient.send_alert()
→ HermesNotification
→ 更新 AlertEvent 通知处理状态
```

发送失败时：

```text
写 HermesNotification(status=failed)
保留 AlertEvent
不删除原始告警事件
不改变任何业务事实
```

## task / command

001B 新增入口：

```text
Celery task: notifications.dispatch_pending_alert_events
management command: dispatch_alert_events
```

二者只负责解析入口参数、生成或传递 trace_id / trigger_source，并调用 `NotificationDispatchService`。

入口层不拼接 Hermes 请求，不承载复杂业务流程。

## 测试方式

001B 默认验收命令：

```bash
python manage.py check
python manage.py makemigrations --check
python manage.py test
```

测试覆盖：

```text
AlertEvent ORM 创建
HermesNotification ORM 创建和关联
Hermes client disabled / mock success / mock failure
notifications service success / failure / skipped
task / command 只调用 service
默认测试不访问真实 Hermes
默认测试不访问 Binance
默认测试不访问生产 MySQL / Redis
```

## 未实现内容

001B 未实现 K 线采集。
001B 未实现 data_quality。
001B 未实现 data_backfill。
001B 未实现 MarketSnapshot。
001B 未实现策略。
001B 未实现风控。
001B 未实现交易。
001B 未自研框架已有基础能力。
