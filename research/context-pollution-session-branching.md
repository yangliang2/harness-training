# 上下文污染（一个会话炒三盘菜）与分流解法：行业认知调研

调研日期：2026-08-20（所有出处均于当日抓取）
对应课程：ep6「一个会话炒三盘菜：支线话题污染主线」，解法 `/btw` / `/fork` / `/subtask` 分流

---

## 问题 1：这个反模式本身是不是行业常识？

**判定：【常识】——官方文档已亲自命名，社区讨论密度高，且有多个约定俗成的名字。**

- Anthropic 官方《Claude Code Best Practices》在 "Avoid common failure patterns" 一节里把这个反模式命名为 **"The kitchen sink session"**，原文："You start with one task, then ask Claude something unrelated, then go back to the first task. Context is full of irrelevant information. Fix: `/clear` between unrelated tasks." —— 课程的反模式与官方命名几乎一一对应。
  出处：https://www.anthropic.com/engineering/claude-code-best-practices （2026-08-20 抓取）
- 社区已有专门的反模式条目：Agent Patterns 收录 **"The Kitchen Sink Session Anti-Pattern in AI Agents"**（2026-06-08），描述"一个会话开一整天、PR review / 新功能 / 调试轮着来，每个任务留下残渣"。
  出处：https://agentpatterns.ai/anti-patterns/session-partitioning/ （2026-08-20 抓取）
- 约定俗成的术语至少有四组，互相重叠：
  - **context pollution / 上下文污染、串味**（工程社区通用语，无单一来源）；
  - **context rot**（Chroma Research 2025-07 技术报告命名，指输入变长性能就劣化）；
  - **lost in conversation**（Microsoft Research / Salesforce 论文命名，多轮对话性能崩塌）；
  - **context poisoning / distraction / confusion / clash** 四分类（Drew Breunig 提出，LangChain 博客 2025-07 引用，Simon Willison 2025-06-29 推荐）。
  出处：https://research.trychroma.com/context-rot ；https://arxiv.org/abs/2505.06120 ；https://www.langchain.com/blog/context-engineering-for-agents ；https://simonwillison.net/2025/Jun/29/how-to-fix-your-context/ （均 2026-08-20 抓取）

结论：问题本身不是课程可以"首曝"的东西——官方都写了，讲了不新鲜。课程的差异化空间在解法的精细度和认知差（见问题 4）。

## 问题 2：主流解法是什么？观众大概率已经会了吗？

**判定：粗粒度解法【常识】；命令级分流决策【几乎没人讲】。**

主流解法（按出现频率排序）：

1. **`/clear` / 新任务开新会话**——官方首推，社区复述极多：
   - Anthropic 官方："Use `/clear` frequently between tasks"，并给了更细的规则："同一问题上纠正超过两次 → 上下文已被失败尝试污染 → `/clear` 后换更好的 prompt 重来"。
     出处：https://www.anthropic.com/engineering/claude-code-best-practices （2026-08-20 抓取）
   - 社区高转述："New task → new session"（Sébastien Dubois 2026-04-16）、"New task, new session"（mindstudio 2026-04-28）、DEV Community "3 rules for cleaner context"（2026-05-18）。
     出处：https://www.dsebastien.net/claude-code-tips-and-best-practices/ ；https://www.mindstudio.ai/blog/claude-code-tricks-and-techniques ；https://dev.to/ouvarov/stop-chatting-with-claude-code-3-rules-for-cleaner-context-and-lower-bills-235d （均 2026-08-20 抓取）
2. **subagent 隔离探索**——官方与社区共识："用 subagent 调查 X，探索发生在独立上下文，主会话只收摘要"。出处同上官方页。
3. **`/btw` 处理支线小问题**——官方 best practices 明确写："For questions that don't need to stay in context, use `/btw`. The answer never enters conversation history." 但社区层面对 `/btw` 的认知仍是"新功能科普"阶段（见问题 5），会用的观众占比预计很低。
4. **`/fork` / `/subtask` 分流**——v2.1.212（2026-07-17）刚定型，距调研日仅一个月，目前只有 cheat sheet 类站点收录，**几乎看不到成体系的"什么时候用哪个"的讲解**。

结论："新任务新会话"观众大概率已会（讲了会水）；但"一条支线进来时，/btw、/fork、/subtask、/clear、/rewind 五选一怎么选"这个决策树，市面上没有成熟内容，是真空白。

## 问题 3：有没有公开翻车案例/数据可以引用？

