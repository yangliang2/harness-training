# 工具输出无节制：行业认知水平调研

调研日期：2026-08-20
调研目的：为"工具输出无节制"一集确定反模式靶心。针对课程实测的三个发现（①verbose 输出 +44% 成本/轮次但模型总能自己 grep 恢复；②Claude Code 对 exit≠0 命令保头截断 ~10KB；③PostToolUse 截断 hook 对 exit≠0 不生效且 hook 输入被预截断 30KB），摸清行业认知水位。

判定口径：
- **常识** = 官方文档/头部工程博客反复论述 + 社区工具生态已成规模
- **有人讲** = 有公开 issue/博客/工具专门讨论，但未形成普遍认知
- **几乎没人讲** = 找不到直接或只有擦边的公开讨论

---

## Q1 "工具输出污染上下文"是不是行业常识？

**判定：常识。**

这是 context engineering 话语圈的核心议题之一，官方和头部社区都在反复讲：

- Anthropic 官方《Effective context engineering for AI agents》明确提出"smallest possible set of high-signal tokens"原则，点名 context rot（上下文越长检索精度越差）、context pollution，并把"工具要返回 token 高效的信息"列为工具设计准则；还专门推荐 tool result clearing（清理历史工具结果）作为最安全的压缩手段。https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Anthropic《Writing tools for agents》同样把 token efficiency 列为工具设计目标。https://www.anthropic.com/engineering/writing-tools-for-agents
- Chroma 2025 年的 Context Rot 研究报告是这个词的源头，被大量二手文章引用（如 Cockroach Labs 2026-06 的成本文、mindstudio 2026-04 的科普文）。https://www.mindstudio.ai/blog/context-rot-ai-coding-agents-how-to-prevent
- 社区工具生态已经成规模：Token-Saver（36 个命令家族的输出压缩，Claude Code 插件，进了 Anthropic 官方社区插件市场）、RTK（Rust 代理压缩 shell 输出，号称省 60-90%）、claude-warden（自动给 git/npm/cargo 加 quiet flag）、context-mode（MCP 层拦截内建工具输出）。https://github.com/ppgranger/token-saver ；https://sanketdaru.com/blog/claude-code-token-optimization/
- 学术圈也在跟进：arXiv 上已有"编码 agent 的 context 凝结策略"和"context rot 在编码 agent 中何时出现"的白盒研究。https://arxiv.org/html/2605.18854v1 ；https://arxiv.org/html/2607.17937v2

结论：观众里关注 AI 编程的人大概率已经知道"要控制工具输出长度"。这层讲了会水。

## Q2 业界的标准解法是什么？

**判定：常识（安静模式）+ 有人讲（hook 截断/包装器）+ 常识（让 agent 自己 grep，且是官方背书）。三条路线并存，没有唯一标准。**

1. **安静模式/terse reporter：被明确推荐，已成套路。**
   - barkain 的 Claude Code 工作流编排框架在系统提示里直接教 compact flag：`git status -sb`、`pytest -q --tb=short`，作为默认开启的 token efficiency 层。https://github.com/barkain/claude-code-workflow-orchestration
   - claude-warden 用 hook 自动把 verbose 命令改写为 quiet flag 版本（git/npm/cargo/pip/docker/ffmpeg）。https://github.com/hesreallyhim/awesome-claude-code/issues/919
   - gordles.io 的实验文量化了 `pytest --tb=short -q` 只省 13% token（格式噪音好去，重复错误难去），说明社区对 quiet flag 的局限也有数据认知。https://gordles.io/blog/llm-friendly-test-suite-outputs-pytest-llm
2. **hook 截断/输出改写：是活跃的社区实践，但有公认的坑（见 Q3）。**
   - Token-Saver 的 README 明确写道："Claude Code 的 PreToolUse hook 无法在执行后修改输出，唯一办法是把命令改写为经过 wrapper 执行"——即社区已经知道 PostToolUse 改写不可靠，绕道走 PreToolUse 命令重写。它还为 exit≠0 设计了专门的 failure routing（失败输出走保守的通用处理器，避免启发式丢错误行）。https://github.com/ppgranger/token-saver
   - 常见的 Claude Code 配置仓库标配 `hooks/filter-output.sh`（PostToolUse 截断 verbose Bash 输出）。https://github.com/pushkarverma3698/claude-code-setup
