# Chapter 13: Model Pricing and Cost Accounting Design

## Chapter Goals

Through this chapter, readers will understand:

- The role of model pricing data in the AI Gateway's cost accounting, quota deduction, and cost allocation;
- The field meanings and validation rules of the `ModelPrice` data model;
- The `model-list.yaml` bulk import format and how to use it;
- The design and implementation of tiered pricing for RMB quota;
- How the BFE Data Plane performs runtime cost calculation based on pricing data;
- The discount pricing model for batch tasks and the export expansion rules;
- The entry-driven billing for image/video and the fallback-to-chat-price rule when a mode has no price entry;
- The relationship and configuration ownership of model pricing after separating the Provider and Cluster concepts.

---

## The Role of Model Pricing Data

In the Rainway AI Gateway, model pricing data (Model Price) is the bridge connecting "resource usage" to "cost accounting". When a business system calls a model service via an API-Key, the BFE Data Plane parses the Token usage in the request and response, and calculates the cost of each call based on the pricing data delivered by the Control Plane.

Model pricing data mainly serves the following scenarios:

- **Quota deduction**: The RMB quota bound to an API-Key or Entity must be deducted precisely according to actual call cost. Missing or inaccurate pricing causes quota consumption to deviate from actual cost, affecting budget control;
- **Cost accounting**: Provides operators with call cost statistics by model, by Provider, by API-Key, and even by Entity, supporting cost insight and optimization decisions;
- **Cost allocation**: When multiple tenants or business lines share the gateway, costs are allocated and bills are generated based on actual consumed prices, enabling internal settlement;
- **Peak/off-peak pricing**: Some model providers quote prices by time period; the gateway must match different prices based on when the request occurs, accurately reflecting the supplier's billing rules;
- **Cache billing**: Supports pricing cached and uncached Tokens of prompt caching separately, encouraging cache reuse and accurately reflecting the cost structure.

Model pricing data is maintained centrally by the AI Gateway API Control Plane and delivered to BFE after being exported via InnerAPI. BFE performs real-time cost calculation while forwarding requests and uses the results for quota deduction and log auditing. The price versions of the Control Plane and Data Plane are kept consistent, avoiding accounting discrepancies caused by multiple data sources.

---

## ModelPrice Data Model

`ModelPrice` is a single model pricing record maintained by the Control Plane, with the primary key being the `(provider, model, mode)` triple. The OpenAPI path `/model-prices` provides full CRUD and bulk import capabilities.

### Core Fields

```json
{
  "id": 1,
  "provider": "deepseek",
  "model": "deepseek-v3",
  "base_model": "deepseek-v3",
  "mode": "chat",
  "capabilities": ["chat", "reasoning", "tools"],
  "supported_parameters": ["temperature", "max_tokens"],
  "limits": {
    "context_window": 128000,
    "max_input_tokens": 128000,
    "max_output_tokens": 8192
  },
  "prices": {
    "input_cost_per_token": 0.000002,
    "output_cost_per_token": 0.000008,
    "cache_read_input_token_cost": 0.0000005
  },
  "tier_prices": {
    "peak": {
      "input_cost_per_token": 0.000004,
      "output_cost_per_token": 0.000016,
      "cache_read_input_token_cost": 0.000001
    }
  },
  "price_currency": "RMB",
  "metadata": {
    "source": "https://platform.deepseek.com/pricing",
    "notes": "DeepSeek V3"
  },
  "create_time": 1716883200,
  "update_time": 1716883200
}
```

The fields are described as follows:

| Field | Description |
|------|------|
| `provider` | Provider / Cluster identifier; serves only as a price grouping identifier and is not required to exist in `/providers` |
| `model` | Model name, e.g. `deepseek-v3` |
| `base_model` | Normalized model name, used for price normalization after model mapping |
| `mode` | Request mode, e.g. `chat`, `completion`, `embedding`, `image_generation`, `batch`, etc. |
| `capabilities` | List of capabilities supported by the model, e.g. `vision`, `reasoning`, `tools` |
| `supported_parameters` | List of supported request parameters, e.g. `temperature`, `max_tokens` |
| `limits` | Limits object, including context window, max input/output Tokens, etc. |
| `prices` | Default price table; used as fallback when no tier matches |
| `tier_prices` | Tiered price table, keyed by tier name (only `peak` is supported initially) |
| `batch_discount` | Batch discount factor, `*float64`, in the range `(0, 1]`; semantically defaults to 0.5 (half price) when absent — see "Batch Task Pricing" below |
| `price_currency` | Price currency; currently fixed to `RMB` |
| `metadata` | Metadata, including price source, notes, etc. |

