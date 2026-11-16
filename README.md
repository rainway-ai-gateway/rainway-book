# Rainway AI Gateway: Principles, Design and Implementation

English | [中文](./README-CN.md)

This book is written for **users and developers of the Rainway AI Gateway**. It systematically covers the technical background, core principles, system design, operations, code implementation, and extension development of the Rainway AI Gateway.

## Version Notes

This book is continuously updated along with the Rainway AI Gateway releases. The current version of this book corresponds to **Rainway AI Gateway v0.9.0**. Release address: <https://github.com/rainway-ai-gateway/ai-gateway/releases/tag/v0.9.0.0>.

The content of this book corresponds to the following component versions:

| Component | Version | Description |
|---|---|---|
| AI Gateway API | v0.1.0 | Control Plane |
| AI Gateway EPP | v0.3.0 | EPP scheduling |
| BFE | v1.9.0 | Data Plane |
| Dashboard | v0.1.0 | Management console (embedded in the API image) |
| conf-agent | v0.0.8 | Configuration hot reload agent |
| Log Reader | v1.6.0 | Access log → Kafka / MySQL (mod_kafka / mod_log_mysql) |
| ai-gateway-controller | v1.1.0 | Kubernetes controller: Service/Endpoints discovery → InnerAPI `k8s_pools` |
| ai-gateway-observability | v0.1.0 | Standard reporting form: Doris / ClickHouse / StarRocks docking assets and deployment scripts |
| bfe-access-pb | v0.3.18 | Access log protocol (with `ai_price_warn` and `ai_client_closed_after_complete` fields) |

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
- [Chapter 16: EPP Scheduling Design](./en_us/design/chapter16-epp-scheduling-design.md)
- [Chapter 17: Security Design](./en_us/design/chapter17-security-design.md)

### Operations
- [Chapter 18: Installation and Deployment](./en_us/operation/chapter18-installation-and-deployment.md)
- [Chapter 19: Dashboard Basics](./en_us/operation/chapter19-dashboard-basics.md)
- [Chapter 20: Provider and Model Configuration](./en_us/operation/chapter20-provider-and-model-config.md)
- [Chapter 21: Cluster and Route Configuration](./en_us/operation/chapter21-cluster-and-route-config.md)
- [Chapter 22: API-Key and Quota Configuration](./en_us/operation/chapter22-apikey-and-quota-config.md)
- [Chapter 23: Rate Limit Policy Configuration](./en_us/operation/chapter23-rate-limit-config.md)
- [Chapter 24: Domain and Certificate Configuration](./en_us/operation/chapter24-domain-and-cert-config.md)
- [Chapter 25: Configuration Hot Reload and Upgrade](./en_us/operation/chapter25-hot-reload-and-upgrade.md)
- [Chapter 26: EPP Scheduling Configuration and Operations](./en_us/operation/chapter26-epp-scheduling-operation.md)

### Implementation
- [Chapter 27: Code Layout and Startup Flow](./en_us/implementation/chapter27-code-layout-and-startup.md)
- [Chapter 28: Interface Layer Implementation: OpenAPI and InnerAPI](./en_us/implementation/chapter28-endpoints-implementation.md)
- [Chapter 29: Model Layer Implementation: The Manager and Storager Pattern](./en_us/implementation/chapter29-model-layer-implementation.md)
- [Chapter 30: Storage Layer Implementation: DAO and Storage](./en_us/implementation/chapter30-storage-layer-implementation.md)
- [Chapter 31: Implementing the AI Route Module: mod_ai_route](./en_us/implementation/chapter31-mod-ai-route.md)
- [Chapter 32: Token Authentication and Quota Module Implementation: mod_ai_token_auth](./en_us/implementation/chapter32-mod-ai-token-auth.md)
- [Chapter 33: Rate Limit Module Implementation: mod_ai_rate_limit](./en_us/implementation/chapter33-mod-ai-rate-limit.md)
- [Chapter 34: Request Body Processing Module Implementation: mod_body_process](./en_us/implementation/chapter34-mod-body-process.md)
- [Chapter 35: Conf Agent Implementation](./en_us/implementation/chapter35-conf-agent-implementation.md)
- [Chapter 36: EPP Component Implementation](./en_us/implementation/chapter36-epp-implementation.md)
- [Chapter 37: ai-gateway-controller Implementation](./en_us/implementation/chapter37-controller-implementation.md)

### Development
- [Chapter 38: How to Extend the Rainway AI Gateway](./en_us/develop/chapter38-how-to-extend.md)
- [Chapter 39: How to Contribute Code to Rainway AI Gateway](./en_us/develop/chapter39-how-to-contribute.md)

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
- `ai-gateway-epp/`: EPP scheduling component (based on llm-d; provides backend selection for EPP-mode Clusters via ext_proc)
- `ai-gateway-controller/`: Kubernetes controller (Service/Endpoints discovery → InnerAPI `k8s_pools`)
- `conf-agent/`: Configuration agent
- `ai-gateway-web/`: Management console (Dashboard) frontend
  - Detailed console usage documentation: <https://github.com/rainway-ai-gateway/ai-gateway-web/tree/refs/tags/v0.1.0/docs/zh-cn>

## License

This book is licensed under the [Creative Commons Attribution 4.0 International License (CC-BY-4.0)](https://creativecommons.org/licenses/by/4.0/).

You are free to share and adapt this book, including for commercial purposes, provided that you give appropriate attribution. See [LICENSE](./LICENSE) for the full license terms.
