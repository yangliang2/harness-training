# 反范式课 第 3 集：改掉这个习惯，让 AI 一学就会（无脑自建 MCP）

> 时长 5 分钟 | 交付物：判税三问决策卡 | 前置：第 1、2 集
> 状态：初稿（2026-08-23，据 `antipattern-course/ep03.md` 逻辑稿 v1 生成；口播文案待用户过审；**三臂主镜头数字占位，待 KB spike 回填**）
> 格式说明：沿用三分法——【实拍】= 终端真跑，禁止生成替代；【真截】= 真实网页/文档截图，禁止生成替代；【可生成】= 制作卡（card2png 管线），卡上数据/引语必须逐字可核。
> 数字纪律：三臂对照数字为 **KB spike 占位**（未跑，三种结果各有预写口径，见 1:40 段与设计档生死判据）；历史两轮数字（10/10、7/8、11.6 轮/$0.368 vs 13.2 轮/$0.459）不上正片，只作防御与边界口径；引用外部数字必须标出处等级（官方博客/预印本/厂商自评/社区实测）。
> 证据提示：`spike/` 不入库；**开拍前待办：KB spike 三臂 A/M0/M1**（千篇级夹具、任务混合预登记、翻车阈值预登记，见 Demo 步骤）。
> 禁说（正片与字幕一律不出现）：知识库 MCP 都是错的 / Skills 淘汰 MCP / 版本历史免费是我们的发现 / grep 永远够用 / 自研比 gh 强 / 模型更会用我们的工具 / AI 熟你的 CLI / "56% 的 MCP 会让 agent 翻车"（56% 是描述 smell 比例）/ agentic search 比 RAG 好当新知讲。

## 逐字稿

### 0:00–0:20 承接 + 行为点名

【口播】
上一课治的是别人塞给你的烂工具。这一课，轮到你自己造的时候。要让 AI 用上团队知识库，你的第一反应是什么？是不是——写个 MCP server？搜索、增删改查、历史记录，一套工具，两周开发。先别动手。这一集可能帮你省下这两周。

【画面】
承接卡 1 秒【可生成】→ 自研知识库 MCP 工具列表滚动（kb_search / kb_create / kb_update / kb_get_history…）【实拍，KB spike 夹具 M0】。

### 0:20–0:50 原理立柱

【口播】
它的"会"，只有两个来源：**训练时见过，或者干活时读到**。两个都没有，它只能猜。所以自建工具是一笔账：全部的收益在"贴合"——接口比通用工具更贴你的活；全部的风险在"分布"——它从没见过你。造轮子，本质是拿贴合的收益，去换分布的税。那有没有一种轮子，税照交、收益是零？有。你们公司可能正在造。

【画面】
原理卡【可生成】两行递进："会 = 训练见过 ∪ 运行读到"→"自建 = 拿贴合收益换分布税"。

### 0:50–1:40 逐件对照（案例即原理）

【口播】
就是知识库。把那套自研工具摊开看。kb_search？它原生有 Grep——检索这件事它熟到什么程度：Claude Code 连向量索引都不用，官方文档原话，**命令行工具是最省上下文的方式**。kb_create、kb_update？它原生有 Write 和 Edit。kb_get_history？git log、git diff、git blame——版本历史、分支、协作，用 Karpathy 的话说，**免费白拿**。你造的每一件，都是它已经会的东西的付费复刻：分布税全额支付，贴合收益一分没有。最狠的是历史那格：git 那边免费，你这边还得自己写一套版本系统。

【画面】
逐件对照表【可生成】三行逐行点亮（kb_search→Grep / kb_create·update→Write·Edit / kb_get_history→git log·diff·blame）。Claude Code 官方 best practices "CLI tools are the most context-efficient way" 真截【真截】。Karpathy llm-wiki gist "You get version history, branching, and collaboration for free" 真截【真截】。

### 1:40–2:20 三臂主镜头【占位待 spike】

【口播】
同一套知识库任务——查资料、改条目、追一条约定的来历——我们跑了三组。A 组：markdown 进 git，原生工具直接干。M0 组：知识库 MCP，文档认真写到合格。M1 组：同一个 MCP，文档写成市面中位数水平——公开研究扫过 103 个主流 server，一半以上连用途都没写清，我们照这个真实水平配。结果：____。【占位：按三臂结果择一叙事——①M1 翻车："这就是半数真实 server 的样子"；②M0 输 A："文档写好了也输——输的是形态，不是文档"（最优先）；③M0 ≈ A："打平——但你为平局多付了开发、文档、和每一轮的 token 三份钱"。预登记见设计档。】

【画面】
三栏分屏实拍【实拍，KB spike 夹具】→ 结果数字卡【可生成，占位待填，字幕标 N 与任务混合比例】。

### 2:20–3:20 解法：判税三问

