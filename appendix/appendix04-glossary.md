# 附4 术语表

## A

**ai-gateway-controller（原名 Service Controller）**
：部署在 Kubernetes 集群内的控制器，监听带 `ai-gateway-api-name` 标签的 Service 及其同名 Endpoints，按具名端口把 Ready 实例经 InnerAPI `PUT /inner-api/v1/k8s_pools/{name}/instances` 全量上报为 K8s 实例池；控制面不做扇出，各消费方按 `k8s_pool_name` 实时查询有效池，再经 `cluster_table` 导出到 BFE。

**AI 业务集群（Cluster）**
：引用 Provider 并配置转发策略的逻辑后端单元，决定流量如何转发、使用哪些模型、Key 权重如何分配。

**API Key**
：调用方访问壬远 AI 网关的凭证，请求头中携带（不需要 Bearer 前缀）。可挂载到 Entity 继承配额、限流与路由策略。

## B

**BFE**
：百度开源的七层负载均衡与流量网关，在壬远 AI 网关中作为数据面转发引擎，负责 AI 请求的鉴权、限流、配额、路由与转发。

**BFE 集群（BFECluster）**
：登记数据面 BFE 引擎地址清单的资源（Server Data）。记录由部署初始化数据（`db_ddl.sql` 种子数据，默认集群引用内置实例池 `BFE.aipool`）维护，未暴露 OpenAPI 管理接口；控制面据此确定配置下发给哪些 BFE 节点。

**Balancer**
：BFE 的负载均衡模块，负责在 Cluster 内部多个后端实例之间分发请求。

## C

**Cluster（集群）**
：见“AI 业务集群”。

**Conf Agent（配置代理）**
：部署在 BFE 侧的组件，周期性地从 AI Gateway API 拉取最新配置，写入本地版本目录并触发 BFE 热加载。

**Condition（条件表达式）**
：BFE 提供的表达式语言，AI 路由规则通过 `Cond` 字段描述命中条件，如 `req_body_json_in("model", "gpt-4", false)`。

**操作日志（Operation Log）**
：平台写操作的只读审计记录，由配置变更自动产生，包含操作人、动作、资源、结果与变更摘要（before/after JSON，敏感字段脱敏）。

**上下文压缩（Context Compression）**
：按目标模型 token 预算对超大上下文做无损裁剪与规则改写两阶段压缩，fail-open。

## D

**Dashboard（控制台）**
：壬远 AI 网关的 Web 管理界面，面向运维人员提供资源管理、消费者管理、路由管理、用户管理等功能。

## E

**有效实例池（Effective Instance Pool）**
：Provider 实际生效的后端实例集合（`k8s_pool` 模式下实时查询 K8s 池，否则取 `instance_pool`），控制面下游唯一消费口径。

**Entity（组织）**
：调用方分组，可表达部门、团队或项目。每个 Entity 可挂载配额计划、限流策略、模型黑白名单与路由规则。

**Entity Type（组织类型）**
：Entity 的层级分类定义，通过 `level` 字段区分层级高低，数值越小层级越高。

**EPP（Endpoint Picker）**
：独立于 BFE 的后端调度器，为 EPP 模式 Cluster 接管后端实例选择（负载均衡），以实例组为部署单元（本期每组恰好 1 个实例）并被登记到 EPP 实例池。

## F

**Fallback（降级）**
：AI 路由规则中 `fallbacks` 指定的备用目标。当首选 `targets` 全部失败时，按顺序尝试 fallback 目标。

**分段计价**
：在服务商侧定义忙时（`peak`）时间段（含 IANA 时区），配合模型定价中的分时段价格实现按时段差异化计费。

## G

**Global 路由表**
：AI 路由规则三级体系中的最底层兜底规则，所有 API-Key 最终都会绑定并查找。

## I

**InnerAPI**
：AI Gateway API 提供的内部接口，主要供 Conf Agent 与 BFE 拉取配置、完成版本同步。

**Instance Pool（实例池）**
：在 Provider 中定义的后端 AI 服务真实地址、端口与权重集合，供 Cluster 引用。控制台支持 IP 模式与服务商域名模式两种接入方式。

