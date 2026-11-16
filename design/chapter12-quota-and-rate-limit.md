# 第十二章 配额与限流设计

## 本章目标

通过本章，读者将理解：

- 壬远AI网关如何通过 `QuotaPlan` 为 API-Key 与 Entity 分配 Token 或 RMB 两种单位的配额；
- 为什么把 **Redis** 作为配额余额的唯一真实来源，以及管理面如何通过直接读取 Redis 查询实时余额；
- 自然周、自然月的周期重置逻辑，以及 `last_reset_at` 在重置边界判断中的作用；
- 周期重置如何通过 Redis 分布式锁在多实例部署下互斥执行，以及原子 SET 与条件更新如何保证重置幂等；
- `quota=0`（无余额计划）在控制面与数据面两侧的一致语义；
- `RateLimitPolicy` 的 TPM、RPM、并发限制模型，以及策略如何按 Entity 层级向上合并并导出到 BFE；
- 批量与异步任务（Batch）场景下 `batch_limits` 限流段与后付费配额的预留-结算-释放模型；
- RMB 配额如何结合 Provider 时段模板与 Model 分时段价格实现高峰/空闲差异化计费；
- 典型的配额与限流配置示例。

---

## 为什么需要配额与限流

企业在统一接入大模型服务后，通常面临两类资源失控风险：

1. **预算失控**：某个业务团队或某个 API-Key 短期内消耗大量 Token 或产生高额费用，导致整体预算被击穿；
2. **流量失控**：突发请求淹没后端模型服务，造成延迟升高、错误率增加，甚至触发上游限流。

配额（Quota）解决的是"一共能用多少"的问题，限流（Rate Limit）解决的是"单位时间内能用多少"的问题。两者相辅相成：配额保证总体预算可控，限流保证瞬时流量平滑。壬远AI网关把配额计划与限流策略都绑定到 API-Key 或 Entity 上，使管理员能够按应用、按团队、按模型细粒度地控制资源使用。

---

## QuotaPlan 模型：total_token 与 RMB 两种单位

`QuotaPlan`（配额计划）是壬远AI网关中最基础的配额抽象。它既可以按 Token 总量计费，也可以按人民币金额计费，分别对应 `unit = total_token` 与 `unit = RMB`。

### 数据表结构

配额计划持久化在 `quota_plans` 表中，核心字段如下（完整定义见 `ai-gateway-api/design-docs/sys-design/details/配额余额同步机制.md`）：

```sql
CREATE TABLE `quota_plans` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `unlimited` TINYINT(1) DEFAULT 1,
  `pass_when_no_enough_quota` TINYINT(1) DEFAULT 0,
  `quota` DECIMAL(18,8) DEFAULT 0,
  `unit` VARCHAR(32) DEFAULT 'total_token',
  `reset_period` VARCHAR(16) DEFAULT 'never',
  `last_reset_at` DATETIME DEFAULT NULL,
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  INDEX `idx_unlimited` (`unlimited`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='配额计划表';
```

| 字段 | 说明 |
|------|------|
| `unlimited` | 是否为无限配额。`true` 时不执行配额检查，余额展示为 sentinel 值 `100000000`。 |
| `pass_when_no_enough_quota` | 配额不足时是否仍放行请求，常用于灰度或测试场景。 |
| `quota` | 配额总量。`total_token` 时存整数，`RMB` 时最多保留 8 位小数。 |
| `unit` | 配额单位，可选 `total_token` 或 `RMB`。 |
| `reset_period` | 重置周期，可选 `never`、`weekly`、`monthly`。 |
| `last_reset_at` | 上次重置时间，周期重置据此判断是否跨越自然周/月边界。 |

### quota=0 的语义：无余额计划

`total_token` 与 `RMB` 两种单位都允许 `quota = 0`，表示"无余额计划"：配置本身合法，但绑定该计划的请求会因余额耗尽被拒绝。这与负值有本质区别——负值是非法配置，会在配置加载阶段直接报错；而 `0` 是一个有意义的业务取值（例如临时冻结某个计划，但不解绑其与 API-Key / Entity 的绑定关系）。两侧的处理机制如下：

- **控制面**：`QuotaPlanManager.adjustQuota`（`ai-gateway-api/model/quota/quota_plan_manager.go`）在配额变化时按 `remaining = max(0, newQuota - used)` 计算并把差额写入 Redis。配额更新为 0 时 remaining 随之清零、Redis 余额同步为 0。
- **数据面 BFE**：
  - 配置加载阶段，`token_rule_load.go` 的 `quotaPlanCheck` 只拒绝 `Quota < 0` 的配置，`quota=0` 作为合法配置正常加载，每个无余额计划独立生效，不影响同产品（Product）下的其他计划；
  - 请求阶段，`QuotaPlan.HasBalance`（`bfe/bfe_modules/mod_ai_token_auth/token.go`）判定余额：Redis 余额 key 存在时按值判断；key 缺失（`redis_client.IsKeyNotFound`）时视为余额 0，同样返回"无余额"，命中该计划的请求返回 429 QuotaExhausted，而不是内部错误。

### 两种单位的适用场景

- **`total_token`**：适用于按 Token 计费的模型（如 OpenAI、Anthropic）。管理员可以直接限制每月可用的输入+输出 Token 总量。
- **`RMB`**：适用于需要把不同模型、不同价格统一折算为成本的场景。系统会根据请求消耗的 Token 和模型单价，实时折算为人民币并从余额中扣减。

