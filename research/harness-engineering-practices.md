# Harness Engineering（脚手架工程）实践深度调研报告

> 调研时间：2026-08-05。用途：为面向一线工程师的 5 分钟短视频培训系列（6–7 期）选题。
> 受众前提：每天都在用 Claude Code / Cursor / Codex 等现成 harness 产品；已上过上下文工程课。
> 因此本报告**不解释"什么是 harness"、不讲从零写 agent loop、不展开 compaction/记忆管理**（仅在边界处标注）。
> 标注约定：✅ = 一手来源已实际抓取验证；⚠️ = 一手来源存在但日期/细节未完全核实；❌ = 未找到一手来源（传言未证实）。

---

## TL;DR（10 条要点）

1. **Harness engineering 已成为 2025 底–2026 年的正式术语**：OpenAI（2026-02）、Thoughtworks 技术雷达 Vol.34（2026-04）、martinfowler.com（2026-04）均有专文；核心公式是 **Agent = Model + Harness**，"不错的模型 + 好的 harness > 顶级模型 + 差的 harness"（Addy Osmani）✅
2. **Claude Code 的扩展面在 2025 下半年爆发**：hooks（2025-06，v1.0.38）、自定义 subagents（v1.0.60）、插件系统（2025-10，v2.0.12）、Agent Skills（2025-10，v2.0.20）、MCP tool search（2025-12，v2.1.7）——这些是一线工程师拉开效果差距的主战场 ✅
3. **官方口诀：事实放 CLAUDE.md、流程放 Skill、强制约束放 Hook**。CLAUDE.md 是"建议性"的，hooks 是确定性的（在固定生命周期点执行的 shell 命令，不依赖模型自觉）✅
4. **CLAUDE.md 的最佳实践是"少即是多"**：官方给出删减判据"删掉这条会导致 Claude 犯错吗？"；OpenAI 的百万行实验用 ~100 行 AGENTS.md 当"地图"指向结构化 docs/，而非千页说明书 ✅
5. **Hooks 是最被低估的能力**：30 个生命周期事件（PreToolUse、PostToolUse、UserPromptSubmit、SessionStart、Stop、SubagentStop、PreCompact……），可阻塞、可改写工具入参/结果、可注入上下文；Cursor（2025-09）和 Codex（2026）也跟进推出了兼容协议 ✅
6. **长任务 harness 的官方范式是"交接文档工程"**：Anthropic 的 initializer/coding agent 两段式 + JSON feature list + progress 文件 + git history 三件套；社区的 Ralph loop（`while :; do cat PROMPT.md | claude; done`）是同一思路的极简版，已被 Anthropic 做成官方插件 ✅
7. **Compound engineering（Every）是"让 harness 越用越聪明"的闭环方法论**：Plan → Work → Review → Compound，把工作教训沉淀回 CLAUDE.md/docs/review agents；配套插件 2.4 万 stars ✅
8. **Evals 驱动 harness 迭代是前沿**：Anthropic《Demystifying evals for AI agents》（2026-01）区分 capability evals（低通过率起步）与 regression evals（接近 100%）；社区已有用 promptfoo 给 agent prompt 集跑 eval 的真实案例（一次 $0.05）✅
9. **沙箱化执行成为默认推荐**：Anthropic 官方沙箱（bubblewrap/seatbelt，减少 84% 权限弹窗）、Codex 三档 sandbox_mode、Thoughtworks 雷达 Trial 级条目——解决"审批疲劳反而不安全"的问题 ✅
10. **多 agent 编排分化明显**：官方走 worktree + subagents + agent teams 路线；社区爆品是 obra/superpowers（26.7 万 stars 的 skills 库）、Claude Squad（worktree 并行 TUI）、Gas Town/Beads（Steve Yegge 的 agent 舰队管理，理念激进但 Beads 已解决真痛点）✅

---

## 1. Claude Code 的可配置点（重点）

> 主要一手来源：官方文档站 `docs.claude.com/en/docs/claude-code/*`（与 `code.claude.com/docs` 同内容）+ 官方 CHANGELOG（`raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md`）。版本号均有 changelog 佐证；当前最新 v2.1.222（2026-08）。

### 1.1 CLAUDE.md 与 `.claude/rules/`

- **是什么**：持久指令文件体系。四层加载：managed policy（`/etc/claude-code/CLAUDE.md` 等）→ user（`~/.claude/CLAUDE.md`）→ project（`./CLAUDE.md`）→ local（`CLAUDE.local.md`，建议 gitignore）。从 cwd 向上走目录树拼接（不覆盖），子目录 CLAUDE.md 按需懒加载。
- **import 语法** `@path/to/file`：相对包含文件解析、递归最深 4 跳；可用 `@AGENTS.md` 复用 AGENTS.md（v0.2.107 引入）。
- **`.claude/rules/*.md`**：主题化规则文件，frontmatter `paths:` 支持 glob（`**/*.ts`、`src/*.{ts,tsx}`），实现路径作用域规则——只在读相关文件时注入。
- **官方最佳实践**：每个文件 <200 行；指令要具体可验证；矛盾规则会被任意取舍；**"强制类约束应用 PreToolUse hook 而非 CLAUDE.md"**（官方原话："Hooks execute as shell commands at fixed lifecycle events and apply regardless of what Claude decides"）。
- 来源：✅ https://docs.claude.com/en/docs/claude-code/memory ；✅ https://www.anthropic.com/engineering/claude-code-best-practices
- **培训价值**：CLAUDE.md 分层 + import + rules 路径作用域是"配置入门第一课"，人人可立即上手。

### 1.2 Hooks（重头戏）

- **是什么**：在 agent 生命周期固定点执行的用户定义 handler（shell 命令 / HTTP 端点 / LLM prompt / MCP 工具调用 / subagent），配置于 settings.json 的 `hooks` 键。
- **事件面**：当前共 **30 个事件**——SessionStart、UserPromptSubmit、PreToolUse、PermissionRequest、PermissionDenied、PostToolUse、PostToolUseFailure、Notification、SubagentStart、SubagentStop、Stop、PreCompact、PostCompact、SessionEnd、ConfigChange、FileChanged、WorktreeCreate/Remove、TaskCreated/Completed、TeammateIdle 等。
- **通信协议**：exit 0 + stdout JSON；**exit 2 = 阻塞**（stderr 回给 Claude）；PreToolUse 的 `hookSpecificOutput.permissionDecision` 支持 allow/deny/ask/defer；`updatedInput` 可改写工具入参；PostToolUse `updatedToolOutput` 可改写工具结果；`additionalContext` 注入上下文（出现在 system reminder）。
- **阻塞能力分级**：PreToolUse 可拦工具、UserPromptSubmit 可拦提示词、Stop/SubagentStop 可强制继续（Ralph 插件的原理）、PreCompact 可拦压缩；SessionStart 只能注入上下文。
- **时间线**（changelog 佐证）：首发 v1.0.38（2025 夏）→ PreCompact v1.0.48 → UserPromptSubmit v1.0.54 → SessionStart v1.0.62 → HTTP hooks v2.1.63 → `if` 条件 v2.1.85 → defer 决策 v2.1.89 → PreCompact 阻塞 v2.1.105。
- **真实用法**：强制 lint/format（PostToolUse）、保护敏感文件（PreToolUse deny）、会话开始注入 git 状态（SessionStart）、阻止 agent 提前宣布完成（Stop）、企业合规审计（http hook 上报）。
- 来源：✅ https://docs.claude.com/en/docs/claude-code/hooks
- **培训价值**：hooks 是"建议"与"强制"的分水岭，是拉开效果差距最大的单点能力，且概念可迁移到 Cursor/Codex。

