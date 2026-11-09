# 第十三章 模型定价与成本核算设计

## 本章目标

通过本章，读者将理解：

- 模型定价数据在 AI 网关成本核算、配额扣减和成本分摊中的作用；
- `ModelPrice` 数据模型的字段含义与校验规则；
- `model-list.yaml` 批量导入格式及其使用方式；
- RMB 配额分时段定价（Tiered Pricing）的设计思路与实现机制；
- BFE 数据面如何基于定价数据完成运行时成本计算；
- 批量任务（batch）的折扣定价模型与导出展开规则；
- image/video entry 驱动计费与"mode 无价格条目回落 chat 价"的兜底计费规则；
- 模型定价与 Provider、Cluster 概念分离后的关系与配置归属。

---

## 模型定价数据的作用

在壬远AI网关中，模型定价数据（Model Price）是连接"资源使用"与"成本核算"的桥梁。当业务系统通过 API-Key 调用模型服务时，BFE 数据面会解析请求与响应中的 Token 用量，并依据控制面下发的定价数据计算本次调用产生的成本。

模型定价数据主要服务于以下场景：

- **配额扣减**：API-Key 或 Entity 绑定的 RMB 配额，需要按实际调用成本精确扣减。如果定价缺失或不准确，会导致配额消耗与实际成本偏离，影响预算控制；
- **成本核算**：为运营者提供按模型、按 Provider、按 API-Key 乃至按 Entity 的调用成本统计，支撑成本洞察与优化决策；
- **成本分摊**：在多租户或多业务共享网关时，按实际消耗价格进行成本分摊与账单生成，实现内部结算；
- **高峰/闲时定价**：部分模型提供商采用分时段报价，网关需要按请求发生时刻匹配不同价格，从而真实反映供应商计费规则；
- **缓存计费**：支持对提示缓存（prompt caching）命中与未命中的 Token 分别计价，鼓励缓存复用并准确反映成本结构。

模型定价数据由 AI Gateway API 控制面统一维护，通过 InnerAPI 导出后下发给 BFE。BFE 在转发请求时完成实时成本计算，并将结果用于配额扣减和日志审计。控制面与数据面的价格版本保持一致，避免了多源数据带来的核算差异。

---

## ModelPrice 数据模型

`ModelPrice` 是控制面维护的单条模型定价记录，主键为 `(provider, model, mode)` 三元组。OpenAPI 路径 `/model-prices` 提供了完整的 CRUD 与批量导入能力。

### 核心字段

```json
{
  "id": 1,
  "provider": "deepseek",
  "model": "deepseek-v3",
  "base_model": "deepseek-v3",
  "mode": "chat",
  "capabilities": ["chat", "reasoning", "tools"],
  "supported_parameters": ["temperature", "max_tokens"],
  "limits": {
    "context_window": 128000,
    "max_input_tokens": 128000,
    "max_output_tokens": 8192
  },
  "prices": {
    "input_cost_per_token": 0.000002,
    "output_cost_per_token": 0.000008,
    "cache_read_input_token_cost": 0.0000005
  },
  "tier_prices": {
    "peak": {
      "input_cost_per_token": 0.000004,
      "output_cost_per_token": 0.000016,
      "cache_read_input_token_cost": 0.000001
    }
  },
  "price_currency": "RMB",
  "metadata": {
    "source": "https://platform.deepseek.com/pricing",
    "notes": "DeepSeek V3"
  },
  "create_time": 1716883200,
  "update_time": 1716883200
}
```

各字段说明如下：

| 字段 | 说明 |
|------|------|
| `provider` | Provider / Cluster 标识，仅作为价格归集标识，不强制要求在 `/providers` 中存在 |
| `model` | 模型名，如 `deepseek-v3` |
| `base_model` | 归一化模型名，用于模型映射后的价格归一 |
| `mode` | 请求模式，如 `chat`、`completion`、`embedding`、`image_generation`、`batch` 等 |
| `capabilities` | 模型支持的能力列表，如 `vision`、`reasoning`、`tools` |
| `supported_parameters` | 支持的请求参数列表，如 `temperature`、`max_tokens` |
| `limits` | 限制对象，包括上下文窗口、最大输入/输出 Token 数等 |
| `prices` | 默认价格表，未命中 tier 时作为 fallback |
| `tier_prices` | 分时段价格表，键为 tier name（初期只支持 `peak`） |
| `batch_discount` | 批量折扣系数，`*float64`，取值 `(0, 1]`；语义上缺省为 0.5（五折），见下文"批量任务定价" |
| `price_currency` | 价格货币，当前固定为 `RMB` |
| `metadata` | 元数据，包括价格来源、备注等 |

