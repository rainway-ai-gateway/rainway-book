# 第五章 壬远AI网关架构与核心概念

## 本章目标

通过本章，读者将理解：
- 壬远AI网关如何解决企业接入大模型服务时的核心问题；
- 壬远AI网关的整体架构，以及控制面与数据面的职责划分；
- 各核心组件（AI Gateway API、Dashboard、BFE、Conf Agent、ai-gateway-controller（原名 Service Controller）、Log Reader）如何协同工作；
- 一条配置从创建到生效的完整生命周期；
- 典型的部署拓扑形态。

---

## 为什么需要AI网关

企业在规模化使用大模型服务时，通常会面临以下几类问题：

**多模型提供商的协议差异**

OpenAI、DeepSeek、Anthropic、Google Gemini 等模型服务商的 API 协议、认证方式、错误码、流式响应格式各不相同。如果让每个业务系统直接对接不同提供商，会导致大量重复适配工作。

**API Key 分散管理的风险**

不同业务团队可能各自申请和管理模型服务商的 API Key，难以统一审计、轮换和回收。一旦某个 Key 泄露，影响范围难以控制。

**成本与配额控制困难**

大模型调用通常按 Token 或按次计费。如果没有统一的配额和限流机制，容易出现某一线程或应用耗尽预算、影响其他业务的情况。

**高可用与故障转移需求**

单一模型提供商可能出现延迟升高、服务不可用或配额耗尽。业务需要能够在多个提供商、多个模型之间自动切换。

**安全审计与合规要求**

企业需要知道谁在调用模型、调用了什么模型、消耗了多少 Token、是否涉及敏感数据。这些都需要统一的接入层来记录和管控。

壬远AI网关正是为了解决上述问题而设计的统一接入层。它在业务系统与底层模型服务之间建立了一个受控的转发平面，将协议适配、认证授权、路由调度、配额限流、成本核算、日志审计等能力集中管理。

---

## 系统总体架构

壬远AI网关采用**控制面（Control Plane）与数据面（Data Plane）分离**的架构。控制面负责策略和配置的创建、存储与分发；数据面负责实际请求转发和策略执行。两者通过配置导出与拉取机制协同工作。

