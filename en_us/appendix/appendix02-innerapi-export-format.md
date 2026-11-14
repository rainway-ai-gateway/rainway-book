# Appendix 2: InnerAPI Configuration Export Format

The Rainway AI Gateway Control Plane exports configuration to the Data Plane (BFE) and the Conf Agent via InnerAPI v1. The export format, data models, and interface descriptions for each configuration topic are fully maintained on GitHub. This book does not repeat them here; readers can refer to the authoritative definitions for the v0.0.11 release via the links below.

## InnerAPI Documentation for v0.0.11

- [InnerAPI Interface Definition Directory](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义)
- [Overview](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/00-overview.md)
- [Common Conventions](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/01-common.md)
- [Interface List](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/02-interface-list.md)
- [Data Models](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/data-models.md)
- [Appendix](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/appendix.md)

## Export Formats by Configuration Topic

| Configuration Topic | Documentation Link |
|---------------------|--------------------|
| Server Data Conf | [server-data-conf.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/server-data-conf.md) |
| GSLB / Cluster Table | [gslb.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/gslb.md), [cluster-table.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/cluster-table.md) |
| Server Cert Conf | [server-cert-conf.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/server-cert-conf.md) |
| Extra Files | [extra-files.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/extra-files.md) |
| mod-ai-key | [mod-api-key.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/mod-api-key.md) |
| mod-body-process | [mod-body-process.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/mod-body-process.md) |
| rate-limit-policy | [rate-limit-policy.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/rate-limit-policy.md) |
| ai-route | [ai-route.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/ai-route.md) |
| ai-cache-rule | [ai-cache-rule.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/ai-cache-rule.md) |
| traffic-mirror-rule | [traffic-mirror-rule.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/traffic-mirror-rule.md) |
| mod-ai-intent | [mod-ai-intent.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/mod-ai-intent.md) |
| ai-context-rule | [ai-context-rule.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/ai-context-rule.md) |

## Notes

- The configuration returned by InnerAPI is typically pulled by the Conf Agent, written to a local version directory, and then triggers a hot reload.
- The JSON structure exported for each topic keeps its field naming consistent with the BFE Data Plane configuration files; see the links above for detailed field descriptions.
- Cache and context compression exports contain top-level global blocks (`Semantic` / `Defaults`), which are always exported and participate in the MD5 signature; intent exports use the `Data` file as-is.

## Export Format Essentials

The following are the key conventions of each topic's export format; the upstream InnerAPI module documents are authoritative for field-level details.

### epp_data Export

The `assignment` section is a cluster→group mapping: entries look like `{"cluster-a": {"group": "g1"}}` and reference only instance group names, without listing instances; every EPP instance receives the identical same-version snapshot. `EPPAddr` in `server_data_conf` is an `[]string` and is always a single-element array (the `host:port` of the sole instance in the assigned group).

### NormalizeUpstreamError in server_data_conf

The `AIConf` of `server_data_conf` carries a `NormalizeUpstreamError` object, corresponding to the Cluster's optional `llm_config.normalize_upstream_error` configuration:

| Export field | Cluster config field | Description |
|--------------|----------------------|-------------|
| `Enabled` | `enabled` | Whether upstream error normalization is enabled |
| `StreamEnabled` | `stream_enabled` | Whether it applies to streaming responses |
| `UnrecognizedAction` | `unrecognized_action` | Handling of unrecognized errors: `passthrough` or `rewrite_generic` |
| `MaxBodyBytes` | `max_body_bytes` | Error body read limit, default 65536 |
| `RedactSecrets` | `redact_secrets` | Whether to redact secrets, default true |

The Control Plane exports only explicitly configured values: when unconfigured the whole field is absent, and BFE synthesizes defaults at load time via `Effective()`.

### Field-Level Encryption of Exports (enc$v1$)

With `[Security].EncryptExports` enabled, sensitive fields in exported files appear as `enc$v1$<base64(...)>` envelopes: the outer keys of `tokens` in the `mod-api-key` topic, and `AIConf.Keys[].Key` in the `server_data_conf` topic; values without the prefix are plaintext passthrough. The envelope format, Control Plane switches, BFE keyring, and failure semantics are fully described in [Chapter 14: Config Export and Version Control Design](../design/chapter14-config-export-and-version-control.md).

### Report Detail Table Unique Key

The dedup unique key `uk_dedup` of the access report detail table `bfe_ai_request_log` spans five columns `(hostid, log_time, ai_apikey_id, ai_requested_model, logid)`, and the `logid` column is `BIGINT NOT NULL DEFAULT 0`; the DDL header ships with migration ALTER statements (backfill `logid`, then rebuild the unique key).
