# 第二十章 Provider 与模型配置

## 本章目标

通过本章学习，读者将掌握 Provider 在壬远 AI 网关中的定位与作用，熟练使用 Dashboard 与 OpenAPI 创建和维护 Provider，正确配置模型端点、模型列表、Provider Keys 与后端实例池（IP 模式与服务商域名模式），使用模型发现工具自动探测模型，理解 `openai`、`anthropic`、`gemini` 三种模型协议的差异，掌握服务商列表常用操作（查询模型价格、详情、分段计价配置、删除）与忙时时间段（peak tier）的配置方法，掌握通过 model-list.yaml 批量导入模型定价的方法，并厘清 Provider 与 Cluster 的关联关系及变更影响，以及为多协议 provider 配置 `protocol_paths` 协议路径映射的方法与注意事项。

## Provider 的概念与作用

在壬远 AI 网关的控制面（Control Plane）中，**Provider** 用于描述一个下游模型提供方：它提供哪些模型、通过什么协议接入、后端实例在哪里、使用哪些 API Key。可以将其理解为“我能访问谁”这一层抽象。与之对应，**Cluster** 负责“如何把请求转发过去”，包括路由匹配、模型映射、Key 权重、重试策略、超时等。

Provider 与 Cluster 分离后，二者职责更加清晰：

- Provider = “我是谁、我能访问哪些模型、我的后端和密钥是什么”。
- Cluster = “我如何转发、用哪些模型、Key 权重如何分配”。

这种分离带来了多方面好处：同一 Provider 被多个 Cluster 引用时，实例池和密钥只需维护一份，避免重复配置；Cluster 不再存储 API Key 明文，只通过 name 引用 Provider 中的 Key，提升了安全性；Provider 可独立创建、更新、删除，Cluster 通过引用获取后端能力，生命周期更独立；新增协议时只需扩展 Provider 的 model_protocols，不会导致 Cluster 模型持续膨胀。

Provider 的核心字段包括：全局唯一的 `name`、可选的 `description`、模型发现端点 `model_endpoint`、模型列表 `models`、API Key 列表 `keys`、后端实例池 `instance_pool`、支持的协议 `model_protocols`、可选的协议路径映射 `protocol_paths`、时区 `time_zone` 与分时段模板 `tiers`。其中 `models` 必填且至少包含 1 个元素，`instance_pool` 必填且至少包含一个权重大于 0 的实例，`model_protocols` 必填且至少包含一个协议。

`time_zone` 默认值为 `Asia/Shanghai`，用于计算当前时间属于哪个 tier。`tiers` 初期只支持 `name=peak`，每个 tier 包含若干 `time_ranges`，采用左闭右开语义。通过 `PUT /v1/providers/{provider_name}/pricing-tiers` 可单独维护时区与 tier，无需在创建 Provider 时传入。

## 创建 Provider 的步骤

### 通过 Dashboard 创建

Dashboard 是面向运维人员的可视化控制台。登录后进入 **资源管理 → 模型服务商**，点击 **创建服务商**，右侧弹出抽屉表单，分为 5 个 Card 分区：基本信息、实例池、模型服务配置、协议路径映射、服务鉴权 Keys。点击操作列 **编辑** 打开同一抽屉：名称字段灰色禁用（服务商名称创建后不可修改），已配置的 Key 值留空表示保持原值不变。

首次接入的最小配置如下：

| 分区 | 必填项 |
|------|--------|
| 基本信息 | 服务商名称 |
| 实例池 | IP 或服务商域名 + 端口 + 权重（单实例填 100） |
| 模型服务配置 | 模型协议（至少一种）、模型列表接口路径、模型列表（必填，至少 1 个模型；点「获取」拉取、输入后回车添加，或「批量添加」粘贴多个名称；须提交表单才持久化） |
| 服务鉴权 Keys | 公有云等需鉴权的服务商通常必填 |

**基本信息**：

| 字段 | 必填 | 校验规则 |
|------|------|----------|
| 名称 | 是 | 1-64 字符；仅允许字母、数字、点（`.`）、下划线（`_`）、中划线（`-`）；不能以点、下划线、中划线开头或结尾；不能包含空白字符；创建时不能与已有服务商重名。唯一标识，创建后不可修改 |
| 描述 | 否 | 不超过 256 字符；不能包含控制字符 |

**实例池**：支持 **IP 模式** 与 **服务商域名模式** 二选一。

