# Chapter 28: Storage Layer Implementation: DAO and Storage

## Chapter Goals

This chapter focuses on the lowest-level data persistence mechanism of AI Gateway API (the Control Plane), explaining the two-layer **DAO (Data Access Object) + Storage** structure implemented under the `storage/rdb` directory. After reading this chapter, you will understand:

- How the `storage/rdb` directory is organized, and where the responsibilities of DAO and Storage are divided.
- The SQL building and scanning mechanism based on `github.com/didi/gendry`.
- The generic code template of the DAO layer, CRUD function naming, and field conventions.
- How the Storage layer exposes interfaces to the `model/*` subpackages and converts business models to database models.
- The Storage mapping for the 25 tables.
- The transaction abstraction `itxn.TxnStorager` and the implementation in `storage/rdb/txn`.
- The design rationale for the absence of physical foreign keys, and how data consistency is guaranteed.

The storage layer serves as the bridge between the Control Plane and the persistent database. It must shield callers from low-level SQL details while providing the model layer with stable, testable, and extensible access interfaces. The layered DAO + Storage design strikes a balance between these two goals.

## storage/rdb Directory Structure

`ai-gateway-api` adopts a three-layer architecture: interface layer (`endpoints/openapi_v1`) → model layer (`model/*`) → storage layer (`storage/rdb`). The storage layer is fully responsible for reading and writing the relational database. It interacts with the model layer above through interfaces, and operates on concrete tables below through DAO functions.

```
storage/rdb/
├── internal/
│   └── dao/                    # DAO implementation, organized by table
│       ├── internal/           # generic CRUD wrapper and gendry adaptation
│       ├── table_*.go          # one DAO file per table
│       └── ...
├── ai_route/                   # AI route rule Storage
├── api_key/                    # API-Key / Token Storage
├── auth/                       # authentication / authorization Storage
├── basic/                      # product line / BFE cluster / extra file Storage
├── cluster_conf/               # cluster / sub-cluster / instance pool / LB matrix / ModelPrice Storage
├── entity/                     # Entity / EntityType Storage
├── model_price/                # model pricing Storage
├── protocol/                   # TLS certificate Storage
├── provider/                   # Provider Storage
├── quota/                      # QuotaPlan Storage
├── rate_limit_policy/          # RateLimitPolicy Storage
├── route_conf/                 # domain / product-level route rule Storage (not used for Cluster selection in AI gateway mode)
├── route_rules/                # API-Key / Entity / Global route rule Storage
├── txn/                        # transaction abstraction implementation
└── version_control/            # configuration version control Storage
```

All DAO files live in `storage/rdb/internal/dao/`, one per database table. Storage files are organized by business domain in the subpackages under `storage/rdb/`, corresponding one-to-one with the `model/*` subpackages. This organization makes adding a new business domain straightforward: define the interface in `model/<domain>`, implement the Storage in `storage/rdb/<domain>`, and add the DAO file for the corresponding table in `storage/rdb/internal/dao/`. The extension path is clear.

## The Two-Layer DAO + Storage Structure

### DAO Layer Responsibilities

The DAO (Data Access Object) layer, located in `storage/rdb/internal/dao/`, is the code layer closest to the database. Each DAO file corresponds to one table and provides CRUD functions for that table. The DAO handles no business logic; it is only responsible for:

- Mapping Go structs to SQL parameters.
- Calling the generic CRUD functions in the `internal` package.
- Completing field mapping using `db:"column_name"` tags.

### Storage Layer Responsibilities

The Storage layer, located in `storage/rdb/<domain>/`, exposes interfaces to the `model/*` subpackages. Storage is responsible for:

- Implementing the Storager interfaces defined in `model/<domain>`.
- Converting model-layer Param / Filter into DAO Params.
- Handling JSON serialization, pagination calculation, and timestamp filling.
- Coordinating reads and writes across multiple tables in the same domain when needed.

