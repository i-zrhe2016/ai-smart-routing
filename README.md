# Codex Smart Routing

面向 Magpie 的 Codex 智能模型路由配置，根据任务复杂度和风险分配模型与推理强度，兼顾成本、质量和返工次数。

## 方案一：Codex 路由表

| 任务 | 模型 | 推理强度 |
| --- | --- | --- |
| 代码阅读、信息提取、文档生成 | GPT-6 Luna | High |
| 简单修改、测试、低风险执行 | GPT-6 Luna | Max |
| 日常功能开发、UI、API（默认） | GPT-6.2 Sol | Medium |
| 需求分析、规划、排障、性能优化 | GPT-6.2 Sol | High |
| 深度审查、复杂重构、反复失败的问题 | GPT-6.2 Sol | XHigh |
| 核心架构、高风险技术决策 | GPT-6.2 Astra | Medium |
| 极复杂工程问题、疑难问题升级 | GPT-6.2 Astra | High |

分类器：`codex/gpt-6.2-sol`。

## 使用方法

1. 按需获取 [Codex](routing.json)、[Claude](claude-routing.json) 或 [DeepSeek + Claude](deepseek-claude-routing.json) 配置。
2. 核对 Magpie 中可用的模型 ID 和推理强度。
3. 在支持 routing-group JSON 的 Magpie 版本中导入配置，测试各类任务的实际路由结果。

## 设计原则

- 优化任务成功完成的总成本，而非仅追求最低 Token 单价。
- 默认建议 Sol Medium；将明确的低风险任务交给 Luna，关键规划和复杂问题使用更强模型。
- 减少不必要的模型切换，避免潜在的缓存失效和重复上下文成本。
- 失败后升级模型，但必须由客户端或用户提供失败上下文；此 JSON 本身不实现自动重试或验证机制。
- 涉及生产环境、权限、安全或不可逆操作时，不应仅依据任务名称降低推理强度。

## 参考资料

- [OpenAI Model selection](https://developers.openai.com/api/docs/guides/model-selection)：推荐从 Sol Medium 开始评估多数 API 工作流，并建议多数 Codex 任务从 Sol 开始。
- [OpenAI GPT-6 Luna](https://developers.openai.com/api/docs/models/gpt-6-luna)：官方 API 支持 `high`、`xhigh` 和 `max` 等推理档位。
- [Reddit: Specialist model workflow](https://www.reddit.com/r/codex/comments/1y37fwm/)：用户分享 Luna Max 执行、Sol 开发与审查、Astra 高风险验收的工作流。
- [Reddit: Model recommendations](https://www.reddit.com/r/codex/comments/1y0cizk/)：用户分享按任务选择模型的经验。
- [Reddit: How to use Astra more efficiently](https://www.reddit.com/r/codex/comments/1wg90ai/)：用户分享规划与执行分离的策略。

社区经验不是严格对照实验，也不代表普遍共识。

## 兼容性

本仓库是配置示例，不是独立路由程序。官方 API 支持的推理档位不等同于 Magpie 实际支持的模型 ID、分类器和导入格式；尚未在实际 Magpie 环境中验证。成本收益及分类器升级效果尚无统一实测数据。

## 方案二：Claude 版本

配置文件：[claude-routing.json](claude-routing.json)。与原 Codex 配置独立，可按需导入。

| 任务 | 模型 | 推理强度 |
| --- | --- | --- |
| 只读、文档 | Haiku 5.5 | Medium |
| 简单修改、测试 | Haiku 5.5 | High |
| 日常开发（首选） | Sonnet 5.5 | Medium |
| 复杂开发、排障 | Sonnet 5.5 | High |
| 规划、架构 | Opus 5.5 | Medium |
| 深度审查、复杂重构 | Opus 5.5 | High |
| 极复杂任务升级 | Fable 5.2 | High |

分类器：`claude/claude-haiku-5-5`。

模型 ID、推理档位和 Fable 可用性均需在实际 Magpie 环境核对；此配置仅通过 JSON 结构检查，未验证实际调用、分类准确率或成本收益。不支持的模型应移除并调整对应规则。参考 [Claude 官方模型文档](https://platform.claude.com/docs/en/models/overview)。

## 方案三：DeepSeek + Claude 混合路由

配置文件：[deepseek-claude-routing.json](deepseek-claude-routing.json)。只使用两个模型，按任务目标而非笼统的复杂度分类：Claude 决策与审查，DeepSeek 执行与信息提取。

| 角色 | 判断边界 | 模型 |
| --- | --- | --- |
| Reader | 只整理已知事实，不修改系统、不做评价或新决策 | DeepSeek V4.1 Flash Low |
| Developer | 目标与处理方案明确，验收标准可确定，低风险 | DeepSeek V4.1 Flash High |
| Architect / Reviewer | 需要确定方案、诊断未知根因、独立审查或评估重大风险 | Claude Sonnet 5.5 High |

分类器：`deepseek/deepseek-flash`。

### 升降级规则

分类器根据当前用户消息重新选择模型，不依赖历史对话、工具结果或上一轮模型。

| 当前指令表达的任务状态 | 目标模型 |
| --- | --- |
| 仅搜索、阅读、提取或整理信息 | Flash Low |
| 只读任务转为开发、修改或执行 | Flash High |
| 方案失败，需要重新分析；出现未知根因或重大风险 | Sonnet High |
| 方案已确定，只需按方案实施 | Flash High |
| 开发完成，只需整理结果 | Flash Low |
| 意图不明确（如仅说“继续”） | Flash High（默认建议） |

优先级：重大风险、未知根因、明确失败或独立决策与审查 > 执行 > 纯信息处理。普通实现细节不需要升级。

例如，先发“设计登录系统方案”选择 Sonnet High，再发“按已确定的方案实现”选择 Flash High；若遇到失败，发“多次修复失败，请重新分析根因”选择 Sonnet High；最后发“整理开发结果”选择 Flash Low。

这些规则表达的是跨用户轮次重新分类的目标，而非自动监控、自动重试或同一轮工具调用期间的模型切换。“继续”“重试”本身不表示失败或方案已经完成。规则优先级和默认档位为分类意图提示，不是独立的硬编码控制流。

### 社区参考

- [Claude + DeepSeek 混合开发](https://www.reddit.com/r/CommandCode/comments/1xw1ev4/how_are_u_using_claude_code_deepseek_v41_flash/)：Claude 拆解任务及审查，DeepSeek 实现。
- [DeepSeek Flash 作为 Claude 的开发子代理](https://www.reddit.com/r/DeepSeek/comments/1xkphiv/deepseek_v41_flash_as_the_coding_worker_under/)：作者报告 Opus Token 消耗降低 55%–63%；该结果涉及特定多代理工作流，不代表本配置实测收益。
- [Flash 与 Sonnet 对比](https://www.reddit.com/r/ClaudeCode/comments/1wu05fx/compared_deepseek_flash_v41_to_sonnet_55/)：任务耗时、返工及计费方式也影响总成本。

不引入 Pro、Opus 或其他模型。失败升级依赖当前用户消息明确表达失败状态，配置本身不自动重试或自动拆分规划与执行。跨供应商路由可能影响缓存命中率，敏感代码和数据须符合两个供应商的数据治理要求。

参考：[Magpie 意图路由](https://usemagpie.ai/docs/zh/intent)、[DeepSeek API 文档](https://api-docs.deepseek.com/)、[Claude 模型概览](https://platform.claude.com/docs/en/models/overview)。导入前核对本地 Magpie 的供应商前缀、模型 ID 和推理档位；实际调用、分类准确率和成本收益尚未验证。
