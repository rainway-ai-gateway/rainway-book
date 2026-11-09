# 附3 常见错误码

本书涉及两类错误码：BFE 数据面在 AI 网关场景下返回的 OpenAI 兼容错误码，以及 AI Gateway API 控制面 OpenAPI 返回的 `ErrNum` 业务错误码。两者分别由 BFE 官方文档与控制面接口定义文档维护，本书不重复罗列，仅说明两类错误码在 AI 网关场景下的关键语义。

## BFE 数据面错误码（v1.9.0）

- [BFE AI 网关错误码说明](https://github.com/bfenetworks/bfe/blob/refs/heads/v1.9.0-dev/docs/zh_cn/sys_design/ai_error_codes.md)

该文档覆盖以下内容：

- 认证与准入层错误码（如 `NO_API_KEY`、`INVALID_API_KEY`、`KEY_DISABLED`、`SUBNET_NOT_ALLOWED`、`MODEL_NOT_ALLOWED`）
- 限流检查层错误码（如 `RPM_LIMIT_EXCEEDED`、`TPM_LIMIT_EXCEEDED`、`CONCURRENCY_LIMIT_EXCEEDED`）
- 配额扣减层错误码（如 `QUOTA_EXHAUSTED`、`QUOTA_EXPIRED`、`INTERNAL_QUOTA_ERROR`）
- 转发与协议适配层错误码（如 `PROVIDER_PROTOCOL_MISMATCH`）
- 标准错误响应体结构与字段说明
- 预留错误码列表
- 错误码与访问日志字段的关联
- 按 HTTP 状态码的排查建议

429 `QUOTA_EXHAUSTED` 覆盖两类耗尽场景：Quota Plan 的 `total_token` 配额允许配置为 0，表示"无余额计划"，绑定该计划的请求将直接被拒绝并返回 429 `QUOTA_EXHAUSTED`（参见 `bfe_modules/mod_ai_token_auth/token_rule_load.go` 的 `quotaPlanCheck()`）；同时 Redis 余额 key 不存在（例如配额为 0 从未同步到 Redis）也视为余额耗尽而非内部错误（参见 `bfe_modules/mod_ai_token_auth/token.go` 的 `HasBalance()`）。

### 网关侧关键错误码

下表列出 AI 网关场景中由网关注入的错误码及其边界。这些响应由网关自行生成，均带 `X-Bfe-Gw-Error: 1`，并恒写 `X-Bfe-Log-Id`；客户端携带 `X-Request-ID` 时原样回显，未携带则不生成。

| HTTP | code | type | 触发场景与要点 |
|------|------|------|----------------|
| 429 | `NO_AVAILABLE_KEY` | `rate_limit_error` | 集群内没有可用上游 API Key：所有 `weight > 0` 的 Key 都处于惩罚期（`Weight=0` 的 Key 从不参与候选），惩罚过滤后候选集为空（message `no available upstream api key for this cluster`）。响应带 `Retry-After` 与 `error.details.retry_after_seconds`，取值等于 429 惩罚 TTL 60s；候选集为空即熔断并合成该 429，TTL 到期自动恢复。两档惩罚时长均可配置（`penalty_ttl_429` / `penalty_ttl_auth`，默认 60s / 300s，见 [第十章 Provider与Cluster设计](../design/chapter10-provider-and-cluster.md)） |
| 500 | `CONTENT_REJECTED` | `invalid_request_error` | 请求阶段内容审查判定拒绝（message `request rejected by content audit`）。网关不重试、不惩罚会话，也不将其归类为 502 |
| 500 | `BK_NO_BACKEND` | `internal_error` | 目标子集群的有效实例池为空（message `no available backend for this request`）；`cluster_table` 中缺失的子集群同样按空 backend 处理。该错误反映配置或实例导出问题，应触发告警 |
| 502 | `BACKEND_UNAVAILABLE` | `internal_error` | 目标集群不存在或后端连接失败（message `upstream backend unavailable for this request`）。与 `BK_NO_BACKEND` 的区别在于：后者是集群存在但没有可用实例，前者是集群不可解析或连接不通 |
| 503 | `PRICE_NOT_CONFIGURED` | `internal_error` | 集群配置 `price_miss_action=reject` 且模型/模式无价格条目（或 `ModelTable` 整体缺失）时**转发前拒绝**：不调用上游、不计费，访问日志照常入库并带缺价信号（详见 [第十三章 模型定价与成本核算设计](../design/chapter13-model-pricing.md)「缺价处置三态」） |
| 429 | `EPP_OVERLOAD` | `rate_limit_error` | EPP 经 ext_proc `ImmediateResponse` 显式拒绝（背压过载），透传上游 `Retry-After`；**不重试、不回退本地均衡、不喂熔断** |
| 503 | `EPP_NO_CAPACITY` | `internal_error` | EPP 准入拒绝（无端点容量），同上过载保护语义 |

## AI Gateway API 控制面错误码

控制面 OpenAPI 统一返回 `{ErrNum, Data, ErrMsg}` 结构，通用约定见 [00-common.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/develop/design-docs/api-define/OpenAPI接口定义/00-common.md)。`ErrNum` 取值约定如下：

| ErrNum | 含义 |
|--------|------|
| 200 | 调用成功 |
| 401 | 鉴权失败 |
| 402 | 没有调用权限 |
| 403 | 访问被禁止（管理面 IP 白名单拒绝），响应体为 `{"ErrNum":403,"Type":"Access Forbidden"}` |
| 404 | 查询/修改/删除的对象不存在 |
| 405 | 方法不允许，响应带 `Allow` 头（逗号+空格分隔该路径支持的方法）；404/200 不加该头 |
| 409 | 资源依赖冲突 |
| 422 | 参数不合法（含表单/URI 参数类型转换错误，如 `stream=not-bool`、`page=abc`，返回 `Param Illegal: invalid <字段名>: <值>`） |
| 500 | 其他业务逻辑错误（未知错误的兜底返回码） |
| 555 | 创建重复对象 |
| 556 | 数据重复 |

其中 409 Conflict 专用于资源依赖冲突场景，控制面 500 保留为未知错误的兜底返回码（数据面的 500 另见上文 `BK_NO_BACKEND` 与 `CONTENT_REJECTED`）。返回 409 的典型场景包括：

- 删除被 AI 集群调度引用的 BFE 集群（`endpoints/openapi_v1/bfe_cluster/delete.go`）；
- 删除被 BFECluster 或 SubCluster 引用的实例池（`model/icluster_conf/pool.go`）；
- 删除被 Product 引用的证书（`model/iprotocol/certificate.go`）；
- 删除被路由规则引用的产品集群，或集群被路由规则引用时更新其模型（`model/iroute_conf/route_rule.go`、`model/route_rules/route_rules.go`，规则 target/fallback 引用检查）；
- 删除仍有子节点的 Entity（`model/entity/entity_manager.go`）；
- 取消已完成或已进入终态的批量任务（`POST /open-api/v1/batches/{batch_id}/cancel` 时任务已处于 `completed`/`expired`/`failed`/`cancelled`，返回 `BATCH_TERMINAL_OR_CONFLICT`；任务处于 `cancelling` 时返回 200 幂等重放，不属于 409）；
- `/ai-cache-rules`、`/ai-context-rules`、`/traffic-mirror-rules` 三个集合资源整组 `PUT` 并发冲突（锁重试耗尽），`ErrMsg` 固定为 `Conflict: collection is being replaced concurrently, retry later`，集合保持原状；
- Provider 收窄 `model_protocols` 且仍被 Cluster 的 `ensure_reasoning_content` 引用（`ProviderProtocolRefChecker`），消息含 provider/协议/引用它的 Cluster。

排查建议：收到 409 时应先根据 `ErrMsg` 中给出的引用方名称解除依赖关系（如调整路由规则、更换证书绑定、迁移子节点）再重试操作；收到 500 且 `ErrMsg` 不含具体依赖信息时，按未知错误收集日志并联系维护方定位；收到 403 且响应体为 `Access Forbidden` 时，说明请求来源 IP 被管理面 IP 白名单拒绝，需调整白名单配置或从允许的网段访问。

## 本书相关章节

- [第十六章 安全设计](../design/chapter17-security-design.md) 中介绍了错误响应体与安全审计日志字段。
- [第十九章 控制台基础操作](../operation/chapter19-dashboard-basics.md) 中介绍了常见控制台错误提示的对照思路。
- [第二十二章 API-Key 与配额配置](../operation/chapter22-apikey-and-quota-config.md) 中介绍了配额耗尽、限流触发等问题的排查方法。
