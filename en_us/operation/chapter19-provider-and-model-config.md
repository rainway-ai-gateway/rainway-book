# Chapter 19: Provider and Model Configuration

## Chapter Goals

Through this chapter, readers will learn the role and positioning of the Provider in the Rainway AI Gateway, become proficient in creating and maintaining Providers via the Dashboard and OpenAPI, correctly configure the model endpoint, model list, Provider Keys, and backend instance pool, use the model discovery tool to automatically probe models, understand the differences among supported model protocols, master the method of batch-importing model pricing via model-list.yaml, and clarify the relationship between Provider and Cluster and the impact of changes.

## The Concept and Role of the Provider

In the Control Plane of the Rainway AI Gateway, a **Provider** describes a downstream model provider: which models it offers, which protocol is used for access, where the backend instances are, and which API Keys are used. It can be understood as the abstraction layer answering "whom I can access." Correspondingly, the **Cluster** is responsible for "how to forward requests there," including route matching, model mapping, Key weighting, retry policy, timeouts, and so on.

After Provider and Cluster are separated, their responsibilities become clearer:

- Provider = "who I am, which models I can access, and what my backends and keys are."
- Cluster = "how I forward, which models I use, and how Key weights are allocated."

This separation brings many benefits: when the same Provider is referenced by multiple Clusters, the instance pool and keys only need to be maintained in one place, avoiding duplicate configuration; the Cluster no longer stores API Key plaintext and only references Keys in the Provider by name, improving security; the Provider can be created, updated, and deleted independently, while Clusters obtain backend capability by reference, making their lifecycles more independent; when a new protocol is added, only the Provider's model_protocols needs to be extended, without causing the Cluster model list to continuously grow.

The core fields of a Provider include: a globally unique `name`, an optional `description`, the model discovery endpoint `model_endpoint`, the supported model list `models`, the API Key list `keys`, the backend instance pool `instance_pool`, the supported protocols `model_protocols`, the time zone `time_zone`, and the time-of-day templates `tiers`. Among them, `instance_pool` is required and must contain at least one instance with weight greater than 0, and `model_protocols` is required and must contain at least one protocol.

The default value of `time_zone` is `Asia/Shanghai`, used to determine which tier the current time belongs to. In the initial phase, `tiers` only supports `name=peak`, and each tier contains several `time_ranges` with a left-closed, right-open semantics. The time zone and tiers can be maintained separately via `PUT /v1/providers/{provider_name}/pricing-tiers`, without being passed when the Provider is created.

## Steps to Create a Provider

### Creating via the Dashboard

The Dashboard is a visual console for operators. The standard workflow for creating a Provider is as follows:

1. Log in to the Dashboard, go to the **Provider Management** page, and click **New Provider**.
2. Fill in the basic information: `name` must be globally unique; lowercase letters and hyphens are recommended, such as `deepseek` or `openai-official`; `description` is optional and helps the team identify the purpose.
3. Configure the **instance pool**: fill in the backend address `addr`, port `port`, and weight `weight`. Within the same Provider, `(addr, port)` must not be duplicated, and at least one instance must have `weight > 0`.
4. Configure the **model endpoint**: the default protocol is `https` and the default URI is `/v1/models`. Most OpenAI-compatible platforms need no change; the Claude official API usually also uses `/v1/models`.
5. Select the **model protocols**: the first phase supports `openai` and `anthropic`; aggregation platforms may select multiple protocols at the same time.
6. Add **Provider Keys**: each Key requires a name `name` and the actual key value `key`. The name must be unique within the Provider, and Clusters later reference the Key by this name.
7. Save and submit. The system validates the fields; on success it returns the complete Provider record including `create_time` and `update_time`. If validation fails, the Dashboard reports the specific field errors, such as a duplicated instance pool, a protocol outside the enum range, an invalid Key name, or duplicated elements in `models`.

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

## Configuring the Model Endpoint, Model List, and Provider Keys

### Model Endpoint

`model_endpoint` is used to call the third-party platform's model list API and contains two fields, `schema` and `uri`. The default value of `schema` is `https`, and `http` is also allowed; the default value of `uri` is `/v1/models`, which must be non-empty and start with `/`. This endpoint is mainly used by the model discovery tool and does not directly affect BFE forwarding target addresses. The system no longer allows configuring `headers.Authorization` in `model_endpoint`; when calling the model discovery API, the authentication header style is determined automatically by `model_protocols`: `openai` uses `Authorization: Bearer {apikey}`, and `anthropic` uses `x-api-key: {apikey}`.

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