```
┌─────────────────────────────────────────────────────────────────┐
│                        管理使用者                                │
│         Dashboard 管理员 / 自动化脚本 / CI-CD 系统               │
└───────────────────────┬─────────────────────────────────────────┘
                        │ HTTP / OpenAPI
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                         控制面                                   │
│  ┌──────────────┐  ┌─────────────────┐  ┌──────────────┐       │
│  │  Dashboard   │  │  AI Gateway API │  │   MySQL      │       │
│  │  (Web UI)    │◄─┤  (Open/Inner)   │◄─┤   Redis      │       │
│  └──────────────┘  └────────┬────────┘  └──────────────┘       │
└─────────────────────────────┼───────────────────────────────────┘
                              │ InnerAPI (HTTP)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         数据面                                   │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐   │
│  │ Conf Agent  │──►│     BFE     │──►│  模型服务提供商      │   │
│  │(配置拉取/热加载)│   │(流量转发)    │   │(OpenAI/DeepSeek/...)│   │
│  └─────────────┘   └─────────────┘   └─────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

这种架构的优势在于：

- **配置变更不影响转发稳定性**：控制面升级、重启或出现故障时，数据面可以继续基于本地缓存配置进行转发。
- **数据面无状态化**：BFE 和 Conf Agent 不保存持久化业务数据，便于水平扩展。
- **配置分发可靠**：Conf Agent 通过版本号比对实现增量同步，降低网络开销。

---

## 核心概念

在使用壬远AI网关之前，需要先理解以下关键概念。它们贯穿控制台操作、API 调用、配置导出与数据面转发各个环节。

| 概念 | 英文 | 一句话说明 |
|---|---|---|
| BFE 集群 | BFECluster | 登记数据面 BFE 引擎地址，控制面据此知道配置要下发给谁（Server Data，由部署初始化数据维护，无 OpenAPI 管理接口） |
| 模型服务商 | Provider | 描述一个模型服务提供方，包括协议类型、后端实例池、模型列表与认证密钥 |
| K8s 实例池 | K8s Pool | 由 ai-gateway-controller 上报到控制面的后端实例池，Provider 可引用池名替代手工维护实例列表 |
| AI 业务集群 | Cluster | 流量实际转发的后端集群，通过引用 Provider 声明其上游能力 |
| 组织 | Entity | 表达部门、团队或项目等组织架构，是配额、限流、模型访问控制与路由规则的挂载点 |
| API 调用凭证 | API-Key | 业务系统调用数据面转发入口时使用的身份凭证，可挂载到 Entity 继承策略 |
| 路由表 | Route Table | 按 Global / Entity / API-Key 三级组织的路由规则，决定请求转发到哪个 Cluster |
| 配额计划 | Quota Plan | 按 RMB 或 Token 为 Entity / API-Key 设置预算上限 |
| 限流策略 | Rate Limit Policy | 按 RPM / TPM / 并发等维度限制请求速率 |

### BFE 集群（BFECluster）

「BFE 集群」对应 BFE 的 Server Data 配置，登记数据面 BFE 的引擎地址清单，用于标识哪些 BFE 节点可以从控制面拉取配置。其记录由部署初始化数据（`db_ddl.sql` 种子数据，默认集群引用内置实例池 `BFE.aipool`）维护，未暴露 OpenAPI 管理接口。它回答的是“配置要下发给谁”的问题，与 Provider 中的后端实例池（`instance_pool`）含义不同：前者是数据面入口，后者是上游模型服务端点。

### 模型服务商（Provider）

「模型服务商」对应 OpenAPI 的 `/providers` 资源，回答“下游是谁、能访问哪些模型、如何认证、后端在哪里”的问题。它持有：

- 后端实例池（`instance_pool`）：真实 AI 服务端点；
- 模型协议（`model_protocols`）：如 `openai`、`anthropic`、`gemini`；
- 模型列表（`models`）与模型发现端点；
- 服务鉴权 Key（`keys`）：用于访问上游模型服务；在数据库中以上游密钥字段级加密（`enc$v1$` 信封）落盘，导出时同样受导出加密保护（见 [第十七章 安全设计](./chapter17-security-design.md)）。

多个 Cluster 可以引用同一个 Provider，实现实例池与密钥的复用。

### K8s 实例池（K8s Pool）

「K8s 实例池」对应 InnerAPI `/inner-api/v1/k8s_pools` 维护的实例池数据，池成员的唯一写入方是部署在 Kubernetes 环境的控制器 ai-gateway-controller。Provider 通过 `instance_source=k8s_pool` 与池名引用 K8s 实例池，实例列表由控制面自动同步刷新，无需手工维护后端地址。

控制器与集群之间的约定可归纳为以下几点：

- **过滤**：只处理带有 `ai-gateway-api-name` 标签的 Service（标签值等于启动参数 `--ai-gateway-api-name`，`*` 表示任意），并受 `--namespace` 的命名空间列表约束。
- **端口与实例**：按 Service 的每个具名端口建立一条池，无名端口跳过；实例取自同名 Endpoints 的 Ready 地址，忽略 `notReadyAddresses`，`(addr, port)` 去重后经 `PUT` 全量替换上报，重复上报幂等。
- **池命名**：`k8s_<namespace>_<service>_<port>[_<cluster>]`；控制器侧超过 64 字符时折叠为前 55 字符加 `_` 与 sha256 前 8 位；控制面对池名的合法上限为 **128** 个字符，字符集为字母、数字与 `_`、`-`、`.`。
- **空池与清理**：无 Ready 地址时上报空数组 `[]`，池保留但零实例；端口改名或移除时按差集删除旧池；Service 删除由 finalizer `k8s.bfenetworks.com/delete-protection` 拦截，先删池再放行；删除仍被 Provider 引用的池时“保留池记录 + 清空实例”（`instance_count=0`），不级联失败。
- **鉴权**：以 `--ai-gateway-api-token` 携带 `Token <identify>` 调用 InnerAPI，Token 需属于 System Scope 并具备 `k8s_pools` 域写权限，否则返回 402。

上报进入控制面后，池成员不做扇出：Provider 不存储实例镜像（无 `k8s_instance_pool` 字段，请求体携带按未知字段静默忽略），由各消费方按 `k8s_pool_name` 实时查询 `k8s_pools` 表解析有效池（`InstancePoolResolver`，池名空/池不存在返回空列表，读取失败 fail-close），K8s 模式 Cluster 的派生池为空占位、不承载实例，最终经 `cluster_table` 导出到 BFE；Provider 视角下「池不存在」与「池存在但零实例」等价。控制面提供只读 OpenAPI `GET /open-api/v1/k8s_pools[/{name}]` 供控制台展示池列表、实例与最近同步时间。详细设计参见 [第十章 Provider 与 Cluster 设计](./chapter10-provider-and-cluster.md)。

### AI 业务集群（Cluster）

「AI 业务集群」对应 OpenAPI 的 `/clusters` 资源，回答“流量如何转发、用哪些模型、Key 权重如何分配”的问题。它通过 `llm_config.provider` 强引用 Provider，并在此之上声明转发模型、Key 权重、超时、健康检查等策略。详细设计动机与数据模型参见 [第十章 Provider 与 Cluster 设计](./chapter10-provider-and-cluster.md)。

### 组织（Entity）与 API-Key

「Entity」用于表达组织架构，例如部门、团队或项目。每个 Entity 拥有独立的模型白名单/黑名单、配额计划（QuotaPlan）、限流策略（RateLimitPolicy）与路由规则。Entity 通过 `parent_id` 字段自底向上形成层级树，其类型由 `entity_types` 表定义，挂载到其上的 API-Key 会继承 Entity 层级的模型策略与配额/限流/路由策略。详细设计参见 [第九章 Entity 与 API-Key 设计](./chapter09-apikey-design.md) 中的“Entity 层级树与模型继承”。

「API-Key」是业务系统调用数据面转发入口时使用的身份凭证。API-Key 可挂载到 Entity 以继承其配额与限流策略，也可拥有独立的 API-Key 级路由规则。详细设计参见 [第八章 认证授权设计](./chapter08-auth-design.md) 与 [第九章 Entity 与 API-Key 设计](./chapter09-apikey-design.md)。

### 路由表

「路由表」对应 Global / Entity / API-Key 三级路由规则。请求进入数据面后按 **API-Key > Entity > Global** 顺序匹配：先查该 Key 的专属表，未命中再查 Key 挂载组织的表，最后回落 Global。路由规则中的 `cond` 为 BFE 条件表达式，`targets` 指定目标集群与权重，`fallbacks` 指定降级目标。详细设计参见 [第十一章 AI 路由规则设计](./chapter11-ai-route-rules.md)。

### 配额与限流

- **配额计划（Quota Plan）**：为 Entity 或 API-Key 设置预算上限，支持按 RMB 或 Token 维度核算成本。详细设计参见 [第十二章 配额与限流设计](./chapter12-quota-and-rate-limit.md)。
- **限流策略（Rate Limit Policy）**：按 RPM / TPM / 并发等维度限制请求速率，防止单点流量冲垮后端或耗尽预算。详细设计参见 [第十二章 配额与限流设计](./chapter12-quota-and-rate-limit.md)。

---

## 核心组件

### AI Gateway API（控制面核心）

AI Gateway API 是壬远AI网关的控制面核心组件，负责：

- 暴露**管理面 OpenAPI**（`/open-api/v1`），供 Dashboard 和管理员脚本调用；
- 暴露**数据面 InnerAPI**（`/inner-api/v1`），供 BFE 和 Conf Agent 拉取配置；
- 完成策略/配置的创建、存储、版本控制和下发；
- 维护 API-Key、Entity、Provider、Cluster、路由规则、配额、限流、证书等核心数据。

AI Gateway API 采用经典的三层架构：

```
接口层 (endpoints)
    ├── openapi_v1/    管理面接口
    ├── innerapi_v1/   数据面导出接口
    └── middleware/    全局中间件

