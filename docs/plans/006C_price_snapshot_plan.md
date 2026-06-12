# 006C PriceSnapshot 实现计划

> 目标路径：`docs/plans/006C_price_snapshot_plan.md`  
> 对应需求：`docs/requirements/price_snapshot.md`  
> 模块编号：006C  
> 当前状态：待实现  
> 上游：`006B Binance Account Sync`、测试 fixture  
> 下游：`007A OrderPlan`、`007B RiskCheck`、后续 `ExecutionPreparation`  
> 核心原则：先实现稳定、可追溯、可测试的价格事实层；P0 不接 WebSocket，不直接调用 Binance。

---

## 0. 实现边界

本计划实现 PriceSnapshot P0。

目标是让下游模块通过：

```text
price_snapshot_id
```

引用一个明确的价格事实，而不是直接读取 K 线、直接读 Binance 原始响应、或直接连接 WebSocket。

P0 只实现：

```text
PriceSnapshot 模型
PriceSnapshotService
从 Binance Account Sync 快照派生价格快照
manual fixture 创建价格快照
TTL 校验辅助
hash / 幂等
测试
```

P0 不实现：

```text
WebSocket
REST mark price 拉取
Binance client 初始化
盘口深度
执行前 price guard
滑点检查
真实下单
```

---

## 1. 实现前必须阅读

Codex 实现前必须完整阅读：

```text
AGENTS.md
README.md
docs/rules/project_invariants.md

docs/requirements/price_snapshot.md

docs/requirements/binance_account_sync.md
docs/plans/006B_binance_account_sync_plan.md

docs/requirements/decision_snapshot.md
docs/plans/006A_decision_snapshot_plan.md

docs/requirements/order_plan.md
docs/plans/007A_order_plan_plan.md

docs/requirements/risk_check.md
docs/plans/007B_risk_check_plan.md
```

并检查当前代码：

```text
apps/binance_account_sync/
apps/order_plan/
apps/risk_check/
apps/notifications/
```

---

## 2. 新增 app

建议新增：

```text
apps/price_snapshot/
```

建议结构：

```text
apps/price_snapshot/
  __init__.py
  apps.py
  models.py
  services.py
  selectors.py
  hashes.py
  constants.py
  management/
    __init__.py
    commands/
      __init__.py
      build_price_snapshot.py
  tests/
```

如果项目已有统一 market price app，可复用现有 app，但不得把 PriceSnapshot 临时塞进 OrderPlan 或 RiskCheck。

---

## 3. settings 配置

新增配置，默认值：

```python
ORDER_PLAN_PRICE_SNAPSHOT_MAX_AGE_SECONDS = 60
RISK_CHECK_PRICE_SNAPSHOT_MAX_AGE_SECONDS = 60
EXECUTION_PRICE_GUARD_MAX_AGE_SECONDS = 30
PRICE_SNAPSHOT_DEFAULT_SCHEMA_VERSION = "price_snapshot.v1"
```

说明：

```text
前两个用于 OrderPlan / RiskCheck 估值价格 TTL。
Execution 的 30 秒只作为后续模块配置预留，本模块不实现 price guard。
```

---

## 4. 模型设计

新增模型：

```text
PriceSnapshot
```

建议字段：

```text
id
exchange
symbol
market_type
account_domain
price_type
mark_price
price_unit
source
source_snapshot_type
source_snapshot_id
source_event_id
as_of_utc
received_at_utc
schema_version
price_snapshot_hash
is_usable
blocked_reason
raw_payload
trace_id
created_at
updated_at
```

建议枚举：

```text
exchange: binance
market_type: usds_m_futures / coin_m_futures
price_type: mark_price
source: account_sync / manual_fixture / rest_mark_price / websocket_mark_price
```

P0 只启用：

```text
price_type = mark_price
source = account_sync / manual_fixture
```

约束建议：

```text
mark_price > 0 由 Django validation 保证。
price_snapshot_hash 唯一。
market identity 字段加索引。
as_of_utc 加索引。
```

MySQL 低版本可能不强制 CHECK，仍需 service 层校验。

---

## 5. hash 实现

新增：

```text
apps/price_snapshot/hashes.py
```

实现：

```text
build_price_snapshot_hash(payload) -> str
```

hash 输入至少包括：

