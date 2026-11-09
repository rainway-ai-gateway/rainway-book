# Appendix 1: OpenAPI Quick Reference

The OpenAPI v1 interface definitions of the Rainway AI Gateway Control Plane (AI Gateway API) have been split into independent documents by module and are continuously maintained on GitHub. This book does not repeat the full interface list; readers can view the authoritative definitions for the corresponding version via the links below.

## v0.0.11 API Documentation

- [OpenAPI Interface Definitions Directory](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义)
- [Common Conventions](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/00-common.md): URL format, authentication method, response format, Method conventions, common Query parameters
- [Key Business Workflows](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/workflows.md)
- [Object Relations Diagram](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/object-relations.md)

## API Entry Points by Module

| Module | Documentation Link |
|------|----------|
| `/api-keys` | [api-keys.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/api-keys.md) |
| `/entity-types` | [entity-types.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/entity-types.md) |
| `/entities` | [entities.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/entities.md) |
| `/global-route-rules` | [global-route-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/global-route-rules.md) |
| `/route-tables` | [route-tables.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/route-tables.md) |
| `/providers` | [providers.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/providers.md) (includes the `instance_source` / `k8s_pool_name` instance-source fields) |
| `/clusters` | [clusters.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/clusters.md) |
| `/epp-assignments` | [epp-assignments.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/epp-assignments.md) |
| `/epp-pool` | [epp-pool.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/epp-pool.md) |
| `/model-prices` | [model-prices.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/model-prices.md) |
| `/certificates` | [certificates.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/certificates.md) |
| `/auth` | [auth.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/auth.md) (includes the user `role` model and `PATCH /auth/users/{user_name}/role`) |
| `/batches`, `/batch-files` | [batches.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/batches.md) (batch task list / detail / cancel and file metadata) |
| `/security` | [security.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/security.md) (key-rotation convergence tasks: trigger, list, detail) |
| `/ai-cache-rules` | [ai-cache-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/ai-cache-rules.md) (AI cache rule collection, full replacement) |
| `/ai-cache-semantic-settings` | [ai-cache-semantic-settings.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/ai-cache-semantic-settings.md) (semantic cache global settings, singleton) |
| `/ai-context-rules` | [ai-context-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/ai-context-rules.md) (context compression rule collection, full replacement) |
| `/ai-context-settings` | [ai-context-settings.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/ai-context-settings.md) (context compression global settings, singleton) |
| `/traffic-mirror-rules` | [traffic-mirror-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/traffic-mirror-rules.md) (traffic mirror rule collection, full replacement) |
| `/intent-config` | [intent-config.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/intent-config.md) (intent configuration, singleton) |
| `/expression/verify` | [expression-verify.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/expression-verify.md) |
| `/operation-logs` | [operation-logs.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/operation-logs.md) |
| `/report/overview`, `/report/timeseries`, `/report/rankings`, `/report/distribution`, `/report/logs` | [report.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/report.md) (supports cache/mirror/intent dimension queries; endpoints return 404 when `[Report]` is not configured) |
| `/k8s_pools` | [k8s-pools.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/k8s-pools.md) (read-only: `GET` list/detail; no write endpoints and no pagination) |

## Endpoint Essentials

The following endpoints are part of the Management Plane OpenAPI (authoritative definitions in [batches.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/batches.md), [security.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/security.md) and [auth.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.1.0-dev/design-docs/api-define/OpenAPI接口定义/auth.md)):

| Endpoint | Method | Description |
|------|------|------|
| `/open-api/v1/auth/users/{user_name}/role` | PATCH | User role change (the only entry point); body is `{"role": "system_admin" \| "system_admin_readonly"}`; the built-in `admin` cannot be downgraded (422) |
| `/open-api/v1/security/reencrypt-sweeps` | GET | History of key-rotation convergence tasks: pagination via `page`/`page_size`/`sort_by` (`started_at`/`task_id`)/`sort_order`; filtering by `status`/`mode`/`scope`/`dry_run`/`start_time`/`end_time`; task objects include `created_by`; a `running` task has an empty object as `summary` and `duration_ms=0`; a successful dry-run task has `error="dry_run"` |
| `/open-api/v1/batches` | GET | Batch task list with primary-key cursor pagination: `cursor` and `limit` are integers (`limit` defaults to 50, range 1–200); `next_cursor` in the response is an integer, where `0` means there is no more data |
| `/open-api/v1/batches/{batch_id}` | GET | Batch task detail; `Data` top level is the task fields plus a `batch_files` array (`file_id`/`provider`/`purpose`/`filename`/`bytes`/`created_at`; the `filename` key is omitted when absent) |
| `/open-api/v1/batches/{batch_id}/cancel` | POST | Cancel a batch task: a database conditional update claims the cancellation. A task already in `cancelling` returns 200 as an idempotent replay; a task in a terminal state (`completed`/`expired`/`failed`/`cancelled`) returns 409 `BATCH_TERMINAL_OR_CONFLICT` |
| `/open-api/v1/batch-files/{file_id}` | GET | Batch task file metadata query |

Several behavioral conventions:

- The `/open-api/v1/auth/users/{user_name}/is_admin` path does not exist (returns 404); role changes go only through the `role` endpoint above;
- Form/URI parameter type-conversion errors (e.g. `stream=not-bool`, `page=abc`) return 422 `Param Illegal: invalid <field>: <value>`, without echoing the raw parse error;
- Repeated cancellation of the same task via `/open-api/v1/batches/{batch_id}/cancel` is safe: a request that hits `cancelling` returns 200 and re-sends the cancellation to the Provider; a recorded cancellation intent is never rolled back;
- `/open-api/v1/operation-logs` uses page-number pagination and returns the nested structure `{list, pagination: {page, page_size, total}}`.
- List default pagination: `/api-keys` without parameters returns `page=1` / `page_size=20` (with `total` counted under the filter conditions); `/entities` and `/entity-types` are the registered exception of "no parameters returns the full list"; 405 responses carry an `Allow` header, and CORS preflight returns 204 + `Access-Control-Allow-*`;
- `/clusters` `llm_config` includes `price_miss_action` / `ensure_reasoning_content` / `key_affinity.penalty_ttl_*`; `/providers` responses have no `k8s_instance_pool` field (carrying it in a request body is silently ignored as an unknown field) and the pool-name limit is 128; narrowing a provider's `model_protocols` while still referenced by a cluster returns 409 (the message names the referencing clusters); `/report/logs` detail rows include `ai_price_miss` / `ai_price_warn` and support an exact `price_warn` filter.

## Notes

- The links above point to a snapshot of the `v0.1.0-dev` branch, suitable for cross-referencing with the content of this book.
- To view the latest API changes, visit the [ai-gateway-api main branch design docs](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/develop/design-docs/api-define/OpenAPI接口定义).