### RMB 配额精度

为了避免浮点误差，`ai-gateway-api` 与 BFE 在 Redis 内部统一把 RMB 金额乘以 `1e8` 后转为定点整数存储；对外展示时统一按 4 位小数输出。例如 `0.0000015` 元在内部表示为 `150`，序列化时通过 `strconv.FormatFloat(v, 'f', -1, 64)` 强制使用十进制表示法，避免科学计数法导致可读性问题或中间层截断。

---

## Redis 作为余额唯一真实来源

配额消耗发生在请求链路（BFE 数据面实时扣减），需要高并发、低延迟；而管理面（AI Gateway API）需要向 OpenAPI 展示 `used` / `remaining`，又希望数据实时、一致。因此，当前架构把 Redis 直接作为配额余额的唯一真实来源：

- Redis 存储当前剩余配额，请求链路直接读写；
- OpenAPI 查询余额时直接读取 Redis；
- `last_reset_at` 只保存在 `quota_plans` 表中，周期重置只操作这一张表；
- 手动重置 / 周期重置时，同步更新 Redis 与 `quota_plans.last_reset_at`。

```mermaid
flowchart TD
    A[API请求] -->|BFE 数据面实时扣减| B["Redis<br/>QUOTA_&lt;key&gt;"]
    B -->|放行或拒绝| C[模型服务]
    D[AI Gateway API<br/>控制面] -->|OpenAPI 查询余额| B
    E[QuotaResetScheduler<br/>每分钟触发] -->|ResetExpiredBalances| D
    D -->|更新 last_reset_at| F[(quota_plans)]
    D -->|SetRemaining / ResetToQuota / ResetToQuotaAtomic| B
```

在该架构中，`QuotaCache` 接口（定义于 `ai-gateway-api/model/quotacache/quotacache.go`，实现于 `ai-gateway-api/model/quotacache/redis.go`）封装了对 Redis 的所有操作，包括 `GetRemaining`、`BatchGetRemaining`、`SetRemaining`、`ResetToQuota`、`ResetToQuotaAtomic` 和 `DeleteKeys`。

### Redis Key 规则

Redis Key 由 `AIUsedQuotaKey` 生成（`ai-gateway-api/stateful/config_redis.go`）：

```go
func AIUsedQuotaKey(key string) string {
    return fmt.Sprintf("QUOTA_%s", key)
}
```

| 对象类型 | Redis Key 示例 | 说明 |
|----------|----------------|------|
| API-Key | `QUOTA_AI_ak-2v8x9k3m7p` | 以 API-Key 的实际 `key` 值作为后缀 |
| Entity | `QUOTA_entity-1` | 以 `entity_id` 作为后缀 |
| 值语义 | 当前剩余配额 | `total_token` 为整数，`RMB` 为 1e-8 元定点整数 |

Key 不再拼接 `KeyCreateAt` 时间戳，生命周期与 API-Key / Entity 保持一致，避免 Key 膨胀。API-Key 或 Entity 删除时，会调用 `DeleteKeys` 主动清理对应 Redis Key。

### 原子扣减与归零策略

不同场景下系统采用两种不同的 Redis 写入策略。

**增量调整（手动重置 / 初始余额写入）**：`SetRemaining` 与 `ResetToQuota` 使用原子 `IncrBy(delta)` 而非 `SET`：

- 并发场景下，`SET` 会覆盖其他请求刚刚扣减的计数，导致配额透支；
- `IncrBy(delta)` 基于当前值做增量调整，可与其他扣减操作串行化；
- Key 不存在时直接 `IncrBy(quotaTotal)` 即可完成初始化。

**原子 SET（周期重置）**：周期重置使用 `ResetToQuotaAtomic`，通过 Lua 脚本执行 `redis.call('set', KEYS[1], ARGV[1])` 把余额直接置为配额总量。`IncrBy(delta)` 需要先读出当前值、再计算差值后写入，读与写之间存在时间窗：若期间有请求完成扣减，基于旧值算出的 delta 会使最终结果偏离配额总量；原子 SET 不依赖当前值，最终结果恒等于配额总量，只取决于 SET 与并发扣减的先后次序，从而消除了 read-modify-write 竞争。配合分布式锁（见下文）把全局重置收敛到单一实例后，SET 与请求链路扣减的竞争窗口也极小。

### 欠费模型：余额可为负

配额扣减采用全额 `DECRBY`（不做 `math.min(current, amount)` 截断），余额允许为负，欠费被如实记录：

- 数据面扣减不做截断，余额可跌破 0；鉴权期 `HasBalance` 对负余额判“无余额”，请求返回 429 QuotaExhausted（行为不变），并提供 `QuotaOverage` 计数 + WARN 供欠费告警；
- 控制面对外展示：`used = quota − remaining` 如实反映超额（**可大于 `quota`**），`Balance.Remaining` 在 Redis 余额为负时仍展示 0；调额路径（`adjustQuota` 等三处）不钳零，“欠费中调额”保留欠费（`remaining` 可为负，鉴权继续拒绝）；
- 控制台侧用量百分比封顶 100% 展示，欠费态可识别；
- 无数据迁移：Redis 单 key 模型不变；注意**升级前已被钉在 0 的存量 key，其历史超额无法找回**。

