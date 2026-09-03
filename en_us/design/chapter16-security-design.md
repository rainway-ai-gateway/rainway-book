# Chapter 16: Security Design

## Chapter Goals

Through this chapter, readers will understand:

- What security mechanisms Rainway AI Gateway provides at the transport layer, the authentication/authorization layer, and the policy enforcement layer;
- How API-Keys, as the direct credential of the request path, are stored in the Control Plane, inherit policies through Entities, and are validated in the Data Plane;
- The Control Plane authentication/authorization model (Visitor, Scope, Feature-Action) and how it integrates with the OpenAPI / InnerAPI;
- Key configuration points for TLS/HTTPS on both the Control Plane and the Data Plane;
- How access-log audit fields support security incident tracing and billing reconciliation;
- The cleanup triggers for Quota Keys and Rate-Limit Keys in Redis, and practices for protecting sensitive data;
- How rate limiting and quota act as security defenses against abuse and runaway costs;
- How model allowlists and blocklists are inherited at the Entity hierarchy;
- Recommended security configuration examples for production environments.

---

## Overall View of Security Design

Rainway AI Gateway's security design follows the **Defense in Depth** principle: from the outside in, it consists of transport-layer security, authentication, authorization, request admission, policy enforcement, data cleanup, and audit tracing. Each layer has independent validation and interception capabilities, so even if one layer is bypassed, the remaining layers continue to provide protection.

