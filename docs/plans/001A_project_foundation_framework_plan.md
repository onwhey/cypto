# 001A_project_foundation_framework_plan.md

# Plan 001A：项目基础框架搭建

## 0. 执行边界

本文件只负责项目基础框架搭建。

本文件不负责 Hermes / notifications 通知模块。

Codex 执行本 plan 时，不得创建、修改或实现以下内容：

```text
AlertEvent
HermesNotification
notifications service
Hermes client
Hermes Celery task
Hermes dispatch_alert_events.py management command
任何 Hermes webhook 调用
任何通知发送逻辑
```

Hermes / notifications 必须等用户完成 MySQL / Redis 配置并明确确认后，再执行独立文件：

```text
docs/plans/001B_project_foundation_hermes_plan.md
```

如果用户没有明确说“开始 001B”或等价确认，Codex 不得实现 Hermes 相关代码。

---

## 1. 阶段目标

本阶段目标是把开发框架和基础配置搭起来。

本阶段完成后，项目应具备：

```text
Django 项目基础结构
Django settings
环境变量配置入口
MySQL 配置入口
Redis 配置入口
Celery / Celery Beat 基础配置
Python logging / Django logging
trace_id 工具
trigger_source 规范
基础异常类型
基础测试框架
基础 implementation 文档
```

本阶段不要求 Codex 连接真实 MySQL / Redis。

真实 MySQL / Redis 由用户在本阶段完成后自行配置和确认。

---

## 2. 本阶段明确不做

本阶段禁止实现：

```text
Hermes / notifications
AlertEvent
HermesNotification
AlertEvent / Hermes 新逻辑
market_data
Kline4h / Kline1d
K 线采集
Binance REST
DataCollection
DataQuality
data_quality
Backfill
BackfillRequest
BackfillRun
data_backfill
MarketSnapshot
FeatureLayer
AtomicSignal
DomainSignal
MarketRegime
StrategySignal
DecisionSnapshot
Binance Account Sync
PriceSnapshot
OrderPlan
RiskCheck
ExecutionPreparation

CandidateOrderIntent / ApprovedOrderIntent
Execution
Binance 下单接口
Binance 账户接口
Binance 仓位接口
Binance 杠杆接口
WebSocket 行情
自动交易
真实交易逻辑
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

必须优先使用：

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

---

## 3. 依赖文档

Codex 开始前必须阅读：

```text
AGENTS.md
docs/rules/project_invariants.md
docs/requirements/01_project_scope.md
docs/requirements/02_system_capabilities.md
docs/requirements/project_foundation.md
docs/architecture/00_project_overview.md
docs/architecture/01_module_map.md
docs/architecture/system_architecture.md
docs/architecture/data_flow.md
docs/plans/001A_project_foundation_framework_plan.md
```

如果本 plan 与 `AGENTS.md` 或 `docs/rules/project_invariants.md` 冲突，必须以后两者为准，并在总结中说明冲突。

如果本 plan 与 `docs/requirements/project_foundation.md` 冲突，必须停止并提示用户确认。

---

## 4. Phase A0：检查仓库现状

Codex 必须先检查：

```text
是否已有 manage.py
是否已有 Django settings
是否已有 config/ 或等价 Django project 包
是否已有 apps/ 或等价 app 目录
是否已有 pyproject.toml / requirements.txt / poetry.lock / uv.lock
是否已有 pytest / Django test 配置
是否已有 Celery 配置
是否已有 docs/implementation/
```

禁止：

```text
未检查现有结构前新建重复 Django project
删除现有配置
清空已有 docs
重置仓库脚手架
创建 Hermes / notifications 代码
```

A0 完成后，Codex 必须明确：

```text
项目使用哪个依赖管理方式
Django project 包名
settings 文件位置
app 目录结构
测试命令
是否需要补齐基础文件
```

---

## 5. Phase A1：依赖安装与 Django 项目基础结构

A1 只允许处理：

```text
Django 依赖
Celery 依赖
Redis client 依赖
MySQL driver 依赖
pytest / pytest-django 依赖
Django project 基础文件
Django app 基础目录
manage.py
```

如依赖缺失，补齐项目依赖：

```text
Django
celery
redis
MySQL driver（如 mysqlclient 或 PyMySQL，优先按项目现有规范）
pytest / pytest-django（如项目使用 pytest）
requests 或 httpx（仅作为后续外部 HTTP client 依赖预留；本阶段不得实现 Hermes client）
```

如 Django 基础结构缺失，补齐：

```text
manage.py
config/settings.py 或项目现有 settings 文件
config/urls.py
config/asgi.py
config/wsgi.py
apps/core/
```

本阶段不创建 `apps/notifications/`，避免 Codex 提前进入 Hermes / notifications。

如果项目已有 `apps/notifications/`，不得删除，但不得实现其中任何逻辑。

A1 禁止：

```text
实现 AlertEvent model
实现 HermesNotification model
实现 Hermes client
实现 notifications service
实现 Celery 通知 task
实现 K 线采集
实现策略、风控、交易
```

A1 验收条件：

```text
Django project 基础结构存在。
manage.py 存在并可被调用。
Django settings 文件存在。
core app 可注册或可导入。
项目依赖文件已包含必要框架依赖。
```

尝试运行：

```bash
python manage.py check
```

如果此时因为 settings 尚未配置导致失败，可以进入 A2；但仍不得进入 Hermes 相关实现。

---

## 6. Phase A2：settings / env / logging 基础配置

A2 负责配置 Django settings、环境变量入口和 logging。

必须配置或预留：

```text
DJANGO_SECRET_KEY
DJANGO_DEBUG
DJANGO_ALLOWED_HOSTS
APP_ENV
LOG_LEVEL

