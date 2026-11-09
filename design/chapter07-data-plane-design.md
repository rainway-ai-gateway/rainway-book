# 第七章 数据面转发设计：BFE

## 本章目标

本章聚焦壬远 AI 网关的数据面组件 BFE（Beyond Front End），介绍 BFE 如何接收、处理并转发 AI 请求。通过阅读本章，读者将能够理解：

- BFE 在壬远 AI 网关中承担的角色及其与控制面（AI Gateway API）的关系；
- BFE 的请求处理生命周期，尤其是 AI 网关模式下的独立转发路径；
- `mod_ai_token_auth`、`mod_ai_cache`、`mod_ai_route`、`mod_ai_intent`、`mod_ai_rate_limit`、`mod_ai_context`、`mod_traffic_mirror`、`mod_body_process` 等 AI 相关模块的执行顺序与协作方式；
- 协议适配层 `bfe_model_protocol` 的职责、适配器接口与协议注册机制；
- BFE 模块框架 `bfe_module` 的回调机制与模块注册方式；
- AI 相关配置文件的加载、校验与热加载机制；
- 关键配置示例与转发降级行为。
- 上游路径按协议改写（`ProtocolPaths`）的执行时机、透传兼容语义与 fallback 重算行为。

## BFE 在壬远 AI 网关中的角色

壬远 AI 网关采用控制面与数据面分离的架构。控制面由 AI Gateway API 负责，完成 Provider、Cluster、API-Key、配额、限流策略等元数据的管理；数据面由 BFE Server 承担，负责把终端用户请求转发到实际的大模型后端。

BFE 是一个开源的七层负载均衡器，起源于百度，现为 CNCF Sandbox 项目。壬远 AI 网关在其基础上扩展了 AI 相关的模块与转发路径，使其具备以下能力：

- 基于 API-Key、Entity、Global 三级优先级进行 AI 路由；
- 对 API-Key 进行鉴权、配额校验与使用量的最终扣除；
- 按产品、API-Key 等维度进行分布式限流；
- 解析流式响应（SSE）中的 token 使用量，用于配额统计；
- 在目标集群失败时按 `fallbacks` 顺序降级，支持多 API-Key 轮换与重试。

数据面需要具备高并发、低延迟、可观测和可热更新等特性。BFE 通过事件驱动的连接处理、模块化的回调框架以及原子化的配置切换，满足这些要求。控制面与数据面的交互方式详见 `bfe/AGENTS.md`：控制面通过配置分发机制将路由、鉴权、限流等数据下发到 BFE；BFE 在运行期加载这些配置并按规则转发流量。数据面不直接访问控制面的数据库，只消费由控制面生成的配置文件，这种解耦使得数据面可以独立扩展和升级。

## BFE 请求处理生命周期

BFE 在启动后监听 HTTP/HTTPS/HTTP2/WebSocket 等连接。每个请求进入 BFE 后，会依次经过连接接入、协议解析、租户识别、模块回调、后端转发、响应发送等阶段。在 AI 网关模式下，BFE 会进入独立的 `ServeHTTPForAI()` 转发路径。

连接接入阶段由 `bfe_server/` 中的监听器完成，负责 TLS 握手、会话管理和协议协商。HTTP 请求解析由 `bfe_http/`、`bfe_http2/` 等协议实现负责，生成 `bfe_basic.Request` 对象。随后 BFE 进入模块回调阶段，按固定顺序调用已注册模块的回调函数。AI 相关模块主要在 `HandleFoundProduct`、`HandleAfterAITargetModel` 与 `HandleForward` 阶段介入，而响应阶段则由 `HandleReadResponse` 与 `HandleRequestFinish` 处理。

### 传统路径与 AI 网关路径的分发

在 `bfe_server/http_conn.go` 的 `conn.serveRequest()` 中，BFE 根据 `EnableAiGateway` 配置决定进入哪条路径：

```go
var ret1 int
if c.server.Config.Server.EnableAiGateway {
    ret1 = c.server.ReverseProxy.ServeHTTPForAI(w, request)
} else {
    ret1 = c.server.ReverseProxy.ServeHTTP(w, request)
}
```

当 `ai_gateway_enabled = false` 时，请求沿用 BFE 原有的 `ServeHTTP()` 路径；当 `ai_gateway_enabled = true` 时，请求进入独立的 `ServeHTTPForAI()` 路径。两条路径共享连接管理、超时、响应发送等基础设施，但 AI 路径不再使用原有租户内集群路由，而是使用 `mod_ai_route` 计算出的 `AiRouteResult` 进行转发。

### AI 网关路径处理流程

```
┌─────────────────────────────────────┐
│  接收 HTTP/HTTPS/HTTP2 请求          │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  setClientAddr()                    │
│  设置原始客户端地址                  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  HandleBeforeLocation               │
│  mod_trust_clientip / mod_logid 等  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  findProduct()                      │
│  识别 BFE 租户（AI 网关场景兼容保留） │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  HandleFoundProduct                 │
│  mod_ai_token_auth                  │
│  mod_ai_cache                       │
│    命中 → BfeHandlerFinish 短路      │
│  mod_ai_route                       │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  AiRouteResult 检查                 │
│  未命中 → 返回 404 Not Found        │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  HandleAfterLocation                │
│  mod_body_process 等                │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  SelectTarget() 加权选择 target      │
│  构造 [target] + fallbacks 尝试列表  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  aiClusterInvoke() 循环转发          │
│  每次 attempt：                      │
│  HandleAfterAITargetModel           │
│  mod_ai_token_auth（目标模型校验）    │
│  mod_ai_rate_limit                  │
│  mod_ai_context（上下文压缩）         │
│  HandleForward                      │
│  mod_traffic_mirror（异步镜像）       │
│  失败时按 fallbacks 顺序降级         │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  HandleReadResponse / 发送响应       │
│  mod_ai_cache（未命中回写）           │
│  mod_ai_token_auth（usage 解析）     │
│  mod_ai_context（压缩注解）           │
│  mod_body_process                   │
└─────────────────────────────────────┘
```

从图中可以看出，AI 网关路径在 `HandleFoundProduct` 阶段依次完成鉴权、缓存查找与路由查找：缓存命中即短路返回缓存响应，未命中才继续执行路由规则求值；在 `ServeHTTPForAI()` 的转发循环内，每次集群 attempt 依次触发 `HandleAfterAITargetModel` 回调（目标模型校验、限流、上下文压缩）与 `HandleForward` 回调（流量镜像），并完成 target 选择、模型覆盖与 fallback 降级；响应阶段由 `HandleReadResponse` 完成缓存回写、usage 解析、压缩注解与 SSE 用量提取，最终由 `HandleRequestFinish` 完成配额结算（缓存命中跳过）。

## AI 相关模块的执行顺序与协作

BFE 的模块通过 `bfe_module` 框架注册到固定回调点。AI 相关模块的注册顺序在 `bfe/bfe_modules/bfe_modules.go` 的 `moduleList` 中显式指定：