> **第三种来源：K8s 实例池**。实例来源切换为 K8s 池时，池名为**下拉选择**（数据源为只读接口 `GET /open-api/v1/k8s_pools`），编辑与详情页按 `GET /open-api/v1/k8s_pools/{name}` **实时查询**展示实例表（地址/端口/权重）与池状态——池不存在（404）、池为空分别给出提示；`disable` 为只读字段不渲染开关；池名合法长度 128 字符。Provider 不保存实例镜像（无 `k8s_instance_pool` 字段），实例列表始终来自池的实时查询结果。详见 [第十章 Provider 与 Cluster 设计](../design/chapter10-provider-and-cluster.md)「K8s 实例池供给」。

IP 模式（默认）适用于自建机房 / 私有化部署（vLLM / Xinference / Ollama 等），可「+ 创建」多行实例：

| 字段 | 必填 | 默认值 | 校验规则 |
|------|------|--------|----------|
| IP 地址 | 是 | 空 | 合法 IPv4 / IPv6；实例间 IP 不能重复 |
| 端口 | 是 | `80` | 1-65535 |
| 权重 | 是 | 首行 `100`，新增行 `0` | 0-100；所有实例权重之和须等于 100 |

服务商域名模式适用于对接公有云模型服务商：只填一个域名（如 `api.deepseek.com`），端口随协议默认（https→443，http→80），权重固定 100。不支持多地址，不能与 IP 模式混用。

**模型服务配置**：

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| 模型协议 | 是 | 空 | 多选；支持 `openai`（OpenAI 兼容）、`anthropic`（Claude Messages API）、`gemini`（Google Gemini API）；至少选一种 |
| 模型列表接口 | 否 | `https://{实例地址}/v1/models`（`gemini` 协议默认 `/v1beta/models`） | 由协议 + 实例地址（只读，来自实例池）+ URI 路径组成；路径须以 `/` 开头。对应 Provider 资源的 `model_endpoint`（`schema` 即所选协议，`addr` 取自实例池） |
| 模型列表 | 是 | 空 | 标签多选，至少 1 个模型，详见下文「模型列表的维护」 |

系统根据所选模型协议自动决定调用模型发现接口时的认证头风格（`openai` 使用 `Authorization: Bearer`，`anthropic` 使用 `x-api-key`，`gemini` 使用 `x-goog-api-key`）。**不再支持**在模型列表接口中自定义 `Authorization` Header。

**模型列表的维护**：模型列表为必填项（至少 1 个模型），标签左侧有红色必填星号。校验规则：提交时若列表为空，提示「模型列表为必填项，请至少添加 1 个模型」；模型名不能为空字符串；同一服务商内模型名不能重复。常用操作如下：

| 操作 | 说明 |
|------|------|
| 「获取」 | 调用 `POST /v1/providers/tools/discover-models` 无状态发现接口，用当前表单的协议、实例池、模型列表路径与首个 Key 拉取上游模型，并**覆盖**回填到下拉框 |
| 「批量添加」 | 打开文本框，按行或用逗号 / 中文逗号 / 分号 / 空白分隔粘贴多个模型名；确认后**合并**进现有列表（去重、跳过已有项），不覆盖 |
| 下拉框输入 | 输入模型名后按回车添加；若剪贴板文本可拆出 2 个及以上模型名，粘贴后会自动拆成多个 Tag，单个名称仍按回车添加 |
| 置灰条件 | 未选择模型协议，或实例池无有效地址时「获取」按钮禁用 |
| 提交 | 「获取」或「批量添加」成功后**不会**自动保存，须点击抽屉底部「提交」才写入 Provider 资源 |

> **常见坑**：点「获取」无反应或失败，先检查实例地址、端口、协议与 Key 是否正确，以及 BFE / API Server 能否访问该后端。

**协议路径映射**：与 Provider 资源的 `protocol_paths` 字段一一对应，用于为已勾选的多协议分别指定上游基路径；配置约束与转发行为详见下文「协议路径配置（protocol_paths）」。`gemini` 协议不支持路径改写。

**服务鉴权 Keys**：当后端模型服务需要鉴权时配置。Key 明文仅保存在 Provider 资源中，Cluster 通过 Key **名称** 引用。

| 字段 | 必填 | 校验规则 |
|------|------|----------|
| Key 名称 | 条件必填（行内任一字段填写后本行必填） | 1-128 字符；同一服务商内不能重复；供 Cluster 侧下拉选择 |
| Key 值 | 条件必填 | 1-512 字符；编辑时留空表示不修改 |

