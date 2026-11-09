# 附1 OpenAPI 接口速查

壬远 AI 网关控制面（AI Gateway API）的 OpenAPI v1 接口定义已按模块拆分为独立文档，并在 GitHub 上持续维护。本书不再重复罗列完整接口列表，读者可通过以下链接直接查看对应版本的权威定义。

## v0.1.0 版本接口文档

- [OpenAPI 接口定义目录](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义)
- [通用约定](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/00-common.md)：URL 格式、鉴权方式、返回值格式、Method 约定、通用 Query 参数
- [关键业务流程](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/workflows.md)
- [对象关系图](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/object-relations.md)

## 各模块接口入口

| 模块 | 文档链接 |
|------|----------|
| `/api-keys` | [api-keys.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/api-keys.md) |
| `/entity-types` | [entity-types.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/entity-types.md) |
| `/entities` | [entities.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/entities.md) |
| `/global-route-rules` | [global-route-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/global-route-rules.md) |
| `/route-tables` | [route-tables.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/route-tables.md) |
| `/providers` | [providers.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/providers.md)（含 `instance_source` / `k8s_pool_name` 实例来源字段） |
| `/clusters` | [clusters.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/clusters.md) |
| `/epp-assignments` | [epp-assignments.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/epp-assignments.md) |
| `/epp-pool` | [epp-pool.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/epp-pool.md) |
| `/model-prices` | [model-prices.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/model-prices.md) |
| `/certificates` | [certificates.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/certificates.md) |
| `/auth` | [auth.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/auth.md)（含用户 `role` 角色模型与 `PATCH /auth/users/{user_name}/role`） |
| `/batches`、`/batch-files` | [batches.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/batches.md)（批量任务列表 / 详情 / 取消与文件元数据查询） |
| `/security` | [security.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/security.md)（密钥轮换收敛任务：触发、列表、详情） |
| `/ai-cache-rules` | [ai-cache-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/ai-cache-rules.md)（AI 缓存规则集合，全量替换） |
| `/ai-cache-semantic-settings` | [ai-cache-semantic-settings.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/ai-cache-semantic-settings.md)（语义缓存全局设置，单例） |
| `/ai-context-rules` | [ai-context-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/ai-context-rules.md)（上下文压缩规则集合，全量替换） |
| `/ai-context-settings` | [ai-context-settings.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/ai-context-settings.md)（上下文压缩全局设置，单例） |
| `/traffic-mirror-rules` | [traffic-mirror-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/traffic-mirror-rules.md)（流量镜像规则集合，全量替换） |
| `/intent-config` | [intent-config.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/intent-config.md)（意图配置，单例） |
| `/expression/verify` | [expression-verify.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/expression-verify.md) |
| `/operation-logs` | [operation-logs.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/operation-logs.md) |
| `/report/overview`、`/report/timeseries`、`/report/rankings`、`/report/distribution`、`/report/logs` | [report.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/report.md)（支持缓存/镜像/意图维度查询；`[Report]` 未配置时端点 404） |
| `/k8s_pools` | [k8s-pools.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/k8s-pools.md)（只读：`GET` 列表/详情，不开放写接口、不分页） |

## 端点要点

以下端点为管理面 OpenAPI 的一部分（权威定义见上表的 [batches.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/batches.md)、[security.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/security.md)、[auth.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/auth.md)）：

| 端点 | 方法 | 说明 |
|------|------|------|
| `/open-api/v1/auth/users/{user_name}/role` | PATCH | 用户角色变更（唯一入口），Body 为 `{"role": "system_admin" \| "system_admin_readonly"}`；内置 `admin` 不可降级（422） |
| `/open-api/v1/security/reencrypt-sweeps` | GET | 密钥轮换收敛任务历史：分页参数 `page`/`page_size`/`sort_by`（`started_at`/`task_id`）/`sort_order`；支持按 `status`/`mode`/`scope`/`dry_run`/`start_time`/`end_time` 过滤；任务对象含 `created_by`；`running` 任务的 `summary` 为空对象、`duration_ms=0`；dry-run 成功的任务 `error="dry_run"` |
| `/open-api/v1/batches` | GET | 批量任务列表，主键游标分页：`cursor` 与 `limit` 均为整型（`limit` 默认 50，取值 1–200），响应中的 `next_cursor` 为整数，`0` 表示无更多数据 |
| `/open-api/v1/batches/{batch_id}` | GET | 批量任务详情，`Data` 顶层即任务字段，附 `batch_files` 数组（`file_id`/`provider`/`purpose`/`filename`/`bytes`/`created_at`；`filename` 缺失时省略该键） |
| `/open-api/v1/batches/{batch_id}/cancel` | POST | 取消批量任务：数据库条件更新抢占取消资格；任务已处于 `cancelling` 时返回 200 幂等重放，任务已进入终态（`completed`/`expired`/`failed`/`cancelled`）时返回 409 `BATCH_TERMINAL_OR_CONFLICT` |
| `/open-api/v1/batch-files/{file_id}` | GET | 批量任务文件元数据查询 |

若干行为约定：

- `/open-api/v1/auth/users/{user_name}/is_admin` 路径不存在（返回 404），角色变更只经上述 `role` 端点；
- 表单/URI 参数类型转换错误（如 `stream=not-bool`、`page=abc`）返回 422 `Param Illegal: invalid <字段名>: <值>`，不回显解析错误原文；
- `/open-api/v1/batches/{batch_id}/cancel` 对同一任务的重复取消是安全的：命中 `cancelling` 的请求返回 200，并重新向 Provider 发送取消；取消意图一旦写入不回退；
- `/open-api/v1/operation-logs` 使用页码分页，响应为嵌套结构 `{list, pagination: {page, page_size, total}}`；
- 列表默认分页：`/api-keys` 无参按 `page=1` / `page_size=20` 返回（`total` 按过滤条件统计）；`/entities`、`/entity-types` 为“无参返回全量”的登记例外；405 响应带 `Allow` 头，CORS 预检返回 204 + `Access-Control-Allow-*`；
- `/clusters` 的 `llm_config` 含 `price_miss_action` / `ensure_reasoning_content` / `key_affinity.penalty_ttl_*`；`/providers` 响应不含 `k8s_instance_pool`（请求携带按未知字段静默忽略）、池名上限 128；provider 收窄 `model_protocols` 仍被 cluster 引用时返回 409（消息含引用方）；`/report/logs` 明细含 `ai_price_miss` / `ai_price_warn` 字段并支持 `price_warn` 精确过滤。

## 说明

- 上述链接指向 `v0.1.0-dev` 分支快照，适合与本书内容对照阅读。
- 若需查看最新接口变更，请访问 [ai-gateway-api 主分支设计文档](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/develop/design-docs/api-define/OpenAPI接口定义)。