### Price Field Enumeration

The keys allowed in `prices` and `tier_prices.<tier>` include:

| Key | Description |
|------|------|
| `input_cost_per_token` | Input cost per Token |
| `output_cost_per_token` | Output cost per Token |
| `cache_read_input_token_cost` | Cost of cached-read input Tokens |
| `cache_creation_input_token_cost` | Cost of cache-creation input Tokens |
| `input_cost_per_token_above_200k_tokens` | Input cost above 200k Tokens |
| `output_cost_per_token_above_200k_tokens` | Output cost above 200k Tokens |
| `output_cost_per_image` | Cost per output image (image_generation mode) |
| `input_cost_per_image_token` | Cost per image input Token (available in both chat and image_generation modes) |
| `output_cost_per_video` | Cost per generated video (video_generation mode) |
| `output_cost_per_pixel` | Output cost per pixel |
| `input_cost_per_audio_per_second` | Input cost per second of audio |
| `input_cost_per_video_per_second` | Input cost per second of video |
| `output_cost_per_second` | Output cost per second |
| `ocr_cost_per_page` | OCR cost per page |
| `output_cost_per_character` | Output cost per character |

The current version has enabled Token-based, cache, image, and video billing price fields; the remaining fields are reserved for future multimodal billing. `input_cost_per_image_token` and `output_cost_per_video` are two price keys configurable at both the global `prices` level and the `tier_prices.<tier>` level; they default to 0 when absent (meaning that dimension is not billed) and cause a configuration load error when negative.

### Price Precision

Price fields in `prices` and `tier_prices` are `float64` floating-point numbers (about 15 significant digits), which in practice support far more than 8 decimal digits of precision, e.g. `0.0000015`, `7.6234102728e-08` (catalog prices with 10–12 decimal digits are represented losslessly). Both the input side (OpenAPI request bodies, `model-list.yaml` import) accept either decimal notation or scientific notation, which are equivalent; the output side is serialized by the standard encoder, where very small values appear in scientific notation (e.g. `1.5e-6`). Both forms are valid JSON numbers with unchanged numeric semantics. The validation rules are: the price must be non-negative, and `|price × 1e8| < 2^53` (about 9e15), ensuring that downstream BFE floating-point billing does not overflow.

---

## model-list.yaml Import Format

When you need to maintain a large number of model prices in bulk, you can import a `model-list.yaml` file via the `/v1/model-prices/import` interface. This file is one of the authoritative data sources for the `model_prices` table.

### Top-Level Structure

```yaml
version: v1.0                    # Format version, required
default_currency: "RMB"          # Global default currency; only RMB is currently supported
models:                          # Model list, required
  - ...
```

### Single-Record Structure

```yaml
models:
  - provider: "deepseek"
    model: "deepseek-v3"
    base_model: "deepseek-v3"
    mode: "chat"
    capabilities: ["chat", "reasoning", "tools"]
    supported_parameters: ["temperature", "max_tokens"]
    limits:
      context_window: 128000
      max_input_tokens: 128000
      max_output_tokens: 8192
    prices:
      input_cost_per_token: 0.000002
      output_cost_per_token: 0.000008
      cache_read_input_token_cost: 0.0000005
    tier_prices:
      peak:
        input_cost_per_token: 0.000004
        output_cost_per_token: 0.000016
        cache_read_input_token_cost: 0.000001
    metadata:
      source: "https://platform.deepseek.com/pricing"
      notes: "DeepSeek V3"
```

### Import Modes

`/v1/model-prices/import` supports two import modes:

- **replace** (default): Clears the `model_prices` table first, then writes the new data; suitable for full updates;
- **merge**: Updates existing `(provider, model, mode)` records and inserts new ones; suitable for incremental maintenance.