At execution time, if `uri` is empty, `/v1/models` is used by default; the system generates the corresponding authentication header based on `model_protocol`, calls `{schema}://{addr}:{port}{uri}`, and uses the corresponding protocol parser to extract the list of model names. The result is a string array that can be copied directly into the Provider's `models` field. This API is a stateless tool and does not modify the Provider directly; to backfill, you must call `PATCH /v1/providers/{provider_name}` to write `models`.

## Supported Model Protocols

A Provider declares the supported model access protocols via the `model_protocols` field. The enum values in the first phase include:

| Enum Value | Description |
|--------|------|
| `openai` | OpenAI-compatible protocol, including most domestic Chinese compatible platforms |
| `anthropic` | Anthropic Claude Messages API |

A Provider can support multiple protocols at the same time; for example, an aggregation platform can be configured with `["openai", "anthropic"]`, but at least one protocol is required.

`model_protocols` affects the forwarding behavior of the BFE Data Plane. For authentication header injection, `openai` uses `Authorization: Bearer` and `anthropic` uses `x-api-key`; Claude requests also require injecting `anthropic-version`. Usage parsing handles different response formats by protocol style—for example, the OpenAI-style `usage` field and the Claude-style `usage` field have different structures. Protocol matching validation checks whether the request's protocol style is in the `model_protocols` of the Provider corresponding to the target Cluster; if not, the request is rejected directly. When the Control Plane generates the BFE configuration, it passes `provider.model_protocols` through to `AIConf.ModelProtocols` for the Data Plane to use.

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

Check whether at least one instance is filled in; whether `(addr, port)` is duplicated; and whether at least one instance has `weight > 0`.

### 2. 409 returned when updating a Provider's Keys or Models

This means a Cluster still references the deleted/renamed Key or the removed Model. Resolution steps: first modify the `llm_config.keys` or `models` of the corresponding Cluster to remove the reference, and then update the Provider.

### 3. 409 returned when deleting a Provider

The Provider is still referenced by at least one Cluster. Delete or modify the Clusters referencing it first, then delete the Provider.

### 4. Model discovery returns an empty list or an error

Check whether `model_protocol` matches the actual platform; whether `addr`, `port`, and `uri` are correct; and whether `apikey` is valid. For the Claude platform, confirm that `model_protocol` is set to `anthropic`.

### 5. Forwarding fails after a Cluster references a Provider

Check whether the Cluster's `llm_config.models` is a subset of the Provider's `models`; whether the `name` values in `llm_config.keys` exist in the Provider's `keys`; and whether the request's protocol style is in the Provider's `model_protocols`.

### 6. Model prices not taking effect

Check whether a `(provider, model, mode)` record exists in `/model-prices`; whether the `model-list.yaml` import succeeded, paying attention to the `errors` list; whether `price_currency` is `RMB`; and whether the price fields in prices and tier_prices are non-negative.

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

Key points of this chapter: the data model and field meanings of the Provider; the flows for creating and updating Providers via the Dashboard and OpenAPI; the configuration methods and constraints of the model endpoint, model list, and Provider Keys; using the stateless model discovery tool `/providers/tools/discover-models`; the impact of the `openai` and `anthropic` protocols on authentication headers, version headers, usage parsing, and protocol matching; the process and caveats of batch-importing model pricing via `model-list.yaml`; the strong reference relationship between Provider and Cluster and the synchronization and conflict handling during changes; and troubleshooting ideas for common issues plus configuration examples.

Properly planning the separation of Provider and Cluster is an important prerequisite for subsequent route rules, API-Key quotas, and rate limiting policies to take effect. It is recommended that in production you first maintain Providers and model prices in a unified way, and then create Clusters for different business lines as needed. Regularly comparing the provider lists returned by `/providers` and `/model-prices/actions/get-providers` helps promptly detect and backfill cases where price records drift from actual Providers, ensuring accurate cost accounting.

## References

- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/providers.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/model-prices.md`
- `ai-gateway-api/design-docs/sys-design/details/provider与cluster概念分离.md`
- `ai-gateway-api/design-docs/sys-design/details/Claude协议转发支持.md`
