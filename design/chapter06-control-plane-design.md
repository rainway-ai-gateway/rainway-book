# 第六章 控制面核心设计：AI Gateway API

## 本章目标

通过本章，读者将理解：

- AI Gateway API 在壬远AI网关中的定位与职责边界；
- 控制面如何以三层架构（接口层、模型层、存储层）组织代码；
- 管理面 OpenAPI 与数据面 InnerAPI 的职责划分与路由组织方式；
- `xreq.Endpoint` 统一抽象如何简化接口注册、鉴权与中间件处理；
- 全局容器（`stateful/container`）与手动依赖注入的实现方式；
- 规则类资源的两类资源形态（全量替换集合与单例）及其鉴权项；
- 操作日志模块的写入机制、脱敏规则与审计价值；
- 资源依赖冲突的 409 Conflict 错误约定；
- 从 `main.go` 到 HTTP 服务启动的完整流程。

---

## AI Gateway API 的定位与职责

**AI Gateway API** 是壬远AI网关的控制面（Control Plane）核心组件，负责对外暴露管理面接口与数据面导出接口，完成策略/配置的创建、存储、版本控制和下发。它向上为 Dashboard 和管理员脚本提供可编程的管理接口，向下为 BFE 数据面和 Conf Agent 生成可消费的配置快照。

在壬远AI网关整体架构中，AI Gateway API 与周边组件的关系如下：

| 组件 | 角色 | 与 AI Gateway API 的交互 |
|------|------|--------------------------|
| Dashboard | 管理控制台 | 调用 OpenAPI 完成可视化操作 |
| BFE | 数据面 | 消费 InnerAPI 导出的配置并执行转发 |
| Conf Agent | 配置代理 | 轮询 InnerAPI，拉取最新配置并触发 BFE 热加载 |
| ai-gateway-controller（原名 Service Controller） | 服务发现 | 把 K8s Service/Endpoints 的 Ready 实例上报为 K8s 实例池（InnerAPI `k8s_pools`），控制面按 `k8s_pool_name` 实时解析有效池，最终经 `cluster_table` 导出到 BFE |

AI Gateway API 当前的功能范围覆盖：API-Key / Entity / Entity-Type 管理、Provider 与 Cluster 管理、模型定价管理、配额计划与限流策略管理、AI 路由规则管理、AI 缓存规则、流量镜像规则、意图配置、上下文压缩规则、K8s 实例池管理、证书与附加文件管理、认证授权、配置操作日志审计以及面向数据面的配置导出。

### 控制面与数据面的边界

控制面负责“决策”，数据面负责“执行”。两者的职责边界可概括如下：

| 职责 | 控制面（AI Gateway API） | 数据面（BFE） |
|------|--------------------------|---------------|
| 配置创建与修改 | ✅ | ❌ |
| 配置持久化 | ✅（MySQL / SQLite） | ❌ |
| 配置版本控制 | ✅ | ❌ |
| 配置导出 | ✅（InnerAPI） | ❌ |
| 请求转发 | ❌ | ✅ |
| API-Key 校验与配额扣减 | ❌ | ✅ |
| 限流执行 | ❌ | ✅ |

这种划分使得控制面可以独立升级、重启，而数据面继续基于本地缓存配置转发流量，保证转发稳定性。

---

## 三层架构

AI Gateway API 采用经典的三层架构，将 HTTP 处理、业务逻辑与数据持久化解耦：

```
┌─────────────────────────────────────────────────────────────┐
│                         接口层 (endpoints)                    │
│   OpenAPI v1 (/open-api/v1)  ·  InnerAPI v1 (/inner-api/v1)   │
├─────────────────────────────────────────────────────────────┤
│                         模型层 (model)                        │
│        业务逻辑 · 事务编排 · 参数校验 · 级联操作                │
├─────────────────────────────────────────────────────────────┤
│                      存储层 (storage/rdb)                     │
│            DAO · Storage 实现 · 关系型数据库读写               │
└─────────────────────────────────────────────────────────────┘
```

### 接口层

接口层位于 `endpoints/`，是 HTTP 请求的入口。其主要职责包括：

- 路由注册与分发；
- 请求参数绑定与基础校验；
- 权限与身份认证；
- 调用模型层 Manager 完成业务处理；
- 统一响应格式渲染。