### 价格字段枚举

`prices` 与 `tier_prices.<tier>` 中可使用的键名包括：

| 键名 | 说明 |
|------|------|
| `input_cost_per_token` | 每 Token 输入成本 |
| `output_cost_per_token` | 每 Token 输出成本 |
| `cache_read_input_token_cost` | 缓存读取输入 Token 成本 |
| `cache_creation_input_token_cost` | 缓存创建输入 Token 成本 |
| `input_cost_per_token_above_200k_tokens` | 超过 200k Token 的输入成本 |
| `output_cost_per_token_above_200k_tokens` | 超过 200k Token 的输出成本 |
| `output_cost_per_image` | 每张输出图像成本（image_generation 模式） |
| `input_cost_per_image_token` | 每图片输入 Token 成本（chat 与 image_generation 模式均可用） |
| `output_cost_per_video` | 每个生成视频成本（video_generation 模式） |
| `output_cost_per_pixel` | 每像素输出成本 |
| `input_cost_per_audio_per_second` | 每秒音频输入成本 |
| `input_cost_per_video_per_second` | 每秒视频输入成本 |
| `output_cost_per_second` | 每秒输出成本 |
| `ocr_cost_per_page` | 每页 OCR 成本 |
| `output_cost_per_character` | 每字符输出成本 |

当前版本已启用按 Token 计费、缓存计费、图片计费与视频计费相关的价格项，其余字段为后续多模态计费预留。`input_cost_per_image_token` 与 `output_cost_per_video` 两个价格键支持配置在全局 `prices` 与 `tier_prices.<tier>` 两级，缺省值为 0（即该维度不计费），负值在配置加载时直接报错。

### 价格精度

`prices` 与 `tier_prices` 中的价格字段为 `float64` 浮点数（有效数字约 15 位），业务上支持远超 8 位的小数精度，例如 `0.0000015`、`7.6234102728e-08`（10~12 位小数的目录价格可无损表示）。输入侧（OpenAPI 请求体、`model-list.yaml` 导入）十进制表示法与科学计数法均可，两者等价；输出侧由标准编码器序列化，极小数值以科学计数法呈现（如 `1.5e-6`），两者均为合法 JSON number，数值语义不变。校验规则为：价格非负，且 `|price × 1e8| < 2^53`（约 9e15），保证下游 BFE 浮点计费不溢出。

---

## model-list.yaml 导入格式

当需要批量维护大量模型定价时，可通过 `/v1/model-prices/import` 接口导入 `model-list.yaml` 文件。该文件是 `model_prices` 表的权威数据源之一。

### 顶层结构

```yaml
version: v1.0                    # 格式版本，必填
default_currency: "RMB"          # 全局默认币种，当前仅支持 RMB
models:                          # 模型列表，必填
  - ...
```

### 单条记录结构

```yaml
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
      cache_read_input_token_cost: 0.0000005
    tier_prices:
      peak:
        input_cost_per_token: 0.000004
        output_cost_per_token: 0.000016
        cache_read_input_token_cost: 0.000001
    metadata:
      source: "https://platform.deepseek.com/pricing"
      notes: "DeepSeek V3"
```

### 导入模式

`/v1/model-prices/import` 支持两种导入模式：

- **replace**（默认）：先清空 `model_prices` 表，再写入新数据，适用于全量更新；
- **merge**：对已有 `(provider, model, mode)` 记录更新，新增记录插入，适用于增量维护。

导入接口仅允许管理员调用。导入过程中会校验必填字段、`prices` 非负、`tier_prices` 键名合法性、`limits` 非负整数以及三元组唯一性。

---

## 批量任务定价（batch 折扣）

批量与异步任务（OpenAI Batch API，见[第十二章 配额与限流设计](./chapter12-quota-and-rate-limit.md)的批量配额模型）按低于在线调用的价格结算，定价模型通过 `mode` 与 `batch_discount` 两个字段表达。