---

## 自然周/月重置与过期重置

### 周期重置策略

`QuotaPlan.reset_period` 支持三种取值：

- `never`：不自动重置；
- `weekly`：自然周，每周一 00:00:00 重置；
- `monthly`：自然月，每月 1 日 00:00:00 重置。

`QuotaResetScheduler`（`ai-gateway-api/model/quota/scheduler.go`）每分钟执行一次 `ResetExpiredBalances`，仅对需要重置的计划进行处理。

### 重置边界判断

`BalanceSyncManager`（`ai-gateway-api/model/quota/balance_sync.go`）通过注入的 `Clock` 接口获取当前时间，再调用 `shouldResetByPeriod` 判断是否跨越周期边界：

```go
func (m *BalanceSyncManager) shouldResetByPeriod(
    lastResetAt *time.Time,
    resetPeriod string,
    now time.Time,
) bool {
    if lastResetAt == nil {
        return true
    }
    switch resetPeriod {
    case "weekly":
        return getWeekStart(now).After(getWeekStart(*lastResetAt))
    case "monthly":
        return getMonthStart(now).After(getMonthStart(*lastResetAt))
    }
    return false
}
```

`getWeekStart` 与 `getMonthStart` 将时间归一化为周一或每月 1 日的 00:00:00（本地时区）。由于判断基于周/月起始时间点的比较，即使调度器因故错过某个时刻，只要当前周期起始点晚于上次重置时的周期起始点，就会触发重置，具备自愈能力。

### 重置执行流程

```mermaid
sequenceDiagram
    participant S as QuotaResetScheduler
    participant L as Redis 分布式锁
    participant M as BalanceSyncManager
    participant DB as quota_plans
    participant R as Redis

    S->>L: Acquire(quota:reset:scheduler:lock, token, TTL=5min)
    alt 加锁成功
        S->>S: 启动看门狗按 TTL/3 续期
        S->>M: 每分钟触发 ResetExpiredBalances
        M->>DB: 查询 reset_period=weekly/monthly 且非无限配额的计划
        loop 每个计划
            M->>M: shouldResetByPeriod(last_reset_at, now)
            alt 需要重置
                M->>R: ResetToQuotaAtomic 原子 SET 余额为 quota
                M->>DB: 条件更新 last_reset_at=now（LastResetAtBefore=周期起点）
            end
        end
        S->>L: Release（校验 instance token 后删除）
    else 加锁失败
        S->>S: 跳过本轮重置
    end
```

### 手动重置接口

除了周期重置，系统还提供手动重置接口：

- `POST /api-keys/{id}/quota-plan/reset`
- `POST /entities/{id}/quota-plan/reset`

手动重置调用 `QuotaPlanManager.ResetBalance(..., updateLastResetAt=false)`，即只重置 Redis 余额，不更新 `last_reset_at`，避免干扰周期调度器对自然周/月的判断。若传入新的 `quota`，则同时更新 `quota_plans.quota`。

### 手动触发周期重置（Inner API）

除按对象重置外，控制面还提供 Inner API `POST /inner-api/v1/quota/trigger-reset`（`ai-gateway-api/endpoints/innerapi_v1/quota_reset/quota_reset.go`），手动触发一次与定时任务完全相同的周期重置流程：同样经过分布式锁保护、执行 `ResetExpiredBalances`，但不影响定时任务的下次执行时间。接口同步返回 `{"status":"ok"}`，主要供内部运维使用。

### 多实例部署与分布式锁

`QuotaResetScheduler` 在每个 AI Gateway API 实例中独立启动，多实例部署时所有实例都会尝试执行重置。调度器通过 Redis 分布式锁保证同一时刻仅一个实例真正执行 `ResetExpiredBalances()`：

- **加锁**：每轮先通过 `quotacache.DistributedLock`（`ai-gateway-api/model/quotacache/lock.go`）申请锁，锁 key 为 `quota:reset:scheduler:lock`，TTL 5 分钟；加锁失败（其他实例持有）则跳过本轮，等待下一轮再竞争。
- **令牌与释放**：每个实例启动时生成唯一的 `instanceToken`（UUID），加锁时写入锁 value；释放锁时通过 Lua 脚本校验 `GET == token` 后才 `DEL`，避免持锁实例宕机后锁被其他实例误删，也避免过期后旧持有者误删新持有者的锁。
- **看门狗续期**：持锁期间启动看门狗协程，按 TTL/3（约 100 秒）周期对锁执行 `EXPIRE` 续期，防止重置耗时超过 TTL 导致锁提前过期、其他实例中途抢锁造成并发重置。
- **幂等兜底**：即使锁意外失效出现同周期重复重置，原子 SET 本身幂等；同时 `last_reset_at` 采用条件更新（`QuotaPlanFilter` 带有 `LastResetAtBefore = 周期起点` 字段，`ai-gateway-api/model/quota/quota_plan.go`），同周期内第二次更新的影响行数为 0，不再推进重置水位。
- **先认领后重置**：调度器先做条件认领更新（`last_reset_at IS NULL OR last_reset_at < 周期起点`，`TQuotaPlanClaimPeriodReset`），仅当影响行数 > 0 时才执行本轮的 Redis 原子重置，同一周期内全局至多重置一次；`weekly` / `monthly` 计划在创建时把 `last_reset_at` 初始化为创建时间（`never` 保持 NULL），使 SQL 三值逻辑（`NULL < x` 恒不成立）不会让存量 NULL 行陷入"认领恒 0 行却仍报成功"的每分钟重复重置。

