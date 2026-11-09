# Chapter 6: Control Plane Core Design: AI Gateway API

## Chapter Goals

Through this chapter, readers will understand:

- The role and responsibility boundaries of the AI Gateway API within the Rainway AI Gateway;
- How the Control Plane organizes code in a three-layer architecture (interface layer, model layer, storage layer);
- The division of responsibilities and route organization between the management-plane OpenAPI and the data-plane InnerAPI;
- How the unified `xreq.Endpoint` abstraction simplifies interface registration, authorization, and middleware handling;
- The implementation of the global container (`stateful/container`) and manual dependency injection;
- The two resource forms of rule-type resources (full-replacement collections and singletons) and their authorization features;
- The write mechanism, masking rules, and audit value of the operation log module;
- The 409 Conflict error convention for resource dependency conflicts;
- The complete flow from `main.go` to HTTP service startup.

---

## Positioning and Responsibilities of the AI Gateway API

The **AI Gateway API** is the core component of the Control Plane of the Rainway AI Gateway. It is responsible for exposing the management-plane interfaces and data-plane export interfaces, and for the creation, storage, version control, and distribution of policies/configurations. It provides programmable management interfaces upward for the Dashboard and administrator scripts, and generates consumable configuration snapshots downward for the BFE Data Plane and Conf Agent.

The relationship between the AI Gateway API and its surrounding components in the overall architecture of the Rainway AI Gateway is as follows:

| Component | Role | Interaction with AI Gateway API |
|------|------|--------------------------|
| Dashboard | Management console | Calls the OpenAPI to perform visual operations |
| BFE | Data Plane | Consumes configurations exported via InnerAPI and performs forwarding |
| Conf Agent | Configuration agent | Polls InnerAPI, pulls the latest configuration, and triggers BFE hot reload |
| ai-gateway-controller (formerly Service Controller) | Service discovery | Reports the Ready instances of K8s Services/Endpoints as a K8s pool (InnerAPI `k8s_pools`); the Control Plane resolves the effective pool on demand by `k8s_pool_name`, and finally exports it to BFE through `cluster_table` |

The current functional scope of the AI Gateway API covers: API-Key / Entity / Entity-Type management, Provider and Cluster management, model pricing management, QuotaPlan and RateLimitPolicy management, AI routing rule management, AI cache rules, traffic mirror rules, intent configuration, context compression rules, K8s Pool management, certificate and extra file management, authentication and authorization, configuration operation log auditing, and configuration export for the Data Plane.

### Boundary Between Control Plane and Data Plane

The Control Plane is responsible for "decision-making", while the Data Plane is responsible for "execution". Their division of responsibilities can be summarized as follows:

| Responsibility | Control Plane (AI Gateway API) | Data Plane (BFE) |
|------|--------------------------|---------------|
| Configuration creation and modification | ✅ | ❌ |
| Configuration persistence | ✅ (MySQL / SQLite) | ❌ |
| Configuration version control | ✅ | ❌ |
| Configuration export | ✅ (InnerAPI) | ❌ |
| Request forwarding | ❌ | ✅ |
| API-Key validation and quota deduction | ❌ | ✅ |
| Rate limit enforcement | ❌ | ✅ |

This division allows the Control Plane to be upgraded and restarted independently, while the Data Plane continues to forward traffic based on locally cached configurations, ensuring forwarding stability.

---

## Three-Layer Architecture

The AI Gateway API adopts a classic three-layer architecture that decouples HTTP handling, business logic, and data persistence:

```
┌─────────────────────────────────────────────────────────────┐
│                      Interface Layer (endpoints)            │
│   OpenAPI v1 (/open-api/v1)  ·  InnerAPI v1 (/inner-api/v1) │
├─────────────────────────────────────────────────────────────┤
│                      Model Layer (model)                    │
│   Business Logic · Transactions · Validation · Cascading    │
├─────────────────────────────────────────────────────────────┤
│                    Storage Layer (storage/rdb)              │
│        DAO · Storage Implementations · Relational DB R/W    │
└─────────────────────────────────────────────────────────────┘
```

### Interface Layer

The interface layer resides in `endpoints/` and is the entry point for HTTP requests. Its main responsibilities include:

- Route registration and dispatch;
- Request parameter binding and basic validation;
- Permission and identity authentication;
- Invoking the model layer Manager to complete business processing;
- Unified response rendering.

Key packages are as follows:

| Package | Responsibility |
|----|------|
| `endpoints/router.go` | Root route registration, mounting global Recovery, Logger, and CORS middleware |
| `endpoints/openapi_v1/` | Management-plane OpenAPI route organization, where business sub-packages are merged |
| `endpoints/innerapi_v1/` | Data-plane InnerAPI export interface registration |
| `endpoints/middleware/` | Recovery, Logger, CORS, Product Probe, User Probe |
| `lib/xreq` | Unified `Endpoint` abstraction and parameter binding utilities |

### Model Layer

The model layer resides in `model/` and adopts a layered pattern of **Manager + Storager interfaces**:

- **Manager**: encapsulates business logic, transaction orchestration, parameter validation, and cascade operations;
- **Storager interface**: defines the persistence operation contract, implemented by the storage layer;
- **Param / Filter**: used for write inputs and query conditions respectively; fields are mostly pointer types to distinguish "not provided" from "zero value".

Key packages are as follows:

| Package | Responsibility |
|----|------|
| `model/api_key/` | API-Key business logic, quota/rate-limit/route cascading, real-time quota queries |
| `model/entity/` | Entity / Entity-Type business logic, hierarchy validation and cascade deletion |
| `model/icluster_conf/` | Cluster, SubCluster, Pool business logic and `AIConf` export |
| `model/iprovider/` | Provider access capability management, model discovery, referenced-by-Cluster checks |
| `model/imodel_price/` | Model pricing import, CRUD, and price queries |
| `model/quota/` | QuotaPlan, BalanceSync, QuotaResetScheduler |
| `model/rate_limit_policy/` | RateLimitPolicy business logic and export |
| `model/route_rules/` | Global / Entity / API-Key three-level AI routing rules |
| `model/ioperlog/` | Operation log Manager, sensitive-field masking, and change summary diff_keys computation |
| `model/imods/` | Export of module configurations such as mod-api-key, mod-body-process, and AI routing |
| `model/ai_cache/` | AI cache rules and semantic cache global settings: business logic and export |
| `model/ai_context/` | Context compression rules and global settings: business logic and export |
| `model/traffic_mirror/` | Traffic mirror rules: business logic and export |
| `model/iintent_config/` | Intent configuration singleton: business logic and export |
| `model/ik8s_pool/` | K8s Pool maintenance (no write fan-out; on-demand resolution at consumption) |
| `model/itxn/` | Transaction abstraction interface `TxnStorager` |
| `model/shared/` | Cross-package shared types and generic Storager interfaces |

### Storage Layer

The storage layer resides in `storage/rdb/` and adopts a two-level **DAO + Storage** structure:

- `storage/rdb/internal/dao/` provides `T<Table>One/List/Create/Update/Delete` functions per table, operating directly on MySQL / SQLite;
- Each sub-package under `storage/rdb/*` implements the `XxxStorager` interface defined by the model layer, converting DAO results to business models;
- `storage/rdb/txn/` provides the transaction implementation of `model/itxn.TxnStorager`.

Key packages are as follows:

| Package | Corresponding Model | Covered Tables |
|----|----------|--------|
| `storage/rdb/auth/` | `model/iauth` | `users`, `user_products` |
| `storage/rdb/basic/` | `model/ibasic` | `bfe_clusters`, `products`, `extra_files` |
| `storage/rdb/api_key/` | `model/api_key` | `api_keys`, `api_key_tokens` |
| `storage/rdb/cluster_conf/` | `model/icluster_conf` | `clusters`, `sub_clusters`, `pools`, `lb_matrices` |
| `storage/rdb/entity/` | `model/entity` | `entity_types`, `entities` |
| `storage/rdb/protocol/` | `model/iprotocol` | `certificates` |
| `storage/rdb/quota/` | `model/quota` | `quota_plans` |
| `storage/rdb/rate_limit_policy/` | `model/rate_limit_policy` | `rate_limit_policies` |
| `storage/rdb/route_conf/` | `model/iroute_conf` | `domains`, `route_*_rules` |
| `storage/rdb/route_rules/` | `model/shared`, `model/route_rules` | `route_rules` |
| `storage/rdb/ioperlog/` | `model/ioperlog` | `operation_logs` |
| `storage/rdb/provider/` | `model/iprovider` | `providers` |
| `storage/rdb/ai_cache/` | `model/ai_cache` | `ai_cache_rules`, `ai_cache_semantic_settings` |
| `storage/rdb/ai_context/` | `model/ai_context` | `ai_context_rules`, `ai_context_settings` |
| `storage/rdb/traffic_mirror/` | `model/traffic_mirror` | `traffic_mirror_rules` |
| `storage/rdb/iintent_config/` | `model/iintent_config` | `intent_config` |
| `storage/rdb/k8s_pool/` | `model/ik8s_pool` | `k8s_pools` |

### Inter-Layer Interaction

The flow of an HTTP request across the three layers is as follows:

```
                         HTTP Request
                              │
           ┌──────────────────┴──────────────────┐
           ▼                  ▼                  ▼
    /open-api/v1      /inner-api/v1        Global Middleware
           │                  │                  │
           └──────────────────┼──────────────────┘
                              ▼
                    endpoints/middleware
                    (Recovery / Logger / CORS /
                     Product Probe / User Probe)
                              │
                              ▼
                     endpoints/xreq.Endpoint
                              │
                              ▼
                       model/* Manager
                              │
                              ▼
                    model/itxn.TxnStorager
                              │
           ┌──────────────────┼──────────────────┐
           ▼                  ▼                  ▼
    storage/rdb/*      storage/rdb/*      storage/rdb/*
                              │
                              ▼
                    storage/rdb/internal/dao
                              │
                              ▼
                         MySQL / SQLite
```

Core constraints:

- The interface layer only calls model layer Managers and never accesses the storage layer directly;
- The model layer only depends on the `XxxStorager` interfaces defined in the same package or in `model/shared`, plus `itxn.TxnStorager`;
- The storage layer is only responsible for database reads and writes, obtaining transactions or regular connections from the context via `lib.DBContextFactory`.

### Design Principles and Conventions

The three-layer architecture follows these design principles:

1. **Layered decoupling**: the interface layer only depends on model layer Managers; the model layer only depends on Storager interfaces and the transaction abstraction; the storage layer is only responsible for database reads and writes.
2. **Interface-oriented**: the model layer defines persistence contracts through `XxxStorager` interfaces, making it easy to replace storage implementations in the future.
3. **Unified Endpoint abstraction**: all HTTP interfaces are described with `xreq.Endpoint`, unifying registration, authorization, and middleware handling.
4. **Sub-package autonomy**: each business sub-package independently maintains its own Endpoint, Action, Manager, and Storager; the top-level `endpoints.go` only merges them.
5. **Global container**: Managers and Storagers exist as global singletons in `stateful/container`, initialized once via `rdb.Init()`.
6. **Version control**: configuration changes are versioned uniformly by `iversion_control`, supporting incremental export via InnerAPI.
7. **No physical foreign keys**: inter-table relationships are maintained by the application layer; DDL does not declare FOREIGN KEY, facilitating database sharding and flexible schema changes.
8. **Transaction orchestration**: cross-table / cross-Storager business operations guarantee atomicity through `TxnStorager.AtomExecute`.

---

## Division of Responsibilities Between OpenAPI v1 and InnerAPI v1

The AI Gateway API divides its external interfaces into two interface families:

| Interface Family | Prefix | Purpose | Middleware | Callers |
|--------|------|------|--------|--------|
| **OpenAPI v1** | `/open-api/v1` | Management-plane interfaces | `McProductProbe` + `McUserProbe` | Dashboard, administrator scripts |
| **InnerAPI v1** | `/inner-api/v1` | Data-plane export interfaces | `McUserProbe` | BFE, Conf Agent |

### Main OpenAPI v1 Modules

OpenAPI v1 is responsible for exposing manageable resources. Typical modules include:

| Sub-package | Path | Function |
|------|------|------|
| `api_key` | `/api-keys` | API-Key management |
| `entity` | `/entities` | Entity management |
| `entity_type` | `/entity-types` | Entity-Type management |
| `provider` | `/providers` | Provider management, model discovery |
| `product_cluster` | `/clusters` | Cluster management |
| `model_price` | `/model-prices` | Model pricing management |
| `global_route_rules` | `/global-route-rules` | Global Route Table |
| `route_tables` | `/route-tables` | Route table list |
| `certificate` | `/certificates` | Certificate management |
| `auth` | `/auth`, `/meta` | Users, Session Key, Token |
| `operation_log` | `/operation-logs` | Configuration operation log queries |
| `ai_cache` | `/ai-cache-rules`, `/ai-cache-semantic-settings` | AI cache rules (full-replacement collection) and semantic cache settings (singleton) |
| `ai_context` | `/ai-context-rules`, `/ai-context-settings` | Context compression rules (full-replacement collection) and global settings (singleton) |
| `traffic_mirror` | `/traffic-mirror-rules` | Traffic mirror rules (full-replacement collection) |
| `intent_config` | `/intent-config` | Intent configuration (singleton) |

### Main InnerAPI v1 Export Interfaces

InnerAPI v1 exports the configurations persisted by the Control Plane by topic, for the Data Plane to consume:

| Path | Function |
|------|------|
| `/configs/tls_conf/server_data_conf` | Export TLS/Server/routing rule configurations |
| `/configs/gslb_data/gslb` | Export GSLB scheduling configuration |
| `/configs/gslb_data/cluster_table` | Export cluster table configuration |
| `/configs/protocol/server_cert_conf` | Export certificate configuration |
| `/configs/extra_files/{filename}` | Export extra files |
| `/configs/mod-api-key` | Export API-Key and quota configuration |
| `/configs/mod-body-process` | Export request body processing configuration |
| `/configs/rate-limit-policy` | Export rate limit policy configuration |
| `/configs/ai-route` | Export AI routing configuration |
| `/configs/ai-cache-rule` | Export AI cache rule configuration |
| `/configs/ai-context-rule` | Export context compression rule configuration |
| `/configs/traffic-mirror-rule` | Export traffic mirror rule configuration |
| `/configs/mod-ai-intent` | Export intent configuration |
| `/quota/trigger-reset` | Manually trigger a quota period reset (see the quota chapter) |

All InnerAPI export interfaces support the `version` query parameter and implement incremental synchronization via `model/iversion_control`: when the requested version matches the current version, `Data: nil` is returned to avoid redundant distribution.

### Middleware Execution Order

Global middleware is registered in order in `RegisterRouters` in `endpoints/router.go`:

```
router.Use(middleware.MCRecovery)
router.Use(middleware.MCLogger)
router.Use(middleware.MCCors)
```