### 1.3 Subagents

- **是什么**：独立上下文窗口的专门化 agent，`.claude/agents/*.md` + YAML frontmatter 定义。
- **关键字段**：`name`/`description`（唯二必填）、`tools`/`disallowedTools`、`model`（可路由到 haiku 省钱）、`permissionMode`、`maxTurns`、`skills`（启动预加载）、`hooks`（生命周期作用域）、`memory`（持久记忆目录）、`isolation: worktree`（v2.1.49）。
- **内置**：Explore（只读调查，v2.0.17）、Plan（v2.0.28）、general-purpose。嵌套深度默认 3，并发上限默认 20（v2.1.217+）。
- **组合玩法**：settings hooks 在 subagent 内同样触发（输入带 `agent_id`/`agent_type`）；`context: fork` 的 skill 在隔离 subagent 中运行。
- 来源：✅ https://docs.claude.com/en/docs/claude-code/sub-agents
- **培训价值**：教会"高产出调查任务丢给 subagent、主上下文保持干净"是现成产品内立竿见影的实践（与 context engineering 有边界重叠，讲分工而非讲窗口管理）。

### 1.4 Skills（Agent Skills）

- **是什么**：`SKILL.md` + 支撑文件的目录，遵循 Agent Skills 开放标准；progressive disclosure——只有 name+description 常驻上下文，正文调用时才注入，支撑文件按需读取。
- **位置**：`~/.claude/skills/`（personal）、`.claude/skills/`（project）、插件 `skills/`、enterprise managed。
- **关键 frontmatter**：`allowed-tools`（调用当轮免审批）、`disable-model-invocation`（仅人可调）、`user-invocable: false`（仅模型可调）、`context: fork`（v2.1.0，forked subagent 中运行）、`hooks`、`paths`（glob 限定激活）。
- **动态注入**：`` !`command` `` 在 skill 加载前执行 shell 并把输出插入正文；`$ARGUMENTS`/`${CLAUDE_SKILL_DIR}` 等变量。
- **与 subagent 的区别**：skill 是可复用的提示词/工作流（默认在主会话上下文运行），subagent 是隔离执行环境。
- **推出时间**：2025-10-16 官宣（✅ https://www.anthropic.com/news/skills 、✅ https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills ）；Claude Code 支持 v2.0.20。"GA"明确日期 ❌ 未找到一手来源（API release notes 仅见 beta 头 `skills-2025-10-02`）。
- 来源：✅ https://docs.claude.com/en/docs/claude-code/skills
- **培训价值**：Skills 是 2026 年跨工具（Claude Code/Cursor/Codex 均支持同一 SKILL.md 格式）的"工作流封装"标准，极适合单开一期。

### 1.5 Permissions 与 settings 分层

- **规则语法**：`Tool(specifier)`，如 `Bash(npm run build)`、`Bash(ls *)`、`Edit(src/**)`、`WebFetch(domain:example.com)`、`mcp__puppeteer__*`、`Agent(Explore)`。评估顺序恒定 **deny → ask → allow**，deny 无豁免。
- **settings 层级**：managed（最高，不可覆盖）> CLI 参数 > local（`settings.local.json`）> project（`settings.json`，入 git 团队共享）> user。权限规则跨层合并 + deny 优先。
- **Permission modes**：default / acceptEdits / plan / **auto（后台分类器自动批准，2026 新能力）** / dontAsk / bypassPermissions。
- **企业管理**：managed settings 三通道（server-managed、MDM、文件 `/etc/claude-code/managed-settings.json` + `managed-settings.d/`）；`allowManagedHooksOnly`、`strictKnownMarketplaces` 等 managed-only 键。
- 来源：✅ https://docs.claude.com/en/docs/claude-code/permissions 、✅ https://docs.claude.com/en/docs/claude-code/settings
- **培训价值**："把项目级 allow 清单和 hooks 提交进 git"是团队落地的第一步，5 分钟讲得完。

### 1.6 MCP 集成与 tool search

- **配置**：`claude mcp add` CLI；三 scope（local/project `.mcp.json` 入版本控制/user）；传输 http/sse/stdio/ws；OAuth 自动 refresh（`claude mcp login`，v2.1.186）。
- **MCP tool search（2025 年底大招）**：默认开启——MCP 工具过多时不再全量注入上下文，Claude 用 ToolSearch 工具按需发现（v2.1.7，2025-12）。对应 API 侧 Tool Search Tool（✅ https://www.anthropic.com/engineering/advanced-tool-use ，beta 2025-11-20）。
- 来源：✅ https://docs.claude.com/en/docs/claude-code/mcp
- **培训价值**：tool search 解决"MCP 装多了上下文爆炸"的真实痛点（与 context engineering 略有边界，讲配置开关即可）。

### 1.7 Output styles / Statusline / Worktree / Headless / SDK / Plugins

- **Output styles**（v1.0.81）：markdown+frontmatter 直接改系统提示，内置 Explanatory/Learning；曾 deprecated 又因社区反馈恢复。✅ https://docs.claude.com/en/docs/claude-code/output-styles
- **Statusline**（v1.0.71）：`statusLine` 配置自定义命令，stdin 收 JSON（model、cost、context_window、rate_limits 等）打印状态栏。✅ https://docs.claude.com/en/docs/claude-code/statusline
- **Worktree**（v2.1.49+）：`claude -w`、subagent `isolation: worktree`、WorktreeCreate/Remove hooks——并行多任务互不干扰的基础设施。独立文档页 ❌ 未找到（散见 hooks/subagents 页）。
- **Headless**（`claude -p`）：`--output-format json|stream-json`、`--json-schema` 结构化输出、`--bare`（CI 用，跳过 hooks/skills/MCP/CLAUDE.md 自发现）。用于 pre-commit、CI、fan-out 批处理。✅ https://docs.claude.com/en/docs/claude-code/headless
- **Agent SDK**：2025-09 由 Claude Code SDK 更名（v2.0.0），Python/TS 包复用同一 agent loop。⚠️ https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk （2025-09，标题日期经搜索确认）
- **Plugins / marketplace（2025-10，v2.0.12）**：插件 = 打包 skills+agents+hooks+MCP+output styles 的分发单位；`/plugin marketplace add user/repo`；官方市场 `anthropics/claude-plugins-official`。**官方定位："plugins will be our standard way to bundle and share Claude Code customizations"**。✅ https://docs.claude.com/en/docs/claude-code/plugins 、⚠️ https://www.anthropic.com/news/claude-code-plugins （public beta 公告，页面无日期，第三方引 2025-10-09）
- **培训价值**：plugins 是团队分发 harness 的"包管理器"，与 1.5 合并可讲"团队共享"一期。

### 1.8 官方组合玩法（文档明确支持）

- hooks + subagents：hook 输入带 agent_id/agent_type，可按 subagent 类型 matcher
- skills + subagents：`context: fork` + `agent: Explore` = skill 驱动只读 subagent
- permissions + hooks：官方推荐"allow 全部 Bash + PreToolUse hook 拦截少数"
- 官方原则：**事实放 CLAUDE.md、流程放 skill、强制约束放 hook**
- 企业：managed settings 可把全部扩展面锁定为仅托管来源（合规场景）

---

## 2. Cursor 的对应机制

> 一手来源：cursor.com/docs、cursor.com/changelog。深度从简，重点是与 Claude Code 的异同。

### 2.1 Rules（`.cursor/rules/*.mdc`）