### How the Two Layers Cooperate

The diagram below shows the data flow of a create request:

```
+-------------+      +------------------+      +------------------+
| model layer | ---> | Storage layer    | ---> | DAO layer        |
| Manager     |      | EntityTypeStorager|     | table_entity_types|
+-------------+      +------------------+      +------------------+
                            |                           |
                            v                           v
                     business model to            gendry builds SQL
                     DAO model                       |
                            |                           |
                            +-----------> DB <----------+
```

The Manager of the model layer holds the Storage interface; Storage obtains the database context via `lib.DBContextFactory`; the DAO executes SQL using the same `DBContexter`. Because `DBContexter` can carry either a normal connection or a transactional one, Storage and DAO do not need to care whether they are currently inside a transaction. They simply execute SQL in a uniform way, and transaction propagation is handled by the context factory.

## Using the gendry SQL Builder

The project uses the `builder` package of `github.com/didi/gendry` to construct SQL and the `scanner` package to scan results. The dependency version declared in `go.mod` is `v1.7.0`:

```go
// ai-gateway-api/go.mod
	github.com/didi/gendry v1.7.0
```

`storage/rdb/internal/dao/internal/builder.go` provides a thin wrapper over gendry, offering `SelectBuilder`, `InsertBuilder`, `UpdateBuilder`, and `DeleteBuilder`, and uniformly handling `db` tags.

`QueryOne`, `QueryList`, `Create`, `Update`, and `Delete` in `storage/rdb/internal/dao/internal/curd.go` are the final execution entry points of the DAO layer:

```go
// storage/rdb/internal/dao/internal/curd.go
func QueryOne(dbCtx lib.DBContexter, table string, where interface{}, rst interface{}) error
func QueryList(dbCtx lib.DBContexter, table string, where interface{}, rst interface{}) error
func Create(dbCtx lib.DBContexter, table string, data ...interface{}) (int64, error)
func Update(dbCtx lib.DBContexter, table string, where interface{}, data interface{}) (int64, error)
func Delete(dbCtx lib.DBContexter, table string, where interface{}) (int64, error)
```

Take `QueryList` as an example: it uses reflection to convert the `where` struct into a `map[string]interface{}`, which is then handed to gendry to generate SQL:

```go
// storage/rdb/internal/dao/internal/curd.go
func queryList(dbCtx lib.DBContexter, table string, where interface{}, rst interface{}, queryOne bool) error {
    tmp := Struct2Where(where)
    if tmp != nil && queryOne {
        tmp["_limit"] = []uint{0, 1}
    }

    build := NewSelectBuilder(table, tmp, nil)
    sql, args, err := build.Compile()
    if err != nil {
        return xerror.WrapDaoError(err)
    }

    rows, err := dbCtx.Conn().QueryContext(dbCtx, sql, args...)
    // ... SQL logging and error handling
    return scanner.Scan(rows, rst)
}
```

`Struct2Where` is implemented in `builder.go`. It reads the `db` tags of struct fields, automatically skips nil pointers and empty slices, and parses tags carrying operators into `column op` form. For example, `db:"id,>"` generates `id > ?`. This design lets query conditions be freely combined: the caller only needs to fill in the desired fields in `T<Table>Param`, and unfilled fields will not appear in the WHERE clause — avoiding the tedium and error-prone nature of concatenating conditions in handwritten SQL.

The `scanner` package of gendry is responsible for scanning `sql.Rows` into structs or slices. In its `init` function, `curd.go` sets the scan tag to `db` via `scanner.SetTagName(tagName)`, ensuring that the `T<Table>` structs of the DAO can be used both for parameter building and for result scanning — one tag serving two purposes.

## The Generic DAO Template

Each DAO file follows a unified five-element template:

1. **Table name constant**: `const t<Table>TableName = "<table_name>"`.
2. **Result struct**: `T<Table>`, with fields tagged `db:"column_name"`, corresponding one-to-one with table columns.
3. **Parameter struct**: `T<Table>Param`, with all fields as pointer types, used for query conditions and assignments; it also contains two control fields:
   - `OrderBy *string `db:"_orderby"``: the ORDER BY clause.
   - `Limit []uint `db:"_limit"``: pagination parameters, in the form `{offset, pageSize}`.
4. **CRUD functions**:
   - `T<Table>One(dbCtx, where)`: query a single record; returns `(nil, nil)` if not found.
   - `T<Table>List(dbCtx, where)`: query multiple records; returns `(nil, nil)` if not found.
   - `T<Table>Create(dbCtx, data...)`: insert a single record or a batch; automatically fills `CreatedAt`.
   - `T<Table>Update(dbCtx, val, where)`: update by condition.
   - `T<Table>Delete(dbCtx, where)`: delete by condition.

The following uses the `entity_types` table as an example:

```go
// storage/rdb/internal/dao/table_entity_types.go
const tEntityTypeTableName = "entity_types"

type TEntityType struct {
    ID          int64     `db:"id"`
    TypeName    string    `db:"type_name"`
    Description string    `db:"description"`
    Level       int       `db:"level"`
    CreatedAt   time.Time `db:"created_at"`
    UpdatedAt   time.Time `db:"updated_at"`
}

type TEntityTypeParam struct {
    ID          *int64     `db:"id"`
    TypeName    *string    `db:"type_name"`
    Description *string    `db:"description"`
    Level       *int       `db:"level"`
    CreatedAt   *time.Time `db:"created_at"`
    UpdatedAt   *time.Time `db:"updated_at"`

    OrderBy *string `db:"_orderby"`
    Limit   []uint  `db:"_limit"`
}

func TEntityTypeOne(dbCtx lib.DBContexter, where *TEntityTypeParam) (*TEntityType, error)
func TEntityTypeList(dbCtx lib.DBContexter, where *TEntityTypeParam) ([]*TEntityType, error)
func TEntityTypeCreate(dbCtx lib.DBContexter, data ...*TEntityTypeParam) (int64, error)
func TEntityTypeUpdate(dbCtx lib.DBContexter, val, where *TEntityTypeParam) (int64, error)
func TEntityTypeDelete(dbCtx lib.DBContexter, where *TEntityTypeParam) (int64, error)
```

Some tables extend the generic template with special capabilities:

| File | Special capability | Description |
|------|--------------------|-------------|
| `table_products.go` | `TProductDeleteByProductID` | Native SQL cascading delete across multiple product-line-related tables |
| `table_route_advance_rules.go` | `ModeForUpdate`, `_lockMode` | Supports `SELECT ... FOR UPDATE` row locks |
| `table_pools.go` | `PoolsList2Map` | Convert list to map |
| `table_clusters.go` | `IDs []int64`, `Names []string` | Supports `IN` queries |
| `table_sub_clusters.go` | Multi-field `IN` queries | Supports `IDs`, `Names`, `ClusterIDs`, `PoolsIDs` |

These extended capabilities are implemented by adding slice fields to `T<Table>Param`. `Struct2Where` puts slice fields into the `where map` as-is, and gendry's `builder` package automatically converts them into `IN` clauses. For `SELECT ... FOR UPDATE`, a `_lockMode` field is added to `T<Table>Param`, and the DAO appends the lock hint after calling `internal.QueryList`. It is worth emphasizing that these special capabilities remain DAO-layer details; the Storage layer only needs to set parameters according to business semantics when using them.

## How Storage Implementations Expose Interfaces Upward

Each Storage implementation follows four conventions:

1. **Dependency injection**: receives `lib.DBContextFactory` via a constructor.
2. **Interface assertion**: uses `var _ <Interface> = &<Storage>{}` for compile-time checking.
3. **Type conversion**: converts between model parameters and DAO parameters.
4. **JSON serialization**: serializes complex structures into strings before storing them in the database.