On top of this, the OpenAPI route subtree additionally mounts `McProductProbe` and `McUserProbe`, while the InnerAPI route subtree mounts only `McUserProbe`. `McUserProbe` invokes the `Authorizer` configured on each Endpoint and decides whether to allow the request based on Feature + Action.

| Middleware | Function |
|--------|------|
| `MCRecovery` | Captures panics and returns a unified error response |
| `MCLogger` | Records request/response logs |
| `MCCors` | Handles CORS preflight and response headers |
| `McProductProbe` | Parses the product-line context from request headers |
| `McUserProbe` | Parses user identity from Session Key or Token and performs permission checks |

---

## Resource Forms: Full-Replacement Collections and Singletons

OpenAPI resources do not come in only one form of "id-addressed CRUD". Rule-type configurations for BFE modules have given rise to two new resource forms, corresponding to different interface surfaces and write semantics.

### Full-Replacement Collection Resources

`ai_cache_rules`, `traffic_mirror_rules`, and `ai_context_rules` are full-replacement collection resources:

- The interface exposes only collection-level `GET` / `PUT`, with no `/{id}` sub-resources;
- A single `PUT` performs a delete-all + insert-all full replacement within one transaction; `rules: null` is treated as clearing the collection;
- Rules are consumed with first-match-wins semantics: the array order is the priority, and an element's position in the array is part of the configuration semantics;
- The database auto-increment id of a collection element is not exposed externally; both the API and the export organize elements by array order.

### Singleton Resources

`intent_config`, `ai_cache_semantic_settings`, and `ai_context_settings` are singleton resources:

- The whole system has a single fixed row (fixed id); a write is a single-row overwrite, with no collection concept;
- The version number `yyyyMMddHHmmss` is generated internally by the Control Plane and embedded in the exported content; it is not exposed through the API;
- Empty-table semantics differ per resource: when `intent_config` has never been published, `GET` returns 404 (an empty question set is the soft switch semantics of disabling classification); for the two settings tables, an empty table means "defaults not overridden" — `GET` returns the documented default values and never returns 404.

Both forms still use the `xreq.Endpoint` registration and `iauth` authorization framework; only the write path inside the Manager changes from "incremental modification by id" to "full replacement / single-row overwrite". The authorization items correspond to `FeatureAICache`, `FeatureAIContext`, `FeatureTrafficMirror`, `FeatureAIIntent`, and `FeatureK8sPool` in `model/iauth/features.go`: the first four correspond to the four AI enhancement policy resources, and the last corresponds to the K8s Pool maintenance endpoints (whose sole writer is the K8s discovery component; see [Chapter 10: Provider and Cluster Design](./chapter10-provider-and-cluster.md)).

---

## Operation Log Module and the 409 Conflict Convention

### Positioning of the Operation Log Module

The operation log module records all configuration write operations on the management plane (OpenAPI), providing traceable auditability for configuration changes. The module resides in `model/ioperlog/` and is designed as follows:

- **Manager design**: `OperationLogManager` implements the two-level interfaces `OperationLogRecorder` (`Record`) and `OperationLogManagerInterface` (including `QueryLogs` and `Close`). Each business Manager injects the recorder via `SetOperationLogManager` and depends only on the minimal `OperationLogRecorder` interface, avoiding a reverse dependency on the query capability;
- **Storage implementation**: `storage/rdb/ioperlog/operation_log.go` implements `BatchCreate` (batch insert) and `List` (filtered query with pagination). The DAO counterpart is `storage/rdb/internal/dao/table_operation_logs.go`;
- **Query interface**: `endpoints/openapi_v1/operation_log/list.go` exposes `GET /open-api/v1/operation-logs`, authorized by `iauth.FeatureOperationLog + ActionReadAll`; both `ScopeSystem` (system administrator) and `ScopeSystemReadonly` (system administrator, read-only) carry that read permission.

### Asynchronous Buffered Write Mechanism

`OperationLogManager` uses an "asynchronous buffer + batch persistence" write path, so that log writes do not amplify latency on management interfaces:

```
Business Manager ──Record()──▶ buffered channel (default 4096 entries)
                                │
                                ▼
                    batchWorker: flush when 200 entries accumulate or 5s elapses
                                │ BatchCreate
                                ▼
                    Retry 3 times on failure (100ms/200ms/300ms backoff)
```

Key parameters and behaviors (`model/ioperlog/manager.go`):

| Parameter | Default | Description |
|------|--------|------|
| Buffer capacity | 4096 | `NewOperationLogManager(storager, bufferSize)`; the default is used when `bufferSize <= 0` |
| Batch size | 200 | Flush as soon as the batch is full |
| Flush interval | 5s | Periodic forced flush |
| Retry count | 3 | Backoff increases by 100ms per attempt |
| Overflow strategy | Sync fallback | When the buffer is full, `OverflowStrategySync` blocks the caller and writes synchronously by default; `OverflowStrategyDiscard` is also available to drop the log with a warning |

On `Close`, the buffer is drained within a 500ms timeout so that buffered logs are persisted before exit.