```go
var moduleList = []bfe_module.BfeModule{
    // ... 其他模块 ...

    // mod_ai_token_auth
    mod_ai_token_auth.NewModuleAITokenAuth(),

    // mod_ai_cache
    // 要求：排在 mod_ai_token_auth 之后（不向未鉴权请求提供缓存内容），
    // 且在 mod_ai_route 之前——缓存命中在 HandleFoundProduct 即短路，
    // 路由规则求值与 req_ai_intent_in 懒解析整体跳过
    mod_ai_cache.NewModuleAiCache(),

    // mod_ai_route
    // 要求：排在 mod_ai_token_auth 之后（需要 ClientApiKey），
    // 且在 mod_ai_cache 之后（命中请求已结束）
    mod_ai_route.NewModuleAiRoute(),

    // mod_ai_intent
    // 顺序无要求：意图分类是懒解析的，Init 只向 bfe_basic 注入解析器
    // 与置信门限，由 req_ai_intent_in 在路由规则求值时触发
    mod_ai_intent.NewModuleAiIntent(),

    // mod_traffic_mirror
    // 要求：排在 mod_ai_route / mod_ai_token_auth 之后（HandleForward
    // 在全部 HandleFoundProduct / HandleAfterLocation 回调之后触发，
    // 镜像规则消费已解析的模型/API-Key 等 AI 上下文），
    // 且在 mod_access_pb3 之前（mirror 日志字段须在访问日志前写入）。
    // 只注册 HandleForward
    mod_traffic_mirror.NewModuleTrafficMirror(),

    // mod_body_process
    mod_body_process.NewModuleBodyProcess(),

    // 依赖 token 计算
    mod_ai_rate_limit.NewModuleAiRateLimit(),

    // mod_ai_context
    // 要求：排在 mod_ai_route 之后（HandleAfterAITargetModel 需要已解析
    // 的 TargetModel 计算 token 预算）、mod_ai_rate_limit 之后（限流按
    // 未压缩请求上下文判定），且在 mod_access_pb3 之前（压缩状态字段
    // 须在访问日志前写入）
    mod_ai_context.NewModuleAiContext(),

    // ...
}
```

该顺序决定了模块在 `HandleFoundProduct` 回调中的执行顺序：`mod_ai_token_auth` → `mod_ai_cache` → `mod_ai_route`。模型白名单校验与限流不在 `HandleFoundProduct` 执行，而是注册在转发阶段的 `HandleAfterAITargetModel` 回调点（目标模型解析后、转发前，按集群 attempt 触发），该点上 `mod_ai_token_auth` 先于 `mod_ai_rate_limit` 执行，保证白名单校验始终先于限流。这些模块通过 `AiBasicInfo` 与 `Request.Context` 共享状态，例如 `ClientApiKey`、`ClientKeyId`、`ClientModel`、`TargetModel`、`AiCacheStatus`、`AiIntent`、`AiRouteResult` 等。

执行顺序的约束由数据依赖决定：

- `mod_ai_token_auth` 最早执行，识别调用方身份并设置 `ClientApiKey` 与 `ClientKeyId`；
- `mod_ai_cache` 紧随其后，向已鉴权请求提供缓存查找，命中即短路返回，未命中保存 miss 上下文后继续；
- `mod_ai_route` 在 `mod_ai_cache` 之后执行，依赖 `ClientApiKey` 完成路由查找，生成 `AiRouteResult`；缓存命中时请求已结束，该模块不再执行；
- `mod_ai_intent` 不注册请求回调，仅在 `Init()` 时注入意图解析器；路由规则求值遇到 `req_ai_intent_in` 时才触发分类；
- `mod_traffic_mirror` 只注册 `HandleForward`，在转发循环内按集群 attempt 异步镜像，不影响主路径转发；
- `mod_ai_token_auth` 的目标模型校验、`mod_ai_rate_limit` 与 `mod_ai_context` 在 `HandleAfterAITargetModel` 阶段执行，此时路由已完成、`AiBasicInfo.TargetModel` 已按路由目标模型覆盖、前缀裁剪与集群 `ModelMapping` 映射解析完毕，三者按该目标模型执行白名单校验、TPM/RPM/并发限流与上下文压缩。

`mod_body_process` 主要在 `HandleReadResponse` 阶段执行，负责解析流式响应中的 Token 用量，其计算结果会供 `mod_ai_token_auth` 在 `HandleRequestFinish` 阶段进行最终配额扣减；同阶段的 `mod_ai_cache`（未命中回写）与 `mod_ai_context`（压缩注解）只做响应侧处理，不影响配额数据。任何顺序调整都会破坏这一依赖链，导致路由、缓存、限流或配额扣减行为异常。因此 `bfe_modules/bfe_modules.go` 中的注册位置与注释需要同步维护。

### 协议适配层（bfe_model_protocol）

AI 网关需要与多种大模型协议对接：OpenAI 兼容族（OpenAI、DeepSeek、Groq、Responses API 等）使用 `Authorization: Bearer` 认证，Anthropic（Claude）使用 `x-api-key` 认证并要求携带 `anthropic-version` 头，不同协议的 usage 字段路径也各不相同。在协议适配层承担这些职责之前，这些协议知识散落在四个包中：协议识别硬编码在 `bfe_basic`、认证头注入写在 `mod_ai_token_auth.SetApiKey`、版本头注入硬编码在 `bfe_server/reverseproxy.go`、usage 字段归一在 `mod_ai_token_auth` 与 `mod_body_process` 中各存在一份重复实现。

协议适配层由 `bfe/bfe_model_protocol/` 包实现（与 `bfe_modules/` 平级），把全部协议知识收敛为按协议组织的适配器：

```
bfe/bfe_model_protocol/
├── protocol.go          # ProtocolAdapter 接口 + 协议常量
├── registry.go          # 编译期注册 + Get / Supports / ValidateProtocols
├── usage.go             # UsageFields 等类型的根包 alias（实体在 utils）
├── errors.go            # ProtocolError / ErrorNormalizer 的根包 alias
├── detect.go            # 协议识别（DetectProtocol / DetectProtocolAndKey）
├── utils/               # 叶子包：UsageFields、usage 解析、ProtocolError
├── openai/              # openai 兼容协议族（含 DeepSeek/Groq/Responses）
└── anthropic/           # anthropic messages 协议
```

每种协议由一个无状态单例适配器承载全部协议知识，接口定义在 `bfe/bfe_model_protocol/protocol.go`：

```go
type ProtocolAdapter interface {
    Key() string

    // InjectAuth 向出站请求注入上游凭证
    InjectAuth(outreq *bfe_http.Request, key string) error

    // ExtraHeaders 协议级补充 header（如 anthropic-version），无则返回 nil
    ExtraHeaders() map[string]string

    // ExtractUsageFields 从响应体/SSE 事件数据中提取 usage 字段
    ExtractUsageFields(data []byte) UsageFields

    // ErrorNormalizer 上游错误归一化；默认实现恒返回 nil（走现有状态码白名单）
    ErrorNormalizer() ErrorNormalizer
}
```

内置适配器在 `registry.go` 的 `init()` 中编译期注册（`Register(openai.New())`、`Register(anthropic.New())`、`Register(gemini.New())`），对外提供三个函数：

- `Get(protocol)`：按协议名取适配器，未知或空协议名回退 openai 适配器，保持历史兜底行为；
- `Supports(protocols, p)`：判断协议是否在 cluster 的 `ModelProtocols` 列表中，空列表默认仅 openai（向后兼容）；
- `ValidateProtocols(protocols)`：校验列表中的协议名是否都为注册表已知，含未知协议名返回 error，空列表合法。

协议选择是**按请求**进行的：`doSingleAIForward()` 根据请求的 `AiBasicInfo.AuthStyle` 取对应适配器，cluster 可以配置 `["openai", "anthropic", "gemini"]` 同时支持多种协议。