Take `entity/entity_type.go` as an example:

```go
// storage/rdb/entity/entity_type.go
type EntityTypeStorager struct {
    dbCtxFactory lib.DBContextFactory
}

func NewEntityTypeStorager(dbCtxFactory lib.DBContextFactory) *EntityTypeStorager {
    return &EntityTypeStorager{dbCtxFactory: dbCtxFactory}
}

var _ entity.EntityTypeStorager = &EntityTypeStorager{}

func (s *EntityTypeStorager) CreateEntityType(
    ctx context.Context,
    param *entity.EntityTypeParam,
) (int64, error) {
    dbCtx, err := s.dbCtxFactory(ctx)
    if err != nil {
        return 0, err
    }

    data := &dao.TEntityTypeParam{
        TypeName:    param.TypeName,
        Description: param.Description,
        Level:       param.Level,
        CreatedAt:   lib.PTimeNow(),
    }

    return dao.TEntityTypeCreate(dbCtx, data)
}
```

The corresponding interface is defined in `model/entity/entity_type.go`:

```go
// model/entity/entity_type.go
type EntityTypeStorager interface {
    CreateEntityType(ctx context.Context, param *EntityTypeParam) (int64, error)
    FetchEntityType(ctx context.Context, filter *EntityTypeFilter) (*EntityTypeParam, error)
    FetchEntityTypeList(ctx context.Context, filter *EntityTypeFilter) ([]*EntityTypeParam, error)
    UpdateEntityType(ctx context.Context, filter *EntityTypeFilter, param *EntityTypeParam) (int64, error)
    DeleteEntityType(ctx context.Context, filter *EntityTypeFilter) error
}
```

The Storage layer obtains a `lib.DBContexter` via `dbCtxFactory(ctx)`. If the call chain is already inside a transaction, the factory reuses the same transaction; otherwise it uses a normal connection. This mechanism gives the Storage layer natural transaction propagation: after the model-layer Manager calls `itxn.TxnStorager.AtomExecute` to open a transaction, the transaction context is injected into `context.Context`, and all subsequent Storage calls obtain the same transactional connection, ensuring atomicity across Storages.

JSON serialization is a common task of the Storage layer. Take `rate_limit_policy/rate_limit_policy.go` as an example: `tpm_configs` and `rpm_configs` are stored in the database as JSON strings:

```go
// storage/rdb/rate_limit_policy/rate_limit_policy.go
func rateLimitPolicyDataToParam(param *rate_limit_policy.RateLimitPolicyParam) *dao.TRateLimitPolicyParam {
    data := &dao.TRateLimitPolicyParam{
        Enabled:        param.Enabled,
        MaxConcurrency: param.MaxConcurrency,
    }

    if len(param.TpmConfigs) > 0 {
        tpmConfigsJSON, _ := json.Marshal(param.TpmConfigs)
        data.TpmConfigs = lib.PString(string(tpmConfigsJSON))
    } else {
        data.TpmConfigs = lib.PString("[]")
    }

    if len(param.RpmConfigs) > 0 {
        rpmConfigsJSON, _ := json.Marshal(param.RpmConfigs)
        data.RpmConfigs = lib.PString(string(rpmConfigsJSON))
    } else {
        data.RpmConfigs = lib.PString("[]")
    }

    return data
}
```

## Storage Mapping for the 25 Tables

According to `ai-gateway-api/design-docs/sys-design/数据库设计文档.md`, the current system has 25 persistent tables in total, divided by business module as follows.

### Basic Configuration (6 tables)