### mode 白名单与 batch_discount

`model_prices` 的 `mode` 白名单含 `batch`，表示该价格行是 batch 模式的显式定价。价格行另含 `batch_discount` 字段：

- 类型为 `*float64`（可空指针），取值区间 `(0, 1]`（`ai-gateway-api/model/imodel_price/validate.go` 对越界值报错）；
- 语义上缺省表示 0.5，即五折：配置了折扣的 chat 价格行在导出时展开为 batch 价格。

### 导出展开规则

控制面导出 `AIConf.ModelTable` 时按如下规则生成 `mode=batch` 的显式价格行：

1. 价格行配置了 `batch_discount` 时，以 `chat` 价 × 折扣系数展开出一条 `mode=batch` 价格行，随 ModelTable 一并下发给 BFE；
2. 若管理员已手工维护了 `(provider, model, mode=batch)` 的价格行，则手工行优先于系数展开——导出不会用展开结果覆盖显式配置；
3. 模型完全未配置任何 batch 价格时，batch 任务按 chat 基价结算。

### batch 结算基价

批量结算在结果文件下载时按 batch 价、按模型执行，结算金额按基价公式计算：`settle_units = usage_in × input_cost_per_token + usage_out × output_cost_per_token`。一期 batch 结算只接入了基价，`tier_prices` 分时段价未接入批量结算。BFE 数据面 `mod_ai_batch` 的预留-结算-释放实现见[第三十二章 Token认证与配额模块实现：mod_ai_token_auth](../implementation/chapter32-mod-ai-token-auth.md)。

---

## RMB 配额分时段定价机制

随着 DeepSeek 等模型提供商采用"高峰 / 空闲"分时段定价策略，RMB 配额扣减需要具备按请求发生时刻匹配不同价格的能力。

### 核心概念

| 概念 | 说明 |
|------|------|
| **Tier** | 按时间维度划分的价格层级，如 `peak`（高峰）。请求命中某个 tier 时使用对应价格，否则 fallback 到默认 `prices` |
| **TimeRange** | 一个时段定义，包含 `weekdays`（星期几，0=周日）、`start` / `end`（HH:MM，左闭右开） |
| **Provider 时段模板** | 定义在 `/providers` 上的 `time_zone` 和 `tiers`，同一 provider 下所有模型共享 |
| **Model tier 价格** | 定义在 `/model-prices` 上的 `tier_prices`，描述某个模型在某个 tier 下的价格 |

### 配置归属与下发链路

```
/provider deepseek
    ├── time_zone: Asia/Shanghai
    └── tiers: [peak]
    ├── model-prices deepseek-v3 chat
    │       ├── prices: 默认/空闲价
    │       └── tier_prices.peak: 高峰价
    └── model-prices deepseek-v4 chat
            └── ...

/cluster my-cluster
    ├── llm_config.provider: deepseek
    └── AIConf.ModelTable（导出时由控制面把 provider 的 time_zone/tiers
        与 model-prices 的 prices/tier_prices 拼接）
```

多个 cluster 引用同一个 provider 时，会各自得到一份相同的 `ModelTable` 数据；provider 的时段规则变更后，所有引用它的 cluster 在下一次配置导出时自动生效。

### Provider 时段模板

`/providers` 提供 `time_zone` 与 `tiers` 字段，用于描述共享的时段规则：

```json
{
  "name": "peak",
  "time_ranges": [
    { "weekdays": [1, 2, 3, 4, 5], "start": "09:00", "end": "12:00" },
    { "weekdays": [1, 2, 3, 4, 5], "start": "14:00", "end": "18:00" }
  ]
}
```

`time_zone` 须为合法 IANA 时区名，默认 `Asia/Shanghai`。同一 tier 内部的 `time_ranges` 不得重叠，且 `end` 必须大于 `start`；跨午夜时段需拆成两段。

### BFE ModelTable 结构

控制面导出给 BFE 的 `AIConf.ModelTable` 结构如下：

