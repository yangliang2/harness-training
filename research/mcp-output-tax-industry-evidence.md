# MCP 整表返回的输出税：行业论证调研

调研日期：2026-08-23（本文所有 URL 抓取日期均为 2026-08-23）
调研目的：为反范式课 ep02（反模式"工具输出无节制"）换骨后的主案例——"第三方 MCP 整表返回大 JSON（如缺陷查询，上百字段只用几个），不加抽取层"及其解法"消费端 PostToolUse 抽取 hook（matcher `mcp__server__*`）+ 落盘 hook 两条分诊"——补齐行业证据：官方机制口径、公开翻车案例、竞品水位、企业 MCP 质量佐证、Cursor 对应物。

判定口径：
- **常识** = 官方文档/头部工程博客反复论述 + 社区工具生态已成规模
- **有人讲** = 有公开 issue/博客/工具专门讨论，但未形成普遍认知
- **几乎没人讲** = 找不到直接或只有擦边的公开讨论

来源纪律：官方文档/官方工程博客/CHANGELOG 为一手；GitHub issue 为一手（标注是否官方确认）；工程博客二手；营销文只作线索。

---

## Q1 MCP 输出过大的官方机制与讨论密度

**判定：三个机制全部官方文档在册（一手确认，个别细节有修正）；"撞 25k 上限"的讨论密度介于有人讲与常识之间——高频公开抱怨，且多家第三方厂商已把这个报错写进自家官方文档。**

官方机制口径（Claude Code 官方 MCP 文档 "MCP output limits and warnings" 一节，一手）：

- **10k 警告**：确认。"Claude Code displays a warning when any MCP tool output exceeds 10,000 tokens"。注意细节：警告阈值**固定不可配**（"the warning threshold is fixed"）。
- **25k 默认上限**：确认。"the default maximum is 25,000 tokens"，通过 `MAX_MCP_OUTPUT_TOKENS` 环境变量调高（示例 `MAX_MCP_OUTPUT_TOKENS=50000`）。不是传闻，是文档明写。
- **超长结果落盘替换为文件引用**：确认。"Without the annotation, results that exceed the default threshold are persisted to disk and replaced with a file reference in the conversation."。server 作者可用 `_meta["anthropic/maxResultSizeChars"]` 为单个工具把阈值抬到 500,000 字符硬顶；该注解对文本内容独立于 `MAX_MCP_OUTPUT_TOKENS` 生效，图片内容不适用。诚实标注：**默认落盘阈值的具体数值文档未给出**（只说 "default persist-to-disk threshold"）。https://code.claude.com/docs/en/mcp

机制的来路与周边（issue 一手）：

- 落盘机制上线前，社区先要过：#20342（2026-01-23）请求"大 MCP 响应自动写文件、返回文件引用"，动机引用 60,000+ tokens 的响应，closed as duplicate——现在文档已含 persist-to-disk 行为，即该需求最终被官方吸收。https://github.com/anthropics/claude-code/issues/20342
- 文档是被社区催出来的：#42869（[DOCS] MCP docs missing `_meta["anthropic/maxResultSizeChars"]`，closed）、#47631（[DOCS] MCP output warning docs lack format-specific recovery recipes，closed）。https://github.com/anthropics/claude-code/issues/42869
- 周边坑（可作口播弹药）：#75267 记录 subagent 内 MCP 输出有**未文档化的 3k 上限**（open）；#78478 记录 Claude Desktop 会丢 `anthropic/maxResultSizeChars` 注解（CLI 尊重、Desktop 忽略，open）；#85169 请求 MCP tool-result 从上下文中可调逐出（open，2026-08）。https://github.com/anthropics/claude-code/issues/75267 ；https://github.com/anthropics/claude-code/issues/78478

讨论密度证据（撞上限是普遍现象）：

- **第三方厂商把这个报错写进自家官方文档**：Figma 官方开发者文档的 MCP 客户端故障页原样引用 `Error: MCP tool "get_design_context" response (351378 tokens) exceeds maximum allowed tokens (25000)`，并教用户调 `MAX_MCP_OUTPUT_TOKENS`。https://developers.figma.com/docs/figma-mcp-server/mcp-clients-issues
- SaaS 厂商专门写帮助中心文章教用户绕过：xpoz 帮助中心《Claude Code: MCP tool exceeds maximum allowed tokens (25000)》。https://help.xpoz.ai/en/articles/12681842-claude-code-mcp-tool-exceeds-maximum-allowed-tokens-25000
- 上游项目收到用户报障：MUI 仓库 #46778（useMuiDocs 工具对 Claude Code 返回过多 token）。https://github.com/mui/material-ui/issues/46778
- Claude Code 仓库自身：#9152（MCP 图片响应超 25k）。https://github.com/anthropics/claude-code/issues/9152

