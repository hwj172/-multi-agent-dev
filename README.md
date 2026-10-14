# 多 Agent 代码开发流水线

把「规划 → 生成 → 审查 → 修复 → 测试」拆成四个角色，交给**三家不同厂商**的模型分别担任，跑完直接产出可运行的代码和测试文件。

同时这也是一个 Claude Code Skill——放在 `~/.claude/skills/` 下即可被 Claude 自动调用。

## 核心设计：跨厂家异源

**写代码的和审代码的必须是不同厂家的模型。**

同一个模型既写又审，会共享同一套盲区和偏好，审查在结构上就会放水——这是 prompt 怎么写都补不了的，只能用模型来源的差异来保证。这是本流水线唯一的硬约束，其余设计都可商量。

## 快速开始

```bash
export DEEPSEEK_API_KEY=...    # Orchestrator / Planner / Coder
export DASHSCOPE_API_KEY=...   # Reviewer（阿里云百炼）
export MOONSHOT_API_KEY=...    # Tester（Moonshot）

python scripts/run_pipeline.py "实现一个带重试的 HTTP 客户端"
```

三个环境变量缺任一，脚本会在启动时立即退出并指出缺哪个。**仅用标准库，无第三方依赖**（Python 3.7+）。

### 参数

| 参数 | 作用 |
| --- | --- |
| `"需求文本"` | 要开发什么。省略则跑内置示例需求 |
| `--fast` | 关掉 Reviewer 的 thinking。单次审查 ~185s → ~4s，但审查变浅，只适合明确的小改动 |
| `--seed` | 用预置的一段有 bug 的代码直接进审查，**必然触发修复回路**。用来验证流水线本身是否正常，不消耗生成阶段的调用 |

产物写到**当前工作目录**下的 `pipeline-out/`（可用环境变量 `PIPELINE_OUT_DIR` 改）。要在哪个工程里产出代码，就在那个工程目录下执行。

## 流水线与模型映射

| 阶段 | 角色 | 模型 | 说明 |
| --- | --- | --- | --- |
| 0 | Orchestrator | `deepseek-flash` | 路由决策，输出下一步该调用谁；不写代码 |
| 1 | Planner | `deepseek-flash` | 拆解需求，产出开发计划与验收标准 |
| 2 | Coder | `deepseek-flash` | 按计划产出完整文件 |
| 3 | Reviewer | `qwen3.8-max` | **跨厂家审查**，输出 `verdict` + `issues` |
| 3.x | Coder | `deepseek-flash` | 按 issues 修复，只输出被修改的文件 |
| 4 | Tester | `kimi-k2.7-code` | 生成单元测试 |

审查不通过时进入修复回路 `needs_fix → Coder 修复 → 复审`，最多 3 轮；3 轮仍不过则终止，**如实标注未解决问题交人工**，不会假装通过。

三家统一走 **Anthropic Messages 协议**（各自的 `/anthropic` 端点），所以只需要一套调用代码。

| 厂家 | Anthropic 端点 |
| --- | --- |
| DeepSeek | `https://api.deepseek.com/anthropic` |
| 阿里云百炼 | `https://dashscope.aliyuncs.com/apps/anthropic` |
| Moonshot | `https://api.moonshot.cn/anthropic`（国内）/ `https://api.moonshot.ai/anthropic`（国际） |

Moonshot 的 base URL 必须与 key 所属区域匹配，错配直接 401。

## 目录结构

```
SKILL.md                      Claude Code Skill 入口
README.md
prompts/                      各角色系统提示词 —— 这才是调参面
  00-orchestrator.md          流程定义 + 模型配置，改行为的起点
  01-planner.md 02-coder.md 03-reviewer.md 04-tester.md
references/
  model-mapping.md            选型依据、单价、实测延迟与吞吐、厂家参数差异
  scale-limits.md             适用边界 —— 投入真实工程前必读
scripts/run_pipeline.py       调度器：路由、修复回路、协议适配
templates/                    各角色的输出模板
```

## 已知坑

这几条都实测踩过，踩上去会浪费你半天。详见 `references/model-mapping.md`。

1. **thinking 的 token 计入 `max_tokens` 总额度。** 额度给小了，thinking 会把正文吃光，一个 JSON 都不输出（实测 `max_tokens=4000` 全被 thinking 吃掉）。所以各角色额度在脚本里是按实测调的，别随手改小。
2. **`kimi-k2.7-code` 关不掉 thinking。** 它拒绝 `thinking: {"type":"disabled"}`，省略该字段时又默认开启。后果是推理时长不可预测，且 `max_tokens` 需给到 24000 以上否则输出被截断。
3. **「Anthropic 协议兼容」≠「参数通用」。** 三家对同一套参数的接受度不一致，接入层必须按厂家组装请求。
4. **截断只告警不重试。** 脚本会打印 `!!! 输出被 max_tokens 截断`，但不会自动重试——看到这行说明该轮产物不可用。
5. **LLM 审查管不了类型/API 兼容性。** 实测 Reviewer 会用错误的理由（「`__str__` 能保证字符串拼接正常工作」——实测不能，`"a" + obj` 抛 `TypeError`）为一处真实的破坏性变更放行。这类判断最终要以编译器/类型检查器为准，别指望提示词。
6. **修复那一轮无人审查。** Coder 修 A 时顺手弄坏 B，只要 A 被修好、B 不在上次 issues 清单里，就会一路绿灯。接入编译器之前，修复轮次产物仍需人工抽查。

## 适用边界

当前设计是「**单文件、单轮、无检索**」。

| 阶段 | 场景 | 是否可用 | 需补 |
| --- | --- | --- | --- |
| 当前 | 单文件 / 小任务 | ✅ 可用 | — |
| 中期 | 单服务多文件 | ⚠️ 勉强 | 上下文检索层、diff 输出 |
| 大项目 | 多服务（Spring Cloud 等） | ❌ **不可用** | 三层全补 |

不适用、别硬套的场景：

- **单文件小改动** —— 直接写更快，跑一遍流水线要 2–5 分钟
- **交互式场景** —— 用户不可能等 156 秒
- **大型多服务工程** —— 先读 `references/scale-limits.md` 确认边界

扩展方向需要补的三层（上下文选择 / diff 输出 / 编译验证）在 `references/scale-limits.md` 里有逐层说明和落地顺序建议。

## 成本

玩具级任务实测约 **$0.21/任务**。按单服务级别估算（约 30 万 token 输入、Reviewer 3 轮）约 **$2–5/任务**（估算值，未实测）。

成本大头是 Coder 和 Tester 的输出量（玩具任务里占 61%），Reviewer 只占 28%——别为了省 Reviewer 那点钱去降质量闸门。

## 想调整时

- **换模型** —— 改 `scripts/run_pipeline.py` 顶部的 `PROVIDERS`，同时同步 `prompts/00-orchestrator.md` 的「模型配置」一节，两边别脱节。
- **调额度 / thinking** —— 同文件的 `MAX_TOKENS`、`THINKING_OFF`、`THINKING_NO_DISABLE`。
- **改角色行为** —— 改 `prompts/` 下的提示词。**那才是这套流水线的调参面，不要改脚本**；脚本只负责调度和协议适配。

## License

未指定。
