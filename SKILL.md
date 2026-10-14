---
name: multi-agent-dev
description: 用多 Agent 流水线开发代码——Planner 规划、Coder 生成、Reviewer 交叉审查、Tester 生成测试，四个角色由 DeepSeek / 阿里云千问 / Moonshot Kimi 分别担任。当用户要求"用多 agent 开发"、"跑流水线"、"多模型协作写代码"，或想要一份**经过审查和测试**的代码产出（而不只是随手生成一段），或希望用 DeepSeek/千问/Kimi 而非单一模型完成开发任务时，使用本 skill。即使用户没有说出"流水线""多 agent"这些字眼，只要意图是"要一份被审查过、带测试的代码交付物"，就应当使用它。
---

# 多 Agent 代码开发流水线

把「规划 → 生成 → 审查 → 修复 → 测试」拆成四个角色，交给三家厂商的模型分别担任，跑完直接产出可运行的代码和测试文件。

**核心价值是跨厂家异源**：写代码的和审代码的必须是不同厂家的模型。同一个模型既写又审会共享同一套盲区和偏好，审查会系统性放水——这是 prompt 层面无法弥补的，只能用模型来源的差异来保证。

## 什么时候用它

适用：
- 用户说「用多 agent / 流水线开发 X」「多模型协作写这个」
- 用户要的是**带审查环节和单元测试**的交付物，不是一次性生成
- 需求有一定复杂度，值得先规划再动手

不适用，别硬套：
- **单文件小改动**——直接写更快，跑一遍流水线要 2–5 分钟
- **交互式场景**——用户不可能等 156 秒
- **大型多服务工程（如 Spring Cloud）**——当前设计撑不住，先读 `references/scale-limits.md` 确认适用边界

## 前置条件

三个环境变量都得在，缺哪个脚本会立刻退出并明确指出：

| 变量 | 供谁使用 |
| --- | --- |
| `DEEPSEEK_API_KEY` | Orchestrator / Planner / Coder |
| `DASHSCOPE_API_KEY` | Reviewer（阿里云百炼） |
| `MOONSHOT_API_KEY` | Tester（Moonshot 国内区，`.cn`） |

三家统一走 **Anthropic Messages 协议**（各自的 `/anthropic` 端点），所以只需要一套调用代码。

## 怎么跑

```bash
python scripts/run_pipeline.py "你的开发需求"
```

| 参数 | 作用 |
| --- | --- |
| `"需求文本"` | 要开发什么。省略则跑内置的示例需求 |
| `--fast` | 关掉 Reviewer 的 thinking。单次审查 ~185s → ~4s，但审查变浅，只适合明确的小改动 |
| `--seed` | 用预置的一段有 bug 的代码直接进审查，**必然触发修复回路**。用来验证流水线本身是否正常，不消耗生成阶段的调用 |

产物写到**当前工作目录**下的 `pipeline-out/`（可用环境变量 `PIPELINE_OUT_DIR` 改）。所以要在哪个工程里产出代码，就在那个工程的目录下执行。

## 流水线与模型映射

| 阶段 | 角色 | 模型 | 说明 |
| --- | --- | --- | --- |
| 0 | Orchestrator | `deepseek-flash` | 路由决策，输出下一步该调用谁；不写代码 |
| 1 | Planner | `deepseek-flash` | 拆解需求，产出开发计划与验收标准 |
| 2 | Coder | `deepseek-flash` | 按计划产出完整文件 |
| 3 | Reviewer | `qwen3.8-max` | **跨厂家审查**，输出 `verdict` + `issues` |
| 3.x | Coder | `deepseek-flash` | 按 issues 修复，只输出被修改的文件 |
| 4 | Tester | `kimi-k2.7-code` | 生成单元测试 |

审查不通过时进入修复回路：`needs_fix → Coder 修复 → 复审`，最多 3 轮；3 轮仍不过则终止并如实标注未解决问题，交人工。

三个角色的提示词是独立文件（`prompts/00-orchestrator.md` ~ `04-tester.md`），**要改行为就改提示词，不要改脚本**。脚本只负责调度和协议适配。

## 产出

- `pipeline-out/` 下的代码文件与 `tests/` 下的测试文件
- 控制台逐步打印每个阶段：模型、耗时、token 用量、审查结论、issues 明细

跑完代码是实际可运行的——生成的测试直接 `pytest` 即可（如果环境里装了的话）。

## 已知坑（都已实测踩过，详见 references/model-mapping.md）

这几条不长，但踩上去会浪费你半天：

1. **thinking 的 token 计入 `max_tokens` 总额度。** 额度给小了，thinking 会把正文吃光，一个 JSON 都不输出。所以各角色的额度在脚本里是按实测调的，别随手改小。
2. **`kimi-k2.7-code` 关不掉 thinking。** 它拒绝 `thinking: {"type":"disabled"}`，省略该字段时又默认开启。后果是它的推理时长不可预测，且 `max_tokens` 必须给到 24000 以上否则输出被截断。
3. **三家对同一套参数的接受度不一致。** 不能假设「Anthropic 协议兼容」就等于参数通用，接入层必须按厂家组装请求。
4. **截断只告警不重试。** 脚本会打印 `!!! 输出被 max_tokens 截断`，但不会自动重试——看到这行说明该轮产物不可用，要手动处理。
5. **LLM 审查管不了类型/API 兼容性。** 实测 Reviewer 会用错误的理由（"`__str__` 能保证字符串拼接正常工作"——不能）为破坏性变更放行。这类判断最终要以编译器/类型检查器为准，别指望提示词。

## 参考文档

- `references/model-mapping.md` — 完整的模型选型依据、单价、实测延迟与吞吐、厂家参数差异表
- `references/scale-limits.md` — **投入真实工程前必读**：本设计的适用边界，以及扩展到单服务 / 多服务项目必须先补的三层

## 想调整时

- **换模型**：改 `scripts/run_pipeline.py` 顶部的 `PROVIDERS`，同时同步 `prompts/00-orchestrator.md` 的「模型配置」一节，两边别脱节。
- **调额度 / thinking**：同文件的 `MAX_TOKENS`、`THINKING_OFF`、`THINKING_NO_DISABLE`。
- **改角色行为**：改 `prompts/` 下的提示词——那才是这套流水线的调参面。
