# Appendix 4: Glossary

## A

**ai-gateway-controller (formerly Service Controller)**
: A controller deployed inside a Kubernetes cluster; it watches Services carrying the `ai-gateway-api-name` label together with their same-named Endpoints, and for each named port reports the Ready instances in full via the InnerAPI `PUT /inner-api/v1/k8s_pools/{name}/instances` as a K8s pool. The Control Plane performs no fan-out: every consumer resolves the effective pool on demand by `k8s_pool_name`, which then reaches BFE through the `cluster_table` export.

**AI Business Cluster (Cluster)**
: A logical backend unit that references a Provider and configures forwarding policies, determining how traffic is forwarded, which models are used, and how Key weights are distributed.

**API Key**
: The credential used by callers to access Rainway AI Gateway, carried in the request header (no Bearer prefix required). It can be attached to an Entity to inherit quota, rate limiting, and routing policies.

## B

**BFE**
: Baidu's open-source Layer-7 load balancer and traffic gateway. In Rainway AI Gateway, it serves as the Data Plane forwarding engine, responsible for authentication, rate limiting, quota enforcement, routing, and forwarding of AI requests.

**BFE Cluster (BFECluster)**
: The resource that registers the list of Data Plane BFE engine addresses (Server Data). Records are maintained by deployment seed data (the `db_ddl.sql` seed, where the default cluster references the built-in instance pool `BFE.aipool`) and no OpenAPI management endpoint is exposed; the Control Plane uses it to determine which BFE nodes configurations are delivered to.

**Balancer**
: BFE's load balancing module, responsible for distributing requests among multiple backend instances within a Cluster.

## C

**Cluster**
: See "AI Business Cluster".

**Conf Agent (Configuration Agent)**
: A component deployed alongside BFE that periodically pulls the latest configuration from AI Gateway API, writes it to a local version directory, and triggers BFE hot reload.

**Condition (Conditional Expression)**
: The expression language provided by BFE. AI routing rules describe matching conditions via the `Cond` field, e.g., `req_body_json_in("model", "gpt-4", false)`.

**Operation Log**
: A read-only audit record of platform write operations, automatically produced by configuration changes, containing the operator, action, resource, result, and a change summary (before/after JSON with sensitive fields masked).

**Context Compression**
: Compresses oversized context against the target model's token budget in two stages — lossless trimming and rule rewriting — with fail-open behavior.

## D

**Dashboard**
: The web administration interface of Rainway AI Gateway, providing operations staff with resource management, consumer management, routing management, user management, and other features.

## E

**Effective Instance Pool**
: The set of backend instances actually in effect for a Provider (queried live from the K8s pool when in `k8s_pool` mode, otherwise `instance_pool`); the sole consumption contract downstream of the Control Plane.

**Entity (Organization)**
: A grouping of callers that can represent a department, team, or project. Each Entity can have quota plans, rate limit policies, model allowlists/blocklists, and routing rules attached.

**Entity Type (Organization Type)**
: The hierarchical classification definition of an Entity, distinguishing levels via the `level` field; the smaller the value, the higher the level.

**EPP (Endpoint Picker)**
: A backend scheduler independent of BFE that takes over backend instance selection (load balancing) for Clusters in EPP mode; it is deployed in instance groups (exactly 1 instance per group in this release) and registered in the EPP instance pool.

## F

**Fallback (Degradation)**
: The backup target specified by `fallbacks` in an AI routing rule. When all preferred `targets` fail, fallback targets are tried in order.

**Tiered Pricing**
: Defines peak (`peak`) time windows (with an IANA time zone) on the Provider side, and works with tier prices in Model Pricing to enable time-differentiated billing.

## G

**Global Route Table**
: The bottom-level fallback rules in the three-tier AI routing rule system; every API-Key is ultimately bound to and looked up against it.

## I

**InnerAPI**
: The internal interfaces provided by AI Gateway API, mainly used by Conf Agent and BFE to pull configurations and complete version synchronization.