## Q2 整表返回的公开翻车/抱怨案例

**判定：有人讲，案例多且带硬数字。四大类主流 server（GitHub、Atlassian、数据库、设计工具）全部有官方仓库一手 issue 可引用，其中两例是 server 官方自己修/自己认。**

- **GitHub 官方 MCP server**：#2383（2026-04-24）——`list_project_items` 返回 50 条约 **400KB / ~100K tokens（约半个典型上下文窗口）**；每条约 8KB+，其中塞满完整 issue/PR 正文、~2KB 的 repository URL 模板对象、每个 ~500B 的完整 user 对象，而实际需要的仅约 **200-300 字节/条**。GitHub 官方指派工程师、经 PR #2563 修复关闭——官方承认"整表返回"是缺陷并按"默认返回最小字段"模式修，且 issue 中点名这是延续 #2023/#2025/#2028 的既定瘦身模式（说明官方 server 也是一路踩坑一路裁）。https://github.com/github/github-mcp-server/issues/2383
- **Atlassian 官方 MCP server**：#17（2025-11-12）"MCP tool responses too verbose - breaks context window for users with many work items"——有 20-50+ 个 assigned items 的用户查询 2-3 次即撑爆上下文，典型查询返回 30-50 个 issue 的臃肿响应；请求默认只回 ID/key/summary/status/assignee。**至今 open、无官方回应**。https://github.com/atlassian/atlassian-mcp-server/issues/17 。另有 Atlassian 官方社区帖引用实际报错：`MCP tool 'getJiraIssue' response (29544 tokens) exceeds maximum allowed tokens (25000)`——**单查一个 Jira issue 就撞 25k 上限**，与 ep02 主案例（缺陷查询整表返回）完全同构。https://community.atlassian.com/forums/Atlassian-Remote-MCP-Server/How-to-limit-responses/m-p/3130657
- **数据库/后端类**：supabase-community/supabase-mcp #124——`get_advisors` 返回 **28,254 tokens** 超 25,000 上限，"正常规模的真实项目"即不可用，请求分页/按严重度过滤。https://github.com/supabase-community/supabase-mcp/issues/124
- **设计工具类**：Figma 官方文档自曝单次 `get_design_context` 返回 **351,378 tokens**（见 Q1 链接）——目前找到的最大单次返回数字，且出自厂商官方文档而非用户抱怨。
- 生态反应佐证"官方 server 整表返回"已催生替代品：github-mcp-lw（轻量 GitHub MCP，只返回必要字段，自称响应比完整 GitHub API 响应小 90%+；自述数字，未独立验证）。https://glama.ai/mcp/servers/@wipiano/github-mcp-lw
- 话语层佐证：StackOne 的四方案对比文明确区分 schema bloat 与 response bloat，并指出"**Response bloat often consumes more context than schema bloat but receives less attention**"——即"返回体积"恰是被讨论不足的那一半。https://www.stackone.com/blog/mcp-token-optimization/

## Q3 消费端抽取/过滤方案的竞品水位（关键）

**判定：机制层"有人讲"（官方文档在册 + 约三篇低可见度文章），但"对第三方 MCP 整表返回做字段抽取的具体实践/配方"几乎没人讲；"抽取 + 落盘两条分诊"的完整结构未找到任何公开先例。主流优化话语全部集中在 server 侧和 gateway 侧，消费端 hook 这条路基本空着。**

官方机制（一手，重要）：

