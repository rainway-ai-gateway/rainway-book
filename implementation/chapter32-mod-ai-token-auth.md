# 第三十二章 Token认证与配额模块实现：mod_ai_token_auth

## 本章目标

通过本章，读者将理解壬远 AI 网关数据面中 `mod_ai_token_auth` 模块的完整实现：

- 模块在 BFE 请求处理链路中的位置与注册时机；
- 如何校验请求携带的 API-Key，并在失败时返回结构化的错误响应；
- 如何将 API-Key 与 `QuotaPlan` 绑定，并通过 Redis 查询实时余额；
- 请求结束时如何根据实际 Token 用量或 RMB 成本扣减配额；
- 扣费幂等、客户端中断不计费与 `/count_tokens` 跳过计费的可靠性语义；
- RMB 配额如何结合 Provider 时段模板与模型分时段价格实现差异化计费；
- image/video entry 驱动计费、mode 无价格条目回落 chat 价等兜底计费规则；
- 计费可观测性：`ai_usage_source` / `ai_zero_cost_reason` / `ai_price_miss` 等访问日志新字段与 usage 识别语义；
- 批量与异步任务支持（一期）：`mod_ai_batch` 的端点识别、key 亲和与预留-结算-释放计费；
- `mod_ai_token_auth` 与 `mod_body_process` 在流式与非流式响应中的协作方式。

## mod_ai_token_auth 模块职责

`mod_ai_token_auth`（AI Token 认证模块）是数据面 BFE 中负责 API-Key 鉴权与配额扣减的核心模块。一个 API-Key 代表业务方对特定大模型服务的访问凭证，同时关联若干 `QuotaPlan`，决定该凭证在一个周期内可消耗的资源总量。

模块的核心职责包括：

1. **API-Key 提取与校验**：从请求 `Authorization: Bearer <api-key>` 头中提取 Key，校验其存在性、启用状态、过期时间、来源 IP 子网等。校验失败时立即构造结构化错误响应返回客户端，避免无效请求进入后端。
2. **配额余额预检**：在请求进入后端前，查询 Redis 中各 `QuotaPlan` 的剩余配额，余额不足或计划过期时拒绝请求。这一步只在请求成功认证后执行，未命中鉴权规则的请求不会触发配额检查。
3. **目标模型校验**：若 Token 配置了 `Models` 或 `BlockModels`，在转发阶段对最终目标模型执行白名单/黑名单校验（见下文"目标模型校验（转发期）"），不匹配返回 400 `CodeModelNotAllowed`。
4. **请求结束扣减**：在响应完成后，根据响应体中的 `usage` 字段或按内容长度估算的 Token 数，对 `total_token` 配额执行扣减；对 `RMB` 配额则先按模型单价折算成本，再执行扣减。客户端中断且未拿到最终 usage 的请求不计费，`/count_tokens` 类端点跳过计费，扣费通过 `deducted` 标记保证幂等。扣减失败会被捕获并记录 Warn 日志，不会影响响应返回。
5. **错误信息结构化**：在 `AiBasicInfo.AiAuthInfo` 中记录拒绝原因与命中的配额计划，便于访问日志与监控分析。这些字段也会被 `mod_access` 等访问日志模块输出，为后续排障提供完整上下文。

## 模块在 BFE 模块链中的位置

BFE 的模块按固定顺序在 `bfe_modules/bfe_modules.go` 中注册。`mod_ai_token_auth` 位于 `mod_unified_waf` 之后、`mod_ai_route` 之前：

```go
// bfe/bfe_modules/bfe_modules.go
var moduleList = []bfe_module.BfeModule{
    // ... 前置模块 ...

    // mod_ai_token_auth
    mod_ai_token_auth.NewModuleAITokenAuth(),

    // mod_ai_route
    // Requirement: after mod_ai_token_auth (needs ClientApiKey)
    mod_ai_route.NewModuleAiRoute(),

    // mod_body_process
    mod_body_process.NewModuleBodyProcess(),

    //depends on token calc
    mod_ai_rate_limit.NewModuleAiRateLimit(),

    // ... 后置模块 ...
}
```

之所以把 `mod_ai_token_auth` 放在 `mod_ai_route` 之前，是因为路由模块需要根据已认证的 API-Key 选择专属路由规则；而 `mod_ai_rate_limit` 位于 `mod_body_process` 之后，是因为限流模块依赖 `mod_body_process` 解析出的 Token 用量进行 TPM 计算。

`HandleRequestFinish` 是 BFE 请求生命周期的最后一个回调点，所有响应数据（包括流式累计的 Token 用量、RMB 成本）此时都已就绪，因此配额扣减放在这里可以确保只扣除一次且金额准确。如果提前到 `HandleReadResponse` 扣减，流式响应尚未结束，会导致少扣或重复扣减。

`mod_ai_token_auth` 在初始化时注册了四个回调：

- `HandleFoundProduct`：`tokenFoundProductHandler`，完成 API-Key 校验与配额预检；
- `HandleAfterAITargetModel`：`targetModelCheckFilter`，在目标模型解析后、转发前按集群 attempt 对最终目标模型执行白名单/黑名单校验；
- `HandleReadResponse`：`tokenReadResponseHandler`，非流式场景下解析响应体中的 Token 用量；
- `HandleRequestFinish`：`tokenRequestFinishHandler`，在所有响应处理完成后执行最终配额扣减。

其中 `HandleAfterAITargetModel` 上 `mod_ai_token_auth` 先于 `mod_ai_rate_limit` 注册，保证模型白名单校验始终先于限流执行。

```mermaid
flowchart LR
    A[请求进入] --> B[mod_unified_waf]
    B --> C[mod_ai_token_auth<br/>HandleFoundProduct]
    C --> D[mod_ai_route]
    D --> E[目标模型解析<br/>doSingleAIForward]
    E --> F[mod_ai_token_auth<br/>HandleAfterAITargetModel<br/>目标模型校验]
    F --> G[mod_ai_rate_limit<br/>HandleAfterAITargetModel]
    G --> H[后端模型服务]
    H --> I[mod_body_process<br/>HandleReadResponse]
    I --> J[mod_ai_token_auth<br/>HandleReadResponse]
    J --> K[mod_ai_token_auth<br/>HandleRequestFinish]
    K --> L[返回客户端]
```

## API-Key 校验流程

请求头中的 API-Key 必须遵循以下格式：

```http
Authorization: Bearer <api-key>
```

`tokenFoundProductHandler` 是模块的入口，其处理逻辑位于 `bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`：

```go
func (m *ModuleAITokenAuth) tokenFoundProductHandler(req *bfe_basic.Request) (int, *bfe_http.Response) {
    meta := req.GetAiBasicInfo()
    if meta == nil {
        return bfe_module.BfeHandlerGoOn, nil
    }

    matched := m.matchTokenRule(req)
    if !matched {
        return bfe_module.BfeHandlerGoOn, nil
    }

    m.state.ReqAuth.Inc(1)
    tok, err := m.ValidateUserTokenByReq(req)
    if err != nil {
        m.state.ReqAuthFail.Inc(1)
        resp := err.CreateErrorResponse(req)
        return bfe_module.BfeHandlerResponse, resp
    }

    promptToken := 0
    if meta.IsAllowEstimateToken() {
        promptToken = int(GetPromptToken(req))
    }
    SetTokenAuthContext(req, tok, int64(promptToken), tok.Tags)

    return bfe_module.BfeHandlerGoOn, nil
}
```

`matchTokenRule` 根据产品（Product）匹配规则表，只有命中 `CHECK_TOKEN` 动作的规则才会触发后续鉴权。这种设计让管理员可以按产品维度灵活开启或关闭鉴权，例如对内部测试产品放行，对线上产品强制鉴权。

真正的校验逻辑在 `bfe/bfe_modules/mod_ai_token_auth/token_rule_table.go` 的 `ValidateUserTokenByReq` 中，按顺序执行：

