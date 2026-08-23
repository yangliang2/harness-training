# repo 堆文档到底有没有影响：一手证据复核

调研日期：2026-08-23（所有 URL 除特别注明外均为当日抓取）。服务对象：反范式课 ep03 裁决前置调研——自有实测四轮 29/29 压不出"旧文档喂毒"（agent 总能自救），但案例组有 ~1.5–2× 轮次/成本税；候选新叙事"语料税/过度喂料"待验。本报告回答根本问题：**repo 里堆文档，到底是否有影响？影响什么（成本/正确率/轮次）、量级多大、什么条件下成立？**正反两面均查。

来源分级：一手 = arXiv 论文（逐条注明是否同行评审）/官方文档/官方工程博客；高质量 = 可复现开源 benchmark；工程博客二手；营销文只作线索。所有数字均出自实际抓取的一手来源，未能逐字核验处逐条标注。引用研究均注明实验条件与迁移边界（模型代际、是否 coding agent 场景、上下文注入方式是常驻还是检索）。

**先亮总判定**（详见文末"对 ep03 裁决的启示"）：

- **成本/轮次维度：有影响，证据强**。常驻 context file 平均 +20–23% 美元成本且不提成功率（ETH，4 agent、438 任务）；agent token 大头本来就是"读"（76.1%）。
- **正确率维度：条件化**。常驻小尺度（数百至两千词的 context file）对成功率**无显著影响**（三篇独立研究一致）；检索尺度大语料有**强负效应**（信息量不变、篇数增加即 −20%；语料 54→1,128 篇准确率 75%→<40%——RAG/QA 场景，非 coding）；**准确性**才是 0%↔100% 级别的变量（surface-bench）。
- **"文档数量/体积 → coding agent 成功率"的受控剂量研究：【未找到】**。这个空白是真实的，自有 spike 恰好落在这个空白里。

---

## 问题 1：无关/干扰上下文降级的一手研究复核——判定【机制真实、跨代际重复确认，但对"repo 静置文档"命题全部隔着两层：无 coding 任务、无自主检索形态】

### 1.1 Shi et al. 2023（arXiv 2302.00093，ICML 2023，同行评审）

- **结论与量级**（据 arXiv 摘要页 + ar5iv 全文，2026-08-23 抓取）：构造 GSM-IC（小学算术 + 无关句），"model performance is dramatically decreased when irrelevant information is included"。CoT prompting 在 GSM8K 基线 95.0% → GSM-IC micro-accuracy 72.4%（**降 22.6pp**）；macro-accuracy（所有干扰变体全对才算对）即使最稳健的 least-to-most prompting 也只有 **18%**。缓解：self-consistency 20 采样投票拉回 11+pp；加一句 "Feel free to ignore irrelevant information" 即持续改善。
- **实验条件**：每题只在题干**插入一条无关句**，58,052 例源自 100 道基题。**单条 prompt 内常驻注入，不涉及检索**。
- **模型代际**：code-davinci-002（Codex）与 text-davinci-003（GPT-3.5 代际）——**2023 年初模型，已退役两代以上**。
- **迁移距离：远**。三重不同：(a) 模型代际过旧；(b) 干扰由实验者强制插入、模型无法回避，而 repo 文档是 agent **自主决定是否检索**的——未读入上下文的文档对模型不可见；(c) 算术任务非代码。**该文只能证明"干扰一旦进入上下文有代价"，不能证明"文档放在 repo 里就有代价"。课程若引用必须带此限定，不能拿它当"repo 堆文档降正确率"的证据。**
- URL：https://arxiv.org/abs/2302.00093（2026-08-23 抓取）。【判定：迁移强度弱】

### 1.2 后续 distractor / 长上下文变体（2024–2026）

- **NoLiMa**（arXiv 2502.05167，ICML 2025，同行评审，Adobe Research；2026-08-23 抓取）：13 个宣称 ≥128K 的模型，needle 与问题无字面重合（仅语义关联）时，**32K 处 11 个模型跌破其短上下文基线的 50%**；GPT-4o 99.3% → 69.7%（32K）。GitHub 更新加入 GPT-4.1（有效长度仅 16K）、o3、Gemini 2.5 Pro 等，退化模式依旧。https://arxiv.org/abs/2502.05167 、https://github.com/adobe-research/NoLiMa 。【迁移强度中：QA 合成任务，非 coding】
- **RULER**（arXiv 2404.06654，COLM 2024，同行评审）：17 个长上下文模型；**干扰 needle 数量增加单调压低性能**；"claimed context size ≫ effective context size"。注意：此条数字系搜索摘要转述，未逐项核出原文表格精确百分比。https://arxiv.org/abs/2404.06654 。【中】
- **GSM-DC**（arXiv 2505.18761，2025-05，是否会议接收未标注）：受控 distractor 注入，确认 2025 年模型仍 "significant vulnerability to extraneous context"；用强 distractor 训练可显著提升鲁棒性。【中弱：摘要未列具体模型与数字】
- **Claude 4+/GPT-5 代际的严格复测：基本【未找到】**。唯一撞到的是 arXiv 2608.03297（2026-08-04，单作者 preprint，未同行评审）：主张此前多个长上下文基准把"朴素截断删掉了答案本身"误记为"长度效应"（25% 保留率下答案幸存率 <1%）；用 distractor-aware 截断后，在 Claude Haiku 4.5 / Sonnet 4.6 / Opus 4.7 / GPT-5.5 上 "performance is preserved or improves"——**即有一手研究认为部分"长上下文降级"是测量伪影，但同时隐含承认 distractor（而非长度本身）才是要害**。https://arxiv.org/abs/2608.03297 。