```go
type TimeRange struct {
    Weekdays []int  // 0=周日，1=周一 ... 6=周六
    Start    string // "HH:MM"
    End      string // "HH:MM"，必须 > Start
}

type PriceTier struct {
    Name       string      // 初期只支持 "peak"
    TimeRanges []TimeRange
}

type PriceMap map[string]float64
type TierPriceMap map[string]map[string]float64

type ModelPrice struct {
    Provider            string
    Model               string
    BaseModel           string
    Mode                string
    Capabilities        []string
    SupportedParameters []string
    Limits              map[string]interface{}
    Prices              PriceMap
    TierPrices          TierPriceMap
    Metadata            map[string]interface{}
}

type ModelTable struct {
    Currency string
    TimeZone string
    Tiers    []PriceTier
    Models   []ModelPrice
}
```

---

## BFE 侧成本计算逻辑

BFE 数据面在加载 `AIConf` 时完成价格数据的预处理，在请求转发结束后根据实际 Token 用量和当前时段计算成本。

### 配置加载阶段

```
AIConf.ModelTable
       │
       ▼
┌──────────────┐
│ 解析 TimeZone │─── 默认 Asia/Shanghai
│ 校验 Tiers    │─── name、时间格式、weekdays、区间重叠
└──────────────┘
       │
       ▼
┌──────────────┐
│ 价格校验         │─── 非负校验；价格保持 float64，不做定点转换
└──────────────┘
```

BFE 加载后价格保持 `float64`（元/token），不做定点预转换。对每个模型的 `prices` 与每个 tier 的 `tier_prices.<tier>` 执行校验：所有价格键（含 `input_cost_per_image_token`、`output_cost_per_video` 等）缺省视为 0，出现负值则整个配置文件加载失败并报出具体模型名与价格键，避免错误价格进入运行时。请求计费时逐项调用 `quota.CalcCostUnits(用量, 价格)`（即 `round(用量 × 价格 × 1e8)`）换算为 1e-8 元定点整数后累加，浮点只参与单项乘法，避免浮点累加误差。

### 运行时时段匹配

BFE 在请求结束时根据当前时间匹配活跃 tier：

```go
func (table *ModelTable) ActiveTierName(now time.Time) string {
    if table == nil || len(table.Tiers) == 0 {
        return ""
    }
    t := now.In(table.tz)
    wd := int(t.Weekday())
    hour, min := t.Hour(), t.Minute()
    cur := hour*60 + min

    for i := range table.Tiers {
        tier := &table.Tiers[i]
        for _, tr := range tier.TimeRanges {
            if len(tr.Weekdays) > 0 && !containsInt(tr.Weekdays, wd) {
                continue
            }
            start := parseHHMM(tr.Start)
            end := parseHHMM(tr.End)
            if start <= cur && cur < end {
                return tier.Name
            }
        }
    }
    return ""
}
```

### 运行时成本计算

成本计算流程如下：

```
请求结束
   │
   ▼
解析 TokenUsage（协议适配层 bfe_model_protocol 统一提取）
   ├── PromptTokens（总输入，含缓存读写与图片/音频输入）
   ├── CompletionTokens
   ├── CacheReadTokens / CacheWriteTokens
   ├── AudioInputTokens / AudioOutputTokens
   ├── ImageInputTokens / ImageCount
   └── VideoCount
   │
   ▼
匹配 ActiveTierName
   │
   ├── 命中 tier ──► 取 tier_prices.<tier> 价格，缺键回退默认 prices
   └── 未命中 tier ──► 取 prices 默认价格
   │
   ▼
按请求模式分别计算（calcChatCost 等，逐项 round(用量 × 价格 × 1e8) 后整数累加）
   │
   ▼
累加为本次请求总成本，用于 RMB 配额扣减与日志输出
```

#### chat 计费公式

`calcChatCost`（`bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`）以总输入 Token 为起点做维度拆分：

```
normalInput = PromptTokens                          // 未配置缓存价格时
            = max(PromptTokens - CacheRead - CacheWrite, 0)   // 配置了缓存价格时

配置了 input_cost_per_image_token 时：
    imageInputTokens = min(ImageInputTokens, normalInput)
    normalInput -= imageInputTokens

配置了 audio 输入价格时：
    audioInputTokens = min(AudioInputTokens, normalInput)
    normalInput -= audioInputTokens

配置了 audio 输出价格时：
    normalOutput = CompletionTokens - AudioOutputTokens
    （audioOutputTokens 上限截断为 CompletionTokens）

成本 = normalInput  × input_cost_per_token
     + CacheRead    × cache_read_input_token_cost
     + CacheWrite   × cache_creation_input_token_cost
     + imageInput   × input_cost_per_image_token
     + audioInput   × input_cost_per_audio_token
     + normalOutput × output_cost_per_token
     + audioOutput  × output_cost_per_audio_token
```

