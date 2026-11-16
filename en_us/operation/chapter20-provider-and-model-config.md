# Chapter 20: Provider and Model Configuration

## Chapter Goals

Through this chapter, readers will learn the role and positioning of the Provider in the Rainway AI Gateway, become proficient in creating and maintaining Providers via the Dashboard and OpenAPI, correctly configure the model endpoint, model list, Provider Keys, and backend instance pool (IP mode and provider domain mode), use the model discovery tool to automatically probe models, understand the differences among the `openai`, `anthropic`, and `gemini` model protocols, master the common operations on the provider list (querying model prices, details, tiered pricing configuration, and deletion) and the configuration of peak time ranges, master the method of batch-importing model pricing via model-list.yaml, and clarify the relationship between Provider and Cluster and the impact of changes, as well as how to configure the `protocol_paths` protocol path mapping for multi-protocol providers and its caveats.

## The Concept and Role of the Provider

In the Control Plane of the Rainway AI Gateway, a **Provider** describes a downstream model provider: which models it offers, which protocol is used for access, where the backend instances are, and which API Keys are used. It can be understood as the abstraction layer answering "whom I can access." Correspondingly, the **Cluster** is responsible for "how to forward requests there," including route matching, model mapping, Key weighting, retry policy, timeouts, and so on.

After Provider and Cluster are separated, their responsibilities become clearer:

- Provider = "who I am, which models I can access, and what my backends and keys are."
- Cluster = "how I forward, which models I use, and how Key weights are allocated."

This separation brings many benefits: when the same Provider is referenced by multiple Clusters, the instance pool and keys only need to be maintained in one place, avoiding duplicate configuration; the Cluster no longer stores API Key plaintext and only references Keys in the Provider by name, improving security; the Provider can be created, updated, and deleted independently, while Clusters obtain backend capability by reference, making their lifecycles more independent; when a new protocol is added, only the Provider's model_protocols needs to be extended, without causing the Cluster model list to continuously grow.

The core fields of a Provider include: a globally unique `name`, an optional `description`, the model discovery endpoint `model_endpoint`, the model list `models`, the API Key list `keys`, the backend instance pool `instance_pool`, the supported protocols `model_protocols`, the optional protocol path mapping `protocol_paths`, the time zone `time_zone`, and the time-of-day templates `tiers`. Among them, `models` is required and must contain at least one element; `instance_pool` is required and must contain at least one instance with weight greater than 0; and `model_protocols` is required and must contain at least one protocol.

The default value of `time_zone` is `Asia/Shanghai`, used to determine which tier the current time belongs to. In the initial phase, `tiers` only supports `name=peak`, and each tier contains several `time_ranges` with a left-closed, right-open semantics. The time zone and tiers can be maintained separately via `PUT /v1/providers/{provider_name}/pricing-tiers`, without being passed when the Provider is created.

## Steps to Create a Provider

### Creating via the Dashboard

The Dashboard is a visual console for operators. After logging in, go to **Resource Management → Model Providers** and click **Create Provider**; a drawer form slides in from the right, divided into five Card sections: Basic Information, Instance Pool, Model Service Configuration, Protocol Path Mapping, and Service Authentication Keys. Clicking **Edit** in the action column opens the same drawer: the name field is grayed out (the provider name cannot be changed after creation), and leaving the value of an already configured Key blank means keeping the existing value.

The minimal configuration for first-time onboarding is as follows:

| Section | Required Items |
|---------|----------------|
| Basic Information | Provider name |
| Instance Pool | IP or provider domain + port + weight (fill 100 for a single instance) |
| Model Service Configuration | Model protocol (at least one), model list API path, model list (required, at least 1 model; click "Get" to fetch, press Enter after typing to add, or use "Batch Add" to paste multiple names; must submit the form to persist) |
| Service Authentication Keys | Usually required for providers that need authentication, such as public clouds |

**Basic Information**:

| Field | Required | Validation Rules |
|-------|----------|------------------|
| Name | Yes | 1–64 characters; only letters, digits, dots (`.`), underscores (`_`), and hyphens (`-`) are allowed; cannot start or end with a dot, underscore, or hyphen; cannot contain whitespace; cannot duplicate an existing provider name at creation. The unique identifier, which cannot be modified after creation |
| Description | No | No more than 256 characters; cannot contain control characters |

**Instance Pool**: supports either **IP Mode** or **Provider Domain Mode**, one of the two.

> **A third source: the K8s Pool**. When the instance source is a K8s pool, the pool name is a **dropdown** (data source: the read-only API `GET /open-api/v1/k8s_pools`), and the edit/detail pages render the instance table (address/port/weight) and the pool state from a **live query** to `GET /open-api/v1/k8s_pools/{name}` — a missing pool (404) and an empty pool each get their own hint; `disable` is read-only and no switch is rendered; the legal pool-name length is 128 characters. Providers store no instance mirror (there is no `k8s_instance_pool` field) — the instance list always comes from the live pool query. See "K8s Pool Provisioning" in [Chapter 10 Provider and Cluster Design](../design/chapter10-provider-and-cluster.md).