支持「+ 添加 Key」增加行、「删除」移除行；空行不参与校验，提交时自动过滤；编辑模式下已配置的 Key 会提示「已配置 Key，如需修改请输入新 Key；留空则保持不变」。

保存时系统校验字段合法性，成功后返回包含 `create_time` 与 `update_time` 的完整 Provider 记录。若校验失败，Dashboard 会提示具体字段错误，例如实例池地址重复、协议不在枚举范围内、Key 名称不合法或 `models` 元素重复等。

### 通过 OpenAPI 创建

OpenAPI 适合自动化脚本、CI/CD 或第三方系统集成。相比 Dashboard，OpenAPI 更适合批量创建、版本化管理以及与内部平台对接。创建 Provider 的端点为：

```http
POST /v1/providers
Content-Type: application/json
```

请求体示例：

```json
{
    "name": "deepseek",
    "description": "DeepSeek 官方 API",
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

若请求合法，接口返回 `ErrNum=200`，并在 `Data` 中携带完整记录。若 `model_endpoint`、`keys`、`time_zone` 未传，系统会按文档默认值填充。后续可通过 `PATCH /v1/providers/{provider_name}` 对部分字段进行更新。

## 服务商列表与常用操作

进入 **资源管理 → 模型服务商**，展示当前已创建的服务商列表。列表一次拉取全量数据，名称 / 描述 / 协议 / 模型的搜索与排序均在浏览器端完成。

| 列名 | 说明 |
|------|------|
| 名称 | 服务商唯一标识，支持按名称搜索和排序 |
| 描述 | 服务商说明 |
| 协议 | 支持的模型访问协议，如 `openai`、`anthropic`、`gemini`；多协议以逗号展示 |
| 模型 | 已配置模型列表；最多展示 2 个 Tag，超出显示 `+N`，悬停 Tooltip 查看完整列表 |
| 操作 | 详情、查询模型价格、分段计价配置、编辑、删除 |

列表上方有「创建服务商」按钮。

### 查询模型价格

点击「查询模型价格」，跳转到 **资源管理 → 模型定价** 页面并按该服务商名称（`provider` 字段）筛选列表；若无定价记录，提示「未找到提供商 {provider} 的模型定价」。服务商名称须与定价表中的 `provider` 一致，便于费用核算与该跳转正确归集，定价字段的维护方式见「模型定价导入」与 [第十三章 模型定价与成本核算设计](../design/chapter13-model-pricing.md)。

### 查看详情

点击「详情」打开只读抽屉，展示基本信息（含创建时间、更新时间）、实例池、模型服务配置（模型协议、模型列表接口、模型列表）、协议路径映射、服务鉴权 Keys（Key 值脱敏），以及分段计价配置摘要（时区、忙时时间段表；未配置时显示「未配置」）。

### 分段计价配置

点击操作列「分段计价配置」，打开独立抽屉，用于配置该服务商的**忙时（peak）**时间段，供模型定价中的分时段价格匹配使用。该配置对应 Provider 资源的 `time_zone` 与 `tiers` 字段（见「Provider 的概念与作用」），但提交时调用 `PUT /v1/providers/{name}/pricing-tiers` 单独保存，不影响服务商其他字段。

| 字段 | 必填 | 说明 |
|------|------|------|
| 服务商名称 | — | 只读 |
| 时区 | 是 | IANA 时区名，如 `Asia/Shanghai`；须通过合法性校验 |
| 计价时段 | — | 只读展示「忙时」标签；初期仅支持 `peak` |
| 时间段 | 是 | 表格配置适用星期与起止时间；支持快捷「全选 / 工作日 / 周末」；至少保留 1 行；同一 tier 内多段为「或」关系 |

时间规则：起止时间格式 `HH:MM`，结束时间须大于开始时间，跨午夜请拆成两段；适用时段为空表示每天；星期取值 0=周日 … 6=周六；同一 tier 内多个时间段按「或」匹配，判定语义为 `start <= 当前时刻 < end`（左闭右开）。配置好后，还需在模型定价页为同一 `provider` 维护 `peak` 分时段价格项，未命中忙时的请求回退使用默认价格。

### 批量任务定价（`batch_discount`）

模型定价支持批量任务折扣：价格行的 `batch_discount`（取值 `(0,1]`）把该行的各 token 价格按系数展开为 `mode=batch` 的批量价行随 `ModelTable` 下发，`mode` 白名单含 `batch`；手工维护的 `mode=batch` 价格行优先于系数展开。未配置折扣的行不生成批量价，批量请求按 chat 基价计费。设计与计费语义见 [第十三章 模型定价与成本核算设计](../design/chapter13-model-pricing.md)。

该折扣在模型定价表单中维护：输入项位于创建 / 编辑抽屉「价格」卡片内、分时段价格之后，字段名「批量折扣系数」，为 0–1、步长 0.1 的数字输入（占位提示 `0.5`），新建价格行时默认 `0.5`。界面规则：

| 项目 | 说明 |
|------|------|
| 校验 | `(0,1]` 区间内的浮点数，非法时提交被拦截并提示「批量折扣系数必须为 (0,1] 区间内的浮点数」 |
| 显示条件 | 「模型模式」（`mode`）为 `batch` 时不渲染该输入；`mode=batch` 的价格行不配置批量折扣系数 |
| 提交语义 | 仅当模型模式不是 `batch` 且填写了系数时才随提交体附带；清空则按未配置处理，导出不展开批量价行 |
| 生效范围 | 只影响下发到 BFE 的批量价格行（含各 token 价格键与 tier 价逐档展开），不改写本行已存价格 |
| 查看态 | 非 `batch` 模式展示系数值（未配置显示 `-`）；`mode=batch` 显示「-（mode=batch 的价格行不配置批量折扣系数）」 |

OpenAPI 侧的等价字段为价格行的 `batch_discount`，可经 `POST/PUT /model-prices` 维护。

### 删除服务商

点击「删除」后确认。若该服务商仍被业务 Cluster 引用，删除失败并提示「删除失败，该服务商可能仍被集群引用」，需先在相关集群中更换「所属服务商」或删除集群后再删。注意：`model-prices` 中存在同名 `provider` **不会**阻止删除服务商。

## 配置模型端点、模型列表、Provider Keys

### 模型端点

`model_endpoint` 用于调用第三方平台的模型列表接口，包含 `schema` 与 `uri` 两个字段。`schema` 默认值为 `https`，可选 `http`；`uri` 默认值为 `/v1/models`（`gemini` 协议为 `/v1beta/models`），非空且必须以 `/` 开头。该端点主要供模型发现工具使用，不直接影响 BFE 转发目标地址。系统不再允许在 `model_endpoint` 中配置 `headers.Authorization`；调用模型发现接口时，认证头风格由 `model_protocols` 自动决定：`openai` 使用 `Authorization: Bearer {apikey}`，`anthropic` 使用 `x-api-key: {apikey}`，`gemini` 使用 `x-goog-api-key: {apikey}`。

### 模型列表

`models` 字段表示该 Provider 支持的模型名称列表，例如 `["deepseek-chat", "deepseek-coder"]`。模型名在创建时可直接填写，也可在创建后通过模型发现工具自动回填。更新 Provider 时，`models` 按全量替换处理；如果删除某个 model 时仍有 Cluster 引用该 model，系统会返回 `409 Conflict`，防止误删导致路由失效。

### Provider Keys

`keys` 字段存储 Provider 可用的 API Key 明文，每个元素包含 `name` 与 `key`。`name` 在 Provider 内唯一，长度 1-128 字符，是 Cluster 引用 Key 的纽带。Cluster 的 `llm_config.keys` 只保留 `name` 与 `weight`，例如：

```json
{
    "keys": [
        { "name": "key-primary", "weight": 70 },
        { "name": "key-secondary", "weight": 30 }
    ]
}
```

更新 Provider 的 `keys` 时，系统会校验是否有 Cluster 仍引用被删除或重命名的 Key；若存在引用，同样返回 `409 Conflict`。这一机制避免了 Key 变更导致正在运行的 Cluster 无法认证。

## 模型发现工具的使用

模型发现工具用于自动探测第三方平台当前支持的模型列表，避免人工维护模型名。端点为：

```http
POST /v1/providers/tools/discover-models
```

请求体示例：

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

执行时，若 `uri` 为空则默认使用 `/v1/models`（`gemini` 协议为 `/v1beta/models`）；系统根据 `model_protocol` 生成对应认证头（`openai` 为 `Authorization: Bearer`，`anthropic` 为 `x-api-key`，`gemini` 为 `x-goog-api-key`），调用 `{schema}://{addr}:{port}{uri}`，并使用对应协议解析器提取模型名列表。返回结果为一个字符串数组，可直接复制到 Provider 的 `models` 字段中。该接口为无状态工具，不直接修改 Provider；若需回填，需再调用 `PATCH /v1/providers/{provider_name}` 将 `models` 写入。Dashboard 创建抽屉中的「获取」按钮即封装了该接口，用当前表单上下文（协议、实例池、模型列表路径与首个 Key）拉取并回填。