### 1.3 RAG 噪声研究

- **Cuconasu et al. "The Power of Noise"**（arXiv 2401.14887，SIGIR 2024，同行评审；2026-08-23 抓取摘要 + v4 全文）：
  - **伤害侧**：语义相关但不含答案的 distracting 文档——加**一篇**准确率即掉约 25%；18 篇 distracting + 金文档放远位时 Llama-2 从 0.5642 → 0.2348（相对降约 58%）。
  - **反直觉侧（反方证据，必须一起讲）**：摘要原话 "adding random documents in the prompt improves the LLM accuracy by up to 35%"——**随机无关文档反而最多提升 35%**。真正伤人的是"相似但不命中"，不是"无关"。
  - **条件与代际**：Llama-2 7B、Falcon 7B、Phi-2 2.7B、MPT 7B——**全是 2023–2024 年 7B 级小模型，无前沿模型**；NQ + Wikipedia QA。量级不可直接外推到前沿 coding agent。
  - **对本命题的意义**：repo 里的过时/相邻经验文档恰属"语义高度相关但可能误导"这一最伤类别——机制吻合，量级存疑。https://arxiv.org/abs/2401.14887 。【中】
- **Wu et al.**（arXiv 2404.03302，COLM 2024，同行评审）：构造"无关→部分相关→高度相关"谱系，构造出的无关信息 "scores highly on similarity metrics, being highly retrieved by existing systems"，LLM "can be easily distracted"，现有缓解手段效果有限。摘要未给具体百分比。https://arxiv.org/abs/2404.03302 。【中】

### 1.4 Chroma "Context Rot"（厂商技术报告，非同行评审）

- 官方页 https://www.trychroma.com/research/context-rot（发布 2025-07-14；2026-08-23 抓取）。**18 个模型，含 Claude Opus 4 / Sonnet 4、o3 / GPT-4.1 全系、Gemini 2.5 Pro/Flash**（不含 GPT-5，其时未发布）。
- 核心发现：任务复杂度固定、只变输入长度，"performance varies significantly as input length changes, even on simple tasks"，降级**非均匀**；**单个 distractor 即可测出下降，4 个叠加更糟**，不同 distractor 杀伤力不均；needle-question 相似度越低，长上下文下降越快。
- 边界：**明确无 coding 任务**（NIAH 变体、LongMemEval、Repeated Words）；逐模型精确百分比主要在图中，本次抓取未提出数字，如实标注；**利益相关**——Chroma 是向量库厂商，结论方向与其"做检索别塞长上下文"的商业主张一致；方法学公开（GPT-4.1 judge 与人工对齐 >99%）、有复现代码。【中：唯一系统覆盖 Claude 4 代际的 distractor 操纵测试】

### 1.5 小结

"上下文内的**语义相关干扰**使 LLM 降级"在 2023–2025 跨代际、跨任务被反复确认（含同行评审与 Claude 4 代际厂商报告），且一致指向**语义相关干扰 > 随机噪声、distractor 效应 > 纯长度效应**。但对"repo 堆文档影响 coding agent"：**没有一项测过 coding 任务，也没有一项测过"文档静置文件系统、agent 自主检索"的形态**——静置文档只有被读入上下文才可能构成干扰。直接证据缺位，迁移全靠机制类推。另需诚实交代：新代模型降级点大幅后移、模型间差异极大、部分历史测量含截断伪影——"2023 年的结论"不能原样搬到 2026 年模型上。

---

## 问题 2：repo 文档量作为自变量的直接研究——判定【受控剂量研究未找到；最接近的观察性长度分析显示"长度与成功率/成本无清晰关系"】

**总判定：没有任何研究做过"同一 repo 放 1 篇 vs 5 篇 vs 20 篇文档、总信息量受控"的实验。这个空白是真实的。**最接近的直接证据两条：ETH 的长度-效果观察性分析（无清晰关系）+ Vercel 的一次体积操纵（40KB→8KB 通过率不变）。

### 2.1 Gloaguen et al.（arXiv 2602.11988，ETH Zurich，v1 2026-02-12 / v2 2026-06-23，预印本、OpenReview 在审）——最核心

据 arXiv 摘要页与 HTML 全文（2026-08-23 抓取）：

