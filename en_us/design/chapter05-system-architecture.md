# Chapter 5: Rainway AI Gateway Architecture and Core Concepts

## Chapter Goals

Through this chapter, readers will understand:
- How the Rainway AI Gateway addresses the core problems enterprises face when integrating LLM services at scale;
- The overall architecture of the Rainway AI Gateway and the division of responsibilities between the Control Plane and the Data Plane;
- How the core components (AI Gateway API, Dashboard, BFE, Conf Agent, Service Controller) work together;
- The full lifecycle of a configuration from creation to taking effect;
- Typical deployment topologies.

---

## Why an AI Gateway Is Needed

When enterprises use LLM services at scale, they typically face the following categories of problems:

**Protocol differences across model providers**

Model providers such as OpenAI, DeepSeek, Anthropic, and Google Gemini each have different API protocols, authentication methods, error codes, and streaming response formats. Having every business system integrate directly with each provider leads to a large amount of duplicated adaptation work.

**Risks of scattered API Key management**

Different business teams may each apply for and manage their own API Keys for model providers, making unified auditing, rotation, and revocation difficult. Once a Key is leaked, the blast radius is hard to control.

**Difficult cost and quota control**

LLM calls are usually billed per Token or per request. Without a unified quota and rate limiting mechanism, it is easy for a single thread or application to exhaust the budget and affect other businesses.

**High availability and failover requirements**

A single model provider may experience elevated latency, service outages, or quota exhaustion. Businesses need the ability to switch automatically among multiple providers and multiple models.

**Security auditing and compliance requirements**

Enterprises need to know who is calling which model, how many Tokens were consumed, and whether sensitive data is involved. All of this requires a unified access layer for recording and control.

The Rainway AI Gateway is a unified access layer designed to solve the problems above. It establishes a controlled forwarding plane between business systems and the underlying model services, centralizing protocol adaptation, authentication and authorization, routing and scheduling, quota and rate limiting, cost accounting, and log auditing.

---

## Overall System Architecture

The Rainway AI Gateway adopts an architecture with a **separated Control Plane and Data Plane**. The Control Plane is responsible for creating, storing, and distributing policies and configurations; the Data Plane is responsible for actual request forwarding and policy enforcement. The two cooperate through a configuration export and pull mechanism.

```
+-----------------------------------------------------------------+
|                     Management Users                            |
|         Dashboard admins / automation scripts / CI-CD           |
+--------------------------------+-------------------------------+
                                 | HTTP / OpenAPI
                                 v
+-----------------------------------------------------------------+
|                        Control Plane                            |
|  +--------------+  +-----------------+  +--------------+        |
|  |  Dashboard   |  | AI Gateway API  |  |   MySQL      |        |
|  |  (Web UI)    |<-+  (Open/Inner)   |<-+   Redis      |        |
|  +--------------+  +--------+--------+  +--------------+        |
+--------------------------------+-------------------------------+
                                 | InnerAPI (HTTP)
                                 v
+-----------------------------------------------------------------+
|                         Data Plane                              |
|  +-------------+   +-------------+   +---------------------+    |
|  | Conf Agent  |-->|     BFE     |-->|  Model Providers     |    |
|  |(pull/hot    |   |(forwarding) |   |(OpenAI/DeepSeek/...) |    |
|  | reload)     |   +-------------+   +---------------------+    |
|  +-------------+                                                |
+-----------------------------------------------------------------+
```

The advantages of this architecture:

- **Configuration changes do not affect forwarding stability**: when the Control Plane is upgraded, restarted, or fails, the Data Plane can continue forwarding based on locally cached configurations.
- **Stateless Data Plane**: BFE and Conf Agent do not persist business data, which makes horizontal scaling easy.
- **Reliable configuration distribution**: Conf Agent achieves incremental synchronization through version comparison, reducing network overhead.

---

## Core Concepts

Before using the Rainway AI Gateway, you need to understand the following key concepts. They run through every stage: Dashboard operations, API calls, configuration export, and Data Plane forwarding.