1. **提取 API-Key**：调用 `bfe_basic.GetApiKey(req)`，若为空返回 `CodeNoApiKey`。
2. **查找 Token**：在规则表 `TokenRuleTable` 中按 `product + key` 查找；不存在返回 `CodeInvalidApiKey`。在校验失败前，模块会尽早把 `KeyId` 与 `ApikeyTags` 写入 `AiBasicInfo`，这样即使请求后续被拒绝，访问日志也能关联到具体 Key 并完成 Entity 归属。
3. **状态校验**：检查 `Enabled` 与 `ExpiredTime`；禁用或过期分别返回 `CodeKeyDisabled`、`CodeKeyExpired`。
4. **配额预检**：遍历 `token.QuotaPlans`，跳过 `Unlimited` 与 `PassNoQuota` 计划；对有限计划调用 `plan.HasBalance` 查询 Redis，余额不足返回 `CodeQuotaExhausted`，计划过期返回 `CodeQuotaExpired`。多个配额计划会依次检查，只有全部通过才放行。
5. **IP 子网校验**：若配置了 `Subnet`，检查 `ClientAddr` 或 `RemoteAddr` 是否在允许子网内；不在则返回 `CodeSubnetNotAllowed`。子网支持 CIDR 表示，可配置多个。

需要特别说明的是，上述所有拒绝路径——配额预检拒绝（429 `QUOTA_EXHAUSTED`）以及 `KEY_DISABLED` / `KEY_EXPIRED` / `QUOTA_EXPIRED` / `SUBNET_NOT_ALLOWED`——都会把 API-Key 的 tags 保留下来并写入访问日志的 `ai_apikeytags` 字段（pb field 702）与 `ClientKeyId`：拒绝行同样能按 Entity 归属做成本与用量归因，而不是只在放行请求上才有归属信息。

模型白名单/黑名单不在上述鉴权期列表中，而是在转发期对最终目标模型校验，详见下一节。

```mermaid
flowchart TD
    A[请求到达 HandleFoundProduct] --> B{命中 CHECK_TOKEN 规则?}
    B -->|否| C[放行]
    B -->|是| D[提取 Authorization Bearer]
    D --> E{Key 存在?}
    E -->|否| F[返回 401 CodeNoApiKey]
    E -->|是| G[查找 Token 配置]
    G --> H{启用/未过期?}
    H -->|否| I[返回 403 CodeKeyDisabled/Expired]
    H -->|是| J[遍历 QuotaPlans 查 Redis]
    J --> K{余额足够?}
    K -->|否| L[返回 429 CodeQuotaExhausted]
    K -->|是| M[子网校验]
    M --> N{通过?}
    N -->|否| O[返回 403 CodeSubnetNotAllowed]
    N -->|是| P[设置 TokenAuthContext]
```

校验成功后，`SetTokenAuthContext` 会把 `KeyId`、`ApikeyTags`、预估的 `PromptTokens` 等信息写入 `AiBasicInfo`，供后续模块使用。对于图片/视频生成模式，它还会在认证阶段预读请求体的 `n` 字段（`GetImageCountFromReq` / `GetVideoCountFromReq`）填入 `ImageCount` / `VideoCount` 作为兜底，`n` 缺失或 `≤ 0` 时取 1，防止响应未返回用量时按 0 计费少收：

```go
if aiBasicInfo.Mode == bfe_basic.ModeImageGeneration {
    tusage.ImageCount = GetImageCountFromReq(req)
}
if aiBasicInfo.Mode == bfe_basic.ModeVideoGeneration {
    tusage.VideoCount = GetVideoCountFromReq(req)
}
```

## 目标模型校验（转发期）

Token 的 `Models`（白名单）与 `BlockModels`（黑名单）在转发期校验，而不是鉴权期读请求体 `model` 字段。校验入口是 `mod_ai_token_auth` 注册在 `HandleAfterAITargetModel` 回调点的 `targetModelCheckFilter`（`bfe/bfe_modules/mod_ai_token_auth/model_check.go`）：`ServeHTTPForAI()` 的每次集群 attempt 在 `doSingleAIForward` 内、调用 `clusterInvoke` 之前触发该回调，此时 `AiBasicInfo.TargetModel` 已由 `computeTargetModel` 完成解析——依次应用路由 target/fallback 的目标模型覆盖、集群 `StripPrefix`/`MatchPrefix` 前缀裁剪、集群 `ModelMapping` 模型映射——校验对象就是发往上游的最终模型名。`ValidateTargetModel` 的逻辑为：

```go
func (m *ModuleAITokenAuth) ValidateTargetModel(req *bfe_basic.Request, targetModel string) *bfe_basic.AiError {
    ctx := GetTokenAuthContext(req)
    if ctx == nil || ctx.Token == nil {
        return nil // 未命中鉴权规则或未认证，不校验
    }
    tok := ctx.Token
    if len(tok.Models) == 0 && len(tok.BlockModels) == 0 {
        return nil // 未配置白名单/黑名单，放行
    }
    model := strings.TrimSpace(targetModel)
    if model == "" {
        // 配置了白名单/黑名单但目标模型不可判定，按"请求体中找不到模型"拒绝
        return ... // CodeInvalidRequest，HTTP 400
    }
    // 先查黑名单，再查白名单（白名单非空时必须命中）
    // 不匹配返回 CodeModelNotAllowed，HTTP 400
}
```

校验不通过时 `targetModelCheckFilter` 返回 `BfeHandlerFinish` 并构造本地 400 错误响应，该 attempt 不再发往上游，也不会触发 API-Key 轮换与集群 fallback；校验结果通过 `SetAiAuthInfo` 记入 `AiBasicInfo.AiAuthInfo.RejectReason`，便于访问日志关联拒绝原因。

按 attempt 触发意味着集群级 fallback 会对每个 attempt 各自解析出的目标模型复校验：主目标被白名单拒绝时请求携带 `RejectReason` 进入备用集群尝试，若备用 attempt 成功，`ServeHTTPForAI()` 会在跳出转发循环时清除残留的 `RejectReason`（及 `RejectQuotaPlans`），保证成功请求的访问日志不携带旧的拒绝原因。因此路由目标模型覆盖与 API-Key 模型白名单按"先重定向、后校验"组合生效：白名单约束的是转发后的目标模型，而不是客户端请求体里的原始模型。

原生 Gemini 路径（`:publishers/.../models/<model>:generateContent` 之外的 `/v1beta/models/<model>:<action>` 形式）的模型名取自 URL path（`bfe_model_protocol/gemini` 的 `ExtractModelFromPath`），同样进入 `TargetModel` 的解析链路，白名单校验对其生效。

## 上游 Key 选择、惩罚与凭证头注入

`aiClusterInvoke()`（`bfe/bfe_server/reverseproxy.go`）在进入单次转发前从集群的 `AIConf.Keys` 中选择上游 Key。选择与失败处理遵循以下规则。

### Key 惩罚与 Weight=0

- 每次选择跳过 `Weight=0` 的 Key：权重为 0 表示该 Key 不参与选择，惩罚过滤也不把它计入候选数；
- 上游返回 429 时，当前 Key 在本次请求内记为已用并轮换；开启会话亲和惩罚（`SessionAffinity` 且 `SessionAffinityPenaltyEnable`）时同时写入 429 档惩罚标记，时长取自 `KeyPolicy.SessionAffinityPenaltyTTL429`（可配置，默认 60s）；
- 上游返回 401/402/403 时，当前 Key 在本次请求内记为 dead；开启会话亲和惩罚时写入 auth 档惩罚标记，时长取自 `SessionAffinityPenaltyTTLAuth`（可配置，默认 **300s**）；
- 上游 5xx 或连接错误保持当前 Key 重试；其余 4xx（如 400）停止 Key 轮换且不惩罚；batch/file 的 404 是例外——清理该 batch 的绑定并对当前 Key 写入 60s 惩罚（开启会话亲和惩罚时，固定 60s 不受两档配置影响），让后续操作重新绑定到持有该资源的 Key；
- Key 尝试状态按 `AIConf.Keys` **下标空间**记账（与惩罚过滤后的子切片下标无关，避免把失败标记打到无辜 Key 上、或"藏住"健康的兄弟 Key 直接合成 429）；
- 惩罚过滤后候选集为空即熔断：不再回退到原始 Key 列表，惩罚中的 Key 必须等 TTL 到期才恢复；TTL 归一规则：`0` 视为"用默认值"而非关闭，负数报配置错误，`>86400` 截断为 86400 并 Warn。

### 429 NO_AVAILABLE_KEY

候选集为空时，`aiClusterInvoke()` 合成 429 响应而不是返回不透明的 5xx：