需要特别强调的概念边界是 **model_protocol（协议知识维度）≠ provider（用户自定义实体）**：provider 是 cluster 级的上游实体（名称、key 池、价格表、地址），由控制面 AI Gateway API 管理；model_protocol 描述的是认证头如何注入、补充什么版本头、usage 字段长什么样等协议知识。OpenAI 兼容生态中的 Groq、DeepSeek、OpenRouter 等品牌都走 `openai` 协议，接入这些 provider 无需新建适配器，只需在 cluster 配置 `model_protocols: ["openai"]` 即可，零代码改动。

适配层在请求生命周期中的挂载点如下：

```
http_conn.serveRequest()
  → GetApiKey / DetectAuthStyle（bfe_basic，内部委托 detect.go）   ← 协议识别
  → reverseproxy.doSingleAIForward
      → Supports(ModelProtocols, AuthStyle)                         ← 协议校验
      → adapter.InjectAuth + ExtraHeaders                           ← 认证头/版本头注入
  → mod_body_process QuotaUsageProcessor
      → SSEEvent/RawEvent.GetQuotaUsage（经 extractUsageFields）     ← 流式 usage
  → mod_ai_token_auth
      → UpdateCtxByUsage（按 AuthStyle 取适配器 ExtractUsageFields） ← 非流式 usage
```

为保证依赖方向不出现环，`bfe_model_protocol` 只允许 import `bfe_http` 与 gjson，不许 import `bfe_basic` / `bfe_config` / `bfe_modules`；`bfe_basic.GetApiKey` / `DetectAuthStyle`、`mod_ai_token_auth.SetApiKey` 等历史入口保留原签名，内部委托给适配层，因此对既有模块行为完全兼容。模型名改写（prefix 剥离、`ModelMapping`）属于协议无关逻辑，仍留在 `computeTargetModel`，不在适配层职责范围内。

**出站凭证头互斥**：协议适配器的 `InjectAuth` 只负责注入本次使用的凭证头，清理由转发层在 header 副本上完成。当网关向上游注入凭证时，出站请求只保留本次注入的那一个凭证头，客户端带来的 `Authorization`、`x-api-key`、`x-goog-api-key`（注册表内各协议 `CredentialHeader` 的并集）一律删除，避免上游同时收到网关凭证与客户端凭证而按错误的那个鉴权。删除只作用于出站 header 副本，入站请求的 header 不被改写；未配置 Keys 的透传集群既不清理也不注入，客户端凭证原样转发。

### 模块协作关系

```
                    ┌─────────────────┐
                    │   HTTP Request  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
    ┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐
    │ mod_ai_token_auth│ │ mod_ai_cache│ │  mod_ai_route   │
    │  API-Key 鉴权    │ │ 缓存查找/回写 │ │   路由查找      │
    └────────┬────────┘ └──────┬──────┘ └────────┬────────┘
             │                 │                 │
             ▼                 ▼                 ▼
    ┌─────────────────────────────────────────────────────┐
    │          AiBasicInfo / Request.Context              │
    │  - ClientApiKey / ClientKeyId                       │
    │  - ClientModel / TargetModel                        │
    │  - AiCacheStatus（缓存命中状态）                     │
    │  - AiIntent（懒解析意图结果）                        │
    │  - QuotaPlan / TokenUsage                           │
    │  - AiRouteResult (targets / fallbacks)              │
    └────────────────────────┬────────────────────────────┘
                             │
                             ▼
                 ┌───────────────────────┐
                 │ ReverseProxy.ServeHTTPForAI │
                 │  HandleAfterAITargetModel： │
                 │   mod_ai_token_auth（校验） │
                 │   mod_ai_rate_limit         │
                 │   mod_ai_context（压缩）    │
                 │  HandleForward：            │
                 │   mod_traffic_mirror        │
                 └───────────────────────┘
```

`mod_ai_intent` 不注册请求回调，仅在规则求值 `req_ai_intent_in` 时通过 `AiBasicInfo` 上的 `AiIntent` 提供意图结果，因此未在上图中单列。

### mod_ai_route：AI 路由模块

`mod_ai_route` 根据 AI 路由规则将请求路由到不同后端集群与模型。它支持按 `apikey → entity → global` 三级优先级组织的规则表，每张表由 `ApikeyRouteTableBindings` 与 API-Key 绑定。命中后返回 `targets` 列表（含 `ClusterName`、`Model`、`Weight`）与 `fallbacks` 列表。

路由查找在 `HandleFoundProduct` 回调中完成。模块从 `AiBasicInfo.ClientApiKey` 获取 API-Key，按绑定顺序查找路由表，命中后将 `AiRouteResult` 写入请求上下文。`AiRouteResult` 的定义位于 `bfe_basic/request_ai_route.go`：

```go
type AiRouteResult struct {
    RouteType string   // apikey / entity / global
    Owner     string   // 路由表属主
    RuleName  string   // 命中规则名
    Targets   []AiRouteTarget
    Fallbacks []AiRouteFallback
}
```

`mod_ai_route` 本身不执行加权选择，也不负责实际转发，只负责把路由结果写入上下文，供 `ServeHTTPForAI()` 使用。

### mod_ai_token_auth：API-Key 鉴权模块

`mod_ai_token_auth` 负责验证请求携带的 API-Key。请求需在 `Authorization` Header 中按如下格式携带 token：

```
Authorization: Bearer <api-key>
```

模块的鉴权流程包括：

1. 从请求中提取 API-Key；
2. 验证 API-Key 的有效性与状态；
3. 校验来源 IP 是否命中允许子网；
4. 检查关联配额计划是否有足够配额；
5. 请求完成后，从响应体中提取 token 使用量并扣除配额。

其中前 4 步在 `HandleFoundProduct` 阶段完成；第 5 步在 `HandleRequestFinish` 阶段完成。模型白名单/黑名单不在鉴权期校验：`ServeHTTPForAI()` 的每次集群 attempt 在调用后端之前触发 `HandleAfterAITargetModel` 回调，`mod_ai_token_auth` 的 `targetModelCheckFilter` 调用 `ValidateTargetModel`，对路由目标模型覆盖、前缀裁剪与集群 `ModelMapping` 映射之后的最终目标模型（`AiBasicInfo.TargetModel`）做白名单/黑名单匹配，不匹配返回 400 `CodeModelNotAllowed` 并结束该次 attempt。模块会将 `ClientApiKey` 与配额计划写入 `AiBasicInfo`，供后续模块使用。

### mod_ai_rate_limit：限流模块

`mod_ai_rate_limit` 对 AI 请求执行分布式限流。它支持基于 Redis 的限流策略，可按产品、API-Key 等维度配置：

- TPM（Tokens Per Minute）：每分钟 token 数上限；
- RPM（Requests Per Minute）：每分钟请求数上限；
- 最大并发数限制。

该模块注册在 `HandleAfterAITargetModel` 回调点（目标模型解析后、转发前，按集群 attempt 触发），依赖 `mod_ai_token_auth` 设置的 `ClientApiKey` 与路由阶段解析出的目标模型识别限流维度，TPM/RPM/最大并发均按转发后目标模型匹配策略。若请求触发限流，模块返回 429 响应并结束该次 attempt；本地限流通过 `bfe_basic.ErrAiRateLimit` 错误标记区分于上游 429，阻断 API-Key 轮换与集群 fallback。

### mod_body_process：请求/响应体处理模块

`mod_body_process` 主要负责响应体解析，运行在 `HandleReadResponse` 回调点。对于流式响应（SSE），它负责从响应流中解析 token 使用量，并将统计结果写入 `AiBasicInfo`，供 `mod_ai_token_auth` 在 `HandleRequestFinish` 阶段进行最终配额扣除。

`mod_body_process` 的加载与顺序对 RMB 配额扣减至关重要。若修改了配额扣除逻辑，需要确保流式响应场景在 `mod_body_process` 加载时仍能正确工作。