当 cache / audio / image 的任何细化价格键都未配置时，回退 legacy 公式 `PromptTokens × input_cost_per_token + CompletionTokens × output_cost_per_token`，保证旧配置行为不变。

#### Anthropic 用量语义归一化

Anthropic 协议的 `input_tokens` 只统计未命中缓存的新鲜 Token，不含 `cache_read_input_tokens` 与 `cache_creation_input_tokens`，与 OpenAI 的 `prompt_tokens`（总输入）语义不同。归一化在解析层（`bfe/bfe_model_protocol/utils/usage_parse.go` 的 `ParseAnthropicUsageFields`）完成：

```
PromptTokens = input_tokens + cache_read_input_tokens + cache_creation_input_tokens
```

使下游的"总输入减缓存"拆分逻辑对 OpenAI 与 Anthropic 完全一致。流式响应中，`message_start` 事件的 usage 嵌套在 `message.usage` 下，解析层同时兼容该形态；`message_delta` 事件只携带最终 output tokens，计费模块保留 `message_start` 阶段已解析的 prompt/cache 字段，避免缓存 Token 丢失或重复计价。缓存创建 Token 只按 `cache_creation_input_token_cost` 计价一次，避免高缓存命中场景下 fresh tokens 被截断为 0 而完全不计费。

Responses API 的 usage 是子集（subset）语义，不做上述 additive 归一化：`input_tokens` 已包含 `input_tokens_details.cached_tokens`（`total_tokens = input_tokens + output_tokens`，与 Chat Completions 主链一致），因此 `ParseOpenAIUsageFields` 对 Responses 链直接取 `input_tokens` 作为 `PromptTokens`。流式 `response.completed` 事件的 usage 嵌套在 `response.usage` 下，非流式 create-response 对象的 usage 位于顶层、字段名为 `input_tokens`/`output_tokens`，两种形态都在 Responses 专用链中解析。

#### 请求模式识别

请求模式由 `bfe_basic.DetectModeFromPath` 根据请求路径判定（实现位于 `bfe_basic/request_ai_basic.go`，端点表由 `bfe_basic/openai_endpoint.go` 的 `openAIEndpointModes` 提供）：先剥离可选的 `/v1` 版本前缀，再查共享 OpenAI 端点表得到模式——`/chat/completions` 与 `/embeddings`、`/images/generations`、`/responses`、`/video/generations` 等主要端点各有专属模式；`/models`、`/moderations`、`/audio/translations` 等无专属模式的端点归入默认 `ModeChat`；未命中端点表的未知路径同样按 `ModeChat` 处理。前缀以 `/v1` 段结尾的 provider 原生入口（OpenAI SDK `base_url` 形式，如 `/compatible-mode/v1/responses`）也会归约到端点表，识别为 Responses 模式而不是误归为 Chat。该端点表与上游路径改写（`bfe_server/ai_path_rewrite.go`）共用同一份定义，改写资格与计费模式不会相互矛盾。

#### 其他请求模式

另外两个请求模式的计费逻辑如下：

| 模式 | 路径 | 计费公式 |
|------|------|----------|
| `responses` | `/responses`（含 `/v1/responses`、`/compatible-mode/v1/responses` 等入口） | 复用 chat 计费（`calcResponsesCost` 直接委托 `calcChatCost`） |
| `video_generation` | `/video/generations`（含 `/v1/video/generations`） | `VideoCount × output_cost_per_video` |

`VideoCount` 优先取响应 `usage.video_count`，回退响应 `data.#`（生成结果条数）；认证阶段还会预读请求体 `n` 字段作为兜底（`n ≤ 0` 时取 1），防止响应未返回用量时少收。图片生成模式（`image_generation`）的费用为 `ImageCount × output_cost_per_image + ImageInputTokens × input_cost_per_image_token`，其中 `ImageInputTokens` 来源为 `usage.input_token_details.image_tokens`，回退 `usage.image_input_tokens`。