- `error.code=NO_AVAILABLE_KEY`、`error.type=rate_limit_error`、message 为 `no available upstream api key for this cluster`；
- 响应头带 `Retry-After: 60`，响应体 `error.details.retry_after_seconds=60`；该值固定，不随触发耗尽的惩罚类型（429 或 401/402/403）变化；
- 该响应标记为网关自生成（`X-Bfe-Gw-Error: 1`），错误归一化不会把它改写成 502；
- 错误码刻意不复用 `ErrAiRateLimit`：本地限流会短路集群 fallback，而 Key 耗尽需要允许 fallback 尝试备用集群；没有备用集群时调用方拿到可重试的 429。

### 出站凭证头互斥

`doSingleAIForward()` 在注入上游凭证前先清理出站请求的凭证头（`sanitizeOutboundCredentials`）：

- 适配器 `InjectAuth` 写入本次选中的那一个凭证头（openai 为 `Authorization`，anthropic 为 `x-api-key`，gemini 为 `x-goog-api-key`）；
- 其余注册表内的凭证头（`modelprotocol.CredentialHeaderNames()` 的并集）从出站请求中删除。上游按自身优先级挑选凭证（例如同时存在 `x-api-key` 与 `Authorization` 时优先前者），残留的客户端凭证会静默覆盖网关注入值，使 401 被错误计入该 Key 的惩罚；
- 删除前对 header map 做 `Clone()`：出站 header 与入站共享底层 map，直接删除会破坏入站请求的鉴权、计费、日志脱敏与 key 亲和会话标识；
- 集群未配置 `Keys`（`len(AIConf.Keys)==0`）时整体跳过 Key 选择与注入，既不清洗也不注入，客户端凭证按透传语义直达上游。

## 配额计划绑定与余额查询（Redis）

### 配置结构

`QuotaPlan` 在 BFE 运行时的定义位于 `bfe/bfe_modules/mod_ai_token_auth/token.go`：

```go
type QuotaPlan struct {
    Id          string
    Unlimited   bool
    PassNoQuota bool
    RedisKey    string
    ExpiredTime int64  // -1 means never expired
    Quota       int64  // 定点整数：total_token 为 Token 数；RMB 为 1e-8 元
    Unit        string // "total_token" or "RMB"
}
```

控制面在导出配置时，已经为每个配额计划生成稳定的 `RedisKey`，格式为 `QUOTA_<stableId>`，其中 `stableId` 对于 API-Key 级配额是 API-Key 值，对于 Entity 级配额是 `entity_id`。BFE 直接使用下发的 `RedisKey`，不再根据计划名称或 ID 自行拼装，从而避免改名导致计数器重置。这一原则与 `mod_ai_rate_limit` 的 Redis key 稳定性设计保持一致：计数器 key 不能依赖任何用户可编辑的业务字段。相关设计可参见 `bfe/docs/zh_cn/sys_design/ai_rate_limit_redis_key.md`。

`QuotaPlan` 中 `Unit` 为空时，`token_rule_load.go` 会将其默认置为 `total_token`，保证旧配置的向后兼容。配置校验阶段还会根据单位类型检查 `Quota` 的合法性：`RMB` 与 `total_token` 均要求不小于 0。其中 `total_token` 配额允许配置为 0，表示这是一个"无余额计划"——绑定该计划的 API-Key 在配额预检阶段即因余额不足被拒绝（429 `CodeQuotaExhausted`），适用于先创建 Key、后续再发放额度的场景。

### 余额查询

`QuotaPlan.HasBalance` 使用 Redis 客户端读取当前剩余量：

```go
func (q *QuotaPlan) HasBalance(client redis_client.Client) (bool, int64, error) {
    if q.Unlimited {
        return true, q.Quota, nil
    }

    if q.RedisKey == "" {
        return false, 0, errors.New("RedisKey is empty")
    }

    current, err := client.GetInt64(q.RedisKey)
    if err != nil {
        // key 未初始化（例如配额为 0 从未同步过 Redis）视为无余额，
        // 而不是内部错误
        if redis_client.IsKeyNotFound(err) {
            return false, 0, nil
        }
        return false, 0, err
    }

    return current > 0, current, nil
}
```

对于 `total_token` 配额，Redis 值就是剩余 Token 数；对于 `RMB` 配额，Redis 值是以 `1e-8` 元为单位的定点整数。`Unlimited` 计划直接返回 `true`，不参与余额检查。注意预检查的是 Redis 中的实时余额而不是配置里的 `Quota` 字段：当余额 key 不存在（如配额为 0 的计划从未同步过 Redis）时，按无余额处理，请求在预检阶段被 429 `CodeQuotaExhausted` 拒绝，而不会返回 500 内部错误。

### 控制面的余额同步

控制面 `ai-gateway-api` 在创建 API-Key / Entity 时，会通过 `QuotaCache.SetRemaining` 把初始余额写入 Redis；周期重置或手动重置时，使用 `IncrBy(delta)` 原子调整余额，而不是直接 `SET`，以避免覆盖并发请求刚扣减的计数。详细机制见 `ai-gateway-api/design-docs/sys-design/details/配额余额同步机制.md`。

## 请求结束时配额扣减

配额扣减发生在 `HandleRequestFinish`，此时响应已经完整返回客户端。`tokenRequestFinishHandler` 只处理 HTTP 200 的请求，并优先使用实际解析到的 Token 用量；如果响应中没有 `usage`、允许估算且响应已正常完成，则按内容长度估算。处理顺序为：先跳过 `/count_tokens` 端点，再校验扣费幂等标记，然后处理客户端中断场景，最后进入正常的用量结算：

```go
func (m *ModuleAITokenAuth) tokenRequestFinishHandler(req *bfe_basic.Request, res *bfe_http.Response) int {
    // /count_tokens 端点只统计 Token 数，永远不计费
    if strings.Contains(req.HttpRequest.RequestURI, "/count_tokens") {
        return bfe_module.BfeHandlerGoOn
    }

    if res == nil || res.StatusCode != bfe_http.StatusOK {
        // only count used quota for successful requests
        return bfe_module.BfeHandlerGoOn
    }

    ctx := GetTokenAuthContext(req)
    if ctx == nil {
        return bfe_module.BfeHandlerGoOn
    }

    // HandleRequestFinish 可能多次触发，扣费只执行一次
    if ctx.deducted {
        return bfe_module.BfeHandlerGoOn
    }

    // 客户端中断（RST/close/写入失败）且未见到最终 usage：不按整请求估算扣费
    aborted := isClientAbortErr(req.ErrCode)
    if aborted && !ctx.aiBasicInfo.IsFinalUsageSeen() {
        ctx.deducted = true
        return bfe_module.BfeHandlerGoOn
    }

    tokenUsage := ctx.aiBasicInfo.GetTokenUsage()
    // 估算值仅在响应正常完成时才可计费；否则清零估算字段，
    // 让下面的 calcCostUnits 直接按 Token 字段计费
    estimateBillable := ctx.aiBasicInfo.IsAllowEstimateToken() &&
        ctx.aiBasicInfo.IsResponseCompleted()
    if !ctx.aiBasicInfo.IsFinalUsageSeen() && !estimateBillable {
        tokenUsage.PromptTokens = 0
        tokenUsage.CompletionTokens = 0
        tokenUsage.UsedQuota = 0
    }
    if tokenUsage.UsedQuota <= 0 && estimateBillable {
        tokenUsage.UsedQuota = CalcReqUsedQuota(req, tokenUsage.PromptTokens, tokenUsage.CompletionTokens)
    }

    // 按已由 mod_body_process（流式）或 tokenReadResponseHandler（非流式）
    // 填充的 TokenUsage 计算 RMB 成本
    if tokenUsage.UsedCost <= 0 && hasRMBPlan(ctx.Token.QuotaPlans) {
        tokenUsage.UsedCost = m.calcCostUnits(req, ctx.serverConf, tokenUsage)
    }

    costUnits := tokenUsage.UsedCost

    if tokenUsage.UsedQuota > 0 || costUnits > 0 {
        for _, plan := range ctx.Token.QuotaPlans {
            if plan.Unlimited {
                continue
            }
            if quota.IsRMB(plan.Unit) {
                if costUnits > 0 {
                    _, err := plan.Deduct(m.redisClient, costUnits)
                    if err != nil {
                        log.Logger.Warn("deduct rmb quota failed: %v", err)
                    }
                }
            } else {
                if tokenUsage.UsedQuota > 0 {
                    _, err := plan.Deduct(m.redisClient, tokenUsage.UsedQuota)
                    if err != nil {
                        log.Logger.Warn("deduct token quota failed: %v", err)
                    }
                }
            }
        }
    }

    ctx.deducted = true
    return bfe_module.BfeHandlerGoOn
}
```