The import interface can only be called by administrators. During import, required fields, non-negative `prices`, valid `tier_prices` key names, non-negative integer `limits`, and triple uniqueness are validated.

---

## Batch Task Pricing (batch Discount)

Batch and asynchronous tasks (the OpenAI Batch API; see the batch quota model in [Chapter 12: Quota and Rate Limit Design](./chapter12-quota-and-rate-limit.md)) are settled at prices lower than online calls. The pricing model is expressed through the two fields `mode` and `batch_discount`.

### Mode Whitelist and batch_discount

The `mode` whitelist of `model_prices` includes `batch`, indicating that the price row is explicit pricing for the batch mode. The price row also has a `batch_discount` field:

- The type is `*float64` (nullable pointer), with the value in the range `(0, 1]` (out-of-range values are rejected in `ai-gateway-api/model/imodel_price/validate.go`);
- Semantically, absence means 0.5, i.e. half price: a chat price row carrying a discount is expanded into a batch price at export time.

### Export Expansion Rules

When the Control Plane exports `AIConf.ModelTable`, explicit `mode=batch` price rows are generated according to the following rules:

1. When a price row carries `batch_discount`, a `mode=batch` price row is expanded from `chat` price × discount factor and delivered to BFE together with the ModelTable;
2. When an administrator has already maintained a `(provider, model, mode=batch)` price row manually, the manual row takes precedence over coefficient expansion — the export never overwrites explicit configuration with the expanded result;
3. When a model has no batch pricing configured at all, batch tasks are settled at the chat base price.

### Batch Settlement Base Price

Batch settlement is performed at the batch price for the model when the result file is downloaded. The settled amount follows the base-price formula: `settle_units = usage_in × input_cost_per_token + usage_out × output_cost_per_token`. In this phase, batch settlement only integrates base prices; `tier_prices` time-of-day prices are not integrated into batch settlement. For the Data Plane `mod_ai_batch` reserve-settle-release implementation, see [Chapter 32: Token Authentication and Quota Module Implementation: mod_ai_token_auth](../implementation/chapter32-mod-ai-token-auth.md).

---

## RMB Quota Tiered Pricing Mechanism

As model providers such as DeepSeek adopt "peak / off-peak" tiered pricing, RMB quota deduction needs the ability to match different prices based on when the request occurs.

### Core Concepts

| Concept | Description |
|------|------|
| **Tier** | A price level divided along the time dimension, e.g. `peak`. When a request matches a tier, the corresponding price is used; otherwise it falls back to the default `prices` |
| **TimeRange** | A time-range definition, including `weekdays` (day of week, 0=Sunday), `start` / `end` (HH:MM, left-closed, right-open) |
| **Provider time template** | `time_zone` and `tiers` defined on `/providers`, shared by all models under the same provider |
| **Model tier price** | `tier_prices` defined on `/model-prices`, describing the price of a model under a given tier |

### Configuration Ownership and Delivery Chain

```
/provider deepseek
    ├── time_zone: Asia/Shanghai
    └── tiers: [peak]
    ├── model-prices deepseek-v3 chat
    │       ├── prices: default/off-peak price
    │       └── tier_prices.peak: peak price
    └── model-prices deepseek-v4 chat
            └── ...

/cluster my-cluster
    ├── llm_config.provider: deepseek
    └── AIConf.ModelTable (on export, the Control Plane concatenates the
        provider's time_zone/tiers with the model-prices prices/tier_prices)
```

When multiple clusters reference the same provider, each receives an identical copy of the `ModelTable` data; after the provider's time rules change, all clusters referencing it take effect automatically at the next configuration export.

### Provider Time Template

`/providers` carries the `time_zone` and `tiers` fields to describe shared time rules:

```json
{
  "name": "peak",
  "time_ranges": [
    { "weekdays": [1, 2, 3, 4, 5], "start": "09:00", "end": "12:00" },
    { "weekdays": [1, 2, 3, 4, 5], "start": "14:00", "end": "18:00" }
  ]
}
```

`time_zone` must be a valid IANA time zone name, defaulting to `Asia/Shanghai`. `time_ranges` within the same tier must not overlap, and `end` must be greater than `start`; periods crossing midnight must be split into two ranges.

### BFE ModelTable Structure