关键包如下：

| 包 | 职责 |
|----|------|
| `endpoints/router.go` | 根路由注册，挂载全局 Recovery、Logger、CORS 中间件 |
| `endpoints/openapi_v1/` | 管理面 OpenAPI 路由组织，各业务子包在此合并 |
| `endpoints/innerapi_v1/` | 数据面 InnerAPI 导出接口注册 |
| `endpoints/middleware/` | Recovery、Logger、CORS、Product Probe、User Probe |
| `lib/xreq` | 统一的 `Endpoint` 抽象与参数绑定工具 |

### 模型层

模型层位于 `model/`，采用 **Manager + Storager 接口** 的分层模式：

- **Manager**：封装业务逻辑、事务编排、参数校验与级联操作；
- **Storager 接口**：定义持久化操作契约，由存储层实现；
- **Param / Filter**：分别用于写入入参与查询条件，字段多为指针类型以区分“未传”与“零值”。

关键包如下：

| 包 | 职责 |
|----|------|
| `model/api_key/` | API-Key 业务逻辑、配额/限流/路由级联、实时配额查询 |
| `model/entity/` | Entity / Entity-Type 业务逻辑、层级校验与级联删除 |
| `model/icluster_conf/` | Cluster、SubCluster、Pool 业务逻辑与 `AIConf` 导出 |
| `model/iprovider/` | Provider 接入能力管理、模型发现、被 Cluster 引用检查 |
| `model/imodel_price/` | 模型定价导入、CRUD 与价格查询 |
| `model/quota/` | QuotaPlan、BalanceSync、QuotaResetScheduler |
| `model/rate_limit_policy/` | RateLimitPolicy 业务逻辑与导出 |
| `model/route_rules/` | Global / Entity / API-Key 三级 AI 路由规则 |
| `model/ioperlog/` | 操作日志 Manager、敏感字段脱敏、变更摘要 diff_keys 计算 |
| `model/imods/` | mod-api-key、mod-body-process、AI 路由等模块配置导出 |
| `model/ai_cache/` | AI 缓存规则与语义缓存全局设置的业务逻辑与导出 |
| `model/ai_context/` | 上下文压缩规则与全局设置的业务逻辑与导出 |
| `model/traffic_mirror/` | 流量镜像规则的业务逻辑与导出 |
| `model/iintent_config/` | 意图配置单例的业务逻辑与导出 |
| `model/ik8s_pool/` | K8s 实例池维护（写入不扇出，消费时实时解析） |
| `model/itxn/` | 事务抽象接口 `TxnStorager` |
| `model/shared/` | 跨包共享类型与通用 Storager 接口 |

### 存储层

存储层位于 `storage/rdb/`，采用 **DAO + Storage** 两层结构：

- `storage/rdb/internal/dao/` 按表提供 `T<Table>One/List/Create/Update/Delete` 函数，直接操作 MySQL / SQLite；
- `storage/rdb/*` 各子包实现模型层定义的 `XxxStorager` 接口，完成 DAO 结果与业务模型的转换；
- `storage/rdb/txn/` 提供 `model/itxn.TxnStorager` 的事务实现。

关键包如下：

| 包 | 对应模型 | 覆盖表 |
|----|----------|--------|
| `storage/rdb/auth/` | `model/iauth` | `users`、`user_products` |
| `storage/rdb/basic/` | `model/ibasic` | `bfe_clusters`、`products`、`extra_files` |
| `storage/rdb/api_key/` | `model/api_key` | `api_keys`、`api_key_tokens` |
| `storage/rdb/cluster_conf/` | `model/icluster_conf` | `clusters`、`sub_clusters`、`pools`、`lb_matrices` |
| `storage/rdb/entity/` | `model/entity` | `entity_types`、`entities` |
| `storage/rdb/protocol/` | `model/iprotocol` | `certificates` |
| `storage/rdb/quota/` | `model/quota` | `quota_plans` |
| `storage/rdb/rate_limit_policy/` | `model/rate_limit_policy` | `rate_limit_policies` |
| `storage/rdb/route_conf/` | `model/iroute_conf` | `domains`、`route_*_rules` |
| `storage/rdb/route_rules/` | `model/shared`、`model/route_rules` | `route_rules` |
| `storage/rdb/ioperlog/` | `model/ioperlog` | `operation_logs` |
| `storage/rdb/provider/` | `model/iprovider` | `providers` |
| `storage/rdb/ai_cache/` | `model/ai_cache` | `ai_cache_rules`、`ai_cache_semantic_settings` |
| `storage/rdb/ai_context/` | `model/ai_context` | `ai_context_rules`、`ai_context_settings` |
| `storage/rdb/traffic_mirror/` | `model/traffic_mirror` | `traffic_mirror_rules` |
| `storage/rdb/iintent_config/` | `model/iintent_config` | `intent_config` |
| `storage/rdb/k8s_pool/` | `model/ik8s_pool` | `k8s_pools` |