| Concept | English | One-line Description |
|---|---|---|
| AI Gateway instance pool | Server Data | Registers Data Plane BFE engine addresses; the Control Plane uses it to know who to deliver configurations to |
| Model provider | Provider | Describes a model service provider, including protocol type, backend instance pool, model list, and authentication keys |
| AI business cluster | Cluster | The backend cluster to which traffic is actually forwarded; declares its upstream capability by referencing a Provider |
| Organization | Entity | Represents organizational structures such as departments, teams, or projects; the mount point for quota, rate limit, model access control, and routing rules |
| API credential | API-Key | The credential used by business systems when calling the Data Plane forwarding entry; can be attached to an Entity to inherit policies |
| Route table | Route Table | Routing rules organized at three levels — Global / Entity / API-Key — determining which Cluster a request is forwarded to |
| Quota plan | Quota Plan | Sets a budget cap for an Entity / API-Key, denominated in RMB or Tokens |
| Rate limit policy | Rate Limit Policy | Limits request rate along dimensions such as RPM / TPM / concurrency |

### AI Gateway Instance Pool

The "AI Gateway instance pool" corresponds to the list of engine addresses of Data Plane BFE instances, and is used to register which BFE nodes can pull configurations from the Control Plane. It answers the question "to whom should configurations be delivered". It differs in meaning from the backend instance pool (`instance_pool`) in a Provider: the former is the Data Plane entry, while the latter is the upstream model service endpoint.

### Model Provider (Provider)

The "model provider" corresponds to the `/providers` resource of the OpenAPI, and answers the questions "who is downstream, which models can be accessed, how to authenticate, and where the backend is". It holds:

- Backend instance pool (`instance_pool`): the real AI service endpoints;
- Model protocols (`model_protocols`): e.g. `openai`, `anthropic`;
- Model list (`models`) and the model discovery endpoint;
- Plaintext service authentication keys (`keys`).

Multiple Clusters can reference the same Provider, enabling reuse of instance pools and keys.

### AI Business Cluster (Cluster)

The "AI business cluster" corresponds to the `/clusters` resource of the OpenAPI, and answers the questions "how traffic is forwarded, which models are used, and how Key weights are distributed". It strongly references a Provider via `llm_config.provider`, and on top of that declares forwarding models, Key weights, timeouts, health checks, and other policies. For detailed design rationale and the data model, see [Chapter 10: Provider and Cluster Design](./chapter10-provider-and-cluster.md).

### Organization (Entity) and API-Key

An "Entity" represents the organizational structure, such as a department, team, or project. Each Entity has its own model allowlist/blocklist, quota plan (QuotaPlan), rate limit policy (RateLimitPolicy), and routing rules. Entities form a hierarchical tree bottom-up through the `parent_id` field; their types are defined by the `entity_types` table, and API-Keys attached to an Entity inherit the Entity-level model policies and quota/rate limit/routing policies. For details, see the "Entity Hierarchy Tree and Model Inheritance" section in [Chapter 9: Entity and API-Key Design](./chapter09-apikey-design.md).

An "API-Key" is the credential used by business systems when calling the Data Plane forwarding entry. An API-Key can be attached to an Entity to inherit its quota and rate limit policies, and can also have its own API-Key-level routing rules. For details, see [Chapter 8: Authentication and Authorization Design](./chapter08-auth-design.md) and [Chapter 9: Entity and API-Key Design](./chapter09-apikey-design.md).

### Route Table

The "route table" corresponds to routing rules at three levels: Global / Entity / API-Key. After a request enters the Data Plane, it is matched in the order **API-Key > Entity > Global**: first the Key's dedicated table is checked; if no match is found, the table of the organization the Key is attached to is checked; and finally it falls back to Global. In a routing rule, `cond` is a BFE condition expression, `targets` specifies the target clusters and weights, and `fallbacks` specifies the degradation targets. For details, see [Chapter 11: AI Routing Rules Design](./chapter11-ai-route-rules.md).

### Quota and Rate Limiting

- **Quota Plan (Quota Plan)**: sets a budget cap for an Entity or API-Key, supporting cost accounting in RMB or Token dimensions. For details, see [Chapter 12: Quota and Rate Limiting Design](./chapter12-quota-and-rate-limit.md).
- **Rate Limit Policy (Rate Limit Policy)**: limits request rate along dimensions such as RPM / TPM / concurrency, preventing a single point of traffic from overwhelming the backend or exhausting the budget. For details, see [Chapter 12: Quota and Rate Limiting Design](./chapter12-quota-and-rate-limit.md).