- **设计**：SWE-bench Lite（300 任务、11 repo，LLM 生成 context file）+ 自建 CTXbench（138 实例、12 个含**开发者手写** context file 的 repo，源自 5,694 个 PR）。4 组 agent：Claude Code + Sonnet 4.5、Codex + GPT-5.2、Codex + GPT-5.1 mini、Qwen Code + Qwen3-30b-coder。context file 平均 641 词，范围 24–2003 词。
- **成功率**：LLM 生成的平均**降** 0.5%（p=87%）/ 2%（p=37%）不显著；开发者手写的 +2.4%（p=21%）也不显著，但显著优于 LLM 生成（p=3.8%）。**两类都不能显著提升成功率**。
- **成本口径（已澄清）**：**+20% 与 +23% 是美元成本（token 量 × API 定价折算）**，来自步数增加——原文 "they increase the # steps in every setting, on average by 2.45 and 3.92 [steps], leading to a significant (p < 0.001%) cost increase of 20% and 23% on average"。
- **长度分析（Figure 13）**：按长度分 bin，"no clear dependency between the success rate or the per-instance cost and the context file length"——**成本增量来自"有无"而非"长短"**。注意这是观察性分析（bin 现有文件），非受控操纵。
- **内容消融（Table 7）**：分别移除 Overview（p=0.15）/ Tooling（p=0.31）/ Testing（p=0.80），**无任何类别对准确率有显著效应**；repository overview 不减少 agent 触达关键文件的步数，反而增加总步数。正面发现：agent 会很好地遵循 context file 里的指令。
- URL：https://arxiv.org/abs/2602.11988 、https://arxiv.org/html/2602.11988v2 。

### 2.2 同族直接研究（调研中新发现）

- **Khatri, arXiv 2607.27250**（2026-07-28，单作者预印本；2026-08-23 抓取）"Do Context Files Help Coding Agents? A Two-Agent Ablation Study on Real Repositories"：Claude Code（claude-sonnet-4-6）+ Codex CLI（gpt-5.5），3 个 Python repo，291 次运行。三条件：无 AGENTS.md / **全文注入** / **按主题拆分 wiki 按需检索**。通过率 Claude 53.3%/55.6%/55.6%（p=1.00）、Codex 58.8%/56.9%/52.9%（p=0.66）——**正确率零差异**（等效检验界定 ≤10pp/≤15pp）；selective 显著降低 Claude cache-creation token（p=0.012）。**"常驻 vs 按需"是组织方式变量，接近但不等于数量变量**。https://arxiv.org/html/2607.27250v1 。
- **surface-bench**（GitHub working paper，非正式发表，有预注册与 Holm–Bonferroni 校正；2026-08-23 抓取 PAPER.md）：**只操纵文档准确性/新旧，明确不操纵数量/体积**（"the only thing that varies across conditions is the documentation block"）。过时文档 C1 成功率 0–32%（单轮 cascade 任务三个 Claude 模型全部 0%、被误导率 100%），准确文档 C2 94–100%；**准确文档还使总成本降 36–46%**（5 模型中 4 个）。意义：**质量是 0%↔100% 级别的变量**，数量研究它没做。https://github.com/Connorrmcd6/surface-bench/blob/main/PAPER.md 。

### 2.3 repo 规模与成功率（间接）——【受控研究未找到】

- SWE-bench-Live（arXiv 2505.23419）：新 repo 文件更少、代码更小，最佳组合 resolve 反而更低（22.96% vs 18.89%）——**repo 更小 ≠ 成功率更高**（混杂训练数据污染因素）。此数字来自搜索结果引文，**未逐字核对原 PDF**。
- 各 repo 间 resolve rate 差异极大（<10% 到 >50%），归因混合，无单变量分离。

### 2.4 检索语料规模研究——"语料规模税"的最强学术证据

- **Levy et al. "More Documents, Same Length"**（arXiv 2503.04388，EMNLP 2025 Findings，同行评审；2026-08-23 抓取摘要页）：**保持 context 总长度与相关信息位置不变，只增加文档篇数**——性能下降**最多 20%**（Qwen2.5 例外基本持平）。这是"篇数本身（信息量不变）是独立于长度的税"的最干净证据。https://arxiv.org/abs/2503.04388 。
- **Subedi et al. "When More Documents Hurt RAG"**（arXiv 2606.11350，预印本；2026-08-23 抓取摘要页）：真实部署语料（Wyoming DOT），**54 篇扩到 1,128 篇（88,907 chunks），准确率从 75% 跌到 40% 以下**；域内 scoping 把 P@10 从 0.77 提到 0.86（p<0.05）。**语料库变大 → 检索稀释 → 答对率暴跌的直接量化**。https://arxiv.org/abs/2606.11350 。
- 相关：arXiv 2510.05381 "Context Length Alone Hurts LLM Performance Despite Perfect Retrieval"——检索完美时仅 context 变长也降性能（**未抓正文核数字**）。
- **迁移边界：全是 RAG/QA 设定，不是 coding agent 在 repo 里跑，属类比证据。**

### 2.5 文档长度/压缩的工程 eval

- **Vercel 官方博客**（2026-01-27；2026-08-23 抓取）：Next.js 16（训练数据未见）API 实现任务（`'use cache'`、`connection()`、`forbidden()` 等 7 类）。基线（无文档）53%，Skills 默认 53%（+0pp，原文 "In 56% of eval cases, the skill was never invoked"），Skills+显式指令 79%，**AGENTS.md 内嵌 8KB 文档索引 100%**（较基线 +47pp；Build/Lint/Test 三项均 100%）。**体积口径纠偏（两组核验后取谨慎读法）**：博客说初始 docs 注入约 40KB、压缩 80% 到 8KB pipe-delimited 索引，agent 按需读 `.next-docs/` 下具体文件——**但 40KB 全量嵌入条件没有单独通过率数据，"全量 vs 索引"对照并不存在**，故只能说"8KB 策划索引拿到 100%"，不能说"压缩 80% 无损"。实验条件缺口：任务总数、agent harness/模型均未披露；作者自述 skill 显式指令条件"措辞微调会产生大的行为波动"。厂商自评博客，不可复现。https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals 。收益变量是**覆盖度 + 策划 + 按需加载**的合取，不是体积。
- 专门的 "CLAUDE.md 长度剂量实验" 社区 eval：【未找到】。ETH Figure 13 是目前最好的长度-效果数据。

