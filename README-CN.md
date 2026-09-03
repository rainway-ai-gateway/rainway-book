# 壬远AI网关：原理、设计与实现

[English](./README.md) | 中文

本书面向**壬远AI网关的使用者和开发者**，系统介绍壬远AI网关的技术背景、核心原理、系统设计、操作方法、代码实现与扩展开发。

## 版本说明

本书会随着壬远AI网关的升级而持续更新。本书的当前版本对应 **壬远AI网关 v0.5.0**， release 地址：<https://github.com/rainway-ai-gateway/ai-gateway/releases/tag/v0.5.0>。

v0.5.0 包含以下组件版本：

| 组件 | 版本 | 说明 |
|---|---|---|
| AI Gateway API | v0.0.8 | 控制面 |
| BFE | v1.8.6 | 数据面 |
| Dashboard | v0.0.8 | 控制台（内嵌于 API 镜像） |
| conf-agent | v0.0.6 | 配置热加载代理 |
| Log Reader | v1.2.0 | 访问日志 → Kafka |
| Observability | v0.0.1 | Doris + Grafana 可观测配置 |

## 全书目录

### 背景篇
- [第一章 壬远AI网关简介](./background/chapter01-what-is-rainway-ai-gateway.md)

### 原理篇
- [第二章 AI网关技术概述](./principle/chapter02-ai-gateway-overview.md)
- [第三章 大模型服务接入的挑战](./principle/chapter03-llm-access-challenges.md)
- [第四章 AI网关的路由与调度原理](./principle/chapter04-routing-and-scheduling.md)

### 设计篇
- [第五章 壬远AI网关架构与核心概念](./design/chapter05-system-architecture.md)
- [第六章 控制面核心设计：AI Gateway API](./design/chapter06-control-plane-design.md)
- [第七章 数据面转发设计：BFE](./design/chapter07-data-plane-design.md)
- [第八章 认证授权设计](./design/chapter08-auth-design.md)
- [第九章 Entity 与 API-Key 设计](./design/chapter09-apikey-design.md)
- [第十章 Provider与Cluster设计](./design/chapter10-provider-and-cluster.md)
- [第十一章 AI路由规则设计](./design/chapter11-ai-route-rules.md)
- [第十二章 配额与限流设计](./design/chapter12-quota-and-rate-limit.md)
- [第十三章 模型定价与成本核算设计](./design/chapter13-model-pricing.md)
- [第十四章 配置导出与版本控制设计](./design/chapter14-config-export-and-version-control.md)
- [第十五章 可观测性设计](./design/chapter15-observability.md)
- [第十六章 安全设计](./design/chapter16-security-design.md)

### 操作篇
- [第十七章 安装部署](./operation/chapter17-installation-and-deployment.md)
- [第十八章 控制台基础操作](./operation/chapter18-dashboard-basics.md)
- [第十九章 Provider与模型配置](./operation/chapter19-provider-and-model-config.md)
- [第二十章 Cluster与路由配置](./operation/chapter20-cluster-and-route-config.md)
- [第二十一章 API-Key与配额配置](./operation/chapter21-apikey-and-quota-config.md)
- [第二十二章 限流策略配置](./operation/chapter22-rate-limit-config.md)
- [第二十三章 域名与证书配置](./operation/chapter23-domain-and-cert-config.md)
- [第二十四章 配置热加载与升级](./operation/chapter24-hot-reload-and-upgrade.md)

### 实现篇
- [第二十五章 代码组织与启动流程](./implementation/chapter25-code-layout-and-startup.md)
- [第二十六章 接口层实现：OpenAPI与InnerAPI](./implementation/chapter26-endpoints-implementation.md)
- [第二十七章 模型层实现：Manager与Storager模式](./implementation/chapter27-model-layer-implementation.md)
- [第二十八章 存储层实现：DAO与Storage](./implementation/chapter28-storage-layer-implementation.md)
- [第二十九章 AI路由模块实现：mod_ai_route](./implementation/chapter29-mod-ai-route.md)
- [第三十章 Token认证与配额模块实现：mod_ai_token_auth](./implementation/chapter30-mod-ai-token-auth.md)
- [第三十一章 限流模块实现：mod_ai_rate_limit](./implementation/chapter31-mod-ai-rate-limit.md)
- [第三十二章 请求体处理模块实现：mod_body_process](./implementation/chapter32-mod-body-process.md)
- [第三十三章 Conf Agent实现](./implementation/chapter33-conf-agent-implementation.md)

### 开发篇
- [第三十四章 如何扩展壬远AI网关](./develop/chapter34-how-to-extend.md)
- [第三十五章 如何向壬远AI网关贡献代码](./develop/chapter35-how-to-contribute.md)

### 附录篇
- [附1 OpenAPI接口速查](./appendix/appendix01-openapi-quick-reference.md)
- [附2 InnerAPI配置导出格式](./appendix/appendix02-innerapi-export-format.md)
- [附3 常见错误码](./appendix/appendix03-common-error-codes.md)
- [附4 术语表](./appendix/appendix04-glossary.md)

## 写作规范

详见 [writing-guide.md](./writing-guide.md)。

## 参考项目

- `ai-gateway-api/`：控制面核心组件
- `bfe/`：数据面转发引擎
- `conf-agent/`：配置代理
- `ai-gateway-web/`：管理控制台（Dashboard）前端
  - 控制台详细使用说明见：<https://github.com/rainway-ai-gateway/ai-gateway-web/tree/refs/tags/v0.0.8/docs/zh-cn>

## 版权许可

本书采用 [Creative Commons Attribution 4.0 International License (CC-BY-4.0)](https://creativecommons.org/licenses/by/4.0/) 许可。

在遵守署名要求的前提下，您可以自由地分享、改编本书内容，包括用于商业用途。完整许可条款见 [LICENSE](./LICENSE)。
