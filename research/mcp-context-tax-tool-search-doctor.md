# MCP 上下文税、Tool Search 与 /doctor：行业认知调研

调研日期：2026-08-20（所有 URL 均于当日抓取/核验）。供 ep1「配置只加不减：MCP/skill/plugin 装太多吃掉上下文」一集使用。课程既定解法：/doctor 体检 + 闲置就删。

来源分级：官方文档/官方工程博客为一手；GitHub issue 为一手（用户报告，注意甄别是否被官方确认）；工程博客为二手；营销向内容（AgentPMT、Albato、puppyone 等带货文）只作线索，数字引用需标注出处性质。

---

## 问题 1：这个反模式是行业常识吗？

**判定：【常识】——有约定俗成的名字，讨论密度高，且被 Anthropic 官方亲自背书。**

- 通用名称已收敛：**"context bloat" / "MCP token tax" / "tool (over)load"**。MCP 官方组织仓库里就有以此为题的讨论：[modelcontextprotocol Discussions #2036 "handling tool bloat - hundreds of tools"](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/2036)（2026-08-20 抓取；70→400 工具的延迟与 token 成本抱怨）和 [python-sdk issue #2619 "Context bloat - the bottle neck of MCP"](https://github.com/modelcontextprotocol/python-sdk/issues/2619)（2026-08-20 抓取；指出 tools/list 全量注入导致延迟、成本、工具选择质量三重退化，且 spec 没有分组/部分加载原语）。
- **最有力的一手来源是 Anthropic 自己的工程博客**：[Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp)（2025-11-04 发布，2026-08-20 抓取）。原话可引："as the number of connected tools grows, loading all tool definitions upfront and passing intermediate results through the context window slows down agents and increases costs"；标杆数字 **150,000 → 2,000 tokens，省 98.7%**。官方博客承认问题 = 该反模式已是行业共识的铁证。Cloudflare 同期发布同构方案 "Code Mode"（上述 Anthropic 博文内提及）。
- 二手讨论密度（2025-12 至 2026-08 持续不断）：[paddo.dev "Claude Code's Hidden MCP Flag: 32k Tokens Back"](https://paddo.dev/blog/claude-code-hidden-mcp-flag/)、[gopubby "MCP Tools Are Eating 82% of Your Context Window"](https://ai.gopubby.com/mcp-tools-are-eating-82-of-your-context-window-the-10-minute-fix-for-claude-code-1619733d00dc)、[getunblocked "MCP Tool Overload: A Measured Guide"](https://getunblocked.com/blog/mcp-tool-overload/)（均 2026-08-20 抓取）。
- 注意：营销向来源（[AgentPMT "The MCP Bloat Tax"](https://www.agentpmt.com/articles/thousands-of-mcp-tools-zero-context-left-the-bloat-tax-breaking-ai-agents)、[puppyone "Hidden MCP Token Tax"](https://www.puppyone.ai/en/blog/hidden-mcp-token-tax)、[Albato](https://albato.com/blog/publications/embedded-mcp-context-bloat-hallucinations)）中流传的"3 个 server 吃掉 143k/200k（72%）"等数字出自 AgentPMT 自家测量，带货性质明显，**课程引用具体数字应优先用 Anthropic 官方的 150k→2k，营销数字最多作"社区流传口径"并标注**。

## 问题 2：主流解法是什么？观众大概率已经会了吗？

**判定：【有人讲，但观众大概率只知其一】——"少装点"是常识；官方机制层的解法（Tool Search、code execution、/mcp toggle 语义）多数人一知半解。**

社区现存解法谱系（按被讨论频率）：

1. **官方机制：Tool Search（延迟加载）**。2026-01-14 由 Anthropic 工程师 Thariq Shihipar 在 X 上宣布进入 Claude Code（announcement 原文转录见 [issue #18298 首条评论](https://github.com/anthropics/claude-code/issues/18298)）：MCP 工具描述超过上下文 10% 时自动启用，工具改由搜索按需加载。官方文档现状见问题 5。
2. **官方机制：code execution / progressive disclosure**。Anthropic 工程博客（同上）把工具呈现为文件树/代码 API，按需 import；这是面向 agent 开发者的方案，普通 Claude Code 用户用不上。
3. **手动节食**：`/mcp` 面板禁用 server、`/context` 查看占用、`ENABLE_TOOL_SEARCH` 环境变量强制开启。gopubby 一文标题即"10 分钟修复"，说明手动节流已是社区教程的常见题材。
4. **第三方网关/代理**：Docker MCP Gateway（宣称 95% 削减）、各类 MCP gateway/聚合器——生态级方案，带货文密集。

观众认知评估：知道"MCP 装多了吃 token"的人已经很多；但"Tool Search 默认开着所以不用管"是典型的半吊子认知（见问题 4）。手动 /mcp disable 的持久化语义（per-project 存 `~/.claude.json` 的 `disabledMcpServers`，配置不丢）在官方文档里（[code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp)，2026-08-20 抓取）但社区教程很少讲透。

## 问题 3：有没有公开翻车案例/数据可引用？

**判定：【有人讲】——翻车证据充足且层次分明。**

- **官方数据**：150k→2k tokens（98.7%），Anthropic 工程博客，最安全的引用。
- **GitHub issue 翻车链（用户实测，Tool Search 失灵）**：
  - [#18298](https://github.com/anthropics/claude-code/issues/18298)（2026-01-15 开，2026-01-18 被 bot 以重复关闭）：v2.1.7，MCP 工具占 14.1%（28.1k tokens）超过 10% 阈值却未触发延迟加载。官方人员 bhosmer-ant 在上游重复 issue 中回复确认"超过 10% 应自动启用，无需手动设置"，并建议升级到 2.1.14+——但报告者随后回复 2.1.14 仍复现（见 [#18263 评论](https://github.com/anthropics/claude-code/issues/18263)）。
  - [#47645](https://github.com/anthropics/claude-code/issues/47645)（2026-04-13 开，2026-04-17 被 bot 以重复关闭）：v2.1.105，**21 个 MCP server 在会话启动时全部急切连接、注入完整 schema（约 30k tokens / 15%）**；debug 日志显示 `ENABLE_TOOL_SEARCH` 读到了但 `result=false`，即使显式 `=true` 也不生效。这是最戏剧性的一条：文档声称默认开启，实测两个开关都不灵。
  - [#40314](https://github.com/anthropics/claude-code/issues/40314)（2026-03-28）：**传输层差异 bug——stdio server 的工具会被 defer，HTTP/Streamable HTTP server 的 50+ 工具、120K tokens 全部内联加载**。`ENABLE_TOOL_SEARCH=auto:5` 无效。
  - 同源还有 #18397、#19890、#25892（Claude.ai/Desktop 侧），构成一条"auto 模式不触发"的 issue 链。
- **关键事实：这些 issue 全部是 bot 以"重复"或"不活跃"自动关闭的，没有一条有官方修复确认**。上游主 issue #18263 于 2026-02-28 因"inactive too long"关闭；#41472 于 2026-05-05 关闭。课程表述应为"社区持续报告、官方未公开确认根治"。
- SDK 侧翻车：[claude-agent-sdk-typescript #356](https://github.com/anthropics/claude-agent-sdk-typescript/issues/356)（2026-06-23）：文档宣称 tool search "applies to all registered tools"，实测 `query()` 路径下 `createSdkMcpServer` 的进程内工具不被 defer；同一 issue 的源码分析还指出**内置工具（Read/Bash 等）永远不被 defer，只有 MCP 工具有资格**。

## 问题 4：有没有"大家都这么以为但其实是错的"的认知差？

**判定：【几乎没人讲透】——这是本集最值钱的点，至少有三个。**

1. **"Tool Search 默认开着，所以上下文税已经不用管了"——错。** 官方文档（2026-08-20 抓取）确实写着 "Tool search is enabled by default"，但：①失灵 issue 链（问题 3）贯穿 2026-01 至 2026-06 无人认领修复；②官方文档自己列明了 tool search 不生效的配置：自定义 `ANTHROPIC_BASE_URL`、`ENABLE_TOOL_SEARCH=false`、Google Cloud Agent Platform 上的旧模型、Bedrock、Microsoft Foundry（走 WaitForMcpServers 回退路径）；③还有文档自相矛盾的实锤：[issue #79033](https://github.com/anthropics/claude-code/issues/79033)（2026-07-19）指出 code.claude.com/docs 与 Anthropic 官方合作伙伴培训材料（Claude Code 101）对默认行为的描述互相冲突。**课程口径：文档说默认开 ≠ 你的配置里真的开了；验证方法是用 `/context` 看 MCP tools 行，而不是信文档。**
2. **"删配置是唯一的节食手段"——错。** `/mcp` 面板的 toggle 是 per-project 持久化的禁用（写 `~/.claude.json` 的 `disabledMcpServers`），配置保留、随时可恢复（官方 MCP 文档 "Disable a server without removing it" 节）。"闲置就删"可以更精细化为"闲置先 disable，确认不用再删"。
3. **"/doctor 是装完 Claude Code 查安装健康的命令"——过时认知。** 早期 `claude doctor` 只是安装/auto-updater 诊断（旧[故障排查文档](https://jackdog668-claude-code.mintlify.app/advanced/troubleshooting)、[issue #5493](https://github.com/anthropics/claude-code/issues/5493) 均为此义）；但 **2026-07 起 /doctor 已变成 bundled skill，做的就是"配置体检"**：从 [issue #76219](https://github.com/anthropics/claude-code/issues/76219) 里的真实输出可见，它检查"nothing installed that's unused, no leftover config, no CLAUDE.md bloat"，并扫描 skills/plugins/MCP servers——与课程"体检"用法一致。且它是 `disableBundledSkills` 唯一关不掉的 bundled skill（[官方 skills 文档](https://code.claude.com/docs/en/slash-commands)，2026-08-20 抓取）。这个行为变化社区几乎没人讲，只在 awesome 列表里有一句"Anthropic's July 2026 bundled setup-checkup skill"（[awesome-harness-engineering](https://github.com/ai-boost/awesome-harness-engineering)）。

## 问题 5：相关官方功能/命令现状（课程引用准确性核对）

**判定：引用时必须按以下最新口径，否则会被观众打脸。**

- **Tool Search**（[code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp)，2026-08-20 抓取）：
  - 文档口径："Tool search is enabled by default. MCP tools are deferred rather than loaded into context upfront"。历史触发阈值为 MCP 工具描述 >10% 上下文（Thariq 2026-01 公告）。
  - 环境变量 `ENABLE_TOOL_SEARCH`（true / auto:N）可强制；`ENABLE_TOOL_SEARCH=false`、自定义 `ANTHROPIC_BASE_URL`、Bedrock/Foundry/部分 Vertex 旧模型下退化为 WaitForMcpServers 路径。
  - API 层：`defer_loading: true` + `tool_search_tool_bm25_20251119`（BM25 检索变体，2025-11-19 版本号，见 [Growth Method 解析](https://growthmethod.com/anthropic-tool-search/)与 SDK issue #356）。
  - 只有 MCP 工具可被 defer；内置工具永不 defer（SDK issue #124/#356 的源码分析）。
  - 已知未确认修复的失灵面：auto 阈值不触发（#18298/#18397/#19890）、HTTP 传输不 defer（#40314）、OAuth 路径 400 报错（[OmniRoute #3974](https://github.com/diegosouzapw/OmniRoute/issues/3974)）、桌面端（#41472/#25892）。
- **/doctor**（[官方 skills 文档](https://code.claude.com/docs/en/slash-commands)，2026-08-20 抓取）：
  - 现状：**bundled skill**（prompt-based，非固定逻辑命令），约 2026-07 上线；`disableBundledSkills` 唯一豁免项。
  - 检查内容（据 issue #76219 真实输出）：未使用的安装、残留配置、CLAUDE.md 臃肿、权限设置、skills/plugins/MCP servers。
  - 旧义残留：`claude doctor` CLI 子命令仍是安装健康诊断（zsh 补全、cheat sheet 里仍是"check the health of your Claude Code installation / auto-updater"）。课程若讲 /doctor，需区分"CLI 子命令的旧义"与"2026-07 后 bundled skill 的新义"。
- **/mcp 面板 toggle**：per-project 禁用持久化在 `~/.claude.json` 的 `disabledMcpServers`（opt-out）/ `enabledMcpServers`（仅对默认关闭的内置 server 生效的 opt-in）；与 `.mcp.json` 审批用的 `enabledMcpjsonServers`/`disabledMcpjsonServers` 是两套不相干机制（官方文档明确警告不要混淆）。
- **/context**：查看 MCP tools 实际 token 占用的官方手段，是验证 Tool Search 是否生效的唯一可靠方法（issue 报告者均用它取证）。
- **MCP 输出侧**（与本集弱相关但可提一句）：单条工具输出 >10k tokens 有警告，`MAX_MCP_OUTPUT_TOKENS` 默认上限 25k；超长结果持久化到磁盘替换为文件引用。

---

## 对课程定位的启示

**观众早知道、讲了会水的部分：**
- "MCP 装多了吃上下文"这个概念本身——已是常识，一句话带过即可，不要当新知讲。
- "150k→2k / 98.7%" 这个数字——被二手文章复读烂了，引用可以，但不能当爆点。
- "用 /mcp 删掉不用的 server"——社区教程常见题材，单独讲不值钱。

**真盲区、讲了值钱的部分（按价值排序）：**
1. **"Tool Search 默认开启"是文档承诺不是事实**——失灵 issue 链（#18298/#47645/#40314，全部 bot 关闭无人认领）+ 文档/官方培训自相矛盾（#79033）+ 文档自己承认的多条失效路径（自定义 BASE_URL、Bedrock/Foundry 等）。给出可操作的验证动作：**`/context` 看 MCP tools 行，不信文档**。这是"大家都这么以为但其实是错的"的标准认知差。
2. **/doctor 的 2026-07 行为升级**——从"安装诊断"变成"配置体检 bundled skill"，正好就是课程的解法，但社区对新行为几乎没有覆盖；讲新行为 = 课程信息差。同时要把旧义/新义区分清楚，避免观众拿旧文档来杠。
3. **disable ≠ delete 的精细化解法**——/mcp toggle 的 per-project 持久化语义（`disabledMcpServers`）让"闲置就删"升级为"闲置先禁用、确认再删"，比社区主流的"删配置"教程更进一步，且有两套易混淆机制（disabledMcpServers vs disabledMcpjsonServers）这个官方文档明确警告的坑。
4. **传输层差异**（HTTP 不 defer、stdio defer，#40314）——对装了大量云端 HTTP connector 的观众是直接命中痛点。

**引用风险提示：**
- 营销号数字（72%/82%/143k）只能标注为"社区流传口径"；严肃引用用 Anthropic 官方 98.7%。
- 所有 Tool Search 失灵 issue 均无官方修复确认，课程表述必须是"社区持续报告"而非"官方承认的 bug"。
- /doctor 新行为的官方文档只有 bundled skill 列表里的一句话 + issue 里的真实输出，没有专门的官方说明页——讲课时出示 issue #76219 的输出截图比引用文档更有说服力。