---

## RateLimitPolicy：TPM、RPM、并发限制

### 策略模型

`RateLimitPolicy`（限流策略）用于控制 API-Key 或 Entity 对后端 AI 模型的访问速率，支持三类限制：

- **TPM**（Tokens Per Minute）：每分钟 Token 消耗上限；
- **RPM**（Requests Per Minute）：每分钟请求次数上限；
- **并发数**（Max Concurrency）：同时处理的请求数上限。

策略持久化在 `rate_limit_policies` 表中（完整定义见 `ai-gateway-api/design-docs/sys-design/details/限流策略与导出.md`）：

```sql
CREATE TABLE rate_limit_policies (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description VARCHAR(255) DEFAULT NULL,
    product_name VARCHAR(255) NOT NULL,
    enabled TINYINT(1) NOT NULL DEFAULT 1,
    max_concurrency INT DEFAULT NULL,
    tpm_configs JSON DEFAULT NULL,
    rpm_configs JSON DEFAULT NULL,
    batch_limits JSON DEFAULT NULL,
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### TPM 与 RPM 规则结构

TPM 与 RPM 规则以 JSON 数组形式存储在 `tpm_configs` 与 `rpm_configs` 字段中：

```json
{
  "tpm_configs": [
    {
      "name": "tpm_gpt4",
      "model": "gpt-4",
      "window_minutes": 1,
      "max_tokens": 100000,
      "step_minutes": 1
    }
  ],
  "rpm_configs": [
    {
      "name": "rpm_gpt4",
      "model": "gpt-4",
      "window_minutes": 1,
      "max_requests": 1000,
      "burst": 1
    }
  ]
}
```

`model` 支持具体模型名或通配符 `*`，未命中具体模型时使用默认限制。`name` 在同一策略内唯一，是规则导出与 Redis Key 的标识。

### 校验规则

`RateLimitPolicyManager` 在创建/更新时执行严格校验：

- `name` 在 `product_name` 内唯一；
- `max_concurrency` 必须 ≥ 0；
- 每条规则的 `name` 必填、非空、长度 1-128 字符，字符集限制为 `[a-zA-Z0-9_-]`；
- `name` 在同一策略内唯一；重命名一条规则等价于删除旧规则并新增规则（见下文“Redis Key 稳定性”）；
- `model` 不能为空，`limit` 必须 ≥ 0；
- `enabled=true` 的策略，`rules` 至少要配置 `tpm`、`rpm`、`max_concurrency`、`batch_limits` 四者之一；四者全部缺失时返回 422，并在文案中提示需要配置的维度；
- `tpm` / `rpm` 每条规则的 `limit`、`max_concurrency`、`batch_limits` 各数值字段均须 ≥ 0；0 是合法取值，`max_concurrency=0` 与 `batch_limits` 中取 0 的字段在数据面不启用该维度。

### batch_limits：批量任务限流段

为支撑批量与异步任务（`/v1/files`、`/v1/batches` 端点族，见[第三十二章 Token认证与配额模块实现：mod_ai_token_auth](../implementation/chapter32-mod-ai-token-auth.md)），限流策略的 `rules` 含可选的 `batch_limits` 段。该段属于共享的 `RateLimitPolicy` 类型，既可出现在 `/rate-limit-policies` 资源中，也可出现在 API-Key 与 Entity 的内联 `rate_limit_policy.rules` 中：

```json
{
  "batch_limits": {
    "max_create_rpm": 60,
    "max_active_batches": 10,
    "max_file_bytes": 104857600,
    "max_file_lines": 1000000
  }
}
```

| 字段 | 说明 |
|------|------|
| `max_create_rpm` | 每分钟允许创建的 batch 数上限（create 操作 RPM） |
| `max_active_batches` | 单个 API-Key 同时处于活跃（未终态）状态的 batch 数上限 |
| `max_file_bytes` | 上传文件字节数硬顶 |
| `max_file_lines` | 上传文件行数硬顶 |

`batch_limits` 省略或为 `null` 表示该策略不参与批量限流。控制面导出时把该段写入 BFE `ai_rate_limit.data` 的 `rules.batch` 段；BFE 侧 `mod_ai_rate_limit` 按操作类型（`bfe_basic.ClassifyBatchOp` 分类的 create / upload 等）分别执行 create RPM、活跃 batch 数、文件字节/行数硬顶检查：create RPM 与活跃 batch 数超限返回 429，文件硬顶超限返回 413 `BATCH_FILE_TOO_LARGE`。文件字节/行数硬顶还会与 `mod_ai_batch.data` 中的全局硬顶合并取最小值作为生效值，即使 API-Key 未绑定任何限流策略，全局硬顶同样生效（实现细节见[第三十三章 限流模块实现：mod_ai_rate_limit](../implementation/chapter33-mod-ai-rate-limit.md)）。

`batch_limits` 的配置入口与 TPM/RPM/并发一致：独立策略资源（`POST/PUT /rate-limit-policies`）与 API-Key / Entity 的内联 `rate_limit_policy` 使用同一 `RateLimitPolicy` 结构，由同一 `validate.RateLimitPolicy` 校验（`endpoints/openapi_v1/api_key/checker.go`、`endpoints/openapi_v1/entity/validator.go`），内联策略同样落库到 `rate_limit_policies` 行。各维度按 API-Key 计数（`max_active_batches` 比较该 `ClientKeyId` 的活跃 batch 数），并随 Entity 层级向上合并：API-Key 与各层 Entity 的策略都会参与检查，同一维度按 AND 生效（等价于取最小值），文件字节/行数再与 `mod_ai_batch.data` 全局硬顶取最小值。控制台当前未提供该段的表单，可经 OpenAPI 配置。

---

## 层级合并与导出

### 引用关系

API-Key 与 Entity 均通过 `rate_limit_policy_id` 字段引用限流策略：

| 维度 | 配额计划 | 限流策略 |
|------|----------|----------|
| 表 | `quota_plans` | `rate_limit_policies` |
| 引用字段 | `quota_plan_id` | `rate_limit_policy_id` |
| 层级合并 | 收集所有层级 QuotaPlan | 收集所有层级 Policy ID |
| 导出 Redis Key | `QUOTA_xxx` | `default_bfe_rlp-<id>_RL_TPM_rlp-<id>_<name>` / `default_bfe_rlp-<id>_RL_RPM_rlp-<id>_<name>` |
| 余额同步 | 有 | 无 |

### Entity 层级向上合并

导出时，`RateLimitPolicyGenerator`（`ai-gateway-api/model/rate_limit_policy/rate_limit_policy_manager.go`）对每个 API-Key 递归收集其自身及所有父 Entity 绑定的策略 ID：

```go
func (m *RateLimitPolicyManager) fetchEntityRateLimitPolicyIDs(ctx context.Context, entity *EntityParam) ([]int64, error) {
    policyIDs := make([]int64, 0)
    if entity.RateLimitPolicyID != nil {
        policyIDs = append(policyIDs, *entity.RateLimitPolicyID)
    }
    if entity.ParentID != nil && *entity.ParentID != "" {
        parent, err := m.entityStorager.FetchEntity(ctx, &EntityFilter{EntityID: entity.ParentID})
        if parent != nil {
            parentPolicyIDs, _ := m.fetchEntityRateLimitPolicyIDs(ctx, parent)
            policyIDs = append(policyIDs, parentPolicyIDs...)
        }
    }
    return policyIDs, nil
}
```

### 导出文件

导出结果写入两个文件：

- `rate_limit_policies.json`：所有启用状态的限流策略；
- `api_key_rl_policy_bindings.json`：API-Key 到策略的绑定关系。

导出的策略名称统一格式为 `rlp-<policy_id>`，避免命名冲突并便于 BFE 索引。只有 `enabled=true` 的策略才会被导出并生成绑定。

### Redis Key 稳定性

控制面在导出时为每条规则生成 Redis Key，规则名是 Key 的组成部分（`model/shared/rate_limit_redis_key.go` 的 `BuildBFERateLimitRedisKey`）：

```go
// model/shared/rate_limit_redis_key.go
func BuildBFERateLimitRedisKey(policyID int64, ruleType string, name string) string {
    return fmt.Sprintf("default_bfe_rlp-%d_%s_rlp-%d_%s", policyID, ruleType, policyID, name)
}
```

TPM / RPM 规则分别传入 `ruleType` 为 `RL_TPM` / `RL_RPM`，`name` 为规则名，生成的 Key 形如 `default_bfe_rlp-1_RL_TPM_rlp-1_tpm_1min`。BFE 侧直接使用配置中下发的完整 `redis_key`。

由于规则名直接编码在 Key 中，重命名一条规则等价于删除旧规则并新增一条规则：`DiffRateLimitRedisKeys` 按 name 比较新旧策略，被移除的旧规则 Key 会被清理，新规则使用新 Key、计数器从零开始。修改规则的 `model` 等其它字段不改变 Key，计数器不重置。

```mermaid
flowchart TD
    A[API-Key] -->|绑定| B[RateLimitPolicy A]
    A -->|挂载| C[Entity]
    C -->|绑定| D[RateLimitPolicy B]
    C -->|parent_id| E[父 Entity]
    E -->|绑定| F[RateLimitPolicy C]

    G[RateLimitPolicyGenerator] -->|收集启用策略| H[rate_limit_policies.json]
    G -->|生成 API-Key 绑定| I[api_key_rl_policy_bindings.json]
    G -->|为每条规则生成 redis_key| J["default_bfe_rlp-&lt;id&gt;_RL_TPM_rlp-&lt;id&gt;_&lt;name&gt;<br/>default_bfe_rlp-&lt;id&gt;_RL_RPM_rlp-&lt;id&gt;_&lt;name&gt;"]

    H --> K[BFE mod_ai_rate_limit]
    I --> K
    J --> K