### mod_ai_cache：AI 缓存模块

`mod_ai_cache` 为 AI 请求提供精确匹配与语义匹配两级缓存。模块注册在 `HandleFoundProduct` 回调点，位于 `mod_ai_token_auth` 与 `mod_ai_route` 之间：鉴权完成后即查找缓存，命中请求直接返回缓存响应（`BfeHandlerFinish` 短路），不再执行路由查找，由 `req_ai_intent_in` 触发的意图懒解析也随之跳过；未命中则保存 miss 上下文继续转发，在 `HandleReadResponse` 阶段回写缓存。命中请求跳过配额结算，`mod_ai_token_auth` 依据 `AiBasicInfo.AiCacheHit` 跳过后续扣减。

缓存键格式为 `{CacheKeyPrefix}:{key_id|unknown}:{sha256(question) 前 16 字节 hex}`，租户标识取自 `AiBasicInfo.ClientKeyId`，未鉴权请求回退到 `unknown` 键空间，不同租户互不可见。请求头 `x-bfe-skip-ai-cache: on` 令请求整体跳过缓存（不读不写）；命中响应携带 `X-Bfe-Ai-Cache: hit` 响应头。

规则文件 `ai_cache.data`（`mod_ai_cache.conf` 的 `ProductRulePath` 指向该文件）按产品线组织规则列表，主要字段：

| 字段 | 说明 |
|------|------|
| `cond` | 命中条件；引用 `req_ai_intent_in` 时加载期输出 WARN 但仍接受（缓存先于路由执行，意图条件仅在未命中路径生效） |
| `cacheKeyStrategy` | 取键策略：`lastQuestion`（默认，取最后一个 user 问题）/ `allQuestions`（全部 user 问题拼接）/ `disabled` |
| `cacheTTL` | 缓存过期时间（秒），缺省取模块配置 `DefaultCacheTTL`（3600） |
| `cacheKeyFrom` / `cacheValueFrom` | GJSON 提取路径覆盖，默认 `messages.@reverse.0.content` / `choices.0.message.content` |
| `responseTemplate` / `streamResponseTemplate` | 命中响应模板，`%s` 为缓存内容占位符 |
| `maxBodyBytes` / `maxValueBytes` | 请求体与缓存值大小上限（默认各 1MB） |
| `enableSemanticCache` | 规则级语义缓存开关（默认 false，`disabled` 策略下被忽略） |

模块配置 `mod_ai_cache.conf` 的 `[basic]` 段提供 `ProductRulePath`、`CacheKeyPrefix`（默认 `ai_cache`）与 `DefaultCacheTTL`，`[redis]` 段配置缓存存储。语义缓存由 `[embedding]`（OpenAI 兼容 `POST /v1/embeddings` 服务，`timeoutMs` 默认 500）与 `[vector]`（Chroma 向量库，`collection` 默认 `ai_cache_semantic`，`timeoutMs` 默认 300）成对配置，二者缺失任一并或初始化失败时整体降级为纯精确缓存；规则文件顶层 `Semantic` 块配置 `topK`（1-10，默认 1）、`threshold`（0-2，默认 0.15）与 `thresholdRelation`（`lt|lte|gt|gte`，默认 `lt`）。语义命中响应头为 `X-Bfe-Ai-Cache: hit_semantic`，计费跳过与精确命中一致。回写时 Redis 以 SETEX 同步写入，向量库异步 upsert；整体 fail-open，Redis 故障时请求照常转发。

### mod_ai_intent：AI 意图模块

`mod_ai_intent` 为语义路由提供意图分类能力，采用懒解析设计：模块 `Init()` 只通过 `bfe_basic.SetAiIntentResolver` 把解析器与置信门限注入 `bfe_basic`，不注册任何请求回调；直到规则求值首次引用 `req_ai_intent_in` 时（典型场景是 `mod_ai_route` 的路由规则求值），解析才被触发。不含意图条件的路由规则求值零开销。

问题集文件 `intent_questions.data` 由 `Version`、全局 `MinConfidence`（默认 0.6）与 `Questions` 数组组成；问题分 `choice`（`Criteria` 选项映射）与 `score`（`Levels` 有序档位）两种类型，可携带问题级 `MinConfidence` 覆盖全局门限。`Questions` 为空数组是合法的软开关：停用意图分类，所有 `req_ai_intent_in` 条件不匹配，流量回落默认路由。文件支持热加载，版本不变时为 no-op。

分类调用决策服务的 System One 协议（`POST /v1/systemone`，超时默认 300ms），被分类文本按 `MaxStateChars`（默认 2000 字符）截断。客户端可通过显式意图头 `X-AI-Intent: <question>=<option>`（多题以分号分隔）直接声明意图，优先级高于模型分类，且该路径不产生分类调用。解析结果经进程内 LRU 缓存（默认 10000 条、1800 秒），决策服务连续失败触发熔断（默认失败阈值 5、探测间隔 5000ms）。决策服务不可用、置信度低于门限或问题未配置时 `req_ai_intent_in` 恒不匹配；引用不存在的问题名按 fail-safe 处理，永不命中。

### mod_ai_context：AI 上下文压缩模块

`mod_ai_context` 在转发前按目标模型的 token 预算压缩请求上下文。模块注册两个回调点：`HandleAfterAITargetModel`（目标模型解析后、转发前，按集群 attempt 触发）执行压缩，`HandleReadResponse` 为压缩过的响应注入注解头。`AiBasicInfo.ContextCompressStatus` 是幂等守卫：回调随 fallback 每次 attempt 触发，但压缩只执行一次，key 轮换与降级不会重复压缩。

预算 = 上下文窗口 − 输出预留：窗口按目标模型名启发（gemini 100 万、claude 20 万、codex 40 万，默认 12.8 万 token），`reserveTokens` 为 0 时自动取 `clamp(窗口×15%, 256, 16000)`；规则可覆盖 `maxContextTokens` 与 `reserveTokens`。估计 token 数超过 `budget × triggerRatio`（默认 0.7）才触发压缩。降级管线依次为：L1 工具结果截断（默认 2000 字符）→ 仅保留最新 2 张内联图 → L2 thinking 块删除（`thinkingPolicy=trim-all-but-last|keep`）；`conservative` 模式止步于此，`balanced`/`aggressive` 继续进入 P2 规则改写（`rewrite.strength=lite|full`），改写结果经保真度门校验（受保护 token 存活率不低于 `protectedSurvivalRate`，默认 0.95），不达标则丢弃改写产物、回退 trim 结果；最后 `repairMessages` 校正 tool-call 配对，不合法则回滚原文。任何异常 fail-open 放行原文。压缩只改写转发用 `OutRequest`，原始请求不动，访问日志与计费仍按原请求统计；压缩过的响应注入 `x-ai-context-compression: tokens=%d->%d; mode=%s` 注解头。压缩状态共 8 个取值：`skip_no_rule`、`skip_protocol`、`skip_body_incomplete`、`skip_parse_err`、`skip_under_threshold`、`trim`、`rewrite`、`repair_rollback`。模块 conf 仅引导配置（`ProductRulePath`），业务调参全部位于规则文件顶层 `Defaults` 块，随规则文件热加载。

### mod_traffic_mirror：流量镜像模块

`mod_traffic_mirror` 把符合规则的 AI 请求异步镜像到影子集群，用于新版本灰度比对与阴影测试。模块只注册 `HandleForward` 回调（转发前，按集群 attempt 触发），镜像的匹配、快照与提交全部异步执行，绝不阻塞主路径。`Request.Context` 中的 `CtxMirrored` once 标记保证 AI fallback 重试重新进入 `HandleForward` 时不会重复镜像。