### 层间交互关系

一条 HTTP 请求在三层层间的流转路径如下：

```
                         HTTP Request
                              │
           ┌──────────────────┴──────────────────┐
           ▼                  ▼                  ▼
    /open-api/v1      /inner-api/v1        全局中间件
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

核心约束：

- 接口层只调用模型层 Manager，不直接访问存储层；
- 模型层只依赖同包或 `model/shared` 中定义的 `XxxStorager` 接口以及 `itxn.TxnStorager`；
- 存储层只负责数据库读写，通过 `lib.DBContextFactory` 从上下文中获取事务或普通连接。

### 设计原则与约定

三层架构遵循以下设计原则：

1. **分层解耦**：接口层只依赖模型层 Manager；模型层只依赖 Storager 接口和事务抽象；存储层只负责数据库读写。
2. **接口导向**：模型层通过 `XxxStorager` 接口定义持久化契约，便于未来替换存储实现。
3. **统一 Endpoint 抽象**：所有 HTTP 接口使用 `xreq.Endpoint` 描述，统一注册、鉴权和中间件处理。
4. **子包自治**：每个业务子包独立维护自己的 Endpoint、Action、Manager、Storager；顶层 `endpoints.go` 仅做合并。
5. **全局容器**：Manager 与 Storager 作为全局单例存在 `stateful/container`，通过 `rdb.Init()` 完成一次性初始化。
6. **版本控制**：配置变更统一由 `iversion_control` 管理版本，支持 InnerAPI 增量导出。
7. **无物理外键**：表间关系由应用层维护，DDL 不声明 FOREIGN KEY，便于分库分表与灵活变更。
8. **事务编排**：跨表/跨 Storager 的业务操作通过 `TxnStorager.AtomExecute` 保证原子性。

---

## OpenAPI v1 与 InnerAPI v1 的职责划分

AI Gateway API 将对外接口划分为两大接口族：

| 接口族 | 前缀 | 用途 | 中间件 | 调用方 |
|--------|------|------|--------|--------|
| **OpenAPI v1** | `/open-api/v1` | 管理面接口 | `McProductProbe` + `McUserProbe` | Dashboard、管理员脚本 |
| **InnerAPI v1** | `/inner-api/v1` | 数据面导出接口 | `McUserProbe` | BFE、Conf Agent |

### OpenAPI v1 主要模块

OpenAPI v1 负责暴露可管理资源，典型模块包括：

| 子包 | 路径 | 功能 |
|------|------|------|
| `api_key` | `/api-keys` | API-Key 管理 |
| `entity` | `/entities` | Entity 管理 |
| `entity_type` | `/entity-types` | Entity-Type 管理 |
| `provider` | `/providers` | Provider 管理、模型发现 |
| `product_cluster` | `/clusters` | Cluster 管理 |
| `model_price` | `/model-prices` | 模型定价管理 |
| `global_route_rules` | `/global-route-rules` | Global 路由表 |
| `route_tables` | `/route-tables` | 路由表列表 |
| `certificate` | `/certificates` | 证书管理 |
| `auth` | `/auth`、`/meta` | 用户、Session Key、Token |
| `operation_log` | `/operation-logs` | 配置操作日志查询 |
| `ai_cache` | `/ai-cache-rules`、`/ai-cache-semantic-settings` | AI 缓存规则（全量替换集合）与语义缓存设置（单例） |
| `ai_context` | `/ai-context-rules`、`/ai-context-settings` | 上下文压缩规则（全量替换集合）与全局设置（单例） |
| `traffic_mirror` | `/traffic-mirror-rules` | 流量镜像规则（全量替换集合） |
| `intent_config` | `/intent-config` | 意图配置（单例） |

### InnerAPI v1 主要导出接口

InnerAPI v1 将控制面持久化的配置按主题导出，供数据面消费：

| 路径 | 功能 |
|------|------|
| `/configs/tls_conf/server_data_conf` | 导出 TLS/Server/路由规则配置 |
| `/configs/gslb_data/gslb` | 导出 GSLB 调度配置 |
| `/configs/gslb_data/cluster_table` | 导出集群表配置 |
| `/configs/protocol/server_cert_conf` | 导出证书配置 |
| `/configs/extra_files/{filename}` | 导出附加文件 |
| `/configs/mod-api-key` | 导出 API-Key 及配额配置 |
| `/configs/mod-body-process` | 导出请求体处理配置 |
| `/configs/rate-limit-policy` | 导出限流策略配置 |
| `/configs/ai-route` | 导出 AI 路由配置 |
| `/configs/ai-cache-rule` | 导出 AI 缓存规则配置 |
| `/configs/ai-context-rule` | 导出上下文压缩规则配置 |
| `/configs/traffic-mirror-rule` | 导出流量镜像规则配置 |
| `/configs/mod-ai-intent` | 导出意图配置 |
| `/quota/trigger-reset` | 手动触发一次配额周期重置（详见配额章节） |

所有 InnerAPI 导出接口均支持 `version` 查询参数，通过 `model/iversion_control` 实现增量同步：当请求版本与当前版本一致时返回 `Data: nil`，避免重复下发。

### 中间件执行顺序

全局中间件在 `endpoints/router.go` 的 `RegisterRouters` 中按顺序注册：

```
router.Use(middleware.MCRecovery)
router.Use(middleware.MCLogger)
router.Use(middleware.MCCors)
```

在此基础上，OpenAPI 路由子树额外挂载 `McProductProbe` 与 `McUserProbe`，InnerAPI 路由子树仅挂载 `McUserProbe`。`McUserProbe` 会调用每个 Endpoint 上配置的 `Authorizer`，根据 Feature + Action 判断是否放行。

| 中间件 | 作用 |
|--------|------|
| `MCRecovery` | 捕获 panic，返回统一错误响应 |
| `MCLogger` | 记录请求/响应日志 |
| `MCCors` | 处理 CORS 预检和响应头 |
| `McProductProbe` | 从请求头解析产品线上下文 |
| `McUserProbe` | 从 Session Key 或 Token 解析用户身份，完成权限校验 |

---

## 资源形态：全量替换集合与单例

OpenAPI 资源并非只有“按 id 寻址的 CRUD”一种形态。面向 BFE 模块的规则类配置衍生出两类新资源形态，对应不同的接口面与写入语义。

### 全量替换集合资源

`ai_cache_rules`、`traffic_mirror_rules`、`ai_context_rules` 属于全量替换集合资源：

- 接口只暴露集合整体的 `GET` / `PUT`，没有 `/{id}` 子资源；
- 一次 `PUT` 在单事务内完成 delete-all + insert-all 整体替换，`rules: null` 视为清空；
- 规则按 first-match-wins 语义消费，数组顺序即优先级，元素在数组中的位置是配置语义的一部分；
- 集合元素的数据库自增 id 不对外暴露，API 与导出均按数组序组织。

### 单例资源

`intent_config`、`ai_cache_semantic_settings`、`ai_context_settings` 属于单例资源：

- 全系统固定一行（固定 id），写入即单行覆盖，无集合概念；
- 版本号 `yyyyMMddHHmmss` 由控制面内部生成并内嵌在导出内容中，不通过 API 暴露；
- 空表语义各异：`intent_config` 从未发布时 `GET` 返回 404（空问题集即停用分类的软开关语义）；两个 settings 空表表示“未覆盖默认值”，`GET` 返回文档默认值，永不 404。

两类形态仍沿用 `xreq.Endpoint` 注册与 `iauth` 鉴权框架，只是 Manager 内部的写入路径从“按 id 增量修改”变为“整体替换 / 单行覆盖”。鉴权项对应 `model/iauth/features.go` 中的 `FeatureAICache`、`FeatureAIContext`、`FeatureTrafficMirror`、`FeatureAIIntent` 与 `FeatureK8sPool`：前四者分别对应四类 AI 增强策略资源，后者对应 K8s 实例池维护端点（写入方唯一为 K8s 发现组件，见[第十章 Provider 与 Cluster 设计](./chapter10-provider-and-cluster.md)）。

---

## 操作日志模块与 409 错误约定

### 操作日志模块定位

操作日志模块记录管理面（OpenAPI）的所有配置写操作，为配置变更提供可追溯的审计能力。模块位于 `model/ioperlog/`，核心设计如下：

- **Manager 设计**：`OperationLogManager` 实现 `OperationLogRecorder`（`Record`）与 `OperationLogManagerInterface`（含 `QueryLogs`、`Close`）两级接口。各业务 Manager 通过 `SetOperationLogManager` 注入 recorder，仅依赖最小接口 `OperationLogRecorder`，避免对查询能力的反向依赖；
- **存储实现**：`storage/rdb/ioperlog/operation_log.go` 实现 `BatchCreate`（批量插入）与 `List`（条件查询 + 分页），DAO 层对应 `storage/rdb/internal/dao/table_operation_logs.go`；
- **查询接口**：`endpoints/openapi_v1/operation_log/list.go` 暴露 `GET /open-api/v1/operation-logs`，鉴权为 `iauth.FeatureOperationLog + ActionReadAll`；`ScopeSystem`（系统管理员）与 `ScopeSystemReadonly`（系统管理员·只读）均具备该读权限。

### 异步缓冲写入机制

`OperationLogManager` 采用“异步缓冲 + 批量落库”的写入路径，避免日志写放大影响管理接口延迟：

```
业务 Manager ──Record()──▶ buffered channel (默认 4096 条)
                                │
                                ▼
                    batchWorker：满 200 条 或 5s 超时即 flush
                                │ BatchCreate
                                ▼
                    失败重试 3 次（100ms/200ms/300ms 退避）