| Table | DAO file | Storage package | Description |
|-------|----------|-----------------|-------------|
| `bfe_clusters` | `table_bfe_clusters.go` | `storage/rdb/basic/bfe_cluster.go` | BFE cluster configuration |
| `products` | `table_products.go` | `storage/rdb/basic/product.go` | Product line configuration |
| `domains` | `table_domains.go` | `storage/rdb/route_conf/domain.go` | Domain configuration |
| `users` | `table_users.go` | `storage/rdb/auth/authentication.go` | User / Token |
| `user_products` | `table_user_products.go` | `storage/rdb/auth/authorization.go` | User/Token-product authorization |
| `config_versions` | `table_config_versions.go` | `storage/rdb/version_control/version_control.go` | Configuration versions |

### Providers and Cluster Instance Pools (5 tables)

| Table | DAO file | Storage package | Description |
|-------|----------|-----------------|-------------|
| `providers` | `table_providers.go` | `storage/rdb/provider/provider.go` | Provider access capabilities |
| `clusters` | `table_clusters.go` | `storage/rdb/cluster_conf/cluster.go` | AI cluster configuration |
| `sub_clusters` | `table_sub_clusters.go` | `storage/rdb/cluster_conf/sub_cluster.go` | Sub-clusters |
| `pools` | `table_pools.go` | `storage/rdb/cluster_conf/pool.go` | Instance pools |
| `lb_matrices` | `table_lb_matrix.go` | `storage/rdb/cluster_conf/cluster.go` | Load balancing matrix |

### Route Rules (6 tables)

| Table | DAO file | Storage package | Description |
|-------|----------|-----------------|-------------|
| `route_basic_rules` | `table_route_basic_rules.go` | `storage/rdb/route_conf/route_rule.go` | Basic route rules |
| `route_advance_rules` | `table_route_advance_rules.go` | `storage/rdb/route_conf/route_rule.go` | Advanced route rules |
| `route_default_rules` | `table_route_default_rules.go` | `storage/rdb/route_conf/route_rule.go` | Default forwarding rules |
| `route_cases` | — | — | Route test cases; no DAO currently |
| `ai_route_rules` | `table_ai_route_rules.go` | `storage/rdb/ai_route/ai_route.go` | AI route rules |
| `route_rules` | `table_route_rules.go` | `storage/rdb/route_rules/route_rules.go` | API-Key / Entity / Global route rules |

### Certificates and Extra Files (2 tables)

| Table | DAO file | Storage package | Description |
|-------|----------|-----------------|-------------|
| `certificates` | `table_certificates.go` | `storage/rdb/protocol/certificate.go` | TLS certificates |
| `extra_files` | `table_extra_files.go` | `storage/rdb/basic/extra_file.go` | Extra files |

### API-Key / Entity / Quota / Rate Limiting (6 tables)

| Table | DAO file | Storage package | Description |
|-------|----------|-----------------|-------------|
| `api_keys` | `table_api_keys.go` | `storage/rdb/api_key/api_key.go` | API-Key |
| `api_key_tokens` | `table_api_key_tokens.go` | `storage/rdb/api_key/api_key.go` | API-Key Token records |
| `entity_types` | `table_entity_types.go` | `storage/rdb/entity/entity_type.go` | Entity types |
| `entities` | `table_entities.go` | `storage/rdb/entity/entity.go` | Entity entities |
| `quota_plans` | `table_quota_plans.go` | `storage/rdb/quota/quota_plan.go` | Quota plans |
| `rate_limit_policies` | `table_rate_limit_policies.go` | `storage/rdb/rate_limit_policy/rate_limit_policy.go` | Rate limit policies |

### Model Pricing (1 table)

| Table | DAO file | Storage package | Description |
|-------|----------|-----------------|-------------|
| `model_prices` | `table_model_prices.go` | `storage/rdb/model_price/model_price.go` | Model pricing |

Note: the `route_cases` table is defined in the DDL, but there is currently no corresponding DAO or Storage implementation in the code, so the tables actually covered by DAO + Storage number 24.