## 支持的模型协议

Provider 通过 `model_protocols` 字段声明支持的模型访问协议。当前枚举值包括：

| 枚举值 | 说明 | 模型发现认证头 | 模型列表接口默认路径 |
|--------|------|----------------|----------------------|
| `openai` | OpenAI 兼容协议，包括大多数国产兼容平台 | `Authorization: Bearer {apikey}` | `/v1/models` |
| `anthropic` | Anthropic Claude Messages API | `x-api-key: {apikey}` | `/v1/models` |
| `gemini` | Google Gemini API | `x-goog-api-key: {apikey}` | `/v1beta/models` |

一个 Provider 可同时支持多种协议，例如聚合平台可配置 `["openai", "anthropic"]`，但至少包含一种协议。`gemini` 协议有两点特殊之处：其一，模型发现接口的默认路径为 `/v1beta/models`；其二，`protocol_paths` 不支持对 `gemini` 做路径改写（其原生路径即标准路径，透传已可用），详见下文「协议路径配置（protocol_paths）」。

`model_protocols` 会影响 BFE 数据面的转发行为。认证头注入方面，`openai` 使用 `Authorization: Bearer`，`anthropic` 使用 `x-api-key`，`gemini` 使用 `x-goog-api-key`；Claude 请求还需要额外注入 `anthropic-version`。Usage 解析会按协议风格处理不同响应格式，例如 OpenAI 风格的 `usage` 字段与 Claude 风格的 `usage` 字段结构不同。协议匹配校验会检查请求协议风格是否在目标 Cluster 对应 Provider 的 `model_protocols` 中，若不匹配则直接拒绝。控制面生成 BFE 配置时，会将 `provider.model_protocols` 透传到 `AIConf.ModelProtocols`，供数据面使用。

