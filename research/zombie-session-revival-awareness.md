# 僵尸会话复活（ep7）：行业认知度调研

调研日期：2026-08-20（所有出处均当日抓取）。调研对象：ep7"僵尸会话复活：缓存 TTL 与复活成本"。补充主题：'prompt cache 省钱' 社区认知度、keepalive 工具流行度与争议、Claude Code 用户账单意识讨论。六家缓存策略对比见 `research/llm-cache-strategies.md`（本报告直接引用，不重复）。

判定口径：【常识】= 官方文档 + 大量社区讨论，目标观众大概率已知道；【有人讲】= 有稳定讨论密度或成型的工具/issue，但限于深度用户圈；【几乎没人讲】= 只有零散一手痕迹，没有成体系的公开讨论。

---

## Q1. 这个反模式是不是行业常识？——判定：机制【常识】，作为"习惯反模式"【有人讲】，无约定俗成的名字

**"prompt cache 省钱"本身已是常识：**

- Anthropic 官方 2024-08 发布 prompt caching（beta），2024-12-17 GA（页面内 update 注记），官方口径"成本最高降 90%、延迟最高降 85%"，写 1.25×、读 0.1× 的定价结构即出自此文：[Prompt caching — Anthropic](https://www.anthropic.com/news/prompt-caching)（2026-08-20 抓取；注意抓取页日期字段显示 "August 14, 2025"，与页面内 2024-12-17 的 GA 更新注记矛盾，疑似官方重标日期，引用时以"2024-08 beta / 2024-12 GA"为准）。
- 2026 年围绕"5 分钟 TTL"的二手技术文章密度很高，且多围绕"多花了钱"这个角度：[Claude Prompt Caching in 2026: The 5-Minute TTL Change That's Costing You Money（dev.to）](https://dev.to/whoffagents/claude-prompt-caching-in-2026-the-5-minute-ttl-change-thats-costing-you-money-4363)、[Fix Claude Code's Prompt Caching TTL and Cut API Costs](https://claudcod.com/blog/claude-code-prompt-caching/)、[Claude Code Prompt Caching Costs: Pick the Right TTL (2026)](https://bmdpat.com/blog/claude-code-caching-costs-2026)、[Prompt Caching Economics on Fable 5](https://www.developersdigest.tech/blog/fable-5-prompt-caching-economics)（均 2026-08-20 检索命中）。说明"TTL 变化影响账单"已出圈到一般技术媒体。

**但"僵尸会话复活"作为被命名的习惯问题，社区没有统一称呼：**

- 社区讨论用的词是机制词汇——"cache miss"、"re-cache"、"cache warm/keepalive"、"session resume invalidates cache"，没有人用"zombie session"这个词（检索 "zombie session" Claude Code 无有效结果，2026-08-20）。这个反模式目前是"人人遇到过、没人起名字"的状态——这对课程反而是好消息：命名权还空着。
- 最接近的公开问题单：anthropics/claude-code#42338 和 #46829（详见 Q3），说明痛点真实且被高规格记录，但讨论都停留在 bug/定价层面，没有上升为"使用习惯"层面的反思。

**结论**：观众大概率知道"有缓存、缓存会过期、过期要重付"，但"任务死了会话还挂着、回来随手一问=为僵尸上下文付全价"这个习惯视角，目前只有 issue 级的碎片证据，没有成文的行业讨论。

## Q2. 主流解法是什么？观众大概率已经会了吗？——判定：分层

**(a) "清上下文、开新会话"——【常识】，观众大概率已会。** 官方成本文档把 `/clear` 列为第一课："Clear between tasks: Use /clear to start fresh when switching to unrelated work. Stale context wastes tokens on every subsequent message"，并明确 "/clear costs nothing"：[Manage costs effectively — Claude Code Docs](https://code.claude.com/docs/en/costs)（2026-08-20 抓取）。各类"Claude Code 省钱"文章也反复讲。讲这个会水。

**(b) keepalive（定时 ping 保缓存）——【有人讲】，但限于小众工具圈，观众大概率不会。** 这是一个有真实生态、但星标量级很小的打法：

- **Aider 内置** `--cache-keepalive-pings VALUE`："Number of times to ping at 5min intervals to keep prompt cache warm (default: 0)"——[Aider 官方配置文档](https://aider.chat/docs/config/options.html)（2026-08-20 抓取）。主流工具里唯一内置该功能的。
- Claude Code 插件 [yujiachen-y/claude-code-cache-keepalive](https://github.com/yujiachen-y/claude-code-cache-keepalive)（2026-04-14 创建，**仅 16 stars**，GitHub API 2026-08-20 查询）：用 Stop hook 的 `decision:"block"` 在 TTL 到期前注入假 turn。README 引用的账本：6 分钟空窗 naive 2.5× vs ping 后 1.95×，省约 22%；引用了 Aider、SillyTavern Cache-Refresh（自称长对话省 ~89%）、cline/cline#414 的数学推导作为 prior art。
- [jianzhichun/permafrost](https://github.com/jianzhichun/permafrost)（2026-06-10 创建，**仅 21 stars**）：DeepSeek 后端的缓存对齐代理 + keepalive，自称实测 64% 请求走廉价缓存。
- OpenClaw 也有同类需求：[openclaw#24711 — 429 限流冷却期 cache 全灭，请求官方加 keepalive ping](https://github.com/openclaw/openclaw/issues/24711)（2026-02-23）。

结论：keepalive 的概念在多个工具圈独立出现（说明需求真实），但没有任何一个实现流行起来（16–21 stars 量级），普通 Claude Code 用户基本不可能已经会了。

**(c) 按闲置时长分档决策（接着干 / 保活 / 放手）——【几乎没人讲】。** 检索未发现任何一篇把"闲置 <5min 接着干 / 5–50min 保活 / >50min 放弃"写成决策卡的内容。现有讨论全是单点：要么讲 keepalive 工具本身，要么讲 /clear。ep7 大纲的"时间轴决策卡"目前是空白地带。

## Q3. 公开翻车案例/数据——判定：【有人讲】，且数据质量高

- **[claude-code#42338：--continue / /resume 会全量摧毁缓存，哪怕只隔 2 秒](https://github.com/anthropics/claude-code/issues/42338)**（2026-08-20 抓取）。最硬的一份公开数据：~500k prompt 的 Opus 4.6 会话，exit 后 2 秒 `--continue`，`cache_read: 0`、全量 `cache_creation` 470,381；3 秒间隔重测又烧 511,847。2.6 小时内 3 次 resume 共烧 **1.43M cache_creation token，是同期有效输出的 9 倍**，小时配额消耗 2.2×。根因指向 v2.1.69 引入的 `deferred_tools_delta` 重排工具结果、破坏缓存前缀。——"僵尸会话复活"的最直接翻车实锤：不是"挂久了才贵"，是**恢复动作本身就可能全价重付**。
- **[claude-code#46829：Claude Code 默认 TTL 从 1h 被静默改回 5min](https://github.com/anthropics/claude-code/issues/46829)**（详见 `llm-cache-strategies.md`；用户侧数据分析，官方未确认）。keepalive 插件 README 给出的金额口径：该用户追到 **~$949（Sonnet-4.6）+ ~$1,582（Opus-4.6）** 的超额账单，其中 $1,198 落在单月——[claude-code-cache-keepalive README](https://github.com/yujiachen-y/claude-code-cache-keepalive)（2026-08-20 抓取）。
- **账单意识的最大公共事件：2025-07-28 Anthropic 宣布周限额**。[TechCrunch: Anthropic unveils new rate limits to curb Claude Code power users](https://techcrunch.com/2025/07/28/anthropic-unveils-new-rate-limits-to-curb-claude-code-power-users/)（2025-07-28，2026-08-20 检索）：2025-08-28 生效，覆盖 Pro $20 / Max $100 / Max $200；官方理由是个别用户 24/7 挂机跑出"远超月付"的算力 + 账号共享转售；称影响 <5% 用户。后续 2026-05-06 官方又把 5 小时窗口额度翻倍并取消高峰限流（[Morph: Claude Code Usage Limits](https://www.morphllm.com/claude-code-usage-limits)，二手，2026-08-20 检索；未找到官方公告原文，课程引用建议表述为"多家技术媒体报道"）。
- 订阅 vs API 的换算震撼案例（二手转述，[ClaudeMeter 博客](https://claude-meter.com/alternative/rate-limit-wall-vs-api-cost)，2026-08-20 检索）：Reddit 用户统计 8 个月 10B token，按 API 价约 $15,000，而 Max 订阅只花了 $800。VentureBeat 报道第三方 harness 封禁时也引用了"一个月 Claude Code 轻松跑出 API 等价 $1,000+"的说法：[VentureBeat 2026-01-09](https://venturebeat.com/technology/anthropic-cracks-down-on-unauthorized-claude-usage-by-third-party-harnesses)。

## Q4. "大家都这么以为但其实是错的"认知差——判定：【几乎没人讲】（课程最值钱的点）

以下每条都有权威出处，但没有任何一篇公开内容把它们串起来：

1. **"缓存 5 分钟是从创建开始计时的"——错，命中即免费刷新 TTL。** 官方文档明确 TTL 每次命中重置（`llm-cache-strategies.md` 已核实）。整个 keepalive 生态就建立在这个被忽视的性质上（keepalive README 称之为 "well-documented but easily forgotten property"）。多数用户的直觉是"离开 5 分钟缓存必死"，因此要么过度焦虑、要么白白重付。
2. **"keepalive 省钱"——对订阅用户是反的。** keepalive 插件 README 用整节警告：Pro/Max 订阅按 5 小时窗口的请求配额计费，"every keepalive turn eats your 5-hour quota. On subscriptions, this plugin makes things worse"（[出处同 Q2](https://github.com/yujiachen-y/claude-code-cache-keepalive)）。而且官方成本文档写明：**订阅计划的缓存寿命本来就是 1 小时**，只有开始动用 usage credits 才掉到 5 分钟；API key/云厂商才是默认 5 分钟（[Manage costs effectively](https://code.claude.com/docs/en/costs)）。也就是说订阅用户装 keepalive 插件保的是一个根本没死的缓存，还倒贴配额。这是"工具流行但适用面被严重误读"的典型案例。
3. **"恢复旧会话比新开会话便宜（至少一样）"——在 Anthropic 上是反的。** 复活 = 全量历史按 1.25× 写入价重建缓存，比新开会话还贵 25%（六家中唯一有写入溢价的，`llm-cache-strategies.md`）；再叠 #42338 的实测——哪怕 2 秒内恢复，resume 机制本身也可能摧毁缓存。"恢复免费的"直觉错得彻底。
4. **"缓存过期 = 被罚款"——只有 Anthropic 一家成立。** OpenAI（写入免费）、DeepSeek、Gemini 隐式、GLM、Kimi 全是"过期只是丢折扣，按标准价重付"（`llm-cache-strategies.md`）。"僵尸复活成本"是个**后端相关**的问题，当成通用定律讲会教错。
5. **官方已经有缓解功能，但几乎没人知道**：Pro/Max 长歇后恢复大会话时 Claude Code 会主动提供"从摘要恢复（resume from a summary）"，避免后续请求继续背全量历史；`ENABLE_PROMPT_CACHING_1H=1` 可让 usage credits 阶段也保持 1h TTL（均出自 [Manage costs effectively](https://code.claude.com/docs/en/costs)，2026-08-20 抓取）。检索未见任何社区文章提到这两个开关。

## Q5. 相关官方功能/命令现状（课程引用要准确）

全部出自 [Manage costs effectively — Claude Code Docs](https://code.claude.com/docs/en/costs) 与 [Extend Claude with skills — Claude Code Docs](https://code.claude.com/docs/en/slash-commands)（均 2026-08-20 抓取）：

- **`/usage`**：当前官方用量/成本入口（不是旧的 `/cost`，成本文档已不以 /cost 为中心；课程不要引用 /cost）。Session 块显示本会话 token 与美元估值——**本地按刊例价计算，不等于实付账单**；`/clear` 后归零（v2.1.211 前跨 /clear 累计）。订阅计划下还有归因分解（skills/subagents/plugins/MCP 各占百分比）与**行为标记：long context、cache misses 等占最近用量 ≥10% 时会被标出并给建议**——官方已经把"cache miss 烧钱"做成了一等功能。
- **`/loop`**：官方内置 bundled skill（与 /doctor、/code-review、/batch、/debug、/claude-api 同列），用法 `/loop 5m /foo`，省略间隔则模型自定节奏。**注意**：官方文档对它的定位是"周期任务"，没有一处说它是缓存保活手段——大纲里"`/loop 4m ok` 保活（官方内置）"的说法，准确表述应为"`/loop` 是官方内置的定时 turn 机制，其副作用恰好能刷新缓存 TTL"，不能说官方提供了保活功能。另据二手来源（[samwize 博客](https://samwize.com/2026/03/14/how-i-got-claude-code-to-monitor-slack-while-i-was-on-holiday/)）loop 3 天自动过期、会话死了 loop 即停——未获官方一手确认，引用需标注。
- **缓存寿命分档（官方现行口径）**：订阅 1h；动用 usage credits 后 5min；API key/云厂商默认 5min；`ENABLE_PROMPT_CACHING_1H=1` 可在 usage credits 阶段保持 1h。——与 #46829 社区实测（订阅上 5m 写 token 2026-03 起占主导）存在张力，课程引用时把官方文档口径与社区实测并列呈现。
- **resume from a summary**：Pro/Max 长歇后恢复大会话时官方提供的减损选项（见 Q4-5）。
- **`/insights`**：官方使用模式分析报告（写 HTML 到 `~/.claude/usage-data/report.html`），可作为"账单意识"环节的官方工具。
- **官方定价锚点**：写 1.25×（5min）/ 2×（1h）、读 0.1×，自 2024-08 公告起结构未变（[Anthropic prompt caching 公告](https://www.anthropic.com/news/prompt-caching)；逐模型现价以 `llm-cache-strategies.md` 当日抓取的官方文档为准）。
- **第三方账单工具的事实标准**：[ccusage](https://github.com/ccusage/ccusage)（原 ryoppippi/ccusage，2025-05-29 创建，**18,062 stars**，GitHub API 2026-08-20 查询）——读本地 JSONL，按日/周/月/会话出报告，支持 5 小时计费窗口监控和 statusline 集成，已扩展到 15 家 CLI（含 Codex、Kimi、Gemini CLI）。1.8 万星说明"看清自己的 Claude Code 账单"已是主流需求，观众对账单工具有认知基础，不需要科普。

## 对课程定位的启示

**讲了会水（观众早知道）：**

- "prompt cache 存在、能省 90%"——2024 年的官方公告 + 两年技术媒体轰炸，已是常识。
- "闲置久了缓存过期要重付"——机制层面已被 #46829 事件和大量 2026 年文章讲透。
- "/clear 省 token、换任务开新会话"——官方成本文档第一课，各类省钱指南反复讲。
- ccusage 看账单——1.8 万星的社区事实标准，提一句即可，不必教。

**讲了值钱（真盲区）：**

1. **认知差三件套（最值钱）**：①TTL 命中即刷新，不是从创建计时；②订阅用户 keepalive 是负收益（配额制 + 订阅本来就有 1h TTL）；③恢复旧会话比新开会话贵（写入溢价 + resume 摧毁缓存的实测）。三条全部有权威出处，但没有一篇公开内容把它们串成体系——这是 ep7 的独占价值。
2. **时间轴决策卡**（<5min 接着干 / 5–50min 保活 / >50min 放手或摘要恢复）：检索证实没有现成的分档决策内容，Q2-(c) 判定【几乎没人讲】。
3. **后端差异性**："复活罚款"只有 Anthropic 成立，其他五家只是丢折扣——纠正"缓存过期=罚款"的过度泛化（与 `llm-cache-strategies.md` 数字卡互为印证）。
4. **官方隐藏开关**：resume from summary、`ENABLE_PROMPT_CACHING_1H=1`、`/usage` 的 cache-miss 行为标记——官方已有但社区几乎零讨论的功能，适合"诚实边界 + 官方解法"环节。
5. **翻车数据可直接引用**：#42338（2 秒 resume 烧 470k token 重建、9× 有效输出）和 #46829（$949+$1,582 超额账单）都是高质量一手素材；注意 #46829 官方未确认，表述为"社区实测"。

**引用红线**：`/cost` 已非官方成本入口（用 `/usage`）；`/loop` 不能说成"官方保活功能"（官方定位是周期任务，保活只是副作用）；Anthropic 公告页日期字段疑似重标（显示 2025，GA 注记 2024-12），引用写"2024-08 beta / 2024-12 GA"；#46829 一律标注"官方未确认"。