- 项目规则必须是 `.mdc` 扩展名；四种触发类型由 frontmatter（`description`/`globs`/`alwaysApply`）组合决定：Always Apply / Auto Attached（按文件 glob）/ Agent Requested（按描述判断）/ Manual（@提及）。
- 优先级：团队规则（Team 仪表盘下发，Cursor 1.7，2025-09-29）→ 项目规则 → 用户规则。官方建议单条 ≤500 行。
- **原生支持 AGENTS.md**（根+子目录），定位为 rules 的"简单替代"（文档未提"迁移"路径）。
- 来源：✅ https://cursor.com/docs/rules
- **培训价值**：Cursor 的四型触发模型比 Claude Code 的"全量/路径"更细，适合做横向对比教学。

### 2.2 Hooks

- **推出时间：Cursor 1.7，2025-09-29（beta）**——注意早于任务书假设的"2025 年底"；Cursor 2.4（2026-01-22）大幅扩充并在 CLI 中兼容 Claude Code hooks。
- 配置：`.cursor/hooks.json`（项目）/`~/.cursor/hooks.json`（用户）/ Enterprise（MDM）/ Team（云端）。
- 事件分三类：Agent 钩子（`sessionStart`、`preToolUse`、`postToolUse`、`beforeSubmitPrompt`、`preCompact`、`stop`、`subagentStart/Stop`、`beforeShellExecution` 等）、Tab 补全钩子（Cursor 独有）、应用生命周期（`workspaceOpen`）。
- **与 Claude Code 的关键异同**：stdio JSON 协议一致、**exit 2 阻止的语义一致（官方明示为兼容设计）**、环境变量提供 `CLAUDE_PROJECT_DIR` 别名；事件命名 camelCase vs PascalCase；Cursor 独有 prompt 型钩子（LLM 判定）与 Tab 钩子；`stop` 钩子的 `loop_limit` 默认 5（Claude Code 默认不限）；Cursor 云端 agent 可跑项目级命令型 hooks。
- 来源：✅ https://cursor.com/changelog/1-7 、✅ https://cursor.com/docs/hooks
- **培训价值**："hooks 概念跨工具可迁移"本身就是一个重要认知——学一次，处处可用。

### 2.3 其他

- **MCP**：`.cursor/mcp.json`，`mcpServers` 顶层键与 Claude Code 格式兼容；三种传输 + OAuth；工具默认每次需批准，可开 auto-run。✅ https://cursor.com/docs/context/mcp
- **Subagents**：Cursor 2.4（2026-01-22）推出，`.cursor/agents/<name>.md` + frontmatter（`name`/`description`/`model`/`readonly`/`is_background`），与 Claude Code 几乎一一对应。✅ https://cursor.com/changelog/2-4 、✅ https://cursor.com/docs/agent/subagents
- **Skills**：2.4 引入，同一 SKILL.md 开放格式。✅ cursor.com/changelog/2-4
- **自定义模式（Custom Modes）已在 Cursor 2.1（2025-11）移除**（⚠️ 仅官方论坛用户帖佐证，changelog 未明示）——生态位由 subagents+skills 接替；说明各家都在向"subagents + skills + hooks"三件套收敛。
- **Commands**：1.6 引入 `.cursor/commands/*.md`，2.4 后与 Skills 合流（现行独立文档页 ❌ 未找到一手 URL）。

### 2.4 机制对应速查表

| 机制 | Claude Code | Cursor |
|---|---|---|
| 项目指令 | `CLAUDE.md` + `.claude/rules/`（paths glob） | `.cursor/rules/*.mdc`（四型触发）+ 原生 AGENTS.md |
| Hooks | settings.json，PascalCase，30 事件 | `.cursor/hooks.json`，camelCase，协议兼容 + Tab/prompt 型独有 |
| Subagents | `.claude/agents/*.md` | `.cursor/agents/*.md`（2.4+） |
| Skills | SKILL.md（2025-10） | 同格式（2.4+，2026-01） |
| MCP | `.mcp.json` / `claude mcp add` | `.cursor/mcp.json`（格式兼容） |
| 权限 | allow/ask/deny 规则 + sandbox | auto-run allowlist + 沙箱终端 |
| AGENTS.md | 不原生读，需 `@AGENTS.md` 导入 | 原生读 |

---

## 3. Codex CLI / AGENTS.md 生态

> 一手来源：agents.md、developers.openai.com/codex、GitHub openai/codex。

### 3.1 AGENTS.md 标准

- **是什么**：仓库根部的纯 Markdown 项目说明书（"README for agents"）。官网原文：由 OpenAI Codex、Amp、Google Jules、Cursor、Factory 等**多方协作产生**，现由 Linux Foundation 旗下 Agentic AI Foundation 托管；60k+ 开源项目采用。（⚠️ "由 Sourcegraph 发起"的说法仅见二手来源，与官网冲突，以官网为准。）
- 无必填字段；冲突时"离被编辑文件最近的 AGENTS.md 优先，用户显式 prompt 最高"；OpenAI 主 repo 有 88 个嵌套 AGENTS.md。
- **Claude Code 原生不读 AGENTS.md**——官方 memory 文档明示 "Claude Code reads CLAUDE.md, not AGENTS.md"，建议 `@AGENTS.md` 导入或符号链接。网上"AGENTS.md fallback"说法不属实。
- 来源：✅ https://agents.md/ 、✅ https://code.claude.com/docs/en/memory
- **培训价值**：AGENTS.md 是跨工具最大公约数，"一份指令三处可用"（Cursor 原生、Codex 原生、Claude Code 用 import）是实用的团队策略。

### 3.2 Codex config.toml 与 sandbox/approval

- **config.toml 分层**：`/etc/codex/config.toml`（系统）→ `~/.codex/config.toml`（用户）→ profiles（`~/.codex/<name>.config.toml`）→ 项目 `.codex/config.toml`（仅信任项目）→ CLI flags（最高）。
- **两个独立旋钮**：`sandbox_mode`（技术上能做什么：read-only / workspace-write / danger-full-access；OS 级 seatbelt/bwrap 强制）× `approval_policy`（何时停下来问：untrusted / on-request / never 等）。这与 Claude Code 的 permission rules + modes 思路对应但实现更偏 OS 强制。
- 企业可用 `requirements.toml` 强制约束（如禁用 `approval_policy="never"`）。
- **AGENTS.md 加载链**：全局 `~/.codex/AGENTS.md`（或 `AGENTS.override.md`）→ 项目根到 cwd 逐级拼接，越近 cwd 优先级越高；上限默认 32KiB。
- **2026 新增**：hooks（`[features] hooks`，hooks.json）、multi_agent、memories（实验性）、`approvals_reviewer = "auto_review"`（reviewer agent 自动审批）。
- 来源：✅ https://developers.openai.com/codex/config-basic 、✅ https://developers.openai.com/codex/agent-approvals-security 、✅ https://developers.openai.com/codex/guides/agents-md
- **培训价值**：sandbox×approval 双旋钮模型是讲"安全放手让 agent 跑"的最清晰框架。

### 3.3 Codex 2025–2026 时间线（一手）

- 2025-04 CLI 开源 → 2025-05 Codex cloud → 2025-09-15 GPT-5-Codex + IDE 扩展 → 2025-10-06（DevDay）GA + Slack 集成 + SDK（✅ https://openai.com/index/introducing-upgrades-to-codex/ ）
- `codex exec` headless：默认 read-only、`--sandbox workspace-write`、`--json` 事件流、`--output-schema` 结构化输出（✅ https://raw.githubusercontent.com/openai/codex/main/docs/exec.md ）

---

## 4. 官方工程博客的 harness 方法论（逐篇精读）

### 4.1 Anthropic: Effective harnesses for long-running agents ✅