The `AIConf.ModelTable` structure exported by the Control Plane to BFE is as follows:

```go
type TimeRange struct {
    Weekdays []int  // 0=Sunday, 1=Monday ... 6=Saturday
    Start    string // "HH:MM"
    End      string // "HH:MM", must be > Start
}

type PriceTier struct {
    Name       string      // Only "peak" is supported initially
    TimeRanges []TimeRange
}

type PriceMap map[string]float64
type TierPriceMap map[string]map[string]float64

type ModelPrice struct {
    Provider            string
    Model               string
    BaseModel           string
    Mode                string
    Capabilities        []string
    SupportedParameters []string
    Limits              map[string]interface{}
    Prices              PriceMap
    TierPrices          TierPriceMap
    Metadata            map[string]interface{}
}

type ModelTable struct {
    Currency string
    TimeZone string
    Tiers    []PriceTier
    Models   []ModelPrice
}
```

---

## BFE-Side Cost Calculation Logic

The BFE Data Plane preprocesses price data when loading `AIConf`, and calculates the cost after request forwarding completes, based on actual Token usage and the current time period.

### Configuration Loading Stage

```
AIConf.ModelTable
       │
       ▼
┌──────────────┐
│ Parse TimeZone │─── default Asia/Shanghai
│ Validate Tiers │─── name, time format, weekdays, range overlap
└──────────────┘
       │
       ▼
┌──────────────┐
│ Validate prices │───  non-negative validation; prices stay float64, no fixed-point conversion
└──────────────┘
```

After loading, BFE keeps prices as `float64` (RMB per token) and performs no fixed-point pre-conversion. Validation is performed on each model's `prices` and each tier's `tier_prices.<tier>`: all price keys (including `input_cost_per_image_token`, `output_cost_per_video`, etc.) default to 0 when absent, and a negative value fails the entire configuration load with the specific model name and price key reported, preventing erroneous prices from entering runtime. At billing time, each line item is converted via `quota.CalcCostUnits(usage, price)` (i.e. `round(usage × price × 1e8)`) into a 1e-8-RMB fixed-point integer and accumulated in int64; floating point participates only in the single multiplication per line item, avoiding floating-point accumulation errors.

### Runtime Tier Matching

At the end of a request, BFE matches the active tier based on the current time:

```go
func (table *ModelTable) ActiveTierName(now time.Time) string {
    if table == nil || len(table.Tiers) == 0 {
        return ""
    }
    t := now.In(table.tz)
    wd := int(t.Weekday())
    hour, min := t.Hour(), t.Minute()
    cur := hour*60 + min

    for i := range table.Tiers {
        tier := &table.Tiers[i]
        for _, tr := range tier.TimeRanges {
            if len(tr.Weekdays) > 0 && !containsInt(tr.Weekdays, wd) {
                continue
            }
            start := parseHHMM(tr.Start)
            end := parseHHMM(tr.End)
            if start <= cur && cur < end {
                return tier.Name
            }
        }
    }
    return ""
}
```

### Runtime Cost Calculation

The cost calculation flow is as follows:

```
Request ends
   │
   ▼
Parse TokenUsage (uniformly extracted by the protocol adapter layer bfe_model_protocol)
   ├── PromptTokens (total input, including cache read/write and image/audio input)
   ├── CompletionTokens
   ├── CacheReadTokens / CacheWriteTokens
   ├── AudioInputTokens / AudioOutputTokens
   ├── ImageInputTokens / ImageCount
   └── VideoCount
   │
   ▼
Match ActiveTierName
   │
   ├── Tier matched ──► take tier_prices.<tier> prices, falling back to default prices for missing keys
   └── No tier match ──► take prices default prices
   │
   ▼
Calculate per request mode (calcChatCost, etc., each line item rounded via usage × price × 1e8, then accumulated as integers)
   │
   ▼
Accumulate into total request cost, used for RMB quota deduction and log output
```

#### Chat Billing Formula

`calcChatCost` (`bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`) starts from the total input Tokens and splits them into billing dimensions:

```
normalInput = PromptTokens                          // when no cache price is configured
            = max(PromptTokens - CacheRead - CacheWrite, 0)   // when a cache price is configured

When input_cost_per_image_token is configured:
    imageInputTokens = min(ImageInputTokens, normalInput)
    normalInput -= imageInputTokens

When an audio input price is configured:
    audioInputTokens = min(AudioInputTokens, normalInput)
    normalInput -= audioInputTokens

When an audio output price is configured:
    normalOutput = CompletionTokens - AudioOutputTokens
    (audioOutputTokens capped at CompletionTokens)

cost = normalInput  × input_cost_per_token
     + CacheRead    × cache_read_input_token_cost
     + CacheWrite   × cache_creation_input_token_cost
     + imageInput   × input_cost_per_image_token
     + audioInput   × input_cost_per_audio_token
     + normalOutput × output_cost_per_token
     + audioOutput  × output_cost_per_audio_token
```

When none of the cache / audio / image refinement price keys is configured, it falls back to the legacy formula `PromptTokens × input_cost_per_token + CompletionTokens × output_cost_per_token`, keeping the behavior of old configurations unchanged.

#### Anthropic Usage Semantics Normalization

In the Anthropic protocol, `input_tokens` only counts fresh (cache-missing) Tokens and excludes `cache_read_input_tokens` and `cache_creation_input_tokens`, which differs from the OpenAI semantics of `prompt_tokens` (total input). This normalization is performed at the parsing layer (`ParseAnthropicUsageFields` in `bfe/bfe_model_protocol/utils/usage_parse.go`):

```
PromptTokens = input_tokens + cache_read_input_tokens + cache_creation_input_tokens
```

This makes the downstream "total input minus cache" splitting logic identical for OpenAI and Anthropic. In streaming responses, the usage of the `message_start` event is nested under `message.usage`, which the parsing layer also accepts; the `message_delta` event carries only the final output tokens, and the billing module retains the prompt/cache fields parsed at the `message_start` stage, avoiding cache Token loss or double counting. Cache-creation Tokens are billed exactly once at `cache_creation_input_token_cost`, preventing fresh tokens from being truncated to 0 and never billed in high cache-hit scenarios.

Responses API usage follows subset semantics and skips the additive normalization above: `input_tokens` already includes `input_tokens_details.cached_tokens` (`total_tokens = input_tokens + output_tokens`, consistent with the Chat Completions main chain), so `ParseOpenAIUsageFields` takes `input_tokens` directly as `PromptTokens`. In streaming, the usage of the `response.completed` event is nested under `response.usage`; in non-streaming, the create-response object keeps usage at the top level with fields named `input_tokens`/`output_tokens`. Both shapes are parsed by the Responses-specific chain.

#### Request Mode Detection

The request mode is determined from the request path by `bfe_basic.DetectModeFromPath` (implemented in `bfe_basic/request_ai_basic.go`; the endpoint table is `openAIEndpointModes` in `bfe_basic/openai_endpoint.go`): an optional `/v1` version prefix is stripped first, and the stripped path is looked up in the shared OpenAI endpoint table to obtain the mode — the main endpoints such as `/chat/completions`, `/embeddings`, `/images/generations`, `/responses`, and `/video/generations` each have a dedicated mode; endpoints without a dedicated mode (`/models`, `/moderations`, `/audio/translations`) fall into the default `ModeChat`, as do unknown paths that miss the endpoint table. Provider-native entries whose prefix ends in a `/v1` segment (the OpenAI SDK `base_url` form, e.g. `/compatible-mode/v1/responses`) are also reduced to the endpoint table and recognized as the Responses mode rather than being misclassified as Chat. This endpoint table is shared with the upstream path rewrite (`bfe_server/ai_path_rewrite.go`), so rewrite eligibility and billing mode can never disagree on what an OpenAI endpoint is.

#### Other Request Modes

Billing is also defined for two more request modes:

| Mode | Path | Billing formula |
|------|------|-----------------|
| `responses` | `/responses` (including entries such as `/v1/responses` and `/compatible-mode/v1/responses`) | Reuses chat billing (`calcResponsesCost` delegates directly to `calcChatCost`) |
| `video_generation` | `/video/generations` (including `/v1/video/generations`) | `VideoCount × output_cost_per_video` |