模型层 (model)
    ├── api_key/       API-Key 业务逻辑
    ├── entity/        Entity / Entity-Type
    ├── icluster_conf/ Cluster 管理
    ├── iprovider/     Provider 管理
    ├── quota/         配额计划
    ├── rate_limit_policy/ 限流策略
    └── ...

存储层 (storage/rdb)
    ├── internal/dao/  DAO 层
    └── */             各业务 Storage 实现
```

详细设计见 [第六章 控制面核心设计：AI Gateway API](./chapter06-control-plane-design.md)。

### Dashboard（管理控制台）

Dashboard 是面向运维和管理员的 Web UI，通过调用 OpenAPI 完成可视化操作。它降低了使用门槛，使非开发者也能完成 Provider 接入、Cluster 配置、API-Key 管理、配额设置等操作。

Dashboard 本身不直接操作数据库，所有变更都通过 AI Gateway API 完成，因此与后端控制面保持松耦合。

### BFE（数据面转发引擎）

BFE 是壬远AI网关的数据面，负责实际接收和转发 AI 请求。它基于开源 BFE 项目扩展了多个 AI 专用模块：

- `mod_ai_route`：根据 AI 路由规则选择目标 Cluster 和模型；
- `mod_ai_token_auth`：校验 API-Key 并执行配额检查与扣减；
- `mod_ai_rate_limit`：执行 TPM/RPM/并发限流；
- `mod_body_process`：解析流式响应中的 Token 用量。

BFE 以插件化方式处理请求，每个模块在请求生命周期的特定阶段执行。详细设计见 [第七章 数据面转发设计：BFE](./chapter07-data-plane-design.md)。

### Conf Agent（配置代理）

Conf Agent 是部署在 BFE 侧的 Sidecar 组件，主要职责包括：

1. 定期轮询 AI Gateway API 的 InnerAPI；
2. 将最新配置持久化到本地磁盘，并按版本管理；
3. 检测到配置变更后，调用 BFE 的热加载接口完成生效；
4. 清理过期版本，避免磁盘无限增长。

Conf Agent 的存在使得控制面无需直接连接数据面，配置下发是**拉取式**的，天然适合跨网络区域的部署场景。

### ai-gateway-controller（服务发现）

ai-gateway-controller 是部署在 Kubernetes 集群内的控制器（基于 controller-runtime），负责把服务发现结果写入控制面：监听带 `ai-gateway-api-name` 标签的 Service 及其同名 Endpoints，按每个具名端口把 Ready 实例经 InnerAPI `PUT /inner-api/v1/k8s_pools/{name}/instances` 全量上报为 K8s 实例池。池成员不扇出到 Provider 只读镜像（无 `k8s_instance_pool` 字段），由控制面在导出与查询时按 `k8s_pool_name` 实时解析，再随 `cluster_table` 导出给 BFE（K8s 模式 Cluster 派生池为空占位）。控制器不直接连接 BFE，也不修改 Provider 或 Cluster 配置；引用池的 Provider 由运维通过 OpenAPI 声明。实现与部署细节见 [第三十七章 ai-gateway-controller实现](../implementation/chapter37-controller-implementation.md)。

### Log Reader（数据面日志采集）

Log Reader 是数据面日志采集组件（仓库 `rainway-ai-gateway/log-reader`），部署在 BFE 侧，负责读取 BFE 产生的访问日志（`bfe-access-pb` 协议），将注册过的字段解析为结构化的 Token 用量、延迟、状态码等记录，并按字段注册结果写入 MySQL（`mod_log_mysql` 插件，101 列，含 `ai_price_miss` / `ai_price_warn` 两列）或 Kafka（`mod_kafka` 插件，JSON 形态），供报表统计与可观测分析消费。详细设计见 [第十五章 可观测性设计](./chapter15-observability.md)。

---

## 控制面与数据面的职责划分

| 职责 | 控制面（AI Gateway API） | 数据面（BFE） |
|------|--------------------------|---------------|
| 配置创建与修改 | ✅ | ❌ |
| 配置持久化 | ✅（MySQL） | ❌ |
| 配置版本控制 | ✅ | ❌ |
| 配置导出 | ✅（InnerAPI） | ❌ |
| 请求转发 | ❌ | ✅ |
| API-Key 校验 | ❌ | ✅ |
| 配额扣减 | ❌ | ✅ |
| 限流执行 | ❌ | ✅ |
| 日志输出 | ❌ | ✅ |
| 监控指标 | 有限的自身监控 | ✅ 丰富的请求级指标 |

这种划分遵循"控制面负责决策，数据面负责执行"的原则。所有策略定义在控制面完成，数据面只负责高效、可靠地执行策略。

---

## 配置生命周期

壬远AI网关的一条配置从创建到最终生效，通常经历以下阶段：

```
管理员/脚本
     │
     ▼
