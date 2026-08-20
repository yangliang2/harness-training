# 逆分布造轮子（自研工具让 agent 出训练分布）：行业认知调研

调研日期：2026-08-20。所有 URL 均于 2026-08-20 抓取核实。服务对象：反范式课第 4 集（`antipattern-course-outline.md` "逆分布造轮子"）。关联实测：`spike/verify/ep04-gh-vs-mcp/`、`spike/verify/ep04-multi-mcp/`（结论：help 合格的自研工具 10/10，自研税只在五个条件显性化）。

判定刻度：【常识】= 高频讨论、可视为行业共识；【有人讲】= 有高质量出处但讨论密度有限；【几乎没人讲】= 只有零星材料或本课程实测独有。

---

## 总览判定

| 问题 | 判定 |
|---|---|
| 1. 反模式是不是行业常识 | 【有人讲】——"模型对主流工具/库更熟"是常识，但"自研工具=给 agent 加税"作为工程选型罪名没有约定俗成的名字 |
| 2. 主流解法、观众是否会了 | "直接用主流工具"【常识】；"兼容外壳"【有人讲】（有 glab 等大牌先例，但作为方法论没人总结）；"文档对冲"【有人讲】 |
| 3. 公开翻车案例/数据 | 【有人讲】——学术侧数据硬（MCP-Universe、包幻觉、描述质量研究），工程侧具体翻车案例多为轶事 |
| 4. 认知差（大家以为对其实错） | 【几乎没人讲】——最值钱的三条见第 4 节 |
| 5. 官方功能/命令现状 | 已全部一手核实，见第 5 节（gh v2.97.0 / glab v1.114.0 / MCP spec 2026-07-28） |

---

## 1. 反模式本身是不是行业常识

**判定：【有人讲】（原理是常识，罪名没名字）。**

"模型的'会'来自训练时见过"这一原理在头部声音里是公开常识：

