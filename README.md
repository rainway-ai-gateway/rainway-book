# Rainway AI Gateway: Principles, Design and Implementation

English | [中文](./README-CN.md)

This book is written for **users and developers of the Rainway AI Gateway**. It systematically covers the technical background, core principles, system design, operations, code implementation, and extension development of the Rainway AI Gateway.

## Version Notes

This book is continuously updated along with the Rainway AI Gateway releases. The current version of this book corresponds to **Rainway AI Gateway v0.5.0**. Release address: <https://github.com/rainway-ai-gateway/ai-gateway/releases/tag/v0.5.0>.

v0.5.0 includes the following component versions:

| Component | Version | Description |
|---|---|---|
| AI Gateway API | v0.0.8 | Control Plane |
| BFE | v1.8.6 | Data Plane |
| Dashboard | v0.0.8 | Management console (embedded in the API image) |
| conf-agent | v0.0.6 | Configuration hot reload agent |
| Log Reader | v1.2.0 | Access log → Kafka |
| Observability | v0.0.1 | Doris + Grafana observability configuration |

## Table of Contents

### Background
- [Chapter 1: Introduction to Rainway AI Gateway](./en_us/background/chapter01-what-is-rainway-ai-gateway.md)

### Principles
- [Chapter 2: AI Gateway Technology Overview](./en_us/principle/chapter02-ai-gateway-overview.md)
- [Chapter 3: Challenges of Integrating Large Language Model Services](./en_us/principle/chapter03-llm-access-challenges.md)
- [Chapter 4: Routing and Scheduling Principles of the AI Gateway](./en_us/principle/chapter04-routing-and-scheduling.md)

### Design
- [Chapter 5: Rainway AI Gateway Architecture and Core Concepts](./en_us/design/chapter05-system-architecture.md)
- [Chapter 6: Control Plane Core Design: AI Gateway API](./en_us/design/chapter06-control-plane-design.md)
- [Chapter 7: Data Plane Forwarding Design: BFE](./en_us/design/chapter07-data-plane-design.md)
- [Chapter 8: Authentication and Authorization Design](./en_us/design/chapter08-auth-design.md)
- [Chapter 9: Entity and API-Key Design](./en_us/design/chapter09-apikey-design.md)
- [Chapter 10: Provider and Cluster Design](./en_us/design/chapter10-provider-and-cluster.md)
- [Chapter 11: AI Route Rule Design](./en_us/design/chapter11-ai-route-rules.md)
- [Chapter 12: Quota and Rate Limit Design](./en_us/design/chapter12-quota-and-rate-limit.md)
- [Chapter 13: Model Pricing and Cost Accounting Design](./en_us/design/chapter13-model-pricing.md)
- [Chapter 14: Config Export and Version Control Design](./en_us/design/chapter14-config-export-and-version-control.md)
- [Chapter 15: Observability Design](./en_us/design/chapter15-observability.md)
- [Chapter 16: Security Design](./en_us/design/chapter16-security-design.md)

### Operations
- [Chapter 17: Installation and Deployment](./en_us/operation/chapter17-installation-and-deployment.md)
- [Chapter 18: Dashboard Basics](./en_us/operation/chapter18-dashboard-basics.md)
- [Chapter 19: Provider and Model Configuration](./en_us/operation/chapter19-provider-and-model-config.md)
- [Chapter 20: Cluster and Route Configuration](./en_us/operation/chapter20-cluster-and-route-config.md)
- [Chapter 21: API-Key and Quota Configuration](./en_us/operation/chapter21-apikey-and-quota-config.md)
- [Chapter 22: Rate Limit Policy Configuration](./en_us/operation/chapter22-rate-limit-config.md)
- [Chapter 23: Domain and Certificate Configuration](./en_us/operation/chapter23-domain-and-cert-config.md)
- [Chapter 24: Configuration Hot Reload and Upgrade](./en_us/operation/chapter24-hot-reload-and-upgrade.md)

### Implementation
- [Chapter 25: Code Layout and Startup Flow](./en_us/implementation/chapter25-code-layout-and-startup.md)
- [Chapter 26: Interface Layer Implementation: OpenAPI and InnerAPI](./en_us/implementation/chapter26-endpoints-implementation.md)
- [Chapter 27: Model Layer Implementation: The Manager and Storager Pattern](./en_us/implementation/chapter27-model-layer-implementation.md)
- [Chapter 28: Storage Layer Implementation: DAO and Storage](./en_us/implementation/chapter28-storage-layer-implementation.md)
- [Chapter 29: Implementing the AI Route Module: mod_ai_route](./en_us/implementation/chapter29-mod-ai-route.md)
- [Chapter 30: Token Authentication and Quota Module Implementation: mod_ai_token_auth](./en_us/implementation/chapter30-mod-ai-token-auth.md)
- [Chapter 31: Rate Limit Module Implementation: mod_ai_rate_limit](./en_us/implementation/chapter31-mod-ai-rate-limit.md)
- [Chapter 32: Request Body Processing Module Implementation: mod_body_process](./en_us/implementation/chapter32-mod-body-process.md)
- [Chapter 33: Conf Agent Implementation](./en_us/implementation/chapter33-conf-agent-implementation.md)

### Development
- [Chapter 34: How to Extend the Rainway AI Gateway](./en_us/develop/chapter34-how-to-extend.md)
- [Chapter 35: How to Contribute Code to Rainway AI Gateway](./en_us/develop/chapter35-how-to-contribute.md)

### Appendix
- [Appendix 1: OpenAPI Quick Reference](./en_us/appendix/appendix01-openapi-quick-reference.md)
- [Appendix 2: InnerAPI Configuration Export Format](./en_us/appendix/appendix02-innerapi-export-format.md)
- [Appendix 3: Common Error Codes](./en_us/appendix/appendix03-common-error-codes.md)
- [Appendix 4: Glossary](./en_us/appendix/appendix04-glossary.md)

## Writing Guide

See [writing-guide.md](./writing-guide.md) (currently available in Chinese only).

## Referenced Projects

- `ai-gateway-api/`: Control Plane core component
- `bfe/`: Data Plane forwarding engine
- `conf-agent/`: Configuration agent
- `ai-gateway-web/`: Management console (Dashboard) frontend
  - Detailed console usage documentation: <https://github.com/rainway-ai-gateway/ai-gateway-web/tree/refs/tags/v0.0.8/docs/zh-cn>

## License

This book is licensed under the [Creative Commons Attribution 4.0 International License (CC-BY-4.0)](https://creativecommons.org/licenses/by/4.0/).

You are free to share and adapt this book, including for commercial purposes, provided that you give appropriate attribution. See [LICENSE](./LICENSE) for the full license terms.