```

关键参数与行为（`model/ioperlog/manager.go`）：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| 缓冲容量 | 4096 | `NewOperationLogManager(storager, bufferSize)`，`bufferSize <= 0` 时使用默认值 |
| 批量大小 | 200 | 累积满即 flush |
| Flush 间隔 | 5s | 定时强制 flush |
| 重试次数 | 3 | 重试间隔按 100ms 递增 |
| 溢出策略 | 同步回退 | 缓冲满时默认 `OverflowStrategySync` 阻塞调用方同步写入；可选 `OverflowStrategyDiscard` 丢弃并告警 |

`Close` 时会在 500ms  drain 超时内排空缓冲，尽量保证退出前日志落库。

### 日志内容与关联

`OperationLogEntry` 的字段分为四类：操作者、资源、结果、请求上下文。其中两点设计值得说明：

1. **与访问日志关联**：`log_id` 与 access log 的 `LogID` 一致。上下文提取器 `operationLogContextExtractor`（`stateful/container/rdb/components.go`）从 `xreq.GetRequestInfo(ctx)` 取出 `LogID`、请求路径、方法、ClientIP、UserAgent 填入日志，使操作日志可与访问日志互相检索、并按 `log_id` 去重；
2. **操作者提取**：从 `iauth.MustGetVisitor(ctx)` 提取，`operator_type` 为 `0`（user，取 `User.ID`）或 `1`（token，取 `Token.ID`）。

失败操作同样记录：`status=2`，`error_msg` 经 `TruncateErrorMessageDefault` 截断为 1024 字符（超长时以 `...` 结尾），与 DDL 中 `error_msg varchar(1024)` 对齐。

### 变更摘要与脱敏

`change_summary`（`mediumtext`，存 JSON）记录变更前后的关键字段快照，由 `model/ioperlog/change_summary.go` 的 `BuildChangeSummary` 构造：

- `before` / `after`：变更前后的字段 map，均先经 `MaskSensitiveFields` 脱敏；
- `diff_keys`：仅 update 类日志携带，表示 after 相对 before **新增或值发生变化的键**（before 独有的键不计入，避免部分更新请求产生假阳性 diff；未提交的字段即 nil 值键，在快照构造时经 `ParamToMap` 丢弃），键名排序后存储。

cluster 资源的审计快照是一个例外：它不使用 `ParamToMap` 的默认映射，而是手写为与 `GET /clusters` 一致的 API 小写词汇（如 `hash_strategy` 落字符串枚举、`epp_config` 落解码 JSON 对象、`balance_mode` 空值规范为 `WRR`），并裁剪 `id` / `ready` / `product_id` / `scheduler` / `sub_clusters` 等 API 未暴露的内部字段，保证 `change_summary` 与 API 语义一致。

脱敏规则（`model/ioperlog/mask.go`）按小写键名匹配，并对嵌套 map 递归处理：

| 键名 | 脱敏结果 |
|------|----------|
| `password` / `secret` / `session_key` / `private_key` / `token` / `access_token` / `refresh_token` / `session_token` / `id_token` / `secret_key` / `client_secret` / `access_key`（及无下划线变体） | `******` |
| `api_key` / `apikey` / `key` | 保留首尾 4 位，中间以 `****` 代替；长度不超过 8 时整体脱敏 |
| `certificate` / `cert` / `cert_body` / `private_key_body` | `[已更新]` |

`error_msg` 在源头构造审计条目时即做值级脱敏：对 API-Key 等敏感明文做全文替换，消息中出现的 Key 明文被替换为保留首尾 4 位的掩码形式（`ioperlog.MaskErrorMessage`，`model/api_key/api_key_operation_log.go` 调用），避免错误回显泄露凭证。

### 覆盖范围

实际产生操作日志的资源与动作如下：

| 资源 | 动作 |
|------|------|
| entities / entity-types | create / update / delete |
| api-keys | create / update / delete |
| clusters（POST/PUT/DELETE） | create / update / delete |
| routes、global-route-rules（PUT） | update；Global 路由表更新记录为 `resource_type=route, resource_id=global` |
| certificates | create / update / delete |
| quota-plans | create / update / delete / reset（配额余额重置） |
| model-prices | create / update / delete / import（整表导入） |
| auth/users（含 session reset） | create / update / delete / reset |
| auth/tokens | create / delete |

quota_plan 与 rate_limit_policy 没有独立的 OpenAPI 资源，但会随 Entity / API-Key 的嵌套创建、更新、删除产生审计日志：`resource_type` 分别为 `quota_plan` / `rate_limit_policy`，`resource_parent_id` 记录所属 Entity 或 API-Key。domain 当前不产生操作日志。

### 查询接口

`GET /open-api/v1/operation-logs` 支持如下查询参数（`xreq.BindForm` 绑定）：

| 参数 | 说明 |
|------|------|
| `operator_name` | 操作人名称 |
| `action` / `resource_type` / `resource_id` / `resource_name` / `resource_parent_id` | 资源维度过滤 |
| `status` | `1`=成功，`2`=失败 |
| `start_time` / `end_time` | Unix 秒时间戳 |
| `page` / `page_size` | 默认 1 / 20，page_size 上限 100 |

响应为 `{list, pagination:{page, page_size, total}}`；日志条目字段含 `id, log_id, operator_type, operator_id, operator_name, action, resource_type, resource_id, resource_name, resource_parent_id, status, error_msg, change_summary, request_path, request_method, client_ip, user_agent, created_at`（`created_at` 为 Unix 秒）。

查询路径上的另一个细节：DAO 层 `TOperationLogCount` 在构造 count SQL 前会删除 where map 中的 `_limit` 与 `_orderby`，避免分页参数污染 `COUNT(*)`，保证任意页码下 `total` 都为符合条件的总条数。

### 409 Conflict 错误约定

`lib/xerror` 中 `WrapConflictErrorWithMsg` 将错误标记为 `Model.Conflict`，由 `resolve.go` 统一映射为 HTTP **409 Conflict**，用于表达“资源被其他资源引用、操作存在依赖冲突”的业务语义：

- BFE 集群被 AI 集群的 `lb_matrix` 调度引用时禁止删除（`endpoints/openapi_v1/bfe_cluster/delete.go` 调用 `ClusterManager.IsBFEClusterUsed` 检查）；
- 实例池被 BFECluster / SubCluster 引用时禁止删除；
- 证书被 Product 引用时禁止删除；
- 产品集群被路由规则（advance / basic）引用时禁止删除；
- 集群被 AI 路由规则的 target / fallback 引用时禁止删除；集群模型更新时被规则 target / fallback 引用时报错；
- Entity 存在子节点时禁止删除。

409 用于依赖冲突，500 作为未知错误的兜底；422 保留给参数错误（`etParam`），三者语义互不混用。

---

## 统一的 xreq.Endpoint 抽象

接口层采用统一的 `xreq.Endpoint` 抽象描述每个 HTTP 接口：

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

每个业务子包导出自己的 Endpoint 变量切片，最终在 `endpoints/openapi_v1/endpoints.go` 或 `endpoints/innerapi_v1/endpoints.go` 中统一合并注册到 `gorilla/mux` 路由器。这种设计带来了三点好处：

1. **接口自描述**：路径、方法、处理器、鉴权函数集中在一处；
2. **统一注册**：顶层 `endpoints.go` 只负责合并，不处理业务逻辑；
3. **鉴权一致**：OpenAPI 与 InnerAPI 均通过 `Authorizer` 注入 `iauth.FA` 或 `iauth.FAP` 完成权限校验。

以 Entity-Type 创建接口为例：

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

该示例同时展示了接口层的典型流程：参数绑定 → 基础校验 → 业务存在性检查 → 调用模型层 → 返回结果。

---

## 全局容器与依赖注入

AI Gateway API 使用**全局容器 + 手动依赖注入**模式。所有 Manager 与 Storager 单例声明在 `stateful/container/components.go`，初始化逻辑集中在 `stateful/container/rdb/components.go:Init()`。

容器中的部分关键单例如下：

```go
// stateful/container/components.go（节选）
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