```

### BFE 侧预期行为

BFE 收到配置后：

1. 根据 `api_key_rl_policy_bindings.json` 找到 API-Key 对应的策略列表；
2. 请求到达时，按转发后目标模型匹配 `rules.tpm` / `rules.rpm` 中的规则，优先匹配具体模型名，未命中时使用 `*` 默认限制；
3. 命中规则后，使用规则中的 `redis_key` 构建 Redis 计数器 key 进行限流检查；
4. 同时检查 `max_concurrency`；
5. 任一限制超出时返回 429 Too Many Requests。

## 上游 Key 耗尽：429 NO_AVAILABLE_KEY

限流返回的 429 表示"调用方超限"，而 `NO_AVAILABLE_KEY` 表示"网关上没有可用的上游凭证"，两者同为 429 但归因不同：

- 集群内候选 Key 全部处于惩罚期（429 档与 401/402/403 档的惩罚时长由集群配置 `key_affinity.penalty_ttl_429` / `penalty_ttl_auth` 决定，默认 60s / **300s**；仅在开启会话亲和惩罚时写入）时，Key 选择返回空候选，转发循环合成 429 `NO_AVAILABLE_KEY`（type `rate_limit_error`，message `no available upstream api key for this cluster`），响应带 `Retry-After` 与 `error.details.retry_after_seconds`，两者固定为 60s；
- Key 惩罚过滤跳过 `Weight=0` 的 Key：它们既不参与选择，也不计入候选数；
- 候选集为空即表现为该集群的熔断：本次 attempt 不再轮换 Key，按 fallback 规则降级到备用集群；TTL 到期后 Key 自动恢复可用，无需人工介入。

该错误不消耗调用方的 QuotaPlan 余额，也不计入 RateLimitPolicy 的限流计数。排障时应检查上游 Provider 的凭证有效性与集群 Key 池规模，而不是调整限流策略。

---

## RMB 配额分时段定价

### 背景

随着 DeepSeek 等模型提供商采用"高峰 / 空闲"分时段定价策略，BFE RMB 配额扣费需要具备按请求发生时刻匹配不同价格的能力。壬远AI网关把时段模板放在 `/providers`，把分时段价格放在 `/model-prices`，实现 Provider 与价格的分离管理。

### 核心概念

| 概念 | 说明 |
|------|------|
| **Tier** | 按时间维度划分的价格层级，如 `peak`（高峰）。 |
| **TimeRange** | 一个时段定义，包含 `weekdays`（0=周日）、`start` / `end`（HH:MM，左闭右开）。 |
| **Provider 时段模板** | 定义在 `/providers` 上的 `time_zone` 和 `tiers`，同一 provider 下所有模型共享。 |
| **Model tier 价格** | 定义在 `/model-prices` 上的 `tier_prices`，描述某个模型在某个 tier 下的价格。 |

### 配置归属与下发链路

```mermaid
flowchart TD
    A[/providers<br/>time_zone + tiers/] -->|llm_config.provider| B[/cluster/]
    C[/model-prices<br/>prices + tier_prices/] -->|同 provider| B
    B --> D[AI Gateway API<br/>导出 AIConf.ModelTable]
    D -->|ai_rate_limit.data / AIConf| E[BFE 数据面]
    E --> F[ActiveTierName]
    F -->|命中 tier| G[按 tier 价格计费]
    F -->|未命中 tier| H[按默认 prices 计费]
    G --> I[Redis 定点数扣减]
    H --> I