IP Mode (the default) is suitable for self-built data centers / private deployments (vLLM / Xinference / Ollama, etc.) and allows creating multiple instance rows with "+ Create":

| Field | Required | Default | Validation Rules |
|-------|----------|---------|------------------|
| IP Address | Yes | Empty | Valid IPv4 / IPv6; IPs must not be duplicated across instances |
| Port | Yes | `80` | 1–65535 |
| Weight | Yes | `100` for the first row, `0` for new rows | 0–100; the weights of all instances must sum to 100 |

Provider Domain Mode is suitable for connecting to public-cloud model providers: fill in a single domain only (e.g., `api.deepseek.com`); the port defaults according to the protocol (https→443, http→80), and the weight is fixed at 100. Multiple addresses are not supported, and it cannot be mixed with IP Mode.

**Model Service Configuration**:

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| Model Protocol | Yes | Empty | Multi-select; supports `openai` (OpenAI-compatible), `anthropic` (Claude Messages API), and `gemini` (Google Gemini API); at least one must be selected |
| Model List API | No | `https://{instance address}/v1/models` (the `gemini` protocol defaults to `/v1beta/models`) | Composed of protocol + instance address (read-only, from the instance pool) + URI path; the path must start with `/`. Corresponds to the Provider resource's `model_endpoint` (`schema` is the selected protocol, `addr` comes from the instance pool) |
| Model List | Yes | Empty | Tag multi-select, at least 1 model; see "Maintaining the Model List" below |

The system automatically determines the authentication header style used when calling the model discovery API based on the selected model protocol (`openai` uses `Authorization: Bearer`, `anthropic` uses `x-api-key`, and `gemini` uses `x-goog-api-key`). Customizing the `Authorization` Header in the model list API is **no longer supported**.

**Maintaining the Model List**: the model list is required (at least 1 model) and carries a red required asterisk on the left of the tags. Validation rules: if the list is empty on submission, the prompt "Model list is required; please add at least 1 model" is shown; a model name cannot be an empty string; model names cannot be duplicated within the same provider. Common operations:

| Operation | Description |
|-----------|-------------|
| "Get" | Calls the stateless discovery API `POST /v1/providers/tools/discover-models` to fetch upstream models using the current form's protocol, instance pool, model list path, and first Key, and **overwrites** the dropdown with the result |
| "Batch Add" | Opens a text box; paste multiple model names separated by lines, commas / Chinese commas / semicolons / whitespace; after confirmation they are **merged** into the existing list (de-duplicated, existing items skipped) without overwriting |
| Dropdown input | Type a model name and press Enter to add it; if the clipboard text can be split into 2 or more model names, pasting automatically splits it into multiple tags; a single name is still added via Enter |
| Grayed-out condition | The "Get" button is disabled when no model protocol is selected or the instance pool has no valid address |
| Submit | A successful "Get" or "Batch Add" does **not** save automatically; you must click **Submit** at the bottom of the drawer to write the Provider resource |

> **Common pitfall**: if "Get" does not respond or fails, first check whether the instance address, port, protocol, and Key are correct, and whether BFE / the API Server can reach that backend.

**Protocol Path Mapping**: corresponds one-to-one with the `protocol_paths` field of the Provider resource and is used to specify the upstream base path for each checked multi-protocol option; see "Protocol Path Configuration (protocol_paths)" below for configuration constraints and forwarding behavior. The `gemini` protocol does not support path rewriting.

**Service Authentication Keys**: configure when the backend model service requires authentication. Key plaintext is stored only in the Provider resource, and Clusters reference Keys by **name**.

| Field | Required | Validation Rules |
|-------|----------|------------------|
| Key Name | Conditionally required (once any field in the row is filled, the whole row becomes required) | 1–128 characters; cannot be duplicated within the same provider; used for selection in the Cluster-side dropdown |
| Key Value | Conditionally required | 1–512 characters; leaving it blank during editing means no change |

Use "+ Add Key" to add a row and "Delete" to remove a row; empty rows do not participate in validation and are filtered out automatically on submission; in edit mode, an already configured Key shows the hint "Key already configured; enter a new Key to change it, or leave blank to keep it unchanged".

On save, the system validates field legality; on success it returns the complete Provider record including `create_time` and `update_time`. If validation fails, the Dashboard reports the specific field errors, such as a duplicated instance pool address, a protocol outside the enum range, an invalid Key name, or duplicated elements in `models`.

### Creating via OpenAPI