镜像规则字段包括：`cond`（本模块特例——为空表示匹配所有请求）、`percentage`（0-100 采样）、`removeHeaders`（敏感头剥离）、`setHeaders`（注入头）、`bodyRewrites`（一期仅支持 `path="model"` 改写模型字段）与 `pathRewrite`。镜像请求默认注入 `X-Bfe-Mirror: true` 标识头。镜像后端挑选使用 `bfe_balance.BalanceGslb.PickBackend()`——这是 bfe_balance 提供的无副作用挑选入口（不读不改请求、无会话保持与重试副作用），`bfe_server` 启动时通过 `SetGlobalBalTable` 进程级注册平衡表。

模块 conf 键包括超时（`ConnectTimeoutMs` 默认 2000、`TTFBTimeoutMs` 默认 30000、`TotalTimeoutMs` 默认 600000）、体积上限（`MaxMirrorBodyBytes` 默认 2MB、`MaxResponseBodyBytes` 默认 16MB）、并发与队列（`MaxConcurrent` 默认 1024、`QueueCapacity` 默认 4096）与熔断（`CircuitBreakerFailThreshold` 默认 50、`CircuitBreakerCooldownSec` 默认 30）。镜像结果（usage/error/finish_reason 解析）只写入模块私有 Prometheus registry，不进访问日志；访问日志仅同步记录 `mirror_hit`（842）、`mirror_cluster`（843）两个字段。

## 模块框架与回调机制（bfe_module）

BFE 的模块框架定义在 `bfe_module/` 目录下，核心抽象是 `BfeModule` 接口与 `BfeCallbacks` 回调管理器。每个模块通过 `Init()` 方法将自身处理函数注册到指定的回调点，BFE 在请求处理过程中按顺序调用这些回调。

### 回调点

常用的回调点包括：

| 回调点 | 触发时机 | AI 相关模块 |
|--------|----------|-------------|
| `HandleBeforeLocation` | 租户识别之前 | mod_trust_clientip、mod_logid 等 |
| `HandleFoundProduct` | 租户识别之后 | mod_ai_token_auth、mod_ai_cache（缓存查找，命中短路）、mod_ai_route |
| `HandleAfterLocation` | 位置/路由确定之后 | mod_body_process 等 |
| `HandleAfterAITargetModel` | 目标模型解析后、转发前，按集群 attempt 触发 | mod_ai_token_auth（目标模型校验）、mod_ai_rate_limit、mod_ai_context（上下文压缩） |
| `HandleForward` | 转发前，按集群 attempt 触发 | mod_traffic_mirror（流量镜像） |
| `HandleReadResponse` | 读取后端响应时 | mod_ai_cache（未命中回写）、mod_ai_token_auth（usage 解析）、mod_ai_context（压缩注解）、mod_body_process |
| `HandleRequestFinish` | 请求处理完成时 | mod_ai_token_auth（配额扣除，缓存命中跳过） |

AI 网关访问日志（`mod_access_pb3`）在上述链路中同步记录缓存命中状态（`ai_cache_status`、`ai_cache_key`）、语义缓存（`ai_cache_semantic`、`ai_cache_similarity`）、上下文压缩（`ai_context_compress_status`、`ai_context_tokens_before/after`、`ai_context_compress_mode`）、意图分类（`ai_intent_*` 七字段）与流量镜像（`mirror_hit`、`mirror_cluster`）等字段，供报表与排障消费；字段编号与完整字段表见[第十五章 可观测设计](./chapter15-observability.md)。

### 回调返回值

模块回调函数返回一个 `int` 状态与可选的 `*bfe_http.Response`：

- `BfeHandlerGoOn`：继续执行后续回调；
- `BfeHandlerFinish`：结束请求，返回错误响应；
- `BfeHandlerResponse`：直接跳过后续处理，进入响应发送阶段；
- `BfeHandlerClose`：直接关闭连接；
- `BfeHandlerRedirect`：返回重定向响应。

AI 相关模块在 `HandleFoundProduct` 阶段通常返回 `BfeHandlerGoOn`，将状态写入上下文；若鉴权失败或触发限流，则返回 `BfeHandlerFinish` 或 `BfeHandlerResponse`；`mod_ai_cache` 在缓存命中时同样以 `BfeHandlerFinish` 结束请求并返回缓存响应。`HandleAfterAITargetModel` 阶段的消费者同样遵循这一约定：目标模型校验失败返回 400，限流命中返回 429。

### 模块注册

新增模块的标准步骤如下：

1. 在 `bfe_modules/mod_<name>/` 下创建模块包；
2. 实现 `BfeModule` 接口及所需回调；
3. 在 `bfe_config/` 下新增配置加载器；
4. 在 `bfe_modules/bfe_modules.go` 的 `moduleList` 中按正确顺序注册；
5. 在 `conf/` 下提供默认配置与文档。

模块顺序非常重要。例如 `mod_ai_route` 必须排在 `mod_ai_token_auth` 之后，否则 `ClientApiKey` 尚未设置，路由查找将无法进行。

## 配置加载与热加载机制

BFE 的配置体系分为服务器配置、路由配置、集群配置、TLS 配置与模块配置。AI 相关模块的配置加载遵循统一模式：

1. **模块基础配置**：`conf/mod_<name>/mod_<name>.conf`，采用 INI 格式，指定数据文件路径与日志开关；
2. **模块数据文件**：`conf/mod_<name>/<data>.data`，采用 JSON 格式，存放实际规则数据；
3. **加载入口**：模块的 `Init()` 调用 `ConfLoad()` 加载基础配置，再调用数据加载函数反序列化并校验规则；
4. **热加载**：通过 Web 监控接口触发，重新加载数据文件并原子替换内存结构。

以 `mod_ai_route` 为例，启动时 `Init()` 会：

1. 调用 `ConfLoad()` 读取 `conf/mod_ai_route/mod_ai_route.conf`；
2. 调用 `AiRouteDataLoad()` 读取 `conf/mod_ai_route/ai_route.data`；
3. 调用 `ValidateRouteTable()` 校验每张路由表的类型、规则名、条件表达式、target 权重；
4. 通过 `routeTable.Update()` 原子替换内存中的路由表。

热加载接口为：

```
GET /reload/mod_ai_route
```

该接口调用 `loadRouteRuleConf()`，先在校验阶段生成新的路由表副本，校验通过后再加锁替换，确保失败时不影响当前运行中的规则。`mod_ai_token_auth` 与 `mod_ai_rate_limit` 也提供类似的热加载接口。

热加载的安全性体现在两个层面：一是配置语法与语义校验在替换前完成，避免非法配置进入内存；二是 `AiRouteTable.Update()` 使用读写锁保护，查找操作在 `RLock` 下读取旧表引用，更新操作在 `Lock` 下替换引用，二者互不阻塞。对于路由规则，条件表达式会在加载时由 `condition.Build()` 编译为可执行的 `Condition` 对象，并在 `ValidateRouteTable()` 中校验 target 权重总和为 100，确保运行时无需重复编译。

### ModelProtocols 启动与热加载校验

cluster 配置的 `AIConf.ModelProtocols` 字段声明该集群支持的模型协议列表，空列表默认 `["openai"]`。协议名会在配置加载期接受注册表校验：`bfe_server/bfe_confdata_load.go` 中的 `validateClusterModelProtocols()` 遍历每个 cluster 的 `AIConf.ModelProtocols`，调用 `modelprotocol.ValidateProtocols()` 检查协议名是否已知。