MYSQL_DATABASE
MYSQL_USER
MYSQL_PASSWORD
MYSQL_HOST
MYSQL_PORT

REDIS_URL
CELERY_BROKER_URL
CELERY_RESULT_BACKEND
```

注意：

```text
本阶段不配置 HERMES_*。
Hermes 配置属于 001B。
```

logging 要求：

```text
必须基于 Python logging / Django logging。
日志至少包含 timestamp、level、logger name、message。
重要业务日志后续必须能带 trace_id。
```

禁止：

```text
自研 settings 框架
硬编码真实数据库密码
硬编码真实 Redis 密码
提交真实 .env
日志中输出 secret
实现 Hermes client
```

A2 验收条件：

```text
settings 可以加载基础配置。
敏感配置来自环境变量或安全默认值。
logging 配置存在。
python manage.py check 至少能进入 Django 检查流程。
```

---

## 7. Phase A3：MySQL 配置入口与 Django migrations 基础验证

A3 目标是配置 Django ORM 使用 MySQL，并准备 migrations 基础能力。

A3 不是让 Codex 替用户配置真实 MySQL 服务。

A3 必须实现：

```text
Django DATABASES 使用 MySQL 配置入口。
INSTALLED_APPS 正确包含基础 app。
Django migrations 目录结构存在。
可以运行 migration 检查命令。
```

如果当前环境没有 MySQL 服务：

```text
不得为了测试改成生产 MySQL。
不得硬编码本地密码。
可以使用测试配置、mock、SQLite test fallback 或说明需要人工集成验证。
```

禁止：

```text
手写 CREATE TABLE 作为业务建表方式
绕过 Django ORM 创建业务表
直接使用 pymysql.connect / mysql.connector 承载业务写入
创建 AlertEvent / HermesNotification model
```

A3 必须尝试运行：

```bash
python manage.py check
python manage.py makemigrations --check
```

如果命令无法运行，必须记录原因。

A3 完成后，用户需要自行配置真实 MySQL 环境并验证连接。

---

## 8. Phase A4：Redis / Celery / Celery Beat 基础配置

A4 目标是配置 Celery 使用 Redis，并准备 Celery Beat 基础能力。

A4 不是让 Codex 自己实现 Redis 队列。

A4 必须实现：

```text
Celery app 可初始化。
Celery broker 使用 Redis 配置。
Celery result backend 可配置。
Celery Beat 配置位置明确。
可以提供最小 no-op / healthcheck task，但不得承载业务逻辑。
```

Redis 在 A 部分只用于：

```text
Celery broker
Celery result backend
基础连接配置
后续短期锁能力预留
```

Redis 禁止用于：

```text
核心业务事实主存储
K 线主存储
AlertEvent 主存储
MarketSnapshot 主存储
订单意图主存储（CandidateOrderIntent / ApprovedOrderIntent）
长期审计数据主存储
```

A4 禁止：

```text
用 Celery task 实现完整业务流程
在 Celery task 中直接调用 Hermes
在 Celery task 中直接访问 Binance
在 Celery task 中写 K 线
实现 notifications 发送逻辑
```

A4 验收条件：

```text
Celery app 可导入。
Celery broker / result backend 来自 settings。
Celery Beat 基础配置存在。
```

A4 完成后，用户需要自行配置真实 Redis 环境并验证连接。

---

## 9. Phase A5：core 基础工具

A5 在框架基础完成后，实现项目轻量 core 工具。

必须实现：

```text
trace_id 生成工具
trigger_source 受控规范
基础异常类型
```

建议文件：

```text
apps/core/trace.py
apps/core/trigger_source.py
apps/core/exceptions.py
```

trace_id 要求：

```text
提供 generate_trace_id()
提供 ensure_trace_id(input_trace_id)
trace_id 用于追踪，不作为业务唯一键。
trace_id 不得替代幂等键。
```

trigger_source 建议取值：

```text
cli
management_command
celery_beat
celery_worker
manual_recovery
retry
system
```

trigger_source 规则：

```text
trigger_source 必须显式传递。
trigger_source 不得由程序随意猜测。
Celery worker 不得覆盖原始 trigger_source。
```

基础异常类型建议：

```text
ProjectFoundationError
ConfigurationError
DatabaseConnectionError
RedisConnectionError
ExternalServiceError
InvalidTriggerSourceError
```

注意：

```text
NotificationSendError 属于 001B。
```

A5 禁止：

```text
调用 Hermes
访问数据库执行业务写入
实现 AlertEvent / HermesNotification model
```

A5 验收条件：

```text
trace_id 可生成。
ensure_trace_id 可复用传入值或生成新值。
trigger_source 合法值可校验。
非法 trigger_source 可被拒绝。
基础异常可导入。
```

---

## 10. Phase A6：A 部分测试

A 部分测试必须覆盖：

```text
Django settings 可加载
trace_id 可生成
ensure_trace_id 可工作
trigger_source 合法值可校验
非法 trigger_source 可被拒绝
基础异常可导入
Celery app 可导入
默认测试不访问真实 Hermes
默认测试不访问真实 Binance
默认测试不访问生产 MySQL
默认测试不访问生产 Redis
```

A 部分至少尝试运行：

```bash
python manage.py check
python manage.py test
```

如果项目使用 pytest，也运行：

```bash
pytest
```

如果命令无法运行，必须说明原因。

---

## 11. Phase A7：A 部分 implementation 文档

A 部分完成后，Codex 必须新增或更新：

```text
docs/implementation/001A_project_foundation_framework.md
```

A 部分 implementation 至少写明：

```text
Django 项目结构
settings / env 配置说明
MySQL 配置入口说明
Redis / Celery 配置说明
logging 说明
trace_id 说明
trigger_source 说明
A 部分测试方式
A 部分未实现 Hermes / notifications 业务逻辑
```

必须明确写入：

```text
A 部分未实现 AlertEvent。
A 部分未实现 HermesNotification。
A 部分未实现 Hermes client。
A 部分未实现 notifications service。
A 部分未实现 K 线采集。
A 部分未实现策略、风控、交易。
```

---

## 12. A 部分完成后的人工门槛

A 部分完成后，Codex 必须停止。

Codex 不得自行进入 B 部分。

用户需要完成：

```text
配置真实 MySQL 环境变量
确认 Django 可以连接 MySQL
确认 migrations 基础命令可运行或明确环境限制
配置真实 Redis 环境变量
确认 Celery 可以使用 Redis 配置
确认 A 部分测试通过或已知环境限制
```

用户明确确认后，才能执行：

```text
docs/plans/001B_project_foundation_hermes_plan.md
```

建议用户确认语：

```text
A 部分已验收，MySQL / Redis 已配置并确认，可以开始 001B。
```

没有上述确认，Codex 不得写 Hermes 相关代码。

---

## 13. A 部分建议目录结构

Codex 应检查并补齐以下目录。

如果目录已存在，只能最小修改，不得删除重建。

```text
config/
apps/
apps/core/
docs/implementation/
tests/
tests/core/
```

说明：

```text
具体目录可根据仓库现有 Django 结构调整。
不得为了匹配本 plan 删除已有结构。
不得清空已有 docs。
不得重置仓库脚手架。
```

---

## 14. A 部分建议文件清单

Codex 应结合现有仓库结构最小实现。

```text
manage.py
config/settings.py
config/celery.py
config/urls.py
config/asgi.py
config/wsgi.py