## 协议路径配置（protocol_paths）

不同 provider 的上游路径前缀各不相同，且同一 provider 的不同协议常挂在不同前缀下：百炼 DashScope 的 OpenAI 兼容在 `/compatible-mode/v1`、Anthropic 兼容在 `/apps/anthropic`；火山方舟按量在 `/api/v3` 与 `/api/compatible`；Kimi Code 会员（api.kimi.com）在 `/coding/v1` 与 `/coding`。不配 `protocol_paths` 时 BFE 对上游路径纯透传，客户端必须按 provider 原生路径发起请求；配置后客户端可以统一用 OpenAI 端点路径访问所有 provider（openai 协议带不带 `/v1` 前缀均可，anthropic 协议走标准入口 `/v1/...`），由 BFE 在转发时改写路径。

`protocol_paths` 是"协议 → 上游 base path"的映射，取值为该协议官方 SDK `base_url` 的 path 部分：`openai` 含 `/v1` 尾（如 `/compatible-mode/v1`），`anthropic` 不含 `/v1`（如 `/apps/anthropic`，Anthropic SDK 会自拼 `/v1/messages`）。配置时可直接照抄 provider 官方文档的 base_url 一栏。常见 provider 的参考值：

| provider | protocol_paths |
|----------|----------------|
| 百炼 DashScope | `{"openai": "/compatible-mode/v1", "anthropic": "/apps/anthropic"}` |
| Kimi 开放平台（api.moonshot.cn） | `{"openai": "/v1", "anthropic": "/anthropic"}` |
| Kimi Code 会员（api.kimi.com） | `{"openai": "/coding/v1", "anthropic": "/coding"}` |
| DeepSeek | `{"openai": "/v1", "anthropic": "/anthropic"}` |
| 火山方舟·按量 | `{"openai": "/api/v3", "anthropic": "/api/compatible"}` |
| 火山方舟·Coding Plan | `{"openai": "/api/coding/v3", "anthropic": "/api/coding"}` |

配置约束：