---

## Core Components

### AI Gateway API (Control Plane Core)

AI Gateway API is the Control Plane core component of the Rainway AI Gateway. It is responsible for:

- Exposing the **management-plane OpenAPI** (`/open-api/v1`) for Dashboard and admin scripts;
- Exposing the **Data Plane InnerAPI** (`/inner-api/v1`) for BFE and Conf Agent to pull configurations;
- Creating, storing, versioning, and distributing policies/configurations;
- Maintaining core data such as API-Keys, Entities, Providers, Clusters, routing rules, quotas, rate limits, and certificates.

AI Gateway API adopts a classic three-layer architecture:

```
Interface layer (endpoints)
    ├── openapi_v1/    Management-plane APIs
    ├── innerapi_v1/   Data Plane export APIs
    └── middleware/    Global middleware

Model layer (model)
    ├── api_key/       API-Key business logic
    ├── entity/        Entity / Entity-Type
    ├── icluster_conf/ Cluster management
    ├── iprovider/     Provider management
    ├── quota/         Quota plans
    ├── rate_limit_policy/  Rate limit policies
    └── ...

Storage layer (storage/rdb)
    ├── internal/dao/  DAO layer
    └── */             Per-business Storage implementations
```

For details, see [Chapter 6: Control Plane Core Design — AI Gateway API](./chapter06-control-plane-design.md).

### Dashboard (Management Console)

Dashboard is the Web UI for operations and administrators. It performs visual operations by calling the OpenAPI. It lowers the usage barrier, allowing non-developers to perform operations such as Provider onboarding, Cluster configuration, API-Key management, and quota settings.

Dashboard does not operate the database directly; all changes go through AI Gateway API, so it stays loosely coupled with the backend Control Plane.

### BFE (Data Plane Forwarding Engine)

BFE is the Data Plane of the Rainway AI Gateway, responsible for actually receiving and forwarding AI requests. It extends the open-source BFE project with several AI-specific modules:

- `mod_ai_route`: selects the target Cluster and model according to AI routing rules;
- `mod_ai_token_auth`: validates the API-Key and performs quota checks and deduction;
- `mod_ai_rate_limit`: enforces TPM/RPM/concurrency rate limiting;
- `mod_body_process`: parses Token usage in streaming responses.

BFE processes requests in a plugin-based manner, with each module executing at a specific stage of the request lifecycle. For details, see [Chapter 7: Data Plane Forwarding Design — BFE](./chapter07-data-plane-design.md).

### Conf Agent (Configuration Agent)

Conf Agent is a Sidecar component deployed alongside BFE. Its main responsibilities include:

1. Periodically polling the InnerAPI of AI Gateway API;
2. Persisting the latest configuration to local disk with version management;
3. After detecting a configuration change, invoking BFE's hot reload interface to make it take effect;
4. Cleaning up expired versions to prevent unbounded disk growth.

Thanks to Conf Agent, the Control Plane does not need to connect to the Data Plane directly — configuration delivery is **pull-based**, which naturally suits deployments across network zones.

### Service Controller (Service Discovery)

Service Controller is used to discover backend services in Kubernetes environments and synchronize service addresses to AI Gateway API. In this way, backend instances in a Cluster can automatically follow K8s service changes, reducing manual maintenance costs.

---

## Division of Responsibilities Between Control Plane and Data Plane

| Responsibility | Control Plane (AI Gateway API) | Data Plane (BFE) |
|------|--------------------------|---------------|
| Configuration creation and modification | ✅ | ❌ |
| Configuration persistence | ✅ (MySQL) | ❌ |
| Configuration versioning | ✅ | ❌ |
| Configuration export | ✅ (InnerAPI) | ❌ |
| Request forwarding | ❌ | ✅ |
| API-Key validation | ❌ | ✅ |
| Quota deduction | ❌ | ✅ |
| Rate limit enforcement | ❌ | ✅ |
| Log output | ❌ | ✅ |
| Monitoring metrics | Limited self-monitoring | ✅ Rich request-level metrics |

This division follows the principle "the Control Plane makes decisions, the Data Plane executes". All policy definitions are completed on the Control Plane, while the Data Plane only executes policies efficiently and reliably.

---

## Configuration Lifecycle