**意图分类（Intent Classification）**
：由决策服务对请求做语义分类（choice/score 问题），路由条件 `req_ai_intent_in` 按分类结果分流；支持显式意图头与置信度门限。

## J

**均衡模式**
：Cluster 的后端负载均衡模式：`WRR`（BFE 本地加权轮询，默认）或 `EPP`（由 EPP 调度器接管后端选择）；EPP 模式集群无有效分配时降级为 WRR。

## K

**K8s 实例池（K8s Pool / k8s_pool）**
：由 ai-gateway-controller 经 InnerAPI `/inner-api/v1/k8s_pools` 全量上报和维护的后端实例集合，池成员的唯一写入方是该控制器；Provider 可切换实例来源（`instance_pool` / `k8s_pool`），以有效实例池契约下发。

**Key Affinity（Key 亲和性）**
：基于 Redis 实现会话级 Key 亲和，同一 `ClientKeyId` 在一定时间内持续命中同一 Provider Key；Redis 连接由 `bfe.conf` 的 `[AIKeyAffinity]` 配置段自持。惩罚时长可配置（`penalty_ttl_429` 默认 60s、`penalty_ttl_auth` 默认 300s）。

**expr 条件引擎（Expression Engine）**
：基于 expr-lang/expr 的条件表达式引擎（`bfe_expr` 包），与 53 个既有条件原语并存；`bfe.conf` 的 `ConditionEngineMode` 控制 `LegacyFirst`（默认）/ `ExprCompatible` / `ExprOnly` 三种模式，动作参数支持 `=expr` 动态求值。

## M

**物化视图（Materialized View）**
：ClickHouse/StarRocks 侧按明细实时聚合的预计算表；ClickHouse 查询需 `GROUP BY` 维度 + `sum(指标)` 兜底（SummingMergeTree），StarRocks 异步 MV 随基表加列需重建。

**Model Mapping（模型映射）**
：Cluster 中将用户请求的模型名映射为后端实际使用的模型名的机制。

**模型重定向（Model Redirect）**
：Cluster 大模型配置中将客户端请求的模型名映射为转发到后端的模型名；模型访问控制与限流的「适用模型」均按重定向后的目标模型判定。

**Model Price（模型定价）**
：维护模型在不同 Provider 与时段下的价格，用于 RMB 配额成本核算。

**Model Protocol（模型协议）**
：Provider 支持的上游协议类型，如 `openai`、`anthropic`、`gemini` 等，用于请求体与响应体的协议适配与认证头风格选择。

## O

**OpenAPI**
：AI Gateway API 提供的对外管理接口，供 Dashboard 与外部程序调用，完成资源创建、查询、更新与删除。

## P

**Provider（模型服务商）**
：持有后端实例池、模型协议、模型列表与服务鉴权 Key 明文的资源；Cluster 通过“所属服务商”引用 Provider。

**Product（产品线）**
：BFE 中的顶层资源隔离单位，AI 网关模式下主要用于产品线识别与配置上下文加载。

**protocol_paths（协议路径映射）**
：Provider 上的可选字段，声明各模型协议（`openai`/`anthropic`）对应的上游 base path；BFE 据此将标准入口 `/v1/...` 改写为 provider 原生前缀，未配置则原样透传（`gemini` 协议不支持路径改写）。

**缺价处置（PriceMissAction）**
：集群级配置 `llm_config.price_miss_action`，模型/模式无价格条目时的三态行为：`fallback_chat`（缺省，按同模型 chat 价计费）/ `bill_zero`（按 0 计费）/ `reject`（转发前拒绝，503 `PRICE_NOT_CONFIGURED`）；信号经 `ai_price_miss` / `ai_price_warn` 访问日志字段全链路上报。

**配额欠费（Quota Overage）**
：余额模型：配额扣减不截断到 0，Redis 余额允许为负，欠费如实记录；鉴权对负余额继续拒绝（429），对外 `remaining` 展示 0、`used` 可大于 `quota`。