其中 `isClientAbortErr` 判定客户端侧中断错误：

```go
func isClientAbortErr(err error) bool {
    return err == bfe_basic.ErrClientWrite ||
        err == bfe_basic.ErrClientClose ||
        err == bfe_basic.ErrClientReset
}
```

### 扣费幂等

`TokenAuthContext` 带有 `deducted` 标记。`HandleRequestFinish` 在某些场景下可能被多次触发（例如连接异常收尾），handler 入口检查 `ctx.deducted`，已扣费则直接返回；正常路径在扣减循环结束后置位。这保证了同一请求无论触发几次，最多只扣一次。

### 客户端中断不计费与估算可靠性语义

按内容长度估算 Token（`EstimateToken`，请求体 `Content-Length/4` 估算输入、响应体长度估算输出）只是兜底手段，只有响应正常完成时才允许参与计费。`EstimateToken` 出厂默认 `true`。usage 的识别按 presence（而非按值）：显式上报全零 usage 对象（如 `prompt_tokens=0, completion_tokens=0`）的响应不会被误判为"无 usage"，`EstimateToken` 只对"从未收集到 usage 对象"的请求兜底。为此 `bfe_basic.AiBasicInfo` 维护两组状态标记：

- `MarkResponseCompleted` / `IsResponseCompleted`：响应正常完成时置位。流式场景由 `mod_body_process` 在见到终止事件（Anthropic `message_stop`、OpenAI `[DONE]`、Responses API `response.completed`）时置位；非流式场景由 `tokenReadResponseHandler` 在完整读出响应体后置位。此外，上游省略终止事件、以干净 EOF 结束且无 usage 的 SSE 响应也被视为完成（clean-EOF 完成标记：`copyResponse` 成功 + HTTP 200 + 非 file/batch 模式才标记），`EstimateToken` 兜底因此可以对这类响应计费。
- `MarkFinalUsageSeen` / `IsFinalUsageSeen`：从响应中解析到最终 usage 时置位。流式场景由 `QuotaUsageProcessor` 在收到最终 usage 事件（如 Anthropic `message_delta`、Responses API `response.completed`，或带 `image_count` / `video_count` 的事件）时置位；非流式场景在解析出非零 `UsedQuota` 后置位。Anthropic `message_start` 中 `output_tokens = 0` 的初始 usage 不算最终 usage。

结算规则为：

1. **客户端中断且未见到最终 usage**（`req.ErrCode ∈ {ErrClientWrite, ErrClientClose, ErrClientReset}` 且 `IsFinalUsageSeen() == false`）：直接置位 `deducted` 返回，不扣费。客户端中途断开时响应体不完整，按整请求估算会严重高估；
2. **非中断但响应未完成**（例如后端异常收尾）：清零 `PromptTokens` / `CompletionTokens` / `UsedQuota` 中的估算字段，再按正常公式计费，避免估算值污染 RMB 成本计算（`calcCostUnits` 直接读取 Token 字段，仅守住 `UsedQuota` 是不够的）；
3. **响应正常完成**：估算值可用，按 `CalcReqUsedQuota` 计算 `UsedQuota` 后进入扣减。

### /count_tokens 跳过计费

Anthropic 等协议的 `/v1/messages/count_tokens` 端点用于预统计 Token 数，不产生真实调用。`tokenRequestFinishHandler` 入口通过 `RequestURI` 包含 `/count_tokens` 判定并直接放行，避免把 Token 统计请求计入配额。

### total_token 扣减

`total_token` 配额使用 Lua 脚本原子扣减，位于 `bfe/bfe_modules/mod_ai_token_auth/token.go`：

```go
func (q *QuotaPlan) deductToken(client redis_client.Client, amount int64) (int64, error) {
    lua := `
        local current = tonumber(redis.call('GET', KEYS[1]) or '0')
        local amount = tonumber(ARGV[1])
        local deduct = math.min(current, amount)
        if deduct > 0 then
            redis.call('DECRBY', KEYS[1], deduct)
        end
        return math.max(0, current - deduct)
    `
    script := client.NewScript(lua)
    result, err := script.Run(q.RedisKey, amount)
    // ...
}
```

脚本使用 `math.min(current, amount)` 保证即使余额不足也不会扣成负数，返回扣减后的剩余量。Lua 脚本在 Redis 单线程中执行，读取余额与扣减操作是原子的，可以避免高并发下多个请求同时读取同一余额后都判断为足够，最终导致超扣。

### RMB 扣减

RMB 配额扣减需要处理 Key 不存在的情况（例如 Redis 数据被清理后首次请求，或新创建的 API-Key 尚未初始化余额）。Lua 脚本会先以配额总量初始化 Key，再执行扣减：

```go
func (q *QuotaPlan) deductRMB(client redis_client.Client, amount int64) (int64, error) {
    lua := `
        local raw = redis.call('GET', KEYS[1])
        local current
        if raw == false then
            current = tonumber(ARGV[2])
            redis.call('SET', KEYS[1], current)
        else
            current = tonumber(raw)
        end
        local cost = tonumber(ARGV[1])
        local deduct = math.min(current, cost)
        if deduct > 0 then
            redis.call('DECRBY', KEYS[1], deduct)
        end
        return math.max(0, current - deduct)
    `
    script := client.NewScript(lua)
    result, err := script.Run(q.RedisKey, amount, q.Quota)
    // ...
}
```

### 非 200 请求不计费

`tokenRequestFinishHandler` 在 `res.StatusCode != 200` 时直接返回，避免后端失败、超时或鉴权被拒绝的请求错误消耗配额。这与 `mod_body_process` 中的 `NewQuotaUsageProcessor` 行为一致：后者也在 `res.StatusCode != 200` 时直接返回 `nil`。

非 2xx 响应还会清除 `EstimateToken` 估算残留，避免把估算值带进失败行的计量。访问日志侧的 token 三字段（`ai_input_tokens` / `ai_output_tokens` / `ai_total_tokens`）按 `usageDisposition` 输出：disposition 为 `none`（非 2xx）时三个字段全部缺席，从而在日志中区分"无用量"与"用量为 0"。同时，`ai_cost_value` 现在对每个 observed 2xx 行都会写入（包括成本为 0 的行）——做计费对账时，"未计费"不能再以 `ai_cost_value IS NULL` 判断，需要改用 `ai_cost_value = 0` 并结合 `ai_zero_cost_reason` 归因。

## RMB 配额与时段定价处理

RMB 配额的核心是把 Token 用量按模型单价折算为人民币成本，再从 Redis 余额中扣减。为了支持 DeepSeek 等模型的高峰/空闲分时段定价，BFE 在 `AIConf.ModelTable` 中维护了一套完整的时段模板与价格表。

### ModelTable 结构

`bfe/bfe_config/bfe_cluster_conf/cluster_conf/cluster_conf_load.go` 定义了运行时价格结构：

```go
type ModelTable struct {
    Currency string      // 固定 "RMB"
    TimeZone string      // 默认 "Asia/Shanghai"
    Tiers    []PriceTier // 时段定义，如 peak
    Models   []ModelPrice
}

type ModelPrice struct {
    Provider            string
    Model               string
    BaseModel           string
    Mode                string
    Prices              PriceMap     // 默认价格
    TierPrices          TierPriceMap // tier name -> 价格表
    // ...
}
```

配置加载时价格保持 `float64` 原值（仅做非负校验），取值经 `ModelPrice.GetPrice(tier, key)`：tier 命中时取 `tier_prices.<tier>` 的价格，缺键或未命中 tier 回退默认 `prices`，未配置的键返回 0（该分项按 0 成本处理）。

### 时段匹配

`ModelTable.ActiveTierName` 根据请求结束时刻（`time.Now()`）和配置的时区，判断当前是否命中某个 tier：

```go
func (table *ModelTable) ActiveTierName(now time.Time) string {
    if table == nil || len(table.Tiers) == 0 || table.tz == nil {
        return ""
    }
    t := now.In(table.tz)
    wd := int(t.Weekday())
    cur := t.Hour()*60 + t.Minute()

    for i := range table.Tiers {
        tier := &table.Tiers[i]
        for _, tr := range tier.TimeRanges {
            if len(tr.Weekdays) > 0 && !containsInt(tr.Weekdays, wd) {
                continue
            }
            start, _ := minutesFromHHMM(tr.Start)
            end, _ := minutesFromHHMM(tr.End)
            if start <= cur && cur < end {
                return tier.Name
            }
        }
    }
    return ""
}
```

