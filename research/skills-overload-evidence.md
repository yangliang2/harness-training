# Skill 装太多的代价：选择退化、上下文成本与"~1% 预算"出考证

调研日期：2026-08-20（UTC；本地 2026-08-21 00:12 CST，所有 URL 均于当日抓取/核验，本地文件基于本机 Claude Code v2.1.235）。供 ep01「配置只加不减」一集的 **skill 侧** 证据使用；MCP/工具侧见姊妹篇 `research/mcp-context-tax-tool-search-doctor.md`。

来源分级：官方文档/官方工程博客/官方 CLI 二进制内嵌文本为一手；GitHub issue 为一手（用户报告，注意甄别是否被官方确认）；严肃工程博客为二手；营销向内容只作线索。

**核心边界（贯穿全篇）：skills ≠ tools。** 工具选择随数量退化有对照研究（tool overload 文献），不能直接当 skill 证据用。本报告只把"skill 专属"的证据标为一手；tool 侧研究一律标注"类比证据"。

---

## 子问题 1：skill 变多后，选择/路由准确率是否下降？

**判定：【有一手证据】——官方设置文档与 bundled doctor skill 的 prompt 都直接承认"超预算 → 描述被丢弃 → 路由退化"这条因果链；另有用户实测 issue 佐证。但没有"skill 数量 vs 选对率"的对照实验数据。**

按强度排序的证据：

