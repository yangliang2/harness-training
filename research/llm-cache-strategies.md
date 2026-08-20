# 各家 LLM API 的 prompt/context 缓存策略对比

调研日期：2026-08-20。所有条目只采用官方文档/官方定价页/官方博客一手来源（例外逐条标注），供"僵尸会话复活要不要付全价"一集使用。关联实测证据：`spike/verify/ep06-ep07/cache-test/`、`spike/verify/ep06-ep07/CONCLUSION.md`、`spike/verify/ep06-ep07/partial-findings.md`。

---

## 1. Anthropic Claude

来源：[Prompt caching 官方文档](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)（2026-08-20 抓取）。

- **类型**：两种并存——自动缓存（请求顶层放一个 `cache_control`，断点自动前移）+ 显式断点（`cache_control` 打在具体 block 上，最多 4 个）。
- **TTL**：默认 **5 分钟**，每次命中免费刷新；可选 **1 小时**（`cache_control: {"type": "ephemeral", "ttl": "1h"}`）。注意官方细节：TTL 从**请求开始**计时，生成耗时也吃 TTL（响应流了 4 分钟，下一条要在约 1 分钟内发出）。
- **价格倍数**：5min 写 = 1.25× 基础输入价；1h 写 = 2×；读 = 0.1×（即命中 90% 折扣）。例：Claude Fable 5 输入 $10/MTok，5min 写 $12.50，1h 写 $20，读 $1。
- **最小可缓存长度**：512–4096 token，因模型而异（Opus 5/Fable 5 为 512）。
- **Claude Code 默认行为**：GitHub issue [anthropics/claude-code#46829](https://github.com/anthropics/claude-code/issues/46829)（2026-08-20 抓取）——用户用两台机器 119,866 次调用的会话 JSONL 证明：2026-02-01 起 Claude Code 全部走 1h TTL，**2026-03-06 起 5m token 重新出现、03-08 起 5m 占主导**，即默认 TTL 从 1h 被改回 5min，造成 17–26% 的缓存创建成本浪费。**注意：这是用户侧数据分析，截至抓取日 Anthropic 官方未在 issue 中确认或否认**——课程引用时应表述为"社区实测发现"，不是官方公告。

**挂机复活成本**：挂机 >5 分钟 → 缓存失效 → 复活时全量上下文按 1.25× **写入价**重付。六家中唯一"复活比新开会话还贵"的（写入有溢价）。

## 2. Moonshot Kimi（K3）

来源：[使用 Kimi API 的 Context Caching 功能](https://platform.kimi.com/docs/guide/use-context-caching-feature-of-kimi-api)（2026-08-20 抓取）；[Kimi K3 定价页](https://platform.kimi.com/docs/pricing/chat-k3)（2026-08-20 抓取）；[Kimi K3 官方博客](https://www.kimi.com/blog/kimi-k3)（2026-08-20 抓取）。

- **类型**：**全自动**，官方原话"无需手动创建、无需引用缓存 ID、无需管理 TTL：缓存的生命周期由系统自动管理"。对**所有模型请求**自动启用。
- **TTL**：**官方不公布具体 TTL**，只承诺由系统管理。命中条件有明确门槛：前一请求 prompt tokens **>256** 才可被缓存/命中（<256 直接丢弃不缓存）。
- **命中判定**：前缀匹配，固定上下文（system prompt、知识文档、工具定义）放 messages 最前面。
- **价格**：K3 缓存命中 $0.30/MTok vs 未命中 $3.00/MTok（90% 折扣；中国区定价页为 ¥2/¥20 每 1M token），输出 $15。K3 博客原话："Powered by Mooncake's disaggregated inference architecture, the official Kimi API achieves a cache hit rate above 90% in coding workloads."——**Mooncake 架构与 90%+ 命中率的一手出处即此博客**。
- **无写入溢价**：未命中就按标准输入价计费，没有 Anthropic 式的 write premium。

**挂机复活成本**：官方无 TTL 承诺，但实测（`spike/verify/ep06-ep07/cache-test/`，2026-08-19）：闲置 12 分钟后 cache_read 40k（部分存活，74k→40k 存在分段失效但从未归零）；**闲置 6 小时后 cache_read 77,824、cache_creation 0，全量命中**。文档"自动管理生命周期"与实测不矛盾——文档没承诺任何 TTL，实测显示该端点 TTL ≥6 小时。课程口径：k3 上"挂机丢缓存"不构成真实痛点。

## 3. DeepSeek

来源：[Context Caching on Disk 官方文档](https://api-docs.deepseek.com/guides/kv_cache)（2026-08-20 抓取）；[官方定价页](https://api-docs.deepseek.com/quick_start/pricing)（2026-08-20 抓取）。

- **类型**：**全自动磁盘缓存**，默认开启，无需改代码。
- **命中判定**：不是任意前缀匹配，而是必须**完整匹配一个"cache prefix unit"**。unit 产生于：①每次请求的用户输入末尾和模型输出末尾；②系统检测到多请求公共前缀时持久化公共前缀；③长输入/输出按固定 token 间隔切分。官方给了两轮对话（A+B → A+B+C 命中）和长文档问答（前两次不命中、第三次命中公共前缀）两个例子。命中情况从 `usage.prompt_cache_hit_tokens` / `prompt_cache_miss_tokens` 读取。
- **TTL**：无固定 TTL。官方原话："Once the cache is no longer in use, it will be automatically cleared, usually within **a few hours to a few days**." 且明确 "best-effort"，不保证 100% 命中。
- **价格**（DeepSeek-V4，peak）：Pro 命中 $0.044 vs 未命中 $1.32 = **30×**；Flash 命中 $0.014 vs 未命中 $0.44 ≈ 31×。off-peak（UTC 01:00–04:00、06:00–10:00 之外）再减半。无写入溢价。

**挂机复活成本**：挂机几分钟到几小时大概率仍命中；闲置数天缓存清除后按**标准输入价**重付——只是丢折扣，没有惩罚性溢价。

## 4. OpenAI（GPT-5 系）

来源：[Prompt caching 官方指南](https://platform.openai.com/docs/guides/prompt-caching)（2026-08-20 抓取）。

- **类型**：**全自动**，gpt-4o 及更新模型默认启用，无代码改动、**无写入费用**（官方 FAQ："Will I be expected to pay extra for writing to Prompt Caching? No."）。
- **触发条件**：prompt **≥1024 token** 才可缓存；按前缀哈希路由（通常取前 256 token），可用 `prompt_cache_key` 提升命中率；同一前缀+key 超过约 15 RPM 会溢出到其他机器、降低命中。
- **TTL**：两档。**in-memory（默认）**：不活跃 **5–10 分钟**即驱逐，最长不超过 1 小时。**extended retention**：`prompt_cache_retention: "24h"`，最长 **24 小时**，限 gpt-5.2 / gpt-5.1 系 / gpt-5 / gpt-5-codex / gpt-4.1 等（KV tensor  offload 到 GPU 本地存储；注意此档不再符合 Zero Data Retention）。两档价格相同。
- **折扣**：官方指南口径为"input token costs 最高降 90%"（即命中价可低至输入价的 10%，具体倍数因模型而异，以定价页为准）。命中量见 `usage.prompt_tokens_details.cached_tokens`。

**挂机复活成本**：默认档挂机 >10 分钟缓存大概率失效，复活按标准输入价计——丢折扣但无写入惩罚。GPT-5.x 可开 24h extended retention 覆盖"隔夜复活"。

## 5. Google Gemini

来源：[Context caching 官方文档](https://ai.google.dev/gemini-api/docs/caching)（页面标注最后更新 2026-08-13，2026-08-20 抓取）；[Gemini API 定价页](https://ai.google.dev/gemini-api/docs/pricing)（2026-08-20 抓取）；[CachedContent API 参考](https://ai.google.dev/api/caching)（2026-08-20 抓取）。

- **类型**：**双轨**。
  - **隐式缓存**：Gemini 2.5 及更新模型**默认开启**，自动命中、自动让利，官方明示不保证命中（"send requests with similar prefix in a short amount of time"以提高概率）。最低 token 门槛：Gemini 2.5 Pro/Flash 2048，Gemini 3.x 各型号 4096。命中量见 `usage.total_cached_tokens`。
  - **显式缓存（CachedContent API）**：手动创建缓存对象、引用 cache 名，`ttl` / `expireTime` 可配（示例含 300s、2h）。注意：新的 Interactions API **只支持隐式缓存**，显式缓存须走 generateContent API。
- **TTL**：隐式——官方未公布数值，只保证"自动"。显式——TTL 由用户设置；**默认 1 小时**的说法广泛见于二手资料，但本次抓取的官方 API 参考页未复述默认值，**标注为未获一手确认**。
- **价格**（付费档）：命中输入约为标准输入的 **10%**（Gemini 2.5 Pro $0.125 vs $1.25；2.5 Flash $0.03 vs $0.30；3.1 Pro $0.20 vs $2.00，均每 1M token）。**显式缓存另收存储费**：2.5 Pro / 3.1 Pro 为 **$4.50/1M token/小时**，2.5 Flash 为 **$1.00/1M token/小时**，按 TTL × token 数计——挂着的显式缓存不用也在烧钱。

**挂机复活成本**：隐式路径挂机一段时间（分钟级，未公布）后丢折扣、按标准价计，无惩罚；显式路径只要 TTL 没过就仍命中，但持续付存储费——这是六家中唯一"缓存闲置本身计费"的模式。

## 6. 智谱 GLM

来源：[上下文缓存 - 智谱AI开放文档](https://docs.bigmodel.cn/cn/guide/capabilities/cache)（2026-08-20 抓取）。

- **类型**：**全自动隐式缓存**，"无需手动配置"，支持 GLM-5.2/5.1/5 系列等；命中量见 `usage.prompt_tokens_details.cached_tokens`。
- **命中判定**：基于内容相似度自动触发，"完全相同的内容缓存命中率最高，轻微的格式差异可能影响缓存效果"。
- **TTL**：**官方未公布具体数值**，只说"缓存有合理的时效性，过期后会重新计算"。
- **价格**：命中 token "**通常为标准价格的 50%**"（官方计费说明原话；仅适用标准 API 按量计费，不含资源包和 GLM Coding Plan）。

**挂机复活成本**：TTL 未公布无法量化，但结构上同 OpenAI/DeepSeek——过期只是按标准价重付，无写入溢价。

---

## 一行结论卡

| 厂商 | 缓存类型 | TTL | 命中折扣 | 挂机 N 分钟后复活是否多付钱 |
|---|---|---|---|---|
| Anthropic Claude | 自动+手动断点 | 5min 默认（可选 1h） | 读 0.1×（90% off）；写 1.25×/2× | **是，且最狠**：>5min 复活按 1.25× 写入价全量重付，比新开会话还贵 |
| Moonshot Kimi K3 | 全自动（Mooncake） | 不公布，系统管理；实测 ≥6h | 0.1×（$3→$0.30） | **否**（实测 6h 仍全量命中；即使 miss 也只是标准价，无写入溢价） |
| DeepSeek | 全自动磁盘缓存（prefix unit 匹配） | 无固定值，"数小时到数天"自动清除 | ~1/30（30–31× 价差） | 分钟内否；闲置数天后丢折扣按标准价，无惩罚 |
| OpenAI GPT-5 系 | 全自动 | in-memory 5–10min（≤1h）；可开 24h 档 | 最高 90% off，写入免费 | 默认档 >10min 丢折扣按标准价；开 24h 档则否 |
| Google Gemini | 隐式自动 + 显式 CachedContent | 隐式未公布；显式用户自设（默认 1h 未获一手确认） | 命中 ~0.1×；显式另收存储费 $1–4.5/MTok/h | 隐式：丢折扣按标准价；显式：TTL 内仍命中但持续付存储费 |
| 智谱 GLM | 全自动隐式 | 未公布（"合理时效性"） | 通常 50% | 未知 TTL；过期按标准价，无惩罚 |

## 对课程的启示

1. **"僵尸复活成本"真正存在惩罚性溢价的只有 Anthropic 一家**：5min TTL + 1.25× 写入价意味着挂机 5 分钟后复活，整个上下文按**高于新开会话的价格**重写缓存。叠上 2026-03 起 Claude Code 官方后端 TTL 1h→5min 的变更（issue #46829，社区实测、官方未确认），这正是 ep7"僵尸会话复活要不要付全价"的核心素材——答案对 Claude 官方后端是"要，而且加价 25%"。
2. **其余五家复活只是"丢折扣"，不是"罚款"**：OpenAI、DeepSeek、Gemini 隐式、GLM 均无写入溢价，缓存过期后无非按标准输入价计——复活成本 ≤ 新开会话成本。课程应区分"复活有罚"（Anthropic）与"复活无罚但无折扣"（其他家），避免一概而论。
3. **k3 实测与官方文档吻合**：官方只承诺"全自动、系统管理生命周期、不公布 TTL"（[Context Caching 文档](https://platform.kimi.com/docs/guide/use-context-caching-feature-of-kimi-api)），实测 6 小时闲置仍全量命中、12 分钟部分失效（74k→40k）从未归零——在文档承诺范围内，且印证 Mooncake 架构下编码负载高命中率的说法（[K3 官方博客](https://www.kimi.com/blog/kimi-k3)）。课程讲 k3 时不能说"无限缓存"，正确口径是"TTL 不公布但实测 ≥6 小时，挂机丢缓存不是该端点的真实痛点"。
4. **keepalive 的价值高度依赖后端**：Anthropic 5min TTL 下 keepalive 有明确盈亏平衡账（官方口径：5min 缓存 1 次命中回本、1h 缓存 2 次回本）；k3/DeepSeek 这类长寿自动缓存端点上 keepalive 演示不出来（ep06-ep07 实测已证）。引用 TTL 数字时必须限定后端。
5. **Gemini 显式缓存是唯一"闲置也计费"的模式**（存储费 $1–4.5/MTok/h），与"僵尸会话"话题正好构成对照：别家担心缓存死了多付钱，Gemini 显式缓存是缓存活着也在收钱。

## 未获一手确认、课程引用需注意的条目

- Claude Code TTL 1h→5min：来源是社区 issue #46829 的用户数据分析，Anthropic 官方未确认。表述为"社区实测"。
- Gemini 显式缓存默认 TTL=1h：官方旧版指南口径，本次抓取的现行官方页面未复述。
- GLM 缓存 TTL：官方完全未公布数值。
- OpenAI 具体命中折扣倍数：官方指南只给"最高 90%"，逐模型缓存价需以定价页为准（定价页为 JS 渲染，本次未能机器读取）。