`rdb.Init()` 按以下顺序完成初始化：

1. 事务与基础 Storage：`TxnStoragerSingleton`、各基础/集群/认证 Storage；
2. 操作日志：`OperationLogStorager` 与 `OperationLogManager`（含上下文提取器 `operationLogContextExtractor`），先行初始化以便注入后续各业务 Manager；
2. 基础 Manager：`ExtraFileManager`、`VersionControlManager`、`BFEClusterManager`、`CertificateManager`、`ProductManager`、`AIRouteRuleManager`、`RouteRuleManager` 等；
3. 集群相关 Manager：`ClusterManager`、`SubClusterManager`、`DomainManager`、`PoolManager`；
4. 认证授权 Manager：`AuthenticateManager`、`AuthorizeManager`；
5. Entity / RateLimit / Quota 相关 Storage 与 Manager；
6. 模块导出 Manager：`APIKeyRuleManager`、`ModBodyProcessManager`、`AIRouteExporter`；
7. API-Key Manager：注入 Quota/RateLimit/Entity 适配器，实现与配额/限流体系的打通；
8. 周期任务：`BalanceSyncManager`、`QuotaResetScheduler`，并启动调度器；
9. 默认 Global 路由表：调用 `RouteRulesManager.EnsureGlobalRouteRules`，确保 `route_rules` 表中存在 `type=global, owner=global` 的默认记录。

