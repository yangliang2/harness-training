# 《改掉这个习惯》课程工作台

> 系列名：《改掉这个习惯》· AI 使用反模式八讲（每集 ≤5 分钟）
> 用法：讨论某一集 = 读本 README + 对应 epXX.md。每集设计档自包含（既定事实 / 待定问题 / 讨论记录）。
> 工作流：逐集讨论 → 定一条改一条 → 状态翻"定稿" → commit。总大纲（背景）见 `antipattern-course-outline.md`；候选与出局记录见 `antipattern-candidates.md`。

## 集序与状态

| 集 | 集名 | 反模式 | 状态 | 调研报告 | 实测证据 |
|---|---|---|---|---|---|
| 1 | 让 AI 立刻变聪明 | 配置只加不减 | 待讨论 | research/mcp-context-tax-tool-search-doctor.md | spike/verify/ep01-mcp-tax/ |
| 2 | 让 AI 听得清 | 工具输出无节制 | 待讨论 | research/tool-output-industry-consensus.md | spike/verify/ep02-tool-output/ + ep02-truncation/ |
| 3 | 让 AI 不记错事 | 文档坟场 | 待讨论 | research/doc-graveyard-industry-scan.md | spike/verify/ep03-doc-graveyard/ |
| 4 | 让 AI 一学就会 | 逆分布造轮子 | 待讨论 | research/out-of-distribution-tooling.md | spike/verify/ep04-gh-vs-mcp/ + ep04-multi-mcp/ |
| 5 | 让 AI 下午不变笨 | 长会话当仓库 | 待讨论 | research/compact-loss-handoff-statusline.md | spike/verify/ep05-compact/ |
| 6 | 让 AI 不走神 | 一个会话炒三盘菜 | 待讨论 | research/context-pollution-session-branching.md | spike/verify/ep06-ep07/（btw-test） |
| 7 | 让 AI 少浪费钱 | 僵尸会话复活 | 待讨论 | research/llm-cache-strategies.md + zombie-session-revival-awareness.md | spike/verify/ep06-ep07/（cache-test） |
| 8 | 让 AI 说到做到 | 空头支票 | 待讨论 | research/ai-fake-done-verification-awareness.md | spike/verify/ep08-hooks-e2e/ + spike/anti-status-smuggle/ |

## 每集设计档模板（四区）

1. **既定事实**：反模式定义 / 原理一句 / 实测结论 / 调研三档（水文·值钱·威胁）——这些是地基，讨论不推翻，除非有新实测。
2. **待定问题**：靶心 / demo 形态 / 干货与交付物 / 诚实边界——逐条讨论，定一条落一条。
3. **讨论记录**：追加式决议日志（日期 + 决议 + 理由）。
4. **交付物清单**：本集最终要带走的文件/配置/卡片。

## 纪律（从课程设计过程沉淀）

- 状态变更必须显式：定稿与否以用户直接回答是/否为准，应答不算授权（"好"≠确认）。
- 标题党不吹牛：集名只给效果；数字开拍实拍对上才算数。
- 每集必须讲诚实边界。
- 引用版本/功能开拍当月复核（产品快速迭代期）。