- key 必须是该 provider `model_protocols` 已声明的 `openai` 或 `anthropic`，未声明的协议不允许配置路径；`gemini` 不支持路径改写，不能作为 key；
- value 必须 `/` 开头、不以 `/` 结尾、不含 `..`/`?`/`#`、长度不超过 128；
- 缺省（不配置）表示关闭，请求路径原样转发；
- PATCH 更新遵循部分更新约定：不显式携带则保持原值，显式传 `null` 清空（恢复透传）。

转发行为（`bfe/bfe_server/ai_path_rewrite.go` 中的 `rewriteUpstreamPath`）：

- openai 协议：客户端入口带不带 `/v1` 前缀均可改写。先剥离可选的 `/v1` 前缀，命中 `bfe_basic` 共享端点表（`bfe/bfe_basic/openai_endpoint.go` 中的 `openAIEndpointModes`，13 个端点）才改写：`/v1/chat/completions` 与 `/chat/completions` 均改写为 `{openai 值}/chat/completions`，`/v1` 或 `/v1/` 改写为 `{openai 值}` 本身；未命中端点表的自定义路径原样透传，兼容存量客户端。
- anthropic 协议：仅标准入口 `/v1/...` 被改写（`/v1/messages` → `{anthropic 值}/v1/messages`），非标准入口原样透传；未配置对应协议路径时不改写。
- gemini 协议：永不改写。`ProtocolPaths` 仅 `openai`/`anthropic` 两个合法 key，gemini 原生路径（如 `/v1beta/models/gemini-2.5-flash:generateContent`）始终透传。

注意事项：

- **计费域错配**：path 在部分 provider 上挂计费语义（火山 `/api/v3` 按量 vs `/api/coding` 订阅），订阅 Key 误配按量前缀会错扣费，配置前需核对 Key 类型与 path 的对应关系；
- **模型发现端点不联动**：`model_endpoint` 不随 `protocol_paths` 自动生成，例如百炼 Anthropic 协议的模型发现需手工把 `model_endpoint.uri` 配为 `/apps/anthropic/v1/models`；
- **发布顺序**：`protocol_paths` 依赖 BFE 数据面支持 `AIConf.ProtocolPaths` 改写，BFE 版本过旧时配置不生效。

配置示例（百炼）：

```json
{
    "name": "bailian",
    "model_protocols": ["openai", "anthropic"],
    "protocol_paths": {
        "openai": "/compatible-mode/v1",
        "anthropic": "/apps/anthropic"
    }
}
```

## 模型定价导入

模型定价用于成本核算与配额扣减，存储在 `/model-prices` 资源中。为便于批量维护，系统支持通过 `model-list.yaml` 文件整表导入。接口为：

```http
POST /v1/model-prices/import
Content-Type: multipart/form-data
```

表单参数包括 `file`（YAML 文件，必填）与 `mode`（导入模式，可选 `replace` 默认全量替换，或 `merge` 增量合并）。

`model-list.yaml` 示例：

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

导入时系统会校验版本、币种、`provider/model/mode` 唯一性、必填字段与价格非负性等。`prices` 至少包含一个价格字段，所有价格字段必须为非负数，支持 8 位及以上小数精度。若记录包含 `tier_prices`，初期 tier name 只支持 `peak`。

`replace` 模式先清空 `model_prices` 表再写入新数据，适合全量刷新官方价目表；`merge` 模式对已有 `(provider, model, mode)` 记录更新、新增记录插入，适合增量补丁。导入接口仅允许管理员调用。

> 说明：`/model-prices` 的 `provider` 字段仅作为价格归集标识，不再强制引用已存在的 `/providers`。因此可以先导入价格，再创建 Provider，二者独立维护。

### 图片输入与视频生成的价格字段与计费模式

BFE 数据面 cluster_conf 的 `ModelTable`（含全局 `Prices` 与 `TierPrices` 两级价格表，tier 初期仅支持 `peak`）包含图片输入与视频生成两个价格字段，在 `prices` 与 `tier_prices` 中均可配置：

| 字段 | 适用模式 | 说明 |
|------|---------|------|
| `input_cost_per_image_token` | `image_generation` | 图片输入 token 单价，与按张计费的 `output_cost_per_image` 叠加计入成本 |
| `output_cost_per_video` | `video_generation` | 视频生成单价，按实际生成的视频个数计费 |

计费模式（mode）还包括以下两种：

| mode | 计费方式 |
|------|---------|
| `responses` | 按 chat 计费，即沿用 `input_cost_per_token` / `output_cost_per_token` 等字段的 token 计费逻辑（`/v1/responses` 及 provider 原生路径形式的 `/responses` 端点识别为该模式） |
| `video_generation` | 成本 = `VideoCount × output_cost_per_video`（`/v1/video/generations` 路径识别为该模式） |

