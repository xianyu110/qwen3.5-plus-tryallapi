# Qwen3.5 Plus API 调用教程：用 TryAllAPI 做 OpenAI 兼容对话接入

> **Qwen3.5 Plus**（TryAllAPI 模型 ID：`qwen3.5-plus`）是通义 Qwen3.5 原生视觉-语言 Plus 档。价格接口描述提到混合线性注意力与稀疏 MoE 架构、相较 3 系在文本与多模态任务上的能力跃升。标签为「对话、思考」。本文记录如何通过聚合站 **[TryAllAPI](https://tryallapi.com/)**，从模型广场 / 价格页选型、读基价与分组倍率，到 `POST /v1/chat/completions` 的完整接入流程。价格与分组倍率取数 **2026-11-06（北京）**，**以页面实时为准**；示例仅为请求形态说明，未跑通真实长任务计费。

**Keywords:** Qwen3.5 Plus · qwen3.5-plus · 通义 · Qwen · TryAllAPI · OpenAI 兼容 · `/v1/chat/completions`

---

## 目录

- [一、为什么用聚合站调 Qwen](#一为什么用聚合站调-qwen)
- [二、在价格页 / 模型广场找到 Qwen3.5 Plus](#二在价格页--模型广场找到-Qwen3.5-Plus)
- [三、模型怎么选：与相邻档对照](#三模型怎么选与相邻档对照)
- [四、价格怎么读](#四价格怎么读)
- [五、调用流程：Key → Base URL → chat/completions](#五调用流程key--base-url--chatcompletions)
- [六、curl / SDK 最小示例](#六curl--sdk-最小示例)
- [七、常见注意](#七常见注意)
- [FAQ](#faq)
- [八、小结与试用入口](#八小结与试用入口)

---

## 一、为什么用聚合站调 Qwen

只用通义、要阿里云官方发票时，直连 DashScope 即可。如果项目里还要在 GPT、Claude、GLM 之间切换，聚合站更省事：

1. **统一鉴权**：一个 Bearer Key，切换 `model` 字段即可；
2. **OpenAI 兼容形态**：Base URL 固定为 `https://tryallapi.com/v1`，Cursor / Dify / 各类 SDK 改地址即可；
3. **模型广场可对照**：标签、分组倍率、端点类型一目了然。

代价是多一跳：延迟与可用性取决于分组上游，建议先用小流量探针。注册试用入口：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 二、在价格页 / 模型广场找到 Qwen3.5 Plus

打开 [价格页](https://tryallapi.com/pricing) 或模型广场，按供应商 **Qwen**、类型 **对话**、关键词 `qwen3.5-plus` 筛选。详情卡截图如下：

![Qwen3.5 Plus 模型广场详情卡：模型 ID、API 与分组价格](model-square.png)

详情卡要点（取自价格接口 2026-11-06（北京） 快照）：

| 字段 | 内容 |
| --- | --- |
| **模型 ID** | `qwen3.5-plus` |
| **供应商** | Qwen（通义） |
| **标签** | 对话、思考 |
| **端点类型** | `openai`（对话主路径：`POST /v1/chat/completions`） |
| **可用分组** | Alibaba-1、Alibaba-2、Alibaba-3 |
| **计费** | 按量（token） |

文中 UI 品牌可能显示为模型广场 / 第三方壳，实际对接产品以 [TryAllAPI](https://tryallapi.com/) 控制台与价格页为准。

---

## 三、模型怎么选：与相邻档对照

只根据价格接口可见信息做对照，**不编造未出现的能力差异**。基价按价格接口换算（$/1M，未乘分组倍率）：

| 模型 ID（可见） | 基价 输入 / 输出 / 缓存读 | 怎么理解 |
| --- | --- | --- |
| **`qwen3.5-plus`** | $0.4 / $2.4 / — | 本文主线：3.5 原生视觉-语言 Plus；标签对话、思考 |
| `qwen3.8-max` | $2 / $6 / $0.25 | 同厂商更高代 Max 旗舰；基价更高 |
| `qwen-max` | $1.6 / $6.4 / $0.32 | 经典 Max 档；标签为对话 |
| `qwen3-vl-plus` | $0.2 / $1.6 / — | VL Plus 档；标签对话、识图，基价更低 |
| `qwen3.6-plus` | $0.5 / $3 / — | 同系列 3.6 Plus；标签含工具、识图 |

实务上：经 TryAllAPI 调用统一写精确字符串 `qwen3.5-plus`；遇限流或分组无货，再在同厂商可见 ID 之间切换，而不是改 Base URL。

---

## 四、价格怎么读

对话模型常见公式：**≈ 基价 × 分组倍率**（以控制台为准）。取数快照：**2026-11-06（北京）**；**以页面实时为准**。

换算约定：基价输入 $= model_ratio×2；输出 $= 输入×completion_ratio；缓存读 $= 输入×cache_ratio（若有）；缓存创建（若有）= 输入×cache_creation_ratio。分组估价 = 基价 × group_ratio。

### 基价（每百万 tokens，由价格接口换算）

| 计费项 | 基价（$/1M，未乘分组） |
| --- | --- |
| 输入 | **$0.4** |
| 输出 | **$2.4** |

公开接口按 **基价 × group_ratio** 估算即可；**实际扣费以控制台为准**。

本模型快照未见 cache_ratio / cache_creation_ratio 字段。 价格接口另有 `step_ratios` 分档字段（超长上下文可能抬升倍率）；**以页面与控制台实时为准**。

### 分组价格估算（基价 × group_ratio，$/1M）

| 分组 | group_ratio | 输入估 | 输出估 |
| --- | --- | --- | --- |
| **Alibaba-1** | 1.1 | ≈ $0.44 | ≈ $2.64 |
| **Alibaba-2** | 1.65 | ≈ $0.66 | ≈ $3.96 |
| **Alibaba-3** | 2.42 | ≈ $0.968 | ≈ $5.808 |

**最终扣费以控制台流水为准**。卡片展示价取自最低 `group_ratio` 分组 **Alibaba-1**（与定价页公开数字一致）。

---

## 五、调用流程：Key → Base URL → chat/completions

```text
控制台创建 API Key（选本模型 enable_groups 中的分组）
  → Base URL = https://tryallapi.com/v1
  → POST /v1/chat/completions   （model=qwen3.5-plus）
  → 读 choices[0].message.content（或流式 delta）
```

建议：

- **超时**：长输出 / 思考模式建议开流式并调高 timeout；
- **密钥**：只放环境变量（如 `TRYALLAPI_KEY`），勿提交仓库；
- **端点类型**：价格接口标注端点类型：`openai`。本文以 OpenAI 兼容 `chat/completions` 为主，其它端点以站点说明为准。

---

## 六、curl / SDK 最小示例

占位 Key `sk-xxx`（**不要**填入真实 Key）。

### 1）OpenAI 兼容 chat/completions

```bash
curl -sS https://tryallapi.com/v1/chat/completions \
  -H 'Authorization: Bearer sk-xxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen3.5-plus",
    "messages": [
      {"role": "user", "content": "把这份多步调研任务拆成可执行的验收清单（不超过 8 条）"}
    ]
  }'
```

### 2）Python（OpenAI SDK）

```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-xxx",  # 建议改为读取环境变量
    base_url="https://tryallapi.com/v1",
    timeout=180,
)

r = client.chat.completions.create(
    model="qwen3.5-plus",
    messages=[{"role": "user", "content": "用三句话说明如何给这段多步 agent 工作流写回归用例。"}],
)
print(r.choices[0].message.content)
```

Cursor：Settings → Models → OpenAI API Key + Override Base URL `https://tryallapi.com/v1` → 自定义模型 `qwen3.5-plus`。Dify：OpenAI-API-compatible，endpoint 填 `https://tryallapi.com/v1`。

---

## 七、常见注意

1. **模型名**：请求体 `"model"` 必须写广场上的精确 ID `qwen3.5-plus`（含连字符 / 点号，大小写敏感）；写混成简称或漏掉后缀容易 404。
2. **基价 × 倍率**：估算用价格接口快照；峰谷 / 促销字段若页面另有说明，以控制台为准。
3. **分组**：403 多为分组无货，换 `enable_groups` 里其他分组；低倍率分组先抽检质量与延迟。
4. **报错速查**：401 查 Bearer / 令牌；404 确认模型名且 base_url 含 `/v1`；429 退避或换分组。
5. **合规**：第三方聚合不是 Qwen 官方服务，敏感代码先看条款。

---

## FAQ

### Qwen3.5 Plus 的模型 ID 是什么？

TryAllAPI 上是 **`qwen3.5-plus`**。请求体 `"model"` 填该字符串。

### TryAllAPI 的对话 endpoint 是什么？

- Base URL：`https://tryallapi.com/v1`
- 对话：`POST /v1/chat/completions`
- 价格接口标注端点类型：`openai`。本文以 OpenAI 兼容 `chat/completions` 为主，其它端点以站点说明为准。

### 基价大概是多少？

按 2026-11-06（北京） 价格接口快照：输入约 $0.4 / 输出约 $2.4（$/1M，未乘分组）。再 × `group_ratio` 得分组估价；**以页面实时为准**。

### 如何试用 TryAllAPI？

站点首页：[https://tryallapi.com/](https://tryallapi.com/)；注册试用入口：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 八、小结与试用入口

把 Qwen3.5 Plus 接到业务里，核心是：**在价格页确认 model id 与 OpenAI 兼容 Base URL → 读懂基价 × 分组倍率 → 用 chat/completions 先跑通非流式 ping，再开流式 / 接工具调用**。

| 资源 | 链接 |
| --- | --- |
| TryAllAPI 首页 | [https://tryallapi.com/](https://tryallapi.com/) |
| 注册试用（aff） | [https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A) |
| 价格页 | [tryallapi.com/pricing](https://tryallapi.com/pricing) |

价格取数 2026-11-06（北京），以页面实时为准。

---

## License

本文档以个人接入笔记形式发布，示例代码可自由复用。截图与模型广场文案归原平台；价格以实时页面为准。TryAllAPI 为第三方聚合，非 Qwen 官方产品。