3. **"让 agent 自己 grep"：被官方背书为正当设计模式。**
   - Anthropic context engineering 文把 "just in time" 上下文策略作为推荐范式：agent 用 head/tail/grep 分析大数据而不全量载入，Claude Code 的 glob/grep 原语就是这个思路。https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
   - Arize 的生产经验文也描述好 agent 的行为是"读预览、grep 定位、按行段读"，而不是整文件灌入。https://arize.com/blog/ai-agent-debugging-four-lessons-from-shipping-alyx-to-production/
   - 这直接印证课程实测①：模型能自己 grep 恢复，是业界公认的 agent 正常能力，不是新发现。

## Q3 PostToolUse hook 截断的坑有人公开讨论过吗？

**判定：有人讲（updatedToolOutput 失效有大量公开记录），几乎没人讲（exit≠0 分流与 30KB 预截断这两个具体机制）。官方无实质回应。**

- **updatedToolOutput 对内置工具静默失效**：至少有四个公开 issue——#54196、#65403、#67442、#68951，全部指向同一缺陷：hook 正确返回 `hookSpecificOutput.updatedToolOutput`，模型仍收到原始输出。官方处理方式是 bot 全部自动关为 duplicate，合并进几个"输出脱敏 feature request"，至今没有修复、没有官方人员的实质回应。#68951 作者专门发文抗议"这是已发布功能坏了，不是新需求"。https://github.com/anthropics/claude-code/issues/68951 ；https://github.com/anthropics/claude-code/issues/67442
- **一个关键澄清**（#68951 评论区，bradfeld）：对 Bash 工具，`updatedToolOutput` 必须传对象 `{stdout, stderr, interrupted}` 而不是裸字符串；对象形式在 v2.1.215 上**确实生效**。即部分"失效"报告其实是 payload 形状不对——文档没写清楚。这个细节本身就是一个好课程素材。
- **exit≠0 的分流**：新版官方文档（2026 年中后）已把事件拆成 `PostToolUse`（After a tool call **succeeds**）和 `PostToolUseFailure`（After a tool call **fails**）——我们实测的"exit≠0 时 hook 不生效"在文档层面已被固化为设计：失败命令走另一个事件，只配 PostToolUse 的 hook 根本收不到。但官方从没解释过这个拆分的迁移影响，社区只有零星的 hooks 教程提到两事件并存。https://code.claude.com/docs/en/hooks ；https://nestenius.se/ai/exploring-claude-code-hooks-with-the-coding-agent-explorer-net/
- **hook 输入 30KB 预截断**：没有专门讨论。能找到的相邻记录是 Bash 工具本身 30,000 字符截断上限（issue #19901 抱怨文档没写；LobeHub 技能文提到"Output truncated at 30,000 characters"），以及 #41799 抱怨 hooks 文档没写 >50KB hook 输出落盘预览行为。"hook 收到的 tool_response 已经是被截断过的版本"这一层，未见公开讨论。https://github.com/anthropics/claude-code/issues/19901 ；https://github.com/anthropics/claude-code/issues/41799
- 同一缺陷在别的 harness 上也存在：Kiro 的 postToolUse hook 同样只能观察不能改输出（源码注释 "We do not support processing the PostToolUse hook output yet"）。https://github.com/kirodotdev/Kiro/issues/7417

## Q4 测试输出冗长导致 agent 故障的公开案例？

**判定：有人讲。案例不少，但最常被讲的变体是"成本/token 浪费"，其次是"context 撑爆后退化"；"截断丢信号导致误诊"主要在对截断的一般性讨论里，专门绑定到测试输出的很少。**

- **成本变体（最多）**：Token-Saver 实测 `pytest` 500 tests 6,744→307 tokens（95%）；gordles.io 用 MCP 把构建输出路由给小模型摘要，Claude Code 构建/测试 token 消耗降 85%；autonomous-dev 的 issue #90 记录 verbose pytest 输出直接导致 agent 冻结，修复方案第一条就是 `pytest -q --tb=line` + token 预算监控。https://github.com/akaszubski/autonomous-dev/issues/90
- **退化/故障变体**：Token-Saver README 的论述有代表性："verbose 工具输出是填满 context window 最快的方式，而 context 满了 agent 就开始退化——忘掉先前的指令、重复读已读过的文件、丢掉多步任务的线索"。https://github.com/ppgranger/token-saver
- **截断丢信号变体（有，但不对准测试场景）**：
  - #36596：Bash 输出被截断后模型**编造**被截掉的内容而不是去读全量文件。https://github.com/anthropics/claude-code/issues/36596
  - #63871：agent 编造工具输出、虚构了一次"prompt injection 事件"。https://github.com/anthropics/claude-code/issues/63871
  - #64577：模型习惯用 `cmd 2>&1 | tail -N` 截输出，结果 EOF 缓冲过滤器把错误和挂起都藏掉了——作者指出这恰好砸在"最需要 agent 发现失败"的场景上。https://github.com/anthropics/claude-code/issues/64577
  - DEV 社区有篇《Tool-Result Truncation: The Silent Bug That Makes Agents Lie》。https://dev.to/gabrielanhaia/tool-result-truncation-the-silent-bug-that-makes-agents-lie-3epe