- 2025-11-26，Justin Young。https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- 核心：长任务两大失败模式——(a) 一次做完导致上下文耗尽留半成品；(b) 后期 agent 看到已有进展就"宣布胜利"。
- 方案：**initializer agent + coding agent 两段式**（同一 harness，仅初始 prompt 不同）。交接三件套：**JSON feature list（200+ 条，模型更不容易乱改 JSON）+ claude-progress.txt + git history**；每 session 固定开场仪式（pwd → git log → progress → 跑 init.sh → 先端到端验证再开发）。
- 关键金句：harness 的灵感来自"高效软件工程师每天做的事"——交接文档、干净提交、先验证再开发。
- **培训价值**：这是"harness 工程化"最完整的一篇官方叙事，适合作为系列开篇的方法论锚点（注意：其中"跨 session 记忆"部分与 context engineering 有边界，聚焦"文件即接口"即可）。

### 4.2 Anthropic: Harness design for long-running application development ✅

- 2026-03-24，Prithvi Rajasekaran。https://www.anthropic.com/engineering/harness-design-long-running-apps
- 核心：两个 compaction 治不好的失败模式——**context anxiety**（接近上下文极限时提前收尾；解药是 context reset：全新 agent + 结构化 handoff）和 **self-evaluation 失灵**（agent 评价自己产出时"自信地夸赞平庸之作"）。
- 借鉴 GAN：**generator/evaluator 分工**——"把一个独立 evaluator 调成怀疑论者，远比让 generator 自我批评可行"；evaluator 用 Playwright MCP 像用户一样点应用，按 sprint contract 四条标准打分。
- **最重要的一课**："harness 的每个组件都编码了一个'模型做不到什么'的假设，而这些假设会随着模型进步迅速过期"——Opus 4.6 发布后他们删掉了 sprint 分解和 context reset。成本实诚对比：solo 20 分钟 $9 vs 完整 harness 6 小时 $200。
- **培训价值**："harness 要随模型能力减重"是反直觉的高级认知，适合高阶一期。

### 4.3 Anthropic: Demystifying evals for AI agents ✅

- 2026-01-09。https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
- 核心：术语体系（task/trial/grader/transcript/**outcome**/evaluation harness vs agent harness）；**三类 grader**（code-based / model-based / human）；**capability evals（从低通过率起步爬坡）vs regression evals（接近 100% 防倒退）**，前者通过率高可"毕业"为后者；pass@k vs pass^k；"评估 agent = 评估 harness + model 的整体"。
- **培训价值**：evals 是"harness 改动有没有效"的度量手段——没有 evals，调 harness 就是玄学。适合作为系列收尾。

### 4.4 Anthropic: Claude Code best practices ✅（持续更新版）

- 原版 2025-04，页面已更新至 2026 版。https://www.anthropic.com/engineering/claude-code-best-practices
- 核心：CLAUDE.md 删减判据（"删掉这条会导致 Claude 犯错吗？否则删掉"）；**审批疲劳**（"批到第十次你其实已经不在 review 了"）→ auto mode / allowlist / sandbox 三条出路；**hooks 是确定性保证**（"Unlike CLAUDE.md instructions which are advisory, hooks are deterministic"）；验证闭环（"给 Claude 一个它能自己跑的检查，否则你就成了人肉验证循环"）；Writer/Reviewer 双 session、fan-out headless 批处理。
- **培训价值**：include/exclude 清单和失败模式列表可直接做讲义。

### 4.5 Anthropic: Beyond permission prompts: Claude Code sandboxing ✅

- 2025-10-20，David Dworken & Oliver Weller-Davies。https://www.anthropic.com/engineering/claude-code-sandboxing
- 核心：权限弹窗 → 审批疲劳 → 反而不安全；沙箱减少 **84%** 权限提示；**文件系统隔离 + 网络隔离缺一不可**；基于 bubblewrap/seatbelt，已开源 sandbox runtime。
- **边界标注**（不展开）：Effective context engineering for AI agents（2025-09，https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents ）属 context engineering 课程范围。

### 4.6 OpenAI: Harness engineering: leveraging Codex in an agent-first world ✅

- 2026-02-11，Ryan Lopopolo。https://openai.com/index/harness-engineering/
- 实验事实：空 repo 起步 5 个月、3 名工程师、**约 1,500 个 PR、约 100 万行代码、0 行手写**；"Humans steer. Agents execute."
- 工程师角色重定义："主要工作不再是写代码，而是**设计环境、明确意图、构建反馈回路**"。
- 三大可操作实践：
  1. **AGENTS.md 是地图不是说明书**：~100 行指向结构化 docs/（design-docs、exec-plans、QUALITY_SCORE.md……），渐进式披露；巨型指令文件四大问题：挤占上下文、全部重要=都不重要、立即腐烂、无法机械校验。
  2. **知识库要机械维护**：专门 linter + CI 校验文档新鲜度/交叉链接；定期 **"doc-gardening" agent** 扫描过时文档自动发修复 PR；分层架构用自定义 linter/结构测试强制。
  3. **为 agent 可读性优化应用本身**：git worktree 起实例、接 Chrome DevTools Protocol 让 Codex 复现 bug、临时可观测性栈让 agent 用 LogQL/PromQL 查日志。
- **培训价值**："地图 vs 说明书"与 Anthropic 的精简原则互相印证；doc-gardening 是极具体的可照搬实践。

### 4.7 Thoughtworks Technology Radar Vol.34（2026-04）✅（注意表述精度）

- https://www.thoughtworks.com/en-us/radar
- **没有名为 "harness engineering" 的独立条目**——概念散见于：Context engineering（Adopt）、**Curated shared instructions for software teams**（Adopt：CLAUDE.md/AGENTS.md 放进服务模板分发）、**Feedback sensors for coding agents**（Trial：编译器/linter/结构测试接入 agent 工作流）、**Sandboxed execution for coding agents**（Trial："sensible default"）、**Feedback flywheel**（Assess）、**Agent instruction bloat**（Caution：手写 AGENTS.md 往往比 LLM 生成的更有效）、Ralph loop（Assess）。
- 另有播客 What is harness engineering?（2026-05-14，Birgitta Böckeler）✅ https://www.thoughtworks.com/en-us/insights/podcasts/technology-podcasts/what-harness-engineering
- **培训价值**：雷达条目给"该采用什么"提供了权威背书排序（Adopt vs Trial vs Assess）。

### 4.8 martinfowler.com: Harness engineering for coding agent users ✅

- 2026-04-02，Birgitta Böckeler。https://martinfowler.com/articles/harness-engineering.html
- 核心框架：**Guides（前馈：行动前引导，如 rules/CLAUDE.md）× Sensors（反馈：行动后观察，如 linter/测试/hooks）**，再正交叠加 **Computational（确定性、毫秒级）× Inferential（LLM 判断、慢而贵）** 2×2 矩阵。
- 金句：sensors 输出应"optimized for LLM consumption"（带修正指令的 linter 消息 = "a positive kind of prompt injection"）；只有反馈→重复犯错，只有前馈→永远不知道规则是否生效。
- **Steering loop**：人类的工作是迭代 harness——同一问题出现多次就改进 guides/sensors。**Harnessability** 是代码库属性（强类型、清晰模块边界天然可驾驭）；"harness is most needed where it is hardest to build"（遗留系统）。
- **培训价值**：这个 2×2 矩阵是目前最系统的 harness 分类法，可作为整个培训系列的理论主框架。

---

## 5. 社区前沿实践（均已核实一手来源）

### 5.1 Compound Engineering（Every）✅