若未命中任何 tier，或 tier 中缺少某个价格键，则 fallback 到默认 `Prices`。这种 fallback 机制保证：管理员可以先只为高峰时段配置差异化价格，非高峰时段自动使用默认价格；也可以逐步迁移现有固定价格配置，无需一次性填写所有 tier 价格。

时区解析发生在配置加载阶段，通过 Go 标准库 `time.LoadLocation` 完成。部署容器镜像时需要确保包含 IANA 时区数据库，否则时区解析会失败并回退到 UTC，可能导致高峰时段判断偏移。

### 成本计算

`calcCostUnits` 位于 `bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`，它会根据目标 Cluster、目标模型、请求模式查找对应 `ModelPrice`，再按模式分发到具体的计费函数：

```go
switch mode {
case bfe_basic.ModeImageGeneration:
    return calcImageGenerationCost(entry, usage, tierName)
case bfe_basic.ModeVideoGeneration:
    return calcVideoGenerationCost(entry, usage, tierName)
case bfe_basic.ModeResponses:
    return calcResponsesCost(entry, usage, tierName)
default:
    return calcChatCost(entry, usage, tierName)
}
```

模式由 `bfe_basic.DetectModeFromPath` 根据请求路径判定（端点表为 `bfe_basic/openai_endpoint.go` 的 `openAIEndpointModes`）：先剥离可选的 `/v1` 版本前缀，再查共享端点表——`/responses` → `ModeResponses`、`/video/generations` → `ModeVideoGeneration`，`/chat/completions`、`/embeddings`、`/images/generations` 等端点各归对应模式，无专属模式的端点与未知路径归默认 `ModeChat`；`/compatible-mode/v1/responses` 这类以 `/v1` 段结尾的 provider 原生入口归约后同样识别为 `ModeResponses`。

`calcChatCost` 支持多种细粒度计费维度：

- 普通输入 / 输出 Token（`input_cost_per_token`、`output_cost_per_token`）
- 缓存命中输入 Token（`cache_read_input_token_cost`）
- 缓存写入输入 Token（`cache_creation_input_token_cost`）
- 图片输入 Token（`input_cost_per_image_token`）
- 音频输入 / 输出 Token（`input_cost_per_audio_token`、`output_cost_per_audio_token`）

计算时会先做卫生处理：确保各分项非负且不超过对应总量，再从总输入中依次拆出缓存读写、图片输入、音频输入 Token 分别计价。配置了 `input_cost_per_image_token` 或音频输入价格时，对应的输入 Token 从 `normalInput` 中拆出单独计价；未配置这些细化价格时，图片/音频输入 Token 按普通输入 Token 计价。所有 cache / audio / image 细化价格键均未配置时，回退 legacy 公式 `promptTokens × inputCost + completionTokens × outputCost`。每个计费项经 `quota.CalcCostUnits(用量, 价格)` 换算（即 `round(用量 × 价格 × 1e8)`，1 个整数单位 = 1e-8 元）后以 int64 累加，浮点只参与单项乘法，不会产生浮点累加误差。

需要特别说明的是缓存拆分的前提：`PromptTokens` 必须是总输入 Token 数。Anthropic 协议的 `input_tokens` 不含缓存读写 Token，协议适配层（`bfe/bfe_model_protocol/utils/usage_parse.go` 的 `ParseAnthropicUsageFields`）已将其归一化为 `input_tokens + cache_read_input_tokens + cache_creation_input_tokens`，与 OpenAI `prompt_tokens` 语义对齐；usage 字段的统一提取也已在协议适配层完成（详见[第七章 数据面转发设计：BFE](../design/chapter07-data-plane-design.md)与[第二十九章 AI 路由模块实现：mod_ai_route](./chapter31-mod-ai-route.md)），本章只关注结算逻辑。

`calcResponsesCost` 直接复用 chat 计费；`calcVideoGenerationCost` 按 `VideoCount × output_cost_per_video` 计费，其中 `VideoCount` 优先取响应 `usage.video_count`，回退 `data.#`；`calcImageGenerationCost` 按 `ImageCount × output_cost_per_image + ImageInputTokens × input_cost_per_image_token` 计费。

image_generation / video_generation 的计费是 entry 驱动的，判别依据是**价格键是否存在**（`ModelPrice.HasPrice`，`bfe/bfe_config/bfe_cluster_conf/cluster_conf/cluster_conf_load.go`）而不是键的值：匹配条目缺少 per-unit 价格键（`output_cost_per_image` / `output_cost_per_video`）时回落 chat token 公式（`calcChatCost`）；混合条目（两类键都有）按 per-unit 计费；显式配置 `output_cost_per_image = 0` 仍按 per-image 计费、成本为 0（video 对称），与"键缺失"语义区分。

此外，价格表没有当前 `(model, mode)` 条目时（例如模型只配置了 `mode=chat` 价格却被 `/responses` 入口调用），`calcCostUnits` 回落到同一模型的 chat 价计费，并按可观测性要求打标：计数器 `PriceFallbackUsed` 记录回落计费、`PriceLookupMiss` 记录连回落基线都不存在的无价格情况；两种 miss 都会把访问日志的 `ai_price_miss` 字段（pb field 763，`bfe-access-pb` ≥ v0.3.14）置 true。files/batches 端点的 `ModeFile` / `ModeBatch` 流量由 `mod_ai_batch` 处理（见下文"批量与异步任务支持"），不会落入上述 chat 计费路径。

控制面如何生成和下发时段配置，详见 `ai-gateway-api/design-docs/sys-design/details/RMB配额分时段定价.md`。

## 批量与异步任务支持（一期）：mod_ai_batch

除按请求计费外，`mod_ai_token_auth` 还与专门的 `mod_ai_batch` 模块（`bfe/bfe_modules/mod_ai_batch/`）配合，为 OpenAI Batch API（`/v1/files`、`/v1/batches` 端点族）提供一期支持。设计背景（预留-结算-释放模型、`batch_files` / `batch_tasks` 表、Redis 键契约、对账 job）见[第十二章 配额与限流设计](../design/chapter12-quota-and-rate-limit.md)，本节聚焦数据面实现。

### 端点识别与流量隔离

`bfe_basic.DetectModeFromPath` 为 files/batches 端点族识别出专用模式 `ModeFile` / `ModeBatch`，`mod_ai_batch` 只对这两种模式的请求生效。batch 流量因此不再落入 chat 计费路径：files/batches 管理端点从未成为可计费模型调用，它们的零成本属设计而非异常。

### 上传计数与流式转发

文件上传在转发的同时进行行数/字节计数（`file_counter.go` 的计数 reader），大文件采用流式转发，不做重试缓冲——上传被客户端中断或上游失败时不缓存已流过的内容，由客户端重传。chunked 编码、没有 `Content-Length` 的上传由 `mod_ai_batch` 的计数 reader 依据生效硬顶（`AiBasicInfo.BatchEffMaxFileBytes` / `BatchEffMaxFileLines`）在中途 enforcing。

### batch 级 key 亲和

batch 任务的 create / retrieve / cancel / 结果下载需要落在同一数据面实例上（任务实体在实例内维护）。`mod_ai_batch` 以 Redis `SETNX` 先到先得的方式建立 batch 级 key 亲和：创建时抢到亲和的实例负责该 batch 的后续操作，四步操作（上传→创建→查询→下载）保持同一 API-Key。

### keyless Provider 的取消请求

keyless Provider（`AIConf.Keys` 为空）的集群整体跳过 Key 选择与凭证注入（`aiClusterInvoke()` 的 `len(cluster.AIConf.Keys)==0` 分支），因此经数据面透传的 batch 取消请求不携带任何网关侧鉴权头；该分支同样不执行出站凭证头清洗，客户端自带的凭证头按透传语义原样转发。取消状态与幂等由控制面维护（命中 `cancelling` 返回 200 并重发上游取消），见[附1 OpenAPI 接口速查](../appendix/appendix01-openapi-quick-reference.md)。

### 后付费配额：预留-结算-释放

batch 创建时最终用量未知，计费采用后付费的预留-结算-释放模型：

