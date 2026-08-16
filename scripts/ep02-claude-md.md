# 第 2 期：约束一级——CLAUDE.md 管"在不在场"

> 时长 5 分钟 | 交付物：分层 rules 目录结构 + 模板仓库（含 promptfoo 冒烟配置，本期下发） | 前置：第 1 期（半衰期判据）

## 逐字稿

### 0:00–0:30 翻车导入

【口播】
先看一段真实运行。我把"别改 `generated/`"这条约定写在 `frontend/` 子目录的 CLAUDE.md 里——就像很多人习惯把规范放在对应模块旁边。让它重新生成桩代码，它干得又快又好——顺手把 `generated/` 里我手打的补丁全覆盖了。规则写了，任务翻了。问题出在哪？这条规则根本不在场：根目录的 CLAUDE.md 会话开始就加载，但子目录里那份是懒加载——任务全程没碰 `frontend/`，它就整场没进过上下文。它不是违抗我，是压根没看见我。

【画面】
录屏：左侧编辑器展示 `frontend/CLAUDE.md` 的位置，右侧终端跑任务。规则"不在场"用协议级证据呈现（2026-08-09 实测，布景 `spike/ep02-scope`，InstructionsLoaded 探针日志）：根 CLAUDE.md 的加载记录是 `session_start`，子目录那份是 `nested_traversal`——任务没进那个目录，日志里就永远没有它。翻车后果用策展 transcript，字幕标注"选自 N 次运行中的典型样本"。B 方案见文末。

### 0:30–1:30 机制一：加载机制——规则失效是静默的

【口播】
CLAUDE.md 体系的第一个属性是加载机制：规则不是写了就生效。先纠正一个普遍错觉——你的规则不是只有根目录那一份就万事大吉：子目录里的 CLAUDE.md 是**懒加载**的，任务没碰那个目录，它就整场不在场。日志卡上看得清清楚楚：根目录那份的加载记录是 session_start，子目录那份是 nested_traversal——没进目录，就没有这条记录。而真正麻烦的是第二件事：**不管哪种没加载，都没有任何提示**。没有报错，没有警告，agent 一切如常，只是那条规则从来不在场。你发现它的方式，是出事故之后回去翻。所以这期给你一个查证工具：InstructionsLoaded——每份 CLAUDE.md 被注入上下文都会触发这个事件，挂个 hook 记下来，"它到底看没看见"就从玄学变成一行日志。需要按路径更精细的控制？有 `.claude/rules/` 的 paths 作用域，进阶选项，语法在卡上自己看。

【画面】
左：加载日志卡——根 CLAUDE.md `session_start` vs 子目录 `nested_traversal`，以及"任务没进目录→日志零记录"的对比（2026-08-09 实测，`spike/ep02-scope/`）。右：层级/import/rules paths 语法速查字幕卡（观众自读，不占口播；rules 标注"进阶可选"）。录屏：把约定从子目录 CLAUDE.md 挪到正确位置（根 CLAUDE.md 或配对 paths 的 rules），跑一次任务，探针日志出现对应条目。字幕："本期动作：把'别改 generated/'放到会被加载的位置 + 挂上加载探针"。

### 1:30–2:00 微段：先泼冷水——这些字可能不值得写

【口播】
配好作用域之前，先泼一盆冷水：你 CLAUDE.md 里写的那些，可能本来就不值得写。有篇系统性评测，结论不好看：context files 总体不提升任务成功率，推理成本反而更高——数字在屏幕上，自己看。官方自己也有一份"不要写"清单。所以判据就一句话：删掉这条，它会犯错吗？不会，就删。多写不是严谨，是稀释。一份 294 行的臃肿样例在模板仓库，今天的作业就是拿它开刀——视频里不演示，省下时间看更重要的东西。

【画面】
字幕卡（静止 4 秒，观众自读）：arXiv 2602.11988 结论三行——总体不提升成功率 / 推理成本平均 +20% / 仅"非标准实践"类指令有正收益。截图：官方 best practices 的 Exclude 清单三类。角标：臃肿样例 `template-repo/ep02/fixtures/bloated-CLAUDE.md`（作业素材）。

### 2:00–3:30 机制二：强制力——建议，不是命令