该校验挂在两个路径上：

- 启动时 `InitDataLoad()` 在校验失败时返回错误，BFE 启动失败；
- 热加载时 `serverDataConfReload()` 在校验失败时记录错误日志并拒绝切换，当前运行中的配置保持不变。

这样未知协议名的配置错误在启动或热加载阶段即可暴露，而不是在请求转发时才失败。

## 关键能力

BFE 在既有架构之上，沿"表达能力、计费完备性、上游契约适配、企业级 TLS"四条主线提供以下能力，均由控制面 API 下发配置、控制台提供表单。

### 条件表达式引擎（expr）

路由条件与模块规则支持两套并存语法：BFE 自研原语（`req_host_in`、`res_code_in` 等 53 个），以及基于 expr-lang/expr 的条件引擎（`bfe_expr` 包，旧原语经兼容层函数化后在其中可用）：

- `bfe.conf [Server]` 的 `ConditionEngineMode` 开关，三模式：`LegacyFirst`（默认，旧引擎优先、失败回退 expr）、`ExprCompatible`（expr 与旧原语混写，编译失败即报错）、`ExprOnly`（纯 expr，旧原语名视为未知函数）；兼容数字 `0/1/2` 与大小写，非法值启动报错；模式为进程级，**热更新不生效，改模式须重启**；
- `mod_redirect`、`mod_header` 等共享动作的动作参数支持 `=expr` 前缀（加载期编译 + 类型检查，结果必须为 string），与 `==字面量`（转义）、普通字面量三态并存；运行期求值失败该参数置空并打 warn，不阻断同动作其他参数；
- 动态数据源：`RedisEnv` 真实可用（`Get`/`GetInt`/`HGetAll`，miss 与错误返回零值不报错），`HttpEnv`/`GlbEnv`/`ConfEnv` 暂为接口占位。

expr 引擎的编译全部发生在配置加载期，运行期只执行求值；表达式出错按 `false` + warn 处理，请求不失败。默认 `LegacyFirst` 与传统原语行为逐字节一致；两套引擎存在有意差异（字符串转义规则、`res_code_in` 字符串比较 vs `ResCode()` 整型比较、哈希空输入的桶稳定性），仅在 `ExprCompatible`/`ExprOnly` 下出现。

### 缺价处置（PriceMissAction）

模型价格未配置时的行为由集群级显式三态配置 `AIConf.PriceMissAction` 决定：`fallback_chat`（缺省，按同模型 chat 价计费）、`bill_zero`（转发但按 0 计费）、`reject`（**转发前拒绝**，503 `PRICE_NOT_CONFIGURED`，不调用上游、不计费但访问日志照常入库）；`ModelTable` 整体缺失视同未定价。信号经访问日志 `ai_price_miss` / `ai_price_warn`（六种形态）全链路上报（详见 [第十五章 可观测性设计](./chapter15-observability.md)）。

### 思考模式推理内容补齐（EnsureReasoningContent）

部分思考模式上游要求回传 `reasoning_content`（缺字段或 `null` 直接 400），而路由切换后的首个请求 assistant 消息来自上一个后端。`AIConf.EnsureReasoningContent`（协议列表，空 = 关闭，仅 `openai`/`anthropic`）在 `doSingleAIForward` 中对校验窗口内的 assistant 消息定点补齐（sjson 写入，字段序与其余字节不变、已有非空值不动）；非 JSON、压缩请求体、批量/文件、`/responses`、超限请求体均原样透传。配套计数 `ReasoningPatched` / `ReasoningSkip` / `ReasoningFail` / `ReasoningStill400`，其中 `ReasoningStill400` 是"上游契约变更"的退出信号。

### Key 惩罚时长可配置

Key 亲和惩罚 TTL 是集群级配置 `AIConf.KeyPolicy.SessionAffinityPenaltyTTL429` / `SessionAffinityPenaltyTTLAuth`（默认 **60 / 300**，`0` 表示用默认值而非关闭，负数报错、`>86400` 截断并 Warn）；缩短配置不追改已存在的 Redis 惩罚键，批量 404 的惩罚固定 60s 不受其影响。详见 [第九章 Entity 与 API-Key 设计](./chapter09-apikey-design.md)。

### 配额欠费模型

配额扣减采用全额 `DECRBY`，余额**允许为负**（欠费如实记录）；鉴权期对负余额判无余额（429）。控制台"已用额度"可大于配额上限，对外 `Remaining` 仍展示 0。详见 [第十二章 配额与限流设计](./chapter12-quota-and-rate-limit.md)。

### 计费与用量采集完备性

- **压缩流式 usage 采集**：上游按 `Accept-Encoding` 压缩 SSE 时不再旁路为逐字节透传——`CompressedStreamDecoder` 原样转发压缩字节、解压副本仅供 usage 解析（设明文预算防解压炸弹）；`bfe.conf [Server]` 的 `AIUpstreamAcceptEncoding`（`auto` 默认 / `identity` / `off`）收窄出站协商到网关可解码集合；
- **客户端中断与上游断流**：`EstimateToken=true` 时客户端流式中断按已转发内容估算计费（`ai_usage_source=aborted`）；上游中途断流区分归因（`upstream_dropped`）并默认补发协议错误事件（`NormalizeUpstreamError.StreamTruncatedError` 可关）；
- **非 2xx 行记录用量**：`ai_total_tokens` 口径统一为 `in + out`，非 2xx 行也记 in/out/total（`ai_usage_source=failed`），`ai_cost_value` 仅 2xx 写出。

### EPP 准入拒绝透传

EPP 经 ext_proc 返回 `ImmediateResponse`（429/503）时，映射为 429 `EPP_OVERLOAD` / 503 `EPP_NO_CAPACITY` 并透传 `Retry-After`；**不重试、不回退本地均衡、不喂熔断**，过载场景表现为部分请求收到 429/503，而非全部放行排队。

### TLS 栈（升级前必读）

BFE 的 TLS 基于标准库 `crypto/tls`（经 `GetCertificate` / `GetConfigForClient` / `VerifyConnection` 回调接入证书与校验逻辑），**支持 TLS 1.3**：

- `HttpsBasic.MaxTlsVersion` 默认 `VersionTLS13`、`MinTlsVersion` 默认 `VersionTLS10`——从旧版本升级且未显式配置时，客户端可能协商到 TLS 1.3；
- `CipherSuites` 由"配置行序即优先级"退化为**纯白名单**（实际顺序由标准库固定表决定），`PreferServerCipherSuites` 成为空操作；
- 废弃字段解析保留、行为忽略：`EnableSslv2ClientHello`、`[SessionCache]` 全部字段（无 Redis TLS 会话缓存，会话恢复为进程内 session ticket）、`SessionTicketKeyName`、tls_rule_conf 的 `Grade` / `Chacha20` / `DynamicRecord`；
- **会导致配置加载失败**：标准库 TLS 不提供 `mod_header` 内置变量 `tls_ja3_raw` / `tls_ja3_hash`，引用它们的 `header_rule.data` 将整文件拒载（从旧版本升级前须清理引用）；文本会话日志不含 `ses_tls_client_random` / `ses_tls_server_random`，pb3 `HandshakeTime` 不再写入。

## 关键配置示例

### 启用 AI 网关模式

在 `bfe/conf/bfe.conf` 中开启 AI 网关：

```ini
[server]
ai_gateway_enabled = true
```

### mod_ai_route 基础配置

`conf/mod_ai_route/mod_ai_route.conf`：