**配额不足时放行**
：配额计划的可选开关；开启后配额余额扣减到 0 仍放行请求（继续统计用量），适合试运行阶段；关闭时余额不足即拒绝。

## Q

**Quota Plan（配额计划）**
：为 API-Key 或 Entity 分配的 Token 总量或 RMB 预算，支持 `total_token` 与 `RMB` 两种单位及周期重置。

## R

**RateLimitPolicy（限流策略）**
：控制 API-Key 或 Entity 对后端 AI 模型访问速率的策略，支持 TPM、RPM 与最大并发数限制。

**Redis 唯一真实来源**
：配额余额、限流计数等运行时状态直接读写 Redis，管理面查询余额时不再维护数据库冷副本。

**Routine Load**
：Doris/StarRocks 持续消费 Kafka 写入表的导入任务；起始偏移 `OFFSET_BEGINNING` 保证任务创建前的消息不丢失。

**Route Rule（路由规则）**
：AI 路由表中的单条规则，包含 `Cond` 命中条件、`targets` 目标列表与可选的 `fallbacks` 降级列表。

**Route Table（路由表）**
：Global / Entity / API-Key 三级 AI 路由规则集合，按 `apikey > entity > global` 优先级依次匹配。

## S

**语义缓存（Semantic Cache）**
：在精确匹配缓存之上按 embedding 向量相似度复用答案的缓存形态；依赖 embedding 服务与 Chroma 向量库，距离门限可配。

**Session Key**
：Dashboard 登录后由 `/auth/session-keys` 生成的会话凭证，格式为 `Authorization: Session {session_key}`。

**Sticky Session（会话保持）**
：将同一客户端请求长期绑定到同一后端实例的机制，AI 场景通常无需开启。

**Sub Cluster（子集群）**
：Cluster 生成 BFE 配置时自动创建的子集群，绑定到 Cluster 对应的实例池。

**数据报表（Report）**
：控制台运营数据可视化模块，提供总览指标卡、时序图表、维度排行、占比分布与日志明细查询；数据后端支持 MySQL（轻量形态）与 Doris（标准形态）。

## T

**Tier（时段层级）**
：模型定价中按时间维度划分的价格层级，初期仅支持 `peak`（忙时）；忙时时间段定义在服务商资源中维护（分段计价配置）。

**推理内容补齐（EnsureReasoningContent）**
：集群级配置 `llm_config.ensure_reasoning_content`（协议列表，仅 `openai`/`anthropic`，空 = 关闭）；对要求回传 `reasoning_content` 的思考模式上游，BFE 在校验窗口内对 assistant 消息定点补齐空值，避免路由切换后首个请求被上游 400 拒绝。

**Token**
：由 `/auth/tokens` 创建的程序访问凭证，Scope 取值为 `System`（完整管理权限）、`SystemReadonly`（只读管理权限）与 `Support`（只读导出权限）。

**TPM（Tokens Per Minute）**
：每分钟 Token 消耗上限，采用滑动窗口计数。

**流量镜像（Traffic Mirror）**
：将命中的 AI 请求副本异步转发到影子集群的能力，用于版本灰度与影子验证；主路径不被阻塞。

**RPM（Requests Per Minute）**
：每分钟请求次数上限，采用固定窗口计数。

## V

**VersionControlManager**
：AI Gateway API 中负责配置导出、MD5 签名与版本号管理的组件。

**Visitor**
：AI Gateway API 认证授权模块中对用户与 Token 的统一抽象。

## W

**加权随机（Weighted Random）**
：AI 路由规则在 `targets` 之间按权重随机选择目标 Cluster 的算法。

**转发后目标模型**
：请求模型经路由「指定模型」覆盖、集群「裁剪前缀」、集群「模型重定向」依次解析后的最终模型名（与后端实际收到的模型一致）；模型访问控制与 TPM/RPM 规则的「适用模型」均按该模型名匹配，而非请求体原始模型名。

## 参考

- `ai-gateway-web/docs/zh-cn/14-appendix.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/README.md`
- `bfe/docs/zh_cn/sys_design/ai_error_codes.md`