【口播】
规则瘦身了、作用域也配对了，它能乖乖听话吗？看实测。这是第二个属性：CLAUDE.md 是建议性的，官方原话对照在屏幕上。第一期说过，模型是优化器，优化的是你给的信号而不是你的意图。同一个布景，我们三天各跑了 16 次：第一天 15 次违反，第二天 7 次，第三天 7 次——最日常的中性指令，不提文件不提规则，三天各违反 3 次、1 次、2 次。注意两件事：第一，无人施压它自己就会跑偏；第二，违反率每天都在漂移。概率不为零和确定执行之间的差值，就是这么具体。所以问题不是"这次会不会"，是后果你承受得起吗？"别改 generated/"被无视，大不了重新生成；但如果背后是数据迁移、计费逻辑、合规红线呢？先把分工说清楚：这一级管"在不在场"，管不了"听不听话"。承受不起的那条规则，下一期升级成 PreToolUse hook，确定性拦截——同一条规则，一级一级变硬。

【画面】
录屏片段（自包含取证版）：中性指令 → agent 直改 generated/ → `git diff` 红框。数据卡（观众自读）：同一任务 4 种说法 ×16 次 × 三天：15/16、7/16、7/16 违反；时间压力组三天 11/12 最稳定；Day3 逐条录屏 + Day1/2 判定记录 `spike/captures/evidence/ep02/`。字幕卡：官方引语对照——CLAUDE.md 指令是 advisory；hooks 是确定性的，"在固定的生命周期点作为 shell 命令执行，无论模型自己怎么决定都生效"。结尾字幕卡："第 3 期：同一条规则 → PreToolUse hook"。

### 3:30–4:00 半衰期判据

【口播】
老问题：模型再强一倍，这期还成立吗？分两半。加载机制——层级、import、paths 作用域——是产品设计，官方随时会改，有半衰期；但"约束要分层级生效、作用域决定在场"这个判断不会变，哪家工具都绕不开。强制力这半更稳：模型变强会压低无视的概率，但概率不为零和确定执行之间的差值永远存在。要学的不是背配置，是判断力：这条约束该放哪一级。

【画面】
字幕卡：判据三行——"加载机制会变 / 分层生效不变 / 建议与强制的差值永远存在"。

### 4:00–5:00 交付物、作业与对应位置

【口播】
带走的东西在模板仓库 ep02 目录：分层 rules 结构、检查脚本，外加 promptfoo 冒烟配置——从这期起你的 harness 改动都可度量，第七期展开。作业十分钟：一，拿仓库里那份 294 行的臃肿 CLAUDE.md 开刀，用"删掉这条它会犯错吗"过一遍，砍到 50 行上下；二，把"别改 generated/"迁进 rules 配对 paths；三，跑 `check-rules.sh`，绿了才算完成。Cursor 和 Codex 的对应位置在这张卡上。这是 Claude Code 主修课，但"分层加载"和"建议不是强制"两个判断，三家通用。下期见。

【画面】
仓库目录树截图 + 作业三步字幕卡（修剪 bloated-CLAUDE.md / 迁移配 paths / 跑 check-rules.sh 全 PASS）+ Cursor/Codex 对应位置卡（20 秒，见文末）。

## Demo 详细步骤

### Demo 1：翻车导入（0:00–0:30，实拍，2026-08-09 改版）

- 布景 `spike/ep02-accident`：约定写在 `frontend/CLAUDE.md`（"generated/ 里有手打补丁，重新生成前先备份"）；`generated/stub.ts` 含 `LEGACY_ALIAS` 手工补丁；探针 hook 挂载。
- 连拍三拍：① `cat frontend/CLAUDE.md` 亮约定位置；② 任务"proto 加了字段，跑 scripts/regen.sh 重新生成"→ agent 照跑（它无从知道那份约定）；③ `git diff` 补丁消失 + `grep frontend` 探针日志零记录——**没加载**与**翻车**同屏。录制脚本 `spike/prod/record-ep02-accident.sh`；选条条件：补丁被覆盖且日志零记录（若 agent 注意到文件内注释而保守处理，重录）。
- **B 方案**：样本不典型时退回静态帧 + 旁白（"它试图覆盖 generated/，里面是我手打的补丁"），探针日志对比卡是确定性部分，必保留。