```ini
[basic]
RouteRulePath = ../conf/mod_ai_route/ai_route.data

[log]
OpenDebug = false
```

### mod_ai_route 路由规则

`conf/mod_ai_route/ai_route.data`：

```json
{
    "Version": "20260718131505",
    "route_rules": {
        "apikey_ak_user_a": {
            "type": "apikey",
            "owner": "ak_user_a",
            "rules": [
                {
                    "name": "user_a-deepseek",
                    "Cond": "req_host_in(\"api.example.org\")",
                    "targets": [
                        {
                            "ClusterName": "cluster_deepseek_a",
                            "Model": "deepseek-v4-pro",
                            "Weight": 70
                        },
                        {
                            "ClusterName": "cluster_deepseek_b",
                            "Model": "deepseek-v4-pro",
                            "Weight": 30
                        }
                    ],
                    "fallbacks": [
                        {
                            "ClusterName": "cluster_deepseek_c",
                            "Model": "deepseek-v3.2"
                        }
                    ]
                }
            ]
        },
        "entity_dept_ai": {
            "type": "entity",
            "owner": "dept_ai",
            "rules": [
                {
                    "name": "dept_ai-default",
                    "Cond": "default_t()",
                    "targets": [
                        {
                            "ClusterName": "cluster_dept_ai",
                            "Model": "",
                            "Weight": 100
                        }
                    ],
                    "fallbacks": []
                }
            ]
        },
        "global_default": {
            "type": "global",
            "owner": "global",
            "rules": [
                {
                    "name": "global-default",
                    "Cond": "default_t()",
                    "targets": [
                        {
                            "ClusterName": "cluster_global",
                            "Model": "",
                            "Weight": 100
                        }
                    ],
                    "fallbacks": []
                }
            ]
        }
    },
    "ApikeyRouteTableBindings": {
        "ak_user_a": [
            "apikey_ak_user_a",
            "entity_dept_ai",
            "global_default"
        ]
    }
}
```

上述配置表示：API-Key 为 `ak_user_a` 的请求，先查找 `apikey_ak_user_a` 路由表；未命中则查找 `entity_dept_ai`；仍未命中则使用 `global_default`。命中 `user_a-deepseek` 规则后，按 70:30 的权重选择 `cluster_deepseek_a` 或 `cluster_deepseek_b`，并在两者均失败时降级到 `cluster_deepseek_c`。

### fallback 触发条件

`ServeHTTPForAI()` 在目标转发失败时按 `fallbacks` 顺序降级。`shouldTriggerFallback()` 的判断逻辑（`bfe/bfe_server/reverseproxy.go`）为：

```go
func shouldTriggerFallback(res *bfe_http.Response, err error) bool {
    if err != nil {
        return true
    }
    code := getResponseStatus(res)

    // 协议级错误归一化挂钩（默认实现恒返回 nil，保持状态码白名单行为）
    if perr := modelprotocol.Get("").ErrorNormalizer().Normalize(code, nil, nil); perr != nil {
        return perr.IsUpstream && (perr.SwapKey || perr.Retryable)
    }

    if code >= 500 {
        return true
    }
    if _, ok := aiFallbackStatusCodes[code]; ok {
        return true
    }
    return false
}
```

触发条件为：

- `clusterInvoke()` 返回错误（连接失败、超时、读写错误等）；
- 后端返回状态码 `>= 500`；
- 后端返回状态码命中 `aiFallbackStatusCodes` 白名单（400/401/402/403/422/429）。

在状态码白名单之前，`shouldTriggerFallback()` 先咨询协议适配层的 `ErrorNormalizer().Normalize` 挂钩：若协议适配器识别出上游错误并返回 `ProtocolError`，则按归一化结果决定是否降级；挂钩的默认实现恒返回 `nil`，实际判定仍由上述状态码白名单完成。该挂钩为新协议按错误体定制降级语义预留了扩展点。

以下情况不触发 fallback：

- 后端返回的 `4xx` 状态码不在 `aiFallbackStatusCodes` 白名单中；
- 请求在 `HandleFoundProduct` 阶段鉴权失败（未进入转发循环）；
- 某次 attempt 被本地限流拒绝：`mod_ai_rate_limit` 触发时置 `bfe_basic.ErrAiRateLimit` 错误标记，转发循环识别该标记后直接终止尝试列表，既不再轮换 API-Key，也不降级到备用集群（上游返回的 429 仍按原有 key 轮换逻辑处理）。

每次 fallback 前，`resetRequestForRetry()` 会重置 `OutRequest`、backend 连接、retry 计数与错误信息，并通过 `rewindRequestBody()` 将请求体重置到起始位置，确保下一次转发使用干净的请求状态。

### fallback 执行流程

```
加权选择 target
    │
    ▼
构造尝试列表 [target] + fallbacks
    │
    ▼
prepareRequestBodyForRetry()
若请求体不可回退 → 禁用 fallback
    │
    ▼
┌─────────────────────────────────┐
│  循环遍历 attempts 列表          │
│                                 │
│  首次？ 否 → resetRequestForRetry()│
│  ClusterTable.Lookup(ClusterName)│
│  模型覆盖 / ModelMapping         │
│  clusterInvoke()                │
│                                 │
│  成功（err == nil 且 status < 500）│
│      → 跳出循环                  │
│  失败且非最后一次                │
│      → shouldTriggerFallback()?  │
│         是 → 关闭响应，继续下一个   │
│         否 → 跳出循环             │
└─────────────────────────────────┘
    │
    ▼
返回最终响应或 500
```

该流程确保只有在后端不可用或出现服务端错误时才尝试降级，而客户端错误不会触发无意义的 fallback，避免将错误请求扩散到备用集群。

### 后端不可用的错误边界

`ClusterTable.Lookup(ClusterName)` 之后，转发循环按集群的有效后端实例池挑选实例。两种失败在语义上不同，排查方向也不同：

- **500 `BK_NO_BACKEND`**：集群在 `cluster_table` 中存在，但可用后端实例池为空——例如 K8s 实例池尚未导出任何实例，或子集群实例被全部移除。响应体为 `{"error":{"code":"BK_NO_BACKEND","type":"internal_error","param":null,"message":"no available backend for this request","details":null}}`。`cluster_table` 中缺失的子集群同样按空 backend 处理；热加载释放旧实例、新实例尚未就绪的时间窗也会返回该错误。该错误反映配置或实例导出问题，应配置告警，而不是当作上游故障盲目重试。
- **502 `BACKEND_UNAVAILABLE`**：目标集群不存在（无法解析到集群配置）或后端连接失败，message 为 `upstream backend unavailable for this request`。它表示"集群不可达"，与"集群存在但没有可用实例"的 `BK_NO_BACKEND` 明确区分。

子集群没有任何 `weight > 0` 的后端时，配置加载只输出 WARN，不会导致整表加载失败；该子集群在请求期按空实例池处理，返回 `BK_NO_BACKEND`。

网关自生成的 AI 错误响应（包括上述两类，以及鉴权、限流、配额、内容审查等拒绝）统一携带 `X-Bfe-Gw-Error: 1`，并恒写 `X-Bfe-Log-Id`，用于把错误响应与访问日志、后端日志关联。`X-Bfe-Log-Id` 按回退链取值：`req.LogId` → 入站 `X-Bfe-Log-Id` 头 → 现场生成。`X-Request-ID` 只做回显：客户端携带则原样返回，未携带则不生成，避免网关伪造调用方的请求标识。

## 上游路径按协议改写（ProtocolPaths）