---

## 问题 3：探索/检索税的量化证据 + few-shot 措辞复核——判定【探索税有硬数字（学术侧）；"案例多了一定差"禁说，"相似但不命中"方向有支撑但"一定"需降格】

### 3.1 探索税量化

- **SWE-Pruner**（arXiv 2601.16746，预印本；2026-08-23 抓取）：Mini SWE Agent + Claude Sonnet 4.5 在 SWE-Bench Verified（500 实例）的 trajectory 分解——**read 类操作占总 token 的 76.1%**，execute 12.1%、edit 11.8%（Figure 2）；跨模型验证 GLM-4.6 read 占 67.5%（2.89M tokens）。基线每实例约 **51.0 轮 / 0.911M tokens**（Sonnet 4.5）。**目前"探索/阅读是 agent 成本大头"的最直接量化。**https://arxiv.org/html/2601.16746v1 。
- **FastContext**（Microsoft，arXiv 2606.14066）：专职 repo explorer 把 token 消耗降最多 60%、resolution rate 升最多 5.5%——反向印证探索是主要成本。**数字来自搜索摘要，未逐字核对原文。**
- **Gloaguen +20%/+23% 美元成本**（见 2.1）——"喂了 context file 反而更贵"的直接量化。
- **厂商官方一手量化：【未找到】**。Anthropic《Effective context engineering for AI agents》只有定性口径（context rot、just-in-time vs 前置注入的权衡 "runtime exploration is slower than retrieving pre-computed data"），无 "most tokens spent on exploration" 类数字。硬数字只能引学术论文。
- **常驻成本的官方口径**（Claude Code memory 文档原文，2026-08-23 抓取）："CLAUDE.md files are loaded into the context window at the start of every session, consuming tokens alongside your conversation"——**每 session 启动全量注入**；@import 的文件同样启动时全量加载（拆文件不省 token）；子目录 CLAUDE.md 例外按需加载。https://code.claude.com/docs/en/memory 。

### 3.2 few-shot 饱和与 many-shot 一手复核

- **Brown et al. 2020（GPT-3，arXiv 2005.14165）**：原文 K 通常取 10–100（受 2048 context 限制）；Figure 1.2/3.8 性能随 K **上升**——"The few-shot SuperGLUE score steadily improves with both model size and with number of examples in the context"。**GPT-3 论文本身没有"例子多了变差"的证据，只有"受 context 限制测不上去"。**https://ar5iv.labs.arxiv.org/html/2005.14165 （2026-08-23 抓取）。
- **Min et al. 2022（arXiv 2202.12837）**：(a) 随机标签替换正确标签仅降 0–5% 绝对值（分类平均 −2.6%）；(b) k ∈ {4,8,16,32} 消融，原文 "model performance does not increase much as k≥8"——**k≥8 后收益平缓**。**代际警告：全是 2022 年前非 instruction-tuned 模型（GPT-2 Large 到 GPT-3 175B），限于分类/多选任务。**https://arxiv.org/abs/2202.12837 。
- **Agarwal et al. 2024 "Many-Shot In-Context Learning"**（arXiv 2404.11018，NeurIPS 2024，同行评审；Gemini 1.5 Pro 1M context）：低资源翻译到 **997 shots 仍在升**、XLSum 到 500；但 **MATH ~125 shots 达峰后下降、XSum >50 例后下降、GPQA 250 shots 退化**、planning ~10 shots 即饱和。many-shot 收益强依赖长上下文模型与任务类型。https://arxiv.org/abs/2404.11018 。
- **"相似但不命中/错误示例"的直接伤害**：
  - Cuconasu（见 1.3）：retriever 打分最高但不含答案的文档显著伤害，随机文档反而 +35%——**与课程"相似但不命中"措辞完全同构（RAG 文档层面）**。
  - Wynn et al.（arXiv 2510.02480，预印本）："incorrect demonstrations harm performance to worse than zero-shot"（8 分类数据集，Llama-3-8B/Llama-2-7B——小模型）。
  - **Wei et al. 2023（arXiv 2303.03846）代际反转证据**：100% flipped labels 下 text-davinci-002 从 90.3% 跌到 22.5%（远低于随机线），PaLM-540B 跌到 ~31%；小模型基本不跌破随机——**模型越强越会"认真学"错误示例，错配伤害随能力增大**。这解释了 Min et al.（旧小模型）测不出标签伤害的原因。https://arxiv.org/abs/2303.03846 。

### 3.3 口播措辞判定

- ①**"案例多了一定差"——文献不支持，禁说**。GPT-3（K↑性能↑）与 Agarwal（数百到近千例仍在涨）都是反例。可守住的弱化版："案例数的收益会饱和（Min：k≥8 后平缓），部分任务过多反而下降（Agarwal：MATH ~125 峰值后降、XSum >50 降）"。
- ②**"相似但不命中的案例多了一定差"——方向有文献支持，但"一定"需降格为"会/往往"**。支持链：Cuconasu（相似不命中伤害最大、随机反而 +35%）+ Wynn（错误示例 < zero-shot）+ Wei（错配伤害随能力增长）。必须保留的限定：(a) Cuconasu 证据在 RAG 文档层面而非示例层面且是 7B 小模型；(b) 伤害强度随模型能力变化方向对课程有利（越强越伤）但出处是 2023 年模型。**建议措辞："相似但不命中的案例是伤害最大的一类上下文——比无关噪声更伤"，最硬来源 Cuconasu SIGIR 2024 + Wei arXiv 2303.03846。**