```text
schema_version
exchange
symbol
market_type
account_domain
price_type
mark_price
price_unit
source
source_snapshot_type
source_snapshot_id
source_event_id
as_of_utc
```

要求：

```text
Decimal 必须标准化为字符串。
UTC datetime 必须标准化为 ISO 格式。
JSON 排序稳定。
```

---

## 6. Service 设计

新增：

```text
PriceSnapshotService
```

核心方法建议：

```python
create_manual_snapshot(
    *,
    symbol: str,
    market_type: str,
    account_domain: str,
    mark_price: Decimal,
    price_unit: str,
    as_of_utc: datetime,
    trace_id: str | None = None,
    dry_run: bool = False,
) -> PriceSnapshot | dict
```

```python
create_from_account_sync(
    *,
    binance_sync_run_id: int,
    symbol: str,
    market_type: str,
    account_domain: str,
    trace_id: str | None = None,
    dry_run: bool = False,
) -> PriceSnapshot | dict
```

```python
validate_for_consumption(
    *,
    price_snapshot: PriceSnapshot,
    symbol: str,
    market_type: str,
    account_domain: str,
    max_age_seconds: int,
    reference_time_utc: datetime,
) -> ValidationResult
```

```python
get_latest_usable(
    *,
    symbol: str,
    market_type: str,
    account_domain: str,
    price_type: str = "mark_price",
    max_age_seconds: int | None = None,
    reference_time_utc: datetime | None = None,
) -> PriceSnapshot | None
```

要求：

```text
service 层必须 fail-closed。
不能伪造 mark_price。
不能跨 market_type / account_domain 查找价格。
不能自动 fallback 到别的 symbol。
```

---

## 7. 从 Account Sync 生成 PriceSnapshot

`create_from_account_sync` 必须：

```text
1. 读取指定 binance_sync_run_id。
2. 校验 BinanceSyncRun.status = succeeded。
3. 校验 exchange / market_type / account_domain / symbol 一致。
4. 从该批次中找到目标 symbol 的 position snapshot 或可提供 mark_price 的快照。
5. 校验 mark_price 存在且 > 0。
6. 创建 PriceSnapshot(source=account_sync)。
```

如果没有可用 mark_price：

```text
正式模式：返回 unusable / blocked summary，不能创建 usable 快照。
dry-run：返回 blocked 摘要，不写库。
```

不得为了通过测试写死价格。

---

## 8. TTL 校验

`validate_for_consumption` 必须检查：

```text
price_snapshot 存在
is_usable = True
symbol / market_type / account_domain 匹配
mark_price > 0
as_of_utc 存在
reference_time_utc - as_of_utc <= max_age_seconds
price_snapshot_hash 可重算并匹配
```

失败时返回明确 reason_code：

```text
price_snapshot_missing
price_snapshot_unusable
price_market_identity_mismatch
invalid_mark_price
stale_price_snapshot
price_snapshot_hash_mismatch
```

本方法不写 AlertEvent。由 OrderPlan / RiskCheck / ExecutionPreparation 根据自己的 blocked 结果写 AlertEvent。

---

## 9. Selector

新增：

```text
apps/price_snapshot/selectors.py
```

建议函数：

```python
get_price_snapshot_by_id(price_snapshot_id: int) -> PriceSnapshot | None
get_latest_price_snapshot(symbol, market_type, account_domain, price_type="mark_price") -> PriceSnapshot | None
```

selector 只读，不做业务判断。

业务判断放在 service。

---

## 10. Management command

新增命令：

```text
python manage.py build_price_snapshot
```

参数建议：

```text
--symbol BTCUSDT
--market-type usds_m_futures
--account-domain usds_m_futures
--binance-sync-run-id 123
--mark-price 100000
--price-unit USDT
--as-of-utc 2026-01-01T00:00:00Z
--source account_sync|manual_fixture
--dry-run
```

规则：

```text
source=account_sync 时必须提供 binance_sync_run_id。
source=manual_fixture 时必须显式提供 mark_price / price_unit / as_of_utc。
dry-run 不写库。
```

命令用于本地验证和测试，不负责真实拉取 Binance 价格。

---

## 11. 与 OrderPlan 的集成要求

OrderPlan 后续实现时必须：

