# 壬远AI网关：原理、设计与实现

[English](./README.md) | 中文

本书面向**壬远AI网关的使用者和开发者**，系统介绍壬远AI网关的技术背景、核心原理、系统设计、操作方法、代码实现与扩展开发。

## 版本说明

本书会随着壬远AI网关的升级而持续更新。本书的当前版本对应 **壬远AI网关 v0.9.0**，release 地址：<https://github.com/rainway-ai-gateway/ai-gateway/releases/tag/v0.9.0.0>。

本书内容对应以下组件版本：

| 组件 | 版本 | 说明 |
|---|---|---|
| AI Gateway API | v0.1.0 | 控制面 |
| AI Gateway EPP | v0.3.0 | EPP 智能调度 |
| BFE | v1.9.0 | 数据面 |
| Dashboard | v0.1.0 | 控制台（内嵌于 API 镜像） |
| conf-agent | v0.0.8 | 配置热加载代理 |
| Log Reader | v1.6.0 | 访问日志 → Kafka / MySQL（mod_kafka / mod_log_mysql） |
| ai-gateway-controller | v1.1.0 | Kubernetes 控制器：Service/Endpoints 发现 → InnerAPI `k8s_pools` |
| ai-gateway-observability | v0.1.0 | 报表标准形态：Doris / ClickHouse / StarRocks 对接资产与部署脚本 |
| bfe-access-pb | v0.3.18 | 访问日志协议（含 `ai_price_warn`、`ai_client_closed_after_complete` 字段） |

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
- [第十六章 EPP智能调度设计](./design/chapter16-epp-scheduling-design.md)
- [第十七章 安全设计](./design/chapter17-security-design.md)

### 操作篇
- [第十八章 安装部署](./operation/chapter18-installation-and-deployment.md)
- [第十九章 控制台基础操作](./operation/chapter19-dashboard-basics.md)
- [第二十章 Provider与模型配置](./operation/chapter20-provider-and-model-config.md)
- [第二十一章 Cluster与路由配置](./operation/chapter21-cluster-and-route-config.md)
- [第二十二章 API-Key与配额配置](./operation/chapter22-apikey-and-quota-config.md)
- [第二十三章 限流策略配置](./operation/chapter23-rate-limit-config.md)
- [第二十四章 域名与证书配置](./operation/chapter24-domain-and-cert-config.md)
- [第二十五章 配置热加载与升级](./operation/chapter25-hot-reload-and-upgrade.md)
- [第二十六章 EPP调度配置与运维](./operation/chapter26-epp-scheduling-operation.md)

### 实现篇
- [第二十七章 代码组织与启动流程](./implementation/chapter27-code-layout-and-startup.md)
- [第二十八章 接口层实现：OpenAPI与InnerAPI](./implementation/chapter28-endpoints-implementation.md)
- [第二十九章 模型层实现：Manager与Storager模式](./implementation/chapter29-model-layer-implementation.md)
- [第三十章 存储层实现：DAO与Storage](./implementation/chapter30-storage-layer-implementation.md)
- [第三十一章 AI路由模块实现：mod_ai_route](./implementation/chapter31-mod-ai-route.md)
- [第三十二章 Token认证与配额模块实现：mod_ai_token_auth](./implementation/chapter32-mod-ai-token-auth.md)
- [第三十三章 限流模块实现：mod_ai_rate_limit](./implementation/chapter33-mod-ai-rate-limit.md)
- [第三十四章 请求体处理模块实现：mod_body_process](./implementation/chapter34-mod-body-process.md)
- [第三十五章 Conf Agent实现](./implementation/chapter35-conf-agent-implementation.md)
- [第三十六章 EPP组件实现](./implementation/chapter36-epp-implementation.md)
- [第三十七章 ai-gateway-controller实现](./implementation/chapter37-controller-implementation.md)

### 开发篇
- [第三十八章 如何扩展壬远AI网关](./develop/chapter38-how-to-extend.md)
- [第三十九章 如何向壬远AI网关贡献代码](./develop/chapter39-how-to-contribute.md)

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
- `ai-gateway-epp/`：EPP 智能调度组件（基于 llm-d，经 ext_proc 为 EPP 模式 Cluster 提供后端选择）
- `ai-gateway-controller/`：Kubernetes 控制器（Service/Endpoints 发现 → InnerAPI `k8s_pools`）
- `conf-agent/`：配置代理
- `ai-gateway-web/`：管理控制台（Dashboard）前端
  - 控制台详细使用说明见：<https://github.com/rainway-ai-gateway/ai-gateway-web/tree/refs/tags/v0.1.0/docs/zh-cn>

## 版权许可

本书采用 [Creative Commons Attribution 4.0 International License (CC-BY-4.0)](https://creativecommons.org/licenses/by/4.0/) 许可。

在遵守署名要求的前提下，您可以自由地分享、改编本书内容，包括用于商业用途。完整许可条款见 [LICENSE](./LICENSE)。