对于 `video_generation` 模式，BFE 在认证阶段预读请求体 `n` 字段作为生成个数兜底：字段缺失或非法时按 1 个计费，避免少计费；响应 `usage` 中的 `video_count` 优先作为最终生成个数。相应地，`usage` 统计包含 `image_input_tokens` 与 `video_count` 字段，分别对应访问日志字段 `ai_image_input_tokens`(786) 与 `ai_video_count`(787)（bfe-access-pb v0.3.5）。

实现参考：`bfe/bfe_config/bfe_cluster_conf/cluster_conf/cluster_conf_load.go` 中的 `ModelPrice` 与价格字段常量、`bfe/bfe_basic/request_ai_basic.go` 中的 `DetectModeFromPath`、`bfe/bfe_basic/openai_endpoint.go` 中的 `openAIEndpointModes`（OpenAI 共享端点表，13 个端点）、`bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go` 中的 `calcVideoGenerationCost` / `calcResponsesCost`。完整计费语义见 `bfe/docs/zh_cn/sys_design/rmb_quota.md`。

模式识别与上游路径改写共用同一张端点表，两者对"什么是 OpenAI 标准端点"的判断永远一致：`DetectModeFromPath` 先剥离可选的 `/v1` 前缀（标准入口带不带 `/v1` 识别结果相同），并把以 `/v1` 段结尾的 provider 原生前缀（OpenAI SDK base_url 形式，如百炼 `/compatible-mode/v1/responses`）归约到同一端点再查表。因此客户端按 provider 原生路径发起的请求也能正确识别模式与计费；未命中端点表的路径按默认 chat 模式处理。

## Provider 与 Cluster 的关联

Provider 与 Cluster 通过 `cluster.llm_config.provider` 建立强引用关系。推荐配置顺序为：

```text
/providers → /model-prices → /clusters → 路由规则
```

一个 Cluster 只能引用一个 Provider，但一个 Provider 可被多个 Cluster 引用。Cluster 的 `llm_config` 示例：

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

关键约束包括：`llm_config.provider` 必填且必须引用已存在的 Provider；`llm_config.models` 必须是 Provider `models` 的子集；`llm_config.keys` 中的 `name` 必须对应 Provider 中存在的 Key。Provider 的 `instance_pool` 变更会自动同步到引用它的所有 Cluster，实现一处修改、全局生效。删除 Provider 前，必须确保没有 Cluster 引用，否则返回 `409 Conflict`。

BFE 最终接收到的配置由控制面自动合并生成：`AIConf.Keys` 通过 `name` join Provider 的 Key 明文与 Cluster 的 Key 权重；`AIConf.ModelProtocols` 透传 Provider 的协议列表；`AIConf.ModelTable` 由控制面根据 Provider 查询 `model-prices` 自动填充。因此 Cluster 无需关心密钥明文与价格数据，只需维护转发策略。

## 常见问题与排查

### 1. 创建 Provider 时报“instance_pool 不合法”

检查是否至少填写了一个实例；IP 模式下实例间 IP 是否重复；端口是否在 1-65535 范围内；权重是否在 0-100 之间，且所有实例权重之和等于 100（服务商域名模式权重固定 100，不受此约束）；是否至少有一个实例的 `weight > 0`。

### 2. 更新 Provider 的 Keys 或 Models 时返回 409

说明当前仍有 Cluster 引用被删除/重命名的 Key，或被移除的 Model。解决步骤：先修改对应 Cluster 的 `llm_config.keys` 或 `models`，解除引用后再更新 Provider。

### 3. 删除 Provider 时返回 409

该 Provider 仍被至少一个 Cluster 引用。需要先删除或修改引用它的 Cluster，再删除 Provider。

### 4. 模型发现返回空列表或报错

检查 `model_protocol` 是否与实际平台匹配；`addr`、`port`、`uri` 是否正确；`apikey` 是否有效。对于 Claude 平台，确认 `model_protocol` 选择 `anthropic`；对于 Google Gemini 平台，选择 `gemini`（模型发现默认路径 `/v1beta/models`，认证头 `x-goog-api-key`）。若点「获取」无反应，先检查实例地址、端口、协议与 Key 是否正确，以及 BFE / API Server 能否访问该后端。

### 5. Cluster 引用 Provider 后转发失败

