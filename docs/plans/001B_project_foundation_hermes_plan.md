# 001B_project_foundation_hermes_plan.md

# Plan 001B：Hermes / notifications 通知底座

## 0. 启动条件

本 plan 只能在 001A 完成后执行。

Codex 开始前必须确认用户已经明确表示：

```text
A 部分已验收，MySQL / Redis 已配置并确认，可以开始 001B。
```

如果用户没有明确确认，Codex 必须停止，不得实现本文件任何代码。

本 plan 依赖：

```text
Django 项目基础结构已存在
Django settings 已存在
MySQL 配置入口已存在
Redis / Celery 配置入口已存在
Django ORM / migrations 基础可用或已明确环境限制
trace_id 已实现
trigger_source 已实现
基础异常类型已实现
```

---

## 1. 阶段目标

本阶段负责实现 Hermes / notifications 通知底座。

本阶段必须实现：

```text
notifications app
AlertEvent model / migration
HermesNotification model / migration
Hermes client
notifications service
notifications Celery task 或 management command
B 部分测试
B 部分 implementation 文档
```

本阶段仍然不做业务数据链路。

---

## 2. 本阶段明确不做

本阶段禁止实现：

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
RiskCheck

CandidateOrderIntent / ApprovedOrderIntent
Execution
Binance 下单接口
Binance 账户接口
Binance 仓位接口
Binance 杠杆接口
WebSocket 行情
自动交易
回测系统
Admin 后台
策略逻辑
大模型分析
```

本阶段禁止自研：

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

---

## 3. 依赖文档

Codex 开始前必须阅读：

```text
AGENTS.md
docs/rules/project_invariants.md
docs/requirements/project_foundation.md
docs/plans/001A_project_foundation_framework_plan.md
docs/plans/001B_project_foundation_hermes_plan.md
docs/implementation/001A_project_foundation_framework.md
```

如果 A 部分 implementation 文档不存在，Codex 必须停止并提示用户。

---

## 4. Hermes 官方定义与集成边界

本项目中的 Hermes 指 Nous Research 的 Hermes Agent / Hermes Gateway 相关能力。

Codex 查阅 Hermes 资料时，只允许使用官方资料：

```text
https://hermes-agent.nousresearch.com/docs
https://hermes-agent.nousresearch.com/docs/llms.txt
https://hermes-agent.nousresearch.com/docs/llms-full.txt
```

禁止使用第三方博客、非官方教程、论坛帖子或非当前仓库文档定义的实现假设来实现 Hermes 集成。

foundation 阶段只允许使用 Hermes 的通知投递能力。

禁止使用 Hermes 的以下能力：

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

正确链路：

```text
业务模块
→ 写 AlertEvent
→ notifications service 异步消费 AlertEvent
→ Hermes client 发送通知
→ HermesNotification 记录发送结果
```

业务模块不得直接调用 Hermes webhook。

---

## 5. Phase B1：notifications app 与 Hermes 配置

如 A 部分未创建 `apps/notifications/`，本阶段创建。

必须配置或确认 settings 中存在：

```text
HERMES_ENDPOINT
HERMES_TOKEN / HERMES_SECRET（如官方接入需要）
HERMES_ENABLED
HERMES_TIMEOUT_SECONDS
```

要求：

```text
Hermes endpoint 从 settings 读取。
Hermes token / secret 从 settings 读取。
发送超时可配置。
不得在日志中输出 secret。
不得提交真实 webhook / token / secret。
```

禁止：

```text
硬编码 Hermes endpoint
硬编码 Hermes token / secret
在 repository / model 中读取 Hermes 配置并发送请求
```

---

## 6. Phase B2：AlertEvent model

AlertEvent 是系统内部告警事件。

本阶段必须实现 AlertEvent model / migration。

字段至少覆盖以下语义：

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

建议状态：

```text
pending
processing
processed
failed
skipped
```

建议 severity：

```text
info
warning
error
critical
```

要求：

```text
AlertEvent 必须存 MySQL。
AlertEvent 不得只写日志。
AlertEvent 必须可被 notifications service 查询。
AlertEvent 必须可追溯 trace_id。
AlertEvent 必须记录 trigger_source。
```

禁止：

```text
手写 CREATE TABLE 替代 migration
在 model 中调用 Hermes
在 model 中承载业务流程
在 repository / model 中调用外部 webhook
```

---

## 7. Phase B3：HermesNotification model

HermesNotification 是 Hermes 发送尝试与结果记录。

字段至少覆盖以下语义：

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

建议状态：

```text
pending
sending
sent
failed
skipped
```

要求：

```text
HermesNotification 必须存 MySQL。
HermesNotification 必须关联 AlertEvent。
Hermes 发送成功必须记录。
Hermes 发送失败必须记录。
Hermes 发送失败不得改变原始 AlertEvent 所代表的业务事实。
```

禁止：

```text
HermesNotification 不得替代 AlertEvent。
HermesNotification 不得作为业务事实。
HermesNotification model 不得调用 Hermes。
```

---

## 8. Phase B4：Hermes client

Hermes client 只负责和 Hermes 外部接口交互。

要求：

```text
Hermes endpoint 从 settings 读取。
Hermes token / secret 从 settings 读取。
发送超时可配置。
发送失败返回明确错误。
不得在日志中输出 secret。
```

Hermes client 应可被 mock 测试。

禁止：

```text
data_collection 直接调用 Hermes client
data_quality 直接调用 Hermes client
data_backfill 直接调用 Hermes client
MarketSnapshot 直接调用 Hermes client
repository 直接调用 Hermes client
model 直接调用 Hermes client
Hermes client 修改业务表
Hermes client 生成策略建议
Hermes client 调用大模型
Hermes client 执行交易
```

---

## 9. Phase B5：notifications service

notifications service 负责：

```text
查询待发送 AlertEvent
调用 Hermes client
写 HermesNotification
更新发送状态
记录发送失败
```

要求：

```text
发送成功写 HermesNotification sent。
发送失败写 HermesNotification failed。
发送失败不得删除 AlertEvent。
发送失败不得改变原业务事实。
发送失败不得抛出导致上游业务回滚的异常。
```

notifications service 禁止：

```text
修改 KlineRecord
修改 DataQualityResult
修改 BackfillResult
修改 MarketSnapshot
触发交易
触发风控
触发回补
生成策略建议
调用大模型
```

---

## 10. Phase B6：task / command

本阶段可以提供：

```text
Celery task：dispatch pending AlertEvent
management command：人工触发 dispatch pending AlertEvent
```

要求：

```text
Celery task 只能调用 notifications service。
management command 只能调用 notifications service。
入口层不得直接写完整发送逻辑。
```

禁止：

```text
task 直接拼接 Hermes 请求
command 直接调用 Hermes webhook
task / command 修改业务事实
task / command 承载复杂业务流程
```

---

## 11. Phase B7：测试

B 部分必须测试：

```text
AlertEvent 可通过 ORM 创建
HermesNotification 可通过 ORM 创建
HermesNotification 可关联 AlertEvent
Hermes client 可 mock 发送成功
Hermes client 可 mock 发送失败
notifications service 可处理 pending AlertEvent
发送成功写 HermesNotification sent
发送失败写 HermesNotification failed
发送失败不删除 AlertEvent
发送失败不改变原业务事实
task / command 只调用 service
默认测试不访问真实 Hermes
默认测试不访问真实 Binance
默认测试不访问生产 MySQL
默认测试不访问生产 Redis
```

B 部分至少尝试运行：

```bash
python manage.py check
python manage.py makemigrations --check
python manage.py test
```

如果项目使用 pytest，也运行：

```bash
pytest
```

如果命令无法运行，必须说明原因。

---

## 12. Phase B8：implementation 文档

B 部分完成后，必须新增或更新：

```text
docs/implementation/001B_project_foundation_hermes.md
```

内容至少包括：

```text
AlertEvent 说明
HermesNotification 说明
notifications service 调用链
Hermes client 集成边界
task / command 说明
B 部分测试方式
B 部分未实现内容
```

必须明确写入：

```text
B 部分未实现 K 线采集。
B 部分未实现 data_quality。
B 部分未实现 data_backfill。
B 部分未实现 MarketSnapshot。
B 部分未实现策略。
B 部分未实现风控。
B 部分未实现交易。
B 部分未自研框架已有基础能力。
```

---

## 13. B 部分建议目录结构

```text
apps/notifications/
tests/notifications/
docs/implementation/
```

如果目录已存在，只能最小修改，不得删除重建。

---

## 14. B 部分建议文件清单

Codex 应结合现有仓库结构最小实现。

```text
apps/notifications/__init__.py
apps/notifications/apps.py
apps/notifications/models.py
apps/notifications/services.py
apps/notifications/hermes_client.py
apps/notifications/tasks.py
apps/notifications/management/commands/dispatch_alert_events.py

