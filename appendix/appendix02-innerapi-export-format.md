# 附2 InnerAPI 配置导出格式

壬远 AI 网关控制面通过 InnerAPI v1 向数据面（BFE）和 Conf Agent 导出配置。各配置主题的导出格式、数据模型与接口说明已在 GitHub 上完整维护。本书不再重复展开，读者可通过以下链接查看 v0.1.0 版本的权威定义。

## v0.1.0 版本 InnerAPI 文档

- [InnerAPI 接口定义目录](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义)
- [接口总览](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/00-overview.md)
- [通用约定](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/01-common.md)
- [接口清单](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/02-interface-list.md)
- [数据模型](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/data-models.md)
- [附录](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/appendix.md)

## 各配置主题导出格式

| 配置主题 | 文档链接 |
|----------|----------|
| Server Data Conf | [server-data-conf.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/server-data-conf.md) |
| GSLB / Cluster Table | [gslb.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/gslb.md)、[cluster-table.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.1.0/design-docs/api-define/InnerAPI接口定义/cluster-table.md) |
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

## 说明

- InnerAPI 返回的配置通常由 Conf Agent 拉取后写入本地版本目录，再触发热加载。
- 各主题导出的 JSON 结构与 BFE 数据面配置文件的字段命名保持一致，详细字段说明见上述链接。
- 缓存与上下文压缩导出含顶层全局块（`Semantic` / `Defaults`），恒导出并参与 MD5 签名；意图导出走 `Data` 文件原样形态。

## 导出格式要点

以下为各主题导出格式的关键约定；字段级细节以上游 InnerAPI 模块文档为准。

### epp_data 导出

`assignment` 段是 cluster→group 映射：条目形如 `{"cluster-a": {"group": "g1"}}`，只引用实例组名，不列出实例；所有 EPP 实例拿到完全相同的同一 version 快照。`server_data_conf` 中的 `EPPAddr` 为 `[]string` 类型，恒为单元素数组（被分配实例组内唯一实例的 `host:port`）。

### server_data_conf 的 NormalizeUpstreamError

`server_data_conf` 的 `AIConf` 含 `NormalizeUpstreamError` 对象，对应 Cluster 的 `llm_config.normalize_upstream_error` 可选配置：

| 导出字段 | Cluster 配置字段 | 说明 |
|----------|------------------|------|
| `Enabled` | `enabled` | 是否启用上游错误规范化 |
| `StreamEnabled` | `stream_enabled` | 流式响应是否启用 |
| `UnrecognizedAction` | `unrecognized_action` | 未识别错误的处理动作：`passthrough` 或 `rewrite_generic` |
| `MaxBodyBytes` | `max_body_bytes` | 错误体读取上限，默认 65536 |
| `RedactSecrets` | `redact_secrets` | 是否对密钥脱敏，默认 true |

控制面只导出显式配置的值：未配置时整个字段不下发，BFE 在加载期通过 `Effective()` 合成默认值。

### 导出字段级加密（enc$v1$）

`[Security].EncryptExports` 开启时，导出文件中的敏感字段为 `enc$v1$<base64(...)>` 信封：`mod-api-key` 主题 `tokens` 的外层键，以及 `server_data_conf` 主题 `AIConf.Keys[].Key`；无该前缀的值按明文直通。加密信封格式、控制面开关、BFE keyring 与失败语义的完整说明见 [第十四章 配置导出与版本控制设计](../design/chapter14-config-export-and-version-control.md)。

### 报表明细表唯一键

访问报表明细表 `bfe_ai_request_log` 的去重唯一键 `uk_dedup` 为五列 `(hostid, log_time, ai_apikey_id, ai_requested_model, logid)`，`logid` 列为 `BIGINT NOT NULL DEFAULT 0`；DDL 头部附有存量迁移 ALTER 语句（先回填 `logid`，再重建唯一键）。