`VideoCount` is taken from `usage.video_count` in the response, falling back to `data.#` (number of generated results); the authentication stage also pre-reads the `n` field of the request body as a fallback (taking 1 when `n <= 0`) to prevent under-billing when the response carries no usage. For image generation mode (`image_generation`), the cost is `ImageCount × output_cost_per_image + ImageInputTokens × input_cost_per_image_token`, where `ImageInputTokens` comes from `usage.input_token_details.image_tokens`, falling back to `usage.image_input_tokens`.

If a tier does not define a specific price key, it automatically falls back to the corresponding key in the default `prices`. Fields in `TokenUsage` such as `CacheReadTokens`, `ImageInputTokens`, and `VideoCount` allow cache hits, image input, and video generation to be priced separately.

#### Entry-Driven Billing for Image/Video

Billing for image_generation and video_generation is driven by the matched price entry, and the discriminator is **whether the price key exists**, not its value (BFE uses `ModelPrice.HasPrice` in `bfe/bfe_config/bfe_cluster_conf/cluster_conf/cluster_conf_load.go`):

- When the entry lacks the per-unit price key (`output_cost_per_image` / `output_cost_per_video`), billing falls back to the chat token formula (`calcChatCost`), avoiding completely free calls when the image/video unit price is not configured;
- When the entry holds both kinds of keys (a mixed entry), per-unit billing applies;
- An explicit `output_cost_per_image = 0` is still billed per image — that dimension simply costs 0; video is symmetric. This differs semantically from "key missing, fall back to the chat formula": the former means "free for this dimension," while the latter means "not separately priced, billed by Tokens."

#### Fallback Billing When a Mode Has No Price Entry (ai_price_miss)

When the price table has no entry for a given `(model, mode)` (for example, a model used via the `/responses` entry only has a `mode=chat` price), BFE falls back to the same model's chat price instead of billing 0. This behavior comes with observability markers:

- The module counters `PriceFallbackUsed` (fallback billing) and `PriceMissZeroBilled` (no fallback baseline at all) track the two cases (`bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`);
- The access log carries the `ai_price_miss` field (pb field 763, `bfe-access-pb` ≥ v0.3.14): both kinds of miss set it to true, distinguishing "under-priced/0-billing rows without pricing" from legitimate zero-cost rows. Field details are in [Chapter 32: Token Authentication and Quota Module Implementation: mod_ai_token_auth](../implementation/chapter32-mod-ai-token-auth.md).

#### Price-Miss Actions

Price-miss behavior is governed by an explicit cluster-level configuration `llm_config.price_miss_action` (snake_case in storage, exported as the PascalCase `AIConf.PriceMissAction`), with three states:

| Value | Semantics |
|------|-----------|
| `fallback_chat` (default) | Bill at the same model's chat price (i.e. the fallback behavior above) |
| `bill_zero` | Forward normally but bill 0 |
| `reject` | **Reject before forwarding**: 503 `PRICE_NOT_CONFIGURED`, no upstream call and no billing, while the access log row is still written with the price-miss signal |

A missing `ModelTable` as a whole is treated as unpriced (and likewise rejected under `reject`); the fail-open behavior when a cluster lookup fails is unchanged. Illegal values are rejected with 422 on the Control Plane export side and by `AIConfCheck` on the Data Plane load side; existing clusters without the field need no migration (BFE normalizes it to `fallback_chat`).

Observability reports a **signal pair**: `ai_price_miss` (bool, field 763) and `ai_price_warn` (string, field **766**, `bfe-access-pb` ≥ v0.3.17, with six forms — `fallback` / `subitem_missing` / `no_entry` / `no_baseline` / `no_model_table` / `reject`; the most specific cause wins). Both fields adopt the **NULL = not reported** semantics, and `ai_price_miss` is written at **request level** (`reject` 503 rows carry the signal too). The report detail table has **101** columns (the two price-miss columns sit after `ai_intent_questions_version`), and the query API supports an exact `price_warn` filter (whitelist-validated; illegal values return 422). The companion counters are `PriceMissZeroBilled` (price miss billed 0) / `PriceRejected` / `PriceFallbackSubItemMiss`. Known and accepted residual: a missing sub-item price (e.g. cache read/write price keys) bills 0 without rejecting.

### Backward Compatibility

The tiered pricing design maintains backward compatibility with fixed pricing:

- When `time_zone` / `tiers` are not set on `/providers`, `ModelTable.TimeZone` / `ModelTable.Tiers` are empty, and behavior is identical to fixed pricing;
- When `tier_prices` is not set on `/model-prices`, billing always uses the default `Prices`;
- When a tier is matched but a price key is not configured for that tier, it automatically falls back to the default `Prices`;
- In chat billing, when none of the cache / audio / image refinement price keys is configured, it falls back to the legacy formula `PromptTokens × input_cost_per_token + CompletionTokens × output_cost_per_token`;
- When `input_cost_per_image_token` / audio prices are not configured, image/audio input Tokens are billed as regular input Tokens;
- The semantics of `TokenUsage.UsedCost`, the Lua deduction logic, and Redis fixed-point number storage are unaffected by tiered configuration.

This compatibility allows existing deployments to enable tiered pricing smoothly, without a one-time full configuration adjustment.

---

## Relationship Between Pricing Data and Provider/Cluster

After the separation of the Provider and Cluster concepts, the ownership and referencing relationships of model pricing also become clearer.

### Concept Separation Recap

- **Provider**: The model provider, including the access endpoint, available models, API Keys, Instance Pool, supported protocols, and the time template (`time_zone`, `tiers`).
- **Cluster**: The forwarding cluster, deciding by which model, which weight, and which policy traffic is forwarded to a provider.
- **Model Price**: A price record keyed by `(provider, model, mode)`; the `provider` field serves only as a price grouping identifier and is not required to reference an existing provider.

### Configuration Delivery Relationship

```
/providers
   ├── name: deepseek
   ├── time_zone: Asia/Shanghai
   └── tiers: [peak]

/model-prices
   ├── provider: deepseek, model: deepseek-v3, mode: chat
   │       ├── prices
   │       └── tier_prices.peak
   └── provider: deepseek, model: deepseek-v4, mode: chat
           ├── prices
           └── tier_prices.peak

/clusters
   └── name: my-cluster
           └── llm_config.provider: deepseek

On exporting AIConf:
   ModelTable.TimeZone ← /providers.deepseek.time_zone
   ModelTable.Tiers    ← /providers.deepseek.tiers
   ModelTable.Models   ← records in /model-prices with provider=deepseek
```

### Flexibility from Weak References

The `provider` field of `/model-prices` no longer requires a reference to `/providers`, bringing the following benefits:

- More flexible configuration order: price data can be maintained first, and the Provider connected later;
- When a Provider is deleted, `model_prices` records with the same name no longer block the deletion;
- Historical price records can be retained even after the corresponding Provider is offline, facilitating cost tracing.

The recommended configuration order is `/providers → /model-prices → /clusters → routing rules`, but `/model-prices` and `/providers` have a weak reference relationship and can in practice be maintained independently.

### Impact on Cost Calculation

The weak reference relationship means: when BFE performs cost calculation, it only cares whether `AIConf.ModelTable` contains the price record for the corresponding `(provider, model, mode)`, not whether that provider still exists. Even if a Provider has been deleted, as long as its price records are retained, cost accounting for historical requests can still be completed correctly.

At the same time, since `ModelTable` is queried and assembled by provider according to `cluster.llm_config.provider` at export time, all model prices referenced by a cluster must exist; otherwise, cost calculation fails for calls to that model, and the related quota deduction also fails. Therefore, when deleting or renaming a Provider, you need to synchronously evaluate its impact on price data and cluster references.

---

## Pricing Configuration Example

The following example shows how to configure model pricing for a Provider that supports peak/off-peak billing.

### Provider Time Template

```json
{
  "name": "deepseek",
  "description": "DeepSeek official API",
  "model_protocols": ["openai"],
  "time_zone": "Asia/Shanghai",
  "tiers": [
    {
      "name": "peak",
      "time_ranges": [
        { "weekdays": [1, 2, 3, 4, 5], "start": "09:00", "end": "12:00" },
        { "weekdays": [1, 2, 3, 4, 5], "start": "14:00", "end": "18:00" }
      ]
    }
  ]
}
```

### model-list.yaml Snippet