- **是什么**：Kieran Klaassen 与 Dan Shipper 提出的四步循环 **Plan → Work → Review → Compound**；关键在第四步——把每次工作的教训 codify 回 CLAUDE.md、`docs/solutions/`、review agents 和 skills，"每个工程单元应让后续工作更容易"。
- 一手来源：✅ https://every.to/guides/compound-engineering ；配套插件 ✅ https://github.com/EveryInc/compound-engineering-plugin （**23,998 stars**，2026-08-05 实测；26 个 agent、23 个 workflow 命令、14 个 review 专家）。
- **为什么火**：把"AGENTS.md 越用越聪明"封装成完整方法论 + 即装即用插件。
- **培训价值**：给"为什么要写 rules"一个复利叙事，适合与 evals 搭配讲"harness 迭代闭环"。

### 5.2 Ralph loop（Geoffrey Huntley）✅

- **是什么**：2025-07-14 提出，一句话："Ralph is a Bash loop"——`while :; do cat PROMPT.md | claude-code ; done`。每轮全新 context 反复喂同一 prompt，记忆靠 git history + 进度文件；每轮只做一件事。名言："deterministically bad in an undeterministic world"。
- 一手来源：✅ https://ghuntley.com/ralph/ ；最流行实现 snarktank/ralph（**21,391 stars**）；**Anthropic 已出官方插件 ralph-wiggum**（✅ anthropics/claude-code 仓库 plugins/ 目录，用 Stop hook 拦截退出重喂 prompt；⚠️ 有批评认为官方插件丢失"每轮全新 context"精髓，https://www.aihero.dev/why-the-anthropic-ralph-plugin-sucks ）。
- Thoughtworks 雷达 Vol.34 列为 Assess。
- **培训价值**：5 分钟讲透一个"无人值守长任务原语"，演示效果极强，天然适合短视频。

### 5.3 Harness as code：配置入仓、团队共享 ✅

- **是什么**：把 `.claude/settings.json`（permissions/hooks/plugins）、`CLAUDE.md`、`.claude/agents/`、`.mcp.json` 提交进 git，团队共享同一份 harness；个人偏好放 gitignore 的 `settings.local.json`。
- 一手来源：✅ 官方 settings 文档明确 Project scope = "committed to git, shared with team"（https://docs.claude.com/en/docs/claude-code/settings ）；理论框架：✅ Addy Osmani《Agent Harness Engineering》https://addyosmani.com/blog/agent-harness-engineering/ （"Agent = Model + Harness"、AGENTS.md 的 **ratchet 原则**：每行规则须可追溯到一次真实失败）。（⚠️ "harness engineering"一词由 Viv Trivedy 提出的说法为二手转述，原文未抓取。）
- **培训价值**：这是所有实践的团队落地形态，可作为"从个人技巧到团队资产"的一期。

### 5.4 多 agent 编排社区玩法 ✅（部分）

- **Claude Squad**：✅ https://github.com/smtg-ai/claude-squad （8,237 stars）——tmux + git worktrees 并行跑多个 Claude Code/Codex 实例的 TUI，简单实用派。
- **Ruflo（原 Claude Flow）**：✅ https://github.com/ruvnet/ruflo （67,090 stars，已改名）——hive-mind swarm 编排；体量大但工程严谨性有社区争议（二手转述）。
- **ccswarm ❌ 未证实**（未检索到可靠信息）。
- **培训价值**：官方 worktree + subagents 已够用，社区工具作为"延伸阅读"提及即可，不必单独成期。

### 5.5 Evals 驱动的 harness 迭代 ✅

- 官方工具链：✅ promptfoo《Evaluate Coding Agents》https://www.promptfoo.dev/docs/guides/evaluate-coding-agents/ ——支持 `anthropic:claude-agent-sdk`、`openai:codex-sdk` provider，trajectory 断言、cost 阈值、LLM-as-judge，"测的是系统（harness）不是模型"。
- 真实案例：✅ jonesrussell 为 agency-agents（184 个 agent prompt 集）做的 eval harness——每个 agent 当 system prompt 喂代表性任务，Haiku 做 judge 按 5 条 rubric 打分，一次约 $0.05，首轮即发现真实质量缺口，失败用例转回归测试（https://jonesrussell.github.io/blog/eval-harness-agency-agents/ ，2026-03）。
- "用 inspect_ai 测 CLAUDE.md/skills 的流行实例" ❌ 未找到一手来源。
- **培训价值**：回应"我改了 CLAUDE.md 到底有没有用"的灵魂拷问。

### 5.6 Beads 与 Gas Town（Steve Yegge）✅

- **Beads**：✅ https://github.com/gastownhall/beads （26,039 stars，2025-10 发布）——给 coding agent 的图结构 issue tracker（git 备份、Dolt 驱动），agent 跨 session 靠 `bd ready`/`bd create` 恢复上下文，解决 plan 散乱 markdown 化的真痛点。Yegge 原话："adderall for your coding agent"。
- **Gas Town**：✅ https://github.com/gastownhall/gastown （17,457 stars，2026-01-01 开源）——tmux 协调 20–30 个并行 agent 实例的 workspace manager（Mayor/Polecats/Witness/Refinery 等角色）；一手索引 ✅ https://yegge.ai/gastown.html 。热度证据：SE Daily、Pragmatic Engineer、Hanselminutes 播客（二手）；也有成本与术语争议。
- **培训价值**：Beads 值得单独演示（"给 agent 一个外部任务板"）；Gas Town 作为"激进前沿"提及，不建议照搬。

### 5.7 补充爆品 ✅

- **obra/superpowers**（Jesse Vincent）：✅ https://github.com/obra/superpowers （**267,097 stars**，当前 Claude Code 生态第一大社区项目）——TDD、调试、writing-plans、subagent-driven-development 等纪律性 skills 库；2026-01-15 收入 Anthropic 官方插件市场，跨 Claude Code/Cursor/Codex/OpenCode。
- **官方插件市场**：✅ https://github.com/anthropics/claude-plugins-official （33,112 stars，含 ralph-wiggum、feature-dev、code-review、hookify）。
- **awesome-claude-code**：✅ https://github.com/hesreallyhim/awesome-claude-code （51,709 stars）——观察社区玩法的最佳入口。
- **培训价值**：superpowers 是"skills 能干什么"的最佳展品。

---

## 6. 培训选题建议（5 分钟/期，6–7 期）

> 原则：每期一个可操作动作 + 一个认知升级；避开 context engineering 已覆盖内容（compaction、上下文窗口管理、记忆机制）；理论框架用 Böckeler 的 Guides×Sensors 矩阵（§4.8）贯穿全系列。

### 推荐 7 期编排

| 期 | 主题 | 核心内容 | 理论锚点 | 与 context engineering 的边界 |
|---|---|---|---|---|
| 1 | **CLAUDE.md 工程：地图而非说明书** | 四层加载、import 语法、`.claude/rules/` paths 作用域、删减判据、OpenAI 的 ~100 行地图模式、ratchet 原则 | Guides（前馈）；OpenAI §4.6 | ⚠️ "少写省 token"只点一句，不展开窗口经济学 |
| 2 | **Hooks：从建议到强制** | PreToolUse/PostToolUse/Stop 三事件实操（强制 lint、保护文件、防止提前收工）、exit 2 协议、Cursor/Codex 同款概念 | Sensors（反馈）；Anthropic "hooks are deterministic" §4.4 | 无重叠 |
| 3 | **Skills 与 Subagents：封装工作流** | SKILL.md 渐进披露、allowed-tools 免审批、`context: fork`；subagent 分工（Explore/调查外包）；superpowers 演示 | Guides 的可复用化；§1.3/1.4 | ⚠️ "subagent 省主上下文"只讲分工收益，不讲窗口管理 |
| 4 | **Permissions 与沙箱：放心放手** | allow/ask/deny 语法、项目 settings 入 git、auto mode、沙箱双隔离（文件+网络）、审批疲劳、Codex sandbox×approval 双旋钮 | Sensors 的"边界"；§4.5 | 无重叠 |
| 5 | **团队共享：Harness as Code** | 项目级 settings/CLAUDE.md/agents/MCP 入 git、plugins 市场分发、Cursor rules 团队下发、AGENTS.md 跨工具最大公约数 | Thoughtworks "Curated shared instructions"（Adopt）§4.7 | 无重叠 |
| 6 | **无人值守长任务：交接文档与 Ralph** | 交接三件套（feature list JSON + progress + git log）、每轮一件事的循环、Ralph loop 与官方 ralph-wiggum 插件、compound engineering 的"教训沉淀" | §4.1/4.2/5.1/5.2 | ⚠️ "跨 session 记忆"与 context engineering 相邻——聚焦"文件即接口"，不讲记忆管理 |
| 7 | **Evals：harness 改动有没有效** | grader 三型、capability vs regression、promptfoo 跑 agent eval 实操（$0.05 案例）、steering loop（问题复发→改 harness） | §4.3/4.8/5.5 | 无重叠 |