若某个 tier 未配置特定价格键，则自动 fallback 到默认 `prices` 中的对应键。`TokenUsage` 中的 `CacheReadTokens`、`ImageInputTokens`、`VideoCount` 等字段使缓存命中、图片输入与视频生成可以分别计价。

#### image/video entry 驱动计费

image_generation 与 video_generation 的计费由匹配到的价格条目（entry）驱动，判别依据是**价格键是否存在**，而不是键的值（BFE 侧以 `ModelPrice.HasPrice` 判别，`bfe/bfe_config/bfe_cluster_conf/cluster_conf/cluster_conf_load.go`）：

- 条目缺少 per-unit 价格键（`output_cost_per_image` / `output_cost_per_video`）时，回落到 chat token 公式计费（`calcChatCost`），避免未配置图片/视频单价时调用完全不计费；
- 条目同时持有两类键（混合条目）时按 per-unit 计费；
- 显式配置 `output_cost_per_image = 0` 仍按 per-image 计费，只是该维度成本为 0；video 对称。这与"键缺失回落 chat 公式"的语义不同：前者是"该维度免费"，后者是"未单独定价、按 token 计价"。

#### mode 无价格条目的回落计费（ai_price_miss）

价格表没有某个 `(model, mode)` 条目时（例如 `/responses` 入口的模型只配置了 `mode=chat` 价格），BFE 回落到同一模型的 chat 价计费，而不是按 0 计费。该行为伴随可观测标记：

- 模块计数器 `PriceFallbackUsed` 统计回落计费的次数，`PriceMissZeroBilled` 统计连回落基线都不存在的无价格次数（`bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`）；
- 访问日志以 `ai_price_miss` 字段（pb field 763，`bfe-access-pb` ≥ v0.3.14）标记价格缺失：两种 miss（回落计费、无基线）都会置 true，用于区分"未定价的低价/0 计费行"与合法的 0 成本行。字段细节见[第三十二章 Token认证与配额模块实现：mod_ai_token_auth](../implementation/chapter32-mod-ai-token-auth.md)。

#### 缺价处置三态

缺价行为由集群级显式配置 `llm_config.price_miss_action` 决定（snake_case 入库，导出为 `AIConf.PriceMissAction` 的 PascalCase），三态：

| 取值 | 语义 |
|------|------|
| `fallback_chat`（缺省） | 按同模型 chat 价计费（即上节的回落行为） |
| `bill_zero` | 正常转发但按 0 计费 |
| `reject` | **转发前拒绝**：返回 503 `PRICE_NOT_CONFIGURED`，不调用上游、不计费，访问日志照常入库 |

`ModelTable` 整体缺失视同未定价（同样按 `reject` 拒）；cluster 查询失败的 fail-open 行为不变。非法值在控制面导出侧 422、数据面加载侧 `AIConfCheck` 同样报错；存量集群未设置该字段时零迁移（bfe 归一为 `fallback_chat`）。

可观测侧以**信号对**上报：`ai_price_miss`（bool，field 763）与 `ai_price_warn`（string，field **766**，`bfe-access-pb` ≥ v0.3.17，取六种形态——`fallback` / `subitem_missing` / `no_entry` / `no_baseline` / `no_model_table` / `reject`，多因取最具体）；两字段统一 **NULL = 未上报** 语义，`ai_price_miss` 为**请求级**写出（`reject` 的 503 行同样带信号）。报表库明细表共 **101** 列（缺价两列位于 `ai_intent_questions_version` 之后），查询参数支持 `price_warn` 精确过滤（白名单校验，非法值 422）。配套计数器为 `PriceMissZeroBilled`（缺价按 0 计）/ `PriceRejected` / `PriceFallbackSubItemMiss`。已知并接受残余：子项缺失（如缺 cache 读写价键）按 0 计但不拒单。

### 向后兼容

分时段定价设计保持了对固定价格的向后兼容：

