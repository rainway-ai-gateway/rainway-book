# 第三十章 存储层实现：DAO 与 Storage

## 本章目标

本章聚焦 AI Gateway API（控制面）最底层的数据持久化机制，解析 `storage/rdb` 目录下 **DAO（Data Access Object）+ Storage** 的两层结构实现。阅读本章后，你将理解：

- `storage/rdb` 的目录组织方式，以及 DAO 与 Storage 的职责边界。
- 基于 `github.com/didi/gendry` 的 SQL 构建与扫描机制。
- DAO 层的通用代码模板、CRUD 函数命名与字段约定。
- Storage 层如何面向 `model/*` 各子包暴露接口，并完成业务模型到数据库模型的转换。
- 34 张表的 Storage 映射关系。
- 事务抽象 `itxn.TxnStorager` 与 `storage/rdb/txn` 的实现方式。
- 无物理外键的设计考量与数据一致性保障思路。

存储层作为控制面与持久化数据库之间的桥梁，既要屏蔽底层 SQL 细节，又要为模型层提供稳定、可测试、可扩展的访问接口。DAO + Storage 的分层设计正是为了在这两者之间取得平衡。

## storage/rdb 目录结构

`ai-gateway-api` 采用三层架构：接口层（`endpoints/openapi_v1`）→ 模型层（`model/*`）→ 存储层（`storage/rdb`）。存储层完全负责关系型数据库的读写，向上通过接口与模型层交互，向下通过 DAO 函数操作具体表。

```
storage/rdb/
├── internal/
│   └── dao/                    # DAO 实现，按表组织
│       ├── internal/           # 通用 CRUD 封装与 gendry 适配
│       ├── table_*.go          # 每张表对应一个 DAO 文件
│       └── ...
├── ai_cache/                   # AI 缓存规则 / 语义缓存设置 Storage
├── ai_context/                 # 上下文压缩规则 / 全局设置 Storage
├── ai_route/                   # AI 路由规则 Storage
├── api_key/                    # API-Key / Token Storage
├── auth/                       # 认证/授权 Storage
├── basic/                      # 产品线 / BFE 集群 / 附加文件 Storage
├── cluster_conf/               # 集群 / 子集群 / 实例池 / LB 矩阵 / ModelPrice Storage
├── entity/                     # Entity / EntityType / Entity ID 序号分配 Storage
├── iintent_config/             # 意图配置单例 Storage
├── ioperlog/                   # 操作日志 Storage
├── k8s_pool/                   # K8s 实例池 Storage
├── model_price/                # 模型定价 Storage
├── protocol/                   # TLS 证书 Storage
├── provider/                   # Provider Storage
├── quota/                      # QuotaPlan Storage
├── rate_limit_policy/          # RateLimitPolicy Storage
├── route_conf/                 # 域名 / 产品级路由规则 Storage（AI 网关模式下不用于 Cluster 选择）
├── route_rules/                # API-Key / Entity / Global 路由规则 Storage
├── traffic_mirror/             # 流量镜像规则 Storage
├── txn/                        # 事务抽象实现
└── version_control/            # 配置版本控制 Storage
```

DAO 文件全部集中在 `storage/rdb/internal/dao/`，按数据库表一一对应；Storage 文件按业务域组织在 `storage/rdb/` 下的各子包中，与 `model/*` 各子包一一对应。这种组织方式使得新增业务域时，只需在 `model/<domain>` 定义接口、在 `storage/rdb/<domain>` 实现 Storage、在 `storage/rdb/internal/dao/` 增加对应表的 DAO 文件即可，扩展路径清晰。

## DAO + Storage 两层结构

### DAO 层职责

DAO（Data Access Object）层位于 `storage/rdb/internal/dao/`，是距离数据库最近的代码层。每个 DAO 文件对应一张表，提供该表的增删改查函数。DAO 不处理业务逻辑，只负责：

