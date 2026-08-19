# 论断核查：结构化/图形化工件替代散文 plan 审查的实践与实证支撑

> 调研日期：2026-08-19。姊妹篇：[`plan-review-practices.md`](./plan-review-practices.md)。
> 待验证的课程设想：反范式 = 人对 agent 生成的散文 plan 盲批；解法 = 命令 agent 用结构化格式输出（mermaid 状态图/时序图、场景遍历、EARS 需求句型），人审图/审结构，利用"封闭网格让遗漏自动显形"。
> 每条论断标注【证实 / 部分证实 / 证伪 / 未找到一手出处】。

---

## 论断 1：Kiro 的 requirements.md 用 EARS 句型；design.md 默认生成 mermaid 图

**EARS：【证实】**（Kiro 官方文档原文）

- Kiro 官方文档 Feature Specs 页：
  > "The `requirements.md` file uses **EARS (Easy Approach to Requirements Syntax) notation** to provide structured, testable requirements. Each requirement follows this pattern: `WHEN [condition/event] THE SYSTEM SHALL [expected behavior]`"
  — <https://kiro.dev/docs/specs/feature-specs/>
- Kiro 官方博客《Introducing Kiro》：
  > "Each user story includes **EARS (Easy Approach to Requirements Syntax) notation acceptance criteria covering edge cases** developers typically handle when building from basic user stories."
  — <https://kiro.dev/blog/introducing-kiro/>

**Mermaid：【证实，但有一个措辞细节】**

- Kiro 官方博客配图文案（design 文档截图说明）：
  > "Kiro design specs with interfaces, **mermaid** and data flow diagrams"
  — <https://kiro.dev/blog/introducing-kiro/>
- AWS 官方 DEV Community 组织账号对 re:Invent 2025 官方 session DEV314（演讲者为 AWS 产品经理/布道师本人）的记述：
  > "The design includes high-level architecture **(with mermaid diagrams)**, component interfaces, data models, and implementation details."
  — <https://dev.to/aws/dev-track-spotlight-spec-driven-development-with-kiro-dev314-45e8>
- **措辞细节（诚实标注）**：kiro.dev 文档正文的表述是 "Build design docs with **sequence diagrams and architecture plans**"（<https://kiro.dev/docs/specs/>），并未逐字承诺"默认一定是 mermaid"。"默认生成 mermaid"这一具体断言的依据是官方博客配图文案 + AWS 官方渠道记述 + 用户实测，不是文档级契约。课程里照此口径表述即可。

---

## 论断 2：GitHub spec-kit 模板强制场景遍历（user scenarios）；是否强制图

**场景遍历：【证实】**（模板原文，直接取自仓库 raw 文件）

- `templates/spec-template.md` 首节即为强制场景遍历：
  > "## User Scenarios & Testing **\*(mandatory)\***"
  > 每个 User Story 必须含："**Independent Test**: [Describe how this can be tested independently…]" 和 "**Acceptance Scenarios**: 1. **Given** [initial state], **When** [action], **Then** [expected outcome]"
  > 且设独立的 Edge Cases 节："### Edge Cases — What happens when [boundary condition]? How does system handle [error scenario]?"
  — <https://raw.githubusercontent.com/github/spec-kit/main/templates/spec-template.md>

**强制图：【证伪】**

- `templates/plan-template.md` 全文无任何图/mermaid 要求；其强制内容是 Technical Context 字段、Constitution Check 门禁（"GATE: Must pass before Phase 0 research"）、目录结构决策和 Complexity Tracking 表。
  — <https://raw.githubusercontent.com/github/spec-kit/main/templates/plan-template.md>
- 结论：spec-kit 的"结构化"押在**场景遍历 + 验收语句 + 门禁检查单**上，不押在图上。这对课程是个有用的事实校准：**业界两大 SDD 工具里，只有 Kiro 一方把图作为默认工件。**

---

## 论断 3：AWS Prescriptive Guidance 对 EARS 的官方推荐

**【未找到一手出处】**