**Instance Pool**
: The set of real backend AI service addresses, ports, and weights defined within a Provider, referenced by Clusters. The Dashboard supports two access modes: IP mode and provider domain mode.

**Intent Classification**
: Semantic classification of requests (choice/score questions) by the decision service; the routing condition `req_ai_intent_in` splits traffic by classification result; supports explicit intent headers and a confidence threshold.

## J

**Balance Mode**
: The backend load balancing mode of a Cluster: `WRR` (BFE local weighted round robin, the default) or `EPP` (backend selection taken over by the EPP scheduler); EPP-mode Clusters without a valid assignment degrade to WRR.

## K

**K8s Pool (k8s_pool)**
: The set of backend instances reported and maintained in full by ai-gateway-controller via the InnerAPI `/inner-api/v1/k8s_pools`; that controller is the sole writer of the pool members. A Provider can switch its instance source (`instance_pool` / `k8s_pool`) and the result is delivered under the Effective Instance Pool contract.

**Key Affinity**
: Session-level Key affinity implemented based on Redis; the same `ClientKeyId` keeps hitting the same Provider Key within a certain period of time; the Redis connection is held by the `[AIKeyAffinity]` section of `bfe.conf`. The penalty durations are configurable (`penalty_ttl_429` defaults to 60s and `penalty_ttl_auth` to 300s).

**expr Condition Engine (Expression Engine)**
: The condition expression engine based on expr-lang/expr (the `bfe_expr` package), coexisting with the 53 legacy condition primitives; `ConditionEngineMode` in `bfe.conf` selects among `LegacyFirst` (default), `ExprCompatible`, and `ExprOnly`, and action parameters support `=expr` dynamic evaluation.

## M

**Materialized View**
: A precomputed table that aggregates detail data in real time on the ClickHouse/StarRocks side; ClickHouse queries must fall back on `GROUP BY` dimensions + `sum(metric)` (SummingMergeTree), and StarRocks asynchronous MVs must be rebuilt when columns are added to the base table.

**Model Mapping**
: The mechanism in a Cluster that maps the model name in a user's request to the model name actually used by the backend.

**Model Redirect**
: In a Cluster's LLM configuration, maps the model name in the client's request to the model name forwarded to the backend; both model access control and the "applicable models" of TPM/RPM rules are evaluated against the redirected target model.

**Model Price**
: Maintains the price of a model under different Providers and time periods, used for RMB quota cost accounting.

**Model Protocol**
: The upstream protocol types supported by a Provider, such as `openai`, `anthropic`, `gemini`, etc., used for request/response body protocol adaptation and authentication header style selection.

## O

**OpenAPI**
: The external management interfaces provided by AI Gateway API, called by the Dashboard and external programs to create, query, update, and delete resources.

## P

**Provider (Model Service Provider)**
: A resource holding the backend instance pool, model protocols, model list, and plaintext service authentication Keys; a Cluster references a Provider via its owning-provider field.

**Product (Product Line)**
: The top-level resource isolation unit in BFE; in AI Gateway mode it is mainly used for product line identification and configuration context loading.

**protocol_paths (Protocol Path Mapping)**
: An optional field on a Provider declaring the upstream base path for each model protocol (`openai`/`anthropic`); BFE rewrites the standard entry `/v1/...` to the provider's native prefix based on it, and forwards unchanged when it is unconfigured (the `gemini` protocol does not support path rewriting).

