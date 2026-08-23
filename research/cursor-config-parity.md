# Cursor 配置机制对照：MCP / Rules / 体检与上下文占用

调研日期：2026-08-23（所有 URL 均于当日抓取/核验）。供 ep01「配置只加不减：MCP/skill/rules 只装不删，常驻内容吃上下文」的「Cursor 对应卡」取材——课程案例全部用 Claude Code 拍摄，本报告回答"同样的方法在 Cursor 里适用吗"。

来源分级：cursor.com/docs（含 cursor.com/help 定制化页）、cursor.com/changelog、cursor.com/blog 为一手官方口径；forum.cursor.com 中 Cursor 员工（如 deanrie）的回复为一手官方口径，普通用户帖为社区报告；工程博客只作线索。注意：本次调研环境无法抓取 web.archive.org（工具屏蔽），凡涉及"旧版文档原文"处均已标注核验状态。另注：docs.cursor.com 已 308 重定向到 cursor.com/docs，引用旧域名的二手文章链接多已失效。

---

## 问题 1：MCP 配置机制现状（两档配置 / toggle / 是否全量注入）

**判定：【属实且对应关系良好】——全局/项目两档属实；server 级 UI toggle 属实（工具级 toggle 文档未载）；"全量注入"已在 2026-01 起被官方的 Dynamic Context Discovery（延迟加载）取代。**