检查 Cluster 的 `llm_config.models` 是否为 Provider `models` 的子集；`llm_config.keys` 中的 `name` 是否存在于 Provider 的 `keys` 中；请求协议风格是否在 Provider 的 `model_protocols` 中。

### 6. 模型价格未生效

检查 `/model-prices` 中是否存在对应的 `(provider, model, mode)` 记录；`model-list.yaml` 导入是否成功，关注 `errors` 列表；`price_currency` 是否为 `RMB`； prices 与 tier_prices 中价格字段是否为非负数。

### 7. 配置了 protocol_paths 但请求未被改写（上游 404）

先确认请求路径命中 OpenAI 共享端点表：openai 协议下带不带 `/v1` 前缀均可改写，但剥离可选 `/v1` 前缀后未命中端点表（`bfe_basic/openai_endpoint.go`）的自定义路径会原样透传，属预期行为；anthropic 协议仅改写标准入口 `/v1/...`，非标准入口透传。再确认请求协议已在 `model_protocols` 中声明且 `protocol_paths` 有对应条目；确认 BFE 版本已支持 `AIConf.ProtocolPaths`（加载期会做 key 白名单校验，非法配置拒绝加载并指明 cluster）；若发生了 fallback，切换到未配置 `protocol_paths` 的备用 cluster 后透传属预期行为。

## 配置示例

### 完整 Provider JSON 配置

```json
{
    "name": "anthropic",
    "description": "Anthropic Claude 官方 API",
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

### 模型发现请求示例

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

`gemini` 协议示例（`uri` 留空时默认 `/v1beta/models`，认证头自动使用 `x-goog-api-key`）：

```bash
curl -X POST https://control-plane.example.com/v1/providers/tools/discover-models \
  -H "Content-Type: application/json" \
  -d '{
    "model_protocol": "gemini",
    "schema": "https",
    "addr": "generativelanguage.googleapis.com",
    "port": 443,
    "apikey": "AIzaSyaaaaaaaaaaaa"
  }'
```

### Cluster TOML 语义示意

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

Cluster 不声明 `instance_pool`、`model_endpoint` 或 `provider_type`，这些信息全部来自 Provider。

## 本章小结

Provider 是壬远 AI 网关控制面中描述下游模型提供方的核心资源。Provider 与 Cluster 职责分离后，Cluster 专注转发策略，Provider 专注接入信息，提升了配置复用性、安全性与可维护性。

本章重点包括：Provider 的数据模型与字段含义；通过 Dashboard 与 OpenAPI 创建、更新 Provider 的流程（Dashboard 抽屉的基本信息、实例池、模型服务配置、协议路径映射、服务鉴权 Keys 五个分区）；实例池 IP 模式与服务商域名模式的差异与权重约束；模型端点、模型列表、Provider Keys 的配置方法与约束；`/providers/tools/discover-models` 无状态模型发现工具的使用；服务商列表常用操作，包括查询模型价格、查看详情、分段计价配置（忙时时间段、IANA 时区、左闭右开匹配）与删除；`protocol_paths` 协议路径映射的配置方法、约束与转发行为；`openai`、`anthropic` 与 `gemini` 协议对认证头、版本头、Usage 解析与协议匹配的影响；通过 `model-list.yaml` 批量导入模型定价的流程与注意事项；图片输入 token 与视频按个计费等价格字段及 `responses`、`video_generation` 计费模式；Provider 与 Cluster 的强引用关系以及变更时的同步与冲突处理；常见问题的排查思路与配置示例。

合理规划 Provider 与 Cluster 的拆分，是后续路由规则、API-Key 配额、限流策略生效的重要前提。建议在生产环境中先统一维护 Provider 与模型价格，再按需创建不同业务线的 Cluster。定期对比 `/providers` 与 `/model-prices/actions/get-providers` 返回的 provider 列表，可及时发现并补录价格记录与实际 Provider 脱节的问题，确保成本核算准确。

## 参考文档

- `ai-gateway-web/docs/zh-cn/03-model-provider.md`（Dashboard 模型服务商用户手册，本章控制台内容的权威依据）
- `ai-gateway-web/docs/zh-cn/06-model-prices.md`（Dashboard 模型定价用户手册）
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/providers.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/model-prices.md`
- `ai-gateway-api/design-docs/sys-design/details/provider与cluster概念分离.md`
- `ai-gateway-api/design-docs/sys-design/details/Claude协议转发支持.md`
