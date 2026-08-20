# 文档坟场（stale docs 误导 AI agent）行业认知调研

调研日期：2026-08-20（所有 URL 均为当日抓取）。服务对象：反范式课第 3 集「改掉这个习惯，让 AI 不记错事」（文档坟场；解法 = doc-audit skill 分诊 + 劝退堆料）。关联实测：`spike/verify/ep03-doc-graveyard/CONCLUSION.md`（简单布景 4/4 压不出喂毒，风险收窄为"代码无法反驳"场景）。

来源分级：一手 = 官方文档/arXiv 论文/官方博客；高质量 = 知名工程博客、HN 高赞讨论、可复现的开源 benchmark；营销软文只作线索，已逐条标注。

---

## 问题 1：这个反模式是行业常识吗？——判定【常识】

讨论密度高，且有约定俗成的名字。证据：

**讨论密度（HN Algolia，2026-08-20 查询）**

- `AGENTS.md` 相关 story 共 **235 条**。头部线程：
  - [AGENTS.md – Open format for guiding coding agents](https://news.ycombinator.com/item?id=44957443)（2025-08，837 赞 / 382 评论）
  - [Evaluating AGENTS.md: are they helpful for coding agents?](https://news.ycombinator.com/item?id=47034087)（2026-02，232 赞 / 161 评论）——即 arXiv 2602.11988 的社区讨论
  - [AGENTS.md outperforms skills in our agent evals (Vercel)](https://news.ycombinator.com/item?id=46809708)（2026-01，524 赞 / 196 评论）
  - [A good AGENTS.md is a model upgrade. A bad one is worse than no docs at all (Augment)](https://www.augmentcode.com/blog/how-to-write-good-agents-dot-md-files)（2026-04，HN 142 赞）
  - Ask HN 系列：[Is it better to have no Agent.md than a bad one?](https://news.ycombinator.com/item?id=47124103)、[Do you still spend time maintaining Claude.md / AGENTS.md files?](https://news.ycombinator.com/item?id=48160604)
- 对照：精确短语 `"documentation rot"` 在 HN 仅 **12 条**命中且多为老帖——**术语本身冷门，讨论都发生在 AGENTS.md/CLAUDE.md 框架下**。讲课时用"文档坟场/文档腐烂"自有词没问题，但引用社区证据要对准 context-file 话语体系。

**约定俗成的名字**（多个并存，无单一标准词）

- *documentation rot / doc rot*：最老的叫法，[Devonair: Documentation Rot](https://devonair.ai/blog/pain-points/documentation-rot)（2025-11）；[fiberplane: We built a linter for documentation rot](https://fiberplane.com/blog/drift-documentation-linter/)（2026-03）。
- *context rot*：AI 语境下的迁移叫法，arXiv 论文 [Context Rot in AI-Assisted Software Development](https://arxiv.org/html/2606.09090v1)（2026-06）直接以此为题；开源插件 [codeplow](https://github.com/waelmas/codeplow) 自述 "fight context rot and doc rot"。
- *documentation drift*：2026 年起被当作运维风险讨论，[Moxie Docs 综述](https://moxiedocs.com/blog/documentation-drift-a-news-roundup-of-2026-trends)（2026-08，营销性质，仅作趋势线索）。
- *stale CLAUDE.md 比没有更糟*已是社区老生常谈：[DEV.to: Your CLAUDE.md Is Lying to Your Agent — Why a Stale Instructions File Is Worse Than None](https://dev.to/alfredoizjr/your-claudemd-is-lying-to-your-agent-why-a-stale-instructions-file-is-worse-than-none-34od)（2026-06）；[IngestThis 指南](https://ingestthis.com/posts/2026/2026-03-07-context-management-blogs-06-claude-code)："A CLAUDE.md that references a framework you migrated away from six months ago actively misleads the agent."

**官方背书**（说明已进入厂商视野）：[Claude Code 官方 memory 文档](https://code.claude.com/docs/en/memory)明确要求 "Review your CLAUDE.md files … periodically to remove outdated or conflicting instructions"，并警告规则互相矛盾时 "Claude may pick one arbitrarily"。

## 问题 2：主流解法是什么？观众大概率已经会了吗？——判定【"写短点"是常识；系统性文档治理是"有人讲"】

**已是常识的层（观众大概率听过）**

- 写短、写具体、定期修剪。官方口径：[Claude Code memory 文档](https://code.claude.com/docs/en/memory)——"target under 200 lines per CLAUDE.md file. Longer files consume more context and reduce adherence."
- 拆分与按需加载：`.claude/rules/` 路径作用域规则、`@import`、monorepo 用 `claudeMdExcludes`（同上官方文档）；社区同构方案见 [Zenn: .claude/rules/ 动态加载](https://zenn.dev/tmasuyama1114/articles/claude_code_dynamic_rules?locale=en)。
- "坏的 context file 不如没有"：[Augment](https://www.augmentcode.com/blog/how-to-write-good-agents-dot-md-files)、[Upsun: your AGENTS.md is probably too long](https://developer.upsun.com/posts/ai/agents-md-less-is-more)（引 ETH 研究）。

**"有人讲"但未普及的层（课程的 doc-audit 落点）**

- **drift 检测工具链**（文档与代码锚定比对）：[fiberplane drift linter](https://fiberplane.com/blog/drift-documentation-linter/)（frontmatter 锚定代码片段，2026-03）；surface-bench 配套工具 `surf check`（见问题 3）；学术侧 [arXiv 2606.09090](https://arxiv.org/html/2606.09090v1) 系统梳理了传统文档一致性技术（code comment checker、API doc checker）向 AI 配置文件的迁移路线，结论：DOCER 类工具对"引用性腐烂"可直接迁移，其余需逐一评估。
- **归档/状态生命周期约定**：ADR 体系是最成熟的先例——编号永久保留、决策变更不改旧文档而是新建 ADR supersede，状态机 `proposed → accepted → [deprecated | superseded by ADR-NNNN]`（[Architecture as Code](https://aac.geon.se/04_adr/)、[OutcomeOps ADR 指南](https://www.outcomeops.ai/blogs/what-is-an-adr-and-why-theyre-critical-for-ai-powered-development)）。即业界对"历史文档"的标准答案是**不删、但显式标注状态**——与课程"archive/ 别读 + 分诊表"同构。
- **"prompt debt"概念**：[ADI Pod](https://adipod.ai/blog/claude-md-best-practices/)（2026-04）记录社区对 "agents.md 写完后没人再更新、与代码漂移" 的焦虑，称之为 prompt debt。

**观众会不会**：写短、拆分大概率会；但"定期逐份验证文档引用的接口还存在吗、产出带证据的分诊表"这一**审计动作**在公开材料里基本只见于工具厂商（fiberplane/surface）的自动化方案，"让 agent 自己做 doc-audit 分诊、人只批表"的玩法几乎没人讲——课程解法落在空白区。

## 问题 3：有没有公开翻车案例/数据？——判定【有人讲，且量化证据很强】

**量化研究（课程可直接引用的硬数据）**

1. **surface-bench**（[PAPER.md, Connor McDonald, 2026-06-16](https://github.com/Connorrmcd6/surface-bench/blob/main/PAPER.md)，2026-08-20 抓取）——**最对口的一项**：预注册、确定性评分（无 LLM judge）、5 模型 3 厂商、3,250 次完成。
   - 单发 pilot：依赖被隐藏（=代码无法反驳）时，过期文档让**所有模型成功率 0%、被误导率 100%**；准确文档恢复 100%；附自动 drift 报告恢复 90–100%。
   - 多轮 agentic 确认实验：过期文档下成功率 0–32%、被误导率 **68–100%**；fresh doc 恢复 94–100%。
   - 注意：这是单人 GitHub working paper（自称预注册、ABC 合规），未经同行评审——引用口径应为"开源可复现实验"，不是"学界定论"。但它的设计与课程 spike 结论（风险在"代码无法反驳"场景）**互为印证**：其 cascade 家族就是 hidden dependency 布景，comprehension 家族（代码可见）则成功率近天花板、只付 token 税——和 `spike/verify/ep03-doc-graveyard/` 的"agent 站代码"完全一致。
2. **Gloaguen et al., [arXiv 2602.11988](https://arxiv.org/abs/2602.11988)**（ETH Zurich，v1 2026-02-12，v2 2026-06-23，被引 30+，OpenReview 在审）：context file **不提升任务成功率，且推理成本平均 +20%**；LLM 生成与开发者手写的文件结论一致；被广泛推荐的 repo overview 部分**无帮助**。HN 讨论 161 评论。
3. **反方数据 Lulla et al., [arXiv 2601.20404](https://arxiv.org/html/2601.20404v2)**（2026-03）：策划良好的 AGENTS.md 让 agent 在 PR 任务上 **runtime -28.6%、输出 token -16.6%**（只测效率不测正确率）。
4. **Vercel 官方 eval**（[AGENTS.md outperforms skills in our agent evals](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)，2026-01）：skill 56% 的情况根本没被触发（+0pp）；8KB 压缩 docs index 嵌进 AGENTS.md 拿到 100% 通过率。注意这是"准确的、版本匹配的文档"的收益——恰好反衬"文档对了收益巨大、错了代价巨大"。

**叙事型翻车案例（轶事级，引用时降级表述）**

- [bswen: 300+ 行 CLAUDE.md 被 Claude 实际忽略](https://docs.bswen.com/blog/2026-04-23-prevent-claudemd-bloat/)（2026-04）——"我说过的规则它说在上下文里看不到"。
- [HN: Copilot committed my repo secrets into AGENTS.md](https://news.ycombinator.com/item?id=46764160)（2026-01）——agent 主动写坏文档的变体事故。
- 真实世界"被 repo 里旧文档带偏"的第一人称事故贴**意外地少**——公开翻车主要是 benchmark 形态，不是事故复盘形态。课程若有自己的实拍 demo，稀缺性反而高。

## 问题 4："大家都这么以为但其实是错的"认知差——判定【几乎没人讲，最值钱】

按值钱程度排序：

1. **"过期文档的问题在于它旧"——错，机制是"任何权威文档都会让 agent 停止验证"**。surface-bench 最反直觉的发现：fresh doc 对验证行为的抑制和 stale doc **一样强**（无文档时 agent ~100% 主动验证隐藏依赖，有文档就大幅不查）；过期文档的危害只是"它让你放弃检查的那件事恰好是假的"。三种失败模式：gpt-5.4 盲信（blind obedience）、Claude 系抑制验证但查了能自救、gemini-3.5-flash 查完仍然屈从文档（verify-then-defer）。**这直接重写课程叙事：不是"旧文档有毒"，是"文档天然替代验证，所以文档必须配审计"——doc-audit 的存在论依据。**
2. **"模型够强就能识破过期文档"——错，能力不免疫**。surface-bench："capability does not buy resistance — the strongest models are among the worst affected"（opus、gpt-5.4 被误导率 68–100%）。与大众"下一代模型会自己解决"的预期相反。
3. **"加一句'本文档可能过时'的警告就能防"——基本无效**。surface-bench H6：泛警告（Cw）恢复远逊于带修正证据的 drift 报告（C3 vs Cw 在所有模型上确认显著）；恢复来自**报告里的修正后代码**，不是"起了疑心"。→ 课程 CLAUDE.md 两行防线的措辞要据此校准："以代码为准"是优先级规则、比泛警告强，但实证支持的是**给证据**（分诊表/drift 报告），不是给提醒。
4. **"AGENTS.md 有用/没用已有定论"——其实学界结论是分裂的**。Gloaguen（降成功率、+20% 成本）vs Lulla（-28.6% runtime）结论相反；[Probe-and-Refine, arXiv 2606.20512](https://arxiv.org/html/2606.20512)（2026-06）的调和实验：同一份指导**在步数预算充裕时有益、紧张时有害**——"有没有用"取决于任务复杂度与预算，不是文件本身。这给"劝退堆料"提供了精确表述：堆的是"未经评估的文档"，不是"文档"。
5. **"文档越全越详细越好（尤其 repo overview）"——官方推荐里最流行的 overview 恰恰是最没用的部分**（Gloaguen：overview 无帮助，具体指令被遵守得最好）。Vercel 补充：40KB 全量文档压到 8KB 索引，100% 通过率不变。

## 问题 5：相关官方功能/命令现状（课程引用口径，2026-08-20 核实）

全部出自 [Claude Code memory 官方文档](https://code.claude.com/docs/en/memory)（2026-08-20 抓取）与 [agents.md 官网](https://agents.md/)（2026-08-20 抓取）：

- **CLAUDE.md vs AGENTS.md**：Claude Code 读 CLAUDE.md、**不直接读 AGENTS.md**；官方建议用 `@AGENTS.md` import 或 symlink 共用一份。`/init` 在 `CLAUDE_CODE_NEW_INIT=1` 下会读 AGENTS.md / .cursor/rules / copilot-instructions 等并吸收进 CLAUDE.md；`/import` 一次性迁入他家 agent 配置，**需 v2.1.213+**。
- **/doctor 修剪 CLAUDE.md**：官方确认存在——"cuts content Claude can derive from the codebase … keeps pitfalls, rationale, and conventions that differ from tool defaults"，**需 v2.1.206+**（与课程 ep1 口径一致）。
- **官方尺寸口径**：CLAUDE.md 每文件 **<200 行**，"longer files consume more context and reduce adherence"；`@import` 拆分只改善组织、不减上下文（import 的文件启动时全量加载）。auto memory 的 MEMORY.md 硬上限为**前 200 行或 25KB**（超出部分不加载）；CLAUDE.md 无硬上限，全量加载。
- **compact 后存活**：项目根 CLAUDE.md 在 /compact 后从磁盘重读重注入；嵌套 CLAUDE.md 和 path-scoped 规则**不自动重注入**（下次读到匹配文件时才回来）——ep3 若讲"防线写在哪"要用这条。
- **AGENTS.md 生态**：官网口径 "used by over 60k open-source projects"；由 OpenAI Codex、Amp、Jules、Cursor、Factory 共同发起，现归 **Linux Foundation 旗下 Agentic AI Foundation** 托管；OpenAI 主 repo 有 **88 个**嵌套 AGENTS.md；OpenAI 系 agent 会自动执行 AGENTS.md 里列的测试命令（官网 FAQ）。
- **官方在替用户生成文档**：Vercel `npx @next/codemod@canary agents-md` 会按项目 Next.js 版本下载匹配文档到 `.next-docs/` 并注入压缩索引——**框架厂商开始官方分发"版本匹配的 agent 文档"，等于官方承认"训练数据/旧文档会误导"是真实问题**（2026-08-20 抓取 Vercel 博客）。

---

## 对课程定位的启示

**观众早知道的（讲了会水，一句带过即可）**

- "CLAUDE.md/AGENTS.md 别写太长"——官方 200 行红线 + 社区大量文章，已是口水。
- "文档会过时、过时会误导 AI"——【常识】，Augment/DEV.to/官方文档都在讲。
- "定期修剪、以代码为准"——官方文档原文级别的基础建议。

**真盲区（讲了值钱，课程的差异化弹药）**

1. **机制叙事**："文档的危害不在旧，在于它让 agent 停止验证"（surface-bench：fresh doc 同样抑制验证）——这把 doc-audit 从"擦屁股"升格为"任何文档的必备配套"，比课程原叙事更强，且几乎没人讲。
2. **能力不免疫**："换更强的模型没用，opus/gpt 一样 100% 被带偏"——反"等等党"直觉，适合做集的钩子。
3. **警告无效、证据有效**：H6（C3≈C2 ≫ Cw）为"分诊表必须带证据（修正后代码/接口验证结果），不能只标'可能过时'"提供实证——**直接塑造 doc-audit skill 的产出物设计**。
4. **研究结论分裂 + 预算依赖**（Gloaguen vs Lulla vs probe-and-refine）：劝退堆料的精确版——"未经 eval 的文档默认是负债"，呼应官方 /doctor 修剪逻辑。
5. **与课程 spike 互证**：课程实测"代码能反驳时 agent 站代码"与 surface-bench comprehension 家族（代码可见→成功率天花板）方向一致；benchmark 的 hidden-dependency 布景正是课程"代码无法反驳时过期文档是唯一事实来源"的实验化——两边可以互相引用，课程的实证链比绝大多数公开内容都完整。
6. **稀缺性**：真实世界事故复盘贴很少，公开证据几乎全是 benchmark——课程的实拍 demo（哪怕需要替代布景）在内容市场上是稀缺供给。

**引用风险提示**：surface-bench 是单人 working paper（未同行评审）；Lulla/Vercel 各只测了效率或单一框架；Gloaguen 的 context file 是 LLM 单次生成的。三者都不支持"文档无用"的极端结论，课程口径应落在"未经审计的文档是高风险资产"。