apps/core/__init__.py
apps/core/apps.py
apps/core/trace.py
apps/core/trigger_source.py
apps/core/exceptions.py

tests/core/test_trace.py
tests/core/test_trigger_source.py

docs/implementation/001A_project_foundation_framework.md
```

如果当前项目已经采用不同 app 命名，Codex 必须优先沿用现有结构，不得强行改名或重排。

---

## 15. A 部分人工审查清单

用户合并前应检查：

```text
是否只完成 A 部分
是否没有实现 Hermes
是否没有实现 AlertEvent / HermesNotification
是否没有创建 notifications service
是否使用 Django settings，而不是自写配置框架
是否使用 Django ORM，而不是自写 ORM
是否使用 Django migrations，而不是手写建表系统
是否使用 Python logging / Django logging，而不是 print
是否使用 Celery / Celery Beat，而不是自写调度
是否使用 Redis 作为 Celery 基础设施
是否未实现 K 线采集
是否未实现策略
是否未实现风控
是否未实现交易
```

建议搜索：

```bash
grep -R "Hermes" apps tests config
grep -R "AlertEvent" apps tests config
grep -R "HermesNotification" apps tests config
grep -R "pymysql.connect" .
grep -R "mysql.connector" .
grep -R "CREATE TABLE" apps scripts tests
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