OpenAPI is suitable for automation scripts, CI/CD, or third-party system integration. Compared with the Dashboard, OpenAPI is better suited for batch creation, version-controlled management, and integration with internal platforms. The endpoint for creating a Provider is:

```http
POST /v1/providers
Content-Type: application/json
```

Example request body:

```json
{
    "name": "deepseek",
    "description": "DeepSeek official API",
    "model_endpoint": { "schema": "https", "uri": "/v1/models" },
    "models": ["deepseek-chat", "deepseek-coder"],
    "keys": [
        { "name": "key-primary", "key": "sk-aaaaaaaaaaaa" },
        { "name": "key-secondary", "key": "sk-bbbbbbbbbbbb" }
    ],
    "instance_pool": [
        { "addr": "api.deepseek.com", "weight": 100, "port": 443 }
    ],
    "model_protocols": ["openai"]
}
```

If the request is valid, the API returns `ErrNum=200` and carries the complete record in `Data`. If `model_endpoint`, `keys`, or `time_zone` are not passed, the system fills in the documented default values. Later, some fields can be updated via `PATCH /v1/providers/{provider_name}`.

## Provider List and Common Operations

Go to **Resource Management → Model Providers** to see the list of currently created providers. The list fetches all data at once; searching and sorting by name / description / protocol / model are all performed in the browser.

| Column | Description |
|--------|-------------|
| Name | The provider's unique identifier; supports search and sorting by name |
| Description | Provider description |
| Protocol | Supported model access protocols, e.g. `openai`, `anthropic`, `gemini`; multiple protocols are shown comma-separated |
| Models | The configured model list; at most 2 tags are shown, with `+N` for the remainder; hover the Tooltip to see the full list |
| Actions | Details, Query Model Prices, Tiered Pricing Configuration, Edit, Delete |

The "Create Provider" button sits above the list.

### Querying Model Prices

Click "Query Model Prices" to jump to the **Resource Management → Model Pricing** page and filter the list by that provider's name (the `provider` field); if no pricing records exist, the prompt "No model pricing found for provider {provider}" is shown. The provider name must match the `provider` field in the pricing table so that cost accounting and this jump are correctly attributed; for how the pricing fields are maintained, see "Model Pricing Import" and [Chapter 13: Model Pricing and Cost Accounting Design](../design/chapter13-model-pricing.md).

### Viewing Details

Click "Details" to open a read-only drawer showing the basic information (including creation time and update time), instance pool, model service configuration (model protocols, model list API, model list), protocol path mapping, service authentication Keys (Key values masked), and a tiered pricing configuration summary (time zone and peak time range table; "Not configured" is displayed when not configured).

### Tiered Pricing Configuration

Click "Tiered Pricing Configuration" in the action column to open a standalone drawer used to configure the provider's **peak (busy hours)** time ranges, which are used for time-of-day price matching in model pricing. This configuration corresponds to the Provider resource's `time_zone` and `tiers` fields (see "The Concept and Role of the Provider"), but on submission it is saved separately by calling `PUT /v1/providers/{name}/pricing-tiers`, without affecting other provider fields.

| Field | Required | Description |
|-------|----------|-------------|
| Provider Name | — | Read-only |
| Time Zone | Yes | IANA time zone name, e.g. `Asia/Shanghai`; must pass validity validation |
| Pricing Tier | — | Shows the "Peak" label read-only; only `peak` is supported in the initial phase |
| Time Ranges | Yes | A table for configuring applicable weekdays and start/end times; supports the "All / Weekdays / Weekend" shortcuts; at least 1 row must be kept; multiple ranges within the same tier are ORed |

Time rules: start/end times use the `HH:MM` format; the end time must be later than the start time—split into two ranges when crossing midnight; empty applicable ranges mean every day; weekday values are 0=Sunday … 6=Saturday; multiple time ranges within the same tier are matched by OR, with the semantics `start <= current time < end` (left-closed, right-open). After configuration, you must also maintain a `peak` tier price item for the same `provider` on the model pricing page; requests that do not hit peak hours fall back to the default price.

### Batch Task Pricing (`batch_discount`)

Model pricing supports a batch-task discount: `batch_discount` on a price row (value in `(0,1]`) expands that row's token prices by the coefficient into `mode=batch` price rows distributed with the `ModelTable`, and the `mode` whitelist includes `batch`; manually maintained `mode=batch` price rows take precedence over coefficient expansion. Rows without a discount produce no batch price, and batch requests are billed at the chat base price. For the design and billing semantics, see [Chapter 13 Model Pricing and Cost Accounting Design](../design/chapter13-model-pricing.md).

The discount is maintained in the model pricing form: the input sits inside the "Price" card of the create / edit drawer, after the time-segmented prices, is labeled "Batch Discount Coefficient", takes a 0–1 number with a step of 0.1 (placeholder `0.5`), and defaults to `0.5` for a new price row. UI rules:

| Item | Description |
|------|------|
| Validation | A floating-point number within `(0,1]`; an invalid value blocks submission with "The batch discount coefficient must be a floating-point number within (0,1]" |
| Display condition | Not rendered when "Model Mode" (`mode`) is `batch`; `mode=batch` price rows do not carry a batch discount coefficient |
| Submit semantics | Attached only when the model mode is not `batch` and a coefficient is filled in; clearing it is treated as not configured and no batch price rows are expanded on export |
| Scope of effect | Affects only the batch price rows distributed to BFE (every token price key plus each tier price expanded by the coefficient); it does not rewrite the stored prices of the row |
| View mode | For non-`batch` modes the coefficient value is shown (`-` when unset); for `mode=batch` it shows "- (mode=batch price rows don't have a batch discount coefficient)" |

The equivalent OpenAPI field is `batch_discount` on the price row, which can be maintained through `POST/PUT /model-prices`.

### Deleting a Provider

Click "Delete" and confirm. If the provider is still referenced by a business Cluster, the deletion fails with the prompt "Deletion failed: this provider may still be referenced by a cluster"; you must first change the "Provider" of the relevant Clusters or delete those Clusters before deleting the provider. Note: a `provider` with the same name existing in `model-prices` does **not** prevent the provider from being deleted.

## Configuring the Model Endpoint, Model List, and Provider Keys

### Model Endpoint

`model_endpoint` is used to call the third-party platform's model list API and contains two fields, `schema` and `uri`. The default value of `schema` is `https`, and `http` is also allowed; the default value of `uri` is `/v1/models` (or `/v1beta/models` for the `gemini` protocol), which must be non-empty and start with `/`. This endpoint is mainly used by the model discovery tool and does not directly affect BFE forwarding target addresses. The system no longer allows configuring `headers.Authorization` in `model_endpoint`; when calling the model discovery API, the authentication header style is determined automatically by `model_protocols`: `openai` uses `Authorization: Bearer {apikey}`, `anthropic` uses `x-api-key: {apikey}`, and `gemini` uses `x-goog-api-key: {apikey}`.

### Model List

The `models` field is the list of model names supported by the Provider, for example `["deepseek-chat", "deepseek-coder"]`. Model names can be filled in directly at creation time, or automatically backfilled later via the model discovery tool. When updating a Provider, `models` is treated as a full replacement; if a model being removed is still referenced by a Cluster, the system returns `409 Conflict` to prevent mis-deletion from breaking routes.

### Provider Keys

The `keys` field stores the API Key plaintext available to the Provider; each element contains `name` and `key`. The `name` must be unique within the Provider, is 1–128 characters long, and is the link through which Clusters reference Keys. A Cluster's `llm_config.keys` only keeps `name` and `weight`, for example:

```json
{
    "keys": [
        { "name": "key-primary", "weight": 70 },
        { "name": "key-secondary", "weight": 30 }
    ]
}
```

When updating a Provider's `keys`, the system checks whether any Cluster still references a deleted or renamed Key; if such a reference exists, it also returns `409 Conflict`. This mechanism prevents Key changes from causing running Clusters to fail authentication.

## Using the Model Discovery Tool

The model discovery tool automatically probes the list of models currently supported by a third-party platform, avoiding manual maintenance of model names. The endpoint is:

```http
POST /v1/providers/tools/discover-models
```

Example request body:

```json
{
    "model_protocol": "openai",
    "schema": "https",
    "addr": "api.deepseek.com",
    "port": 443,
    "uri": "/v1/models",
    "apikey": "sk-aaaaaaaaaaaa"
}
```

At execution time, if `uri` is empty, `/v1/models` is used by default (`/v1beta/models` for the `gemini` protocol); the system generates the corresponding authentication header based on `model_protocol` (`openai` uses `Authorization: Bearer`, `anthropic` uses `x-api-key`, and `gemini` uses `x-goog-api-key`), calls `{schema}://{addr}:{port}{uri}`, and uses the corresponding protocol parser to extract the list of model names. The result is a string array that can be copied directly into the Provider's `models` field. This API is a stateless tool and does not modify the Provider directly; to backfill, you must call `PATCH /v1/providers/{provider_name}` to write `models`. The "Get" button in the Dashboard creation drawer simply wraps this API, using the current form context (protocol, instance pool, model list path, and first Key) to fetch and backfill.

## Supported Model Protocols

A Provider declares the supported model access protocols via the `model_protocols` field. The current enum values include:

| Enum Value | Description | Model Discovery Auth Header | Default Model List API Path |
|--------|------|----------------|----------------------|
| `openai` | OpenAI-compatible protocol, including most domestic Chinese compatible platforms | `Authorization: Bearer {apikey}` | `/v1/models` |
| `anthropic` | Anthropic Claude Messages API | `x-api-key: {apikey}` | `/v1/models` |
| `gemini` | Google Gemini API | `x-goog-api-key: {apikey}` | `/v1beta/models` |