---

## 问题 4：业界实践与厂商口径——判定【"控制语料面"有明确官方口径，Anthropic/Cursor/OpenAI 三家一手原文俱在；"archive/ 不入检索"惯例未找到】

### 4.1 Claude Code 官方（https://code.claude.com/docs/en/memory 、settings-reference、large-codebases；2026-08-23 抓取）

- **claudeMdExcludes**：官方动机原文——"In large monorepos, ancestor CLAUDE.md files may contain instructions that aren't relevant to your work"；large-codebases 页："Use this for directories you never work in, such as other teams' packages, **legacy code**, or vendored subtrees"（官方示例 glob `"**/packages/legacy-*/**"`——最接近"归档不入语料"的官方表述）。
- **200 行红线原文**："Size: target under 200 lines per CLAUDE.md file. Longer files consume more context and reduce adherence."；单文件 >4 MiB 直接跳过。
- **搜索默认尊重 .gitignore**："Claude's content searches respect `.gitignore` by default"；另有 `permissions.deny` 的 Read 规则，"best-effort attempt to leave denied paths out of the results of the built-in Grep and Glob tools"。
- auto memory：MEMORY.md 每 session 注入前 200 行或 25KB，官方提醒 "merge or drop stale entries"。
- 【判定：强支持——官方提供整套缩小语料面机制，且明文动机是"无关内容进 context、长文件降遵循度"】

### 4.2 各家 ignore 机制

- **Cursor**（https://cursor.com/docs/context/ignore-files ，官方文档）：`.cursorignore` 动机**安全+降噪双重**——"In large codebases or monorepos, exclude irrelevant portions for faster indexing and **more accurate file discovery**"；`.cursorindexingignore` 纯降噪（"large generated files or vendored dependencies that shouldn't appear in search results"）；默认吃 .gitignore。
- **GitHub Copilot content exclusion**：官方定位偏隐私/IP；限制——"Copilot CLI and Agent mode … do not support content exclusion"。
- **标准化**：`.agentignore`/`.aiignore` 统一标准仅停留在社区提案（agentclientprotocol Discussion #49、gemini-cli Issue #4688）；JetBrains 用 `.aiignore`，Google 官方用 `.aiexclude`。**跨厂商标准未统一。**

### 4.3 Anthropic 官方工程博客（最强官方证据）

- 《Effective context engineering for AI agents》（2025-09；2026-08-23 抓取）原文："Context, therefore, must be treated as a **finite resource with diminishing marginal returns**." / "Good context engineering means finding the **smallest possible set of high-signal tokens** that maximize the likelihood of some desired outcome." / 指出 CLAUDE.md 被 "naively dropped into context up front"，推荐 just-in-time：维护轻量标识符（文件路径等）运行时按需加载。https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents 。
- 《Writing effective tools for agents》："return only high signal information" / "Optimizing the quality of context is important. But so is optimizing the **quantity**"。
- Claude Code best practices（已并入 https://code.claude.com/docs/en/best-practices ）："Keep it concise. For each line, ask: 'Would removing this cause Claude to make mistakes?' If not, cut it. **Bloated CLAUDE.md files cause Claude to ignore your actual instructions!**" Exclude 表明列："Anything Claude can figure out by reading code"、"Detailed API documentation (link to docs instead)"、"File-by-file descriptions of the codebase"。

### 4.4 Cursor / OpenAI / Google 的"多≠好"表述

- Cursor rules 官方文档："Keep rules under 500 lines"、"Reference files instead of copying their contents"。https://cursor.com/docs/context/rules 。
- OpenAI Codex best practices 官方："**A short, accurate AGENTS.md is more useful than a long file full of vague rules.**" 只在发现重复错误后再加规则。
- Google Jules：官方仅确认读 AGENTS.md，**"保持简短"类表述未找到**——如实标注。

### 4.5 RAG 侧语料治理

- **去重**：Microsoft 官方（Azure Databricks RAG pipeline 文档）原文要点："Duplicate documents in your corpus can result in highly redundant chunks… that can **decrease the performance** of your application"，dedup 是正式 pipeline 步骤。https://learn.microsoft.com/en-us/azure/databricks/agents/tutorials/ai-cookbook/quality-data-pipeline-rag 。
- **时效性**：Azure AI Search / AWS Bedrock KB / Pinecone 三家官方均有"保持新鲜、清理 stale"的机制性文档（Pinecone 官方 blog "prune stale records"），但明说"过期内容伤质量"这句的主要是 Microsoft 系文档。

### 4.6 docs 归档惯例——【未找到】

"archive/ 目录默认排除出 AI 检索"作为正式惯例无任何官方表述；MkDocs/Docusaurus exclude 机制用于 AI 场景的官方案例也未找到。邻近证据：Claude Code 官方以 legacy 包为 excludes 示例（4.1）；llms.txt 规范（llmstxt.org，社区规范、多厂商采用）本质是"人工精选极小文档集"，理念支持策展而非全量投喂。