例如：

```go
// stateful/container/rdb/components.go（节选）
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
    // ... 其它依赖
)
```

为了降低跨包依赖，`model/quota` 还提供适配器，将 `entity.EntityStorager`、`rate_limit_policy.RateLimitPolicyStorager`、`quota.QuotaPlanStorager` 适配为 `model/shared` 中定义的通用接口，供 `APIKeyManager` / `EntityManager` 统一调用。

```go
quota.NewQuotaPlanStoragerAdapter(quotaPlanStorager)
quota.NewRateLimitPolicyStoragerAdapter(rateLimitPolicyStorager)
quota.NewEntityStoragerAdapter(entityStorager)
```

---

## 启动流程

AI Gateway API 的启动流程从 `main.go` 开始，经过配置加载、数据库初始化、依赖注入、路由注册，最终启动 HTTP 服务：

```
main.go
  │
  ├─ flag.Parse()                 # 解析命令行参数
  ├─ stateful.LoadConfig()        # 加载 TOML 配置文件
  ├─ config.Init()                # 初始化路径变量
  ├─ config.InitLog()             # 初始化日志
  ├─ config.Depends.Init()        # 初始化 i18n、nav_tree 等依赖
  ├─ config.InitDB()              # 初始化数据库连接池
  ├─ rdb.Init()                   # 依赖注入：初始化 Manager/Storager
  │
  └─ serverStartUp()
       ├─ NewMonitorServerWithRun()  # 启动监控端口
       ├─ endpoints.RegisterRouters() # 注册 OpenAPI/InnerAPI 路由
       └─ graceful.Run()             # 启动 HTTP 服务
```