### 备选/加餐主题

- **MCP 与 tool search**（§1.6）：若受众 MCP 使用重，可替换第 3 期的 subagent 部分。
- **多 agent 编排前沿**（Claude Squad/Gas Town/Beads，§5.4/5.6）：适合做"番外篇"而非正片——理念炫但落地成本高。
- **headless/CI 集成**（`claude -p`、`codex exec`，§1.7/3.3）：适合面向平台/DevOps 观众的加餐。

### 明确不建议做的选题

- ❌ "什么是 harness / agent loop 原理"——受众每天用现成产品，无价值。
- ❌ compaction / 上下文窗口管理 / 记忆机制——context engineering 课程已覆盖（仅在第 1、3、6 期以一句话标注边界）。
- ❌ 从零写多 agent 框架——与"在现成产品之上做配置"的定位相悖。

---

## 附：未证实/存疑事项清单

| 事项 | 状态 | 说明 |
|---|---|---|
| Agent Skills "GA" 明确日期 | ❌ | API release notes 仅见 2025-10-16 beta 记录 |
| 插件公告博客精确日期 | ⚠️ | 页面无日期，第三方引 2025-10-09，changelog 对应 v2.0.12 |
| "A harness for every task"（官方 dynamic workflows 指南） | ⚠️ | docs overview 页有链接，正文未抓取；第三方称 2026-06 |
| Cursor 自定义模式移除的官方说明 | ⚠️ | 仅论坛用户帖，changelog 未明示 |
| Cursor slash commands 现行文档页 | ❌ | 旧 URL 跳转 Skills 页 |
| Codex `on-failure` approval 档 | ⚠️ | 本次抓取的官方安全页未列出（旧版文档有） |
| ccswarm 项目 | ❌ | 未检索到可靠信息 |
| inspect_ai 测 harness 配置的流行实例 | ❌ | 社区案例均基于 promptfoo |
| "harness engineering 一词由 Viv Trivedy 提出" | ⚠️ | 经 Addy Osmani 二手转述，原文未抓取 |
| AGENTS.md "由 Sourcegraph 发起" | ⚠️ | 与 agents.md 官网"多方协作"表述冲突，以官网为准 |

*（§1–§6 完成于第一轮调研。所有标注 ✅ 的 URL 均经调研代理实际 FetchURL 验证；star 数为 2026-08-05 GitHub API 实测。）*

---

## 7. 模型变强之后：harness 的消亡与幸存（第二轮调研，2026-08-05 追加）

> 问题：随着 2026 前沿模型能力增强，harness 的哪些部分正在被模型吸收而变得不必要，哪些反而更重要？
> 核心判据（Anthropic 官方原话）："every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions… can quickly go stale as models improve"；同时 "the space of interesting harness combinations doesn't shrink as models improve. Instead, it **moves**" —— harness 不归零，而是随模型代际迁移位置。✅ https://www.anthropic.com/engineering/harness-design-long-running-apps （2026-03-24）

### 7.0 官方与权威对"harness 厚薄"的三种立场（并存，不矛盾）

- **Anthropic（校准派）**：harness 设计空间不缩小而是"移动"；每次新模型发布都要对 harness 组件做减法压力测试。他们身体力行：Sonnet 4.5 需要 context resets → Opus 4.5 后不再必要 → Opus 4.6 后删除 sprint 分解（"the model could natively handle the job without this sort of decomposition"）。✅ 同上文
- **OpenAI 研究侧（吸收派）**：Noam Brown："people are building scaffolding on top of the reasoning models right now. But I think in many ways, those scaffolds will also just be replaced…"（含 model router）。⚠️ 经 swyx Latent Space 播客转引核实（https://www.latent.space/p/noam-brown ，2025-06-19），未追原始录音。
- **工程界（Build to Delete 派）**：Philipp Schmid（Google DeepMind）专章论述 "The 'Bitter Lesson' of building Agents"："To survive the Bitter Lesson, our infrastructure (Harness) must be lightweight… build harnesses that allow them to **rip out the 'smart' logic they wrote yesterday**." 案例：Manus 六个月重构 harness 五次；Vercel 砍掉 agent 80% 工具后步数/token/延迟全面改善（✅ https://vercel.com/blog/we-removed-80-percent-of-our-agents-tools ，2025-12）。✅ https://www.philschmid.de/agent-harness-2026 （2026-01-05）；姊妹篇 ✅ https://www.philschmid.de/context-engineering-part-2 （2025-12-04："The harness you build today will likely be obsolete when the next frontier model lands."）
- **量化背景（METR 时间线，一手已验证）**：agent 可完成任务长度全周期约 7 个月翻倍，但 **2023 年以来约 4.3 个月翻倍、2024 年以来约 3 个月**；Opus 4.5 的 50% 可靠性 horizon 已达 ~5.3 小时，2026 年 frontier 超 12 小时。**关键限定**：80% 可靠性 horizon 远短于 50%（2026 年初约 3–4h vs 16–20h），且任务集偏"干净、可自动评分"——真实工作更 messy。✅ https://metr.org/blog/2026-1-29-time-horizon-1-1/ （2026-01-29）、✅ https://metr.org/time-horizons

### 7.1 消亡中的组件清单（带证据）