- **两档配置属实**。官方 MCP 文档（[cursor.com/docs/mcp](https://cursor.com/docs/mcp)，2026-08-23 抓取）"Configuration locations" 节：项目级 "Create `.cursor/mcp.json` in your project for project-specific tools"；全局级 "Create `~/.cursor/mcp.json` in your home directory for tools available everywhere"。与 Claude Code 的 `.mcp.json` / `~/.claude.json` 两档结构对应。
- **server 级禁用不删配置，GUI + CLI 双通道**：
  - GUI：同一文档页 "Toggle servers on/off without removing them: 1. Open **Customize** in the sidebar 2. Find the MCP server you want to change 3. Use the toggle to enable or disable it."——语义上就是 Claude Code `/mcp` 面板 toggle 的对应物（"禁用 ≠ 删除"论点在 Cursor 成立）。
  - CLI（Cursor CLI/cursor-agent）：`agent mcp list` / `agent mcp list-tools <identifier>` / `agent mcp enable <identifier>` / `agent mcp disable <identifier>`，交互模式有 `/mcp list`（[cursor.com/docs/cli/mcp](https://cursor.com/docs/cli/mcp)，2026-08-23 抓取）。文档未写明启停状态持久化到哪个文件——【未找到】。
  - **单个工具级 toggle：当前文档未载**。文档中唯一的工具粒度控制是企业管理端的 "Tool allowlists restrict which tools from an approved server can run automatically"。但官方员工 deanrie 在论坛回复中建议用户"manually disabling unused tools"（见问题 2 引文），且大量社区帖描述 MCP 设置里可点选单个工具（如 [forum: Disabled tools in MCP](https://forum.cursor.com/t/disabled-tools-in-mcp/133385)、[forum: 预配置工具开关的功能请求](https://forum.cursor.com/t/add-the-ability-to-pre-configure-which-mcp-tools-are-enabled-disabled-via-the-mcp-json-configuration-file-eliminating-the-need-to-manually-enable-tools-in-the-ui-after-every-cursor-restart/148139)，后者还抱怨工具开关状态重启后不保留）。口径：**UI 有此能力（社区+员工口径），但官方文档现状只写到 server 级**——上卡需标注。
- **是否全量注入：曾经是，2026-01 起官方转向延迟加载**。Cursor 官方博客 [Dynamic context discovery](https://cursor.com/blog/dynamic-context-discovery)（2026-01-06 发布，作者 Jediah Katz 等，2026-08-23 抓取）承认旧行为并描述新机制：把 MCP 工具描述同步为文件夹，agent 启动时只拿到极小的静态上下文（工具名列表），需要时再查询工具详情。2.4 版 changelog（[cursor.com/changelog/2-4](https://cursor.com/changelog/2-4)，2026-01-22 发布，2026-08-23 抓取）落地表述："MCP servers now live as JSON files in `.cursor`. Agents discover and load MCPs only when needed, reducing token usage and keeping the context focused."——这是 Claude Code Tool Search 的同构物（同样是 progressive disclosure 思路）。博客称 "These improvements will be live for all users in the coming weeks"，即 A/B 渐进推送；**是否已对全体用户/全部模型默认生效，官方无后续确认——【未找到】，待开拍实测**。

## 问题 2：传闻中的"只发送前 40 个 MCP 工具"限制

**判定：【曾真实存在，官方员工两度确认；现行文档已无此限制表述，但无官方"已移除"公告】。**

- **限制曾存在，且有官方员工一手确认**：
  - [forum: Tools limited to 40 total](https://forum.cursor.com/t/tools-limited-to-40-total/67976)（2025-03-21 发帖，2026-08-23 抓取）：用户抱怨 MCP 工具总数超 40 后部分工具不可用；Cursor 团队成员 **deanrie** 2025-04-04 回复，就工具级开关表示 "we will be able to implement this soon"。
  - [forum: Request: Increase MCP Tools Limit](https://forum.cursor.com/t/request-increase-mcp-tools-limit-for-enhanced-development-workflow/108637)（2025-06-24 发帖，2026-08-23 抓取）：**deanrie 官方口径原话："The 40 tool limit is actually there for a good reason - if we increased it, the AI would struggle to effectively choose between all the available tools."** 并称团队 "working on a better system for the AI to intelligently select the right tools"（即后来的 dynamic context discovery），建议手动关闭不用的工具。这是"限制=40、动机=工具选择质量/上下文管理"的最硬一手来源。
  - 同期还有 [forum #81627](https://forum.cursor.com/t/mcp-server-40-tool-limit-in-cursor-is-this-frustrating-your-workflow/81627)、[forum #79686](https://forum.cursor.com/t/mcp-40-tools-way-to-less/79686) 等多条讨论链；超限时产品内弹 "Exceeding total tools limit" 警告（警告原文完整文本仅见于二手博客转述，一手截图未能核验——【未找到】）。
- **旧版文档曾写明"只发送前 40 个工具"**：多个论坛帖（2025-03/06）引用当时 docs.cursor.com 的表述，二手文章（truefoundry、nxcode 等）转述为 "Cursor will only send the first 40 tools to the Agent"。因 web.archive.org 无法抓取，**旧文档原文本次未能一手核验**，课程引用时应以 deanrie 论坛原话为准，不要直接引"文档原文"。
- **现状**：现行 [cursor.com/docs/mcp](https://cursor.com/docs/mcp)（2026-08-23 抓取）全页无任何工具数量上限表述。社区报告（[forum: Regarding the quantity limit of MCP tools](https://forum.cursor.com/t/regarding-the-quantity-limit-of-mcp-tools/153432)，2026-03-03，2026-08-23 抓取）称启用 80+ 工具已无任何警告，并归因于 dynamic context discovery；该帖无官方回复。**结论口径：硬上限大概率已随延迟加载机制废弃，但"已移除"没有官方公告或文档确认——课程表述应为"旧机制（官方确认过）+ 新机制（文档已删限制）+ 现状待实测"。**

## 问题 3：Cursor Rules 的常驻机制

**判定：【四类机制属实，但官方命名已更新】——Always Apply 常驻每轮、其余三类按需；"Agent Requested"是旧名，现名 "Apply Intelligently"；.cursorrules 官方明文 legacy；官方有"500 行"纪律但无 token 数字。**

- **四类及注入方式**（[cursor.com/docs/rules](https://cursor.com/docs/rules)，2026-08-23 抓取，frontmatter 映射为文档原文）：
  1. **Always Apply**（`alwaysApply: true`）："Apply to every chat session"——每轮常驻，是 Cursor 侧的"常驻税"主体，对应 Claude Code 的 CLAUDE.md 常驻段。
  2. **Apply Intelligently**（`alwaysApply: false` + 有 `description`）："When Agent decides it's relevant based on description"——description 供 Agent 判断、正文按需拉入，即 progressive disclosure 形态，对应 Claude Code skill 的 name+description 常驻/正文按需。
  3. **Apply to Specific Files**（`alwaysApply: false` + 有 `globs`）："When file matches a specified pattern"——文件命中才附带。
  4. **Apply Manually**（`alwaysApply: false`、无 description/globs）："When @-mentioned in chat"。
  - 注入位置官方原话："When applied, rule contents are included at the start of the model context."；多源合并优先级 "Team Rules → Project Rules → User Rules. All applicable rules are merged."
  - **命名沿革注意**：课程若沿用社区熟知的旧四名 Always / Auto Attached / Agent Requested / Manual，需注明为旧版文档命名——旧名在官方论坛大量存在（如 [forum #84787](https://forum.cursor.com/t/for-thinking-models-agent-requested-and-auto-attached-rules-are-pulled-in-after-thinking-has-occurred/84787)、[forum #100672](https://forum.cursor.com/t/correct-way-to-specify-rule-type/100672)，2026-08-23 抓取），现行文档已改为上述新名。旧名与新名逐一对应（Auto Attached→Apply to Specific Files，Agent Requested→Apply Intelligently）。
- **常驻成本（token）官方口径：【未找到】数字**。官方唯一的体积纪律是 "Keep rules under 500 lines. Split large rules into smaller, focused files."（[cursor.com/docs/rules](https://cursor.com/docs/rules) 与 [cursor.com/help/customization/rules](https://cursor.com/help/customization/rules)，均 2026-08-23 抓取）。行数纪律 = 官方默认承认规则体积有代价，但没有任何"规则占多少 token"的官方数字。
- **.cursorrules 现状**：官方帮助页原话 **"The `.cursorrules` file in your project root is legacy and will be deprecated."**（[cursor.com/help/customization/rules](https://cursor.com/help/customization/rules)，2026-08-23 抓取）。注意主文档 [cursor.com/docs/rules](https://cursor.com/docs/rules) 已不再提 .cursorrules——迁移路径为 `.cursor/rules/*.mdc`（YAML frontmatter + 正文）。另外 `.cursor/rules` 里的纯 `.md` 文件因无 frontmatter 会被规则系统忽略（帮助页明确）。
- **2026 年新变量：Skills**。Cursor 2.4（[changelog](https://cursor.com/changelog/2-4)，2026-01-22）引入 Agent Skills，官方定位原话："Compared to always-on, declarative rules, skills are better for dynamic context discovery and procedural 'how-to' instructions."；skills 文档（[cursor.com/docs/skills](https://cursor.com/docs/skills)，2026-08-23 抓取）："Skills load resources on demand, keeping context usage efficient."，目录为 `.cursor/skills/`、`.agents/skills/` 及各自 `~/` 全局版，**且兼容加载 Claude/Codex 的 skills 目录**。内置 `/migrate-to-skills` 会把 "Apply Intelligently" 规则和 slash commands 转为 skills，`alwaysApply: true` 或带 globs 的规则保留为规则。即：Cursor 官方自己把"常驻 vs 按需"划成了 rules vs skills 两层——与课程的 progressive disclosure 论点同构。
- 补充：AGENTS.md 在现行文档中是 `.cursor/rules` 的 "plain markdown alternative"，支持子目录嵌套（就近优先），User Rules 随账号跨设备同步、不作用于 Inline Edit（Tab）。

## 问题 4：配置体检 / 上下文占用查看的对应物

**判定：【/doctor 无对应物；/context 只有半个对应物】——Cursor 有总量百分比指示器（且 2026-07 起退化为 hover 才显示），无分项明细，无官方配置体检命令。**

- **/doctor（配置体检）对应物：【未找到】**。官方文档、changelog、论坛均无任何"审计配置健康/清理闲置安装"的内置命令。社区有第三方补位工具（如 dev.to 用户 nedcodes 的 "cursor-doctor"，宣称做 rules 冲突/冗余/token budget 分析——二手线索，[dev.to 文章](https://dev.to/nedcodes/i-rewrote-my-cursor-linter-into-a-full-diagnostic-tool-and-added-auto-fix-5ehb)，2026-08-23 抓取），恰好反证官方空缺。
- **/context（占用明细）对应物：只有总量，无明细**：
  - 官方一手：1.3 版 changelog（[cursor.com/changelog/1-3](https://cursor.com/changelog/1-3)，2025-07-29 发布，2026-08-23 抓取）："At the end of a conversation you can now see how much of the context window is used."——上下文用量可见性从此进入产品。
  - 社区口径（无官方文档页）：百分比可 hover 显示原始 token 数（"X / Y tokens used"）；**3.11.13 版（2026-07）起百分比从状态栏常显退化为 hover 才显示**，用户请愿恢复常显（[forum #165379](https://forum.cursor.com/t/please-bring-back-the-always-visible-context-usage-percentage/165379)，2026-07-10 发帖；另见 [#139914](https://forum.cursor.com/t/the-consumption-indicator-of-the-context-window-appears-to-have-been-removed/139914)、[#139957](https://forum.cursor.com/t/context-window-usage/139957)，均 2026-08-23 抓取，无官方回复）。
  - **关键差异：Cursor 的指示器不提供 Claude Code /context 那样的分项明细（system prompt / MCP tools / rules / messages 各占多少）**——即课程里"用 /context 取证 MCP tools 占用"的动作在 Cursor 无法复刻，只能看总量。分项功能【未找到】任何官方或社区证据。
  - 相邻机制：上下文满时自动摘要 + 手动 `/summarize`（1.6 版 changelog 原话 "Cursor automatically summarizes long conversation for you when reaching the context window limit. You can now summarize context on-demand with the `/summarize` slash command."，[cursor.com/changelog/1-6](https://cursor.com/changelog/1-6)，2025-09-12 发布，2026-08-23 抓取）。

## 问题 5：官方有无"装太多有害/上下文税"的承认表述

**判定：【有，且比 Anthropic 版更直白】——一手可逐字引用的至少三条。**

1. **官方博客承认 MCP 工具膨胀上下文**（[cursor.com/blog/dynamic-context-discovery](https://cursor.com/blog/dynamic-context-discovery)，2026-01-06 发布，2026-08-23 抓取）：
   > "Some MCP servers include many tools, often with long descriptions, which can significantly bloat the context window. Most of these tools go unused even though they are always included in the prompt."
   
   官方数字（同文）：
   > "In an A/B test, we found that in runs that called an MCP tool, this strategy **reduced total agent tokens by 46.9%** (statistically significant, with high variance based on the number of MCPs installed)."
   
   ——Cursor 侧的"150k→2k"对应物；引用时注意官方自带的两个限定：仅统计"调用过 MCP 工具的运行"、方差随安装数量大。
2. **官方员工承认"工具多了模型选不准"**（deanrie，论坛官方回复，2025-06-24）："The 40 tool limit is actually there for a good reason - if we increased it, the AI would struggle to effectively choose between all the available tools."（[forum #108637](https://forum.cursor.com/t/request-increase-mcp-tools-limit-for-enhanced-development-workflow/108637)，2026-08-23 抓取）——把"装太多有害"直接写进了产品设限动机。
3. **changelog 承认按需加载省 token**（[cursor.com/changelog/2-4](https://cursor.com/changelog/2-4)，2026-01-22）："Agents discover and load MCPs only when needed, reducing token usage and keeping the context focused."；同版对 rules 的定性 "Compared to always-on, declarative rules…" 中 "always-on" 一词即官方对 rules 常驻性的白纸黑字。
4. 弱一档但可用：rules 文档的 "Keep rules under 500 lines. Split large rules into smaller, focused files."（问题 3）——体积纪律型承认，无 token 数字。

---

## 对 ep01 Cursor 对应卡的启示

**在 Cursor 侧成立、可直接写上卡的论点：**
- **反模式本身在 Cursor 更"实锤"**：Cursor 曾因上下文/工具选择问题给 MCP 工具设过 40 个硬上限（官方员工两度确认），"装太多有害"在 Cursor 是被产品设限背书过的事实，不只是最佳实践。
- **两档配置结构同构**：全局 `~/.cursor/mcp.json` vs 项目 `.cursor/mcp.json`，"全局装的东西每个项目都背着"论点直接迁移。
- **"闲置先禁用、确认再删"成立**：Customize 面板 server toggle（不删配置）+ CLI `agent mcp enable/disable`，对应 Claude Code /mcp toggle。
- **"官方自己承认上下文税"成立**：46.9% + "significantly bloat the context window / Most of these tools go unused" 两条一手原话，与 Anthropic 150k→2k 并排引用效果佳。
- **progressive disclosure 论点成立且官方分层更显式**：Always Apply（常驻）vs Apply Intelligently（description 触发、按需）vs Skills（"load resources on demand"）；2.4 起 MCP 也延迟加载。"常驻的才是税，按需的不是"这条主论点两边通用。

**不成立或无对应物、卡上须注明差异的：**
- **无 /doctor 对应物**：Cursor 没有任何官方配置体检命令；课程解法"/doctor 体检"在 Cursor 只能手动做（打开 Customize/设置逐项盘点），或用第三方社区工具。这是对应卡上最大的空格。
- **/context 只有半个对应物**：Cursor 仅有总量百分比（3.11.13 起还要 hover 才看得到），**没有分项明细**——"用 /context 看 MCP tools 吃了多少"这个取证动作在 Cursor 做不到，卡上应改写为"hover 看总量百分比，装/卸前后对比"。
- **工具级开关口径分裂**：UI 实际可关单个工具（社区+员工口径）但现行文档只写 server 级 toggle，且有社区报告称工具开关状态重启不保留——卡上不要断言"支持工具级禁用"。
- **rules 术语已换代**：卡上用 Always/Auto Attached/Agent Requested/Manual 旧名会与现行产品 UI（Always Apply/Apply Intelligently/Apply to Specific Files/Apply Manually）对不上，须用新名或并列注明。
- **官方 token 数字缺位**：除 46.9% 外，rules/工具常驻占用无任何官方数字（只有"500 行"行数纪律），卡上不要出现编造的 token 数。

**待开拍复核（时效敏感）：**
1. 40 工具上限现状实测：装 40+/80+ 工具，看是否还有 "Exceeding total tools limit" 警告、工具是否全部可用（社区称已无限制，无官方公告）。
2. Dynamic context discovery 是否已对当前版本/所选模型默认生效（博客 2026-01 称"coming weeks"渐进推送，无后续官方确认）。
3. 单个工具 toggle 的当前 UI 位置与持久化行为（重启是否保留）。
4. 上下文百分比指示器在最新版的确切位置与 hover 行为（3.11.13 后有变动、用户请愿中，可能再改）。