- Claude Code hooks 官方文档 PostToolUse decision control 表中明列两个字段：`updatedToolOutput`（"Replaces the tool's output with the provided value before it is sent to Claude"，全工具适用）和 `updatedMCPToolOutput`（"Replaces the output for MCP tools only. Prefer `updatedToolOutput`, which works for all tools"）。且明写 "**MCP tool output is passed through without schema validation**"——对比内建工具必须匹配输出对象形状（Bash 需 `{stdout, stderr, interrupted, isImage}`），**MCP 工具反而是输出改写最顺的那一类**。文档示例只有替换 Bash 输出一例，**没有任何针对第三方 MCP 的字段抽取示例**。https://code.claude.com/docs/en/hooks
- 历史脉络（issue 一手）：`updatedMCPToolOutput` 是**先有的 MCP 专用能力**，内建工具的 `updatedToolOutput` 反而是社区后来要来的——#36843（2026-03-20，请求 built-in 支持）、#32105（2026-03-08，同类请求，点名"context budget recovery"）、#54161（2026-04-28，[DOCS] 文档滞后只写 MCP-only 字段）。即：对 MCP 输出的消费端改写是这套机制里**最老、最稳的路径**，却最没人用。https://github.com/anthropics/claude-code/issues/36843 ；https://github.com/anthropics/claude-code/issues/32105
- 已知 caveat（需在课程中避雷）：#47859——Agent SDK 的 **callback hook** 里返回 `updatedMCPToolOutput` 被静默丢弃（closed）；#24788——PostToolUse 对 MCP 工具调用返回的 `additionalContext` 不显示（closed as not planned，无官方确认）。命令行 hook 路径未见同类失效报告。https://github.com/anthropics/claude-code/issues/47859 ；https://github.com/anthropics/claude-code/issues/24788

公开讲过"消费端改写 MCP 输出"的全部材料（能找到的仅此三篇，全是 2026 年 3-4 月后的低可见度文章）：

- Claude Code for Non-Coders（Substack，2026-04-21）《MCP Is the Access Layer. Hooks Are the Enforcement Layer.》——**最接近的先行者**。原文明确写 "verbose API payloads trimmed to the fields the model actually uses"、"`updatedMCPToolOutput` replaces the tool output entirely. The model never sees the raw response."，并给出返回 `updatedMCPToolOutput` 的 Python 归一化示例。但其示例是时间戳/状态码/字段名**归一化**，matcher 用的是 `Write|Edit` 而非 `mcp__` 前缀，没有"针对某个具名第三方 server 做字段抽取"的配方。https://claudecodefornoncoders.substack.com/p/mcp-is-the-access-layer-hooks-are
- dev.to speedy_devv（2026-04-24）《MCP Tool Hooks in Claude Code》——提及 `updatedMCPToolOutput` 字段能力（"replaces what Claude sees as the tool's output before it enters the conversation"），定位是介绍 mcp_tool 类型 hook，非抽取实践。https://dev.to/speedy_devv/mcp-tool-hooks-in-claude-code-24f6
- buildingbetter.tech《I Read the Claude Code Source Code》——源码考古文提及该字段（当时文档未写全）。https://buildingbetter.tech/p/i-read-the-claude-code-source-code

主流教程/对比文**不讲**这条路（反面证据）：

- Paul Schick 的 hooks 教程（2026-02-28）明说 PostToolUse "The action already happened, so you can't undo it"，只讲事后 react（格式化/lint），完全未提输出改写字段。https://paul-schick.com/posts/claude-code-hooks-pretooluse-posttooluse/
- StackOne《MCP Token Optimization: 4 Approaches Compared》（2026-03-31）四条路线：schema 压缩、search-first discovery、response filtering、code execution——其中 response filtering 指 **server/gateway 侧**按字段请求（约省 95%/次），**通篇没有 consumer-side hook 这条路线**。https://www.stackone.com/blog/mcp-token-optimization/
- Scott Spence 的 MCP 上下文优化文只做 schema 侧（合并工具定义、选择性启用 server），不涉及响应侧。https://scottspence.com/posts/optimising-mcp-server-context-usage-in-claude-code

竞品全在别的位置（确认"消费端"空档）：

- **Gateway/代理侧**：mcp-rtk（代理压缩工具响应，自称 8 级过滤管道省 60-90%）；agentgateway #1349（响应 JSON→紧凑格式压缩提案）；mcp-filter / mcp-tool-filter（只做工具面裁剪，不动响应）；Atlassian 官方开源 mcp-compressor（schema 压缩代理）。https://github.com/ThomasTartrau/mcp-rtk ；https://github.com/agentgateway/agentgateway/issues/1349 ；https://www.atlassian.com/blog/developer/mcp-compression-preventing-tool-bloat-in-ai-agents/amp
- **context-mode**：PreToolUse 拦截的是**内建工具**（Bash/Read/Grep/WebFetch/Agent）改道沙箱执行 + FTS5 索引，不是对第三方 MCP 返回做字段抽取。https://github.com/mksglu/context-mode
- **code execution 路线**（Anthropic 官方，见 Q4）：架构级替代，让中间结果根本不过上下文——解决同一问题但要求改接入方式，不是"现有 MCP server 不动、消费端加一层"的低成本路径。