Dashboard / OpenAPI
     │
     ▼
AI Gateway API（校验、存储、生成新版本）
     │
     ▼
MySQL + Redis（持久化与实时配额）
     │
     ▼
InnerAPI（按主题导出配置）
     │
     ▼
Conf Agent（轮询、比对版本号）
     │
     ▼
本地配置文件 + Symlink 切换
     │
     ▼
BFE /reload（热加载）
     │
     ▼
请求转发（按新策略执行）
```

**阶段说明**：

1. **配置录入**：管理员通过 Dashboard 或 OpenAPI 提交配置变更，例如新增一个 API-Key 或修改路由规则。
2. **校验与存储**：AI Gateway API 对参数进行校验，写入 MySQL，并更新配置版本号。
3. **配置导出**：InnerAPI 将数据库中的配置按主题（如 `mod-api-key`、`ai-route`、`rate-limit-policy` 等）导出为 BFE 可消费的 JSON/TOML 文件。
4. **配置拉取**：Conf Agent 定期调用 InnerAPI，比较本地版本号与远程版本号。
5. **本地切换**：当发现新版本时，Conf Agent 将新配置写入新的版本目录，并通过 Symlink 原子切换到新目录。
6. **热加载**：Conf Agent 调用 BFE 的 `/reload/{module}` 接口，BFE 在不重启的情况下加载新配置。
7. **生效执行**：后续 AI 请求按照新的策略进行路由、认证、配额和限流。

详细配置导出机制见 [第十四章 配置导出与版本控制设计](./chapter14-config-export-and-version-control.md)。

---

## 部署拓扑

### 最小化部署

适用于开发测试或业务量较小的场景：

```
┌─────────────────┐
│  AI Gateway API │
│  Dashboard      │
│  MySQL / Redis  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Conf Agent     │
│  BFE            │
└────────┬────────┘
         │
         ▼
   模型服务提供商