默认配置文件 `conf/ai_gateway_api.toml` 包含服务端口、日志、数据库、依赖路径和运行时开关：

```toml
# conf/ai_gateway_api.toml（节选）
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

路由注册入口 `endpoints/router.go` 负责挂载全局中间件，并将 OpenAPI 与 InnerAPI 分别注册到不同前缀下：

```go
// endpoints/router.go（示意）
func RegisterRouters(router *mux.Router) {
    router.Use(middleware.MCRecovery)
    router.Use(middleware.MCLogger)
    router.Use(middleware.MCCors)

    openapi_v1.RegisterEndpoints(router)
    innerapi_v1.RegisterRouter(router)
}
```

`graceful.Run` 在启动 HTTP 服务的同时，注册优雅关闭逻辑：当收到退出信号后，等待正在处理的请求完成或超过 `GracefulTimeOutInMs` 设定的超时时间，再关闭监听。

---

## 配置示例与代码片段

### Cluster 转发策略示例

Provider 与 Cluster 分离后，Cluster 的 `llm_config` 只保留转发策略，后端能力通过 `provider` 引用 Provider 获取：

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

### 模型层事务编排示例

模型层通过 `itxn.TxnStorager.AtomExecute` 编排跨多个 Storager 的原子操作。以下片段展示了 API-Key 创建时级联创建 QuotaPlan、RateLimitPolicy 与 RouteRules 的典型写法：

```go
// model/api_key/api_key.go（示意）
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