- 未检索到 docs.aws.amazon.com/prescriptive-guidance 下的 EARS 专页。AWS 对 EARS 的官方背书全部经由 **Kiro 产品文档/博客**与 **re:Invent 官方 session**（见论断 1），不是 Prescriptive Guidance 渠道。
- 补充事实校准：EARS 并非 AWS 发明，原始出处是 Mavin 等人的 IEEE 论文（2009 年 17th IEEE International Requirements Engineering Conference，"Easy Approach to Requirements Syntax"），在航空/工业界（Rolls-Royce、Intel）有早于 AI 浪潮的真实使用史——学术文献中可见引用链："EARS is a template-system created by Mavin et al. [MWHN09] and currently used by companies like Intel Corporation or Rolls-Royce"（[Comparative evaluation of template-systems for requirements documentation, Univ. of Cádiz 仓库](https://rodin.uca.es/bitstream/handle/10498/18769/BA_FCaballero.pdf?sequence=1&isAllowed=y)；另见 [arXiv:2509.14294 参考文献"Easy approach to requirements syntax (EARS). In 2009 17th IEEE"](https://arxiv.org/pdf/2509.14294)）。
- 对课程的含义：EARS 是"有 15 年工业史的需求句型被 AWS/Kiro 采纳"，不是"AWS 论文级推荐"。表述时别虚构 AWS PG 出处。

---

## 论断 4：有没有官方/团队 best practices 明确推荐"审图/结构化工件而不是散文"

**【未找到一手出处（作为"明确推荐"）；存在部分支持】**

找到的最近邻证据：

- **部分支持**：Kiro 把 mermaid 图、接口定义放进 design.md，而 design.md 位于强制审批门禁之后（"For well-understood features where you don't need approval gates between phases, select Quick Spec instead"——即默认流程有门禁，<https://kiro.dev/docs/specs/feature-specs/>）。即"图作为被审批工件存在"是事实，但 Kiro 官方**没有**说"审图代替审散文"。
- **反向提醒**：Thoughtworks Technology Radar（2025-11，Spec-driven development 列为 Assess）反而警告这些工具产生的长 spec 审不动：
  > "some generate **lengthy spec files that are hard to review**, and when they produce PRDs or user stories, it's sometimes unclear who their intended user is."
  — <https://www.thoughtworks.com/radar/techniques/spec-driven-development>
- Anthropic 官方 best practices 推荐审的是"**验收步骤/证据**"（见姊妹篇），不是图。
- 结论：**"人审图不审散文"目前没有任何官方或团队的一手推荐**。业界实际状态是"图/结构化工件被放进了审批门禁"（Kiro），至于人审门禁时该看图还是看文，没有公开 guidance。

---

## 论断 5：程序理解/review 中"图 vs 文字"效率的实证研究

**【部分证实：有经典实证支持大方向，但无针对本场景的直接实验】**

- **认知理论基础【证实】**：Larkin & Simon 1987《Why a Diagram is (Sometimes) Worth Ten Thousand Words》（Cognitive Science 11:65–99）。核心论点：图式表示把相关信息空间上并置、用知觉推断替代符号搜索，从而降低认知成本。注意标题里的 "**(Sometimes)**"——原作者自己限定了效果有条件（[书目记录与摘要，Carleton SERC](https://serc.carleton.edu/resources/961.html)；[CMU Koedinger 1992 论文引用页](https://pact.cs.cmu.edu/pubs/Koedinger%2092.pdf)）。这正好支持"封闭网格让遗漏显形"的机理版本：**当任务需要检查'完备性/邻接关系'时，图式表示把搜索变成知觉扫视**——但论文没有测"遗漏检出率"这个指标。
- **程序理解受控实验【证实（单项、年代早）】**：Scanlan 1989《Structured flowcharts outperform pseudocode: An experimental comparison》（IEEE Software 6(5):28–36，DOI 10.1109/52.35587）。用低/中/高复杂度算法做双组实验，结论：流程图组的理解时间显著少于伪代码组（文献记录见 [uni-saarland 代码理解实验 40 年系统综述的参考条目](https://www.se.cs.uni-saarland.de/publications/docs/wyrich202340.pdf) 与 [UFPE 软件工程实验教材对该实验方法的复述](https://www.cin.ufpe.br/~fmcf2/Doutorado/Software_Engineering_Experimentation.pdf)；结论转述见 [modeling-languages.com](https://modeling-languages.com/structured-flowcharts-outperform-pseudocode/)）。
- **缺口（诚实标注）**：未找到任何一手实验直接测"review AI 生成的设计图/spec vs 散文 plan 时的缺陷/遗漏检出率"。Scanlan 测的是"理解算法"，Larkin & Simon 是认知机理，两者到"审 plan 抓遗漏"之间隔着一层外推。

---

## 论断 6：反面证据——agent 生成的 mermaid "漂亮但错"；人审图是否也有盲批风险

**语法级失败：【证实】**（一手 issue/官方论坛）

- GitNexus 仓库 issue #1504（2026-05）：
  > "Mermaid diagrams use literal `\n` for line breaks (invalid syntax, breaks rendering) — The resulting diagrams **fail to render in any Mermaid renderer** (mermaid.js, GitHub, mermaid.live), showing a syntax error instead of the diagram."
  — <https://github.com/abhigyanpatwari/GitNexus/issues/1504>
- Cursor 官方论坛（2026-01）：用户在 plan 生成中遇到 "the Mermaid's diagram is failing"。
  — <https://forum.cursor.com/t/error-with-mermaid-when-creating-a-plan/147818>
- 即：**"agent 生成 mermaid 常见语法错误"有公开的一手记录**。课程若让观众抄"输出 mermaid"这一招，应附带一个兜底动作（如先渲染一遍/commit 前检查），否则反范式集会制造新的失败模式。

**语义级"渲染正常但内容错"：【未找到一手出处】**

- 未找到系统性一手研究或工程报告量化"LLM 生成的图渲染正常但语义错误"的发生率。只有泛化的 LLM 幻觉文献（如 [arXiv:2508.19882](https://arxiv.org/pdf/2508.19882) 综述提及 LLM 生成物存在幻觉与次优输出），不针对图。

**"图更可信 → 审图也会盲批"：【证据相互冲突，不可作为课程论据】**

- 正向：Tal & Wansink 2016《Blinded with science: Trivial graphs and formulas increase ad persuasiveness and belief in product efficacy》（Public Understanding of Science 25(1):117–25，[PubMed 25319823](https://pubmed.ncbi.nlm.nih.gov/25319823/)）：加入琐碎图表会提升对陈述的信任。
- 反向（复现失败）：Dragicevic & Jansen 2017 四次复现实验（N=623）：
  > "we were unable to replicate the original study's findings, as text with chart appeared to be **no more persuasive – and sometimes less persuasive – than text alone**… the chart's contribution to **understanding** was clearly larger than its contribution to persuasion."
  — <https://aviz.fr/blinded>（VIS 2017 论文配套页，含全部实验材料）
- 结论：原效应不稳，复现研究反而**支持**"图促进理解多于促进盲信"。即：没有可靠实证说"审图同样会盲批"，但也没有实证说"审图就不会盲批"——这个方向上诚实的课程表述是"图降低的是理解成本，不是审查责任"。

---

## 总评：这一集的解法是"有真实实践背书"还是"我们的想象"？

**判定：骨架有真实实践背书，具体动作是合理外推——课程需要把表述强度降半档。**

拆开看：

| 课程设想中的成分 | 证据状态 |
|---|---|
| 用 EARS 句型替代散文需求 | **有背书**：Kiro 官方默认（文档原文证实），EARS 本身有 15 年工业史（Rolls-Royce/Intel） |
| 用场景遍历（Given/When/Then + Edge Cases）替代散文 plan | **有背书**：spec-kit 模板原文强制（mandatory 字样） |
| 用 mermaid 状态图/时序图作为被审工件 | **有背书（单侧）**：Kiro 官方默认生成（博客+AWS 官方渠道证实）；spec-kit 不强制图 |
| "人审图不审散文"作为明确推荐做法 | **无背书**：未找到任何官方/团队一手推荐；Thoughtworks 反而警告长 spec 审不动 |
| "封闭网格让遗漏自动显形"（图 → 遗漏检出率提升） | **机理有支撑、指标无实证**：Larkin & Simon（认知机理）+ Scanlan（理解速度）支持方向，但没有"遗漏检出率"的直接实验 |
| agent 生成 mermaid 的失败模式 | **有记录的坑**：语法级失败有一手 issue；语义级错误无系统研究 |

**给课程的三条修正建议**：

1. 表述降级：不要说"业界证明审图优于审散文"，说"两大 SDD 工具链（Kiro、spec-kit）已经把结构化工件放进了强制审批门禁，我们把人审的重心押在结构和图上"——前者无出处，后者逐字可核。
2. 解法的"可执行"部分应押在**证据最强的两个格式**上：EARS 句型 + 场景遍历/Edge Cases 清单（两家工具都强制），mermaid 图作为第三个可选项并附渲染兜底（论断 6 的语法失败记录）。
3. "封闭网格显形"可以讲，但要归功于认知科学机理（Larkin & Simon 的"sometimes"限定照实讲），并承认这是外推——这恰好呼应姊妹篇的结论：图的价值是**降低验证复杂度**，而不是保证审查质量。
