# 模型映射与厂家差异

本文件记录选型的**依据**和**实测数据**。改模型前先读这里，否则容易把已经踩过的坑再踩一遍。

## 目录

- [选型结论](#选型结论)
- [为什么这么选](#为什么这么选)
- [单价](#单价)
- [实测延迟与吞吐](#实测延迟与吞吐)
- [厂家参数差异](#厂家参数差异)
- [接入点](#接入点)
- [一条方法论教训](#一条方法论教训)

## 选型结论

| 角色 | 模型 | 环境变量 |
| --- | --- | --- |
| Orchestrator | `deepseek-flash` | `DEEPSEEK_API_KEY` |
| Planner | `deepseek-flash` | `DEEPSEEK_API_KEY` |
| Coder | `deepseek-flash` | `DEEPSEEK_API_KEY` |
| Reviewer | `qwen3.8-max` | `DASHSCOPE_API_KEY` |
| Tester | `kimi-k2.7-code` | `MOONSHOT_API_KEY` |

单任务成本约 **$0.21**（玩具级任务实测）。

## 为什么这么选

### Reviewer 与 Coder 必须异源（硬约束）

其他任何一条都可以商量，这一条不行。同一个模型既写又审，会共享同一套盲区和偏好，审查在结构上就会放水——这是 prompt 怎么写都补不了的。

### Reviewer 选 `qwen3.8-max`

它在横评里 14 项共有基准全胜 `qwen3.7-plus`，其中就包含 **IFBench（指令遵循）**。而 Reviewer 的两条硬规则——「没有问题时如实写 pass，不要硬找问题」和「critical/major 未解决时 verdict 必须是 needs_fix」——**本质都是指令遵循，不是知识**。

推理深度的差异也是实打实的：开着 thinking 的那一轮，它抓出了三个真 bug（两条错误的测试断言、一条 `casefold` 展开导致列表比较误判）；关掉 thinking 的那几轮给的都是 pass。

### Coder 选 DeepSeek 而不是 Kimi

直觉上「代码专用模型」该写代码，但数据不支持：

| 基准 | DeepSeek V4.1-Flash | Kimi K2.7 Code |
| --- | --- | --- |
| llmrun 编码分 | 116 | 119 |
| Terminal-Bench 2.1 | **74.5%** | 67.0% |
| SciCode | **51.8%** | 47.4% |
| 上下文 | 1M | 262K |
| 输出价 | $1.20 | $4.00 |

编码能力基本持平甚至反超。更关键的是：**Kimi 真正的优势在 MCP 工具编排和高吞吐，而本流水线的 Coder 根本不调工具**——它只吞一个计划、吐一个带文件内容的 JSON。用不上的强项，换不来便宜三倍的输出价和四倍的上下文。

### Tester 选 `kimi-k2.7-code`（吞吐优先）

同任务实测吞吐：**k2.7-code 79 tok/s vs k2.6 43 tok/s**，快 1.85 倍，且代码专精。

代价是它**关不掉 thinking**（见下），推理时长不可预测。用 `max_tokens` 上限 + 截断检测兜底，不设硬延迟 SLA。

如果哪天你需要给流水线定硬性延迟上限，就换 `kimi-k2.6`（慢但可控）或 `qwen3.7-plus`（2.7s）。这是个五五开的取舍，取决于你要吞吐还是要可预测性。

## 单价

美元 / 百万 token：

| 模型 | 输入（未命中） | 输入（命中） | 输出 |
| --- | --- | --- | --- |
| `deepseek-flash` | 0.15–0.30（谷/峰） | 0.003–0.006 | 0.60–1.20 |
| `qwen3.8-max` | 1.65–2.00 | — | 4.95–6.00 |
| `kimi-k2.7-code` | 0.95 | 0.19 | 4.00 |
| `qwen3.7-plus`（备选） | 0.32–0.40 | — | 1.28–1.60 |
| `kimi-k2.6`（备选） | 0.95 | 0.19 | 4.00 |

DeepSeek 有峰谷差价（谷时五折，峰时=北京时间工作日 9–12、14–17 点）。Kimi 和 Qwen 的缓存命中能砍输入价，但**输出价没有缓存**——所以成本大头永远是 Coder 和 Tester 的输出量，不是 Reviewer。

**成本归因**：玩具任务里 Coder+Tester 占 61%，Reviewer 只占 28%。别为了省 Reviewer 那点钱去降质量闸门。

## 实测延迟与吞吐

同一任务下：

| 配置 | 耗时 | 吞吐 |
| --- | --- | --- |
| `qwen3.8-max` thinking **开** | 183.6s | — |
| `qwen3.8-max` thinking **关** | 3.6s | — |
| `kimi-k2.7-code` thinking 默认 | 18.3s | 79.0 tok/s |
| `kimi-k2.6` thinking 关 | 37.0s | 42.8 tok/s |
| `kimi-k2.6` thinking 默认 | 72.0s | 46.3 tok/s |

**Reviewer 的 thinking 开关是数量级的延迟差**（183.6s vs 3.6s）。这是 `--fast` 参数存在的意义，也是为什么默认不关它——质量闸门值得那三分钟。

整条流水线端到端约 **156–184 秒**，其中 Tester 占 85% 左右。

## 厂家参数差异

三家虽然都提供 Anthropic Messages 端点，**行为并不一致**。接入层必须按厂家组装请求。

### `thinking` 能否关闭

| 模型 | `{"type":"disabled"}` | 备注 |
| --- | --- | --- |
| `deepseek-flash` | ✅ 接受 | |
| `qwen3.8-max` | ✅ 接受 | |
| `qwen3.7-plus` | ✅ 接受 | |
| `qwen3.8-flash` | ✅ 接受 | |
| `kimi-k2.6` | ✅ 接受 | |
| **`kimi-k2.7-code`** | ❌ **拒绝** | 报 `only type=enabled is allowed for this model`；**省略该字段时仍默认开启** |

`kimi-k2.7-code` 这一条是相对 k2.6 的**功能倒退**——它是三家唯一你想快也快不了的模型。

### `max_tokens` 与 thinking 的关系

**thinking 的 token 计入 `max_tokens` 总额度。**

- 实测 `qwen3.8-max` 在 `max_tokens=4000` 时，thinking 把额度吃光，**正文一个字符都没输出**。
- 实测 `kimi-k2.7-code` 在玩具级任务上 thinking 达 27326 字符，`max_tokens=8000` 时输出被截断，整个测试 JSON 不可用；提到 24000 后正常。

所以：**开启 thinking 的角色必须给足额度**，且必须检测 `stop_reason == "max_tokens"`。当前脚本只打印告警不重试——看到 `!!! 输出被 max_tokens 截断` 说明该轮产物不可用。

### 其他

- 千问的 `temperature` 取值域是 `[0, 2)`，宽于 Anthropic 原生的 `[0, 1]`。
- 千问 Anthropic 端点**没有模型列表接口**，核模型走 `https://dashscope.aliyuncs.com/compatible-mode/v1/models`。
- 千问的 `thinking.budget_tokens` 正被 `output_config.effort` 取代。

## 接入点

| 厂家 | Anthropic 端点 | 环境变量 |
| --- | --- | --- |
| DeepSeek | `https://api.deepseek.com/anthropic` | `DEEPSEEK_API_KEY` |
| 阿里云百炼 | `https://dashscope.aliyuncs.com/apps/anthropic` | `DASHSCOPE_API_KEY` |
| Moonshot | `https://api.moonshot.cn/anthropic`（国内）/ `https://api.moonshot.ai/anthropic`（国际） | `MOONSHOT_API_KEY` |

两点：

- Moonshot 的 base URL **必须与 key 所属区域匹配**（国内 key 配 `.cn`，国际配 `.ai`），错配直接 401。
- 百炼的 Anthropic 路径在 `dashscope.aliyuncs.com` 下**不需要 WorkspaceId**；`{WorkspaceId}.cn-beijing.maas.aliyuncs.com/apps/anthropic` 那个形态是分区域部署才用的。

## 一条方法论教训

评估模型快慢时，**固定任务、比吞吐（tok/s），不要比绝对耗时**。

我犯过这个错：曾拿「k2.7-code 138s vs k2.6 5.2s」判定 k2.7-code 更慢，据此换掉了它。但那两次运行的**工作量根本不同**（输出 181 token vs 6601 token）。固定任务重测后，结论完全反转——k2.7-code 吞吐是 k2.6 的 1.85 倍。

绝对耗时里混着输出长度、thinking 长度、输入长度三个变量，单看它什么都说明不了。