```yaml
version: v1.0
default_currency: "RMB"

models:
  - provider: "deepseek"
    model: "deepseek-v3"
    base_model: "deepseek-v3"
    mode: "chat"
    capabilities: ["chat", "reasoning", "tools", "structured_outputs", "prompt_caching"]
    supported_parameters: ["temperature", "top_p", "max_tokens", "tools", "tool_choice", "response_format", "reasoning"]
    limits:
      context_window: 128000
      max_input_tokens: 128000
      max_output_tokens: 8192
    prices:
      input_cost_per_token: 0.000002
      output_cost_per_token: 0.000008
      cache_read_input_token_cost: 0.0000005
      cache_creation_input_token_cost: 0.000001
    tier_prices:
      peak:
        input_cost_per_token: 0.000004
        output_cost_per_token: 0.000016
        cache_read_input_token_cost: 0.000001
    metadata:
      source: "https://platform.deepseek.com/pricing"
      notes: "DeepSeek V3 official API"
```

### Cluster Reference

```json
{
  "name": "deepseek-cluster",
  "llm_config": {
    "provider": "deepseek",
    "models": ["deepseek-v3"],
    "keys": [
      { "name": "key-primary", "weight": 100 }
    ]
  }
}
```

After the above configuration is exported to BFE, calls to `deepseek-v3` between 09:00–12:00 and 14:00–18:00 on weekdays will be billed at the peak price, and at the default price at other times.

---

## Chapter Summary

- Model pricing data is the foundation of the AI Gateway's cost accounting, quota deduction, and cost allocation; it is maintained centrally by the AI Gateway API Control Plane and delivered to BFE.
- `ModelPrice` uses `(provider, model, mode)` as its primary key and includes fields such as capabilities, limits, default prices, and tiered prices.
- `model-list.yaml` provides bulk import capability, supports `replace` and `merge` modes, and is the main data source format for maintaining model prices in the current version.
- RMB quota tiered pricing is implemented by combining the Provider time template with Model tier prices; BFE matches tiers such as `peak` based on when the request occurs and falls back to default prices when no tier matches.
- The BFE Data Plane keeps prices as `float64` after loading (negative values fail loading); at runtime, based on Token usage and the active tier, each line item is converted via `quota.CalcCostUnits` into a 1e-8-RMB fixed-point integer and accumulated, while Redis deduction and the monetary semantics of existing configurations remain unchanged.
- Chat billing starts from the total input Tokens and splits out cache read/write, image input, audio input, and other dimensions; when no refinement price keys are configured, it falls back to the legacy formula.
- image_generation / video_generation discriminate by "whether the price key exists" (`ModelPrice.HasPrice`): a missing per-unit price key falls back to the chat token formula, while an explicit zero price is still billed per unit (cost 0).
- When the price table has no entry for a `(model, mode)`, billing falls back to the same model's chat price, with observability via the counters `PriceFallbackUsed` / `PriceMissZeroBilled` and the access log field `ai_price_miss`.
- Batch tasks are priced through `mode=batch` price rows and the `batch_discount` factor: export expands them into explicit batch price rows, manually maintained batch rows take precedence over coefficient expansion, and batch settlement uses the base price `usage_in × input + usage_out × output` (tiered prices are not integrated in this phase).
- Anthropic usage is normalized at the protocol adapter layer to total-input semantics (`input_tokens + cache_read + cache_creation`), aligning with OpenAI's `prompt_tokens`; Responses API `input_tokens` follows subset semantics that already include `cached_tokens` and are taken directly as `PromptTokens` without additive normalization; two billing modes are supported: `responses` (reusing chat billing) and `video_generation` (billed as `output_cost_per_video` × video count).
- After the Provider and Cluster concepts were separated, `model-prices.provider` serves only as a price grouping identifier and has a weak reference relationship with `/providers`, making configuration more flexible; `AIConf.ModelTable` is assembled by the Control Plane by provider at export time.

---

## References

- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/model-prices.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/batches.md`
- `ai-gateway-api/design-docs/sys-design/details/RMB配额分时段定价.md`
- `ai-gateway-api/design-docs/sys-design/details/provider与cluster概念分离.md`
- `bfe/bfe_config/bfe_cluster_conf/cluster_conf/cluster_conf_load.go`
- `bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`
- `bfe/bfe_model_protocol/utils/usage_parse.go`