## Q4 企业内 MCP 质量的佐证

**判定：有人讲——"MCP 生态大面积是 API 薄包装、整表返回"有官方工程博客 + 学术研究两条硬证据，spec 层也有输出体积的公开讨论；但"企业内自建 MCP 质量差"这一具体断言【未找到】直接公开数据（企业内部不可见），只能靠推理链口播。**

- **Anthropic 官方工程博客《Code execution with MCP》**（一手，最硬）：明确批评直连 MCP 时"每个中间结果都必须过模型上下文"——Google Drive→Salesforce 例子里全文转录在上下文出现两次，2 小时录音"可能额外多出 50,000 tokens"；改走 code execution 后同任务 **150,000 tokens → 2,000 tokens，省 98.7%**；并指出大文档在工具间复制时"模型更容易抄错"。这是官方对"整表结果灌上下文"的正面开炮。https://www.anthropic.com/engineering/code-execution-with-mcp
- **学术一手**：arXiv 2507.16044《From REST to MCP: An Empirical Study of API Wrapping and Automated Server Generation for LLM Agents》——**88.6% 的 MCP server 完全或部分是 REST API 包装，92% 的工具实现为 bare API wrappers（裸包装）**。直接支撑"很多 MCP 是 API 薄包装"的断言。https://arxiv.org/abs/2507.16044
- 相邻学术：Queen's University 团队对 1,899 个开源 MCP server 的健康度/安全性/可维护性研究（arXiv 2506.13538）。注意：二手 Medium 文引用的"API-wrapper 模式平均 5.3x 工具调用次数"**无法在论文摘要中证实，禁止引用该数字**。https://arxiv.org/abs/2506.13538
- **MCP spec 层的输出体积讨论**（modelcontextprotocol org，一手）：Discussion #2211《Response size limit for MCP responses to prevent context overflow in AI Agents》——指出 server 常整表返回、多数客户端把整个响应直接灌进模型上下文；Discussion #799 提案给 tool request/response 加分页。现状：**spec 的 pagination 只覆盖 list 操作（listTools 等），工具调用结果无任何协议级体积约束**。https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/2211 ；https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/799 ；https://modelcontextprotocol.io/specification/2025-11-25/server/utilities/pagination
- 二手话语佐证（丰富但只作线索）：StackOne "returning 50 fields when the model needs 3 burns money"；Speakeasy 汇总"MCP servers as thin API wrappers"是最常见批评之一；apideck/kloia/thenewstack 等一批"Your MCP server is eating your context"体裁文章。https://www.speakeasy.com/mcp/mcp-for-skeptics/common-criticisms/
- **企业自建 MCP 的直接数据**：【未找到】。可用的推理链：92% 裸包装（学术）+ GitHub/Atlassian/Supabase/Figma 四家**官方** server 全部有整表返回撞限记录（Q2）→ "资源投入最多的官方 server 尚且如此，企业内薄包装只会更糙"——此推理可口播，不可上卡当数据。

## Q5 Cursor 侧对应物

**判定：可写"照抄解法"栏。Cursor hooks 官方文档在册，`postToolUse` 有 MCP 专用输出改写字段 `updated_mcp_tool_output`，与 Claude Code 解法同构；但有一个官方已确认的坑（payload 无 server 身份字段）需注明。**

- **官方文档一手**：Cursor hooks 配置于 `~/.cursor/hooks.json` / 项目 `.cursor/hooks.json`（另有 team/enterprise 层）；事件含 `preToolUse` / `postToolUse` / `postToolUseFailure`、`beforeShellExecution` / `afterShellExecution`、`beforeMCPExecution` / `afterMCPExecution` 等。`postToolUse` 输出字段 `updated_mcp_tool_output` 原文："**For MCP tools only: replaces the tool output seen by the model**"。https://cursor.com/docs/agent/hooks
- **与 Claude Code 的协议兼容是官方口径**："Cursor supports loading hooks from third-party tools like Claude Code"；exit code 2 阻断行为"matches Claude Code behavior for compatibility"。
- **关键差异**（课程需注明，不能说"完全照抄"）：
  1. 字段名与形态：`updated_mcp_tool_output`（snake_case）vs Claude Code 的 `updatedMCPToolOutput`/`updatedToolOutput`。
  2. 覆盖面相反：Cursor 的 postToolUse **只能改写 MCP 工具输出，不能改写内建工具输出**；Claude Code 的 `updatedToolOutput` 全工具适用。对 ep02 主案例（MCP 抽取）恰好够用。
  3. `afterMCPExecution` 是纯观察事件——能拿到完整 JSON 结果但**返回值被丢弃，不能改写**；能改写的 `postToolUse` payload 里却**没有 MCP server 身份字段**。这个"知道来源的不能动手、能动手的不知道来源"的错位由社区实测报告（forum 2026-08-02），**Cursor 员工 Mohit Jain 2026-08-05 官方确认**并认可加字段方案，无实现时间线。实操影响：按 server 分策略只能靠 tool 名匹配兜底。https://forum.cursor.com/t/aftermcpexecution-return-value-is-discarded-and-posttooluse-carries-no-mcp-server-identity/167261
  4. Cloud agents 不支持 `beforeMCPExecution`/`afterMCPExecution`（`postToolUse` 支持）。