tests/notifications/test_alert_event_model.py
tests/notifications/test_hermes_notification_model.py
tests/notifications/test_notifications_service.py
tests/notifications/test_hermes_client.py

docs/implementation/001B_project_foundation_hermes.md
```

如果当前项目已经采用不同 app 命名，Codex 必须优先沿用现有结构，不得强行改名或重排。

---

## 15. B 部分人工审查清单

用户合并前应检查：

```text
是否在用户确认后才开始 B 部分
是否使用 Django ORM
是否使用 Django migrations
是否实现 AlertEvent
是否实现 HermesNotification
是否业务模块只写 AlertEvent，不直接调用 Hermes
是否 Hermes 失败不会改变业务事实
是否没有实现 K 线采集
是否没有实现 data_quality
是否没有实现 data_backfill
是否没有实现 MarketSnapshot
是否没有实现策略
是否没有实现风控
是否没有实现交易
```

建议搜索：

```bash
grep -R "CREATE TABLE" apps scripts tests
grep -R "pymysql.connect" .
grep -R "mysql.connector" .
grep -R "schedule.every" apps scripts tests
grep -R "threading.Timer" apps scripts tests
grep -R "print(" apps scripts tests
grep -R "order" apps scripts tests
grep -R "position" apps scripts tests
grep -R "leverage" apps scripts tests
grep -R "account" apps scripts tests
```

如果出现上述结果，Codex 必须解释原因。

---

## 16. B 部分 Codex 输出总结要求

B 部分完成后，Codex 总结必须包含：

```text
是否在用户确认后才开始 B 部分
修改了哪些文件
新增了哪些文件
是否使用 Django ORM
是否新增 migrations
是否实现 AlertEvent
是否实现 HermesNotification
是否实现 notifications service
是否实现 Hermes client
是否新增 tests
测试命令和结果
是否有无法运行的命令
是否存在未完成项
```

B 部分总结必须明确说明：

```text
B 部分没有实现 K 线采集。
B 部分没有实现 data_quality。
B 部分没有实现 data_backfill。
B 部分没有实现 MarketSnapshot。
B 部分没有实现策略。
B 部分没有实现风控。
B 部分没有实现交易。
B 部分没有自研框架已有基础能力。
```

---

## 17. B 部分最终验收标准

B 部分完成后，必须满足：

```text
AlertEvent 可以通过 Django ORM 写入。
HermesNotification 可以通过 Django ORM 写入。
notifications service 可以消费 mock AlertEvent。
Hermes client 可以 mock 发送成功和失败。
Hermes 发送失败可以记录失败结果。
业务模块不需要直接调用 Hermes webhook。
默认测试可以运行或说明环境限制。
默认测试不访问真实外部服务。
未实现 K 线采集。
未实现策略。
未实现风控。
未实现交易。
未自研框架已有基础能力。
```

---

## 18. 下一阶段

001A 和 001B 全部完成并验收后，才能进入：

```text
docs/plans/002A_market_data_models_plan.md
```

下一阶段才开始实现：

```text
data_collection
data_quality
data_backfill
MarketSnapshot
```