**判定：【有人讲】——有高质量定量研究，且至少一篇拿了顶会最佳论文。**

可直接引用的三组数据：

- **"Lost in Conversation" 效应**：Microsoft Research + Salesforce Research，arXiv 2505.06120（2025-05），模拟 20 万+ 对话、15 个主流模型（含 GPT-4.1、Gemini 2.5 Pro、o3、DeepSeek-R1），指令拆到多轮后**平均性能下降 39%**；根因是模型在早期轮次做出错误假设后"越陷越深且无法自行恢复"。**该论文获 ICLR 2026 Best Paper**，权威性足够上镜。
  出处：https://arxiv.org/abs/2505.06120 ；https://www.microsoft.com/en-us/research/publication/llms-get-lost-in-multi-turn-conversation/ ；https://github.com/Microsoft/lost_in_conversation （均 2026-08-20 抓取）
- **Context Rot**：Chroma Research 技术报告（2025-07），18 个前沿模型、194,480 次调用，任务复杂度不变只拉长度，结论是"性能随输入变长而下降，远在窗口填满之前就开始，信息的排布方式比是否在场更影响结果"。注意它是行业报告非同行评审，引用时建议标注。
  出处：https://research.trychroma.com/context-rot （2026-08-20 抓取）
- **会话分支实证**：arXiv 2512.13914（2025-12）"A Version Control Approach to Exploratory Programming"，实验结论是 conversation branching（即课程的分流思路）能缓解 exploratory programming 中的 context pollution——课程解法的直接学术背书。
  出处：https://arxiv.org/html/2512.13914v1 （2026-08-20 抓取）
- 奠基文献还可提 "Lost in the Middle"（Liu et al. 2023）作为长上下文注意力衰减的源头。

注：没有找到"某工程师一个会话炒三盘菜导致具体事故"的叙事型翻车案例；定量研究比轶事更硬，建议课程用数据而非故事。

## 问题 4：有没有"大家都这么以为但其实是错的"的认知差？

**判定：【几乎没人讲】——这是本课最值钱的部分，至少有四个可打的点。**

1. **"窗口没满就没事"是错的安全感。** Context Rot 的实证是性能远在窗口填满前就开始劣化（Chroma），但社区大量讨论仍以"百分比用量条"为唯一信号。Anthropic 官方也只说 "performance degrades as it fills"，没量化"从多早开始"。
   出处：https://research.trychroma.com/context-rot ；https://www.anthropic.com/engineering/claude-code-best-practices （2026-08-20 抓取）
2. **"compact 一下就去污染了"是错的。** `/compact` 释放的是 token 空间，不是相关性：压缩摘要会把早期错误假设一并浓缩保留，而 Lost in Conversation 证明模型无法从早期错误假设自行恢复。官方自己的隐含答案也是"纠正两次失败就该 `/clear` 重开，而不是 `/compact` 续命"——但没有内容把这两件事连起来讲。
   出处：https://arxiv.org/abs/2505.06120 ；https://www.anthropic.com/engineering/claude-code-best-practices （2026-08-20 抓取）
3. **`/fork` 的语义换过三次，存量认知全是错的。** 时间线：v2.1.77–161 `/fork` 是 `/branch` 的别名；v2.1.161–211 它启动会话内 forked subagent；v2.1.212（2026-07-17）起 `/fork` 变成"把整个对话复制成独立后台会话（在 `claude agents` 里单列一行）"，旧行为改名 `/subtask`。在 2026-07 之前学会 `/fork` 的人，肌肉记忆全部过时。
   出处：https://blakecrosley.com/guides/claude-code ；https://github.com/luongnv89/claude-howto/blob/main/zh/01-slash-commands/README.md ；https://pub.towardsai.net/the-month-claude-code-became-a-fleet-and-got-brakes-b8dad5664166 （均 2026-08-20 抓取）
4. **"支线问题直接问一句没关系"是错的。** 直接提问会把问答永久写进主会话历史、消耗后续每轮 token；`/btw` 的答案不进历史（官方明示）。多数用户不知道这个代价结构。qwen-code 社区 issue #2370 的动机描述（"quick clarifying questions … add noise to the context and consume unnecessary tokens"）佐证这是跨工具的真实痛点。
   出处：https://www.anthropic.com/engineering/claude-code-best-practices ；https://github.com/QwenLM/qwen-code/issues/2370 （2026-08-20 抓取）

## 问题 5：相关官方功能/命令的现状（课程引用准确性）

**判定：功能真实存在，但行为细节新、易讲错，需按下表引用。**