```
┌─────────────────────────────────────────────────────────────────┐
│                  Client / Business Systems                       │
└───────────────────────┬─────────────────────────────────────────┘
                        │
           ┌────────────┴────────────┐
           ▼                         ▼
    TLS/HTTPS Transport        Client IP / Subnet
        Encryption                 Allowlist
           │                         │
           └────────────┬────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     BFE Data Plane                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ IP Subnet   │  │ API-Key     │  │ Model Allowlist /       │ │
│  │ Control     │  │ Auth & Quota│  │ Blocklist             │ │
│  │             │  │             │  │ (from API-Key / Entity)│ │
│  └─────────────┘  └──────┬──────┘  └─────────────────────────┘ │
│                          ▼                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ RPM Limit   │  │ TPM Limit   │  │ Concurrency Limit       │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Protocol adaptation, routing & forwarding,              │   │
│  │ backend model service invocation                        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                 AI Gateway API Control Plane                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ User/Token  │  │ Feature-    │  │ API-Key / Entity        │ │
│  │ Auth        │  │ Action Auth │  │ Management              │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ Quota Plan  │  │ Rate Limit  │  │ Redis Key Cleanup       │ │
│  │             │  │ Policy      │  │ (Quota / Rate-Limit)    │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

The diagram above shows how security mechanisms are distributed between the Control Plane and the Data Plane. The Control Plane is responsible for policy definition, credential management, and data cleanup; the Data Plane performs real-time authentication, rate limiting, quota enforcement, and blocklist checks. The following sections expand layer by layer.

---

## API-Key Secure Storage and Transport

### The Role and Storage of API-Keys

In Rainway AI Gateway, the **API-Key** is the direct credential used by business systems when calling large-model services. The Control Plane stores API-Keys in the `api_keys` table, with fields including the key value, status, expiration time `expired_time`, allowed subnets `allowed_subnets`, the attached Entity, and so on. The Data Plane BFE validates the API-Key from the request header in `mod_ai_token_auth`.

An API-Key is associated with an **Entity (business organization unit)** via `api_keys.entity_id`. Once attached to an Entity, the API-Key inherits from that Entity and its parent Entities:

- The model allowlist `allow_models` and blocklist `block_models`;
- The quota plan QuotaPlan;
- The rate limit policy RateLimitPolicy;
- The routing rules RouteRules.

This design allows administrators to uniformly manage policies at the organization level, while the API-Key serves only as the final usage credential. See `ai-gateway-api/design-docs/sys-design/details/API-Key与Entity关联及模型继承.md` for details.

### Transport Security

API-Keys are transported in HTTP headers and must be encrypted with **TLS/HTTPS** to prevent theft or tampering by man-in-the-middle attacks. In production, plaintext HTTP entry points should be disabled and HTTPS access enforced. It is also recommended to:

- Rotate API-Keys regularly and set expiration times for keys that have been unused for a long time;
- Use `allowed_subnets` to restrict the client IP ranges allowed to call;
- Avoid displaying the full API-Key in the Dashboard; show only the prefix and the last few characters;
- Immediately clean up the API-Key's Quota Keys and Rate-Limit Keys in Redis upon deletion, to prevent residual data from being reused.

---

## TLS/HTTPS Configuration

### Data Plane TLS Termination

BFE, as the Data Plane entry point, supports HTTPS listening and TLS termination. TLS-related configuration is located under `bfe_config/bfe_tls_conf/` and `conf/tls_conf/`, including certificate files, private key files, and TLS rule tables. Administrators can configure different certificates for different domains, and specify the minimum TLS version and cipher suites.

Production recommendations:

- Disable TLS 1.0/1.1; enable only TLS 1.2 and above;
- Use certificates issued by a trusted CA; avoid self-signed certificates;
- Set file permissions on certificate and private key files to `600` to prevent unauthorized reads;
- Enable the HSTS (HTTP Strict Transport Security) header to force clients to use HTTPS.

### Control Plane HTTPS

AI Gateway API and the Dashboard should also expose management interfaces over HTTPS. Although the current default configuration starts over HTTP, production deployments should place a reverse proxy such as Nginx/Envoy in front of AI Gateway API to perform TLS termination, or enable TLS directly at the application layer. The management plane involves sensitive operations such as API-Key creation, Token issuance, and password changes, so transport encryption is mandatory.

---

## Authentication and Authorization Mechanisms

### Visitor Model

The authentication/authorization module of AI Gateway API is located in `model/iauth`, and is responsible for visitor identity recognition and access control for both the OpenAPI and the InnerAPI. This module reuses a design from BFE's legacy code in which the `users` table stores both "users" and "tokens", distinguished by the `type` field:

| `type` value | Meaning | Typical use |
|-------------|---------|-------------|
| `0` | Regular user | Dashboard administrator login, manual operations |
| `1` | Token | Programmatic calls; Conf Agent / BFE pulling the InnerAPI |

The code uniformly abstracts users and tokens via `Visitor`:

```go
type Visitor struct {
    User  *User
    Token *Token
}
```

`Visitor` implements the `Loginer` interface, uniformly providing the `GetName`, `GetScopes`, `GetType`, and `IsAdmin` methods. Subsequent authorization checks only care about the `Visitor`, not whether the underlying identity is a user or a token. See `ai-gateway-api/design-docs/sys-design/details/认证授权机制.md` for details.

### Four Authentication Methods

The Control Plane supports four authentication methods:

```go
const (
    AuthTypePassword   = "Password"
    AuthTypeSessionKey = "Session"
    AuthTypeToken      = "Token"
    AuthTypeSkip       = "Skip"
)
```

| Method | Example request header | Applicable scenario | Session written? |
|--------|------------------------|---------------------|------------------|
| `Password` | `Authorization: Password <base64(user:pass)>` | Login to obtain a Session Key | Yes |
| `Session` | `Authorization: Session <session_key>` | Regular OpenAPI calls | No; only checks whether the ticket has expired |
| `Token` | `Authorization: Token <token_value>` | Programmatic calls / InnerAPI | No; the Token is valid long-term |
| `Skip` | `Authorization: Skip System` | Debug bypass | No; generates a forged Visitor |

`Skip` authentication takes effect only when `RunTime.SkipTokenValidate = true`; **it must never be enabled in production**.

### Feature-Action Permission Model

Authorization adopts the **Feature (functional dimension) + Action (operation dimension)** model:

```go
type FeatureAuthorition struct {
    Feature Feature
    Action  Action
}
```

`Action` uses bit masks: `ActionRead`, `ActionUpdate`, `ActionCreate`, `ActionDelete`, `ActionExport`, etc. Each OpenAPI endpoint declares its required permission via the `Authorizer` field, for example:

```go
var APIKeyCreateRoute = &xreq.Endpoint{
    Path:       "/api-keys",
    Method:     http.MethodPost,
    Authorizer: iauth.FA(iauth.FeatureAPIKey, iauth.ActionCreate),
}
```

`AuthorizeManager.Authorizate` performs the following steps:

1. Extract the `Visitor` from the `context`;
2. If `Visitor.IsAdmin()` returns `true`, allow the request directly;
3. Iterate over `Visitor.GetScopes()` and look up the Action bitmap of the corresponding Feature in `scope2permission`;
4. Determine whether the required Action is allowed;
5. If necessary, further validate the binding between the Visitor and the current product line.

### Middleware Chain

The Control Plane's global middleware chain is:

```
HTTP Request
    │
    ▼