```

所有组件可以运行在同一台机器或同一个容器中，便于快速体验。

### 生产部署

生产环境通常采用分离部署：

```
                    ┌─────────────┐
                    │   Admin     │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  Dashboard    │  │ AI Gateway API│  │   MySQL       │
│  (Nginx 前端)  │  │  (多实例)      │  │   Redis       │
└───────────────┘  └───────────────┘  └───────────────┘
                           │
                           │ InnerAPI
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ Conf Agent    │  │ Conf Agent    │  │ Conf Agent    │
│ BFE           │  │ BFE           │  │ BFE           │
│ (可用区 A)     │  │ (可用区 B)     │  │ (可用区 C)     │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘
        └──────────────────┼──────────────────┘
                           ▼
                    模型服务提供商
```

生产部署要点：

- AI Gateway API 多实例部署，前面通过负载均衡器访问；
- MySQL 和 Redis 使用高可用方案；
- BFE 在多个可用区部署，Conf Agent 独立拉取配置；
- Dashboard 可独立部署，也可与 AI Gateway API 同容器运行。

详细部署步骤见 [第十八章 安装部署](../operation/chapter18-installation-and-deployment.md)。

---

## 本章小结

- 壬远AI网关作为统一接入层，解决了多模型提供商接入、API-Key管理、成本与配额控制、高可用、安全审计等问题。
- 系统采用控制面与数据面分离架构，控制面负责策略管理，数据面负责请求转发与策略执行。
- 核心组件包括 AI Gateway API、Dashboard、BFE、Conf Agent、ai-gateway-controller 和 Log Reader。
- 配置生命周期涵盖创建、存储、导出、拉取、热加载和生效六个阶段。
- 支持从最小化部署到多可用区生产部署的多种拓扑。

---

## 参考文档

- `ai-gateway-api/README.md`
- `ai-gateway-api/design-docs/sys-design/总体设计文档.md`
- `ai-gateway-api/design-docs/sys-design/details/InnerAPI配置导出与版本控制.md`
- `bfe/AGENTS.md`
- `conf-agent/AGENTS.md`
