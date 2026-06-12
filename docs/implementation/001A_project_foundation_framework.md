# 001A 项目基础框架实现说明

## 阶段边界

本阶段只实现 `docs/plans/001A_project_foundation_framework_plan.md` 中的 A 部分基础框架。

本阶段没有实现：

```text
AlertEvent
HermesNotification
Hermes client
notifications service
Hermes Celery task
Hermes webhook 调用
任何通知发送逻辑
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
Binance 账户 / 仓位 / 下单 / 杠杆接口
WebSocket 行情
自动交易
回测系统
```

## Django 项目结构

001A 新增标准 Django 项目基础结构：

```text
manage.py
config/
apps/core/
tests/core/
```

`config/settings.py` 使用 Django settings 管理项目配置，没有自研配置框架。

settings 启动早期通过 `python-dotenv` 加载项目根目录 `.env`：

```text
BASE_DIR / ".env"
```

加载规则为 `override=False`，即已存在的系统环境变量优先，`.env` 用作本地开发配置入口。

`apps/core/` 只放基础框架工具：

```text
trace_id 生成与补齐
trigger_source 受控枚举与校验
基础异常类型
Celery 最小 healthcheck task
```

## settings / env 配置

基础环境变量入口位于 `.env.example`，包含：

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

001A 不配置 `HERMES_*`，Hermes 属于 001B。

生产环境中如果仍使用 001A 默认开发密钥，settings 会抛出 `ImproperlyConfigured`。

## MySQL 配置入口

`settings.FOUNDATION_MYSQL_CONFIG` 提供 MySQL 配置入口。

非测试运行默认使用：

```text
django.db.backends.mysql
```

001A 不要求真实 MySQL 连接成功，不写真实密码，不执行真实业务写库。

测试运行时使用 SQLite 内存库 fallback，避免默认测试访问生产或本地 MySQL。

## Redis / Celery 配置入口

Redis 只作为以下配置入口：

```text
REDIS_URL
CELERY_BROKER_URL
CELERY_RESULT_BACKEND
```

Celery app 位于 `config/celery.py`，从 Django settings 读取配置。

Celery Beat 基础配置位于：

```text
settings.CELERY_BEAT_SCHEDULE = {}
```

001A 不要求真实 Redis 连接成功，不把 Redis 作为核心业务事实存储。

## logging

`settings.LOGGING` 使用 Python logging / Django logging。

日志格式包含：

```text
timestamp
level
logger name
message
```

后续业务日志应在 message 或 LoggerAdapter 上下文中显式携带 `trace_id`，不得打印密钥、token、数据库密码或 webhook。

## trace_id

`apps/core/trace.py` 提供：

```text
generate_trace_id()
ensure_trace_id(input_trace_id)
```

`trace_id` 用于链路追踪，不作为业务幂等键。

## trigger_source

`apps/core/trigger_source.py` 提供受控来源：

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

非法 `trigger_source` 会抛出 `InvalidTriggerSourceError`。

## 基础异常

`apps/core/exceptions.py` 提供：

```text
ProjectFoundationError
ConfigurationError
DatabaseConnectionError
RedisConnectionError
ExternalServiceError
InvalidTriggerSourceError
```

`NotificationSendError` 未实现，属于 001B。

## 测试方式

001A 默认验收命令：

```bash
python manage.py check
python manage.py makemigrations --check
python manage.py test
```

通过标准：

```text
Django settings 可加载
MySQL 配置入口存在
测试不访问生产 MySQL
Redis / Celery 配置入口存在
Celery app 可导入
trace_id 可生成和补齐
trigger_source 可校验
非法 trigger_source 可拒绝
基础异常可导入
```

失败标准：

```text
默认测试访问真实 MySQL / Redis / Binance / Hermes
出现 AlertEvent / HermesNotification / notifications / Hermes client 代码
绕过 Django settings / Django ORM / Django migrations / Celery / logging / Django test framework
出现交易、风控、策略、行情采集或 Binance 真实接口实现
```

## 人工门槛

001A 完成后必须停止。

进入 001B 前，用户需要自行配置并确认：

```text
真实 MySQL 环境变量
Django 可连接 MySQL
migrations 基础命令可运行或已知环境限制
真实 Redis 环境变量
Celery 可使用 Redis 配置
001A 测试已通过或环境限制已明确
```