A Provider can support multiple protocols at the same time; for example, an aggregation platform can be configured with `["openai", "anthropic"]`, but at least one protocol is required. The `gemini` protocol has two special properties: first, the default path of its model discovery API is `/v1beta/models`; second, `protocol_paths` does not support path rewriting for `gemini` (its native paths are already the standard paths and work in pass-through mode). See "Protocol Path Configuration (protocol_paths)" below for details.

`model_protocols` affects the forwarding behavior of the BFE Data Plane. For authentication header injection, `openai` uses `Authorization: Bearer`, `anthropic` uses `x-api-key`, and `gemini` uses `x-goog-api-key`; Claude requests also require injecting `anthropic-version`. Usage parsing handles different response formats by protocol style—for example, the OpenAI-style `usage` field and the Claude-style `usage` field have different structures. Protocol matching validation checks whether the request's protocol style is in the `model_protocols` of the Provider corresponding to the target Cluster; if not, the request is rejected directly. When the Control Plane generates the BFE configuration, it passes `provider.model_protocols` through to `AIConf.ModelProtocols` for the Data Plane to use.

## Protocol Path Configuration (protocol_paths)

Different providers use different upstream path prefixes, and different protocols of the same provider often live under different prefixes: Bailian DashScope exposes OpenAI compatibility at `/compatible-mode/v1` and Anthropic compatibility at `/apps/anthropic`; Volcano Ark pay-as-you-go uses `/api/v3` and `/api/compatible`; Kimi Code membership (api.kimi.com) uses `/coding/v1` and `/coding`. Without `protocol_paths`, BFE forwards the upstream path unchanged and clients must send requests using the provider's native paths; with it, clients can uniformly access all providers through OpenAI endpoint paths (for the openai protocol, with or without the `/v1` prefix; for the anthropic protocol, through the standard entry `/v1/...`), and BFE rewrites the path during forwarding.

`protocol_paths` is a mapping of "protocol -> upstream base path", whose value is the path part of the protocol's official SDK `base_url`: `openai` includes the `/v1` tail (e.g., `/compatible-mode/v1`), while `anthropic` does not (e.g., `/apps/anthropic`, since the Anthropic SDK appends `/v1/messages` itself). When configuring, you can copy the base_url column directly from the provider's official documentation. Reference values for common providers:

| Provider | protocol_paths |
|----------|----------------|
| Bailian DashScope | `{"openai": "/compatible-mode/v1", "anthropic": "/apps/anthropic"}` |
| Kimi Open Platform (api.moonshot.cn) | `{"openai": "/v1", "anthropic": "/anthropic"}` |
| Kimi Code Membership (api.kimi.com) | `{"openai": "/coding/v1", "anthropic": "/coding"}` |
| DeepSeek | `{"openai": "/v1", "anthropic": "/anthropic"}` |
| Volcano Ark · Pay-as-you-go | `{"openai": "/api/v3", "anthropic": "/api/compatible"}` |
| Volcano Ark · Coding Plan | `{"openai": "/api/coding/v3", "anthropic": "/api/coding"}` |

Configuration constraints:

- The key must be `openai` or `anthropic` already declared in the Provider's `model_protocols`; paths cannot be configured for undeclared protocols; `gemini` does not support path rewriting and cannot be used as a key.
- The value must start with `/`, must not end with `/`, must not contain `..`/`?`/`#`, and must be no longer than 128 characters.
- Omitting the field means it is disabled and request paths are forwarded unchanged.
- PATCH updates follow the partial-update convention: omitting the field keeps the current value; explicitly passing `null` clears it (restoring pass-through).

Forwarding behavior (see `rewriteUpstreamPath` in `bfe/bfe_server/ai_path_rewrite.go`):

- openai protocol: client entries with or without the `/v1` prefix can both be rewritten. An optional `/v1` prefix is stripped first, and the path is rewritten only when it hits the `bfe_basic` shared endpoint table (`openAIEndpointModes` in `bfe/bfe_basic/openai_endpoint.go`, 13 endpoints): both `/v1/chat/completions` and `/chat/completions` are rewritten to `{openai value}/chat/completions`, and `/v1` or `/v1/` is rewritten to the `{openai value}` itself. Custom paths that miss the endpoint table are forwarded unchanged, remaining compatible with existing clients.
- anthropic protocol: only the standard entry `/v1/...` is rewritten (`/v1/messages` -> `{anthropic value}/v1/messages`); non-standard entries are forwarded unchanged. When no path is configured for the protocol, no rewrite occurs.
- gemini protocol: never rewritten. `ProtocolPaths` only allows `openai`/`anthropic` as valid keys, and gemini native paths (e.g. `/v1beta/models/gemini-2.5-flash:generateContent`) are always passed through.