### Demo 2：动作——修正作用域 + 挂加载探针（0:30–1:30，确定性）

- 录屏：把约定从子目录 CLAUDE.md 挪到根 CLAUDE.md（或改成配对 paths 的 rules）；再把 `log-instructions.sh` 探针挂进 settings.json 的 InstructionsLoaded 事件。
- 注入证据不走 `/context`（UI 易变）：用探针日志对比——挪动前后各跑一次任务，日志从"只有 session_start"变成出现 `nested_traversal` / 目标文件条目。确定性，已实测（`spike/ep02-scope/`，2026-08-09：根 CLAUDE.md 为 session_start，子目录为 nested_traversal）。
- 探针交付物在模板仓库 `ep02/.claude/hooks/log-instructions.sh` + settings 片段。

### ~~Demo 3：手工修剪~~（2026-08-09 评审撤拍，降级为作业素材）

- 撤拍理由（评审结论）：与主线逻辑冲突——主线说"规则是弱信号"，修剪 demo 却在教"怎么把弱信号写精致"；arXiv 证据的诚实结论是"少写"，一张证据卡 30 秒足够，不配占 90 秒 demo。/doctor 此前已撤拍。
- 素材去向：`ep02/fixtures/bloated-CLAUDE.md`（294 行）保留为**作业素材**——观众自己动手修剪（模拟验证：删四节后剩 46 行）。

### 失效现场（2:00–3:30，实测素材已在手）

- 用三天三组采样里中性指令的违反录屏（`spike/captures/evidence/ep02/D1/D4` 等，自包含：指令横幅→行为→git diff），数据卡写三天三组（15/16、7/16、7/16；时间压力组 11/12）。
- B 方案不变：样本全守规矩时改旁白"概率不为零不等于这次会发生——你要回答的是'承受得起吗'"。

## 交付物文件内容

模板仓库 `ep02/` 目录结构：

```
ep02/
├── CLAUDE.md                    # 瘦身骨架，<200 行，只含幸存三类
├── .claude/
│   ├── rules/                   # 进阶可选：需要按路径精细控制时用（paths 作用域）
│   │   ├── generated.md
│   │   ├── api.md
│   │   └── testing.md
│   └── hooks/
│       └── log-instructions.sh  # 加载探针（InstructionsLoaded → 追加日志）
├── scripts/
│   └── check-rules.sh           # 作业外部判据脚本
└── evals/
    └── promptfoo.smoke.yaml     # 冒烟配置（第 7 期展开）
```

`.claude/rules/generated.md` 完整内容：

```markdown
---
paths: ["generated/**", "proto/**", "scripts/**"]
---

# generated/ 目录保护

`generated/` 下全部是 `scripts/gen.sh` 从 proto 生成的代码，禁止直接编辑。
需要改动时：改 `proto/` 下的源文件，然后运行 `scripts/gen.sh` 重新生成。
理由：该目录历史上被手改后与 proto 漂移，导致线上序列化不兼容（2025-11 事故）。
```

`CLAUDE.md` 瘦身骨架：

```markdown
# 项目指令

## 坑（读代码看不出来的）
- 迁移必须走 tools/migrate.sh，直连数据库绕过审计日志
- vendor/ 里的 foo 库打过本地补丁，升级前先看 vendor/FOO.PATCH.md

## 理由
- 不用 ORM：2019 年查询性能事故后的决定，见 docs/adr/003.md

## 非默认约定
- 错误码用项目内 errors.go 注册，不用 sentinel error
- 测试数据库端口 5433，不是默认 5432

@AGENTS.md
```

`check-rules.sh` 判据（摘要）：

1. `CLAUDE.md` ≤ 200 行；
2. `CLAUDE.md` 不含目录树 / 依赖列表模式（启发式 grep：`├──`、`npm install`、版本号清单）；
3. `.claude/rules/` 下每个 `.md` 文件有合法 `paths:` frontmatter 且 glob 非空；
4. 至少一条规则文件的 paths 与仓库实际目录匹配。

## 作业（≤10 分钟，带外部判据）

