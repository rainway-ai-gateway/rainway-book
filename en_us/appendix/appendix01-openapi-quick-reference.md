# Appendix 1: OpenAPI Quick Reference

The OpenAPI v1 interface definitions of the Rainway AI Gateway Control Plane (AI Gateway API) have been split into independent documents by module and are continuously maintained on GitHub. This book does not repeat the full interface list; readers can view the authoritative definitions for the corresponding version via the links below.

## v0.0.8 API Documentation

- [OpenAPI Interface Definitions Directory](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义)
- [Common Conventions](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/00-common.md): URL format, authentication method, response format, Method conventions, common Query parameters
- [Key Business Workflows](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/workflows.md)
- [Object Relations Diagram](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/object-relations.md)

## API Entry Points by Module

| Module | Documentation Link |
|------|----------|
| `/api-keys` | [api-keys.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/api-keys.md) |
| `/entity-types` | [entity-types.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/entity-types.md) |
| `/entities` | [entities.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/entities.md) |
| `/global-route-rules` | [global-route-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/global-route-rules.md) |
| `/route-tables` | [route-tables.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/route-tables.md) |
| `/providers` | [providers.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/providers.md) |
| `/clusters` | [clusters.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/clusters.md) |
| `/model-prices` | [model-prices.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/model-prices.md) |
| `/certificates` | [certificates.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/certificates.md) |
| `/auth` | [auth.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/auth.md) |
| `/alb-pool` | [alb-pool.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/alb-pool.md) |
| `/expression/verify` | [expression-verify.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/expression-verify.md) |

## Notes

- The links above point to a static snapshot of the `v0.0.8` tag, suitable for cross-referencing with the content of this book.
- To view the latest API changes, visit the [ai-gateway-api main branch design docs](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/develop/design-docs/api-define/OpenAPI接口定义).