1. **模拟推理的 prompt 技巧（CoT 咒语、角色扮演、情感勒索）**——OpenAI 官方推理模型指南："Avoid chain-of-thought prompts: Since these models perform reasoning internally, prompting them to 'think step by step'… is unnecessary"且"can sometimes hinder it"。✅ https://developers.openai.com/api/docs/guides/reasoning-best-practices ；Ethan Mollick：角色扮演"aren't magical… sometimes giving the AI a role can actually lower accuracy"。✅ https://www.oneusefulthing.org/p/getting-started-with-ai-good-enough
2. **任务分解脚手架（sprint 分解、initializer/coding 两段式、context reset 编排）**——Anthropic 官方三代谢了三代（见 §7.0）；Opus 4.6 发布语几乎是"吸收清单"："plans more carefully, sustains agentic tasks for longer… has better code review and debugging skills to catch its own mistakes"。✅ https://www.anthropic.com/news/claude-opus-4-6 （2026-02-05）；Sonnet 4.5 "maintaining focus for more than 30 hours"。✅ https://www.anthropic.com/news/claude-sonnet-4-5
3. **系统提示中的行为纠偏段落（官方 harness 自己在删）**——CHANGELOG 一手证据：**v2.1.154 "The lean system prompt is now the default for all models except Haiku, Sonnet, and Opus 4.7 and earlier"**（官方按模型代际切换系统提示厚薄，最直接的"吸收"承认）；v2.1.201 "Claude Sonnet 5 sessions no longer use the mid-conversation system role for harness reminders"；v2.1.126 Read 工具移除 malware-assessment 提醒（"spurious refusals… on legacy models"）。Anthropic 官方《The new rules of context engineering for Claude 5 generation models》（Thariq Shihipar）给出 Then→Now：给规则→给判断、给示例→设计接口、全部前置→progressive disclosure、CLAUDE.md 记忆→auto-memory，并喊话用户 "you may need to simplify just like we did"。✅ https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models ；✅ https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md
4. **CLAUDE.md 中"模型读代码就能知道"的内容**——官方 Exclude 清单："Anything Claude can figure out by reading code"、"Standard language conventions Claude already knows"、"Self-evident practices like 'write clean code'"。且已工具化：**v2.1.206 /doctor 自动提议修剪 CLAUDE.md**（砍掉 directory layouts、dependency lists、architecture overviews，保留 pitfalls、rationale、非默认 conventions）。实证：arXiv 2602.11988 系统性评测发现 context files 总体不提升任务成功率且推理成本 +20%，只有"非标准实践"类指令有用（✅ 摘要验证 https://arxiv.org/abs/2602.11988 ）。反例：有人删掉 CLAUDE.md 后"taste"类质量回退（⚠️ stackademic 实验文，搜索验证未抓全文）。
5. **手写规划提示词（"请先列计划再动手"类）**——规划已内置为 harness 一等原语（plan mode、TodoWrite、Plan subagent）；用户层不再需要咒语，但仍需决定"何时规划"。✅ https://www.anthropic.com/engineering/claude-code-best-practices （注意：TodoWrite/plan mode 本身**未被移除反而在增强**——v2.1.16 新 task management 系统——说明 Anthropic 判断这些结构仍是 load-bearing 的）
6. **配置向导类 UI**——最纯粹的"模型吸收 harness 功能"案例：v2.1.198 移除 /agents 交互式创建向导（"ask Claude to create or manage subagents"）；v2.0.70 移除 `#` 快捷记笔记入口（"tell Claude to edit your CLAUDE.md instead"）。自然语言取代了 GUI 流程。✅ CHANGELOG
7. **人工配置项（被默认化/分类器化吸收）**——MCP tool search 默认化（v2.1.7）、auto mode 免 opt-in（v2.1.152）、subagent 默认后台（v2.1.198）、thinking 默认开启（v2.0.67）、默认 effort 持续上调、1M 上下文默认化（v2.1.75）。✅ CHANGELOG
8. **model router / 手工模型路由**——Noam Brown 明确预测 router 会被统一模型取代（⚠️ 转引，同 §7.0）；Explore subagent 从"固定 Haiku"改为"继承主会话模型"（v2.1.198）是官方层面对路由决策的重估。✅ CHANGELOG
9. **胁迫式 skills/规则写法（"EXTREMELY-IMPORTANT" 大写块）**——最大社区 skills 库 superpowers 正在讨论为新模型"减重"（⚠️ open issue，✅ https://github.com/obra/superpowers/issues/2087 、https://github.com/obra/superpowers/issues/2017 ，2026-07/08）：把胁迫式 preamble 改为判断式表述，因官方指南报告 Claude 5 代"removed over 80% of Claude Code's system prompt… with no measurable loss"（该 80% 数字为 issue 转述，⚠️ 未逐字核实官方原文）。

**需要纠正的预期**：Geoffrey Huntley 本人**不认为**模型变强会杀死 Ralph loop——"ralph is about getting the most out of how the underlying models work through context engineering and that pattern is GENERIC"，模型变强恰是其愿景的条件（"What if the models don't stop getting good?"）。✅ https://ghuntley.com/loop/ （2026-01-17）。"Huntley 说 Ralph 会被淘汰"❌ 未证实（证据指向反面）。同理，"社区已大规模抛弃 subagents"❌ 未证实——现状是分化（Mario Zechner 的极简 agent Pi 明确不做 subagents，✅ https://mariozechner.at/posts/2025-11-30-pi-coding-agent/ ；而 OpenAI/Osmani 认为 maker/checker 分离更重要）。

### 7.2 幸存且升值的组件清单（带论据）

1. **验证/反馈信号（ground truth 在外部世界，不在权重里）**——最强幸存项。Anthropic："Give Claude a check it can run… Without a check it can run, 'looks done' is the only signal available, and you become the verification loop"。✅ https://www.anthropic.com/engineering/claude-code-best-practices ；Leonardo de Moura（Lean 作者）："As general-purpose AI improves, the bottleneck shifts entirely to the verification platform… The barrier to verified software is no longer AI capability. It is platform readiness." ✅ https://leodemoura.github.io/blog/2026-2-28-when-ai-writes-the-worlds-software-who-verifies-it/ （2026-02-28）；METR 的 50% vs 80% 可靠性鸿沟意味着可靠性工程长期是 harness 职责。✅ metr.org（同上）
2. **权限与安全边界（信任问题≠能力问题）**——模型再强也不该被信任。Simon Willison "lethal trifecta"：LLM"unable to reliably distinguish the importance of instructions based on where they came from"——架构性缺陷，"Guardrails won't protect you"，唯一可靠解是在 harness 层砍掉三要素之一。✅ https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/ ；OWASP LLM Top 10 榜首：prompt injection"is unclear if there are fool-proof methods of prevention"。✅ https://genai.owasp.org/llmrisk/llm01-prompt-injection/ ；Anthropic 沙箱："even a successful prompt injection is fully isolated"。✅ https://www.anthropic.com/engineering/claude-code-sandboxing ；Codex 官方把 sandbox 与 approval 作为能力之外的两个独立层。✅ https://developers.openai.com/codex/agent-approvals-security 。**反向收紧证据**：v2.1.183 auto mode 分类器对 `git reset --hard`/`terraform destroy` 硬性阻断；v2.1.215/218 收回模型自主触发 /verify、/code-review、/deep-research 的能力——模型越自主，官方越在装刹车。✅ CHANGELOG
3. **组织知识与意图的外化（AGENTS.md/docs 作为记录系统）**——模型权重里没有你公司的架构决策和"为什么"。Thoughtworks 雷达 Adopt："relying on individual developers to write prompts from scratch is emerging as an anti-pattern… treating AI guidance as a collaborative engineering asset"。✅ https://www.thoughtworks.com/en-us/radar/techniques/curated-shared-instructions-for-software-teams ；OpenAI 百万行实验的全部关键实践都是厚在**结构**而非字数（地图式 AGENTS.md + CI 校验 + doc-gardening）。✅ https://openai.com/index/harness-engineering/ 。**注意边界**：消亡的是"仓库概览/依赖列表"类内容（模型自己能读出来），幸存的是"pitfalls、rationale、非默认 conventions"（模型读不出来的）。
4. **Evals**——换模型时代的元能力。Anthropic："teams without evals face weeks of testing while competitors with evals can… upgrade in days"；"When we evaluate 'an agent,' we're evaluating the harness and the model working together"。✅ https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents 。且每次模型升级都要重新决定"harness 哪部分该删"——没有 evals，这个决定就是玄学（§7.0 的"减法压力测试"依赖它）。
5. **环境接入（MCP）**——"连接≠能力"：模型不知道你的内部系统。MCP 官方定位就是"connecting AI applications to external systems"。✅ https://modelcontextprotocol.io/introduction
6. **harness 的"可拆性"工程本身（新元能力）**——Schmid 的 "Build to Delete"；Boris Cherny 自述 Claude Code "rewritten from scratch probably every three to four weeks"（⚠️ 播客转引，✅ https://www.latent.space/p/ainews-is-harness-engineering-real ，2026-03-04）。
7. **编排层上移（loop/graph 工程）**——2026 年的收敛形态是"薄入口 + 厚结构"：prompt 层变薄，但 Steinberger "You should be designing loops that prompt your agents"、Cherny "My job is to write loops"（均 ✅ 经 https://addyosmani.com/blog/loop-engineering/ 2026-06-07 转引核实）；Osmani 的落点："Verification is still on you… Build the loop. Stay the engineer."

