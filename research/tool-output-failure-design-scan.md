# 失败输出的形态设计：行业水位调研（EP02 候选新支柱存废判定）

调研日期：2026-08-20
调研目的：实测推翻"verbose 输出压垮 agent"原靶心后（见 research/tool-output-industry-consensus.md），评估候选新支柱「失败输出的形态设计」的行业水位——即教用户把自建脚本/测试命令设计成"成功路径零输出或一行；失败时摘要前置（放头部）、完整失败详情控制在 harness 截断线（~10KB）以内"。

判定口径（沿用 tool-output-industry-consensus.md）：
- **常识** = 官方文档/头部工程博客反复论述 + 社区工具生态已成规模
- **有人讲** = 有公开 issue/博客/工具专门讨论，但未形成普遍认知
- **几乎没人讲** = 找不到直接或只有擦边的公开讨论

---

## Q1 "quiet on success, verbose on failure" 作为源头设计模式，有没有公开论述？

**判定：原则层面是常识（Unix 经典教条）；但作为"教脚本/CI 命令作者这么设计"的可操作处方，只是有人讲。**

原则层（常识侧）证据：

- ESR《The Art of Unix Programming》的 Rule of Silence（"When a program has nothing surprising to say, it should say nothing"）和同书的 Rule of Repair（"When you must fail, fail noisily and as soon as possible"）——这两条拼起来就是本支柱的上半句，且是被引用了二十多年的 Unix 头号教条。http://www.faqs.org/docs/artu/ch01s06.html ；https://wiki.c2.com/?RuleOfSilence
- Automake silent-rules 的官方文档明确给出设计动机：安静输出"让编译警告更容易被看到"——即"噪音遮蔽信号"的论述在构建工具官方文档里就有。https://www.gnu.org/software/automake/manual/1.16.5/automake.html ；https://autotools.info/automake/silent.html
- 工具实体化的先例成规模：moreutils 的 `chronic`（"runs a command quietly unless it fails"）和 cronic（cron 输出只在失败时发信）是发行版仓库里的常备工具。https://manpages.debian.org/testing/moreutils/chronic.1.en.html ；https://github.com/justincase/cronic

处方层（有人讲侧）证据与缺口：

- 能找到的处方都是"消费端"的（用 chronic 包一层、给命令加 quiet flag），或者是 CLI 通用规范（clig.dev 的 stdout/stderr 分工、-q/--verbose 纪律）。"你自己写的构建/测试脚本，应该默认安静、失败才吐详情"这种直接面向脚本作者的设计倡导，没有形成有名字的实践流派——它散落在各个工具的默认行为里（见 Q3），没人把它抽出来当成一条可教的原则讲。

**小结**：上半句（成功要安静）讲了会水，是 50 年的老常识；课程价值全在下半句的形态设计（Q2/Q4）。

## Q2 "把失败摘要放输出头部 / 让关键信息躲过 agent 截断"，有人教过吗？

**判定：有人讲（MCP 工具作者侧有明确处方）；绑定到"Bash 命令输出被 harness 保头截断、所以脚本摘要要前置"的处方几乎没人讲。**

- **最接近的处方**：Gabriel Anhaia《Tool-Result Truncation: The Silent Bug That Makes Agents Lie》明确提出"summarize at the tool layer"——工具返回 `summary`（含 `first_failures` 前 5 条失败）+ 完整报告句柄，动机正是截断会让模型"看着半截 JSON 自信作答"。这是"摘要前置以对抗截断"的完整处方，但对象是 MCP 工具作者，不是 shell/测试命令作者；且他没有点名"保头截断"这个方向性。https://dev.to/gabrielanhaia/tool-result-truncation-the-silent-bug-that-makes-agents-lie-3epe
- **官方只到"高信号、要分页"为止**：Anthropic《Writing tools for agents》教 concise/detailed 响应格式、"只返回 high-signal 信息"，但通篇没有"顺序"概念——没有"把最重要的放前面因为后面可能被砍掉"。https://www.anthropic.com/engineering/writing-tools-for-agents
- **agent-friendly CLI 写作流派已成形但同样不讲顺序**：justin.poehnelt.com 的 agents-first CLI 设计文（field masks、NDJSON 分页、context window discipline）、Speakeasy 的 agent-friendly CLI 改造文，都在讲"减少输出量"，没人讲"输出内部的排布顺序决定截断后剩什么"。https://justin.poehnelt.com/posts/rewrite-your-cli-for-ai-agents/ ；https://www.speakeasy.com/blog/engineering-agent-friendly-cli/
- 上一轮调研已确认：讨论 head/tail 截断丢信号的（#36596、#64577、Token-Saver FAQ）都是抱怨截断或工具作者自救，没有沉淀出"所以你写脚本时摘要放头部"的用户处方。

**小结**：这是一个"零件齐全、没人拼给用户看"的点，水位低，是四问里第二值钱的。

## Q3 测试框架/CI 工具侧有原生支持此模式的先例吗？

**判定：常识。"抑制成功输出、只展示失败输出"是主流测试/构建工具的默认行为；"失败摘要与详细日志分离"也是成熟实践。但要注意：主流工具把摘要放尾部，恰与 agent 截断方向相冲。**