MCRecovery ──► MCLogger ──► MCCors
    │
    ▼
/open-api/v1: McProductProbe ──► McUserProbe
/inner-api/v1: McUserProbe
```

`McUserProbe` parses the `Authorization` header and completes authentication, writing the `Visitor` into the `context`; `McProductProbe` parses the product-line context from the URL path. Each Endpoint with an `Authorizer` creates a separate Subrouter and mounts the authorization middleware.

---

## Access Log Auditing

### Data Plane Error Codes and Log Fields

At each stage of request processing (authentication, rate limiting, quota, forwarding), the BFE Data Plane returns structured error responses and outputs access logs via `mod_access_pb3`. Log fields related to security auditing include:

| Log field | Description |
|-----------|-------------|
| `ai_auth_reject_reason` | Error code when authentication/quota is rejected |
| `ai_auth_reject_quota_plans` | List of Quota Plan IDs with insufficient balance when rejected due to insufficient quota |
| `ai_auth_hit_quota_plans` | List of Quota Plan IDs that passed authentication with sufficient balance |
| `ai_rate_limit_hits` | List of triggered rate limit policies and rule names |

Typical error codes include `NO_API_KEY`, `INVALID_API_KEY`, `KEY_DISABLED`, `KEY_EXPIRED`, `SUBNET_NOT_ALLOWED`, `MODEL_NOT_ALLOWED`, `QUOTA_EXHAUSTED`, `RPM_LIMIT_EXCEEDED`, `TPM_LIMIT_EXCEEDED`, etc. See `bfe/docs/zh_cn/sys_design/ai_error_codes.md` for details.

### Error Response Body

Data Plane error responses adopt the OpenAI-compatible format, making it easier for upstream business systems to handle them uniformly:

```json
{
  "error": {
    "code": "QUOTA_EXHAUSTED",
    "type": "quota_error",
    "message": "Quota plan qplan-0001 exhausted.",
    "param": null,
    "details": {
      "api_key": "ak-2v8x9k3m7p",
      "key_id": "key-001",
      "quota_plan_id": "qplan-0001",
      "limit_type": "api_key_quota",
      "model": "gpt-4",
      "retry_after_seconds": 0
    }
  }
}
```

By centrally collecting access logs, the security team can identify abnormal call patterns, such as:

- A certain API-Key frequently triggering `INVALID_API_KEY` or `KEY_DISABLED` within a short period;
- Specific IP ranges triggering `QUOTA_EXHAUSTED` in large numbers;
- A model receiving many requests while most of them return `MODEL_NOT_ALLOWED`.

---

## Redis Key Cleanup and Sensitive Data Protection

### Sensitive Keys in Redis

At runtime, the Control Plane writes two types of critical state keys to Redis:

- **Quota Key**: records the real-time remaining quota of an API-Key / Entity, in the format `QUOTA_<api-key-token>` or `QUOTA_<entity-id>`;
- **Rate-Limit Key**: records the tokens / requests consumed in the current window by a rate limit rule, in formats such as `RL_TPM_rlp-<policyID>_<name>` and `RL_RPM_rlp-<policyID>_<name>`.

When BFE actually accesses Redis, it also prepends a prefix `default_bfe_<policyId>_...` to form the full key.

### Cleanup Trigger Scenarios

To avoid leaving useless keys in Redis after an API-Key or Entity is deleted, the Control Plane actively cleans up in the following scenarios:

1. API-Key deletion: cleans up that API-Key's Quota Key and the Rate-Limit Keys of its directly bound RateLimitPolicy;
2. Entity deletion: cleans up that Entity's Quota Key and the Rate-Limit Keys of its directly bound RateLimitPolicy;
3. An API-Key update causing rate-limit rules to be deleted: precisely matches the old rule keys by `name` and cleans them up;
4. An Entity update causing rate-limit rules to be deleted: same as above, cleaning up only the current Entity's keys.

Routine changes to a quota plan (changing the quota, changing the unit, switching to unlimited) do not delete the Quota Key; they only overwrite its value. Cleanup operations are executed in batches via Redis Pipeline using `UNLINK` through `QuotaCache.DeleteKeys` (small keys can be deleted directly with `DEL`). See `ai-gateway-api/design-docs/sys-design/details/Redis Key 清理机制.md` for details.

### Sensitive Data Protection

- Redis should have authentication (`requirepass`) and TLS connections enabled to prevent unauthorized access;
- Avoid printing full API-Keys or Redis keys in logs;
- Network-isolate the Redis instance so that only AI Gateway API and BFE can access it;
- Audit Redis regularly for orphaned keys that have not been updated for a long time.

---

## Rate Limiting and Quota as Security Defenses

### Multi-Level Policy Inheritance

An API-Key can be attached to an Entity, thereby inheriting quota plans and rate limit policies from the Entity hierarchy upward. When exported to BFE:

- Collect all **non-unlimited** quota plans from the API-Key itself and from the Entity hierarchy upward;
- Collect all **enabled** rate limit policies from the API-Key itself and from the Entity hierarchy upward;
- Each quota plan generates an independent `QuotaPlan.RedisKey`; an API-Key may be governed by multiple Redis keys simultaneously.

This design allows security policies to take effect at multiple levels — organization, department, project, and application — avoiding omissions from single-point configuration.

### Preventing Abuse and Runaway Costs

Rate limiting and quota are the second line of defense after an API-Key is leaked or abused:

- **RPM/TPM/concurrency limits**: prevent burst traffic from overwhelming the backend or exhausting provider quotas;
- **Quota Plan**: sets a budget cap by token count or amount, preventing runaway costs;
- **Model allowlist/blocklist**: restricts the range of callable models, avoiding calls to high-cost models.

When rate limiting is triggered or quota is exhausted, BFE returns a standardized error response such as `RPM_LIMIT_EXCEEDED` or `QUOTA_EXHAUSTED`; business systems can perform backoff retries based on fields such as `retry_after_seconds`.

---

## Model Allowlist and Blocklist

### Model Allowlist and Blocklist

Rainway AI Gateway provides model-level allowlists and blocklists through the Entity inheritance mechanism:

- `allow_models`: takes the intersection across levels; only models allowed by every level are accessible;
- `block_models`: takes the union across levels; a model blocked by any parent level is intercepted.

If the intersection of the API-Key's own `allow_models` and the Entity-inherited result is empty, the API-Key is disabled (`Enabled=false`) when exported, and requests are rejected directly at the Data Plane.

### Request Admission Flow

```
Client Request
    │
    ▼
