# Chapter 13: Model Pricing and Cost Accounting Design

## Chapter Goals

Through this chapter, readers will understand:

- The role of model pricing data in the AI Gateway's cost accounting, quota deduction, and cost allocation;
- The field meanings and validation rules of the `ModelPrice` data model;
- The `model-list.yaml` bulk import format and how to use it;
- The design and implementation of tiered pricing for RMB quota;
- How the BFE Data Plane performs runtime cost calculation based on pricing data;
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
| `mode` | Request mode, e.g. `chat`, `completion`, `embedding`, `image_generation`, etc. |
| `capabilities` | List of capabilities supported by the model, e.g. `vision`, `reasoning`, `tools` |
| `supported_parameters` | List of supported request parameters, e.g. `temperature`, `max_tokens` |
| `limits` | Limits object, including context window, max input/output Tokens, etc. |
| `prices` | Default price table; used as fallback when no tier matches |
| `tier_prices` | Tiered price table, keyed by tier name (only `peak` is supported initially) |
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
| `output_cost_per_image` | Cost per output image |
| `output_cost_per_pixel` | Output cost per pixel |
| `input_cost_per_audio_per_second` | Input cost per second of audio |
| `input_cost_per_video_per_second` | Input cost per second of video |
| `output_cost_per_second` | Output cost per second |
| `ocr_cost_per_page` | OCR cost per page |
| `output_cost_per_character` | Output cost per character |

The current version primarily uses Token-based price fields; the remaining fields are reserved for future multimodal billing.

### Price Precision

Price fields in `prices` and `tier_prices` are floating-point numbers; 8 or more decimal digits are supported, e.g. `0.0000015`, `0.00000075`. To prevent the default JSON encoder from outputting very small values in scientific notation (e.g. `1.5e-6`), both AI Gateway API and BFE implement a custom `MarshalJSON` for `PriceMap` / `TierPriceMap`, forcing decimal representation. This representation only affects the readability of the configuration text; it does not change the `float64` value semantics nor affect BFE's internal fixed-point integer deduction logic.

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

`/providers` adds the `time_zone` and `tiers` fields to describe shared time rules:

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
│ Convert prices │───  prices / tier_prices converted to integer units
│ to fixed point │
└──────────────┘
```

BFE stores prices as fixed-point integers to avoid errors introduced by floating-point arithmetic at runtime. All price fields are scaled by a unified precision before participating in deduction calculations.

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
Parse TokenUsage
   ├── prompt_tokens
   ├── completion_tokens
   └── cached_tokens
   │
   ▼
Match ActiveTierName
   │
   ├── Tier matched ──► take tier_prices.<tier> prices
   └── No tier match ──► take prices default prices
   │
   ▼
Calculate separately
   ├── Cache-hit input = cached_tokens × cache_read_input_token_cost
   ├── Regular input   = (prompt_tokens - cached_tokens) × input_cost_per_token
   └── Output          = completion_tokens × output_cost_per_token
   │
   ▼
Accumulate into total request cost, used for RMB quota deduction and log output
```

If a tier does not define a specific price key, it automatically falls back to the corresponding key in the default `prices`. After `TokenUsage` adds the `CachedTokens` field, cache-hit and cache-miss input Tokens can be priced separately.

### Backward Compatibility

The tiered pricing design maintains backward compatibility with fixed pricing:

- When `time_zone` / `tiers` are not set on `/providers`, `ModelTable.TimeZone` / `ModelTable.Tiers` are empty, and behavior is identical to fixed pricing;
- When `tier_prices` is not set on `/model-prices`, billing always uses the default `Prices`;
- When a tier is matched but a price key is not configured for that tier, it automatically falls back to the default `Prices`;
- `TokenUsage.UsedCost`, the Lua deduction logic, and Redis fixed-point number storage require no changes.

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
- The BFE Data Plane converts prices to fixed-point integers at load time and performs pure-integer cost calculation at runtime based on Token usage and the active tier, avoiding floating-point errors.
- After the Provider and Cluster concepts were separated, `model-prices.provider` serves only as a price grouping identifier and has a weak reference relationship with `/providers`, making configuration more flexible; `AIConf.ModelTable` is assembled by the Control Plane by provider at export time.

---

## References

- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/model-prices.md`
- `ai-gateway-api/design-docs/sys-design/details/RMB配额分时段定价.md`
- `ai-gateway-api/design-docs/sys-design/details/provider与cluster概念分离.md`