From the mapping it can be seen that Storage subpackages are divided by business domain rather than by the number of database tables. For example, the `cluster_conf` subpackage manages four tables — `clusters`, `sub_clusters`, `pools`, and `lb_matrices` — because these tables jointly serve the business concept of cluster configuration. The `route_conf` subpackage manages `domains`, `route_basic_rules`, `route_advance_rules`, and `route_default_rules` at the same time, because together they form the product-level route rules (in AI gateway mode they are not used for Cluster selection of AI requests, and are only used for product line identification context or non-AI traffic scenarios). This business-domain aggregation makes Storage interfaces closer to the call patterns of the model-layer Manager, avoiding the complexity of a Manager depending on multiple fine-grained Storages simultaneously.

## Transaction Implementation (storage/rdb/txn)

The model layer does not operate database transactions directly; instead it goes through the `model/itxn.TxnStorager` interface:

```go
// model/itxn/txn.go
type TxnStorager interface {
    AtomExecute(ctx context.Context, do func(context.Context) error) error
}
```

`storage/rdb/txn/txn.go` provides a relational-database-based implementation:

```go
// storage/rdb/txn/txn.go
type RDBTxnStorager struct {
    dbCtxFactory lib.DBContextFactory
}

func NewRDBTxnStorager(dbCtxFactory lib.DBContextFactory) *RDBTxnStorager {
    return &RDBTxnStorager{dbCtxFactory: dbCtxFactory}
}

var _ itxn.TxnStorager = &RDBTxnStorager{}

func (ps *RDBTxnStorager) AtomExecute(ctx context.Context, do func(context.Context) error) error {
    dbCtx, err := ps.dbCtxFactory(ctx, lib.OpenTxn())
    if err != nil {
        return err
    }

    return lib.RDBTxnExecute(dbCtx, do)
}
```

`lib.OpenTxn()` instructs the factory to open a transaction; `lib.RDBTxnExecute` decides whether to commit or roll back based on the error after the callback finishes.

Cross-table operations are usually orchestrated by the model-layer Manager inside the `AtomExecute` callback. For example, `EntityManager.CreateEntity` creates the QuotaPlan, RateLimitPolicy, RouteRules, and Entity in sequence; `RouteRuleStorager.UpsertProductRule` locks first, then deletes, then batch-inserts. The Storage layer itself generally does not open transactions; it is only responsible for read/write conversion of a single table or a few tables within the same domain.

This layering makes transaction boundaries very clear: only the model-layer Manager knows which operations need atomicity, so it decides when to open a transaction; Storage and DAO are only responsible for executing SQL within the provided context. During testing, `itxn.TxnStorager` can also be conveniently replaced with an in-memory implementation to isolate the database dependency.

## Design Rationale for No Physical Foreign Keys

According to the database design document, the DDL of all tables **declares no physical FOREIGN KEY constraints**. Inter-table relationships are all logical foreign keys, maintained by application-layer code. The main considerations are:

- **Performance**: MySQL's foreign key checks add overhead to write operations and easily become a bottleneck under high concurrency.
- **Flexibility**: facilitates database sharding, data migration, and gray releases; physical foreign keys restrict the freedom of schema changes.
- **Batch deletion**: e.g., `TProductDeleteByProductID` uses native SQL to delete data from multiple product-line-related tables in one go, avoiding the uncontrollable behavior of cascading triggers.
- **Clearer reference relationships**: fields such as `quota_plan_id`, `rate_limit_policy_id`, and `route_rules_id` in `api_keys` carry no physical constraints, but indexes accelerate queries, and the model layer performs existence checks.

The absence of physical foreign keys also means the application layer must guarantee consistency by itself. A common practice in the project is: orchestrating multi-table operations in the model-layer Manager via `itxn.TxnStorager`, and explicitly cleaning up associated data on critical paths (such as deleting a product or deleting an API-Key). For example, when deleting a product, the Manager corresponding to `basic/product.go` calls `TProductDeleteByProductID` to clean up tables such as `products`, `domains`, `route_basic_rules`, `route_advance_rules`, and `route_default_rules` in one go, avoiding routing anomalies caused by leftover data.