### Log Content and Correlation

The fields of `OperationLogEntry` fall into four groups: operator, resource, result, and request context. Two design points deserve explanation:

1. **Correlation with access logs**: `log_id` is identical to the `LogID` in the access log. The context extractor `operationLogContextExtractor` (`stateful/container/rdb/components.go`) pulls `LogID`, request path, method, ClientIP, and UserAgent from `xreq.GetRequestInfo(ctx)` into the entry, so operation logs can be cross-searched with access logs and deduplicated by `log_id`;
2. **Operator extraction**: the operator is extracted from `iauth.MustGetVisitor(ctx)`; `operator_type` is `0` (user, taking `User.ID`) or `1` (token, taking `Token.ID`).

Failed operations are also recorded: `status=2`, and `error_msg` is truncated to 1024 characters by `TruncateErrorMessageDefault` (appending `...` when truncated), aligned with the `error_msg varchar(1024)` column in the DDL.

### Change Summary and Masking

`change_summary` (a `mediumtext` column storing JSON) records before/after snapshots of the key fields involved in a change, constructed by `BuildChangeSummary` in `model/ioperlog/change_summary.go`:

- `before` / `after`: field maps before and after the change, both passed through `MaskSensitiveFields` first;
- `diff_keys`: present only on update actions. It lists the keys that are **newly added in after or whose values changed relative to before** (keys that exist only in before are excluded, so partial update requests do not produce false-positive diffs; unsubmitted fields, i.e. nil-valued keys, are dropped by `ParamToMap` during snapshot construction), stored in sorted order.

The audit snapshot of the cluster resource is an exception: instead of the default `ParamToMap` mapping, it is hand-written in the API lower-case vocabulary consistent with `GET /clusters` (e.g. `hash_strategy` as a string enum, `epp_config` as a decoded JSON object, `balance_mode` normalized to `WRR` when empty), with internal fields not exposed by the API — `id` / `ready` / `product_id` / `scheduler` / `sub_clusters` — trimmed, keeping `change_summary` consistent with API semantics.

Masking rules (`model/ioperlog/mask.go`) match lower-cased key names and recurse into nested maps:

| Key name | Masked result |
|------|----------|
| `password` / `secret` / `session_key` / `private_key` / `token` / `access_token` / `refresh_token` / `session_token` / `id_token` / `secret_key` / `client_secret` / `access_key` (and their variants without underscores) | `******` |
| `api_key` / `apikey` / `key` | First and last 4 characters kept, the middle replaced with `****`; fully masked when the length is 8 or less |
| `certificate` / `cert` / `cert_body` / `private_key_body` | `[已更新]` |

`error_msg` is masked at the source when the audit entry is constructed: sensitive plaintext values such as API-Keys are replaced throughout the message, with any Key plaintext rendered in the masked form that keeps the first and last 4 characters (`ioperlog.MaskErrorMessage`, invoked from `model/api_key/api_key_operation_log.go`), so error echoes never leak credentials.

### Coverage

The resources and actions that actually produce operation logs are as follows:

| Resource | Actions |
|------|------|
| entities / entity-types | create / update / delete |
| api-keys | create / update / delete |
| clusters (POST/PUT/DELETE) | create / update / delete |
| routes, global-route-rules (PUT) | update; updates to the Global route table are recorded as `resource_type=route, resource_id=global` |
| certificates | create / update / delete |
| quota-plans | create / update / delete / reset (quota balance reset) |
| model-prices | create / update / delete / import (full-table import) |
| auth/users (including session reset) | create / update / delete / reset |
| auth/tokens | create / delete |

quota_plan and rate_limit_policy have no standalone OpenAPI resources, but they do produce audit logs through the nested create / update / delete of Entity / API-Key: `resource_type` is `quota_plan` / `rate_limit_policy` respectively, and `resource_parent_id` records the owning Entity or API-Key. Domains currently do not produce operation logs.

### Query Interface

`GET /open-api/v1/operation-logs` supports the following query parameters (bound via `xreq.BindForm`):

| Parameter | Description |
|------|------|
| `operator_name` | Operator name |
| `action` / `resource_type` / `resource_id` / `resource_name` / `resource_parent_id` | Resource-dimension filters |
| `status` | `1` = success, `2` = failed |
| `start_time` / `end_time` | Unix timestamp in seconds |
| `page` / `page_size` | Defaults 1 / 20; `page_size` capped at 100 |

The response is `{list, pagination:{page, page_size, total}}`; each log entry carries `id, log_id, operator_type, operator_id, operator_name, action, resource_type, resource_id, resource_name, resource_parent_id, status, error_msg, change_summary, request_path, request_method, client_ip, user_agent, created_at` (with `created_at` as a Unix timestamp in seconds).

Another detail on the query path: the DAO function `TOperationLogCount` removes `_limit` and `_orderby` from the where map before constructing the count SQL, so pagination parameters do not pollute `COUNT(*)` and `total` always reflects the full match count on any page.

### The 409 Conflict Convention