1. **官方设置文档自认路由退化机制**（最强）。`skillListingBudgetFraction` 条目（[code.claude.com/docs/en/settings](https://code.claude.com/docs/en/settings)，2026-08-20 抓取）原文：
   > "Default: 0.01. Fraction of the model's context window reserved for the skill listing Claude sees each turn… **When the listing exceeds the budget, descriptions for the least-used skills are dropped and only their names are listed, so Claude can still invoke them but can't see what they do.**"
   "can still invoke them but can't see what they do" 就是官方口径的"路由信号被削"——skill 的 description 是模型决定何时调用的唯一依据（见子问题 2 的官方描述），描述没了就只剩裸名字。

2. **bundled doctor skill 的 prompt 文本自认**（本机一手文件）。从本机官方 CLI 二进制（`~/.local/share/claude/versions/2.1.235`，ELF/bun 单文件，strings 提取）中取得 doctor skill 的完整内嵌 prompt，其中一句：
   > "The skill listing is budgeted at ~1% of the context window; when summed descriptions exceed it, **entries get truncated and skill routing degrades** — so a bloated listing matters even before raw token cost does."
   这是 Anthropic 自己写的 skill，白纸黑字用了 "skill routing degrades"。本机一次真实 `/doctor` 运行（`spike/verify/ep01-mcp-tax/runs/slash-doctor.json`）的输出与此一致："the skill listing (~5k est. tokens, over budget)…shrinks the listing (which is over its ~1% budget, degrading skill routing)"。

3. **用户实测 issue：描述丢失 → 选择退化为"看名字猜"**。[issue #68677](https://github.com/anthropics/claude-code/issues/68677)（2026-06-15 开，2026-06-17 关）：27 个用户 skill 里约一半（~14/27）在 system-reminder 里只剩名字、description 被静默丢弃；报告者原话 "The omitted descriptions are the primary routing signal agents use to decide which skill to invoke. Without them, **skill selection degrades to guessing from the name alone**." 评论区用户 yurukusa 反编译二进制定位到两个内部设置（`skillListingBudgetFraction` 默认 0.01、`skillListingMaxDescChars` 默认 1536），报告者确认**调大 `skillListingBudgetFraction` 后修复**——即这不是随机 bug，而是预算机制的行为，当时该机制"只在二进制里有、文档没写"（评论原话 "documented-only-in-the-binary"）。
   - 注意甄别：此 issue 由 bot 判重、用户自行关闭，无官方人员确认；但机制本身现已被官方设置文档收录（证据 1），issue 与文档互相印证。

4. **"大规模 skill 库需要检索层"的用户主张，官方未回应**。[issue #20143](https://github.com/anthropics/claude-code/issues/20143)（2026-01-22 开，2026-02-28 被 inactivity bot 关闭）：请求参照 Tool Search 给 skill 做 `skill_search` 检索层，理由是 "Decision quality degrades: Too many options increases confusion and mis-selection"。被 [issue #84666](https://github.com/anthropics/claude-code/issues/84666)（2026-08-06 开，**至今 open**）重新提起，指出 skill 元数据是"每个已装 skill 都急切注入、无上限、无过滤"，与 MCP 工具的 Tool Search 待遇不对称，并引用一个组织"一个月内 67 → 183 个 skill"的案例。**这两条 issue 均无官方人员回复**，只能表述为"社区持续主张"，不能表述为"官方承认"。

5. **相邻现象（非数量驱动，但同回路）**：[issue #30387](https://github.com/anthropics/claude-code/issues/30387)（2026-03-03，[MODEL] 标签）：自定义 skill 约 50% 情况下不被自动触发，尤其当 skill 与模型训练时就会的行为（git、shell）重叠时——原因推测是训练先验与 skill 指令竞争，**与 skill 数量无关**，引用时须与"装多了"区分。

**未找到**（如实标注）：没有任何官方或第三方对"skill 数量 vs 触发准确率"做过对照测量（类似 tool 侧的 benchmark）。课程若要量化曲线，只能引用 tool 侧研究并明确标注"类比证据"。

## 子问题 2：skill 的常驻成本机制与官方数字

**判定：【有一手证据】——机制有官方工程博客 + 官方文档双重描述；每个 skill 的元数据上限有官方数字（1536 字符）。**

- **Progressive disclosure 的官方描述**（[Anthropic 工程博客 "Equipping agents for the real world with Agent Skills"](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)，2025-10 发布、2025-12-18 有更新注记，2026-08-20 抓取）：
  > "At startup, the agent **pre-loads the `name` and `description` of every installed skill** into its system prompt. This metadata is the **first level** of progressive disclosure…"
  即：装得多 ≠ 全文进上下文，但**每个 skill 的 name+description 是逐轮常驻的**；SKILL.md 正文（第二级）和附属文件（第三级）只在触发时加载。
- **每个 skill 的元数据开销上限，官方数字**：官方 skills 文档 frontmatter 表（[code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills)，2026-08-20 抓取）："the combined description and when_to_use text is **truncated at 1,536 characters** in the skill listing to reduce context usage"。对应设置 `skillListingMaxDescChars` 默认 1536（官方设置文档，同上）。**没有官方给出的"平均每 skill token 数"**；doctor 内部用 chars/4 估算（bundled prompt 原文："Token figures are estimates: tokens ≈ characters / 4. Label them 'est.' everywhere"）——课程可沿用此口径，标注为估算。
- **触发后的成本也有官方口径**：skill 正文一旦被调用，"its content stays in context across turns, so **every line is a recurring token cost**"（官方 skills 文档 "Types of skill content" 节）；auto-compaction 后每个最近调用的 skill 重挂前 5,000 tokens、合计共享 25,000 tokens 预算（同文档 "Skill content lifecycle" 节）。即 skill 的成本是两段的：常驻元数据 + 触发后常驻正文。
- **官方给的节流手段**（均为设置文档/skills 文档一手）：`skillListingBudgetFraction`（默认 0.01）、`skillListingMaxDescChars`（默认 1536）、`skillOverrides`（on / name-only / user-invocable-only / off，/skills 菜单可写）、`disableBundledSkills`（唯独关不掉 /doctor；v2.1.235 起另有 `DISABLE_DOCTOR_COMMAND` 隐藏入口）。

## 子问题 3："~1% 预算"说法的出处考证

**判定：【有一手证据，出处链完整】——从"只在二进制里"到"写进官方文档"，2026 年 6→8 月完成了官方化。**

出处链（按时序）：

1. **二进制内嵌设置 schema**（最早可考形态）：官方 CLI 二进制中的设置描述字符串（本机 v2.1.235 strings 提取，issue #68677 评论者 2026-06 在当时的 npm 二进制中独立发现同样文本）：
   > `skillListingBudgetFraction`: "Fraction of the context window (in characters) reserved for the skill listing sent to Claude (default: 0.01 = 1%). When the listing exceeds this, descriptions are shortened to fit."
   2026-06 时该机制**只有二进制里有、文档未收录**（#68677 评论原话 "documented-only-in-the-binary budget mechanism"）。
2. **官方设置文档正式收录**（现状，2026-08-20 抓取）：`skillListingBudgetFraction` 条目明确 "Default: 0.01… the default reserves 1%"，并注明 "**/doctor estimates the listing cost against the budget**"——即 /doctor 那句 "over its ~1% budget" 是官方设计行为，不是模型口嗨。
3. **bundled doctor skill prompt 原文**（本机一手）："The skill listing is budgeted at ~1% of the context window…"（完整原文见子问题 1 证据 2）。doctor 判断"over budget"的依据是磁盘估算（chars/4），它自己也声明 "Recommend `/context` for the exact live measurement; your figures are disk-based estimates"。
4. **历史形态**：2025-12 的 [issue #14549](https://github.com/anthropics/claude-code/issues/14549) 提到当时预算表现为"documented ~15,000 char budget"（约 15k 字符的总额度），与今天"上下文的 1%"口径不同——说明预算的**计价方式**随版本演变过（固定字符数 → 窗口比例），讲课时用当前官方口径（1% of context window）即可，不要引用旧的 15k 数字。

## 类比证据边界（skills ≠ tools）

- "选项一多、选择质量下降"在 **tool** 侧有研究和官方方案（Tool Search、Anthropic 工程博客 Code execution with MCP 的 150k→2k，见姊妹篇）。这些**只能类比引用**，课程表述应为"工具侧已有实测，skill 侧官方文档承认了同一机制的退化路径，但无量化研究"。
- #20143/#84666 正是用这个不对称性（工具有 Tool Search、skill 没有）立论的——这本身是课堂上好用的论证结构，但它是**用户 feature request 的论证**，不是官方结论。

---

## 对 ep01 的启示

**证据扎实的部分，可以放心讲：**

1. **"skill 装多了也有问题"现在是可以拿官方文档逐字讲的**，不再是推测：skill 越多 → name+description 常驻总量越大 → 超过上下文的 1% 预算 → **最久未用的 skill 被丢到只剩名字** → 模型"还能调用但不知道它是干嘛的"（官方设置文档原话）。这条因果链的每一环都有一手出处。
2. **~1% 预算的出处完全坐实**：官方设置文档 + bundled doctor prompt + 本机 /doctor 实跑输出三重印证。课程可以直接放 `slash-doctor.json` 的实跑截图 + 设置文档条目，说服力拉满。
3. **课程解法与官方机制对齐**："闲置就删"可以精细化为官方三板斧——`/skills` 菜单/`skillOverrides` 关闲置（可逆）、`skillListingBudgetFraction` 调预算（治标）、`disable-model-invocation: true` 把不该自动触发的 skill 移出 listing（连元数据都不占）。比社区"删目录"教程高一层。
4. **金句可用**：doctor prompt 的 "a bloated listing matters even before raw token cost does"——skill 膨胀的危害**先于** token 成本到来，是路由质量问题，这是本集区别于"MCP 吃 token"老话题的独特角度。

**诚实边界，必须画清：**

1. **没有"skill 数量 vs 准确率"的量化曲线**。不要编数字（"超过 N 个 skill 准确率下降 X%"这类说法无任何一手来源）。可以讲的极限是：官方文档承认超预算后"can't see what they do"，用户实测 27 个 skill 就触发过半数丢描述（#68677）。
2. **"路由退化"的官方表述限于"描述被丢弃"这一机制**；至于描述还在、纯粹因为选项多而选错，只有用户 feature request（#20143/#84666，无官方回应）和 tool 侧类比研究。讲课时用"工具侧已有实测、skill 侧官方承认了同款退化路径"的表述。
3. **issue 链的关闭方式**：#68677 是用户自行关闭（自行找到设置修复），#20143 是 inactivity bot 关闭——表述为"社区报告 + 官方文档后来收录了该机制"，不要表述为"官方承认的 bug 已修复"。
4. **/doctor 的预算是磁盘估算不是实测**：doctor 自己声明数字是 chars/4 估算、以 `/context` 为准——课程演示时这一步要演出来，否则会被较真的观众抓。

---

## 追加：MCP/skill 的作用域（scope）机制（2026-08-21 补调，供 ep01 解法二"归位"使用）

除注明外，本节 MCP 侧声明均出自 [code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp)（2026-08-21 抓取，本地 2026-08-21 00:21 CST），skill 侧出自 [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills)（2026-08-20 抓取，当日复核未变）；CLI 行为以本机 v2.1.235 `--help` 输出与 bundled doctor prompt（本机二进制 strings 提取）为一手本地验证。

### 1. MCP server 的 scope

**判定：【有一手证据】，机制文档完整。**

- **三种 scope 的官方定义表**（"MCP installation scopes" 节，逐字要点）：
  | Scope | Loads in | Shared with team | Stored in |
  |---|---|---|---|
  | Local | Current project only | No | `~/.claude.json`（该项目的条目下） |
  | Project | Current project only | Yes, via version control | 项目根 `.mcp.json` |
  | User | All your projects | No | `~/.claude.json`（顶层 `mcpServers`） |
- **`claude mcp add --scope` 的默认是 local，不是 user**——本机 v2.1.235 `claude mcp add --help` 原文：`-s, --scope <scope>  Configuration scope (local, user, or project) (default: "local")`；官方文档同口径："Each command writes to local scope unless you add --scope project or --scope user"。**这是"归位"解法最有力的官方事实：CLI 的默认行为就是"只装本项目"，装全局是主动 opt-in。**
- 官方对 local scope 的用途定位（逐字）："Use local scope for personal development servers, experimental configurations, or servers with credentials you don't want in version control."；对 user scope 的定位（逐字）："This scope works well for personal utility servers, development tools, or services **you frequently use across different projects**."
- **project scope（.mcp.json）的审批行为**：签入仓库后，队友首次在交互会话使用时弹审批——原文 "For security reasons, Claude Code prompts for approval in interactive sessions before using project-scoped servers from .mcp.json files"；审批前 `claude mcp list`/`get` 显示 `⏸ Pending approval`；重置用 `claude mcp reset-project-choices`。三个重要例外：① `claude -p`、Agent SDK、cloud 会话**不弹审批直接加载**；② bypassPermissions + `skipDangerousModePermissionPrompt` 也跳过；③ v2.1.196 起 untrusted 文件夹里，仓库自己签入的 `enableAllProjectMcpServers`/`enabledMcpjsonServers` 被忽略（"A cloned repository can't approve its own servers"），须先过 workspace trust 对话框。排除单个 server 用 `disabledMcpjsonServers`（任何模式下都阻断）。
- **用户级→项目级的迁移路径**：`claude mcp` **没有 move/copy 子命令**（本机 `claude mcp --help` 全量子命令：add / add-from-claude-desktop / add-json / get / list / login / logout / remove / reset-project-choices / serve）。实际操作 = `claude mcp remove <name>` 后用 `--scope project` 重新 add，或直接手写 `.mcp.json` 提交。注意 doctor prompt 的警告（本机一手）："Never use `claude mcp remove` to disable: it permanently deletes the server config (env vars, headers) and **wipes its OAuth tokens**"——迁移带 OAuth 的 server 须先记录配置，remove 后需重新 login。

### 2. skill 的 scope

**判定：【有一手证据】，但"项目级无需审批"是文档无审批流程的消极事实，引用时注明。**

- **四级位置表**（"Where skills live" 节，逐字要点）：Enterprise（managed settings，全组织）/ Personal `~/.claude/skills/<name>/SKILL.md`（"All your projects"）/ Project `.claude/skills/<name>/SKILL.md`（"This project only"）/ Plugin（随插件启用范围）。
- **同名冲突优先级（逐字）："enterprise overrides personal, and personal overrides project."** ——注意方向：personal 压 project。所以"归位"到项目级后**必须删掉全局同名 skill**，否则项目级那份永远不生效。这是课程演示里最容易翻车的点。
- **项目级 skill 没有审批/信任 gate**：官方文档没有 per-skill 审批流程（与 .mcp.json 的审批机制不同），clone 下来的 `.claude/skills/` 直接加载。官方唯一相关的安全提示（逐字）："A skill can grant itself broad tool access, so **review the `allowed-tools` of skills checked into a repository before you run Claude Code there**"——项目级 skill 的 `allowed-tools` 在 untrusted 文件夹也生效（"Claude Code applies a project skill's allowed-tools whenever you or Claude invoke the skill, including in a `-p` run in a folder you've never trusted"）。
- **移动 = 移动目录**：skill 无注册表，加载纯按路径扫描（personal/project 目录有 live change detection，会话内热加载，无需重启）；"Remove a skill" 节对 personal/project skill 的删除方式就是 "delete the skill's directory"。故归位操作就是 `mv ~/.claude/skills/<name> <repo>/.claude/skills/<name>`。【文档无专门"迁移"章节，此为机制的直接推论】

### 3. 官方有没有"项目专用的东西别装全局"的引导表述

**判定：【只有间接引导】——公开文档无逐字口号；最强的"官方引导"在 bundled doctor prompt（非公开文档）。**

- 公开文档侧最接近的三处（均为间接）：① `claude mcp add` 默认 local scope（机制即引导）；② user scope 的官方定位词是 "frequently use across different projects"（言下之意：不跨项目常用的不该放 user）；③ local scope 定位词 "personal development servers, experimental configurations"。**没有**"don't install project-specific servers globally"这类逐字句子；skill 文档的 "Where you store a skill determines who can use it" 是纯描述，无建议性表述。【skill 侧引导：未找到】
- **Bundled doctor prompt（本机一手，v2.1.235）里有实质的"归位"动作**：check 4（CLAUDE.md 迁移）规定 "Task-specific workflows ('how to deploy', 'release checklist', API references) → turn into a skill at `.claude/skills/<name>/SKILL.md`"——即官方工具自身会把全局/根级常驻内容迁移成**项目级**懒加载 skill；且对全局文件有明确的 scope 纪律："`~/.claude/CLAUDE.md` and ancestor-directory `CLAUDE.local.md` files load in EVERY project… Only propose removing content from them when it is **clearly specific to this project**"。这等于官方工具内置了"项目专用内容应归位到项目"的判断逻辑，可作为课程的权威背书（注明出处形态是 bundled skill prompt，不是公开文档页）。

### 4. /doctor 是否感知 scope

**判定：【有一手证据（本机实跑 + prompt 原文）】——报告区分 scope 展示，清理建议利用 scope 选择落点，但没有"跨 scope 迁移"类建议。**

- **实跑确认**（`spike/verify/ep01-mcp-tax/runs/slash-doctor.json`，2026-08-20 实跑）：Detail table 有 **Scope 列**，出现的值：`user`（plugin/skill/MCP server）、`this session only`（svc00–11 临时 server）、`user, every project`（~/.claude/CLAUDE.md）。scope 直接参与判断：session 级 server 被排除在持久成本外（"gone when the session ends"、"Of your *persistent* setup…"）。
- **prompt 层面的 scope 运用**（本机二进制提取）：① 表格 schema 明确要求 Scope 列（"| Component | Type | Scope | Uses (total since install) | …"）；② disable 落点按 skill 来源分流——"Skill: `skillOverrides` … in `.claude/settings.local.json` (project skill) or `~/.claude/settings.json` (skill from `~/.claude/skills/`)"；③ MCP 建议 `/mcp disable` 并强制提示 per-project 语义（"say so in the proposal… advise repeating `/mcp disable` in any other project"）；④ plugin 的 `false` 落点遵守 settings 优先级（user < project < local，写错层会被静默覆盖）。
- **边界（诚实标注）**：doctor 的动作空间是 disable / remove / dedup / trim / migrate-to-lazy，**没有"把 user 级的项目专用项移到项目级"的建议类型**；唯一的"归位式"动作是上述 CLAUDE.md→`.claude/skills/` 迁移。实跑输出中 gitnexus（user 级 MCP）的建议是 remove/disable 而非迁移到项目级。课程的"归位"解法超出 /doctor 现动作空间，是增量价值而非复述。

### 对 ep01"归位"解法的可行性判断

**可行，且有大半官方机制背书，但课程提炼的规则本身不是官方原文。** 具体来说：

1. **可以直接讲"官方默认就是项目级"**：`claude mcp add` 默认 local scope 是一手可验证事实（`--help` 即可现场演示），把"归位"从"最佳实践观点"升级为"对齐默认行为"，这是本解法最硬的一句。
2. **团队共享路径成熟**：`.mcp.json` 签入 + 首次使用审批 + `reset-project-choices`，官方文档完整；可直接演示。
3. **必须讲的三个坑**（均有一手出处）：①同名 skill personal 压 project，归位后必须删全局同名项；②`.mcp.json` 有审批而 `.claude/skills/` 无审批、且其 `allowed-tools` 在 untrusted 仓库也生效——团队签入 skill 的信任模型比 MCP 松，官方原话要求 review checked-in skill 的 allowed-tools；③`claude mcp remove` 会清 OAuth token，迁移带认证的 server 要预告重登录成本。
4. **表述边界**：官方文档没有"别把项目专用东西装全局"的逐字表述（【未找到】），课程措辞应为"官方机制的设计取向 + doctor 内置的迁移逻辑都指向同一原则"，而非"官方建议"。/doctor 不会主动提归位建议——这恰是课程相对工具的信息差。