---

## 问题 5：反方证据（堆文档有益的条件）——判定【"策划过的准确文档"有强收益证据；检索中介下"语料扩大带来新覆盖"也有益（Ning）；收益变量是覆盖度与准确性，不是数量；"命中率 ÷ 语料规模"的除号形式无文献出处，但"命中率随语料规模下降"有 IR 一手证据】

### 5.1 文档显著有益的量化证据

- **Vercel eval**（见 2.5，厂商博客，条件缺口已标）：覆盖训练盲区的 8KB 策划索引 = 53%→100%（+47pp）。**收益条件五合一：策划+压缩+索引化+按需加载+版本匹配**，缺一不可外推。
- **Lulla et al.**（arXiv 2601.20404，2026-01-28 提交，预印本，方法论完整；2026-08-23 抓取 abs+html 全文）：配对对照（同 repo 同任务有/无 AGENTS.md，任务为复现 ≤100 LoC、≤5 文件的真实 PR），中位 wall-clock 98.57s → 70.34s（**−28.64%**）、中位输出 token 2,925 → 2,440（**−16.58%**），输入 token 基本不变（−3.41%），Wilcoxon p<0.05。**关键限制：仅 Codex（gpt-5.2-codex）一个 agent；10 repo/124 PR；AGENTS.md 是三重筛选后的策划文件**（132 repo → 89 单文件候选 → 内容必须含惯例/架构/项目描述三类，LLM 分类 + 人工核验 → 26 个，实验用 10 个）；**未评估正确率**——原文 "These metrics do not capture whether agent-produced changes are correct, maintainable, or aligned with developer intent"；均值降幅大于中位数，节省主要来自少数高成本 run。https://arxiv.org/abs/2601.20404 。
- **surface-bench C2**（见 2.2）：准确文档恢复 94–100% 成功率**且总成本降 36–46%**——准确文档同时买到正确率和省钱。
- **DocPrompting**（Zhou et al., arXiv 2207.05987，ICLR 2023，同行评审）：检索文档再生成代码，CoNaLa 上 CodeT5 pass@1 +2.85pp（相对 +52%）；tldr 上 exact match 最高 +6.9pp。条件：2022 代小模型、函数级 NL→code、**检索的是与意图匹配的文档片段（策划/检索型，非堆）**。https://arxiv.org/abs/2207.05987 。
- **CodeRAG-Bench**（arXiv 2406.14497）：两面结论——"retrieving high-quality contexts" 有 notable gains，但检索器 "struggle to fetch useful contexts especially with limited lexical overlap"，且部分模型 "fail to improve with limited context lengths or abilities to integrate additional contexts"。**收益以"高质量上下文"为前提。**https://arxiv.org/abs/2406.14497 。
- Context7 效果数据：**【未找到可靠 eval】**——"70–90% 减少过时建议"来自评测聚合站，无方法论，营销性质不可引。docs.rs MCP eval、llms.txt 有效性 eval：**【未找到】**严肃量化研究。

### 5.2 RAG 侧"更多语料有益"的真反方证据

- **Ning et al. "Less LLM, More Documents: Searching for Improved RAG"**（arXiv 2510.02657，CMU Callan 组，预印本；2026-08-23 抓取摘要页）：开放域 QA 中 **"corpus scaling consistently strengthens RAG and can in many cases match the gains of moving to a larger model tier"**，但 "diminishing returns at larger scales"；机制是 **"increased coverage of answer-bearing passages"**（利用效率基本不变）；中型模型受益最大。**注意两个边界：这是"经检索器中介的语料扩大"而非常驻堆放；且收益来自新增的 answer-bearing 覆盖**——与 Subedi（扩语料掺入域外内容 → 稀释暴跌）并读，合成结论是：**语料扩大带来新答案覆盖时有益，带来的是相似不命中内容时有害**。https://arxiv.org/abs/2510.02657 。
- **Cuconasu 随机噪声 +35%**（见 1.3；确切条件：MPT-7B near positioning +36%，Llama2-7B 最高 +35%）：连完全随机的无关文档都可能有益——**"堆料一定有害"在文献里同样立不住**。但此反例条件极窄：7B 小模型、事实 QA、完全无关随机文档、gold 文档在场——与 repo 堆文档场景（相关但可能过时/错版本、编码任务、大模型）**几乎点对点相反，不能引为堆文档辩护**；同一论文里 related（相近但不含答案）文档加 1 篇即 −25%、多篇远位最多 −67%。
- 知识库覆盖率与质量正相关的其他研究：搜索摘要有 "performance increases monotonically with KB coverage" 类说法但**出处未能核实，按纪律不采用**。

### 5.3 与 Gloaguen 的表面冲突及调和

Lulla（效率 −28.6%）vs Gloaguen（成本 +20%）方向相反，但**并不直接矛盾**：agent 不同（Codex vs 4 组）、指标不同（runtime/输出 token vs 美元总成本，且 Lulla 输入 token 不变）、文件来源不同（三重筛选的开发者手写 vs 含 LLM 生成）、任务不同（PR 复现 vs SWE-bench 修复）。加上 ep03 前次调研已录的 Probe-and-Refine（arXiv 2606.20512：同一指导在步数预算充裕时有益、紧张时有害），文献的合成结论是：**"文档有没有用"取决于文档质量×任务×预算，不是文件本身**。

