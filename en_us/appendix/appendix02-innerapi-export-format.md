# Appendix 2: InnerAPI Configuration Export Format

The Rainway AI Gateway Control Plane exports configuration to the Data Plane (BFE) and the Conf Agent via InnerAPI v1. The export format, data models, and interface descriptions for each configuration topic are fully maintained on GitHub. This book does not repeat them here; readers can refer to the authoritative definitions for the v0.0.8 release via the links below.

## InnerAPI Documentation for v0.0.8

- [InnerAPI Interface Definition Directory](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义)
- [Overview](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/00-overview.md)
- [Common Conventions](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/01-common.md)
- [Interface List](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/02-interface-list.md)
- [Data Models](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/data-models.md)
- [Appendix](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/appendix.md)

## Export Formats by Configuration Topic

| Configuration Topic | Documentation Link |
|---------------------|--------------------|
| Server Data Conf | [server-data-conf.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/server-data-conf.md) |
| GSLB / Cluster Table | [gslb.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/gslb.md), [cluster-table.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/cluster-table.md) |
| Server Cert Conf | [server-cert-conf.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/server-cert-conf.md) |
| Extra Files | [extra-files.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/extra-files.md) |
| mod-ai-key | [mod-api-key.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/mod-api-key.md) |
| mod-body-process | [mod-body-process.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/mod-body-process.md) |
| rate-limit-policy | [rate-limit-policy.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/rate-limit-policy.md) |
| ai-route | [ai-route.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/ai-route.md) |

## Notes

- The configuration returned by InnerAPI is typically pulled by the Conf Agent, written to a local version directory, and then triggers a hot reload.
- The JSON structure exported for each topic keeps its field naming consistent with the BFE Data Plane configuration files; see the links above for detailed field descriptions.
