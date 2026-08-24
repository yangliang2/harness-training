# 《改掉这个习惯》课程工作台

> 系列名：《改掉这个习惯》· AI 使用反模式七讲（每集 ≤5 分钟）
> 用法：讨论某一集 = 读本 README + 对应 epXX.md。每集设计档自包含（既定事实 / 待定问题 / 讨论记录）。
> 工作流：逐集讨论 → 定一条改一条 → 状态翻"定稿" → commit。总大纲（背景）见 `antipattern-course-outline.md`；候选与出局记录见 `antipattern-candidates.md`。

## 集序与状态

| 集 | 集名 | 反模式 | 状态 | 调研报告 | 实测证据 |
|---|---|---|---|---|---|
| 1 | 让 AI 立刻变聪明 | 配置只加不减 | 定稿（2026-08-21） | research/mcp-context-tax-tool-search-doctor.md + research/skills-overload-evidence.md + research/cursor-config-parity.md | spike/verify/ep01-mcp-tax/ |
| 2 | 让 AI 听得清 | 工具输出无节制 | 定稿（2026-08-21） | research/tool-output-industry-consensus.md + research/tool-output-failure-design-scan.md + research/mcp-output-tax-industry-evidence.md | spike/verify/ep02-tool-output/ + ep02-truncation/ + ep02-mcp-extract/ |
| 3 | 让 AI 一学就会 | 无脑自建 MCP（工具=MCP 条件反射） | 方向定稿（2026-08-23，docs as code 主案例 + 判税三问；逻辑稿 v1 过审；KB spike 三臂为唯一 blocking） | research/out-of-distribution-tooling.md + research/docs-as-code-vs-kb-mcp.md | spike/verify/ep04-gh-vs-mcp/ + ep04-multi-mcp/（历史编号，属本集） |
| 4 | 让 AI 不记错事 | 参考文档只加不理（堆料） | 方向定稿（2026-08-23，语料税反转 + 三道门 + 多级索引；v2 索引税 A/B 为唯一 blocking） | research/doc-graveyard-industry-scan.md + research/repo-doc-hoarding-impact.md | spike/verify/ep03-doc-graveyard/ + ep03-experience-docs/ + ep03-anr-cases/（历史编号，属本集） |
| 5 | 让 AI 下午不变笨 | 长会话当仓库 | 待讨论 | research/compact-loss-handoff-statusline.md | spike/verify/ep05-compact/ |
| 6 | 让 AI 不走神 | 一个会话炒三盘菜 | 待讨论 | research/context-pollution-session-branching.md | spike/verify/ep06-ep07/（btw-test） |
| 7 | 让 AI 说到做到 | 空头支票（压轴） | 待讨论 | research/ai-fake-done-verification-awareness.md | spike/verify/ep08-hooks-e2e/ + spike/anti-status-smuggle/ |

> 2026-08-21 决议：原第 7 集"僵尸会话复活"整集出局，课程定版七讲、不补位；耐久结论并入第 5 集收尾，完整决策链与资产处置见 `../antipattern-candidates.md` #18。原第 8 集（空头支票）顺延为第 7 集。
> 2026-08-23 决议：第 3、4 集互换（工具集前置，成"挂什么→治别人的→别自己造坏的"三部曲；文档集后置接会话篇）。**spike 与交付物目录一律沿用历史集号不改名**（ep03-* 属现第 4 集、ep04-* 属现第 3 集），映射见各集设计档头部。

## 每集设计档模板（四区）

1. **既定事实**：反模式定义 / 原理一句 / 实测结论 / 调研三档（水文·值钱·威胁）——这些是地基，讨论不推翻，除非有新实测。
2. **待定问题**：靶心 / demo 形态 / 干货与交付物 / 诚实边界——逐条讨论，定一条落一条。
3. **讨论记录**：追加式决议日志（日期 + 决议 + 理由）。
4. **交付物清单**：本集最终要带走的文件/配置/卡片。

## 每集评估维度（2026-08-20 定，讨论/定稿每集必过）

全部围绕一个目标：**观众 5 分钟学到干货，带走一个可落地的实践**。用户亲定前三条，其余为扩展：

1. **行业证据**：有没有一手研究、官方背书或公开翻车案例撑这个反模式？（营销号口径不算数）
2. **犯病率**：目标观众真的常犯吗？社区讨论密度 / 身边观察佐证？不常犯的病不值得一集。
3. **整改简单度**：改进动作当天能落地吗？理想形态 = 一条命令 / 一个配置 / 一张清单。
4. **信息差**：讲了不水——观众大概率还不知道的部分是什么？已是常识的一句话带过，不当新知讲。
5. **可实拍可验证**：翻车和解法的数字有没有 spike 撑？拍不出来的必须有替代方案（口播/截图/换后端），不许编。
6. **诚实边界**：哪些话不能说满？失灵面、分布漂移、版本门槛、模型生成内容——措辞逐条守得住吗？
7. **交付物带走性**：看完观众手里多了什么？清单 / 判据卡 / 模板 / 脚本，必须具体。
8. **容量与边界**：一集只装一个反模式 + 一组动作；装不下的砍掉或挪集，不提前消耗后续集次的弹药。

## 纪律（从课程设计过程沉淀）

- 状态变更必须显式：定稿与否以用户直接回答是/否为准，应答不算授权（"好"≠确认）。
- 标题党不吹牛：集名只给效果；数字开拍实拍对上才算数。
- 每集必须讲诚实边界。
- 引用版本/功能开拍当月复核（产品快速迭代期）。
- **原理是脊柱，不是过场**（2026-08-21 用户）：原理一句仍是铁律（不展开理论），但每集按"原理 → 案例即原理的表现 → 解法即原理的应用 → 收尾可迁移判据"组织；机械执行比不教更糟——判据要能让观众面对新型情况自己推。