### InnerAPI 增量导出示例

InnerAPI 导出接口通过 `export_util.NewExportFromReq` 解析 `version` 参数，并交给对应 Manager 的 `ConfigExport` 方法处理：

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

当本地版本与远程一致时，`ConfigExport` 返回空数据，Conf Agent 不会触发 BFE 热加载，从而降低网络与控制面压力。

---

## 本章小结

- AI Gateway API 是壬远AI网关的控制面核心，承担管理面 OpenAPI 与数据面 InnerAPI 的双重职责。
- 系统采用接口层、模型层、存储层三层架构，各层之间通过清晰的接口边界解耦。
- OpenAPI 面向管理员和 Dashboard，负责资源配置；InnerAPI 面向 BFE 和 Conf Agent，负责配置导出与增量同步。
- `xreq.Endpoint` 统一了接口的描述、注册与鉴权，降低了接口开发的样板代码。
- `stateful/container` 提供全局单例容器，`stateful/container/rdb/components.go:Init()` 按依赖顺序完成手动依赖注入。
- 操作日志模块（`model/ioperlog`）以异步缓冲 + 批量落库的方式记录管理面写操作，`log_id` 与访问日志关联，敏感字段脱敏后存入 `operation_logs` 表。
- 资源依赖冲突统一返回 409 Conflict（`xerror.WrapConflictErrorWithMsg`），500 回归为兜底未知错误。
- 启动流程从 `main.go` 的配置加载开始，经过数据库初始化、依赖注入、路由注册，最终通过 `graceful.Run` 启动 HTTP 服务。

---

## 参考文档

- `ai-gateway-api/design-docs/sys-design/总体设计文档.md`
- `ai-gateway-api/design-docs/sys-design/接口层设计文档.md`
- `ai-gateway-api/design-docs/sys-design/模型层设计文档.md`
- `ai-gateway-api/design-docs/sys-design/存储层设计文档.md`
- `ai-gateway-api/design-docs/sys-design/details/操作日志模块.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/operation-logs.md`
- `ai-gateway-api/AGENTS.md`
- `ai-gateway-api/conf/ai_gateway_api.toml`