### 5.4 "知识库价值 = 命中率 ÷ 语料规模"能否被文献支撑

- **除号形式【未找到】文献直接支撑**；文献支持的是"减法/成本项"形式。
- **但"命中率本身随语料规模下降"有 IR 一手证据**：Reimers & Gurevych "The Curse of Dense Low-Dimensional Information Retrieval for Large Index Sizes"（arXiv 2012.14210，ACL 2021 short，同行评审）——dense 检索假阳性概率随 index size 上升；dense 随语料增大退化快于 sparse，存在 sparse 反超的 tipping point；"1M 文档上 SOTA 的系统在更大索引上可能表现很差"。（注：**"index bloat" 是 SEO 用语，不是 IR 文献术语，课程不要引这个词**。）https://arxiv.org/abs/2012.14210 。
- **各项对应的文献锚点**：命中率递减 = Reimers & Gurevych + Subedi；错配成本 = Cuconasu（related −25%~−67%）+ Chroma（"even a single distractor lowers accuracy"，工程结论 "curate and compress"）；常驻成本 = Gloaguen +20%；收益项 = Ning（answer-bearing coverage，饱和曲线）。
- **修正建议**：课程若用，标注"自家表述"；更接近文献的形式是——**价值 ≈ 命中收益（饱和曲线）−（相似不命中密度 × 错配成本）− 常驻/重读 token 成本，且命中率是语料规模的递减函数**。除法形式是修辞不是测量。

### 5.5 文档有益成立的条件清单（文献合成）

1. **覆盖模型训练盲区**（Vercel：训练外新 API +47pp；DocPrompting 动机相同；模型已会的写了也白写——Claude Code best practices 官方 Exclude 表）；
2. **准确且 answer-bearing / 版本匹配**（surface-bench 0%↔100%；Ning 的覆盖机制；反面 Cuconasu related −25%~−67%）；
3. **策划/压缩过，非原样堆放**（Vercel 8KB 索引；Lulla 样本是三重筛选的策划文件）；
4. **索引化 + 按需加载，常驻的只是地图**（Vercel 索引指向 `.next-docs/` 按需读；Chroma "curate and compress"；Khatri：selective 省 cache token 不降正确率）；
5. **具体指令而非综述**（Gloaguen：指令被遵守，overview 无效应）；
6. **预算充裕的任务**（Probe-and-Refine）；
7. **收益衡量必须同时含成本与正确率**（Lulla 只测效率结论为正；Gloaguen 加测成功率后结论翻转）。

**"堆文档"（未策划、大量、常驻、给人看的经验文档原样入库）同时踩中三个失效模式：(a) 常驻 token 成本每 run 都付（Gloaguen +20%）；(b) 语料变大命中精度下降（Reimers & Gurevych）；(c) 堆进来的"相关但不精确"内容正是危害最大的 distractor 类型（Cuconasu、Chroma）。反方证据恰好划出了它自己的边界。**

---

## 对 ep03 裁决的启示

### ① "repo 堆文档有影响"的最终判定

**影响维度与量级（按证据强度排序）：**

| 维度 | 判定 | 量级 | 证据强度 |
|---|---|---|---|
| **成本/轮次** | 有影响，方向确定 | 常驻 context file：+20–23% 美元成本、+2.45–3.92 步（Gloaguen，4 agent，预印本）；背景事实：read 类占 agent 总 token 76.1%（SWE-Pruner）；官方口径 CLAUDE.md 每 session 全量注入 | **强**（多来源一致；但"文档越多税越重"的剂量关系未被测过——ETH Figure 13 长度与成本无清晰关系） |
| **正确率（常驻小尺度）** | 无显著影响 | Gloaguen 不显著（−0.5%~+2.4%）；Khatri 三条件零差异（p=1.00/0.66） | **强（否定性结论）**——数百至两千词级别的文档不伤也不帮正确率 |
| **正确率（检索尺度大语料）** | 有强负效应，但证据是 RAG/QA 场景 | 篇数本身 −20%（Levy，EMNLP 2025 Findings）；54→1,128 篇准确率 75%→<40%（Subedi，预印本，真实部署） | **中**（同行评审 + 真实部署各一，但**均非 coding agent**，迁移靠类比） |
| **正确率（文档不准确时）** | 0%↔100% 级别 | surface-bench；但自有实测边界条件：测量可得时 agent 自救 | **中**（可复现开源 benchmark，未同行评审；与自有 29/29 的分歧由"测量可得性"调和） |

**成立条件**：影响只在文档**进入上下文**后发生（静置文档无影响——所有干扰研究的共同前提）；正确率伤害集中在**语义相关但不命中**的内容（Cuconasu）与**测量/验证不可得**的场景（自有实测 + surface-bench 的 hidden dependency 布景）；成本税则**无条件**发生（读了就付，常驻注入更是每 session 付）。

**一句话答案：repo 堆文档确定有影响，影响的主要是成本/轮次（强证据），正确率影响是条件化的——小尺度常驻无显著影响，大语料检索稀释与相似不命中干扰有伤害（但量化证据来自 RAG/QA 而非 coding agent）。**

### ② "语料税"叙事的证据地基评级