It should be pointed out that not using physical foreign keys does not mean giving up data integrity. The project compensates in three ways: first, indexes are created on related fields in the DAO to accelerate validation and queries; second, the model layer performs explicit existence checks before writing; third, critical deletion paths use transactions to guarantee synchronized cleanup across tables. This design better suits a gateway control plane scenario that is read-heavy and write-light, with frequent schema changes.

## Key Code Snippets

### 1. Generic DAO Template: `table_entity_types.go`

```go
// storage/rdb/internal/dao/table_entity_types.go
const tEntityTypeTableName = "entity_types"

type TEntityType struct {
    ID          int64     `db:"id"`
    TypeName    string    `db:"type_name"`
    Description string    `db:"description"`
    Level       int       `db:"level"`
    CreatedAt   time.Time `db:"created_at"`
    UpdatedAt   time.Time `db:"updated_at"`
}

type TEntityTypeParam struct {
    ID          *int64     `db:"id"`
    TypeName    *string    `db:"type_name"`
    Description *string    `db:"description"`
    Level       *int       `db:"level"`
    CreatedAt   *time.Time `db:"created_at"`
    UpdatedAt   *time.Time `db:"updated_at"`

    OrderBy *string `db:"_orderby"`
    Limit   []uint  `db:"_limit"`
}

func TEntityTypeOne(dbCtx lib.DBContexter, where *TEntityTypeParam) (*TEntityType, error) {
    t := &TEntityType{}
    err := internal.QueryOne(dbCtx, tEntityTypeTableName, where, t)
    if err == nil {
        return t, nil
    }
    if xerror.Cause(err) == internal.ErrRecordNotFound {
        return nil, nil
    }
    return nil, err
}
```

### 2. Storage Interface Implementation: `entity_type.go`

```go
// storage/rdb/entity/entity_type.go
var _ entity.EntityTypeStorager = &EntityTypeStorager{}

func (s *EntityTypeStorager) CreateEntityType(
    ctx context.Context,
    param *entity.EntityTypeParam,
) (int64, error) {
    dbCtx, err := s.dbCtxFactory(ctx)
    if err != nil {
        return 0, err
    }

    data := &dao.TEntityTypeParam{
        TypeName:    param.TypeName,
        Description: param.Description,
        Level:       param.Level,
        CreatedAt:   lib.PTimeNow(),
    }

    return dao.TEntityTypeCreate(dbCtx, data)
}
```

### 3. Full Replacement of Product-Level Route Rules: `route_rule.go`

> Note: the following Storage function handles the persistence of product-level route rules (`route_basic_rules` / `route_advance_rules` / `route_default_rules`). In AI gateway mode, these rules are not used for Cluster selection of AI requests; they are only used for product line identification context or non-AI traffic scenarios.

```go
// storage/rdb/route_conf/route_rule.go
func (rs *RouteRuleStorager) UpsertProductRule(
    ctx context.Context,
    product *ibasic.Product,
    rule *iroute_conf.ProductRouteRule,
) error {
    // ... build daoBasicRules and daoAdvanceRules

    dbCtx, err := rs.dbCtxFactory(ctx)
    if err != nil {
        return err
    }

    if _, err := dao.TRouteAdvanceRuleList(dbCtx, &dao.TRouteAdvanceRuleParam{
        ProductID: &product.ID,
        LockMode:  &dao.ModeForUpdate,
    }); err != nil {
        return err
    }

    if _, err := dao.TRouteAdvanceRuleDelete(dbCtx, &dao.TRouteAdvanceRuleParam{
        ProductID: &product.ID,
    }); err != nil {
        return err
    }

    if _, err := dao.TRouteBasicRuleDelete(dbCtx, &dao.TRouteBasicRuleParam{
        ProductID: &product.ID,
    }); err != nil {
        return err
    }

    if len(daoAdvanceRules) > 0 {
        if _, err := dao.TRouteAdvanceRuleCreate(dbCtx, daoAdvanceRules...); err != nil {
            return err
        }
    }

    if len(daoBasicRules) > 0 {
        if _, err := dao.TRouteBasicRuleCreate(dbCtx, daoBasicRules...); err != nil {
            return err
        }
    }

    return nil
}
```