┌─────────────────┐
│ IP Subnet       │ ──► If not matched, return SUBNET_NOT_ALLOWED
│ Allowlist       │
└────────┬────────┘
         │ Matched
         ▼
┌─────────────────┐
│ API-Key Auth    │ ──► On failure, return NO_API_KEY / INVALID_API_KEY, etc.
└────────┬────────┘
         │ Passed
         ▼
┌─────────────────┐
│ Model Allowlist/│ ──► On failure, return MODEL_NOT_ALLOWED
│ Blocklist       │
└────────┬────────┘
         │ Passed
         ▼
┌─────────────────┐
│ Quota / Rate    │ ──► On failure, return QUOTA_EXHAUSTED / RPM_LIMIT_EXCEEDED, etc.
│ Limit Check     │
└────────┬────────┘
         │ Passed
         ▼
   Backend Model Service
```

The diagram above shows the layered interception order from the network layer to the application layer. IP subnet allowlists, API-Key authentication, model allowlists/blocklists, and quota/rate-limit checks execute in sequence, forming a multi-layer admission control.

---

## Security Configuration Examples

### Control Plane Runtime Security Configuration

Configuration items in `ai_gateway_api.toml` directly related to security:

```toml
[RunTime]
# Must be disabled in production; never set to true
SkipTokenValidate = false