Caveats:

- **Billing-domain mismatch**: on some providers the path carries billing semantics (Volcano `/api/v3` pay-as-you-go vs `/api/coding` subscription). A subscription Key mistakenly configured with a pay-as-you-go prefix will be billed incorrectly, so verify the Key type matches the path before configuring.
- **Model discovery endpoint is not linked**: `model_endpoint` is not derived from `protocol_paths`. For example, Anthropic-protocol model discovery on Bailian requires manually setting `model_endpoint.uri` to `/apps/anthropic/v1/models`.
- **Release ordering**: `protocol_paths` depends on the BFE Data Plane supporting the `AIConf.ProtocolPaths` rewrite; the configuration has no effect on older BFE versions.

Configuration example (Bailian):

```json
{
    "name": "bailian",
    "model_protocols": ["openai", "anthropic"],
    "protocol_paths": {
        "openai": "/compatible-mode/v1",
        "anthropic": "/apps/anthropic"
    }
}
```

## Model Pricing Import

Model pricing is used for cost accounting and quota deduction and is stored in the `/model-prices` resource. To facilitate batch maintenance, the system supports importing the entire table via a `model-list.yaml` file. The API is:

```http
POST /v1/model-prices/import
Content-Type: multipart/form-data
```

The form parameters include `file` (YAML file, required) and `mode` (import mode; optional `replace` for full replacement by default, or `merge` for incremental merge).

Example `model-list.yaml`:

```yaml
version: v1.0
default_currency: "RMB"

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
    tier_prices:
      peak:
        input_cost_per_token: 0.000004
        output_cost_per_token: 0.000016
    metadata:
      source: "https://platform.deepseek.com/pricing"
      notes: "DeepSeek V3"
```

During import, the system validates the version, currency, uniqueness of `provider/model/mode`, required fields, and non-negativity of prices. `prices` must contain at least one price field; all price fields must be non-negative and support precision of 8 or more decimal places. If a record contains `tier_prices`, in the initial phase the tier name only supports `peak`.

The `replace` mode first clears the `model_prices` table and then writes the new data, suitable for fully refreshing an official price list; the `merge` mode updates existing `(provider, model, mode)` records and inserts new ones, suitable for incremental patches. The import API can only be called by administrators.

> Note: the `provider` field of `/model-prices` serves only as a price-aggregation identifier and no longer enforces a reference to an existing `/providers`. Therefore, prices can be imported first and the Provider created later; the two are maintained independently.

### Price Fields and Billing Modes for Image Input and Video Generation

The `ModelTable` in the BFE Data Plane cluster_conf (which includes two levels of price tables, global `Prices` and `TierPrices`; in the initial phase only `peak` is supported as a tier name) includes two price fields for image input and video generation, configurable in both `prices` and `tier_prices`:

| Field | Applicable Mode | Description |
|-------|----------------|-------------|
| `input_cost_per_image_token` | `image_generation` | Unit price for image input tokens; added on top of the per-image fee `output_cost_per_image` |
| `output_cost_per_video` | `video_generation` | Unit price per generated video, billed by the number of videos generated |

Two more billing modes (modes) are supported:

| mode | Billing Method |
|------|---------------|
| `responses` | Billed like chat, i.e., token billing with `input_cost_per_token` / `output_cost_per_token` and related fields (the `/v1/responses` endpoint and provider-native path forms of `/responses` are recognized as this mode) |
| `video_generation` | Cost = `VideoCount × output_cost_per_video` (the `/v1/video/generations` path is recognized as this mode) |

For `video_generation` mode, BFE pre-reads the `n` field of the request body as a fallback for the generation count during the authentication phase: if the field is missing or invalid, it is billed as 1 to avoid under-billing; `video_count` in the response `usage` takes precedence as the final count. Correspondingly, usage statistics include the `image_input_tokens` and `video_count` fields, which map to the access log fields `ai_image_input_tokens` (786) and `ai_video_count` (787) respectively (bfe-access-pb v0.3.5).

Implementation references: `ModelPrice` and the price field constants in `bfe/bfe_config/bfe_cluster_conf/cluster_conf/cluster_conf_load.go`, `DetectModeFromPath` in `bfe/bfe_basic/request_ai_basic.go`, `openAIEndpointModes` in `bfe/bfe_basic/openai_endpoint.go` (the shared OpenAI endpoint table, 13 endpoints), and `calcVideoGenerationCost` / `calcResponsesCost` in `bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`. See `bfe/docs/zh_cn/sys_design/rmb_quota.md` for the complete billing semantics.