- Simon Willison（Django 联创、LLM 社区最高频声音之一）2025-03-11 长文《Here's how I use LLMs to help me write code》明确把"选流行库"列为选库标准："I now deliberately consider this when picking a library—I try to stick with libraries with good stability and that are **popular enough that many examples of them will have made it into the training data**. I like applying the principles of boring technology"，并说分布外的库"LLMs can still help you work with... but you need to put in more work—you'll need to feed them recent examples"。来源：[simonwillison.net/2025/Mar/11/using-llms-for-code/](https://simonwillison.net/2025/Mar/11/using-llms-for-code/)（2026-08-20 抓取）。
- HN 2026-03-01 热帖《When does MCP make sense vs CLI?》（447 分、284 评论）原帖《MCP is dead, long live the CLI》（ejholmes.github.io）核心论点即本集原理："LLMs are really good at using command-line tools. **They've been trained on millions of man pages, Stack Overflow answers, and GitHub repos full of shell scripts.** When I tell Claude to use `gh pr view 123`, it just works." 来源：[原帖](https://ejholmes.github.io/2026/02/28/mcp-is-dead-long-live-the-cli.html)、[HN 讨论](https://news.ycombinator.com/item?id=47208398)（均 2026-08-20 抓取）。
- 学界有专门的 bench 维度：MCP-Universe（arXiv:2508.14704，2025-08）把 **"unknown-tools challenge"** 列为 LLM agent 的三大挑战之一："LLM agents often lack familiarity with the precise usage, parameter specifications, and expected behaviors of diverse MCP servers"。来源：[arXiv:2508.14704](https://arxiv.org/html/2508.14704v1)（2026-08-20 抓取）。

**但是**：这个反模式作为"工程选型错误"没有约定俗成的名字。社区用的邻近术语是 out-of-distribution / "not in training data" / boring technology / agent-friendly（或 Anthropic 造的 ACI, agent-computer interface）。中文圈偶见"给 AI 选老技术栈"的说法。也就是说：观众对"原理"有体感，对"这是一条可命名的选型纪律"没有认知框架——课程自造"逆分布造轮子/模型税"的命名空间是空的，可以占。

## 2. 主流解法是什么？观众大概率已经会了吗

**"直接用主流工具"【常识】；"兼容外壳"【有人讲，有先例无方法论】；"文档对冲"【有人讲，官方亲自讲】。**

- **解法一（直接用主流工具）**：CLI 派 vs MCP 派是 2025–2026 持续高热争论（HN 447 分帖、OneUptime《Why CLI is the New MCP for AI Agents》2026-02-03、DeployHQ《CLIs or MCP for Coding Agents?》2026-04-13、lunar.dev《CLI vs MCP: You're Asking the Wrong Question》2026-05-17）。观众大概率已经"会用 gh"——这集如果只讲"用主流工具别自研"，就是水。
- **解法二（兼容外壳）**：先例是大牌且可查的，但没人把它总结成方法论。
  - **glab 照 gh 做**：GitLab CLI 官方 README "Inspiration" 一节一手写明："The GitLab CLI was adopted from Clement Sam in 2022 to serve as the official CLI of GitLab. Over the years the project has been inspired by both the **GitHub CLI** and Zaq? Wiedmann's lab." 原 profclems/glab 仓库置顶："now officially adopted by GitLab as the official CLI tool"。来源：[gitlab.com/gitlab-org/cli README](https://gitlab.com/gitlab-org/cli/-/raw/main/README.md)、[github.com/profclems/glab](https://github.com/profclems/glab)（均 2026-08-20 抓取）。命令面确实对齐 gh：`glab mr list/create` ↔ `gh pr list/create`、`glab ci view` ↔ `gh run view`。
  - **更大的先例是"OpenAI-compatible API"**：整个推理生态（vLLM、Ollama、LiteLLM、各云厂商）都把自家接口做成 OpenAI 请求格式，因为框架/agent 假设它——这是"兼容外壳降分布税"在 API 层的行业级实证。来源线索（工程材料非软文）：[TabbyML issue #4410](https://github.com/TabbyML/tabby/issues/4410)（"Frameworks like SWE-Agent and OpenHands rely on the OpenAI-standardized..."）、[LiteLLM openai_compatible 文档](https://www.aidoczh.com/litellm/docs/providers/openai_compatible)（2026-08-20 抓取）。
- **解法三（文档对冲）**：Anthropic 官方工程博客《Writing effective tools for AI agents》（2025-09-11）是权威出处：工具描述本质是 prompt engineering；要用 eval 驱动迭代（"Run an evaluation… collaborate with agents to improve your tools"）；并给出具体案例——web search 工具上线后发现 Claude 总往 query 里塞 "2025"，靠改工具描述纠偏。该文同时给出"返回自然语言标识符而非 UUID 显著降低幻觉"的实测结论。来源：[anthropic.com/engineering/writing-tools-for-agents](https://www.anthropic.com/engineering/writing-tools-for-agents)（2026-08-20 抓取）。MCP-Universe 的 exploration phase（先让模型自由试工具再做任务）在部分域 +7.5pp，也支持"曝光可以对冲不熟"。

观众真实水平：第一档人人会；第二档见过 glab 但很少有人在自研内部工具时自觉"照主流做接口"；第三档（写描述对冲、裸写测试判税）是专业圈做法，普通观众基本不会。

## 3. 公开翻车案例/数据

**判定：【有人讲】（学术数据硬，工程轶事散）。**可引用：

- **MCP-Universe**（Salesforce，arXiv:2508.14704）：真实 MCP server 上 GPT-5 成功率仅 43.72%、Claude-4.0-Sonnet 29.44%；错误分析指出模型对不熟工具反复犯参数级错误（Yahoo Finance 要求 start≠end date，模型总填成一样）；挂 7 个 server/94 工具后成功率明显下滑（Claude 位置导航 22.22%→11.11%）。**另一个对本集极有用的数据点**：Cursor Agent 整体跑输裸 ReAct（26.41% vs 29.44%），作者归因"Cursor's reliance on **internal tools** rather than the benchmark's MCP servers"——商业产品用自研内部工具反而掉分的公开量化记录。来源：[arXiv:2508.14704](https://arxiv.org/html/2508.14704v1)（2026-08-20 抓取）。
- **包幻觉研究**（Spracklen et al., arXiv:2406.10279，USENIX Security 2025）：16 个模型 × 57.6 万代码样本，商业模型平均 ≥5.2%、开源模型 21.7% 的包引用是编造的，共 205,474 个不存在的包名——"分布外就编"的定量证据（已衍生 slopsquatting 供应链攻击讨论）。来源：[arXiv:2406.10279](https://arxiv.org/abs/2406.10279)（2026-08-20 抓取）。
- **MCP 工具描述质量研究**（arXiv:2602.14878，2026-02）：扫了 103 个主流 MCP server 的 856 个工具，**97.1% 的描述至少有一个"smell"，56% 连用途都没说清**。来源：[arXiv:2602.14878](https://arxiv.org/abs/2602.14878)（2026-08-20 抓取）。
- **工程侧翻车（轶事级）**：HN 447 分帖评论区高亮："试过 Jira MCP，一团糟。还不如让 LLM 直接调 API 自己写脚本"；"我的 AI agent 通过 shell 命令控制整个开发流程……agent 就凭 --help 输出就能搞定从没见过的 CLI flag，而我用过的每个 MCP server 都需要人盯着"。来源：[HN item?id=47208398](https://news.ycombinator.com/item?id=47208398)（2026-08-20 抓取，转述自 uucode.org 整理的引文亦一致）。

## 4. 认知差：大家这么以为但其实错/没人讲的点

**判定：【几乎没人讲】——本集最值钱的部分。**

1. **"自研必翻车"是错的，真正的变量是接口/文档质量，不是流行度本身。** 本课程实测（`spike/verify/ep04-*`）：help 合格的自研工具 10/10 零工具错误，多工具组轮次成本反超 gh 组；自研税只在五条件显性化（文档缺失或误导/无 schema MCP/轮次紧/工具数量爆炸/隐式状态）。公开侧支持这个反转而非"主流迷信"：HN 高赞"agent 就凭 --help 就能搞定从没见过的 CLI flag"；Anthropic 官方教你用 eval+描述把自研工具调教到可用；MCP-Universe 的 exploration phase 有效。**流行度只是"零文档时的先验"，不是税本身**——这个层次公开讨论里几乎没人点破，社区还在"CLI 好/MCP 坏"的二元站队阶段。
2. **主流工具自己也有接口税，且没人量化过。** 课程实测 gh 组反而被 base64/jq 摩擦拖累（13.2 轮/$0.459 vs 自研 11.6 轮/$0.368）。公开材料只有 Anthropic 博客的零散案例（web search 塞 "2025"、UUID 幻觉），没有"主流 CLI 对 agent 的摩擦成本"的定量讨论——肌肉记忆的真实溢价是**零方差/首轮正确率**，不是绝对效率，这是课程的独家口径。
3. **"描述写得越全越好"也是错的。** arXiv:2602.14878 实测：把描述补全所有成分，任务成功率中位数 +5.85pp，但**执行步数 +67.46%，且 16.67% 的案例回退**；紧凑变体往往更优。即"文档对冲"有自己的税——这与课程"文档对冲"档必须讲剂量完全一致，公开传播中这个 trade-off 几乎无人提。

## 5. 官方功能/命令现状（2026-08-20 一手核实）

- **gh（GitHub CLI）**：最新 release **v2.97.0**（github.com/cli/cli/releases/latest 重定向核实，2026-08-20）。
- **glab（GitLab CLI）**：最新 release **v1.114.0**（GitLab API releases 核实，2026-08-20）。项目历史：社区项目 profclems/glab → 2022 被 GitLab 官方收养为官方 CLI（原仓库归档置顶说明）；官方 README "Inspiration" 节明说受 GitHub CLI 启发。注意 glab 现在有实验性 `glab mcp` 子命令（README core commands 列表，标注 EXPERIMENTAL）——"CLI 和 MCP 融合"正在发生，课程若讲 CLI vs MCP 需留这个活口。SemVer 声明：实验特性可随时改。
- **MCP 规范**：当前版本 **2026-07-28**（modelcontextprotocol.io/specification 307 重定向核实，2026-08-20；上一版 2025-11-25）。Tools 章对 `description` 的定义仍只有一句 "Human-readable description of functionality"——**规范正文至今不给描述写法指导**。可引用的规范内指导只有：工具名规则（1–128 字符、大小写敏感、限 `[A-Za-z0-9._-]`、server 内唯一、聚合时建议加 server 前缀消歧）；工具执行错误 SHOULD 返回给模型用于自我纠正；新增非规范性的 Stateful Tools 设计指导（显式 handle、生命周期写进创建工具的描述）。来源：[MCP spec 2026-07-28 server/tools](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)（2026-08-20 抓取）。
- **SEP-1382《Documentation Best Practices for MCP Tools》**（2025-08-23 提出）：提议在规范中加"Tool Documentation Best Practices"一节——工具描述只写高层用途（给谁选工具看），参数细节放 inputSchema 的属性 description（给谁构造调用看）。**截至 2026-08-20，该内容未出现在 2026-07-28 规范正文，即仍是提案、未合入**（GitHub API 限额未能核到 issue 开闭状态，以规范正文为准）。来源：[modelcontextprotocol#1382](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1382)（2026-08-20 抓取）。
- **Anthropic 官方工具写法最佳实践**：事实标准在工程博客而非规范——《Writing effective tools for AI agents》（2025-09-11，链接见第 2 节），要点：少而精的任务导向工具（`schedule_event` 而非 list+create 三件套）、命名空间消歧、返回高信号自然语言字段、token 效率、描述即 prompt engineering、eval 驱动迭代。

## 6. 对课程定位的启示

**观众早知道（讲了会水）：**

- "模型对主流工具/库熟、对冷门自研编"——Simon Willison 级别的大 V 讲烂了，HN 月经帖。原理一句带过即可，不能当主菜。
- "能用 gh 就别自建 GitHub MCP"——CLI 派观点已在 2026 上半年成为社区多数派，再讲是复读。

**真盲区（讲了值钱）：**

1. **把"逆分布造轮子"命名成选型纪律 + 三档模型税框架**。社区只有"原理体感"和"CLI/MCP 站队"，没有人给"内部工具该怎么造"的可操作分层（直接用主流 / 兼容外壳照抄接口 / 自研必须文档对冲），更没有"让它裸写一段调用——一次写对零税、查文档写对低税、查完还编错高税"这种当天可用的判据。这是空生态位。
2. **方向反转的诚实边界**：help 合格的自研工具可以零税，"主流迷信"本身也是反模式。这直接对着社区二元站队的认知差打，且本课程有 N 轮实测数据（spike/verify/ep04-*）背书——别人没有。
3. **兼容外壳作为方法论**：glab 照 gh、OpenAI-compatible API 是两个大牌先例，但"内部工具照主流同款接口造、差异写进 --help"作为打法无人总结。
4. **文档对冲的剂量**：97.1% 的 MCP 描述有 smell（arXiv:2602.14878），但补全描述 +67% 步数、16.67% 回退——"写描述"不是"写满描述"。Anthropic 的 eval 驱动迭代是唯一官方口径，观众基本不知道。
5. **引用准确性红利**：MCP 规范最新是 2026-07-28 且仍不管描述写法（SEP-1382 未合入）、glab 最新 v1.114.0 且已有实验性 `glab mcp`——这些细节讲对了就是可信度，讲错了（比如说"MCP 官方有描述规范"）会被弹幕抓。

**风险提示**：第 4 集原 demo 叙事（"自研 MCP 编错参数重试三轮 vs gh 一次到位"）与本课程 spike 实测方向冲突（实测结论是税在五条件下才显性化）。本调研进一步确认：公开社区预期恰恰站在原叙事一边（"自研必翻车"），所以按实测反转讲既有真实性又有认知差——但 demo 布景必须体现五条件之一（如文档缺失/无 schema），否则拍不出翻车。