【口播】
以后每次有人说"给 AI 写个 MCP 吧"，先过三问。**第一问：它是不是已经会了？**文件、git、grep、主流命令行覆盖的，别造。知识库的正解就在这——markdown 进 repo，读写、检索、历史全是它的原生技能。别觉得这是土办法：Anthropic 自家给 agent 装知识的官方载体 Skills，就是 markdown 文件夹进 git，现在还是多家采纳的开放标准。**第二问：真有它不会的业务逻辑，要造——选它能自学的形态。**命令行加 --help 优先：它没见过你的工具，但它见过一亿次"不会就 --help"。接口照主流的抄——GitLab 官方 CLI 就是照着 GitHub CLI 做的；企业内部要拼需求、CI、提交这一整条链，更要照着主流的命令面拼。**第三问：只能上 MCP**——远程服务、要 OAuth、用户没有终端——那把文档当提示词写，报错也是文档的一部分。存量的 Confluence、Notion？先查官方 server，别自己重复造。

【画面】
三问决策树卡【可生成】逐问点亮 → 历史 spike 转录特写【实拍，spike/verify/ep04-multi-mcp/ 转录】：模型面对陌生 CLI **自发**打出 `--help` 的那一行，字幕："没人教它这么做。" → glab README "Inspiration" 节真截【真截】。

### 3:20–4:10 诚实边界（三句丑话 + 一个总账）

【口播】
三句丑话，加一个总账。丑话一：**写知识的人不在 git 工作流里，这套不适用**——接企业知识库 MCP 的用户里，近一半来自非工程团队，他们没有终端，托管 MCP 就是他们的正当通道；这集说的是工程团队自研那种。丑话二：**工单、状态机、业务数据，别往 git 里搬**——那是数据库的活，反着犯病一样是病；上一课的缺陷库就该是 MCP，所以我们治它的输出，而不是劝你搬家。丑话三：**文档真堆到几万篇，纯靠盲搜会翻**——但先别急着建服务，下一集就讲怎么用一页索引解决。总账：小规模，你的 MCP 输给文件加 git；真大规模，它输给厂商级检索产品——**自建知识库 MCP，在哪个量级都不是最优解**。真被合规逼着自建？立正经项目，配专职团队，不是顺手写仨工具。

【画面】
三例外卡【可生成】逐条弹出（非工程用户 44% 角标：厂商官方博客口径）→ 挤压图【可生成】：横轴语料规模，左端"文件+git 赢"（角标：文件系统 agent 小语料实测占优，预印本）、右端"厂商检索产品赢"（角标：语义检索 +12.5%，厂商官方博客、代码库语境）、中间标"自建三件套：无获胜区间"。

### 4:10–4:35 收尾判据

【口播】
判据带走：**造工具之前，先问它是不是已经会了。**给它已经会的东西套一层它没见过的壳，是花钱教它变笨。三问顺序不变：原生会吗？会自学的形态？只能 MCP，就把文档当提示词写。

【画面】
判税三问卡【可生成，card2png】停 3 秒供截图。

### 4:35–5:00 交付物 + Cursor + 下集钩子

【口播】
描述区一张卡：正面三问，背面逐件对照表和三条例外。Cursor 用户照用——它一样跑终端、一样读文件，判据一个字不用换。知识库搬进 repo 了？别高兴太早——堆进来不等于能用，它会把三年前的复盘当成今天的事实。下一集：怎么喂、怎么验。下集见。

【画面】
卡正反面一屏【可生成】→ 下集预告卡【可生成】："让 AI 不记错事"。

---

## Demo 详细步骤

### KB spike 三臂（唯一 blocking，预算 3 × N=8 × ~$0.4 ≈ $10）

- **夹具**：千篇级 markdown 知识库（= 文件方案实证获胜区间；含版本演化历史供 git log/blame 任务），同一套语料三种交付形态：
  - A：repo 直挂，原生 Read/Edit/Grep/Glob + git；
  - M0：stdio MCP server（真协议，挂 `--mcp-config`），kb_search/kb_create/kb_update/kb_get_history，description 按六成分写到合格线；
  - M1：同 server，description 按 arXiv:2602.14878 六 smell 中位数配烂（用途不清 + 参数无 intent + 无使用指引），**报错同烂**（`internal error` 一句，不递答案）。
- **任务混合（预登记，不挑食）**：增删改查、跨文档检索、历史追溯（"这条约定什么时候改的、为什么"）按固定比例混合，比例随夹具定稿写死后再跑。
- **翻车阈值（预登记，随夹具定稿量化）**：成功率差 X、工具错误次数 Y、轮次/成本差 Z——三档口径先写死再开跑，防事后找补。
- **生死判据**：M1 翻 → 叙事①；M0 输 A → 叙事②（最优先）；M0 ≈ A → 叙事③（三重成本账）。三个出口同一骨架，集子不赌 spike。
- **选条纪律**：录中位数条；--help 自发镜头选历史 spike 转录（ep04-multi-mcp，历史编号目录）。
- M2（误导档）可选加拍，先补误导流行度出处，否则只按"可构造极端情形"收窄措辞（设计档待定问题 #2）。