```

多个 cluster 引用同一个 provider 时，会各自得到一份相同的 `ModelTable` 数据；provider 的时段规则变更后，所有引用它的 cluster 在下一次配置导出时自动生效。

### BFE ModelTable 结构

导出后的 `AIConf.ModelTable` 结构如下（定义于 BFE 侧相关配置加载代码）：

```go
type ModelTable struct {
    Currency string      // 固定 "RMB"
    TimeZone string      // 默认 "Asia/Shanghai"
    Tiers    []PriceTier // 时段定义
    Models   []ModelPrice
}

type PriceTier struct {
    Name       string
    TimeRanges []TimeRange
}

type ModelPrice struct {
    Provider   string
    Model      string
    Prices     PriceMap     // 默认价格
    TierPrices TierPriceMap // tier name -> 价格表
}
```

### 运行时时段匹配

BFE 加载阶段解析 `TimeZone` 并校验 `Tiers`，运行时使用 `ActiveTierName` 根据请求发生时刻匹配 tier：

```go
func (table *ModelTable) ActiveTierName(now time.Time) string {
    if table == nil || len(table.Tiers) == 0 {
        return ""
    }
    t := now.In(table.tz)
    wd := int(t.Weekday())
    hour, min := t.Hour(), t.Minute()
    cur := hour*60 + min

    for i := range table.Tiers {
        tier := &table.Tiers[i]
        for _, tr := range tier.TimeRanges {
            if len(tr.Weekdays) > 0 && !containsInt(tr.Weekdays, wd) {
                continue
            }
            start := parseHHMM(tr.Start)
            end := parseHHMM(tr.End)
            if start <= cur && cur < end {
                return tier.Name
            }
        }
    }
    return ""
}
```

命中 tier 且该 tier 配置了对应价格键时，使用 tier 价格；否则 fallback 到默认 `Prices`。时间区间左闭右开，跨午夜需拆成两段。

### 向后兼容

- `/providers` 不填 `time_zone` / `tiers` 时，`ModelTable.TimeZone` / `Tiers` 为空，行为与固定价格一致；
- `/model-prices` 不填 `tier_prices` 时，按默认 `Prices` 计费；
- 命中 tier 但该 tier 未配置某个价格键时，自动 fallback 到默认 `Prices` 中的对应键；
- `TokenUsage.UsedCost`、Lua 扣减逻辑、Redis 定点数存储均不需要修改。

---

## 批量任务的后付费配额：预留-结算-释放

批量与异步任务（OpenAI Batch API，`/v1/files` + `/v1/batches`）的计费与普通请求有本质区别：batch 创建时最终 Token 用量未知，结果文件由客户在异步任务完成后按需下载，费用不能在请求结束时立即结算。为此系统引入"预留-结算-释放"三态对账模型：

1. **预留（reserve）**：`POST /v1/batches` 创建时做配额预检并预留额度（数据面 `mod_ai_batch` 以 Redis `SETNX` 先到先得的方式做 batch 级 key 亲和，同一 batch 的后续操作路由到同一数据面实例）；
2. **结算（settle）**：结果文件下载时按 batch 价、按模型结算实际费用；
3. **释放（release）**：batch 被取消或进入终态时释放预留额度。

幂等由 `BATCH_RESERVE_BATCH:<batch_id>`（预留簿记）与 `BATCH_SETTLED`（结算去重 SETNX 锁）两个 Redis 标记保证：重复结算、重复释放、404 后重试等路径都收敛到同一终态。结算金额按基价计算：`settle_units = usage_in × input_cost_per_token + usage_out × output_cost_per_token`（batch 价与折扣的定义见[第十三章 模型定价与成本核算设计](./chapter13-model-pricing.md)；tier 时段价一期未接入批量结算）。

### 控制面数据与对账

控制面以两张表承载批量任务数据：

- `batch_files`：上传的文件实体（含方向、行数、字节数等元数据）；
- `batch_tasks`：batch 任务实体，承载 OpenAI batch 状态机（`validating` / `in_progress` / `completed` / `failed` / `cancelled` 等）以及预留-结算-释放三态对账状态。

Redis 键契约（`ai-gateway-api/model/ibatch/redis_keys.go`）：

| Redis Key | 说明 |
|-----------|------|
| `BATCH_FILE:<cluster>:<file_id>` | 文件实体（dir 等字段） |
| `BATCH_TASK:<batch_id>`（hash） | 任务实体与状态 |
| `BATCH_RESERVE_BATCH:<batch_id>` | 预留簿记，取消/终态时释放 |
| `BATCH_SETTLED:<batch_id>` | 结算去重标记（SETNX 锁） |

对账由控制面 batch reconcile job 执行：通过 MySQL named lock 在多个 AI Gateway API 实例间互斥协调，使用 Redis `SCAN` 遍历任务键，同步任务实体、推进状态、兜底结算（例如数据面实例宕机导致的结算残留）。该 job 默认关闭，在配置中设置 `[BatchJob] Enable=true` 开启（`ai-gateway-api/stateful/config.go` 的 `BatchJobConfig`）。

数据面 `mod_ai_batch` 模块的实现细节（上传行/字节计数、大文件流式转发、key 亲和、按 batch 价结算）见[第三十二章 Token认证与配额模块实现：mod_ai_token_auth](../implementation/chapter32-mod-ai-token-auth.md)；批量限流的 `batch_limits` 实现见[第三十三章 限流模块实现：mod_ai_rate_limit](../implementation/chapter33-mod-ai-rate-limit.md)。

---

## 配额与限流配置示例

### QuotaPlan 示例

以下是一个按月重置、总量为 1 亿 Token 的配额计划：

```json
{
  "quota_plan": {
    "unlimited": false,
    "pass_when_no_enough_quota": false,
    "quota": 100000000,
    "unit": "total_token",
    "reset_period": "monthly",
    "balance": {
      "used": 50000000,
      "remaining": 50000000
    }
  }
}
```

以下是一个按 RMB 计费、每月 5000 元预算的配额计划：

```json
{
  "quota_plan": {
    "unlimited": false,
    "pass_when_no_enough_quota": false,
    "quota": 5000.00,
    "unit": "RMB",
    "reset_period": "monthly",
    "balance": {
      "used": 1234.56,
      "remaining": 3765.44
    }
  }
}
```

### RateLimitPolicy 示例

以下是一个限制 gpt-4 模型 TPM/RPM 并设置最大并发为 50 的策略：

```json
{
  "rate_limit_policy": {
    "enabled": true,
    "rules": {
      "tpm": [
        {
          "name": "tpm_gpt4",
          "model": "gpt-4",
          "window_minutes": 1,
          "max_tokens": 100000,
          "step_minutes": 1
        },
        {
          "name": "tpm_default",
          "model": "*",
          "window_minutes": 1,
          "max_tokens": 500000,
          "step_minutes": 1
        }
      ],
      "rpm": [
        {
          "name": "rpm_gpt4",
          "model": "gpt-4",
          "window_minutes": 1,
          "max_requests": 1000,
          "burst": 1
        },
        {
          "name": "rpm_default",
          "model": "*",
          "window_minutes": 1,
          "max_requests": 5000,
          "burst": 1
        }
      ],
      "max_concurrency": 50,
      "batch_limits": {
        "max_create_rpm": 60,
        "max_active_batches": 10,
        "max_file_bytes": 104857600,
        "max_file_lines": 1000000
      }
    }
  }
}
```

### RMB 分时段定价示例

Provider 时段模板：

```json
{
  "name": "deepseek",
  "time_zone": "Asia/Shanghai",
  "tiers": [
    {
      "name": "peak",
      "time_ranges": [
        { "weekdays": [1, 2, 3, 4, 5], "start": "09:00", "end": "12:00" },
        { "weekdays": [1, 2, 3, 4, 5], "start": "14:00", "end": "18:00" }
      ]
    }
  ]
}
```

Model 价格：

```json
{
  "provider": "deepseek",
  "model": "deepseek-v4-pro",
  "mode": "chat",
  "prices": {
    "input_cost_per_token": 0.000001,
    "output_cost_per_token": 0.000002,
    "cache_read_input_token_cost": 0.0000005
  },
  "tier_prices": {
    "peak": {
      "input_cost_per_token": 0.000002,
      "output_cost_per_token": 0.000004,
      "cache_read_input_token_cost": 0.000001
    }
  }
}
```

---

## 本章小结

- `QuotaPlan` 支持 `total_token` 与 `RMB` 两种配额单位，分别适用于按 Token 计费和按成本计费场景。RMB 配额在 Redis 内部以 1e-8 元定点整数存储，对外统一按 4 位小数展示。
- Redis 是配额余额的唯一真实来源，OpenAPI 查询直接读取 Redis；手动重置/初始写入使用原子 `IncrBy(delta)`，周期重置在分布式锁保护下改用 Lua 原子 SET（`ResetToQuotaAtomic`），消除 read-modify-write 竞争。
- 周期重置支持 `weekly`（每周一）和 `monthly`（每月 1 日），由 `QuotaResetScheduler` 每分钟触发，基于 `last_reset_at` 判断周期边界；手动重置不更新 `last_reset_at`，避免干扰周期调度；`last_reset_at` 采用条件更新作为同周期重复重置的幂等兜底。
- 多实例部署下，周期重置由 Redis 分布式锁（key `quota:reset:scheduler:lock`，TTL 5 分钟，按 TTL/3 看门狗续期，instance token 校验释放）保证仅一个实例执行；`POST /inner-api/v1/quota/trigger-reset` 可手动触发一次同等保护的周期重置。
- `quota=0` 表示无余额计划：控制面正常计算并同步余额，数据面将其视为合法配置，命中请求返回 429 QuotaExhausted，Redis 余额 key 缺失视为耗尽而非 500。
- `RateLimitPolicy` 提供 TPM、RPM、并发数三类限制，规则支持具体模型名或 `*` 默认匹配；导出时按 Entity 层级向上合并，生成 `rate_limit_policies.json` 与 `api_key_rl_policy_bindings.json`。
- 控制面为每条 TPM/RPM 规则按规则名生成 Redis Key（形如 `default_bfe_rlp-<id>_RL_TPM_rlp-<id>_<name>`）：重命名规则等价于删除旧规则并新增规则（计数器清零），修改 `model` 等其它字段不会重置计数器。
- `rate_limit_policies` 含可空的 `batch_limits` 段（create RPM、活跃 batch 数、文件字节/行数硬顶），省略即不参与批量限流；导出后生成 BFE `ai_rate_limit.data` 的 `rules.batch` 段，文件硬顶与 `mod_ai_batch.data` 全局硬顶合并取最小值，未绑定策略的 API-Key 同样受全局硬顶约束。
- 批量任务采用后付费配额的"预留-结算-释放"三态对账：create 时预检预留、结果文件下载时按 batch 价结算、取消/终态释放；控制面以 `batch_files` / `batch_tasks` 表与 `BATCH_FILE` / `BATCH_TASK` / `BATCH_RESERVE_BATCH` / `BATCH_SETTLED` Redis 键契约承载，对账 job 由 `[BatchJob] Enable=true` 开启，多实例间以 MySQL named lock 协调。
- RMB 配额支持分时段定价：Provider 维护 `time_zone` 与 `tiers`，Model 维护 `tier_prices`，导出后由 BFE 根据请求时刻匹配 tier 并选择对应价格，未命中时 fallback 到默认价格。

---

## 参考文档

- `ai-gateway-api/design-docs/sys-design/details/配额余额同步机制.md`
- `ai-gateway-api/design-docs/sys-design/details/限流策略与导出.md`
- `ai-gateway-api/design-docs/sys-design/details/RMB配额分时段定价.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/api-keys.md`
- `bfe/docs/zh_cn/sys_design/ai_rate_limit_redis_key.md`
- `ai-gateway-api/model/quotacache/quotacache.go`
- `ai-gateway-api/model/quotacache/redis.go`
- `ai-gateway-api/model/quotacache/lock.go`
- `ai-gateway-api/model/quota/balance_sync.go`
- `ai-gateway-api/model/quota/scheduler.go`
- `ai-gateway-api/endpoints/innerapi_v1/quota_reset/quota_reset.go`
- `ai-gateway-api/model/quota/quota_plan.go`
- `bfe/bfe_modules/mod_ai_token_auth/token_rule_load.go`
- `bfe/bfe_modules/mod_ai_token_auth/token.go`
- `ai-gateway-api/model/rate_limit_policy/rate_limit_policy.go`
- `ai-gateway-api/model/rate_limit_policy/rate_limit_policy_manager.go`
- `bfe/bfe_modules/mod_ai_rate_limit/data_load.go`
- `bfe/bfe_modules/mod_ai_rate_limit/policy_limiter.go`
- `bfe/bfe_modules/mod_ai_rate_limit/batch_check.go`
- `ai-gateway-api/model/ibatch/redis_keys.go`
- `ai-gateway-api/stateful/config.go`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/batches.md`