1. **create 预检预留**：`POST /v1/batches` 先做配额预检并预留额度；
2. **结果下载时结算**：结果文件下载时按行内 model 分组，按 batch 价（`mode=batch` 价格行，见[第十三章 模型定价与成本核算设计](../design/chapter13-model-pricing.md)）逐模型结算；
3. **取消/终态释放**：batch 被取消或进入终态时释放预留。

幂等由 `BATCH_RESERVE_BATCH:<batch_id>`（预留簿记）与 `BATCH_SETTLED`（结算去重 SETNX 锁）保证，控制面另有 reconcile job 做兜底对账。

### 访问日志 batch 字段

`bfe-access-pb` ≥ v0.3.13 为 batch 场景提供 7 个访问日志字段（pb 901-907），由 `mod_access_pb3` 从 `AiBasicInfo` 读取：

| 字段 | pb 编号 | 说明 |
|------|---------|------|
| `ai_batch_id` | 901 | batch 任务 ID |
| `ai_file_id` | 902 | 文件 ID |
| `ai_batch_op` | 903 | batch 操作类型（create / upload / retrieve / cancel / download 等） |
| `ai_file_lines` | 904 | 上传文件行数 |
| `ai_file_bytes` | 905 | 上传文件字节数 |
| `ai_batch_status` | 906 | batch 状态 |
| `ai_batch_settle` | 907 | 结算状态/结果标记 |

## 与 mod_body_process 的协作

`mod_body_process` 负责解析流式（SSE）响应体中的 Token 用量，而 `mod_ai_token_auth` 负责非流式响应体的解析以及最终的配额扣减。两者通过 `AiBasicInfo.TokenUsage` 共享数据。

### 非流式响应

对于非流式请求，`mod_ai_token_auth` 在 `HandleReadResponse` 中直接读取完整响应体：

```go
func (m *ModuleAITokenAuth) tokenReadResponseHandler(req *bfe_basic.Request, res *bfe_http.Response) int {
    ctx := GetTokenAuthContext(req)
    if ctx == nil {
        return bfe_module.BfeHandlerGoOn
    }
    tokenUsage := ctx.aiBasicInfo.GetTokenUsage()
    if res.StatusCode == bfe_http.StatusOK && res.ContentLength >= 0 {
        // 完整响应体已读出：响应正常完成
        ctx.aiBasicInfo.MarkResponseCompleted()
        if bodyAccessor, err := res.GetBodyAccessor(); err == nil {
            body, _ := bodyAccessor.GetBytes()
            UpdateCtxByUsage(ctx, body)
        }
        if tokenUsage.UsedQuota > 0 {
            ctx.aiBasicInfo.MarkFinalUsageSeen()
        }
        // 若仍无 usage 且允许估算
        if tokenUsage.UsedQuota <= 0 && ctx.aiBasicInfo.IsAllowEstimateToken() {
            tokenUsage.CompletionTokens = int64(res.ContentLength) / 4
            tokenUsage.UsedQuota = CalcReqUsedQuota(req, tokenUsage.PromptTokens, tokenUsage.CompletionTokens)
        }
    }

    return bfe_module.BfeHandlerGoOn
}
```

`UpdateCtxByUsage` 将字段提取委托给协议适配层：根据请求阶段识别出的 `AiBasicInfo.AuthStyle` 取对应适配器（`modelprotocol.Get(authStyle).ExtractUsageFields`），由 `bfe_model_protocol` 按协议提取 `PromptTokens`、`CompletionTokens`、缓存读写、图片输入、图片/视频数量等字段后回填到 `TokenUsage`。这样既兼容 OpenAI、DeepSeek、Anthropic 等协议的 usage 差异（如 DeepSeek 的 `prompt_cache_hit_tokens`、Anthropic 的 `message.usage` 嵌套形态），也让新增协议的用量提取可以在适配层独立扩展。

当响应体中确实没有 `usage` 字段（例如某些私有部署模型未返回用量），且配置允许估算时，模块会按 `Content-Length / 4` 粗略估算输出 Token 数，并结合请求体估算的输入 Token 数得到一个近似用量。估算逻辑仅作为兜底，不建议用于精确计费场景。

### 流式响应

流式响应由 `mod_body_process` 在 `HandleReadResponse` 中逐段解析 SSE 事件，并通过 `QuotaUsageProcessor.Process` 累计 Token 用量到 `AiBasicInfo.TokenUsage`。由于流式响应在 `HandleReadResponse` 阶段尚未结束，`mod_ai_token_auth` 的 `tokenReadResponseHandler` 通常拿不到完整用量；最终扣减由 `tokenRequestFinishHandler` 在 `HandleRequestFinish` 阶段读取已填充的 `TokenUsage` 完成。

`QuotaUsageProcessor` 在解析每个 SSE 事件时同步维护计费可靠性状态：见到终止事件（Anthropic `message_stop`、OpenAI `[DONE]`、Responses API `response.completed`）时调用 `MarkResponseCompleted`；收到最终 usage 事件（Anthropic `message_delta`、Responses API `response.completed`——该事件同时是终止事件与最终 usage 事件，usage 嵌套在 `response.usage` 下）时调用 `MarkFinalUsageSeen`。Anthropic 流式的 `message_start` 只带初始 usage（`output_tokens = 0`），`message_delta` 才携带最终 output tokens 但不带 prompt 字段——Processor 对最终 usage 事件始终处理，并在 `message_delta` 场景下保留 `message_start` 阶段已解析的 `PromptTokens`、`CacheReadTokens`、`CacheWriteTokens` 等字段，只更新 completion 部分，避免缓存 Token 丢失或重复计价。这两个状态标记正是 `tokenRequestFinishHandler` 判断"客户端中断不计费、估算仅正常完成时可用"的依据。

```mermaid
sequenceDiagram
    participant Client
    participant BFE
    participant Backend
    participant TokenAuth as mod_ai_token_auth
    participant BodyProc as mod_body_process

    Client->>BFE: 流式请求
    BFE->>TokenAuth: HandleFoundProduct 校验 API-Key / 配额
    TokenAuth-->>BFE: 设置 TokenAuthContext
    BFE->>Backend: 转发请求
    loop SSE 流式响应
        Backend-->>BFE: 数据块
        BFE->>BodyProc: HandleReadResponse 解析 SSE
        BodyProc->>BodyProc: 累计 TokenUsage
    end
    BFE->>TokenAuth: HandleRequestFinish
    TokenAuth->>TokenAuth: calcCostUnits / Deduct
    TokenAuth-->>BFE: 完成扣减
    BFE-->>Client: 响应结束
```

这种分工保证了：无论流式还是非流式，`TokenUsage` 在 `HandleRequestFinish` 阶段都已经可用，`mod_ai_token_auth` 只需要执行统一的扣减逻辑。

## 计费可观测性：usage 来源与零成本归因

为了让"计费为 0 / 计费跳过"在对账时可解释、可查询，计费链路在访问日志中输出一组归因字段（`bfe_modules/mod_access_pb3` 写入，`bfe/docs/zh_cn/sys_design/ai_access_log_fields.md` 有完整字段手册）。

### usage 来源标记 ai_usage_source

`ai_usage_source`（string，pb field 764，`bfe-access-pb` ≥ v0.3.15）标记本次计费的 usage 来源，取值为：

| 值 | 含义 |
|----|------|
| `parsed` | 使用上游真实 usage 计费（含显式上报全零 usage 对象的响应） |
| `estimated` | 上游未返回 usage，`EstimateToken` 估算获批计费 |
| `unrecognized` | usage 未识别，计费跳过 |
| `aborted` | 客户端提前断开，计费跳过 |

非 2xx 行不写该字段。它与计费判定在同一点设置，使计费旁路在对账中可以直接查询。

### 零成本归因 ai_zero_cost_reason

`ai_zero_cost_reason`（string，pb field 765，`bfe-access-pb` ≥ v0.3.16）解释"2xx、绑定了 RMB 配额计划、计费却为 0"的成因，取值包括：`unrecognized` / `aborted` / `cache_hit` / `batch_release` / `price_miss` / `price_zero` / `model_table_missing` / `config_missing` / `no_usage_tokens` / `cost_zero`。已计费、未绑定 RMB 计划或非 2xx 的行不写该字段；批量与 files 管理端点（`mode` 为 `file`/`batch`）以及未解析出 `target_model` 的请求也不写——它们从未成为可计费模型调用，零成本属设计。AI 缓存命中只写 `cache_hit` 标记、不计入 `BillingZeroCost` 计数器。