**两个重要的反向/复杂化证据**（避免一边倒叙事）：
- **Claude Code 源码泄露（2026-03-30/31）**：npm source map 失误泄露约 512,000 行 TypeScript——"最薄 wrapper"叙事受到嘲讽，泄露物显示庞大的记忆/多 agent/注入防御系统，成为 thick 派最强实物论据（⚠️ Ars Technica 论坛帖佐证存在性 ✅ https://arstechnica.com/civis/threads/entire-claude-code-cli-source-code-leaks-thanks-to-exposed-map-file.1512359/ ；技术解读见 Nemo Operans《A Structural Theory of Harnesses》✅ https://nemooperans.com/a-structural-theory-of-harnesses ，2026-04-14："AGI is a harness engineering problem rather than a model scaling problem"）。
- **harness 反向塑造模型**：Armin Ronacher《Better Models: Worse Tools》——Opus 4.8/Sonnet 5 在第三方 harness 里发明假 schema 字段，根因是新模型在 Claude Code 的"宽容 harness"里做 RL："The more post-training happens inside one dominant harness, the more every other harness will have to inherit its quirks"——第三方 harness 反而更需要确定性护栏。✅ https://lucumr.pocoo.org/2026/7/4/better-models-worse-tools/ （2026-07-04）；同作者《The Coming Loop》警告无人值守 loop 用于长期维护代码"produce worse code than what we were producing last autumn"。✅ https://lucumr.pocoo.org/2026/6/23/the-coming-loop/
- **"harness 税"量化**：Portkey 实测同一对话 Claude Code 发出 ~27,000 input tokens vs 极简 agent Pi 的 ~2,600，40 轮会话约一半是 harness 开销。✅ https://portkey.ai/blog/the-harness-tax/ （2026-04-13）；但 METR/SWE-Atlas 的基准数据显示顶级 harness 与基础 scaffold 差异"接近噪声"（⚠️ 二手转引自 latent.space AINews，未读原始评测）。

### 7.3 对培训选题的影响（§6 编排的修订）

**总判断**：上一轮 7 期编排的方向没错（重点本就在 hooks/权限/团队共享/evals 等"幸存项"上），但需要两层修正：(a) 每期开头增加"为什么模型变强了这还值得学"的 30 秒论证——这是观众 2026 年最大的疑问，§7.1/7.2 就是弹药库；(b) 两处实质性调整如下。

| 原期次 | 处置 | 理由（证据见上） |
|---|---|---|
| 1. CLAUDE.md 工程 | **保留但重心转移**：从"怎么写"改为"写什么还剩"——pitfalls/rationale/非默认 conventions 留下，仓库概览/依赖列表/通用惯例删掉；教 /doctor 自动修剪（v2.1.206）与官方"给判断不给规则"新原则 | 官方 Exclude 清单 + arXiv 实证（context files 总体不提升成功率）说明这一期不重构就会教观众写"已经死了的内容" |
| 2. Hooks：从建议到强制 | **保留不动** | 安全边界是信任问题（lethal trifecta / OWASP），官方还在给自主行为装刹车（v2.1.183/215/218）——反而升值 |
| 3. Skills 与 Subagents | **保留但加免责框架**：skills 教学聚焦"流程封装+免审批脚本"（幸存价值），明确提示"纪律性 preamble 类 skill 半衰期短"（superpowers 减重讨论）；subagents 教学聚焦"隔离与并行"而非"提示词分身术" | 模型吸收了"扮演专家"类内容，但吸收不了隔离、权限边界与免审批执行 |
| 4. Permissions 与沙箱 | **保留不动** | 同上，信任≠能力 |
| 5. 团队共享 | **保留且升级论据**：Thoughtworks Adopt 级背书 + OpenAI doc-gardening，加"个人 prompt 是反模式"论点 | 组织知识不可吸收，且这是 thick 派的结构性论据 |
| 6. 无人值守长任务 | **重构**：Ralph/交接文档仍是当下可用实践，但必须配"半衰期"框架——METR 3–4 个月翻倍曲线 + Anthropic 三代删三层案例；同时加 Ronacher 的警告（无人值守 loop 产码质量可能回退），落点从"学会这个技巧"改为"学会判断自己的任务是否在模型 horizon 之内" | 这是最可能被模型吸收的一期，不讲时效性就是误导 |
| 7. Evals | **保留不动，且可作为系列最高价值一期**：它是"每次换模型决定 harness 删什么"的元能力（"upgrade in days vs weeks"） | 模型每强一代，evals 更值钱一代 |

**新增建议（可选第 0 期或第 8 期）："harness 的消亡与幸存：Build to Delete"**——用 §7 的三方立场（Anthropic"位置移动论" / Noam Brown"吸收论" / Schmid"Build to Delete"）+ lean system prompt（v2.1.154）+ 泄露事件（512k 行）+ harness 税测量，给观众一个判断框架而非技巧清单。5 分钟结构：1 分钟问题（模型都 12 小时 horizon 了还要学配置？）→ 2 分钟消亡证据 → 1.5 分钟幸存逻辑（ground truth / 信任 / 组织知识三个"不可吸收"）→ 30 秒落点（harness 工程不是堆配置，是持续做减法的能力）。

**观众最可能带走的三句话**（可作系列 slogan）：
1. "harness 的每个组件都是对模型能力的假设，而假设会过期。"（Anthropic）
2. "模型吸收的是'它做不到什么'的补丁，吸收不了 ground truth、信任边界和你公司的知识。"（§7.2 综合）
3. "Build to Delete：好的 harness 设计以'可拆'为约束。"（Philipp Schmid）

### 7.4 本轮未证实/存疑清单

| 事项 | 状态 | 说明 |
|---|---|---|
| Anthropic《Agent Harness Design: 3 Patterns》（2026-04） | ❌ | anthropic.com 查无此文，仅见于某 GitHub awesome list，疑似对 3 月博文的误引 |
| Noam Brown / Boris Cherny 的原话 | ⚠️ | 均经 Latent Space 转引核实，未追原始播客 |
| "Anthropic 为 Claude 5 代删除 80% 系统提示且无损" | ⚠️ | 出自 superpowers issue #2087 转述官方指南，未在官方原文逐字核到该数字 |
| Portkey 所述"Opus 4.6 后删除 sprint 分解" | ⚠️ | 与 Anthropic 原文互相印证（Rajasekaran 文确有此意），但"sprint decomposition entirely stripped"的表述为 Portkey 转述 |
| METR/SWE-Atlas"harness 差异接近噪声" | ⚠️ | 二手转引自 latent.space AINews，未读原始评测报告 |
| OpenAI CISO "prompt injection remains a frontier, unsolved security problem" | ⚠️ | 多处二手转引，一手页未含该句 |
| Klaassen 对"模型变强后 compound engineering 是否持久"的正面回应 | ❌ | 未找到专文 |
| GPT-5.1-Codex-Max"compaction-native、连续自主 24h+" | ⚠️ | 多家媒体一致引用，未抓取 openai.com 原页 |
| Simon Willison 的 Opus 4.6↔4.7 系统提示 diff 原帖 | ⚠️ | 被引用存在，原帖 URL 未验证 |

*报告完（含 §7 第二轮追加）。*