- **成功静默是默认行为，不是可选项**：
  - cargo test 默认捕获通过测试的 stdout/stderr，只在失败时展示（Rust 官方书明确写）。https://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/book/second-edition/ch11-02-running-tests.html
  - nextest 默认"hide test output for passing tests, show them for failing tests"。https://nexte.st/docs/reporting/
  - CTest 有专门的 `--output-on-failure` 及同名环境变量。https://cmake.org/cmake/help/latest/manual/ctest.1.html
  - Bazel `--test_output=errors`（只显示失败测试输出）是默认；`--test_summary=terse`（只列失败/flaky 测试）也是默认；详细日志分离到 test.log。https://bazel.build/docs/user-manual ；https://blog.engflow.com/2023/09/18/bazel-testing-tips/
  - ninja 官方手册："Command output is always buffered … when a command fails we can print its failure output next to the full command line"——成功命令的输出根本不出现。https://ninja-build.org/manual.html
- **摘要与详细日志分离也是成熟机制**：TAP 协议把机器可读的结果行（ok/not ok）与诊断信息分层、由 harness 聚合（https://testanything.org/tap-version-14-specification.html ）；GitHub Actions 提供 `::group::` 折叠日志、`::error::` annotation 把失败信息提到 UI 层、`$GITHUB_STEP_SUMMARY` 专门放摘要（https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-commands ）。
- **关键的反方向事实**：这套行业惯例是为"人类在 CI 网页里滚屏"设计的，所以失败摘要几乎都在**尾部**（pytest `-r` 的 short test summary info、cargo 的 `failures:` 列表、Bazel 的 summary 都在末尾）。即工具侧解决了"量"的问题（成功静默），但"摘要位置"的惯例方向与 harness 保头截断正好相反——这个错位没人讨论（接 Q4）。

## Q4 "失败那次输出最长"这个结构性观察有人命名过吗？与保头截断拼起来的完整故障模式呢？

**判定：几乎没人讲。不对称性本身有古老命名（Rule of Repair），"失败输出天然最长"作为结构性观察无人命名；与保头截断拼合的完整故障模式无公开叙述。**

- 不对称性最老的表述还是 ESR 的 Rule of Repair（"fail noisily"）和 chronic/cronic 的存在本身——它们承认了"失败才需要大量输出"，但没人把这句话反过来说成"所以失败那次的输出永远是最大的一次"。
- 人类侧的抱怨存在：cypress-io/github-action #542 "log output is extremely long … makes it difficult to scroll to the actual test failures"——是"人滚屏找不到失败"的抱怨，不是截断丢摘要。https://github.com/cypress-io/github-action/issues/542
- 与 harness 截断的拼合：上一轮调研已确认"保头截断恰好杀死尾部摘要"无公开叙述（tool-output-industry-consensus.md Q5），本轮未找到任何新材料改变这个结论。Q3 的反方向事实（行业惯例把摘要放尾部）进一步证明：这个完整故障模式（失败输出最长 → 摘要在尾部 → 保头截断杀摘要 → 留下最没信息量的 progress 头）是四个各自有据的事实拼出来的，公开语料里没人拼过。

---

## 对课程定位的启示

- **Q1 上半句（成功要安静）不能当卖点**：Rule of Silence、chronic、cargo test 默认行为——观众但凡写过 CI 就知道。讲了就是"Unix 哲学复读机"。
- **Q3 的工具清单可以用来"捧杀"**：快速列举 cargo/nextest/Bazel/ninja/CTest 全都内置了 output-on-failure，建立"这个问题业界早就解决了"的预期——然后反转：它们全是为人类滚屏设计的，摘要在尾部；你的 agent 的 harness 保头截断，恰好把这份摘要看不见。Q3 的常识性恰恰是反转的弹药，不是要教的内容。
- **Q2 是可教的真空区**：把 Anhaia 在 MCP 工具侧的处方（summary-first + 详情句柄）平移到"自建脚本/Make 目标/npm script"上——"失败时第一行就是失败摘要，完整 traceback 收进 10KB 以内，超出的写文件给路径"——这个具体处方没有公开竞品。
- **Q4 提供叙事骨架**："成功一次输出一行，失败一次输出 500 行——所以被截断的永远是失败那次"——这个不对称性是 Rule of Repair 的暗面，没人命名过，课程可以命名。

## 水位结论：这个支柱能撑起一集 5 分钟反模式课吗？

**单独撑不起，但作为 EP02 修订版的"解法段"（绑定已有最强靶心"保头截断杀尾部摘要"）非常合适，足以让整集成形。**

理由：

- 支柱的前半（quiet on success）是常识，前半部分讲了必水——5 分钟独立成集意味着至少 2 分钟在讲观众已知的东西。
- 支柱真正有信息量的部分（摘要前置 + 控制失败详情在截断线内）是**处方**，不是故障模式。反模式课的结构是"先让观众疼，再给药"；疼的部分（保头截断杀尾部摘要、exit≠0 hook 缺席）在上一轮调研里已确认为几乎没人讲的真盲区，这个支柱恰好是那味药。
- 推荐结构：EP02 靶心不变（"防线在失败时缺席"，实测②③），本支柱作为收尾解法——"业界工具（Q3 清单）早就解决了成功静默，但它们的摘要位置惯例是为人类滚屏设计的，在 agent 的保头截断下恰好失效；你自己的脚本要反过来设计：失败摘要第一行、详情收进 10KB、超出落盘给路径"。这样 Q1/Q3 的常识各得一句带过，Q2/Q4 的低水位内容占据主体。
- 若坚持让本支柱独立成集，风险是：去掉实测②③的故障演示后，剩下的内容（Unix 哲学 + 工具默认行为 + 一条没有公开竞品的处方）深度不够 5 分钟，且"反模式"名不副实——它是一条设计原则，不是反模式。