### final usage 自动注入

为让流式响应尽可能携带真实 usage，BFE 提供 `Server.InjectStreamUsage` 配置（`bfe.conf`，出厂默认开启）：OpenAI 流式 chat/completions 请求未显式设置 `stream_options.include_usage` 时，转发前自动注入 `stream_options.include_usage=true`（`bfe_server/reverseproxy.go` 的 `maybeInjectStreamUsage`），从而从上游拿到真实的 final usage chunk，减少估算计费的比例。

### 计费副本与日志视图分离

估算清零、兜底填充等卫生处理只作用于计费副本；访问日志视图保留观测值。例如客户端中断的行虽然计费为 0，日志中仍能看到中断前已观测到的 token 数，便于排查"为什么这一行没扣费"。

### Kimi 嵌套 usage 解析

部分 OpenAI 兼容实现（如 Kimi）把 usage 嵌在流式 finish chunk 的 `choices[i].usage` 下而不是顶层。`ParseOpenAIUsageFields`（`bfe/bfe_model_protocol/utils/usage_parse.go`）对该嵌套形态做回退解析，并把嵌套 cache 字段 `cached_tokens` 映射为缓存读 Token，Kimi 流式计费统计因此不会归零。访问日志出口处，负 `CompletionTokens` 哨兵值被钳为 0，避免异常值进入日志与对账。

### 非流式 gzip 响应的 usage 采集

`Content-Type` 带 charset 参数（如 `application/json; charset=utf-8`）的 gzip 响应同样被识别为压缩体，usage 采集仅对解析副本解压，不影响转发给客户端的原始响应。

### 模块计数器

计费可观测性配套的模块计数器（定义于 `mod_ai_token_auth`，输出到 BFE Web 监控接口）：

| 计数器 | 说明 |
|--------|------|
| `UsageUnrecognized` | 上游 usage 未识别、计费跳过的请求数（语义收窄为仅"未识别"） |
| `AbortBillingSkipped` | 客户端中止、按零扣费跳过的请求数 |
| `BillingZeroCost` | 可计费 2xx 但成本为 0 的请求数（AI 缓存命中只标记不计数） |
| `DeductFailed` | 配额扣减失败的请求数 |
| `AiUsageSuppressed` | usage 被抑制（如显式全零 usage 的识别修正）的次数 |
| `PriceFallbackUsed` / `PriceLookupMiss` | 回落 chat 价计费 / 无价格基线的次数（见上文"成本计算"） |
| `AiUsageInconsistent` | usage 数据不一致检测 |

## 监控指标

`mod_ai_token_auth` 在 `ModuleAITokenAuthState` 中暴露了核心请求量计数器（定义于 `bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`）：

| 指标名称 | 类型 | 描述 |
| -------- | ---- | ---- |
| `REQ_TOTAL` | Counter | 进入模块的总请求数 |
| `REQ_AUTH` | Counter | 命中鉴权规则并触发校验的请求数 |
| `REQ_AUTH_FAIL` | Counter | 鉴权或配额预检失败的请求数 |

此外，上节"计费可观测性"列出的 `UsageUnrecognized`、`AbortBillingSkipped`、`BillingZeroCost`、`DeductFailed`、`AiUsageSuppressed`、`PriceFallbackUsed`、`PriceLookupMiss` 等计数器用于计费质量观测。这些指标通过 BFE 内置的 Web 监控接口输出，运维人员可以据此计算认证失败率、按产品和 Key 维度拆分失败原因，并设置告警阈值。例如 `REQ_AUTH_FAIL` 突增可能意味着存在大量无效 Key 请求或某个配额计划已耗尽；`DeductFailed` 突增则需要检查 Redis 配额链路。

## 关键代码片段

### 模块初始化与回调注册

`bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`：

```go
func (m *ModuleAITokenAuth) Init(cbs *bfe_module.BfeCallbacks, whs *web_monitor.WebHandlers,
    cr string) error {
    // 加载基础配置与规则
    confPath := bfe_module.ModConfPath(cr, m.name)
    m.conf, err = ConfLoad(confPath, cr)
    // 创建 Redis 客户端
    client := redis_client.NewRedisClient(options)
    m.redisClient = client
    m.loadProductRuleConf(nil)

    // 注册四个回调
    cbs.AddFilter(bfe_module.HandleFoundProduct, m.tokenFoundProductHandler)
    // 模型白名单/黑名单校验注册在 AI 转发阶段回调点（目标模型已解析），
    // 按集群 attempt 复校验；先于 mod_ai_rate_limit 注册，
    // 保证白名单校验始终先于限流执行。
    cbs.AddFilter(bfe_module.HandleAfterAITargetModel, m.targetModelCheckFilter)
    cbs.AddFilter(bfe_module.HandleReadResponse, m.tokenReadResponseHandler)
    cbs.AddFilter(bfe_module.HandleRequestFinish, m.tokenRequestFinishHandler)

    // 注册监控与热加载接口
    web_monitor.RegisterHandlers(whs, web_monitor.WebHandleMonitor, m.monitorHandlers())
    web_monitor.RegisterHandlers(whs, web_monitor.WebHandleReload, m.reloadHandlers())
    return nil
}
```

### Token 校验与配额预检

`bfe/bfe_modules/mod_ai_token_auth/token_rule_table.go`：

```go
func (m *ModuleAITokenAuth) ValidateUserTokenByReq(req *bfe_basic.Request) (token *Token, err *bfe_basic.AiError) {
    key := bfe_basic.GetApiKey(req)
    if key == "" {
        return nil, bfe_basic.NewAiError(bfe_basic.CodeNoApiKey, ...)
    }

    token, ok := m.ruleTable.GetToken(product, key)
    if !ok {
        return nil, bfe_basic.NewAiErrorWithDetails(bfe_basic.CodeInvalidApiKey, ...)
    }

    // 状态校验 ...

    for _, plan := range token.QuotaPlans {
        if plan.Unlimited || plan.PassNoQuota {
            continue
        }
        hasBalance, _, err := plan.HasBalance(m.redisClient)
        if err != nil {
            return nil, bfe_basic.NewAiErrorWithDetails(bfe_basic.CodeInternalQuotaError, ...)
        }
        if !hasBalance {
            return nil, bfe_basic.NewAiErrorWithDetails(bfe_basic.CodeQuotaExhausted, ...)
        }
    }

    // 模型/子网校验 ...
}
```

### 响应体用量提取

`bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`：

```go
func UpdateCtxByUsage(ctx *TokenAuthContext, data []byte) {
    // 按请求阶段识别出的 AuthStyle 取协议适配器，
    // 由适配层统一提取各协议的 usage 字段
    fields := modelprotocol.Get(ctx.aiBasicInfo.AuthStyle).ExtractUsageFields(data)
    // fields 包含 UsedQuota / PromptTokens / CompletionTokens /
    // CacheReadTokens / CacheWriteTokens / AudioInputTokens /
    // AudioOutputTokens / ImageInputTokens / ImageCount / VideoCount
    tokenUsage := ctx.aiBasicInfo.GetTokenUsage()
    if fields.UsedQuota > 0 {
        // 回填全部字段到 TokenUsage
        // ...
    } else if fields.PromptTokens > 0 || fields.CompletionTokens > 0 ||
        fields.ImageCount > 0 || fields.VideoCount > 0 {
        // 无 total_tokens 时按分项合成 UsedQuota：
        // 图片/视频生成以数量为 quota，文本以 prompt + completion
        // ...
    }
}
```

### 扣费幂等标记

`bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`：

```go
type TokenAuthContext struct {
    Token       *Token
    aiBasicInfo *bfe_basic.AiBasicInfo
    // serverConf 缓存 SvrDataConf，供请求结束时计算 RMB 成本
    serverConf bfe_basic.ServerDataConfInterface
    // deducted 标记本请求是否已执行扣费，
    // 防止 HandleRequestFinish 重复触发导致重复扣费
    deducted bool
}
```

### 配置示例

`bfe/bfe_modules/mod_ai_token_auth/testdata/mod_ai_token_auth/token_rule.data` 给出了最小配置示例，包含 `Config`、`QuotaPlans`、`Tokens` 三个顶层字段：