Mode detection and the upstream path rewrite share the same endpoint table, so the two can never disagree on what an OpenAI endpoint is: `DetectModeFromPath` first strips the optional `/v1` prefix (the recognition result is the same whether or not the standard entry carries `/v1`), and reduces a provider-native prefix ending in a `/v1` segment (the OpenAI SDK base_url form, e.g. Bailian's `/compatible-mode/v1/responses`) to the same endpoint before the lookup. Requests sent via the provider's native path are therefore recognized with the correct mode and billed correctly; paths that miss the endpoint table fall back to the default chat mode.

## The Relationship Between Provider and Cluster

A Provider and a Cluster are linked by a strong reference via `cluster.llm_config.provider`. The recommended configuration order is:

```text
/providers → /model-prices → /clusters → route rules
```

A Cluster can reference only one Provider, but one Provider can be referenced by multiple Clusters. Example of a Cluster's `llm_config`:

```json
{
    "name": "my-cluster",
    "llm_config": {
        "provider": "deepseek",
        "models": ["deepseek-chat", "deepseek-coder"],
        "keys": [
            { "name": "key-primary", "weight": 70 },
            { "name": "key-secondary", "weight": 30 }
        ],
        "key_policy": { "strategy": "weighted_random", "max_retries": 3 },
        "match_prefix": "deepseek/",
        "strip_prefix": true
    }
}
```

Key constraints: `llm_config.provider` is required and must reference an existing Provider; `llm_config.models` must be a subset of the Provider's `models`; the `name` values in `llm_config.keys` must correspond to Keys that exist in the Provider. Changes to a Provider's `instance_pool` are automatically synchronized to all Clusters referencing it, so one change takes effect globally. Before deleting a Provider, you must ensure that no Cluster references it; otherwise, `409 Conflict` is returned.

The configuration ultimately received by BFE is generated automatically by the Control Plane: `AIConf.Keys` joins the Provider's Key plaintext with the Cluster's Key weights by `name`; `AIConf.ModelProtocols` passes through the Provider's protocol list; `AIConf.ModelTable` is automatically populated by the Control Plane by querying `model-prices` based on the Provider. Therefore, a Cluster does not need to care about Key plaintext or price data and only needs to maintain forwarding policies.

## Common Issues and Troubleshooting

### 1. "instance_pool invalid" when creating a Provider

Check whether at least one instance is filled in; whether IPs are duplicated across instances in IP Mode; whether the port is in the 1–65535 range; whether weights are between 0 and 100 and the weights of all instances sum to 100 (in Provider Domain Mode the weight is fixed at 100 and not subject to this constraint); and whether at least one instance has `weight > 0`.

### 2. 409 returned when updating a Provider's Keys or Models

This means a Cluster still references the deleted/renamed Key or the removed Model. Resolution steps: first modify the `llm_config.keys` or `models` of the corresponding Cluster to remove the reference, and then update the Provider.

### 3. 409 returned when deleting a Provider

The Provider is still referenced by at least one Cluster. Delete or modify the Clusters referencing it first, then delete the Provider.

### 4. Model discovery returns an empty list or an error

Check whether `model_protocol` matches the actual platform; whether `addr`, `port`, and `uri` are correct; and whether `apikey` is valid. For the Claude platform, confirm that `model_protocol` is set to `anthropic`; for the Google Gemini platform, choose `gemini` (the model discovery default path is `/v1beta/models` and the authentication header is `x-goog-api-key`). If clicking "Get" does not respond, first check whether the instance address, port, protocol, and Key are correct, and whether BFE / the API Server can reach that backend.

### 5. Forwarding fails after a Cluster references a Provider

Check whether the Cluster's `llm_config.models` is a subset of the Provider's `models`; whether the `name` values in `llm_config.keys` exist in the Provider's `keys`; and whether the request's protocol style is in the Provider's `model_protocols`.

### 6. Model prices not taking effect

Check whether a `(provider, model, mode)` record exists in `/model-prices`; whether the `model-list.yaml` import succeeded, paying attention to the `errors` list; whether `price_currency` is `RMB`; and whether the price fields in prices and tier_prices are non-negative.

### 7. protocol_paths is configured but requests are not rewritten (upstream 404)

First confirm that the request path hits the shared OpenAI endpoint table: for the openai protocol, entries with or without the `/v1` prefix can both be rewritten, but a custom path that misses the endpoint table (`bfe_basic/openai_endpoint.go`) after the optional `/v1` prefix is stripped will be forwarded unchanged, which is expected. For the anthropic protocol, only the standard entry `/v1/...` is rewritten; non-standard entries pass through. Then confirm that the request protocol is declared in `model_protocols` and has a corresponding `protocol_paths` entry; that the BFE version supports `AIConf.ProtocolPaths` (a key whitelist is validated at load time, and invalid configurations are rejected with the offending cluster identified); and, if a fallback occurred, that pass-through after switching to a backup cluster without `protocol_paths` is expected behavior.

## Configuration Examples

### Complete Provider JSON Configuration

```json
{
    "name": "anthropic",
    "description": "Anthropic Claude official API",
    "model_endpoint": { "schema": "https", "uri": "/v1/models" },
    "models": ["claude-3-5-sonnet-20241022", "claude-3-opus-20240229"],
    "keys": [
        { "name": "key-prod", "key": "sk-ant-api03-xxxxxxxx" }
    ],
    "instance_pool": [
        { "addr": "api.anthropic.com", "weight": 100, "port": 443 }
    ],
    "model_protocols": ["anthropic"],
    "time_zone": "America/New_York",
    "tiers": [
        {
            "name": "peak",
            "time_ranges": [
                { "weekdays": [1, 2, 3, 4, 5], "start": "09:00", "end": "18:00" }
            ]
        }
    ]
}
```

### Model Discovery Request Example

```bash
curl -X POST https://control-plane.example.com/v1/providers/tools/discover-models \
  -H "Content-Type: application/json" \
  -d '{
    "model_protocol": "openai",
    "schema": "https",
    "addr": "api.deepseek.com",
    "port": 443,
    "apikey": "sk-aaaaaaaaaaaa"
  }'
```

Example with the `gemini` protocol (when `uri` is left empty, `/v1beta/models` is used by default and the `x-goog-api-key` authentication header is used automatically):

```bash
curl -X POST https://control-plane.example.com/v1/providers/tools/discover-models \
  -H "Content-Type: application/json" \
  -d '{
    "model_protocol": "gemini",
    "schema": "https",
    "addr": "generativelanguage.googleapis.com",
    "port": 443,
    "apikey": "AIzaSyaaaaaaaaaaaa"
  }'
```

### Cluster TOML Semantics Illustration

```toml
[cluster.my-cluster.llm_config]
provider = "deepseek"
models = ["deepseek-chat", "deepseek-coder"]
match_prefix = "deepseek/"
strip_prefix = true

[[cluster.my-cluster.llm_config.keys]]
name = "key-primary"
weight = 70

[[cluster.my-cluster.llm_config.keys]]
name = "key-secondary"
weight = 30

[cluster.my-cluster.llm_config.key_policy]
strategy = "weighted_random"
max_retries = 3
```

A Cluster does not declare `instance_pool`, `model_endpoint`, or `provider_type`; all of this information comes from the Provider.

## Chapter Summary

The Provider is the core resource in the Control Plane of the Rainway AI Gateway for describing downstream model providers. After the responsibilities of Provider and Cluster are separated, the Cluster focuses on forwarding policies while the Provider focuses on access information, improving configuration reusability, security, and maintainability.

Key points of this chapter: the data model and field meanings of the Provider; the flows for creating and updating Providers via the Dashboard and OpenAPI (the Dashboard drawer's five sections: Basic Information, Instance Pool, Model Service Configuration, Protocol Path Mapping, and Service Authentication Keys); the differences and weight constraints between the instance pool IP Mode and Provider Domain Mode; the configuration methods and constraints of the model endpoint, model list, and Provider Keys; using the stateless model discovery tool `/providers/tools/discover-models`; the common operations on the provider list, including querying model prices, viewing details, tiered pricing configuration (peak time ranges, IANA time zone, left-closed right-open matching), and deletion; the configuration methods, constraints, and forwarding behavior of the `protocol_paths` protocol path mapping; the impact of the `openai`, `anthropic`, and `gemini` protocols on authentication headers, version headers, usage parsing, and protocol matching; the process and caveats of batch-importing model pricing via `model-list.yaml`; the price fields for image input tokens and per-video billing, and the `responses` and `video_generation` billing modes; the strong reference relationship between Provider and Cluster and the synchronization and conflict handling during changes; and troubleshooting ideas for common issues plus configuration examples.

Properly planning the separation of Provider and Cluster is an important prerequisite for subsequent route rules, API-Key quotas, and rate limiting policies to take effect. It is recommended that in production you first maintain Providers and model prices in a unified way, and then create Clusters for different business lines as needed. Regularly comparing the provider lists returned by `/providers` and `/model-prices/actions/get-providers` helps promptly detect and backfill cases where price records drift from actual Providers, ensuring accurate cost accounting.

## References

- `ai-gateway-web/docs/zh-cn/03-model-provider.md` (Dashboard model provider user manual, the authoritative basis for the console content in this chapter)
- `ai-gateway-web/docs/zh-cn/06-model-prices.md` (Dashboard model pricing user manual)
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/providers.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/model-prices.md`
- `ai-gateway-api/design-docs/sys-design/details/provider与cluster概念分离.md`
- `ai-gateway-api/design-docs/sys-design/details/Claude协议转发支持.md`
