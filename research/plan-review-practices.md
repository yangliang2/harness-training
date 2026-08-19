# AI 生成 plan/spec 的"盲批"问题：业界已验证实践调研

> 调研日期：2026-08-19。只采一手来源（官方文档、工程博客、论文原文），每个关键论断附 URL 与原文引用。查不到一手出处的论断在文末"证据边界"明确标注。

## 0. 核心 reframing（调研中发现的更好 framing）

人类因素文献和工业实践收敛到同一个结论：**问题不是"人没认真审 plan"，而是"审一篇 800 字散文 plan"这个任务本身的验证复杂度（verification complexity）超出了人的认知带宽**。说教、培训、问责、双人复核在临床上都被实证证明无效或收效甚微。所有被验证有效的做法都在干同一件事：**把"审 plan"改造成一个低认知负载的动作**——要么缩小被审对象，要么把被审对象换成可执行工件（测试/验收命令/必答题），要么干脆减少审批次数。

这为课程反范式"plan 盲批"提供了一个有实证支撑的总原则：**不要让人读得更仔细，要让"验证"变得更便宜。**

### 0.1 反面证据：哪些"直觉解法"已被实证否掉（论文原文）

- **说教/培训/问责基本无效**。Lyell & Coiera 对 1983–2015 年 40 项 automation bias 实验的系统综述（JAMIA，原文开放获取）：
  > "To date, interventions designed to counter automation bias have had little or no impact. Interventions tested thus far have manipulated user accountability for performance, which had only a mild effect on novice subjects and **no effect on expert subjects**. … Providing subjects with feedback on performance had no impact, and **training interventions resulted in no significant reduction in rates of automation bias**."
  > "If automation bias is partly due to cognitive demands that exceed the user's capacity, interventions seeking to reduce automation bias that do not address this cognitive overload are unlikely to succeed."
  — [Lyell D, Coiera E. "Automation bias and verification complexity: a systematic review." J Am Med Inform Assoc 2017](https://pmc.ncbi.nlm.nih.gov/articles/PMC7651899/)
  → 直接支持课程约束 1（说教式整改不可持续不是态度问题，是有 40 项实验背书的人因事实）。

- **双人制（two-person crew）不是解药**。Skitka, Mosier, Burdick & Rosenblatt 在模拟飞行任务中对比双人机组与单人：
  > "**Teams and solo performers were equally likely** to fail to respond to system irregularities or events when automated devices failed to indicate them, and to incorrectly follow automated directives when they contradicted other system information."
  — [Skitka et al. "Automation Bias and Errors: Are Crews Better Than Individuals?" Int. J. Aviation Psychology 10(1), 2000（摘要原文镜像）](https://docslib.org/doc/9413383/exploration-of-anchoring-confirmation-and-overconfidence-bias-in-diagnostic-decision-making)
  → "找第二个人一起看 plan"不能作为课程整改动作。

- **唯一被综述指向的有效杠杆是降低验证复杂度**：同一综述发现，单任务环境下出现 automation bias 的实验全部是"中—高验证复杂度"任务；结论建议 "Strategies to minimize AB might focus on **cognitive load reduction**"，并提出具体设计方向："Designing interfaces that support effective verification, eg, by **presenting critical verification information side-by-side with decision support**"（同上文）。→ 这是候选 1/2/3 的共同理论底座。

---

## 候选 1：批准"验收工件"，不批准 plan（验收前置 / evidence-based approval）

**机制**：plan 不以后缀散文结尾，而以一组**可运行的验收工件**结尾——验收测试、契约测试、或一份"跑完应输出什么"的命令清单。人类批准的对象从"读 800 字判断是否周全"降级为"看这份验收清单是否覆盖了我关心的场景 + 跑一次看输出"。判断漏洞（漏边界情况）在"清单里没有这条验收"这一层变得肉眼可见，而不是埋在散文里。

**一手出处**（四家独立收敛）：

1. Anthropic 官方《Claude Code Best Practices》：
   > "The most useful specs are self-contained: they name the files and interfaces involved, state what is out of scope, and **end with an end-to-end verification step that proves the feature works**."
   > "Have Claude **show evidence rather than asserting success**: the test output, the command it ran and what it returned… **Reviewing evidence is faster than re-running the verification yourself**."
   — [anthropic.com/engineering/claude-code-best-practices](https://www.anthropic.com/engineering/claude-code-best-practices)
2. GitHub spec-kit 官方方法论文档，其"宪法"第三条把"先批准测试再写实现"设为不可协商项：
   > "This is NON-NEGOTIABLE: All implementation MUST follow strict Test-Driven Development. No implementation code shall be written before: 1. Unit tests are written, 2. **Tests are validated and approved by the user**, 3. Tests are confirmed to FAIL (Red phase)."
   — [github/spec-kit/spec-driven.md](https://github.com/github/spec-kit/blob/main/spec-driven.md)
   → 注意这里人审的对象被刻意换成了**测试**（小、具体、可判对错），不是 plan 散文。
3. Kiro 官方文档：requirements.md 强制用 EARS 记法（`WHEN [条件] THE SYSTEM SHALL [行为]`），官方自述其收益第一条就是可测试性：
   > "**Testability**: Each requirement can be directly translated into test cases."
   — [kiro.dev/docs/specs/feature-specs](https://kiro.dev/docs/specs/feature-specs/)
4. StrongDM 工程博客（"Software Factory"）把这条推到极限——人只定义场景（scenarios），验收由机器对数字孪生环境执行，审查环节整个被验证取代：
   > "Humans define intent: what the system should do, the scenarios it needs to handle, the constraints that matter. After that, the agents take it from there. They generate code, validate it against real-world behavior, and iterate until it converges, **without hand-tuning or human review**. … It's what happens when **validation replaces code review**."
   — [strongdm.com/blog/the-strongdm-software-factory-building-software-with-ai](https://www.strongdm.com/blog/the-strongdm-software-factory-building-software-with-ai)
   （完整形态需要数字孪生基础设施，当天落不了地；但其最小内核——"人审 scenario 清单而非实现"——当天可用。）

**可执行度（当天落地）**：高。一条 CLAUDE.md / plan 模板规则即可：
> "任何 plan 必须以『验收清单』结尾：每条是一句可执行命令（或一个测试名）+ 预期结果；每条必须对应一个边界情况。没有验收清单的 plan 不准批准。"
观众当天抄走模板即可。若已在用 spec-kit/Kiro，则直接启用其内置机制（Article III / EARS），零新增配置。

**拍摄可行性**：高，对照强烈。A 组：盲批散文 plan（含一个漏掉的边界情况），执行阶段爆掉；B 组：plan 附验收清单，人扫清单 10 秒发现"没有并发场景的验收"，打回——漏洞在批准前被抓。5 分钟内可完整呈现。

**与已否决方案对比**：
- vs 方案 2（限长+事实断言带证据）：是其超集。事实断言的证据只查"说得对不对"；验收清单还查"漏没漏"——**缺失的验收项本身就是边界情况漏洞的可视化**。正好补上方案 2 不解决判断漏洞的短板。
- vs 方案 3（grill 批评家）：不依赖批评家 agent 的信噪比；漏洞由"清单缺项"这个结构性信号暴露，不需要人去读攻击清单。

---

## 候选 2：把 plan 变小（plan 尺寸上限 / 拆小提交单元）

**机制**：不改善"审"的动作，而是缩小被审对象本身，使其低于认知带宽阈值。代码评审领域 20 年的工业共识（大 CL 必被盲批）直接平移到 plan：plan 超过一屏 = 必然被扫一眼就批。

**一手出处**：

1. Google 官方工程实践《Small CLs》：
   > "Small, simple CLs are: **Reviewed more thoroughly.** With large changes, reviewers and authors tend to get frustrated by large volumes of detailed commentary shifting back and forth—sometimes to the point where **important points get missed or dropped**."
   > "**reviewers have discretion to reject your change outright for the sole reason of it being too large.** … In general it's better to err on the side of writing CLs that are too small vs. CLs that are too large."
   — [google.github.io/eng-practices/review/developer/small-cls.html](https://google.github.io/eng-practices/review/developer/small-cls.html)
2. Anthropic 官方对"什么时候根本不该写 plan"的口径：
   > "If you could describe the diff in one sentence, **skip the plan**."
   — 同候选 1 出处 1。
3. 人因证据支撑其机理：Lyell & Coiera 发现 automation bias 与验证复杂度（≈ 认知负载）挂钩，"the addition of secondary tasks appears to increase demands on a user to the point where errors emerge"——工件越长越复杂，盲批概率越高（出处同 §0.1）。

**可执行度**：高。一条规则："plan 超过 N 行（例如 30 行）则拆成多个 plan，逐个批准；一句话能说完的改动跳过 plan"。当天可落地。

**拍摄可行性**：中—高。对照直观（800 字 vs 5 行的批准质量），但单独成集略显单薄——它更适合作为候选 1 的前置条件（"plan 只能有一屏，省下的空间全给验收清单"）。

**与已否决方案对比**：与方案 2 的"限长"部分重合，但依据更硬（Google 官方给了 reviewer 拒收大 CL 的制度化先例，可直接类比为"拒收大 plan"）；同样不单独解决判断漏洞，需与候选 1 组合。

---

## 候选 3：把"审查"变成"答题"——机器先找洞，人只回答被点名的具体问题

**机制**：在人审之前，由工具对 spec 做跨条目的逻辑分析，产出**一串必须回答的具体问题**（每个问题附：涉及哪几条需求、问题是什么、建议的修改选项），人逐条点选回答后 spec 才被更新放行。人的动作从"通读散文找漏洞"（高验证复杂度）变成"回答 N 个是非/选择题"（低验证复杂度）。这正面覆盖"漏边界情况"——漏项是机器按 happy-path 之外的模式系统扫出来的。

**一手出处**：

1. Kiro 官方文档"Analyze Requirements"功能：
   > "you can ask Kiro to **analyze your requirements for logical inconsistencies, ambiguities, and gaps**… to find issues that are **hard to catch in a read-through**."
   > 覆盖的问题类型包括："**Missing edge cases** — failure modes, boundary conditions, and concurrent access scenarios not covered by the happy path."
   > "Each question includes the requirements involved, a plain-language explanation of the issue, and **suggested fixes you can select**."
   — [kiro.dev/docs/specs/analyze-requirements](https://kiro.dev/docs/specs/analyze-requirements/)
2. spec-kit 的等价机制（模板层强制）：spec 模板强制对不确定处打 `[NEEDS CLARIFICATION]` 标记而非猜测，并配"质量检查单"作为批准门槛：
   > "**Don't guess**: If the prompt doesn't specify something, mark it [as `[NEEDS CLARIFICATION: …]`]"
   > 批准前检查单："No [NEEDS CLARIFICATION] markers remain / Requirements are testable and unambiguous"
   — 同候选 1 出处 2。

**可执行度**：高。用 Kiro 的观众一键启用；不用 Kiro 的观众可抄其机制为一条 prompt 规则："plan 交付前，先列出你认为我可能反对/遗漏的 ≤5 个具体问题（每条附建议答案），逐条问我；我不答完不许进入执行。"——当天可用。

**拍摄可行性**：高。对照：A 组人读 800 字 plan 没发现并发问题；B 组屏幕上弹出 3 个必答题，第 2 题正是"两个用户同时提交同一订单怎么办？"，人点选答案——漏洞被抓。视觉对比非常适合短视频。

**与已否决方案 3（grill）的关系——这是关键区分点**：机制上游相同（agent 找洞），但**交付形态不同**，而这正是信噪比问题的解法：
- grill 的失败形态是"攻击清单"——一篇要人读的次生散文，万金油条目混在里面，人照样不读；
- 本候选的形态是"**阻塞式必答题队列**"——每条是一个带默认建议的具体决策点，人不回答流程不走。读清单可以扫一眼就批，答题没法扫一眼就批（不点按钮过不去）。
- Anthropic 官方也独立证实了 grill 形态的信噪比问题：> "A reviewer prompted to find gaps **will usually report some, even when the work is sound**, because that is what it was asked to do. Chasing every finding leads to over-engineering… Tell the reviewer to **flag only gaps that affect correctness or the stated requirements**."（同候选 1 出处 1）→ 即"约束找洞范围 + 改变交付形态"，而不是放弃机器找洞本身。
- **诚实的保留**：Kiro 官方没有公布 Analyze Requirements 的查全率/误报率实测数据（见证据边界），其"答题形态能压住万金油问题"目前只有机制层面的论证，没有公开 A/B 数据。

---

## 候选 4：减少审批次数，按风险分层（治"审批疲劳"本身）

**机制**：盲批的一个近因是审批请求太多——第 10 次确认时人已经不看内容了。把低风险动作移出人工审批（allowlist、沙箱、分类器自动放行），只把高风险/不可逆动作留给人，让每次人工审批都"稀有而重要"。

**一手出处**：

1. Anthropic 官方直接承认了审批疲劳 → 盲批的因果链：
   > "In Manual mode… Claude Code asks before actions that might modify your system: file writes, Bash commands, MCP tools. **That's safe but tedious. After the tenth approval you're clicking through rather than reviewing.**"
   其对策是 auto mode："a separate classifier model reviews most actions instead of you and **blocks only what looks risky**, such as scope escalation, unknown infrastructure, or hostile-content-driven actions."
   — 同候选 1 出处 1。
2. OpenAI 官方《A Practical Guide to Building Agents》对 agent 产品的 HITL 设计口径一致：人工介入应作为**高风险动作的升级触发器**（human-in-the-loop escalation），而非全程审批（该指南面向 agent 产品设计而非编程 plan 审查，作类比引用）。
   — [openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)

**可执行度**：高（权限配置：`/permissions` allowlist + 沙箱，或等价配置）。当天可落地。

**拍摄可行性**：中。可以拍"30 次确认弹窗 vs 3 次"的数量对照，冲击力强；但它治的是**执行阶段**的盲批，对"plan 内容里埋的错误假设"只是间接缓解（审批总量降了，人留给 plan 审批的注意力预算才可能回来）。**不适合单独成集，适合作为同集里的配套动作。**

**与已否决方案对比**：与三个已否决方案不冲突，正交。

---

## 证据边界（明确标注：以下论断未查到一手出处，不写入课程内容）

- **没有任何公开的一手实测数据**直接量化"某种 plan 审查干预把盲批率从 X% 降到 Y%"。上述候选的依据是：(a) 官方文档/工程博客描述的工作流设计（存在性证据），(b) 人因文献对同类机制的实证（迁移性证据）。两者之间没有公开的桥接实验。
- **Kiro Analyze Requirements 的查全率/误报率**：官方文档只描述功能，无效果数据。
- **StrongDM Software Factory 的效果指标**（scenario 覆盖率、逃逸缺陷率）在其博客中未给出可核数字，只有机制描述和 "$1,000/天/人 token 消耗"这类结构性指标；其"无需人审"结论是否适用于安全关键以外的场景，无第三方验证。
- **"答题形态比攻击清单形态信噪比高"**（候选 3 对 grill 的修正）：目前只有机制论证 + Anthropic 对清单形态缺陷的官方承认，无直接对比实验。我们自己的 grill 实测（信噪比差）是唯一一方数据点。
- Tessl（spec-centric development）公开材料以市场叙事为主，未找到描述其 review/approval 工作流机制的一手技术文档，本报告未采纳。
- OpenAI 侧未找到针对"编程 plan 审查疲劳"的专门机制设计，只有通用 agent HITL 升级原则。

---

## 推荐：如果只能选一个进课程

**选候选 1（批准验收工件，不批准 plan），以候选 2（plan 限一屏）作为同集的前置规则。**

理由：

1. **证据最厚且多源独立收敛**：Anthropic（spec 以 e2e 验证步骤结尾 / 审证据比重新验证快）、spec-kit（用户批准测试是 NON-NEGOTIABLE 的宪法条款）、Kiro（EARS 让每条需求可直接转测试）、StrongDM（validation replaces code review）——四个互不相干的来源指向同一个动作，这是本次调研中唯一达到这个证据强度的实践。
2. **唯一直面"判断漏洞"的方案**：漏边界情况在散文里不可见，在验收清单里是"少了一行"——缺失被结构化了。已否决的方案 2（事实断言+证据）管不了漏项，grill（方案 3）靠人读清单管不住注意力，本方案绕开两者。
3. **可执行度满分**：整改动作 = 一条 plan 模板规则（"plan 限一屏，末尾必须有验收清单：每条一个可运行命令+预期结果+对应边界情况；无清单不批准"）。观众当天抄走即用，不依赖任何特定工具；用 spec-kit/Kiro 的人则直接打开内置开关。
4. **拍摄对照最锋利**：同一份埋了边界漏洞的 plan，A 组散文形态盲批通过 → 执行爆炸；B 组验收清单形态，漏掉的场景在清单上肉眼可见地缺席 → 批准前打回。30 秒内能拍完的强对照。
5. **顺应人性有实证背书**：它不叫人"更仔细"，而是按 Lyell & Coiera 综述指出的唯一有效杠杆——降低验证复杂度、把关键验证信息并排呈现——来重新设计被审对象。课程可以在 5 分钟里用一句话带出这个依据："说教、培训、双人复核在这 40 项实验里全败了，唯一有效的是让验证变便宜。"

候选 3（答题形态）是最强的备选，适合作为下一集：它共享同一理论底座，且修正了你们已否决的 grill 方案的交付形态缺陷，但缺公开效果数据，进课需要你们自己做一次小规模实测兜底。