- 将 Go 结构体映射为 SQL 参数。
- 调用 `internal` 包中的通用 CRUD 函数。
- 使用 `db:"column_name"` 标签完成字段映射。

### Storage 层职责

Storage 层位于 `storage/rdb/<domain>/`，面向 `model/*` 各子包暴露接口。Storage 负责：

- 实现 `model/<domain>` 中定义的 Storager 接口。
- 将 model 层的 Param / Filter 转换为 DAO 的 Param。
- 处理 JSON 序列化、分页计算、时间戳填充。
- 在需要时协调同域内多张表的读写。

### 两层协作流程

下图展示了一次创建请求的数据流：

```
+-------------+      +------------------+      +------------------+
| model 层    | ---> | Storage 层       | ---> | DAO 层           |
| Manager     |      | EntityTypeStorager|      | table_entity_types|
+-------------+      +------------------+      +------------------+
                            |                           |
                            v                           v
                     业务模型转 DAO 模型            gendry 构建 SQL
                            |                           |
                            +-----------> DB <----------+
```

模型层的 Manager 持有 Storage 接口；Storage 通过 `lib.DBContextFactory` 获取数据库上下文；DAO 使用同一个 `DBContexter` 执行 SQL。由于 `DBContexter` 既能承载普通连接也能承载事务连接，因此 Storage 和 DAO 无需关心当前是否处于事务中，只需按统一方式执行 SQL，事务传播由上下文工厂负责。

## gendry SQL 构建器使用方式

项目使用 `github.com/didi/gendry` 的 `builder` 包构建 SQL，使用 `scanner` 包扫描结果。`go.mod` 中声明的依赖版本为 `v1.7.0`：

```go
// ai-gateway-api/go.mod
github.com/didi/gendry v1.7.0
```

`storage/rdb/internal/dao/internal/builder.go` 对 gendry 做了一层薄封装，提供 `SelectBuilder`、`InsertBuilder`、`UpdateBuilder`、`DeleteBuilder`，并统一处理 `db` 标签。

`storage/rdb/internal/dao/internal/curd.go` 中的 `QueryOne`、`QueryList`、`Create`、`Update`、`Delete` 是 DAO 层的最终执行入口：

```go
// storage/rdb/internal/dao/internal/curd.go
func QueryOne(dbCtx lib.DBContexter, table string, where interface{}, rst interface{}) error
func QueryList(dbCtx lib.DBContexter, table string, where interface{}, rst interface{}) error
func Create(dbCtx lib.DBContexter, table string, data ...interface{}) (int64, error)
func Update(dbCtx lib.DBContexter, table string, where interface{}, data interface{}) (int64, error)
func Delete(dbCtx lib.DBContexter, table string, where interface{}) (int64, error)
```