1. 把模板仓库 `ep02/` 拷到你的项目根（或对照改造）。
2. 用"删掉这条它会犯错吗"修剪你自己的 `CLAUDE.md`；目录树、依赖列表、自明条目全删。
3. 把至少一条真实约束（建议就是"别改某某目录"这类）迁进 `.claude/rules/`，paths 配到该约束真正涉及的目录。
4. **外部判据**：运行 `bash scripts/check-rules.sh`，全部 PASS 才算完成；学有余力再跑 `promptfoo eval -c evals/promptfoo.smoke.yaml` 确认冒烟通过（为第 7 期打底）。

## Cursor / Codex 对应位置卡片（20 秒文案）

【口播】
用 Cursor 的同事：对应物是 `.cursor/rules/` 的 `.mdc` 文件，触发分四档、比 Claude Code 更细，档位在卡上——配错效果一样：规则不在场。用 Codex 的同事：对应物是 AGENTS.md，逐级拼接、越近优先级越高，默认上限 32KiB——超了等于没写。Claude Code 不原生读 AGENTS.md，`@AGENTS.md` 一行导入解决。

【画面】
三栏对照卡：CLAUDE.md + rules paths / `.cursor/rules/*.mdc` 四型触发（Always Apply / 按 globs 自动附加 / 按描述 agent 自取 / 手动 @ 提及；团队规则 1.7 起仪表盘下发）/ AGENTS.md 逐级拼接 + 32KiB 上限。

## 事实核对清单

| 事实声明 | 来源 | 状态 |
|---|---|---|
| 四层加载：managed policy → user → project → local；cwd 向上拼接不覆盖；子目录懒加载 | docs.claude.com/en/docs/claude-code/memory（调研报告 §1.1） | ✅ 一手 |
| import 语法 `@path`，递归最深 4 跳；`@AGENTS.md` 复用 | 同上（报告 §1.1） | ✅ 一手 |
| `.claude/rules/*.md` frontmatter `paths:` 支持 glob，只在读相关文件时注入 | 同上（报告 §1.1） | ✅ 一手 |
| 官方建议每个文件 <200 行 | 同上（报告 §1.1） | ✅ 一手 |
| "CLAUDE.md is advisory, hooks are deterministic"（官方原话）；hooks"execute as shell commands at fixed lifecycle events…regardless of what Claude decides" | anthropic.com/engineering/claude-code-best-practices + memory 文档（报告 §1.1/§4.4） | ✅ 一手 |
| 官方 Exclude 清单：模型读代码能知道的 / 标准语言惯例 / "write clean code"类自明条目 | claude-code-best-practices（报告 §7.1-4） | ✅ 一手 |
| 删减判据"删掉这条会导致 Claude 犯错吗？否则删掉" | 同上（报告 §4.4） | ✅ 一手 |
| arXiv 2602.11988：context files 总体不提升成功率、推理成本 +20%、仅"非标准实践"类有正收益 | arxiv.org/abs/2602.11988（报告 §7.1-4，摘要验证） | ✅ 摘要已验，⚠️ 开拍前复读摘要确认表述 |
| `/doctor` 自动提议修剪 CLAUDE.md（v2.1.206） | 官方 CHANGELOG + 发布报道 | ✅ 特性存在性已核实；**决策：不拍**（2026-08-07）——版本门槛 + 提议内容模型生成 + 教学核心不依赖它。Demo 3 已改为手工按 Exclude 清单修剪（确定性，一条过） |
| OpenAI 百万行实验：~100 行 AGENTS.md 当地图，指向结构化 docs/ | openai.com/index/harness-engineering（报告 §4.6） | ✅ 一手 |
| "概率不为零"表述替代精确失效率数字 | 大纲事实纪律（禁用"5%"类无出处数字） | ✅ 已遵守 |
| Cursor rules：`.mdc`、四型触发、团队规则 1.7 起下发 | cursor.com/docs/rules（报告 §2.1） | ✅ 一手 |
| Codex AGENTS.md：全局→项目逐级拼接、越近优先级越高、默认 32KiB 上限 | developers.openai.com/codex/guides/agents-md（报告 §3.2） | ✅ 一手 |
| Claude Code 不原生读 AGENTS.md，需 `@AGENTS.md` | code.claude.com/docs/en/memory（报告 §3.1） | ✅ 一手 |
| 失效现场"12 次运行取典型样本" | 本课制作纪律（策展 transcript 必须标注） | ⚠️ 开拍前实跑，次数以实际为准更新字幕 |