## Q5 "失败测试的输出尾部才是摘要"（保头截断杀尾部）有人讲过吗？

**判定：几乎没人讲。这是本次调研里水位最低的点。**

- 没有找到任何公开材料把"Claude Code 保头截断"和"pytest/jest 的失败摘要在输出尾部"这两件事连起来讲。
- 相邻证据链是齐的，但没人拼起来：
  - pytest 官方文档：`-r` 的 short test summary info 在测试会话**末尾**输出。https://docs.pytest.org/en/stable/how-to/output.html
  - Claude Code 保头截断：pi（另一 harness）的 issue #5662 实测 bash 工具输出在 ~12KB 处从中间砍断、只留前面 58 行；Claude Code 侧 #40100 也描述截断保留开头、其余落盘。https://github.com/earendil-works/pi/issues/5662 ；https://github.com/anthropics/claude-code/issues/40100
  - Token-Saver 的 FAQ 批判了反方向的盲截（"`tail -50` 不知道错误在第 12 行……盲截保留最后 50 行然后祈祷"），并因此设计了 head+tail 双向保留加错误行回收——说明"截哪头会丢信号"这个思考在工具作者层面存在，但没有沉淀为面向用户的公共知识。
- 即：每个零件都有人摸过，"保头截断恰好杀死失败摘要"这个完整故障模式没有公开叙述。

---

## 对课程定位的启示

**观众早知道的（讲了会水）：**

- "工具输出污染上下文、要控制输出长度"（Q1）——官方+社区+学术三重覆盖，一句话带过即可，不要当卖点。
- "pytest -q / --tb=short 等安静模式"（Q2 路线 1）——已是社区教程标配，当成 baseline 而不是解法。
- "模型可以自己 grep/head/tail 恢复，不用全量灌入"（Q2 路线 3，实测①）——Anthropic 官方背书的 just-in-time 范式。课程里这个实测结论只能作为"佐证官方说法"，不能作为新发现来讲。

**真盲区（讲了值钱）：**

1. **保头截断杀尾部摘要（Q5，实测②）——最强靶心。** 零件散落各处但没人拼过：失败时摘要恰恰在尾部，而 harness 保头截断恰好把摘要杀掉、把最没信息量的 progress/header 留下。这是一个"每个资深用户都隐性感知过、但没人命名过"的故障模式，且直接推翻"截断就截断，反正模型能 grep"的天真乐观——截断丢的恰恰是 grep 之前最需要的那份地图。
2. **exit≠0 时 PostToolUse 截断 hook 完全不生效（Q3，实测③）——次强靶心。** 公开 issue 证明"updatedToolOutput 静默失效"广为人知，但我们的发现更精确：失败命令被分流到 PostToolUseFailure，只在 PostToolUse 上挂的截断 hook 对**恰恰最需要截断的场景**（失败的构建/测试输出）完全不生效——安全感和实际保护是反的。这个"保护在最需要时缺席"的反讽结构非常适合反模式叙事。附带可讲的实操细节：Bash 的 updatedToolOutput 要传 `{stdout, stderr, interrupted}` 对象才生效（文档没写清，issue 评论区才有人试出来）。
3. **"hook 输入已被 30KB 预截断"（Q3，实测③的一部分）——补充靶心。** 即 hook 层拿到的已经不是真相，想在 hook 里做"从完整输出里提取尾部摘要"都做不到。几乎无公开讨论。
4. **量化数据（实测①的 +44%）依然有价值**：行业里有 Token-Saver 的压缩率数据和 gordles 的 85% 案例，但"verbose vs terse 在同一任务上的端到端成本/轮次对比 + 模型能否自愈"这种对照实验不多。把 +44% 定位为"给常识补上定量证据"，而不是"发现新现象"。

**叙事建议**：开场用 Q1/Q2 一句话建立共识（"控制输出长度，地球人都知道"），立刻反转到盲区——"但失败的那一次运行，你的三道防线（安静 flag、截断 hook、harness 自带截断）恰好全部失效或起反作用"，然后逐个引爆实测②③。靶心应定在 **"防线在失败时缺席"**（hook 分流 + 保头杀尾的组合拳），而不是定在"输出太长费钱"这个已是常识的点。