```json
{
    "Version": "1.0",
    "Config": {
        "AI_product": [
            {
                "cond": "default_t()",
                "action": { "cmd": "CHECK_TOKEN" }
            }
        ]
    },
    "QuotaPlans": {
        "AI_product": [
            {
                "id": "plan-total-token",
                "unlimited": false,
                "pass_no_quota": false,
                "redis_key": "QUOTA_ak-2v8x9k3m7p",
                "expired_time": -1,
                "quota": 100000000,
                "unit": "total_token"
            }
        ]
    },
    "Tokens": {
        "AI_product": {
            "ak-2v8x9k3m7p": {
                "key": "ak-2v8x9k3m7p",
                "key_id": "apikey-001",
                "enabled": true,
                "expired_time": -1,
                "unlimited_quota": false,
                "allow_models": "gpt-4,gpt-3.5-turbo",
                "block_models": null,
                "subnet": null,
                "tags": [],
                "quota_plans": ["plan-total-token"]
            }
        }
    }
}
```

## 本章小结

`mod_ai_token_auth` 是壬远 AI 网关数据面中连接认证与计费的关键模块。本章要点如下：

- 模块在 BFE 模块链中位于 `mod_ai_route` 之前，负责在请求路由前完成 API-Key 校验与配额预检。
- 通过 `HandleFoundProduct`、`HandleAfterAITargetModel`、`HandleReadResponse`、`HandleRequestFinish` 四个回调，分别完成认证、目标模型校验、用量解析、配额扣减。
- API-Key 从 `Authorization: Bearer <api-key>` 中提取，鉴权期校验项包括存在性、启用状态、过期时间、来源子网以及各 `QuotaPlan` 的 Redis 余额；模型白名单/黑名单在转发期按最终目标模型校验（`ValidateTargetModel`），不匹配返回 400 并结束该次 attempt。
- `QuotaPlan.RedisKey` 由控制面生成并下发，BFE 直接使用，避免改名导致计数器重置；`total_token` 与 `RMB` 两种单位分别使用不同的 Lua 脚本扣减。
- RMB 配额按 `AIConf.ModelTable` 中的模型价格（float64）与当前时段 tier 逐项 `quota.CalcCostUnits` 换算为 1e-8 元定点整数后累加扣减；chat 计费从总输入中拆分缓存读写、图片/音频输入维度，均未配置时回退 legacy 公式，`responses` 复用 chat 计费。
- image/video 计费按价格键是否存在（`ModelPrice.HasPrice`）判别：缺 per-unit 价键回落 chat token 公式，显式 0 价仍按 per-unit 计费；`(model, mode)` 无价格条目时回落同模型 chat 价，计数器 `PriceFallbackUsed` / `PriceLookupMiss` 与日志字段 `ai_price_miss` 暴露可观测性。
- 计费可靠性由 `AiBasicInfo` 的 `MarkResponseCompleted` / `MarkFinalUsageSeen` 两组标记支撑：usage 识别按 presence 而非值（显式全零 usage 不再误判为无 usage），`EstimateToken` 出厂默认开启且只对从未收集到 usage 的请求兜底，干净 EOF 结束的流式响应视为完成；客户端中断且未见到最终 usage 不扣费，估算值仅在响应正常完成时可用；`TokenAuthContext.deducted` 保证扣费幂等；`/count_tokens` 端点跳过计费。
- 计费可观测性：访问日志含 `ai_usage_source`（764，parsed/estimated/unrecognized/aborted）、`ai_zero_cost_reason`（765，十种零成本归因）、`ai_price_miss`（763）字段；`Server.InjectStreamUsage` 默认向 OpenAI 流式请求注入 `include_usage`；清零/兜底只作用于计费副本、日志视图保留观测值；Kimi `choices[N].usage` 嵌套 usage 参与解析，非流式 gzip（带 charset）的 usage 采集按解压副本处理；`ai_cost_value` 对每个 observed 2xx 行都写（含 0），对账判断需用 `= 0` 而非 `IS NULL`。
- 配额预检拒绝（429）与 `KEY_DISABLED` / `KEY_EXPIRED` / `QUOTA_EXPIRED` / `SUBNET_NOT_ALLOWED` 等拒绝路径保留 API-Key tags 并写入访问日志 `ai_apikeytags`（702）与 `ClientKeyId`，拒绝行也能做 Entity 归属。
- 批量与异步任务由 `mod_ai_batch` 支持：`/v1/files`、`/v1/batches` 识别为 `ModeFile` / `ModeBatch`（batch 流量不落入 chat 计费），上传行/字节计数与大文件流式转发（不重试缓冲），batch 级 key 亲和（SETNX 先到先得），后付费配额预留-结算-释放（`BATCH_RESERVE_BATCH` / `BATCH_SETTLED` 幂等），访问日志含 901-907 batch 字段。
- 计费可靠性由 `AiBasicInfo` 的 `MarkResponseCompleted` / `MarkFinalUsageSeen` 两组标记支撑：客户端中断且未见到最终 usage 不扣费，估算值仅在响应正常完成时可用；`TokenAuthContext.deducted` 保证扣费幂等；`/count_tokens` 端点跳过计费。
- 上游 Key 按 `AIConf.Keys` 加权选择并跳过 `Weight=0`；429 与 401/402/403 触发 Key 轮换，开启会话亲和惩罚时分别写入 429 档 / auth 档惩罚（时长取 `SessionAffinityPenaltyTTL429` / `SessionAffinityPenaltyTTLAuth`，默认 60s / 300s，可配置），候选集为空时合成 429 `NO_AVAILABLE_KEY`（带 `Retry-After` 与 `error.details.retry_after_seconds=60`）；注入上游凭证前清理客户端凭证头、只保留本次注入的那一个，keyless 集群整体跳过选择与注入。
- 配额欠费模型：扣减为全额 `DECRBY`（不做截断），Redis 余额允许为负；鉴权期 `HasBalance` 对负余额判无余额（429），并输出 `QuotaOverage` 计数 + WARN。
- `total_token` 配额允许为 0（无余额计划，绑定请求被 429 拒绝），Redis 余额 key 不存在视为无余额而非内部错误。
- 流式响应的 Token 用量由 `mod_body_process` 解析并累计（`message_delta` 保留 `message_start` 的 prompt/cache 字段），非流式响应由 `mod_ai_token_auth` 直接解析；usage 提取统一委托给 `bfe_model_protocol` 协议适配层，最终统一在 `HandleRequestFinish` 中扣减。

理解 `mod_ai_token_auth` 的实现，有助于排查 API-Key 鉴权失败、配额误扣、RMB 计费不准等问题，也为后续扩展新的认证方式或计费维度打下基础。

## 参考文档

- `bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`
- `bfe/bfe_modules/mod_ai_token_auth/token.go`
- `bfe/bfe_modules/mod_ai_token_auth/token_rule_table.go`
- `bfe/bfe_modules/mod_ai_token_auth/token_rule_load.go`
- `bfe/bfe_modules/mod_ai_token_auth/conf_mod_ai_token_auth.go`
- `bfe/bfe_modules/mod_ai_batch/mod_ai_batch.go`
- `bfe/bfe_modules/mod_ai_batch/file_counter.go`
- `bfe/bfe_modules/mod_body_process/content_quota_usage.go`
- `bfe/bfe_modules/mod_body_process/mod_body_process.go`
- `bfe/bfe_modules/mod_access_pb3/request_log.go`
- `bfe/bfe_server/reverseproxy.go`
- `bfe/bfe_config/bfe_cluster_conf/cluster_conf/cluster_conf_load.go`
- `bfe/bfe_basic/request_ai_basic.go`
- `bfe/bfe_model_protocol/utils/usage_parse.go`
- `bfe/bfe_modules/bfe_modules.go`
- `bfe/docs/zh_cn/modules/mod_ai_token_auth/mod_ai_token_auth.md`
- `bfe/docs/zh_cn/sys_design/ai_rate_limit_redis_key.md`
- `bfe/docs/zh_cn/sys_design/ai_access_log_fields.md`
- `ai-gateway-api/design-docs/sys-design/details/配额余额同步机制.md`
- `ai-gateway-api/design-docs/sys-design/details/RMB配额分时段定价.md`
- [第二十一章 配额与限流设计](../design/chapter12-quota-and-rate-limit.md)
- [第二十二章 API-Key 与配额配置](../operation/chapter22-apikey-and-quota-config.md)