- 落盘分诊的另一半：hook 本身是任意命令，写文件无障碍；但 Cursor 侧**没有** Claude Code 那套"超长 MCP 结果自动落盘换文件引用"的 harness 内建机制——落盘完全靠 hook 自己做。

---

## 对 ep02 的启示

**① 信息差声明的最终判定与措辞。** 不能说"没人讲过用 hook 改 MCP 输出"——字段在 Claude Code 和 Cursor 官方文档都在册，且至少三篇低可见度文章（Substack/dev.to/源码考古）提过机制，其中 Substack 那篇连"裁到模型实际用的字段"的话都说了。可以站住的声明是位置差 + 配方差："**全行业教你在 server 侧改（裁字段/分页）或在中间加 gateway（压缩代理），官方给的第四条路——消费端 hook 抽取——文档里写了字段名，却几乎没人真的走：找不到一篇对具名第三方 server（GitHub/Jira）做字段抽取的公开配方，'抽取 + 落盘两条分诊'的结构更是零先例。**"额外可用的反讽：`updatedMCPToolOutput` 是这套机制里最老的能力（内建工具支持反而是后来社区要来的），且 MCP 输出改写不校验 schema、比改 Bash 输出更顺——最好走的路最没人走。

**② 犯病率判断的佐证形态。** "无意识撞上"有硬证据链可上卡：四大官方 server 全部有整表返回撞限的一手 issue（Q2，带 29,544 / 28,254 / ~100K / 351,378 tokens 四个数字）；Figma 和 xpoz 把报错写成官方帮助文档说明撞的人足够多。"企业自建 MCP 质量更差"无直接公开数据——按计划只作口播观点，推理链用"92% 裸包装（arXiv）+ 官方 server 尚且如此"，不给编造数字留口子。

**③ 行业证据段建议引用清单（按证据强度排序）。**
1. Claude Code 官方 MCP 文档（10k 警告/25k 上限/落盘换引用，三机制一手）——https://code.claude.com/docs/en/mcp
2. Anthropic《Code execution with MCP》（150,000→2,000 tokens、98.7%、"每个中间结果都过模型"）——官方工程博客
3. github/github-mcp-server #2383（50 条 ≈400KB/~100K tokens vs 实需 200-300B/条；官方修复）——官方仓库一手 + 官方行动
4. atlassian/atlassian-mcp-server #17（20-50 items 用户 2-3 次查询撞墙；open 无回应）+ Atlassian 社区帖 getJiraIssue 29,544 tokens——与主案例同构度最高
5. arXiv 2507.16044（88.6% REST-backed、92% bare wrappers）——学术一手
6. Cursor hooks 官方文档（`updated_mcp_tool_output` 逐字口径）——Cursor 栏依据
7. supabase-mcp #124 / Figma 官方文档 351,378 tokens——数字弹药补充
8. MCP spec Discussion #2211/#799（协议层无响应体积约束、有人在要）——"这不是谁的 bug，是生态结构性空档"的论据

**④ Cursor 对应栏：可写"照抄解法"。** 事件对应（PostToolUse ↔ postToolUse）、改写字段对应（`updatedMCPToolOutput` ↔ `updated_mcp_tool_output`）、协议兼容是 Cursor 官方口径。必须注明三点：字段名/大小写不同；Cursor 只能改 MCP 输出（对本案例恰好够）；payload 缺 server 身份字段（官方已确认的坑），matcher 只能靠 tool 名兜底；落盘无 harness 内建机制，hook 自己写文件。