**Price-Miss Action (PriceMissAction)**
: A cluster-level configuration (`llm_config.price_miss_action`) defining the behavior when a model/mode has no price entry, with three states: `fallback_chat` (default; bill at the same model's chat price) / `bill_zero` (bill 0) / `reject` (reject before forwarding with 503 `PRICE_NOT_CONFIGURED`); signals are reported end-to-end via the `ai_price_miss` / `ai_price_warn` access-log fields.

**Quota Overage**
: The balance model: quota deduction is not truncated at 0, so the Redis balance may go negative and overages are recorded faithfully; authentication keeps rejecting negative balances (429), while externally `remaining` displays 0 and `used` may exceed `quota`.

**Allow When Quota Exhausted**
: An optional switch on a quota plan; when enabled, requests are still allowed after the quota balance reaches 0 (usage continues to be tracked), which suits the trial-run phase; when disabled, insufficient balance causes rejection.

## Q

**Quota Plan**
: The total Token amount or RMB budget allocated to an API-Key or Entity, supporting two units, `total_token` and `RMB`, with periodic reset.

## R

**RateLimitPolicy (Rate Limit Policy)**
: A policy controlling the rate at which an API-Key or Entity accesses backend AI models, supporting TPM, RPM, and maximum concurrency limits.

**Redis as the Single Source of Truth**
: Runtime state such as quota balances and rate limit counters is read from and written to Redis directly; the management plane no longer maintains a cold database copy when querying balances.

**Routine Load**
: An import job in Doris/StarRocks that continuously consumes Kafka and writes to a table; the starting offset `OFFSET_BEGINNING` ensures messages produced before the job is created are not lost.

**Route Rule**
: A single rule in the AI route table, containing a `Cond` matching condition, a `targets` destination list, and an optional `fallbacks` fallback list.

**Route Table**
: The three-tier AI routing rule collection of Global / Entity / API-Key, matched in priority order `apikey > entity > global`.

## S

**Semantic Cache**
: A caching form that reuses answers by embedding vector similarity on top of exact-match caching; it depends on the embedding service and the Chroma vector store, with a configurable distance threshold.

**Session Key**
: The session credential generated by `/auth/session-keys` after Dashboard login, in the format `Authorization: Session {session_key}`.

**Sticky Session**
: A mechanism that binds requests from the same client to the same backend instance for a long period; it is usually unnecessary in AI scenarios.

**Sub Cluster**
: A sub-cluster automatically created when a Cluster generates BFE configuration, bound to the instance pool corresponding to the Cluster.

**Data Report**
: The console's operational data visualization module, providing overview metric cards, time-series charts, dimension rankings, distribution pies, and log detail queries; the data backend supports MySQL (lightweight form) and Doris (standard form).

## T

**Tier (Time Period Level)**
: The price level divided by time dimension in model pricing; only the `peak` (busy hours) tier is supported initially. The busy-hours windows are maintained in the Provider resource (tiered pricing configuration).

**Reasoning-Content Completion (EnsureReasoningContent)**
: A cluster-level configuration (`llm_config.ensure_reasoning_content`, a protocol list with only `openai`/`anthropic`; empty = disabled); for reasoning-mode upstreams that require `reasoning_content` to be echoed back, BFE patches empty values into assistant messages within the validation window, preventing the first request after a route switch from being rejected with 400.

**Token**
: A programmatic access credential created via `/auth/tokens`, with Scopes `System` (full management permissions), `SystemReadonly` (read-only management permissions) and `Support` (read-only export permissions).

**TPM (Tokens Per Minute)**
: The upper limit of Token consumption per minute, using sliding window counting.

**Traffic Mirror**
: The capability to asynchronously forward copies of matched AI requests to a shadow cluster, used for version gray release and shadow validation; the main path is not blocked.

**RPM (Requests Per Minute)**
: The upper limit of requests per minute, using fixed window counting.

## V

**VersionControlManager**
: The component in AI Gateway API responsible for configuration export, MD5 signatures, and version number management.

**Visitor**
: The unified abstraction of users and Tokens in the AI Gateway API authentication and authorization module.

## W

**Weighted Random**
: The algorithm by which an AI routing rule randomly selects a target Cluster among `targets` according to weights.

**Resolved Target Model**
: The final model name after the requested model is resolved in sequence through the route rule's "specified model" override, the Cluster's "strip/match prefix", and the Cluster's "model redirect" (consistent with the model name the backend actually receives); model access control and the "applicable models" of TPM/RPM rules are matched against this model name rather than the original model name in the request body.

## References

- `ai-gateway-web/docs/zh-cn/14-appendix.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/README.md`
- `bfe/docs/zh_cn/sys_design/ai_error_codes.md`