```text
1. 通过 price_snapshot_id 或 selector 获取 PriceSnapshot。
2. 调用 validate_for_consumption(max_age_seconds=ORDER_PLAN_PRICE_SNAPSHOT_MAX_AGE_SECONDS)。
3. 将 price_snapshot_id 写入 OrderPlan / CandidateOrderIntent。
4. 使用 mark_price 计算 target_quantity / target_contracts。
5. 不直接读取 Kline close price。
6. 不直接调用 Binance。
```

如果 PriceSnapshot 缺失、过期、不可用：

```text
OrderPlan.status = BLOCKED
reason_code = stale_price_snapshot / price_snapshot_missing / price_snapshot_unusable
写 AlertEvent
```

---

## 12. 与 RiskCheck 的集成要求

RiskCheck 后续实现时必须：

```text
1. 读取 CandidateOrderIntent 绑定的 price_snapshot_id。
2. 调用 validate_for_consumption(max_age_seconds=RISK_CHECK_PRICE_SNAPSHOT_MAX_AGE_SECONDS)。
3. 使用 mark_price 估算 increase_risk component 的 notional / margin_required。
4. 将 price_snapshot_id / mark_price 写入 risk_measures / evidence。
5. 不执行最终 30 秒 / 0.5% price guard。
```

如果 PriceSnapshot 缺失、过期、不可用：

```text
RiskCheckResult.status = BLOCKED
reason_code = stale_price_snapshot / price_snapshot_missing / price_snapshot_unusable
写 AlertEvent
```

---

## 13. 测试计划

新增测试目录：

```text
tests/price_snapshot/
```

或按项目现有测试风格放置。

必须覆盖：

```text
1. manual_fixture 可创建 usable PriceSnapshot。
2. mark_price 缺失 / 0 / 负数 → unusable 或 validation error。
3. market identity 缺失 → fail-closed。
4. 相同输入重复创建 → 幂等返回已有或不重复创建。
5. 不同 as_of_utc / mark_price → 生成不同 hash。
6. 从 succeeded BinanceSyncRun 创建 PriceSnapshot。
7. BinanceSyncRun 不存在 / failed / market_type mismatch → blocked summary。
8. Account Sync 快照缺 mark_price → blocked summary。
9. validate_for_consumption 成功。
10. validate_for_consumption 发现 stale_price_snapshot。
11. validate_for_consumption 发现 market_identity_mismatch。
12. validate_for_consumption 发现 hash mismatch。
13. dry-run 不写库。
14. management command dry-run 不写库。
```

---

## 14. 实现与接入

实现时需要：

```text
新增 app 到 INSTALLED_APPS。
新增 PriceSnapshot migration。
确保 makemigrations --check 通过。
```

不得修改：

```text
Kline 模型
MarketSnapshot 模型
DecisionSnapshot 已有数据
RiskCheckResult 已有历史数据
```

如果 OrderPlan / RiskCheck 当前还未接入 PriceSnapshot，本计划只新增基础设施，不强制改下游业务代码。

---

## 15. 非目标提醒

Codex 不得在本任务中实现：

```text
WebSocket
REST 价格拉取
Binance client
ExecutionPreparation
price_deviation <= 0.5%
真实下单
订单查询
成交追踪
滑点模型
盘口深度
```

如果发现下游缺价格，应通过 PriceSnapshotService / manual fixture / account_sync source 补足，不得直接在下游硬编码价格。

---

## 16. 完成后运行

完成后运行：

```powershell
.\.venv\Scripts\python.exe manage.py makemigrations --check --dry-run
.\.venv\Scripts\python.exe manage.py test
.\.venv\Scripts\pytest.exe
```

如果环境没有 pytest 或已有跳过项，必须在结果中说明。

---

## 17. 完成后汇总

完成后请汇总：

```text
1. 新增 / 修改了哪些文件。
2. PriceSnapshot 模型字段。
3. PriceSnapshot hash 如何计算。
4. PriceSnapshot 如何从 Account Sync 创建。
5. manual fixture 如何创建。
6. TTL 校验如何实现。
7. OrderPlan / RiskCheck 后续如何消费。
8. 新增 / 修改了哪些测试。
9. 测试结果。
10. 是否发现与 006A / 006B / 007A / 007B 文档冲突。
11. 是否有任何越界实现。
```