- `/providers` 不填 `time_zone` / `tiers` 时，`ModelTable.TimeZone` / `ModelTable.Tiers` 为空，行为与固定价格完全一致；
- `/model-prices` 不填 `tier_prices` 时，始终按默认 `Prices` 计费；
- 命中 tier 但该 tier 未配置某个价格键时，自动 fallback 到默认 `Prices`；
- chat 计费中未配置任何 cache / audio / image 细化价格键时，回退 legacy 公式 `PromptTokens × input_cost_per_token + CompletionTokens × output_cost_per_token`；
- 未配置 `input_cost_per_image_token` / audio 价格时，图片/音频输入 Token 按普通输入 Token 计价；
- `TokenUsage.UsedCost` 的语义、Lua 扣减逻辑与 Redis 定点数存储不受分时段配置影响。

这种兼容方式使得现有部署可以平滑启用分时段能力，无需一次性全量调整配置。

---

## 定价数据与 Provider/Cluster 的关系

在 Provider 与 Cluster 概念分离后，模型定价的归属与引用关系也变得更加清晰。

### 概念分离回顾

- **Provider**：模型提供方，包含接入端点、可用模型、API Key、实例池、支持的协议以及时段模板（`time_zone`、`tiers`）。
- **Cluster**：转发集群，决定把流量按什么模型、什么权重、什么策略转发到某个 provider。
- **Model Price**：一条 `(provider, model, mode)` 的价格记录，`provider` 字段仅作为价格归集标识，不强制引用已存在的 provider。

### 配置下发关系

```
/providers
   ├── name: deepseek
   ├── time_zone: Asia/Shanghai
   └── tiers: [peak]

/model-prices
   ├── provider: deepseek, model: deepseek-v3, mode: chat
   │       ├── prices
   │       └── tier_prices.peak
   └── provider: deepseek, model: deepseek-v4, mode: chat
           ├── prices
           └── tier_prices.peak

/clusters
   └── name: my-cluster
           └── llm_config.provider: deepseek

导出 AIConf 时：
   ModelTable.TimeZone ← /providers.deepseek.time_zone
   ModelTable.Tiers    ← /providers.deepseek.tiers
   ModelTable.Models   ← /model-prices 中 provider=deepseek 的记录
```

### 弱引用带来的灵活性

`/model-prices` 的 `provider` 字段不再强制引用 `/providers`，带来了以下好处：

- 配置顺序更灵活，可以先维护价格数据，再接入 Provider；
- 删除 Provider 时，同名 `model_prices` 记录不再阻塞删除；
- 历史价格记录可以保留，即使对应 Provider 已下线，也便于成本追溯。

推荐配置顺序为 `/providers → /model-prices → /clusters → 路由规则`，但 `/model-prices` 与 `/providers` 之间为弱引用关系，实际可独立维护。

### 对成本计算的影响

弱引用关系意味着：BFE 在进行成本计算时，只关心 `AIConf.ModelTable` 中是否包含对应 `(provider, model, mode)` 的价格记录，而不关心该 provider 是否仍然存在。即使某个 Provider 已被删除，只要其价格记录保留，历史请求的成本核算依然可以正确完成。

同时，由于 `ModelTable` 在导出时按 `cluster.llm_config.provider` 查询并拼接，一个 cluster 引用的所有模型价格必须存在，否则该模型调用将无法完成成本计算，相关配额扣减也会失败。因此，在删除或重命名 Provider 时，需要同步评估其对价格数据和 cluster 引用的影响。

---

## 定价配置示例

以下示例展示如何为一个支持高峰/闲时计价的 Provider 配置模型定价。

### Provider 时段模板

```json
{
  "name": "deepseek",
  "description": "DeepSeek 官方 API",
  "model_protocols": ["openai"],
  "time_zone": "Asia/Shanghai",
  "tiers": [
    {
      "name": "peak",
      "time_ranges": [
        { "weekdays": [1, 2, 3, 4, 5], "start": "09:00", "end": "12:00" },
        { "weekdays": [1, 2, 3, 4, 5], "start": "14:00", "end": "18:00" }
      ]
    }
  ]
}
```

### model-list.yaml 片段