A configuration in the Rainway AI Gateway typically goes through the following stages from creation to taking effect:

```
Admin / script
     |
     v
Dashboard / OpenAPI
     |
     v
AI Gateway API (validation, storage, new version)
     |
     v
MySQL + Redis (persistence and real-time quota)
     |
     v
InnerAPI (export configurations by topic)
     |
     v
Conf Agent (poll, compare version numbers)
     |
     v
Local config files + symlink switch
     |
     v
BFE /reload (hot reload)
     |
     v
Request forwarding (executed under new policies)
```

**Stage descriptions**:

1. **Configuration entry**: an admin submits a configuration change through the Dashboard or OpenAPI, e.g. adding an API-Key or modifying a routing rule.
2. **Validation and storage**: AI Gateway API validates the parameters, writes to MySQL, and updates the configuration version number.
3. **Configuration export**: InnerAPI exports configurations from the database by topic (such as `mod-api-key`, `ai-route`, `rate-limit-policy`) into JSON/TOML files consumable by BFE.
4. **Configuration pull**: Conf Agent periodically calls the InnerAPI and compares the local version number with the remote one.
5. **Local switch**: when a new version is detected, Conf Agent writes the new configuration into a new version directory and atomically switches to it via a symlink.
6. **Hot reload**: Conf Agent calls BFE's `/reload/{module}` interface, and BFE loads the new configuration without restarting.
7. **Taking effect**: subsequent AI requests are routed, authenticated, quota-checked, and rate-limited according to the new policies.

For the detailed configuration export mechanism, see [Chapter 14: Configuration Export and Versioning Design](./chapter14-config-export-and-version-control.md).

---

## Deployment Topologies

### Minimal Deployment

Suitable for development/testing or low-traffic scenarios:

```
+-----------------+
| AI Gateway API  |
| Dashboard       |
| MySQL / Redis   |
+--------+--------+
         |
         v
+-----------------+
| Conf Agent      |
| BFE             |
+--------+--------+
         |
         v
   Model providers
```

All components can run on the same machine or in the same container, making it easy to get started quickly.

### Production Deployment

Production environments typically use separated deployment:

```
                    +-------------+
                    |   Admin     |
                    +------+------+
                           |
        +------------------+------------------+
        v                  v                  v
+---------------+  +---------------+  +---------------+
|  Dashboard    |  | AI Gateway API|  |   MySQL       |
|  (Nginx front)|  |  (multi-instance)|  |   Redis    |
+---------------+  +---------------+  +---------------+
                           |
                           | InnerAPI
        +------------------+------------------+
        v                  v                  v
+---------------+  +---------------+  +---------------+
| Conf Agent    |  | Conf Agent    |  | Conf Agent    |
| BFE           |  | BFE           |  | BFE           |
| (Zone A)      |  | (Zone B)      |  | (Zone C)      |
+-------+-------+  +-------+-------+  +-------+-------+
        +------------------+------------------+
                           v
                    Model providers
```

Production deployment points:

- AI Gateway API is deployed with multiple instances, accessed through a load balancer;
- MySQL and Redis use high-availability solutions;
- BFE is deployed across multiple availability zones, with Conf Agent pulling configurations independently;
- Dashboard can be deployed standalone or run in the same container as AI Gateway API.

For detailed deployment steps, see [Chapter 21: Installation and Deployment](../operation/chapter17-installation-and-deployment.md).

---

## Chapter Summary

- As a unified access layer, the Rainway AI Gateway solves problems such as multi-provider integration, API-Key management, cost and quota control, high availability, and security auditing.
- The system adopts a separated Control Plane and Data Plane architecture: the Control Plane manages policies, and the Data Plane forwards requests and enforces policies.
- Core components include AI Gateway API, Dashboard, BFE, Conf Agent, and Service Controller.
- The configuration lifecycle covers six stages: creation, storage, export, pull, hot reload, and taking effect.
- Multiple topologies are supported, from minimal deployment to multi-zone production deployment.

---

## References

- `ai-gateway-api/README.md`
- `ai-gateway-api/design-docs/sys-design/总体设计文档.md`
- `ai-gateway-api/design-docs/sys-design/details/InnerAPI配置导出与版本控制.md`
- `bfe/AGENTS.md`
- `conf-agent/AGENTS.md`
