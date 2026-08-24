# 自建知识库 MCP vs docs as code（文件+git）：行业认知调研

调研日期：2026-08-23。所有标注"2026-08-23 抓取"的 URL 均于当日实际抓取核实；个别沿用 2026-08-20 前次调研（`out-of-distribution-tooling.md`）已核实的材料，单独标注。服务对象：反范式课第 4 集候选主题裁决——"反模式 = 为企业/团队知识库自建 MCP/检索服务（CRUD+搜索+历史三件套）；解法 = 知识以 Markdown 进 git repo，agent 用 Read/Grep/Glob + git 原生操作"。

判定刻度：【常识】= 高频讨论、可视为行业共识；【有人讲】= 有高质量出处但讨论密度有限；【几乎没人讲】= 只有零星材料或本课程独有。

---

## 总览判定

| 问题 | 判定 |
|---|---|
| 1. 反模式是不是行业常识 | **拆两层**：底层原理"agent 用文件+grep 胜过 RAG/检索服务"在 coding-agent 圈已是【常识】（Boris Cherny 名言病毒式传播 + Karpathy 2026-04 gist 爆红）；但把"自建知识库 MCP"点名为反模式、与 docs as code 对置的选型框架【几乎没人讲】 |
| 2. 主流解法与观众水平 | "agentic search vs RAG"争论【常识】（且 2025-11 起有 Cursor 官方数据的强反方）；"docs as code"一词被用作 AI 选型论据【有人讲】（做法流行、术语未被征用）；Claude Code 不用 embeddings 的一手出处已锁死（Latent Space 播客原话 + 官方文档，见第 2 节） |
| 3. 公开翻车案例/数据 | 【有人讲】且两边数据都硬——正方：官方 Notion MCP 21 工具 ~26k tokens、同功能 MCP 输入 token 差 4.98×；反方：Cursor 实测语义检索让 agent 准确率 +12.5%、BM25-at-scale 论文实测文件系统 agent 大语料下崩盘且 39× token 成本（诚实边界数据，见第 3、6 节） |
| 4. 认知差 | ①"版本历史免费"【有人讲】——**Karpathy gist 原话与课程口径逐字级撞车**（"You get version history, branching, and collaboration for free"），独家口径失守，须改打法；②"KB MCP 复刻原生能力=纯亏"结构【有人讲】（零星）——Filesystem-MCP-vs-cat/grep 的复刻结构被点破过，但"分布税全付、贴合红利为零"的双轴经济学框架无人给 |
| 5. 官方现状 | 已全部一手核实（Skills 开放标准 2025-12-18 起、Claude Code 仍无索引/embeddings、Notion 本地开源 server 官方明言可能 sunset、Atlassian MCP GA、MCP spec 2026-07-28 resources=application-driven 且对知识库场景零指导），见第 5 节 |

---

## 1. 反模式本身是不是行业常识

**判定：原理【常识】，点名【几乎没人讲】。**

底层原理——"agent 拿文件+grep 做检索胜过专门检索服务"——在 coding-agent 圈已经讲烂：