```yaml
version: v1.0
default_currency: "RMB"

models:
  - provider: "deepseek"
    model: "deepseek-v3"
    base_model: "deepseek-v3"
    mode: "chat"
    capabilities: ["chat", "reasoning", "tools", "structured_outputs", "prompt_caching"]
    supported_parameters: ["temperature", "top_p", "max_tokens", "tools", "tool_choice", "response_format", "reasoning"]
    limits:
      context_window: 128000
      max_input_tokens: 128000
      max_output_tokens: 8192
    prices:
      input_cost_per_token: 0.000002
      output_cost_per_token: 0.000008
      cache_read_input_token_cost: 0.0000005
      cache_creation_input_token_cost: 0.000001
    tier_prices:
      peak:
        input_cost_per_token: 0.000004
        output_cost_per_token: 0.000016
        cache_read_input_token_cost: 0.000001
    metadata:
      source: "https://platform.deepseek.com/pricing"
      notes: "DeepSeek V3 官方 API"
```

### Cluster 引用

```json
{
  "name": "deepseek-cluster",
  "llm_config": {
    "provider": "deepseek",
    "models": ["deepseek-v3"],
    "keys": [
      { "name": "key-primary", "weight": 100 }
    ]
  }
}
```

当上述配置导出到 BFE 后，工作日 09:00–12:00 与 14:00–18:00 之间对 `deepseek-v3` 的调用将按高峰价计费，其余时间按默认价格计费。

---

## 本章小结

- 模型定价数据是 AI 网关成本核算、配额扣减与成本分摊的基础，由 AI Gateway API 控制面统一维护并下发给 BFE。
- `ModelPrice` 以 `(provider, model, mode)` 为主键，包含能力、限制、默认价格和分时段价格等字段。
- `model-list.yaml` 提供批量导入能力，支持 `replace` 与 `merge` 两种模式，是当前版本维护模型价格的主要数据源格式。
- RMB 配额分时段定价通过 Provider 时段模板与 Model tier 价格配合实现，BFE 按请求发生时刻匹配 `peak` 等 tier，未命中时 fallback 到默认价格。
- BFE 数据面加载后价格保持 `float64`（负值加载报错），运行时根据 Token 用量和活跃 tier 逐项 `quota.CalcCostUnits` 换算为 1e-8 元定点整数再累加，Redis 扣减与存量配置的金额语义保持不变。
- chat 计费以总输入 Token 为起点拆分缓存读写、图片输入、音频输入等维度；所有细化价格键均未配置时回退 legacy 公式。
- image_generation / video_generation 按"价格键是否存在"（`ModelPrice.HasPrice`）判别：缺 per-unit 价键回落 chat token 公式，显式 0 价仍按 per-unit 计费（成本为 0）。
- 价格表无 `(model, mode)` 条目时回落同模型的 chat 价计费，并通过计数器 `PriceFallbackUsed` / `PriceMissZeroBilled` 与访问日志字段 `ai_price_miss` 暴露可观测性。
- 批量任务通过 `mode=batch` 价格行与 `batch_discount` 折扣系数定价：导出时展开为显式 batch 价格行，手工维护的 batch 行优先于系数展开；batch 结算按 `usage_in × input + usage_out × output` 基价，tier 价一期未接入。
- Anthropic 用量在协议适配层归一化为总输入语义（`input_tokens + cache_read + cache_creation`），与 OpenAI 的 `prompt_tokens` 对齐；Responses API 的 `input_tokens` 是包含 `cached_tokens` 的子集语义，直接取作 `PromptTokens`（不做 additive 累加）；支持 `responses`（复用 chat 计费）与 `video_generation`（按 `output_cost_per_video` × 视频数）两种计费模式。
- Provider 与 Cluster 概念分离后，`model-prices.provider` 仅作为价格归集标识，与 `/providers` 为弱引用关系，配置更灵活；`AIConf.ModelTable` 由控制面在导出时按 provider 拼接生成。

---

## 参考文档

- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/model-prices.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/batches.md`
- `ai-gateway-api/design-docs/sys-design/details/RMB配额分时段定价.md`
- `ai-gateway-api/design-docs/sys-design/details/provider与cluster概念分离.md`
- `bfe/bfe_config/bfe_cluster_conf/cluster_conf/cluster_conf_load.go`
- `bfe/bfe_modules/mod_ai_token_auth/mod_ai_token_auth.go`
- `bfe/bfe_model_protocol/utils/usage_parse.go`