### 开拍当月复核

- Claude Code best practices 原文（"CLI tools are the most context-efficient way…" + git-history 示范 prompt）仍在线且措辞未变；
- Skills / Agent Skills 开放标准状态与采纳名单（agentskills.io）；官方口径仍为 complement MCP；
- Notion 开源 server "may sunset" 声明现状；Atlassian MCP GA 状态（只说 GA，不报版本号——版本号无一手）；
- MCP 规范当前版（拍摄时复核是否仍为 2026-07-28、resources/tools 定义与知识库指导空位是否仍成立）。

---

## 交付物文件内容

判税三问决策卡（待建 `antipattern-course/deliverables/ep03-tool-tax/`，入版本库）：
- 正面：三问决策树（原生会吗 → 会自学的形态 → 只能 MCP 过文档合格线）+ 裸写验收格（"让它裸写一段调用——一次写对零税，查文档写对低税，查完还编错高税"）；
- 背面：逐件对照表（kb_* vs 原生）+ 三条例外（非工程用户 / 细粒度权限 / 事务性业务数据）+ 合规自建活口一句。
- **Cursor 对应栏**：Cursor 同样以 terminal + 文件工具为原生面，判据不换；知识文件对 Cursor 走 AGENTS.md 常驻（与第 4 集对应栏同口径）。

---

## 事实核对清单

| # | 本集事实声明 | 来源 | 状态 |
|---|---|---|---|
| 1 | 历史两轮：help 合格自研 10/10、三工具 7/8 零工具错误、11.6 轮/$0.368 vs gh 13.2 轮/$0.459 | spike/verify/ep04-gh-vs-mcp/ + ep04-multi-mcp/（历史编号，属本集） | ✅ 自有实测；**不上正片**，防御与边界口径用 |
| 2 | 三臂 A/M0/M1 对照数字 | KB spike | ⏳ 占位，三出口口径已预写 |
| 3 | 103 server/856 工具，97.1% 有 smell、56% 用途不清 | arXiv:2602.14878（2026-02） | ✅ 一手；56% 是描述 smell 比例，禁翻译成翻车率 |
| 4 | "CLI tools are the most context-efficient way to interact with external services" + git-history 示范 prompt | code.claude.com/docs best practices（2026-08-23 抓取） | ✅ 官方一手，真截 |
| 5 | Claude Code 无 embeddings、agentic search 起家 | Latent Space 播客 Cherny 原话 + how-claude-code-works 官方文档 | ✅ 一手；播客含 "mostly vibes"，不称严格实验结论 |
| 6 | "You get version history, branching, and collaboration for free" | Karpathy llm-wiki gist（2026-04，5k+ stars） | ✅ 一手，真截；**借力引用，禁称独家** |
| 7 | Skills = 文件夹形态、2025-12-18 起开放标准、OpenAI/Cursor 阵营采纳 | claude.com/blog/skills + agentskills.io | ✅ 一手；官方口径 complement MCP，禁说淘汰 |
| 8 | glab 照 gh 做（Inspiration 节） | GitLab CLI 官方 README | ✅ 一手（2026-08-20 核实），真截 |
| 9 | Atlassian MCP 日 500 万调用、44% 用户非软件团队、1/3 写操作 | Atlassian 官方博客（2026-07-01） | ✅ 一手；厂商自评口径，字幕标注 |
| 10 | 文件系统 agent 小语料赢、51 万篇级掉 ~20 分且 39–60× query token | arXiv:2607.26497《BM25 Wins at Scale》 | ✅ 一手；测的是**裸盲搜 agent**，禁外推到索引形态 |
| 11 | Cursor 语义检索 agent 准确率 +12.5% | cursor.com/blog/semsearch（2025-11） | ✅ 一手；代码库 + 自训 embedding 语境，一句交代不展开 |
| 12 | MCP spec 2026-07-28：resources=application-driven / tools=model-controlled，知识库场景零指导 | modelcontextprotocol.io 规范原文 | ✅ 一手；开拍当月复核版本 |
| 13 | Notion 开源 server "may sunset" / 21 工具 ~26k tokens | makenotion GitHub README / notion-slim 项目自测 | ✅ 前者官方一手；后者标"社区实测" |
| 14 | 禁说清单（页首九条） | —— | ❌ 正片与字幕一律不出现 |

**未引用、不越界的弹药**：文档六成分合格线细则与兼容外壳方法论展开（描述区/延伸阅读）；工具数量 context 税（第 1 集已拍，本集不重讲）；MCP 输出税（第 2 集领地）。