- **Boris Cherny（Claude Code 作者）在 Latent Space 播客（2025-05）**的原话是该圈层的病毒级引文："one is it outperformed everything. By a lot. By a lot. And this was surprising."（指 agentic search 对比早期 RAG+本地向量库版本）；"agentic search just sidesteps all of that. So essentially, at the cost of latency and tokens, you now have really awesome search without security downsides." 来源：[latent.space/p/claude-code](https://www.latent.space/p/claude-code)（2026-08-23 抓取）。他后来在 X 上的同口径表述（"Early versions of Claude Code used RAG + a local vector db, but we found pretty quickly that agentic search generally works better. It is also simpler and doesn't have the same issues around security, privacy, staleness, and reliability."，[x.com/bcherny/status/2017824286489383315](https://x.com/bcherny/status/2017824286489383315)）**未能直接抓取 X 原页**，内容经搜索快照与播客原话互证，仅作密度佐证。
- **Karpathy 2026-04-04 发布 llm-wiki gist**（[gist.github.com/karpathy/442a6bf555914893e9891c11519de94f](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)，2026-08-23 抓取，5,000+ stars）：主张对个人/小团队知识库不要默认 RAG，改让 LLM 维护"a structured, interlinked collection of markdown files"；对比 RAG 的措辞是 RAG "rediscovers knowledge from scratch on every question"。大 V + 爆款传播（gist 数日破 5k stars），把"markdown 文件目录当知识库"推入大众视野。
- 2026 上半年出现一批"你的 agent 可能不需要 RAG"腰部文章（如 [dev.to/remybuilds](https://dev.to/remybuilds/considering-rag-for-your-agent-build-this-instead-4ihf)、landeranalytics 等，2026-08-23 搜索确认存在，质量中等），说明该论点已下沉到二线传播层——**下沉到腰部即接近常识**。

**但是**：以上全部是"文件 vs RAG/向量库"的叙事。把**自建知识库 MCP server（增删改查工具+搜索工具+历史记录工具）**点名为反模式、并与 docs as code 对置的框架，公开材料里没有成体系的版本。反证：2026-03 的 HN 447 分帖《MCP is dead, long live the CLI》评论区（[news.ycombinator.com/item?id=47208398](https://news.ycombinator.com/item?id=47208398)，2026-08-23 复抓核实）**没有任何针对知识库/文档/Notion/Confluence MCP 的专门讨论**，最接近的一条反而是替文档 MCP 说话的（"One clear use case where MCP is better than anything else: design system documentation… Context7 is a good MCP"）。即：CLI-vs-MCP 大讨论打的是运维工具，知识库场景是讨论空位。

## 2. 主流解法与观众水平

**"agentic search vs RAG"【常识】且有强反方；"docs as code"术语征用【有人讲】；Claude Code 一手出处已锁死。**

- **Claude Code 不用 RAG/embeddings 的一手出处**（课程可直接引用的三件套，均 2026-08-23 抓取）：
  1. Latent Space 播客原话（上节，含 benchmark 追问的诚实答复："This was just vibes, so internal vibes. There's some internal benchmarks also, but mostly vibes."——引用时别把它说成严格实验结论）；
  2. 官方文档 [code.claude.com/docs/en/how-claude-code-works](https://code.claude.com/docs/en/how-claude-code-works)：内置工具就是 File operations / Search（"Find files by pattern, search content with regex"）/ Execution（"run shell commands… use git"），无任何索引/embedding 组件；
  3. 官方 best practices [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)："**CLI tools are the most context-efficient way to interact with external services**"——官方把 CLI 置于 MCP 之前推荐，并给出 git-as-knowledge 的官方示范 prompt："look through ExecutionFactory's **git history** and summarize how its api came to be"。
- **反方是官方且带数字的**：Cursor 2025-11-06 博客《Improving agent with semantic search》（[cursor.com/blog/semsearch](https://cursor.com/blog/semsearch)，2026-08-23 抓取）：自训 embedding 模型后，agent 回答准确率平均 **+12.5%（按模型 6.5%–23.5%）**，大代码库（1,000+ 文件）上 code retention +2.6%。配套 X 口径"especially in large codebases where grep alone falls short"。Milvus（向量库厂商，利益相关）博客《Why I'm Against Claude Code's Grep-Only Retrieval》（[milvus.io](https://milvus.io/blog/why-im-against-claude-codes-grep-only-retrieval-it-just-burns-too-many-tokens.md)，2026-08-23 抓取）：混合检索下 "Token usage dropped by over 40%, without any loss in recall"。**争论现状是两派均有官方/数据背书，不是一边倒**——2026-05 的综述帖（wowelec《Agentic, Semantic, or Both?》）也停在"agentic 为骨、语义索引按需"的折中。
- **"docs as code"术语征用**：做法在扩散、术语没被征用。密度证据：Show HN《We want to displace Notion with collaborative Markdown files》仅 29 分/14 评（[news.ycombinator.com/item?id=47236374](https://news.ycombinator.com/item?id=47236374)，2026-08-23 抓取），卖点确实是 "coding agents (claude, amp, copilot, opencode, etc.) are good enough now"；openknowledge.sh 等小产品打 "Markdown Knowledge Bases for AI Agents"。用"docs as code"四个字做 AI 选型论据的头部出处：没找到。**课程把这个词征用为 AI 选型论据的空间还在，但"markdown+git 放知识"这件事本身观众很可能已经在做**（CLAUDE.md/skills 就是官方教的这个形态）。

## 3. 公开翻车案例/数据

**判定：【有人讲】，正反数据都硬。**

正方（KB/服务式 MCP 的税）：

- **官方 Notion MCP 的 token 账**：开源版 21 个工具吃掉 **~26,073 tokens** 工具定义（notion-slim 项目实测并以此立项，声称重组后 -73%：[github.com/mcpslim/notion-slim](https://github.com/mcpslim/notion-slim)，2026-08-23 搜索确认，一手为该项目自测）；社区实测 7 个 MCP server 光工具定义吃 67,300 tokens（deploystack，2026-08-20 前次调研核实过同类数据）。搜索工具返回"a dump of whatever they decide to return… JSON data dumps with many token-unfriendly data-points like identifiers and URLs"是 HN 反复出现的吐槽口径（2026-08-23 搜索多处确认，轶事级）。
- **同功能不同接口的 5× 实验**：HN《Bad MCP design costs your agent 5x more tokens》（[news.ycombinator.com/item?id=48407391](https://news.ycombinator.com/item?id=48407391)，2026-08-23 抓取）：同一后端两套 MCP，输入 token 差 **4.98×**、步骤差 1.29×。注意仅 17 分/1 评——数据点可用，密度不可夸大。
- **MCP-Universe**（arXiv:2508.14704，2026-08-20 前次调研核实）：真实 MCP server 上模型对不熟工具反复参数级犯错、GPT-5 成功率仅 43.72%——"模型没见过你的接口"的通用量化背书，可复用。

反方/边界（检索服务仍然正当的实测）：

- **《BM25 Wins at Scale》**（arXiv:2607.26497v3，USTC 等，2026-07-30；[arxiv.org/html/2607.26497](https://arxiv.org/html/2607.26497)，2026-08-23 抓取）：28 级语料阶梯（1,144→511,959 篇文档 / 1.7M→601M tokens）上，File-System Agent（只有 list/search/read 工具，80 次 LLM 调用预算）**小语料赢**（77.4 vs BM25 74.7），**全量级崩**（30.7 vs 50.5，差约 20 分），且 query token 成本是 BM25 的 **39–60 倍**。这是"grep 到多大规模不再可行"的最硬一手数据。
- **《Is Grep All You Need?》**（arXiv:2605.15184，PwC，2026-05-14；[arxiv.org/html/2605.15184v1](https://arxiv.org/html/2605.15184v1)，2026-08-23 抓取）：LongMemEval 116 题上 "inline grep exceeds inline vector for every harness–model pair"（最小差距 76.7% vs 75.0%），但结论强调交付方式/harness 会"reshuffle the comparison"——grep 赢在"recovering literal witnesses: exact dates, counts, preferences"类任务。
- **Atlassian 官方护盘数据**（[atlassian.com/blog/company-news/inside-rovo-mcp-usage](https://www.atlassian.com/blog/company-news/inside-rovo-mcp-usage)，2026-07-01 发布，2026-08-23 抓取）：官方 MCP 日调用 500 万+，**44% 的 MCP 用户来自非软件团队**，约 1/3 调用是写操作；并宣称 "agents grounded in Teamwork Graph context deliver 44% more accurate answers using 48% fewer tokens"（厂商自测，语境是其图谱上下文 vs 无图谱，不是 vs 文件+grep）。**这组数字反着用最有价值：企业知识库 MCP 的真实用户近半不是工程师——他们根本没有 git+终端这条路。**

## 4. 认知差（课程最关心）

**判定：①【有人讲】且撞车，②【有人讲】（零星、无框架）。**

1. **"版本历史免费"已被 Karpathy 原话说过**。llm-wiki gist（2026-04-04，一手，2026-08-23 抓取）："**The wiki is just a git repo of markdown files. You get version history, branching, and collaboration for free.**"——与课程预设口径逐字级重合。另有 DiffMem（[github.com/Growth-Kinetics/DiffMem](https://github.com/Growth-Kinetics/DiffMem)，902 stars，2026-08-23 抓取）把 git 历史当**检索与推理机制**而非附赠："No vector databases, no embeddings, no BM25 — just git and an LLM"；"Git diffs and logs provide a natural way to track how memories evolve. Agents can ask 'How has this fact changed over time?'"（agent 用 grep/git log/diff/blame 探索）。Claude Code 官方 best practices 也示范了 git history 当知识源的 prompt（第 2 节）。**结论：这条不能再当独家发现讲，但可以升级讲法——别人只说"免费拿到"，没人把它对照"自建 MCP 里那个要自己造的 history 工具"算成本账。**
2. **"复刻原生能力=纯亏"的结构被点破过一次，但没有经济学框架。** Tomaz Bratanic《One Flexible Tool Beats a Hundred Dedicated Ones》（Towards Data Science，2026-05-18；[towardsdatascience.com](https://towardsdatascience.com/one-flexible-tool-beats-a-hundred-dedicated-ones/)，2026-08-23 抓取）："**Filesystem MCPs vs. `cat`, `ls`, `mv`, `grep` glued by pipes. Same instinct every time, same CLI counterpart every time.**"——点名了"给 agent 造专用工具=复刻它已有的 CLI 对应物"这个结构。但：对象是泛型 filesystem MCP 而非知识库 MCP；论证轴是灵活性/可组合性，**没有人用"分布税全额支付 + 贴合红利为零 → 纯亏"的双轴框架算这笔账**，更没有人把"知识库 MCP 的 search 工具 vs Grep、history 工具 vs git log"逐件对照。这个框架位仍然是空的。

## 5. 官方现状一手核实（2026-08-23 当日口径）

- **Anthropic Skills**：2025-10-16 发布（[claude.com/blog/skills](https://claude.com/blog/skills)，原 anthropic.com/news/skills 308 重定向至此，2026-08-23 抓取）："Skills are folders that include instructions, scripts, and resources"。2025-12-18 起成为开放标准 **Agent Skills**（[agentskills.io](https://agentskills.io)，2026-08-23 抓取）：skill = 含 SKILL.md 的文件夹，三段渐进披露，官网原话把知识形态定死在文件系统——"packaging procedural knowledge and company-, team-, and user-specific context into **portable, version-controlled folders**"；采用方 showcase 含 OpenAI ChatGPT/Codex、Cursor、GitHub Copilot、Gemini CLI 等数十家。与 MCP 的官方关系口径（工程博客 [anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)，2026-08-23 抓取）：探索 Skills "complement Model Context Protocol (MCP) servers"——**官方叙事是互补不替代，课程别讲成"Skills 淘汰 MCP"**。但结构事实站在课程一边：Anthropic 自家的"给 agent 装知识"官方载体就是 markdown 文件夹进 git，不是 MCP server。
- **Claude Code 检索机制**：仍无 embeddings/索引。官方文档口径见第 2 节两个链接（内置 Search=glob/regex/文件读取；上下文管理靠 CLAUDE.md/auto memory/skills，全部是 markdown 文件）。Simon Willison 对 Skills 的独立判断可作旁证（[simonwillison.net/2025/Oct/16/claude-skills/](https://simonwillison.net/2025/Oct/16/claude-skills/)，2026-08-23 抓取）：Skills 是 "Markdown with a tiny bit of YAML metadata and some optional scripts"，对比 MCP 的 "whole protocol specification, covering hosts, clients, servers…"，且 "LLMs know how to call `cli-tool --help`"。
- **Notion MCP**：官方 hosted server（mcp.notion.com，OAuth）为唯一主推（[developers.notion.com/docs/mcp](https://developers.notion.com/docs/mcp)，2026-08-23 抓取）；开源本地版 [github.com/makenotion/notion-mcp-server](https://github.com/makenotion/notion-mcp-server)（4.6k stars，2026-08-23 抓取）README 一手原话："We are prioritizing, and only providing active support for, Notion MCP (remote)"、"**We may sunset this local MCP server repository in the future**"、"Issues and pull requests here are not actively monitored"。
- **Atlassian MCP**：官方仓库 [github.com/atlassian/atlassian-mcp-server](https://github.com/atlassian/atlassian-mcp-server)（2026-08-23 抓取）：官方徽章 "Status: Generally Available"，覆盖 Jira/Confluence/JSM/Bitbucket/Compass 读写搜。二手材料称 2026-02 GA、当前 v1.3.0（2026-06-08）——**版本号未在官方页一手核到，引用时只说 GA、不报版本号**。
- **MCP 规范 2026-07-28（当前版）resources vs tools**：Resources 章（[modelcontextprotocol.io/specification/2026-07-28/server/resources](https://modelcontextprotocol.io/specification/2026-07-28/server/resources)，2026-08-23 抓取）："Resources in MCP are designed to be **application-driven**, with host applications determining how to incorporate context"——即规范里给"文档/知识"准备的原语**不是**给模型主动检索用的；Tools 章（[…/server/tools](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)，2026-08-23 抓取）："Tools in MCP are designed to be **model-controlled**"。规范定义了 `git://` URI scheme 但仅一行带过；**全规范对"知识库/文档访问该用 resources 还是 tools、该怎么建模"零指导**。这是可引用的可信度细节：连协议自己都没给知识库场景答案，而市面知识库 MCP 全部把检索做成 tools（Notion 18 个、Atlassian 数十个）——等于在 model-controlled 轨道上手工重造 application 层。

## 6. 附加：诚实边界（知识库服务/RAG 仍然正当的条件）

1. **语料规模**：最硬阈值数据是《BM25 Wins at Scale》（第 3 节）：纯文件系统 agent 在小语料（~千篇级）赢，**~50 万文档/600M tokens 级别掉 20 分且 39–60× query token**。Cursor 实测 1,000+ 文件的代码库语义检索优势放大（+12.5% 准确率语境）。课程口径建议：团队知识库通常在"千篇 markdown"量级——正好落在文件系统 agent 的获胜区间；但引用时必须交代曲线另一端。
2. **细粒度权限**：hosted MCP 的 OAuth per-user 权限（Notion "with your permissions"、Atlassian OAuth 2.1）是 git repo 权限模型（repo 级粒度）给不了的；MCP spec 明文允许 tools/resources 列表随凭证 scope 变化（2026-07-28 规范原文，2026-08-23 抓取）。页面级/字段级权限需求 = 别进 git。
3. **非工程用户**：Atlassian 官方数据（2026-07-01）：**44% 的 MCP 用户来自非软件团队且逐月上升**。这些人没有终端、不会 git——Wiki UI + hosted MCP 是他们唯一可用的形态。docs as code 的适用前提是"写知识的人在 git 工作流里"。
4. **事务性/结构化业务数据**：Atlassian 数据显示 1/3 MCP 调用是写操作（建 issue、更新工单）——工单流转、状态机、并发写入是数据库业务，不是文档，进 git 是错的。课程需把"知识库"与"业务系统"切开。
5. **反方最强单点**（威胁项，必须在片中主动交代）：Cursor 官方 A/B——语义检索全模型提升 agent 准确率，"especially in large codebases where grep alone falls short"（[cursor.com/blog/semsearch](https://cursor.com/blog/semsearch)）。应对口径：那是自训 embedding + 海量 session 数据的厂商级投入，与"团队自建 KB MCP"不是一个物种；且同期学术数据（Is Grep All You Need）显示小-中语料 grep 仍全面占优。

## 7. 对课程定位的启示

**观众早知道（讲了会水）：**

- "agentic search 比 RAG 好、Claude Code 不用 embeddings"——Cherny 名言 + 一堆解读文 + Karpathy gist，coding-agent 圈月经话题。当背景板一句带过，不能当发现。
- "markdown 文件当个人知识库"——Karpathy llm-wiki 之后已是大众玩法（衍生教程成灾）。
- "MCP 工具定义吃 token"——HN 反复讨论过，且官方已有 Tool Search/代码执行等缓解方案，单讲 token 账不新鲜。

**真盲区（讲了值钱）：**

1. **点名"自建知识库 MCP"这个具体物种**：CLI-vs-MCP 大战没打到知识库场景（HN 447 分帖零讨论，2026-08-23 复核）；"你司那套 kb_search/kb_create/kb_history 工具，每一件都是 Grep/Write/git log 的付费复刻"的逐件对照无人做过。
2. **税/红利双轴经济学框架**：TDS 文章点破过复刻结构但只论灵活性；"分布税全额支付、贴合红利为零→纯亏"的框架位是空的。注意措辞：说"没人给这个框架"，别说"没人讲过复刻结构"。
3. **"版本历史免费"改为借力打法**：Karpathy 原话（"You get version history, branching, and collaboration for free"）+ DiffMem 的 git-log-as-retrieval + Claude Code 官方 git-history prompt 三点连线——课程的增量是把它对照"自建 MCP 里 history 工具的建造+学习成本"算账，而不是宣称发现。**若按原计划宣称独家，会被懂行观众用 Karpathy 原话打脸。**
4. **MCP 规范自己的知识库空位**：resources=application-driven / tools=model-controlled 的错配 + 规范对文档场景零指导——讲对了是硬核可信度，市面没人讲这个角度。
5. **诚实边界本身就是内容**：规模阈值（BM25-at-scale 的 39×/20 分曲线）、44% 非工程用户、1/3 写操作——这三组数字把"什么时候该建服务"讲清楚，恰好是"纯亏轮子"绝对化叙事的解毒剂。

**威胁项/风险提示：**

- **理论框架的"纯亏"二字过强**：Cursor +12.5% 与 BM25-at-scale 都证明检索服务在大语料下有真实贴合红利（准确率与 token 效率）；"贴合红利为零"只在**中小规模、工程团队、文本知识**的限定域内成立。框架保留可以，全称命题必须收窄，否则被一条 Cursor 博客反杀。
- **"版本历史免费"独家口径已失守**（Karpathy 逐字撞车），需按上文改为借力。
- **Skills-MCP 关系别讲过头**：官方口径是 complement 不是 replace（工程博客原话），而且 MCP 已捐 Linux Foundation、Agent Skills 被 OpenAI 阵营采纳——"文件形态赢了协议形态"可以作为趋势观察，不能说成官方结论。
- Atlassian/Notion 官方 MCP 都是 GA 且用户量真实（日 500 万调用）——"知识库 MCP"作为**厂商 hosted 产品**是成立的市场；课程打击面必须收在"**自研自建**"上，别误伤官方 hosted server（那是给非工程用户的正当通道）。