# SQL logging only for debugging
RecordSQL = false

# Session Key expiration time; recommended not to exceed 7 days
SessionExpireInDay = 7

# Disable response Debug information
Debug = false
```

### Data Plane TLS Configuration Example

BFE's TLS configuration is usually located in `conf/tls_conf/tls_rule_conf.data`; an example fragment:

```json
{
  "Config": {
    "example_product": {
      "DefaultNextProtos": "http/1.1;h2",
      "CertName": "example_product",
      "SniConf": null
    }
  },
  "DefaultSubProtocols": "http/1.1;h2",
  "DefaultMaxVersion": "VersionTLS13",
  "DefaultMinVersion": "VersionTLS12"
}
```

File permissions on the certificate file `conf/tls_conf/example_product.crt` and the private key `conf/tls_conf/example_product.key` should be strictly restricted.

### API-Key Security Policy Example

When creating an API-Key, it is recommended to configure the following together:

```json
{
  "name": "app-llm-proxy",
  "entity_id": "ent-project-x",
  "expired_time": "2026-01-01T00:00:00Z",
  "allowed_subnets": ["10.0.0.0/24", "192.168.1.0/24"],
  "allowed_models": ["gpt-4", "gpt-3.5-turbo"]
}
```

Combined with the Entity hierarchy, `block_models: ["gpt-4-32k"]` can be set at a higher level to achieve top-down model governance.

---

## Chapter Summary

- Rainway AI Gateway adopts the defense-in-depth principle; security mechanisms cover the transport layer, the authentication/authorization layer, the request admission layer, and the policy enforcement layer.
- The API-Key is the direct credential of the request path; it should be transported over HTTPS and combined with Entity inheritance for organization-level policy control.
- The Control Plane uses the Visitor abstraction and the Feature-Action permission model, and supports four authentication methods — Password, Session, Token, and Skip; `SkipTokenValidate` must be disabled in production.
- The Data Plane BFE outputs structured error responses and access log fields, supporting security auditing, anomaly detection, and billing reconciliation.
- Quota Keys and Rate-Limit Keys in Redis are actively cleaned up when an API-Key / Entity is deleted or a policy changes, avoiding the security risks of residual data.
- Rate limiting and quota are key defenses against API-Key leakage, abuse, and runaway costs, and support multi-level inheritance from API-Keys and Entities.
- IP subnet control, API-Key authentication, model allowlists/blocklists, and quota/rate limiting together form the security barrier for request admission, intercepting layer by layer in the order "IP allowlist → API-Key authentication → model allowlist/blocklist → quota/rate limit".

---

## References

- `ai-gateway-api/design-docs/sys-design/details/认证授权机制.md`
- `ai-gateway-api/design-docs/sys-design/details/API-Key与Entity关联及模型继承.md`
- `ai-gateway-api/design-docs/sys-design/details/Redis Key 清理机制.md`
- `bfe/docs/zh_cn/sys_design/ai_error_codes.md`
- `ai-gateway-api/docs/zh_cn/config_param.md`