## 16. A 部分 Codex 输出总结要求

A 部分完成后，Codex 总结必须包含：

```text
是否只完成 A 部分
修改了哪些文件
新增了哪些文件
是否使用 Django settings
是否配置 MySQL 连接入口
是否配置 Redis / Celery 连接入口
是否实现 logging
是否实现 trace_id
是否实现 trigger_source
是否新增 tests
测试命令和结果
是否有无法运行的命令
是否存在未完成项
是否等待用户配置 MySQL / Redis 后再进入 B 部分
```

A 部分总结必须明确说明：

```text
A 部分没有实现 AlertEvent。
A 部分没有实现 HermesNotification。
A 部分没有实现 Hermes client。
A 部分没有实现 notifications service。
A 部分没有实现 K 线采集。
A 部分没有实现策略、风控、交易。
```

---

## 17. A 部分最终验收标准

A 部分完成后，必须满足：

```text
Django 项目可以启动。
settings 可以加载基础配置。
MySQL 配置入口存在。
Redis 配置入口存在。
Celery app 可初始化。
Celery Beat 基础配置存在。
logging 可用。
trace_id 可生成和传递。
trigger_source 有受控规范。
默认测试可以运行或说明环境限制。
默认测试不访问真实外部服务。
未实现 AlertEvent。
未实现 HermesNotification。
未实现 Hermes client。
未实现 notifications service。
未实现 K 线采集。
未实现策略。
未实现风控。
未实现交易。
未自研框架已有基础能力。
```

---

## 18. 下一步

A 部分完成并经用户验收后，用户配置并确认 MySQL / Redis。

之后才允许执行：

```text
docs/plans/001B_project_foundation_hermes_plan.md
```
