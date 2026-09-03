# Chapter 6: Control Plane Core Design: AI Gateway API

## Chapter Goals

Through this chapter, readers will understand:

- The role and responsibility boundaries of the AI Gateway API within the Rainway AI Gateway;
- How the Control Plane organizes code in a three-layer architecture (interface layer, model layer, storage layer);
- The division of responsibilities and route organization between the management-plane OpenAPI and the data-plane InnerAPI;
- How the unified `xreq.Endpoint` abstraction simplifies interface registration, authorization, and middleware handling;
- The implementation of the global container (`stateful/container`) and manual dependency injection;
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
| Service Controller | Service discovery | Syncs backend service instance information to the Control Plane |

The current functional scope of the AI Gateway API covers: API-Key / Entity / Entity-Type management, Provider and Cluster management, model pricing management, QuotaPlan and RateLimitPolicy management, AI routing rule management, certificate and extra file management, authentication and authorization, and configuration export for the Data Plane.

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
| `model/imods/` | Export of module configurations such as mod-api-key, mod-body-process, and AI routing |
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
| `storage/rdb/provider/` | `model/iprovider` | `providers` |

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
- The startup flow begins with configuration loading in `main.go`, goes through database initialization, dependency injection, and route registration, and finally starts the HTTP service via `graceful.Run`.

---

## References

- `ai-gateway-api/design-docs/sys-design/总体设计文档.md`
- `ai-gateway-api/design-docs/sys-design/接口层设计文档.md`
- `ai-gateway-api/design-docs/sys-design/模型层设计文档.md`
- `ai-gateway-api/design-docs/sys-design/存储层设计文档.md`
- `ai-gateway-api/AGENTS.md`
- `ai-gateway-api/conf/ai_gateway_api.toml`
