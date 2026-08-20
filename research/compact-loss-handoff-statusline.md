# ep5 行业认知调研：长会话当仓库 / compact 丢约束 / statusline + HANDOFF 解法

调研日期：2026-08-20（所有出处抓取日期同日）
调研对象：反范式课第 5 集「改掉这个习惯，让 AI 下午不变笨（长会话当仓库）」
课程解法：①statusline 仪表 ②阈值换班（60% 备交接 / 70% 必换班）③HANDOFF.md 落盘 ④PreCompact hook 兜底

---

## 问题 1：这个反模式是不是行业常识？

**判定：常识（"compact 有损"已是社区共识），但"链式 compact 约束整组丢失"的量化演示【有人讲】。**

讨论密度很高，且已沉淀出约定俗成的词汇：

- **"context rot"** 是最成型的名字。Chroma 2025-07 发布 18 模型研究《Context Rot: How Increasing Input Tokens Impacts LLM Performance》，证明输入变长所有模型都退化（[trychroma.com](https://www.trychroma.com/research/context-rot)，2026-08-20）；Anthropic 官方工程博客《Effective context engineering for AI agents》直接引用该概念，提出"attention budget"模型：上下文是边际收益递减的有限资源（[anthropic.com](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)，2026-08-20）。
- **"Memento problem / 顺行性遗忘"** 是社区自造的另一套叙事：长会话像《记忆碎片》主角，压缩后"只记得大概，没有可靠细节"（[mycarta/llm-operational-discipline](https://github.com/mycarta/llm-operational-discipline)，2026-08-20）。
- GitHub anthropics/claude-code 上 compact 丢上下文是**高频长红主题**，跨 2025-11 到 2026-05 持续有新 issue：#13112（"auto compact is the worst… it functionally makes everything worse"，2025-12，[链接](https://github.com/anthropics/claude-code/issues/13112)）、#10960（压缩后忘记切换过的仓库路径，反复犯同一个错，2025-11，[链接](https://github.com/anthropics/claude-code/issues/10960)）、#24572（压缩后丢 CLAUDE.md 感知，2026-02，[链接](https://github.com/anthropics/claude-code/issues/24572)）、#27419（多次压缩后会话"void or stuck"，2026-02，[链接](https://github.com/anthropics/claude-code/issues/27419)）、#63001（会话一关全忘，用户被迫同时挂着多个会话续命——正是"长会话当仓库"的直接证据，2026-05，[链接](https://github.com/anthropics/claude-code/issues/63001)）。
- 工程博客侧：DoltHub 工程博客的判词"**Claude Code is definitely dumber after the compaction**"被 Alexander Golev 的综述《Claude Saves Tokens, Forgets Everything》引用并扩散（[golev.com](https://golev.com/post/claude-saves-tokens-forgets-everything/)，2026-08-20）。

注意区分：大众抱怨集中在"单次 compact 后变笨"；课程实测的"**链式 compact ≥9 环约束整组丢失**"（spike/verify/ep05-compact 的 A2 样本）属于更锋利的定量版本，公开讨论里只有定性报告（#27419"压到不能再压"、issue 自称"100% 可复现"），没人给出 N 环逐条核对的证据。**这是课程相对公开讨论的增量。**

## 问题 2：主流解法是什么？观众大概率已经会了吗？

**判定：有人讲，但解法高度分散、无共识标准。观众大概率会"最糙的一档"（弃会话重开），不会课程的完整三件套。**

公开解法按普及度排序：

1. **弃疗重开（最普及）**：Du'An Lightfoot 的标准操作——compact 一开始直接 `/quit`，新开会话重喂上下文（[duanlightfoot.com](https://www.duanlightfoot.com/posts/what-to-do-when-claude-code-starts-compacting/)，2026-08-20）。Golev 综述总结："experienced users treat compaction as a failure mode to avoid rather than a feature to rely on"。
2. **手动 /compact 掐点 + 保真指令**：`/compact preserve the coding patterns we established`，在逻辑断点主动压，不等 auto-compact（Golev 综述、Steve Kinney 课程，同上）。
3. **CLAUDE.md / 外部文件做持久记忆**：社区默认动作；OpenClaw 生态甚至出现"pre-compaction flush"机制（临近压缩时让 agent 自己把要点写进 memory/ 目录）及其失效惨案（[openclaw/openclaw#8275](https://github.com/openclaw/openclaw/issues/8275) 称压缩静默丢信息导致 agent 杀掉自己启动的进程；[openclawvps.io](https://openclawvps.io/blog/openclaw-memory) 记录写作 bot 三周剧情上下文一次压缩全丢，均 2026-08-20）。
4. **HANDOFF 交接文档**：mattpocock/skills 的 `handoff` skill 是这条路的头部实现——仓库约 12.6–16 万 star，被多个 awesome 清单列为"社区高频推荐"（[aihero.dev/skills-handoff](https://www.aihero.dev/skills-handoff)、[flaqai/awesome_claude_code_skills](https://github.com/flaqai/awesome_claude_code_skills/blob/main/README_zh.md)，2026-08-20）。但注意：官方说明页自己承认它是**窄工具**——只为"上下文需要搬家"（换 harness / 换目录 / 给人 / 并行 fork）而设计，同 harness 同目录续命场景官方推荐 `/compact`；文件写 temp 目录是最多人踩的坑，且"它记 what 不记 why，把没验证过的假设写成事实会污染下一个 agent"。另有 npm 包 claude-code-handoff（[npmx](https://npmx.dev/package/claude-code-handoff)）、FlineDev/Recall（压缩后自动恢复上下文，[GitHub](https://github.com/FlineDev/Recall)）等同赛道工具。
5. **关掉 auto-compact**：有争议的一派（[DEV 社区文章](https://dev.to/agentic-engineer/taming-context-windows-disable-auto-compact-for-better-ai-4gbm)、[cc-guard](https://claudecode.to/cc-guard/disable-auto-compaction.html)，2026-08-20）；Anthropic 官方立场是 auto-compact 利大于弊（[matsuoka.com 综述](https://hyperdev.matsuoka.com/p/how-claude-code-got-better-by-protecting)，2026-08-20）。

**观众基线判断**：重度用户大概率会 1 和 3，一部分会 2；handoff 类工具"听过没用过"（知道 mattpocock 的人远多于能说出 handoff 写进 temp、不记 why 的人）；**"60% 备交接、70% 必换班"这种带阈值的仪表化纪律，公开讨论里几乎没有**——社区解法全是"出事了怎么办"，没有人系统讲"看着表提前换班"。

## 问题 3：公开翻车案例 / 数据

**判定：有人讲（定性案例丰富），缺系统性定量数据——课程实测样本本身就是稀缺证据。**

可直接引用的：

- **#9796**（经 Golev 综述转引）：用户报告压缩前项目规则 100% 遵守、压缩后 100% 违反（TodoWrite/venv/不许道歉等逐条列举），自述完全可复现（[golev.com 综述](https://golev.com/post/claude-saves-tokens-forgets-everything/)，2026-08-20）。
- **#13919**：auto-compact 后 Claude 完全丢失正在使用的 Skills 及其方法论，连"压缩后重读 skill 文件"的 CLAUDE.md 显式指令也被忽略（同上）。
- **DoltHub 工程博客（2025-06）**："Claude Code is definitely dumber after the compaction… It will make mistakes you specifically corrected again earlier in the session."（经 Golev 转引，同上）。
- **Opus 4.5 thinking blocks 破坏 compact**：#12311，extended thinking 的 `<thinking>` 块不可修改导致压缩直接失败、会话不可恢复， workaround 是降级到 Sonnet——"付最贵的钱拿最差的上下文管理"（经 Golev 转引，同上）。
- **OpenClaw #8275**：压缩丢状态后 agent 干扰/终止自己启动的进程，定性为 "dangerous actions"（[GitHub](https://github.com/openclaw/openclaw/issues/8275)，2026-08-20）。
- **Chroma context rot 数据**（二手转引版）：18 个前沿模型全部随长度退化；相关事实落在 20 文档上下文的第 5–15 位时准确率掉 30+ 分；屏蔽干扰项后纯长度仍带来 7.9% 地板下降（[particula.tech](https://particula.tech/blog/chroma-context-rot-long-context-degradation)，2026-08-20；一手是 [Chroma 原文](https://www.trychroma.com/research/context-rot)）。
- **Anthropic 官方口径 vs 社区口径的张力本身可引用**：官方博客称 compaction "minimal performance degradation"（[anthropic.com](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)，2026-08-20），社区用"definitely dumber"回敬——官方营销叙事与用户体感之间的落差是最好的课程引子。

## 问题 4："大家都这么以为但其实是错的"认知差（最值钱的点）

**判定：几乎没人讲。以下四条在公开讨论里要么没有、要么被错误默认，是课程差异化空间。**

1. **"CLAUDE.md 会被 compact 压掉" vs "CLAUDE.md 其实幸存"——两边都错一半。** GitHub 上多个 issue（#24572、#9796）坚称压缩后丢项目指令；但 CLAUDE.md 是每轮系统提示注入的，架构上不进压缩管道，课程 spike 实测（2026-08-19）也显示 CLAUDE.md 组 8–9 环逐字幸存。真正会丢的是**会话中口头下达的约束**——issue #9796 丢的其实是 `.claude/project-context.md` 经对话建立的规则。公开讨论把两类指令混为一谈，**"写进 CLAUDE.md 的能活、说在对话里的会死"这条分界线没人讲清楚过**。（出处：#24572 [链接](https://github.com/anthropics/claude-code/issues/24572)；#9796 经 [Golev 综述](https://golev.com/post/claude-saves-tokens-forgets-everything/)转引；2026-08-20）
2. **"PreCompact hook 是可靠安全气囊"——错，它在 auto-compact 路径上可能根本不触发。** PreCompact 于 v2.1.105（2026-04-13）落地，支持 exit 2 / `{"decision":"block"}` 否决压缩；但 issue #50467 报告：**注册后 auto-compaction（trigger:"auto"）全程不调用该 hook，从 v2.1.105 到 v2.1.114 跨 10+ 个小版本未修**，前序 issue #13572 被 stale 关闭（[GitHub](https://github.com/anthropics/claude-code/issues/50467)，2026-08-20）。**这对课程交付物是硬约束：PreCompact 兜底在自动压缩路径上不可信，真正兜底的是 statusline 仪表 + 阈值纪律，hook 只能当彩蛋并必须标注该 bug。**
3. **"仪表盘数字可信"——部分错。** statusline 官方 JSON 提供预计算的 `context_window.used_percentage`，但已知 bug：1M 上下文会话分母卡在 200k 导致百分比钉死 100%（[#76751](https://github.com/anthropics/claude-code/issues/76751)）；Windows Git Bash 下恒为 0（[#57983](https://github.com/anthropics/claude-code/issues/57983)）；/context、statusline、内置警告三处口径互相打架（[#18241](https://github.com/anthropics/claude-code/issues/18241)）（均 2026-08-20）。**"看着表开车"得加半句"表本身会失准，阈值要留冗余"——60/70% 阈值设计恰好天然对冲这一点，这是课程阈值论的隐藏论据。**
4. **"会话越长越懂我"——研究层面证伪，用户层面普遍默认。** Golev 精确描述了这种错觉："花三十轮教会模型你的偏好，会话终于在变好用的路上"——然后压缩把一切概括成 "user prefers certain coding conventions"。Chroma 数据 + Anthropic 的 attention budget 模型都说明"懂我"的感觉从某个长度起就是负资产。Anthropic 官方甚至承认 compaction 的艺术在于取舍，"overly aggressive compaction can result in the loss of subtle but critical context whose importance only becomes apparent later"（[anthropic.com](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)，2026-08-20）。
5. （次要）**"handoff 文档 = 更好用的 /compact"——官方自己否定。** mattpocock 官方说明页明确定位：handoff 买的是可携带性不是压缩率，同 harness 同目录应该 /compact；且文档会把"以为"写成"事实"污染下一个 agent（[aihero.dev](https://www.aihero.dev/skills-handoff)，2026-08-20）。课程 HANDOFF.md 四格模板若强制"已验证/未验证"分区，就是对这条公开批评的直接回应。

## 问题 5：相关官方功能 / 命令现状（课程引用校准）

- **auto-compact**：默认开启，接近上下文上限时自动压缩整个会话历史（官方工程博客描述为"保留架构决策、未解决 bug、实现细节 + 最近访问的 5 个文件"）。社区测量触发点说法不一（~80% / ~95%），课程引用用"接近上限"模糊表述，不要写死数字。
- **/compact**：支持保真指令参数（`/compact 保留我们定的编码约定`），这是官方机制、Golev 综述确认（2026-08-20）。
- **/statusline**：官方原生功能。`/statusline` 命令可用自然语言让 Claude Code 自动生成脚本；JSON 载荷含预计算的 `context_window.used_percentage` / `remaining_percentage`（按 input+cache 计，不含 output）、`context_window_size`（默认 200k，扩展上下文 1M）、rate_limits 等；终端宽度 COLUMNS/LINES 需 v2.1.153+。脚本在 /compact 完成后等事件时重跑。（[官方文档 code.claude.com/docs/en/statusline](https://code.claude.com/docs/en/statusline)，2026-08-20）
- **PreCompact hook**：v2.1.105（2026-04-13）新增，支持 exit 0 放行 / exit 2 或 `{"decision":"block"}` 否决 / 自定义替换摘要（[Zenn 版本指南](https://zenn.dev/kai_kou/articles/231-claude-code-precompact-worktree-guide?locale=en)、[Qiita](https://qiita.com/nogataka/items/95efda0c7c9ea2405139)，2026-08-20）。**已知缺陷：auto-compact 路径不触发（#50467，2026-08-20 检索时未见修复公告）**；另有无条件 block 会把 /compact 整个废掉的插件事故先例（[MemPalace#1172](https://github.com/MemPalace/mempalace/issues/1172)，2026-08-20）——课程 hook 必须按 trigger 过滤 + 设释放条件。
- **statusline 精度 bug**：见问题 4 第 3 条（#76751 / #57983 / #18241）。
- **监控生态**：ccusage（本地 JSONL 用量分析，社区标配）、ccstatusline / starship-claude（官方文档点名推荐的预制 statusline）、Claude HUD 等插件化仪表盘（2026-08-20）。
- **mattpocock handoff skill**：安装 `npx skills@latest add mattpocock/skills`，`/handoff` 手动触发，写 OS temp 目录，密钥先脱敏，已落盘的 spec/plan/diff 只引用不复制（[aihero.dev](https://www.aihero.dev/skills-handoff)，2026-08-20）。

---

## 对课程定位的启示

**观众早知道（讲了会水，一笔带过即可）：**
- "compact 会丢东西、压缩后变笨"——社区共识，DoltHub 金句级别传播。开场用一句 #13112 的 "it functionally makes everything worse" 点火就够，不要花时间论证。
- "出事了就 /quit 重开"——重度用户的肌肉记忆。课程应该直接承认这是当前最优野路子，再指出它的代价（重喂上下文 = 重新付全价 + 交接质量全靠人脑）。
- CLAUDE.md 做持久记忆——默认动作。

**真盲区（讲了值钱，课程主体压在这里）：**
1. **"口头约束 vs CLAUDE.md"的存活分界线**（认知差 #1）——观众普遍以为"CLAUDE.md 也会被压掉"或反向以为"会话里说的跟 CLAUDE.md 一样安全"，两边都错。课程 spike 的 CLAUDE.md 组 8-9 环逐字幸存 + A2 约束组 0/3 恰好是这条分界线的独家实证。
2. **阈值纪律本身**（60% 备交接 / 70% 必换班）——公开解法全是事后处置，没有人把 compact 当"事故"做前置预防。这是课程相对 mattpocock 官方叙事（"同目录用 /compact 就行"）的立场差异：官方把 compact 当常规操作，课程把它当事故，这个分歧点要讲透并用 #9796 的"100% 违反"作证。
3. **PreCompact hook 的 auto 路径失效 bug**（#50467）——交付物可信度问题。课程必须把"安全气囊可能不弹"说出来，否则懂行的观众现场拆台；反过来说，诚实标注这个 bug 反而强化"仪表 + 阈值才是真兜底"的主线。
4. **仪表会失准**（#76751 等三个 bug）——"看着表开车，但表要留冗余"，这恰好是阈值不定 90% 而定 60/70% 的工程理由，值得一句口播。
5. **HANDOFF 的"已验证/未验证"分区**——回应 mattpocock 官方自己承认的"记 what 不记 why、假设变事实污染下一棒"批评，四格模板若含这一格就是差异化卖点。

**引用纪律提醒**：compact 触发百分比、CLAUDE.md 是否被压、handoff star 数三个数字社区口径混乱，课程引用一律按问题 5 的官方文档口径 + "截至 2026-08"时间戳。