In `lib/xerror`, `WrapConflictErrorWithMsg` marks an error as `Model.Conflict`, which `resolve.go` maps uniformly to HTTP **409 Conflict**. It expresses the business semantics of "the resource is referenced by other resources and the operation has a dependency conflict":

- A BFE cluster cannot be deleted while referenced by the `lb_matrix` (scheduler) of any AI cluster (`endpoints/openapi_v1/bfe_cluster/delete.go` calls `ClusterManager.IsBFEClusterUsed`);
- A Pool cannot be deleted while referenced by a BFECluster / SubCluster;
- A certificate cannot be deleted while referenced by a Product;
- A product cluster cannot be deleted while referenced by routing rules (advance / basic);
- A cluster cannot be deleted while referenced as the target / fallback of AI routing rules; updating cluster models also fails when they are referenced by rule target / fallback;
- An Entity with children cannot be deleted.

409 is used for dependency conflicts, 500 is the fallback for unknown errors, and 422 is reserved for parameter errors (`etParam`), so the three semantics are never conflated.

---

## The Unified xreq.Endpoint Abstraction

The interface layer uses a unified `xreq.Endpoint` abstraction to describe each HTTP interface:

```go
// lib/xreq/endpoint.go
type Endpoint struct {
    Path            string
    Method          string
    Handler         Handler
    Authorizer      func(*http.Request) error
    RegisterHandler func(*mux.Router) *mux.Route
}
```

Each business sub-package exports its own slice of Endpoint variables, which are ultimately merged and registered on the `gorilla/mux` router in `endpoints/openapi_v1/endpoints.go` or `endpoints/innerapi_v1/endpoints.go`. This design brings three benefits:

1. **Self-describing interfaces**: path, method, handler, and authorization function are centralized in one place;
2. **Unified registration**: the top-level `endpoints.go` is only responsible for merging, without handling business logic;
3. **Consistent authorization**: both OpenAPI and InnerAPI inject `iauth.FA` or `iauth.FAP` via `Authorizer` to perform permission checks.

Take the Entity-Type creation interface as an example:

```go
// endpoints/openapi_v1/entity_type/create.go
var EntityTypeCreateRoute = &xreq.Endpoint{
    Path:   "/entity-types",
    Method: http.MethodPost,
    Handler: xreq.Convert(EntityTypeCreateAction),
    Authorizer: iauth.FA(iauth.FeatureEntityType, iauth.ActionCreate),
}

func EntityTypeCreateAction(req *http.Request) (interface{}, error) {
    param := &entity.EntityTypeParam{}
    if err := xreq.BindJSON(req, param); err != nil {
        return nil, err
    }

    if param.TypeName == nil || *param.TypeName == "" {
        return nil, xerror.WrapParamErrorWithMsg("type_name is required")
    }
    if param.Level == nil || *param.Level < 1 || *param.Level > 5 {
        return nil, xerror.WrapParamErrorWithMsg("level must be between 1 and 5")
    }

    existing, err := container.EntityTypeManager.FetchEntityType(req.Context(), &entity.EntityTypeFilter{
        TypeName: param.TypeName,
    })
    if err != nil {
        return nil, err
    }
    if existing != nil {
        return nil, xerror.WrapDuplicateData("entity type")
    }

    if _, err := container.EntityTypeManager.CreateEntityType(req.Context(), param); err != nil {
        return nil, err
    }

    return container.EntityTypeManager.FetchEntityType(req.Context(), &entity.EntityTypeFilter{
        TypeName: param.TypeName,
    })
}
```

This example also illustrates the typical flow of the interface layer: parameter binding → basic validation → business existence check → invoking the model layer → returning the result.

---

## Global Container and Dependency Injection

The AI Gateway API uses a **global container + manual dependency injection** pattern. All Manager and Storager singletons are declared in `stateful/container/components.go`, and the initialization logic is centralized in `stateful/container/rdb/components.go:Init()`.

Some key singletons in the container are as follows:

```go
// stateful/container/components.go (excerpt)
var (
    TxnStoragerSingleton itxn.TxnStorager

    ProductStoragerSingleton    ibasic.ProductStorager
    ClusterStoragerSingleton    icluster_conf.ClusterStorager
    ProviderStoragerSingleton   iprovider.ProviderStorager
    APIKeyStorager              api_key.APIKeyStorager
    QuotaPlanStorager           quota.QuotaPlanStorager
    RateLimitPolicyStorager     rate_limit_policy.RateLimitPolicyStorager
    RouteRulesStorager          shared.RouteRulesStorager

    ProductManager           *ibasic.ProductManager
    ClusterManager           *icluster_conf.ClusterManager
    ProviderManager          *iprovider.ProviderManager
    APIKeyManager            *api_key.APIKeyManager
    QuotaPlanManager         *quota.QuotaPlanManager
    RateLimitPolicyManager   *rate_limit_policy.RateLimitPolicyManager
    RouteRulesManager        *route_rules.RouteRulesManager
)
```

`rdb.Init()` completes initialization in the following order:

1. Transaction and basic Storage: `TxnStoragerSingleton`, and basic/cluster/auth Storages;
2. Operation logs: `OperationLogStorager` and `OperationLogManager` (with the context extractor `operationLogContextExtractor`), initialized early so that it can be injected into all subsequent business Managers;
2. Basic Managers: `ExtraFileManager`, `VersionControlManager`, `BFEClusterManager`, `CertificateManager`, `ProductManager`, `AIRouteRuleManager`, `RouteRuleManager`, etc.;
3. Cluster-related Managers: `ClusterManager`, `SubClusterManager`, `DomainManager`, `PoolManager`;
4. Authentication and authorization Managers: `AuthenticateManager`, `AuthorizeManager`;
5. Entity / RateLimit / Quota related Storages and Managers;
6. Module export Managers: `APIKeyRuleManager`, `ModBodyProcessManager`, `AIRouteExporter`;
7. API-Key Manager: injects Quota/RateLimit/Entity adapters, integrating with the quota/rate-limit system;
8. Periodic tasks: `BalanceSyncManager`, `QuotaResetScheduler`, and starts the scheduler;
9. Default Global Route Table: calls `RouteRulesManager.EnsureGlobalRouteRules` to ensure a default record with `type=global, owner=global` exists in the `route_rules` table.

For example:

```go
// stateful/container/rdb/components.go (excerpt)
container.TxnStoragerSingleton = txn.NewRDBTxnStorager(stateful.NewBFEDBContext)
container.ProductStoragerSingleton = basic.NewProductStorage(stateful.NewBFEDBContext)
container.ClusterStoragerSingleton = cluster_conf.NewRDBClusterStorager(stateful.NewBFEDBContext)

container.ProductManager = ibasic.NewProductManager(
    container.TxnStoragerSingleton,
    container.ProductStoragerSingleton,
)
container.ClusterManager = icluster_conf.NewClusterManager(
    container.TxnStoragerSingleton,
    container.ClusterStoragerSingleton,
    // ... other dependencies
)
```

To reduce cross-package dependencies, `model/quota` also provides adapters that adapt `entity.EntityStorager`, `rate_limit_policy.RateLimitPolicyStorager`, and `quota.QuotaPlanStorager` to the generic interfaces defined in `model/shared`, so that `APIKeyManager` / `EntityManager` can call them uniformly.

```go
quota.NewQuotaPlanStoragerAdapter(quotaPlanStorager)
quota.NewRateLimitPolicyStoragerAdapter(rateLimitPolicyStorager)
quota.NewEntityStoragerAdapter(entityStorager)
```

---

## Startup Flow

The startup flow of the AI Gateway API begins at `main.go`, goes through configuration loading, database initialization, dependency injection, and route registration, and finally starts the HTTP service:

```
main.go
  │
  ├─ flag.Parse()                 # Parse command-line arguments
  ├─ stateful.LoadConfig()        # Load the TOML configuration file
  ├─ config.Init()                # Initialize path variables
  ├─ config.InitLog()             # Initialize logging
  ├─ config.Depends.Init()        # Initialize dependencies such as i18n, nav_tree
  ├─ config.InitDB()              # Initialize the database connection pool
  ├─ rdb.Init()                   # Dependency injection: initialize Managers/Storagers
  │
  └─ serverStartUp()
       ├─ NewMonitorServerWithRun()  # Start the monitoring port
       ├─ endpoints.RegisterRouters() # Register OpenAPI/InnerAPI routes
       └─ graceful.Run()             # Start the HTTP service
```

The default configuration file `conf/ai_gateway_api.toml` contains the service ports, logging, database, dependency paths, and runtime switches:

```toml
# conf/ai_gateway_api.toml (excerpt)
[Server]
ServerPort = 8183
MonitorPort = 8284
GracefulTimeOutInMs = 5000

[Databases.bfe_db]
DBName = "open_bfe"
Addr   = "127.0.0.1:3306"
Net    = "tcp"
User   = "{user}"
Passwd = "{password}"
Driver = "mysql"
MaxOpenConns = 500
MaxIdleConns = 100

[RunTime]
SkipTokenValidate = false
RecordSQL = true
SessionExpireInDay = 10
StaticFilePath = "./static"
Debug = false
```

The route registration entry `endpoints/router.go` is responsible for mounting global middleware and registering OpenAPI and InnerAPI under different prefixes:

```go
// endpoints/router.go (illustrative)
func RegisterRouters(router *mux.Router) {
    router.Use(middleware.MCRecovery)
    router.Use(middleware.MCLogger)
    router.Use(middleware.MCCors)

    openapi_v1.RegisterEndpoints(router)
    innerapi_v1.RegisterRouter(router)
}
```

`graceful.Run` starts the HTTP service while registering graceful shutdown logic: upon receiving an exit signal, it waits for in-flight requests to complete or for the timeout set by `GracefulTimeOutInMs` to elapse, and then closes the listener.

---

## Configuration Examples and Code Snippets

### Cluster Forwarding Policy Example

With Provider and Cluster separated, the Cluster's `llm_config` retains only the forwarding policy, while backend capabilities are obtained by referencing a Provider via `provider`:

```json
{
    "name": "deepseek-cluster",
    "llm_config": {
        "provider": "deepseek",
        "models": ["deepseek-chat", "deepseek-coder"],
        "model_mappings": [
            {"source_model": "gpt-4", "target_model": "deepseek-chat"}
        ],
        "keys": [
            {"name": "key-primary", "weight": 70},
            {"name": "key-secondary", "weight": 30}
        ],
        "key_policy": {
            "strategy": "weighted_random",
            "max_retries": 3,
            "retry_backoff_initial": 500,
            "retry_backoff_max": 5000
        },
        "key_affinity": {
            "enabled": true,
            "ttl": 600,
            "redis_prefix": "bfe:ai:key_affinity",
            "penalty_enable": true
        },
        "match_prefix": "deepseek/",
        "strip_prefix": true
    }
}
```

### Model Layer Transaction Orchestration Example

The model layer orchestrates atomic operations across multiple Storagers via `itxn.TxnStorager.AtomExecute`. The following snippet shows the typical pattern when creating an API-Key, cascading the creation of QuotaPlan, RateLimitPolicy, and RouteRules:

```go
// model/api_key/api_key.go (illustrative)
func (m *APIKeyManager) CreateAPIKey(ctx context.Context, param *APIKeyParam) error {
    return m.txn.AtomExecute(ctx, func(ctx context.Context) error {
        if param.QuotaPlan != nil {
            quotaPlanID, err := m.quotaPlanStorager.CreateQuotaPlan(ctx, param.QuotaPlan)
            if err != nil {
                return err
            }
            param.QuotaPlanID = &quotaPlanID
        }
        if param.RateLimitPolicy != nil {
            rateLimitPolicyID, err := m.rateLimitPolicyStorager.CreateRateLimitPolicy(ctx, param.RateLimitPolicy)
            if err != nil {
                return err
            }
            param.RateLimitPolicyID = &rateLimitPolicyID
        }
        if param.RouteRules != nil {
            routeRulesID, err := m.routeRulesStorager.CreateRouteRules(
                ctx, shared.RouteRulesTypeAPIKey, param.Key, param.RouteRules)
            if err != nil {
                return err
            }
            param.RouteRulesID = &routeRulesID
        }
        _, err := m.storager.CreateAPIKey(ctx, param)
        return err
    })
}
```

### InnerAPI Incremental Export Example

InnerAPI export interfaces parse the `version` parameter via `export_util.NewExportFromReq` and hand it off to the corresponding Manager's `ConfigExport` method:

```go
// endpoints/innerapi_v1/mod_api_key/export.go
var ExportRoute = &xreq.Endpoint{
    Path:   "/configs/mod-api-key",
    Method: http.MethodGet,
    Handler: xreq.Convert(ExportAction),
    Authorizer: iauth.FA(iauth.FeatureAPIKey, iauth.ActionExport),
}

func ExportAction(req *http.Request) (interface{}, error) {
    param, err := export_util.NewExportFromReq(req)
    if err != nil {
        return nil, err
    }
    return container.APIKeyRuleManager.ConfigExport(req.Context(), param.Version)
}
```

When the local version matches the remote one, `ConfigExport` returns empty data, and Conf Agent will not trigger a BFE hot reload, thereby reducing network and Control Plane load.

---

## Chapter Summary

- The AI Gateway API is the Control Plane core of the Rainway AI Gateway, bearing the dual responsibilities of the management-plane OpenAPI and the data-plane InnerAPI.
- The system adopts a three-layer architecture of interface layer, model layer, and storage layer, with each layer decoupled through clear interface boundaries.
- OpenAPI faces administrators and the Dashboard, and is responsible for resource configuration; InnerAPI faces BFE and Conf Agent, and is responsible for configuration export and incremental synchronization.
- `xreq.Endpoint` unifies the description, registration, and authorization of interfaces, reducing boilerplate code in interface development.
- `stateful/container` provides a global singleton container, and `stateful/container/rdb/components.go:Init()` performs manual dependency injection in dependency order.
- The operation log module (`model/ioperlog`) records management-plane write operations via asynchronous buffering and batch persistence; `log_id` correlates with access logs, and sensitive fields are masked before being stored in the `operation_logs` table.
- Resource dependency conflicts uniformly return 409 Conflict (`xerror.WrapConflictErrorWithMsg`), while 500 is restored as the fallback for unknown errors.
- The startup flow begins with configuration loading in `main.go`, goes through database initialization, dependency injection, and route registration, and finally starts the HTTP service via `graceful.Run`.

---

## References

- `ai-gateway-api/design-docs/sys-design/总体设计文档.md`
- `ai-gateway-api/design-docs/sys-design/接口层设计文档.md`
- `ai-gateway-api/design-docs/sys-design/模型层设计文档.md`
- `ai-gateway-api/design-docs/sys-design/存储层设计文档.md`
- `ai-gateway-api/design-docs/sys-design/details/操作日志模块.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/operation-logs.md`
- `ai-gateway-api/AGENTS.md`
- `ai-gateway-api/conf/ai_gateway_api.toml`