不同 provider 的上游路径前缀各不相同（如百炼 `/compatible-mode/v1`、火山 `/api/v3`、Kimi Code `/coding/v1`），同一 provider 的不同协议还常挂在不同前缀下。为了让客户端统一使用标准入口 `/v1/...` 发起请求，BFE 在转发循环内按 cluster 的 `AIConf.ProtocolPaths` 改写出站请求的 URL path：

- anthropic 请求：`/v1/messages` → `{ProtocolPaths[anthropic]}/v1/messages`（如 `/apps/anthropic/v1/messages`）；
- openai 请求：`/v1/chat/completions` → `{ProtocolPaths[openai]}/chat/completions`（如 `/compatible-mode/v1/chat/completions`）。

改写由 `bfe_server/ai_path_rewrite.go` 中的纯函数 `rewriteUpstreamPath` 计算，在 `doSingleAIForward` 创建出站请求拷贝之后、`clusterInvoke` 之前执行。两个协议分支的判定规则不同：

- **openai 分支**：先剥离可选的 `/v1` 版本前缀（`bfe_basic.StripV1Prefix`，`/v1/chat/completions` 与 `/chat/completions` 等价），再查 `bfe_basic` 的共享 OpenAI 端点表（`bfe_basic/openai_endpoint.go` 的 `openAIEndpointModes`，含 `/chat/completions`、`/embeddings`、`/images/generations`、`/responses`、`/video/generations` 等主要端点）：命中端点表的路径改写为 `base + 剥离后路径`（如 `/chat/completions` 与 `/v1/chat/completions` 都改写为 `/compatible-mode/v1/chat/completions`）；未命中端点表的自定义路径原样透传。路径恰好为 `/v1` 或 `/v1/` 时改写结果为 base 本身。
- **anthropic 分支**：仍只认标准入口（`/v1` 精确值或 `/v1/` 前缀），命中后拼为 `base + 原始路径`（如 `/v1/messages` → `/apps/anthropic/v1/messages`）；`/v10/xxx` 等非标准入口透传。
- **gemini 分支**：`ProtocolPaths` 只有 `openai`/`anthropic` 两个合法 key，gemini 永不改写，原生路径（如 `/v1beta/models/gemini-2.5-flash:generateContent`）始终透传。

未配置 `ProtocolPaths` 或对应协议无条目时同样原样透传。改写只落在出站拷贝上、不改入站请求，因此 fallback 的每次 attempt 都基于原始客户端路径重算——切换到不同前缀配置的备用 cluster 后，上游路径自动跟随新 cluster 的配置。

`ProtocolPaths` 由控制面按 cluster 引用的 provider 恒透传（配置入口为 Provider 的 `protocol_paths` 字段，见[第十章 Provider 与 Cluster 设计](./chapter10-provider-and-cluster.md)）。BFE 在配置加载与热加载时校验 key 白名单（`openai`/`anthropic`）与 value 格式，非法配置拒绝加载，作为控制面校验被手工配置等路径绕过时的兜底。

## batch / file 透传的模型改写边界

模型改写（请求体 `model` 字段覆盖、前缀裁剪与 `ModelMapping`）只适用于 JSON 的推理类请求。`ModeFile` / `ModeBatch` 以及非 JSON `Content-Type` 的请求跳过 `shouldRewriteBodyModel` 判定：`/v1/files` 的 multipart 请求体原样透传，`/v1/batches` 不注入 `model` 字段。批量任务与文件上传的语义完全由上游 Provider 决定，网关只做认证、路由与限额，不对请求体做模型相关改写。

## 本章小结

本章介绍了壬远 AI 网关数据面组件 BFE 的设计。

- BFE 负责实际转发 AI 请求，与控制面 AI Gateway API 通过配置分发协同工作。
- AI 网关模式下，请求进入独立的 `ServeHTTPForAI()` 路径，复用原有回调与转发基础设施。
- `mod_ai_token_auth`、`mod_ai_cache` 与 `mod_ai_route` 在 `HandleFoundProduct` 阶段按固定顺序执行，缓存命中即短路（路由与意图懒解析跳过，配额结算跳过）；目标模型白名单校验、`mod_ai_rate_limit` 与 `mod_ai_context` 注册在 `HandleAfterAITargetModel` 回调点（按集群 attempt 触发），通过 `AiBasicInfo` 与 `Request.Context` 共享状态。
- `mod_ai_intent` 以懒解析方式提供意图分类能力，不注册请求回调；`mod_traffic_mirror` 只注册 `HandleForward`，异步镜像不阻塞主路径。
- `mod_body_process` 在 `HandleReadResponse` 阶段解析 SSE 响应中的 token 使用量，供 `mod_ai_token_auth` 最终扣减配额；同阶段的 `mod_ai_cache` 回写与 `mod_ai_context` 压缩注解只影响响应侧。
- `bfe_model_protocol` 协议适配层把散落在 `bfe_basic`、`mod_ai_token_auth`、`mod_body_process`、`bfe_server` 四处的协议知识收敛为按协议组织的适配器；`AIConf.ModelProtocols` 在启动与热加载时接受注册表校验，未知协议名会使加载失败。
- BFE 的 `bfe_module` 框架通过回调点与返回值机制组织模块，`bfe_modules/bfe_modules.go` 中的注册顺序直接影响行为正确性。
- 配置加载采用 INI + JSON 双层结构，支持通过 Web 接口热加载，新配置在校验完成后原子替换旧配置。
- `mod_ai_route` 支持 `apikey → entity → global` 三级路由、`targets` 加权选择与 `fallbacks` 顺序降级，是 AI 网关转发的核心。
- 上游路径按协议改写：`AIConf.ProtocolPaths` 中 openai 分支剥离可选 `/v1` 前缀后按 `bfe_basic` 共享端点表判定改写（`/chat/completions` 与 `/v1/chat/completions` 都会拼上 provider 前缀），anthropic 分支只认标准 `/v1` 入口，gemini 永不改写；改写只作用于出站拷贝，fallback 每次 attempt 独立重算。
- 条件表达式引擎支持 expr（`ConditionEngineMode` 三模式，默认 `LegacyFirst` 与传统原语行为一致），动作参数支持 `=expr`；缺价处置三态（`PriceMissAction`）、推理内容补齐（`EnsureReasoningContent`）、Key 惩罚时长配置与配额欠费模型均为集群级配置；压缩 SSE 的 usage 采集、中断/断流计费归因、非 2xx 用量记录覆盖了计费旁路；TLS 基于标准库实现并支持 TLS 1.3，升级前须清理 JA3 变量引用并评估默认版本变化。

## 参考文档

- `bfe/AGENTS.md`
- `bfe/docs/zh_cn/modules/mod_ai_route/mod_ai_route.md`
- `bfe/docs/zh_cn/modules/mod_ai_token_auth/mod_ai_token_auth.md`
- `bfe/docs/zh_cn/modules/mod_ai_rate_limit/mod_ai_rate_limit.md`
- `bfe/docs/zh_cn/modules/mod_ai_cache/mod_ai_cache.md`
- `bfe/docs/zh_cn/sys_design/mod_ai_route.md`
- `bfe/docs/zh_cn/sys_design/mod_ai_route_bfe_changes.md`
- `bfe/docs/zh_cn/sys_design/model_protocol_adapter.md`
- `bfe/docs/zh_cn/sys_design/ai_protocol_paths.md`
- `bfe/docs/zh_cn/sys_design/ai_cache.md`
- `bfe/docs/zh_cn/sys_design/mod_ai_intent.md`
- `bfe/docs/zh_cn/sys_design/traffic_mirror.md`