This function shows how the Storage layer coordinates multiple tables within the same domain: it first takes a `SELECT ... FOR UPDATE` lock, then deletes the old rules, and finally batch-inserts the new rules. The transaction boundary is guaranteed by the caller via `itxn.TxnStorager`.

### 4. In-Memory Pagination of Route Rule Lists: `route_rules.go`

```go
// storage/rdb/route_rules/route_rules.go
func (s *RouteRulesStorager) FetchRouteRulesList(
    ctx context.Context,
    filter *shared.RouteRulesFilter,
) ([]*shared.RouteTableParam, int64, error) {
    dbCtx, err := s.dbCtxFactory(ctx)
    if err != nil {
        return nil, 0, err
    }

    where := routeRulesFilterToParam(filter)

    allList, err := dao.TRouteRulesList(dbCtx, where)
    if err != nil {
        return nil, 0, err
    }
    total := int64(len(allList))

    page := 1
    pageSize := 20
    if filter.Page != nil && *filter.Page > 0 {
        page = *filter.Page
    }
    if filter.PageSize != nil && *filter.PageSize > 0 {
        pageSize = *filter.PageSize
        if pageSize > 100 {
            pageSize = 100
        }
    }
    offset := (page - 1) * pageSize
    where.Limit = []uint{uint(offset), uint(pageSize)}

    list, err := dao.TRouteRulesList(dbCtx, where)
    // ... convert to RouteTableParam
    return result, total, nil
}
```

Here the full record set is queried first to compute `total`, and then the paginated records are queried. The returned result contains only metadata such as `type`, `owner`, and `enabled`, and does not return the potentially very large `rules` field. This design sacrifices a certain amount of database queries in exchange for a controllable API response size, and avoids the complex logic of generating both `COUNT(*)` and `LIMIT` in the gendry builder at the same time. For a table like route rules, where a single record may contain a large amount of JSON data, in-memory pagination is a simple and effective compromise.

## Chapter Summary

This chapter introduced in detail the storage layer implementation of the Rainway AI Gateway Control Plane:

- `storage/rdb` adopts a **DAO + Storage** two-layer structure: the DAO stays close to database tables and provides generic CRUD; the Storage faces business domains and implements the interfaces defined in `model/*`.
- The DAO layer builds SQL based on `github.com/didi/gendry`; the generic CRUD wrapper lives in `storage/rdb/internal/dao/internal/curd.go`.
- Each DAO file follows a unified template: table name constant, `T<Table>` result struct, `T<Table>Param` parameter struct, and CRUD functions.
- Storage obtains the database context via `lib.DBContextFactory` and is responsible for model conversion, JSON serialization, pagination calculation, and timestamp filling.
- The 25 tables are mapped to different Storage subpackages by business module; `route_cases` currently has no DAO/Storage implementation.
- Transactions are abstracted through `model/itxn.TxnStorager`; `storage/rdb/txn/txn.go` provides an RDB-based implementation, and the model-layer Manager is responsible for orchestrating cross-table transaction boundaries.
- The database design uses no physical foreign keys; consistency is guaranteed by the application layer through logical foreign keys and transactions, balancing performance and flexibility.

## References

- `ai-gateway-api/design-docs/sys-design/存储层设计文档.md`
- `ai-gateway-api/design-docs/sys-design/数据库设计文档.md`
- `ai-gateway-api/storage/rdb/internal/dao/`
- `ai-gateway-api/storage/rdb/txn/`
- `ai-gateway-api/go.mod`