以 `QueryList` 为例，它通过反射将 `where` 结构体转换为 `map[string]interface{}`，再交给 gendry 生成 SQL：

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
    // ... SQL 记录与错误处理
    return scanner.Scan(rows, rst)
}
```

`Struct2Where` 在 `builder.go` 中实现，它读取结构体字段的 `db` 标签，自动忽略 nil 指针和空切片，并将带操作符的标签解析为 `column op` 形式。例如 `db:"id,>"` 会生成 `id > ?`。这种设计让查询条件可以自由组合：调用方只需要在 `T<Table>Param` 中填充需要的字段，未填充的字段不会出现在 WHERE 子句中，从而避免了手写 SQL 时拼接条件的繁琐和出错风险。

gendry 的 `scanner` 包则负责将 `sql.Rows` 扫描到结构体或切片中。`curd.go` 在 `init` 函数中通过 `scanner.SetTagName(tagName)` 将扫描标签设置为 `db`，保证 DAO 的 `T<Table>` 结构体既能用于参数构建，也能用于结果扫描，实现一套标签两用。

## DAO 通用模板

每个 DAO 文件遵循统一的五要素模板：

1. **表名常量**：`const t<Table>TableName = "<table_name>"`。
2. **结果结构体**：`T<Table>`，字段使用 `db:"column_name"` 标签，与表字段一一对应。
3. **参数结构体**：`T<Table>Param`，字段均为指针类型，用于查询条件和赋值；同时包含两个控制字段：
   - `OrderBy *string `db:"_orderby"``：排序子句。
   - `Limit []uint `db:"_limit"``：分页参数，格式为 `{offset, pageSize}`。
4. **CRUD 函数**：
   - `T<Table>One(dbCtx, where)`：查询单条，不存在返回 `(nil, nil)`。
   - `T<Table>List(dbCtx, where)`：查询多条，不存在返回 `(nil, nil)`。
   - `T<Table>Create(dbCtx, data...)`：单条或批量插入，自动填充 `CreatedAt`。
   - `T<Table>Update(dbCtx, val, where)`：按条件更新。
   - `T<Table>Delete(dbCtx, where)`：按条件删除。

以下以 `entity_types` 表为例：

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

部分表在通用模板基础上扩展了特殊能力：

| 文件 | 特殊能力 | 说明 |
|------|----------|------|
| `table_products.go` | `TProductDeleteByProductID` | 原生 SQL 级联删除产品线相关多张表 |
| `table_route_advance_rules.go` | `ModeForUpdate`、`_lockMode` | 支持 `SELECT ... FOR UPDATE` 行锁 |
| `table_pools.go` | `PoolsList2Map` | 列表转 map |
| `table_clusters.go` | `IDs []int64`、`Names []string` | 支持 `IN` 查询 |
| `table_sub_clusters.go` | 多字段 `IN` 查询 | 支持 `IDs`、`Names`、`ClusterIDs`、`PoolsIDs` |
| `table_operation_logs.go` | `TOperationLogCount` | 独立 count 查询，构造 SQL 前删除 `_limit`/`_orderby`，避免分页参数污染 `COUNT(*)` |
| `table_entity_id_seq.go` | `TEntityIDSeqAllocate` | 原子序号分配：MySQL 单条 `INSERT ... ON DUPLICATE KEY UPDATE` + `LAST_INSERT_ID`，SQLite `INSERT OR IGNORE` + `UPDATE`，分配值不复用 |

这些扩展能力通过 `T<Table>Param` 中新增切片字段实现。`Struct2Where` 对切片字段会原样放入 `where map`，gendry 的 `builder` 包会自动将其转换为 `IN` 子句。对于 `SELECT ... FOR UPDATE`，则在 `T<Table>Param` 中增加 `_lockMode` 字段，由 DAO 在调用 `internal.QueryList` 后追加锁提示。需要强调的是，这些特殊能力仍然是 DAO 层的细节，Storage 层在使用时只需按业务语义设置参数即可。

## Storage 实现如何向上暴露接口

每个 Storage 实现遵循四个约定：

1. **依赖注入**：通过构造函数接收 `lib.DBContextFactory`。
2. **接口断言**：使用 `var _ <Interface> = &<Storage>{}` 编译期校验。
3. **类型转换**：在 model 参数与 DAO 参数之间转换。
4. **JSON 序列化**：复杂结构序列化为字符串后存入数据库。

以 `entity/entity_type.go` 为例：

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

对应的接口定义在 `model/entity/entity_type.go`：

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

Storage 层通过 `dbCtxFactory(ctx)` 获取 `lib.DBContexter`。如果调用链已处于事务中，工厂会复用同一个事务；否则使用普通连接。这一机制让 Storage 层天然支持事务传播：模型层 Manager 调用 `itxn.TxnStorager.AtomExecute` 开启事务后，事务上下文会被注入到 `context.Context` 中，后续所有 Storage 调用都会拿到同一个事务连接，确保跨 Storage 的原子性。

JSON 序列化是 Storage 层的常见工作。以 `rate_limit_policy/rate_limit_policy.go` 为例，`tpm_configs` 和 `rpm_configs` 在数据库中以 JSON 字符串存储：

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

## 34 张表的 Storage 映射关系

根据 `ai-gateway-api/design-docs/sys-design/数据库设计文档.md`，当前系统共 34 张持久化表，按业务模块划分如下。

### 基础配置（6 张表）

| 表名 | DAO 文件 | Storage 包 | 说明 |
|------|----------|------------|------|
| `bfe_clusters` | `table_bfe_clusters.go` | `storage/rdb/basic/bfe_cluster.go` | BFE 集群配置 |
| `products` | `table_products.go` | `storage/rdb/basic/product.go` | 产品线配置 |
| `domains` | `table_domains.go` | `storage/rdb/route_conf/domain.go` | 域名配置 |
| `users` | `table_users.go` | `storage/rdb/auth/authentication.go` | 用户 / Token |
| `user_products` | `table_user_products.go` | `storage/rdb/auth/authorization.go` | 用户/Token-产品授权 |
| `config_versions` | `table_config_versions.go` | `storage/rdb/version_control/version_control.go` | 配置版本 |

### Provider 与集群实例池（5 张表）

| 表名 | DAO 文件 | Storage 包 | 说明 |
|------|----------|------------|------|
| `providers` | `table_providers.go` | `storage/rdb/provider/provider.go` | Provider 接入能力 |
| `clusters` | `table_clusters.go` | `storage/rdb/cluster_conf/cluster.go` | AI 集群配置 |
| `sub_clusters` | `table_sub_clusters.go` | `storage/rdb/cluster_conf/sub_cluster.go` | 子集群 |
| `pools` | `table_pools.go` | `storage/rdb/cluster_conf/pool.go` | 实例池 |
| `lb_matrices` | `table_lb_matrix.go` | `storage/rdb/cluster_conf/cluster.go` | 负载均衡矩阵 |

### 路由规则（6 张表）

| 表名 | DAO 文件 | Storage 包 | 说明 |
|------|----------|------------|------|
| `route_basic_rules` | `table_route_basic_rules.go` | `storage/rdb/route_conf/route_rule.go` | 基础路由规则 |
| `route_advance_rules` | `table_route_advance_rules.go` | `storage/rdb/route_conf/route_rule.go` | 高级路由规则 |
| `route_default_rules` | `table_route_default_rules.go` | `storage/rdb/route_conf/route_rule.go` | 默认转发规则 |
| `route_cases` | — | — | 路由测试用例，当前无 DAO |
| `ai_route_rules` | `table_ai_route_rules.go` | `storage/rdb/ai_route/ai_route.go` | AI 路由规则 |
| `route_rules` | `table_route_rules.go` | `storage/rdb/route_rules/route_rules.go` | API-Key / Entity / Global 路由规则 |

### 证书与附加文件（2 张表）

| 表名 | DAO 文件 | Storage 包 | 说明 |
|------|----------|------------|------|
| `certificates` | `table_certificates.go` | `storage/rdb/protocol/certificate.go` | TLS 证书 |
| `extra_files` | `table_extra_files.go` | `storage/rdb/basic/extra_file.go` | 附加文件 |

### API-Key / Entity / 配额 / 限流（6 张表）

| 表名 | DAO 文件 | Storage 包 | 说明 |
|------|----------|------------|------|
| `api_keys` | `table_api_keys.go` | `storage/rdb/api_key/api_key.go` | API-Key |
| `api_key_tokens` | `table_api_key_tokens.go` | `storage/rdb/api_key/api_key.go` | API-Key Token 记录 |
| `entity_types` | `table_entity_types.go` | `storage/rdb/entity/entity_type.go` | Entity 类型 |
| `entities` | `table_entities.go` | `storage/rdb/entity/entity.go` | Entity 实体 |
| `quota_plans` | `table_quota_plans.go` | `storage/rdb/quota/quota_plan.go` | 配额计划 |
| `rate_limit_policies` | `table_rate_limit_policies.go` | `storage/rdb/rate_limit_policy/rate_limit_policy.go` | 限流策略 |

### 模型定价（1 张表）

| 表名 | DAO 文件 | Storage 包 | 说明 |
|------|----------|------------|------|
| `model_prices` | `table_model_prices.go` | `storage/rdb/model_price/model_price.go` | 模型定价 |

### AI 增强策略与 K8s 实例池（7 张表）

| 表名 | DAO 文件 | Storage 包 | 说明 |
|------|----------|------------|------|
| `ai_cache_rules` | `table_ai_cache_rules.go` | `storage/rdb/ai_cache/` | AI 缓存规则集合 |
| `ai_cache_semantic_settings` | `table_ai_cache_semantic_settings.go` | `storage/rdb/ai_cache/` | 语义缓存全局设置单例 |
| `ai_context_rules` | `table_ai_context_rules.go` | `storage/rdb/ai_context/` | 上下文压缩规则集合 |
| `ai_context_settings` | `table_ai_context_settings.go` | `storage/rdb/ai_context/` | 上下文压缩全局设置单例 |
| `traffic_mirror_rules` | `table_traffic_mirror_rules.go` | `storage/rdb/traffic_mirror/` | 流量镜像规则集合 |
| `intent_config` | `table_intent_config.go` | `storage/rdb/iintent_config/` | 意图配置单例 |
| `k8s_pools` | `table_k8s_pools.go` | `storage/rdb/k8s_pool/` | K8s 实例池，写入方唯一为 K8s 发现组件 |

其中 `ai_cache_rules`、`ai_context_rules`、`traffic_mirror_rules` 为全量替换集合表（数组序即优先级），`intent_config`、`ai_cache_semantic_settings`、`ai_context_settings` 为单例行覆盖表，`k8s_pools` 的实例快照写入后不扇出，消费方按 `k8s_pool_name` 实时解析有效池（见 [第十章 Provider 与 Cluster 设计](../design/chapter10-provider-and-cluster.md)）。

### 操作日志与序号分配（2 张表）

| 表名 | DAO 文件 | Storage 包 | 说明 |
|------|----------|------------|------|
| `operation_logs` | `table_operation_logs.go` | `storage/rdb/ioperlog/operation_log.go` | 配置操作日志 |
| `entity_id_seq` | `table_entity_id_seq.go` | `storage/rdb/entity/id_generator.go` | Entity ID 序号分配 |

`operation_logs` 表记录管理面所有配置写操作，是操作日志模块（`model/ioperlog`，详见 [第六章 控制面核心设计](../design/chapter06-control-plane-design.md)）的持久化载体。表的要点：

- `log_id` 与 access log 的 `LogID` 一致，用于关联与去重；
- `change_summary` 为 `mediumtext` JSON，记录脱敏后的变更前后快照与 `diff_keys`；
- 建有 `idx_operator`、`idx_resource`、`idx_action`、`idx_created_at`、`idx_log_id`、`idx_resource_parent` 六个索引，覆盖查询接口的全部过滤维度。

`entity_id_seq` 是 Entity ID 的序号分配表，仅有一行固定记录（`name='entity'`），`next_seq` 保存下一个可用序号。Entity ID 由此前的时间戳/随机数方案改为序列表分配（生成形如 `entity-{seq}` 的业务 ID），`TEntityIDSeqAllocate` 通过原子语句分配序号且已分配值不复用。

注意：`route_cases` 表在 DDL 中定义，但当前代码中暂无对应 DAO 与 Storage 实现，因此实际由 DAO + Storage 覆盖的表为 33 张。

### DDL 与升级说明

项目不提供增量 migration 脚本，`db_ddl.sql`（SQLite 对应 `db_ddl_sqlite.sql`）为全量建表脚本。从旧版本升级时，需要手工在现有库中执行新增表的建表语句（如 `operation_logs`、`entity_id_seq`），应用层不会自动补表。

新增可选字段通过加列方式演进，例如 `providers` 表在 v0.0.10 增加 `protocol_paths` TEXT 列（JSON，存"协议 → 上游 base path"映射），存量数据为 NULL 即未配置，无需数据迁移，应用层不读即忽略。

`providers` 表的 K8s 实例供给能力以加列方式承载：`instance_source` VARCHAR(32) 列默认 `instance_pool`（取值 `instance_pool` / `k8s_pool`），`k8s_pool_name` VARCHAR(255) 列可空，存量数据为 NULL 即缺省手工实例池模式，无需数据迁移。当前 DDL 不含 `k8s_instance_pool` 镜像列；若数据库曾应用过 v0.0.10 → v0.0.11 增量预览脚本，需手工 `ALTER TABLE providers DROP COLUMN k8s_instance_pool;` 清理。应用层经 `InstancePoolResolver` 按实例来源实时查询 `k8s_pools` 表读取有效池（见[第十章 Provider 与 Cluster 设计](../design/chapter10-provider-and-cluster.md)）。

从映射关系可以看出，Storage 子包的划分依据是业务域而非数据库表数量。例如 `cluster_conf` 子包同时管理 `clusters`、`sub_clusters`、`pools`、`lb_matrices` 四张表，因为这几张表共同服务于集群配置这一业务概念；`route_conf` 子包同时管理 `domains`、`route_basic_rules`、`route_advance_rules`、`route_default_rules`，因为它们共同组成产品级路由规则（AI 网关模式下不用于 AI 请求的 Cluster 选择，仅用于产品线识别上下文或非 AI 流量场景）。这种按业务域聚合的方式，让 Storage 接口更贴近模型层 Manager 的调用需求，避免了 Manager 同时依赖多个细粒度 Storage 的复杂局面。

## 报表存储实现（独立报表库）

报表数据存放在独立于控制面库的报表库中（MySQL `bfe_report`、Doris `bfe_observability` 或 ClickHouse / StarRocks 中同名库），不在上述控制面表之列，DDL 由 ai-gateway-api 仓库单独发布（`db_ddl_report_mysql.sql`，schema-first）。查询侧实现为 `storage/mysqlreport`、`storage/dorisreport`、`storage/clickhousereport`、`storage/starrocksreport` 四个 Storage 包（DAO 惯例与 gendry 构建器同控制面一致），SQL 方言差异封在各自包内。

两张表的设计要点：

- **明细表 `bfe_ai_request_log`**（99 列，与 Doris 同名同列）：唯一键 `(hostid, log_time, ai_apikey_id, ai_requested_model)` 与 Doris UNIQUE KEY 一致，log-reader 以 `INSERT ... ON DUPLICATE KEY UPDATE` 幂等覆盖写入；`ARRAY<STRUCT>` 类字段在 MySQL 侧为 JSON 列（仅明细展示用，不做过滤条件）；长文本列使用 TEXT 以规避 utf8mb4 行长度上限；按天 RANGE 分区滚动。
- **聚合表 `bfe_ai_metrics_1m`**（40 维 + 24 指标，维度集合与 Doris / ClickHouse / StarRocks 聚合表一致）：不设唯一键——维度列过多无法构成 InnoDB 唯一键，且幂等性由聚合 JOB 的事务语义保证。

MySQL 形态的两个后台 JOB（`storage/mysqlreport/job.go`，Doris 形态不启动）：

- **分钟聚合 JOB**：`DELETE` 上一整分钟窗口 + `INSERT SELECT` 单事务写入，窗口重放幂等；维度列写入时 `IFNULL(col,'')` 归一（与 Doris JOB 的 COALESCE 语义对齐）；多副本部署用 MySQL `GET_LOCK('report_agg_job', 0)` 抢锁防重；进程重启不补历史窗口（接受 ≤1 分钟空洞，与 Doris INSERT JOB 语义对称）。
- **分区管理 JOB**：每 6h 巡检，向前预建 3 天分区、DROP 超出 `RetentionDays` 的过期分区，非分区形态自动降级 `DELETE ... LIMIT` 分批清理。MySQL 的 `information_schema.PARTITIONS` 对表达式边界回显求值后的整数（如 `TO_DAYS('2026-09-18')` 回显 `738886`），解析需同时支持整数反解、字面文本与 `MAXVALUE` 三种形态，否则会把现有分区误判为缺失而重复 `ADD PARTITION`（Error 1493），或让初始分区永远不参与过期 DROP。

另一个跨方言的口径要点是时间渲染：目标是"时区无关的 Unix 秒"。MySQL 与 Doris（含 StarRocks）实现统一用 `TIMESTAMPDIFF(SECOND, '1970-01-01 00:00:00', col)` 算术差（两条 DATETIME 的整数秒差，不做时区解读），避免 `UNIX_TIMESTAMP()` 按会话时区解读墙钟导致的偏移——log-reader 按 UTC 墙钟写入，两侧口径一致；ClickHouse 实现则直接用 `toUnixTimestamp` / `fromUnixTimestamp` / `intDiv` 原生函数处理 DateTime 列（列固定 UTC，无时区歧义）。

## 事务实现（storage/rdb/txn）

模型层不直接操作数据库事务，而是通过 `model/itxn.TxnStorager` 接口：

```go
// model/itxn/txn.go
type TxnStorager interface {
    AtomExecute(ctx context.Context, do func(context.Context) error) error
}
```

`storage/rdb/txn/txn.go` 提供基于关系型数据库的实现：

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

`lib.OpenTxn()` 指示工厂开启事务；`lib.RDBTxnExecute` 负责在回调结束后根据错误决定提交或回滚。

跨表操作通常由模型层 Manager 在 `AtomExecute` 回调内编排。例如 `EntityManager.CreateEntity` 会依次创建 QuotaPlan、RateLimitPolicy、RouteRules、Entity；`RouteRuleStorager.UpsertProductRule` 会先加锁、再删除、再批量插入。Storage 层本身一般不开启事务，只负责单表或同域内几张表的读写转换。

这种分层让事务边界非常清晰：只有模型层 Manager 知道哪些操作需要原子性，因此由它决定何时开启事务；Storage 和 DAO 只负责在已提供的上下文中执行 SQL。测试时也可以方便地替换 `itxn.TxnStorager` 为内存实现，从而隔离数据库依赖。

## 无物理外键的设计考量

根据数据库设计文档，所有表的 DDL **未声明物理 FOREIGN KEY 约束**，表间关系均为逻辑外键，由应用层代码维护。主要考量包括：

- **性能**：MySQL 的外键检查会增加写操作开销，高并发场景下容易成为瓶颈。
- **灵活性**：便于分库分表、数据迁移和灰度发布；物理外键会限制 schema 变更自由度。
- **批量删除**：如 `TProductDeleteByProductID` 使用原生 SQL 一次性删除产品线关联的多张表数据，避免级联触发器的不可控行为。
- **引用关系清晰化**：`api_keys` 中的 `quota_plan_id`、`rate_limit_policy_id`、`route_rules_id` 等字段虽然不做物理约束，但通过索引加速查询，并在模型层做存在性校验。

无物理外键也意味着应用层必须自行保证一致性。项目中常见的做法是：在模型层 Manager 中通过 `itxn.TxnStorager` 编排多表操作，并在关键路径（如删除产品、删除 API-Key）上显式清理关联数据。例如在删除产品时，`basic/product.go` 对应的 Manager 会调用 `TProductDeleteByProductID` 一次性清理 `products`、`domains`、`route_basic_rules`、`route_advance_rules`、`route_default_rules` 等表，避免残留数据导致路由异常。

需要指出的是，不使用物理外键并不代表放弃数据完整性。项目通过三种手段弥补：第一，DAO 中为关联字段建立索引，加速校验与查询；第二，模型层在写入前做显式存在性检查；第三，关键删除路径通过事务保证多表同步清理。这种设计更适合网关控制面这种读多写少、需要频繁变更 schema 的场景。

## 关键代码片段

### 1. DAO 通用模板：`table_entity_types.go`

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

### 2. Storage 接口实现：`entity_type.go`

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

### 3. 产品级路由规则全量替换：`route_rule.go`

> 说明：以下 Storage 函数负责产品级路由规则（`route_basic_rules` / `route_advance_rules` / `route_default_rules`）的持久化。在 AI 网关模式下，这些规则不用于 AI 请求的 Cluster 选择，仅用于产品线识别上下文或非 AI 流量场景。

```go
// storage/rdb/route_conf/route_rule.go
func (rs *RouteRuleStorager) UpsertProductRule(
    ctx context.Context,
    product *ibasic.Product,
    rule *iroute_conf.ProductRouteRule,
) error {
    // ... 构造 daoBasicRules、daoAdvanceRules

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

该函数展示了 Storage 层如何协调同域多张表：先 `SELECT ... FOR UPDATE` 加锁，再删除旧规则，最后批量插入新规则。事务边界由调用方通过 `itxn.TxnStorager` 保证。

### 4. 路由规则列表内存分页：`route_rules.go`

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
    // ... 转换为 RouteTableParam
    return result, total, nil
}
```

这里先查全量记录计算 `total`，再查分页记录。返回结果只包含 `type`、`owner`、`enabled` 等元信息，不返回可能很大的 `rules` 字段。这种设计牺牲了一定量的数据库查询次数，换取了接口响应大小的可控性，并且避免了在 gendry 构建器中同时生成 `COUNT(*)` 与 `LIMIT` 的复杂逻辑。对于路由规则这种单条记录可能包含大量 JSON 数据的表，内存分页是一种简单有效的折中方案。

## 本章小结

本章详细介绍了壬远 AI 网关控制面的存储层实现：

- `storage/rdb` 采用 **DAO + Storage** 两层结构：DAO 贴近数据库表，提供通用 CRUD；Storage 面向业务域，实现 `model/*` 中定义的接口。
- DAO 层基于 `github.com/didi/gendry` 构建 SQL，通用 CRUD 封装在 `storage/rdb/internal/dao/internal/curd.go` 中。
- 每个 DAO 文件遵循统一模板：表名常量、`T<Table>` 结果结构体、`T<Table>Param` 参数结构体、CRUD 函数。
- Storage 通过 `lib.DBContextFactory` 获取数据库上下文，负责模型转换、JSON 序列化、分页计算和时间戳填充。
- 34 张表按业务模块映射到不同的 Storage 子包，`route_cases` 当前无 DAO/Storage 实现。
- 事务通过 `model/itxn.TxnStorager` 抽象，`storage/rdb/txn/txn.go` 提供基于 RDB 的实现，模型层 Manager 负责编排跨表事务边界。
- 数据库设计不使用物理外键，由应用层通过逻辑外键和事务保证一致性，兼顾性能与灵活性。

## 参考文档

- `ai-gateway-api/design-docs/sys-design/存储层设计文档.md`
- `ai-gateway-api/design-docs/sys-design/数据库设计文档.md`
- `ai-gateway-api/storage/rdb/internal/dao/`
- `ai-gateway-api/storage/rdb/txn/`
- `ai-gateway-api/db_ddl.sql`
- `ai-gateway-api/db_ddl_sqlite.sql`
- `ai-gateway-api/go.mod`