Claude Code（课程三个命令的出处，注意 v2.1.212 是 Claude Code 版本号，2026-07-17 发布）：

| 命令 | 现状 | 出处（2026-08-20 抓取） |
|---|---|---|
| `/btw` | v2.1.72 引入（约 2026-03），Erik Schluntz 开发。流式输出中也可用的支线提问，答案在浮层展示、**不写入主会话历史** | https://github.com/anthropics/claude-code/issues/35585 ；https://www.contextstudios.ai/blog/3-outils-qui-ont-chang-notre-faon-de-coder-avec-les-agents-ia ；https://www.anthropic.com/engineering/claude-code-best-practices |
| `/fork` | v2.1.212 起：**复制当前对话为一个独立后台会话**（在 `claude agents` 中单独成行），适合"大一点的支线/长任务不阻塞主会话" | https://blakecrosley.com/guides/claude-code ；https://pub.towardsai.net/the-month-claude-code-became-a-fleet-and-got-brakes-b8dad5664166 |
| `/subtask` | v2.1.212 起：接管旧的**会话内 forked subagent** 行为，适合短小的子任务 | 同上；https://claude-commands.com/ |
| `/clear` `/compact` `/rewind` | 官方 best practices 的三件套：`/clear` 用于不相关任务之间和"纠正两次失败"之后；`/rewind`（Esc×2）支持从检查点"Summarize from here"做部分压缩 | https://www.anthropic.com/engineering/claude-code-best-practices |

生态呼应（佐证该设计方向是行业共识）：qwen-code 有 `/btw` 提案（issue #2370）；GitHub Copilot 2026-08 也上线了 `/btw`（"side chat shares the context and prompt …"）；Kimi Code CLI 同样有 `/btw` 和 `/fork`。
出处：https://github.com/QwenLM/qwen-code/issues/2370 ；https://github.blog/changelog/2026-08-07-github-copilot-weekly-releases-august-3/ ；https://www.kimi.com/code/docs/en/kimi-code-cli/reference/slash-commands.html （均 2026-08-20 抓取）

**跨工具陷阱（课程必须讲清楚，否则误导）**：同名命令语义不同。Kimi Code 的 `/fork` 是"从当前会话叉出独立副本、你留在原会话、去 `/sessions` 找副本（v0.33.0 起不再自动切过去，且保存的 `/goal` 不随副本）"；其 `/btw` 是"在 forked sub-Agent 里开支线对话"。**Kimi Code 没有 `/subtask` 命令**（子任务隔离走 Agent/subagent 机制）。出处：https://www.kimi.com/code/docs/en/kimi-code-cli/guides/sessions.html ；https://www.kimi.com/code/docs/en/kimi-code-cli/release-notes/changelog.html （2026-08-20 抓取）

---

## 对课程定位的启示

**讲了会水（观众早知道，一笔带过即可）：**
- "一个任务一个会话""不相关任务之间 `/clear`"——Anthropic 官方亲手命名 kitchen sink session，社区复述铺天盖地，是本课的背景板而非卖点。
- "上下文塞满性能会降"——官方第一课，无人不知。

**讲了值钱（真盲区，课程主体应该放这里）：**
1. **分流决策树**：支线进来时 `/btw`（碎问题，不留痕）/ `/subtask`（会话内短子任务）/ `/fork`（独立后台长支线）/ `/clear`（彻底换任务）/ `/rewind`（回溯止损）五选一怎么选——市面无成体系内容，是本课可独占的框架。
2. **`/fork` 语义三迁**：2026-07-17 之前学的 `/fork` 全是旧行为；v2.1.212 才一个月大，观众心智模型大概率过时。
3. **compact ≠ 去污染**：压缩省 token 不省相关性，错误假设会被摘要固化——有 ICLR 2026 最佳论文（39% 多轮劣化、无法从早期错误假设恢复）背书，是最硬的一击。
4. **"没满就没事"的错觉**：context rot 远在窗口填满前开始（Chroma，18 模型 19 万次调用），用量条不是安全信号。
5. **跨工具同名不同义**：Claude Code vs Kimi Code 的 `/fork` 行为不同、Kimi Code 无 `/subtask`——对多工具用户是防踩坑内容。

引用口径提醒：Chroma 报告非同行评审需标注；Microsoft/Salesforce 论文可直接以"ICLR 2026 Best Paper"引用；"kitchen sink session"作为官方命名可直接借用，与课程的"一个会话炒三盘菜"形成官方/俗语对照，是个好开场。