**可上卡（有一手支撑，注明出处等级）：**

- "context file 不提升成功率、平均 +20% 成本"——Gloaguen arXiv 2602.11988（注明预印本在审、4 agent 条件）。
- "agent 的 token 大头本来就是读：76.1%"——SWE-Pruner arXiv 2601.16746（预印本，Sonnet 4.5 + SWE-Bench Verified）。
- "信息量不变、多分几篇文档本身就降 20%"——Levy EMNLP 2025 Findings（同行评审；注明 RAG/QA 场景）。
- "语料从 54 篇堆到 1,128 篇，答对率 75% 跌破 40%"——Subedi arXiv 2606.11350（预印本、真实部署；注明非 coding）。
- "相似但不命中的文档比随机噪声更伤（随机反而 +35%，related 加 1 篇即 −25%）"——Cuconasu SIGIR 2024（同行评审；注明 7B 模型代际）。
- "语料库越大，检索命中精度越差（dense 检索假阳性随 index size 上升）"——Reimers & Gurevych ACL 2021（同行评审；注明是 IR 层结论非 agent 层）。
- "官方口径：context 是有限资源，追求最小高信号 token 集"——Anthropic 官方博客原文；"Bloated CLAUDE.md files cause Claude to ignore your actual instructions"——官方 best practices 原文。
- "CLAUDE.md 每 session 全量注入、200 行红线"——Claude Code 官方文档原文。

**只能作自家实测口径（文献无对应，注明 N 与场景）：**

- "案例组 ~1.5–2× 轮次/成本税"（ANR 试点 A 组，N 小）。
- "29/29 压不出中毒；测量可得时 agent 自救"——这是对 surface-bench 的**边界条件发现**，文献里没有人测过"测量可得性"这个调节变量，属自有贡献，但只能以"我们的实测"口径讲。
- "知识库价值 = 命中率 ÷ 语料规模"——无文献出处，只能作自家启发式；更严谨形式见 5.3。

**禁说清单：**

- ✗"案例多了一定差"（GPT-3/many-shot 直接反例；说饱和 + 部分任务下降）。
- ✗"堆文档会让 agent 答错"（常驻小尺度三篇一致无显著影响 + 自有 29/29；正确率主张必须挂条件）。
- ✗ 拿 Shi 2023 当"repo 堆文档降正确率"的证据（模型代际两代前、强制注入非自主检索、算术非代码——只能作机制起点，且要提新模型降级点后移与截断伪影研究）。
- ✗ 把 RAG 语料稀释数字直接说成 coding agent 数字（必须注明场景与迁移边界；且反向证据存在——Ning：检索中介下语料扩大带来新覆盖时持续有益）。
- ✗"Vercel 证明压缩 80% 无损"（40KB 全量嵌入条件没有通过率数据，"全量 vs 索引"对照不存在——只能说"8KB 策划索引拿到 100%"）。
- ✗"文献证明了语料税公式"（除号形式无出处；"命中率随规模下降"可引 Reimers & Gurevych，但公式整体只能作自家表述）。
- ✗ 用 "index bloat" 一词冒充 IR 文献术语（SEO 用语）。

### ③ 自有实测与文献的关系

- **互证**：29/29 未中毒 + 税实 ↔ Gloaguen"成功率不动、成本 +20%"、Khatri"正确率零差异"——**三方独立指向同一结构：喂料动成本不动正确率**。这是"语料税"叙事最硬的三角互证，且自有实测的场景（经验文档 + 诊断任务）恰好补了两篇论文没测的文档类型。
- **冲突（已调和）**：surface-bench 0–32% 中毒 vs 自有 29/29 自救——调和变量是**信息通道结构**：surface-bench 里文档是唯一信息载体（hidden dependency），自有夹具里测量始终可得（bench/cProfile/埋点）。两边合起来的完整命题："文档的正确率危害 = f(竞争性验证通道是否在场)"——这比任何单边结论都强，且与 ep03 已录的"构造三原则"完全一致。
- **互补（填空白）**：文献缺"repo 文档量为自变量的受控剂量实验"（本报告问题 2 的核心【未找到】）——自有 spike 系列恰好站在这个空白里；N 小、单场景，不能称 benchmark，但"文献没人测过"本身就是课程内容的稀缺性依据。同时文献补了自有实测测不了的：跨 agent 重复（Gloaguen 4 组）、检索尺度（RAG 两篇）、机制谱系（Cuconasu 的噪声分类）。

**对裁决的净结论**：调研支持"语料税"叙事骨架——税（成本/轮次）有强的多来源一手证据，且"正确率不伤"的否定性结论本身也有三篇独立研究背书（这让 29/29 从"压不出的失败"变成"与文献一致的发现"）；危险条件收窄为"测量不可得 ∧ 相似不命中密度高"与文献的条件化结论吻合。诚实边界必须保留：反方证据同样真实——策划过的准确文档收益巨大（Vercel +47pp、surface-bench 成本 −36–46%、Lulla −28.6%），检索中介下语料扩大带来新答案覆盖时持续有益（Ning）——所以课程靶心必须打"未策划的堆"而非"文档"，这与"劝退堆料不劝退文档"的既有口径一致。若跑 v2，其价值不在推翻 29/29，而在补"相似不命中密度"这个文献只在 RAG 场景量化过的变量在 coding 场景的首个数据点。
