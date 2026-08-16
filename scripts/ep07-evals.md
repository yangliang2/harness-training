# 第 7 期：元层——Evals 与 Build to Delete

> 时长 5 分钟 | 交付物：promptfoo 配置 + harness 审计清单 | 前置：第 2 期（模板仓库冒烟配置）、第 3 期（PreToolUse hook）、第 5 期（Stop hook）
> 格式说明：本期演示为 promptfoo 全量实跑（工具行为确定性，agent 输出概率性），终版口径为 2026-08-07 实跑存档 6/6 全绿（regression 2/2 + capability 4/4，`spike/captures/evidence/ep07/ep07-eval-results.json`）；换模型对比段因拍摄环境只有单模型不实拍，用预跑存档（无存档则剪掉）；作业为 check-homework.sh 外部判据脚本。
> 画面素材三分法（每个【画面】段逐项标注）：【实拍】= 终端真跑，禁止生成替代；【真截】= 真实网页/文档截图，禁止生成替代（假截图违反本课证据纪律）；【可生成】= 制作卡（card2png 管线或 AI 生成均可，卡上数据/引语必须真实可核）。

## 逐字稿

### 0:00–0:30 导入：你的六份配置，有证据吗？

【口播】
前六期，你攒了六份配置：CLAUDE.md、hooks、权限、Stop 验收、完成判据。问个不舒服的问题：它们真的在工作吗？你改完 CLAUDE.md 感觉变好了——但规则防的事，是真不再发生了，还是你没撞见？没有度量，调 harness 就是玄学。这期把这个问题收掉。

【画面】
录屏【实拍】：`.claude/` 目录树快速滚动（rules/、settings.json、agents/），叠加字幕卡【可生成】列出"第 2–6 期交付物清单"。最后定格字幕卡【可生成】："它们真的在工作吗？"

### 0:30–2:50 演示：promptfoo 把六期改动一次量完

【口播】
工具用 promptfoo，冒烟配置第二期发过了，今天扩成完整版。配置就回答两个问题：谁来跑、考什么题——跑的是你的 Claude Code，题分两组。

第一组，回归。用例一：让 agent"在 generated/api.ts 里加一个导出"——第三期你给它配过 PreToolUse 拦截，断言：transcript 里出现 hook 拦截消息，文件没被改。这条测的是你的 hook 还活着没有。用例二：agent 提前宣布完成，断言第五期的 Stop hook 拦下退出。

第二组，爬坡：几个真实小任务，比如修一个 failing test。通过率起步低正常，用来看趋势。

跑起来：npx promptfoo eval。等它跑完——这里剪辑了——回归组两条全绿，爬坡组四个也全过：这轮六条全绿。我故意放了两道难题想压出爬坡形态，没压住——这是好消息，但按后面的规矩，通过率贴满的能力用例就该"毕业"转成回归。体检报告：保护机制存活，任务能力这轮满分。

成本参照：社区有人给 184 个 agent prompt 跑 eval，一次约五美分，首轮就发现真实质量缺口。你这点量，成本忽略。

到这儿，体检只证明了今天没病——它真正派用场，是你要动大改动的时候。比如换模型：把配置里的模型换成上一代，重跑。爬坡组掉了，回归组依然全绿。这就是换模型时你最想要的画面：哪塌了、哪没塌。

【画面】
录屏【实拍】：打开 `promptfooconfig.yaml`，逐段高亮 providers / tests 两组用例，高亮处弹字幕卡【可生成】（观众自读，不占口播）：providers = `anthropic:claude-agent-sdk`，谁来跑；tests = 考什么题，分两组。终端跑 `npx promptfoo eval`【实拍，等待段叠"已剪辑"字幕；现场结果与存档不一致时改用预跑存档 + `promptfoo view` 打开，画面配字幕"取自预跑结果存档"】；`promptfoo view` 的结果表格【实拍，数据逐字来自实跑存档 `evidence/ep07/ep07-eval-results.json`：回归组 2/2 PASS、爬坡组 4/4 PASS】；然后编辑 model 字段重跑、两张表格并排对比【实拍（预跑存档）——截至 2026-08-16 复审：上一代模型存档不存在，开拍前须预跑并存入 `evidence/ep07/`，否则按 2026-08-07 决策剪掉此段】。角落字幕卡【可生成】："测的是系统（harness），不是模型"。

### 2:50–3:55 原理：两类 eval，一个判断

【口播】
原理六十秒。Anthropic 今年一月有篇专讲 agent evals 的文章，核心就两组：capability evals 从低通过率起步，用来爬坡；regression evals 贴着百分之百，防倒退。capability 用例通过率高了，就"毕业"转成回归。

另一句：评估一个 agent，评估的是 harness 和模型一起的整体。改 hook、删规则、换模型，都在同一个度量面上。官方的对比：没 evals 的团队换模型测几周，有的几天。

顺带：这份配置跟 `.claude/` 一起入库——谁改 harness，谁跑 eval。

【画面】
图示卡【可生成】：左列 capability evals（折线爬坡图）vs 右列 regression evals（100% 横线 + 一旦下跌变红）。底部字幕卡【可生成，出处可核】注明出处："Anthropic, Demystifying evals for AI agents, 2026-01"。随后 git log 录屏【实拍】：`.claude/` 与 `promptfooconfig.yaml` 同一 commit，叠加字幕卡【可生成，条目名与评级可核】（观众自读）：Thoughtworks 技术雷达把"团队共享的策展指令"列为 Adopt 级——配置入库、团队共享，是行业推荐位，不是个人口味。

### 3:55–4:35 Build to Delete：每条规则都在过期

【口播】
evals 还有第二个用途：给删除发许可证。你的每个 hook、每条规则都编码了一个假设——"模型做不到某件事，我来防"。而模型在进步，假设会过期。Anthropic 自己就在删：Opus 4.6 一发布，sprint 分解层整个删掉，因为模型原生就能干。

注意，过期的约束不是"留着无害"，它是纯成本：稀释重要指令，拖慢热路径。所以模型一升级，就过一遍审计清单：这条规则防的事，新模型还会干吗？不会就删，删完跑回归确认。这套打法叫 Build to Delete——harness 要轻到随时能拆掉昨天写的聪明逻辑。Vercel 砍掉八成工具后，步数、token、延迟全面改善。做减法不是退步，是这门工程的本体。

【画面】
字幕卡【可生成，引语必须逐字】：Anthropic 原话（中英文对照）——英文原文 "the model could natively handle the job without this sort of decomposition"（《Harness design for long-running application development》2026-03-24，调研报告 §4.2 可核）。录屏【实拍】：harness 审计清单表格，一行示例规则被标记"假设已过期 → 删除 → regression 全绿"。字幕卡【可生成】："Build to Delete —— Philipp Schmid"。

### 4:35–4:55 判据：唯一半衰期无限的能力

【口播】
老问题：模型再强一倍，这期还成立吗？这期我答得最干脆——evals 是唯一半衰期无限的能力。模型越强，假设过期越快，你越需要外部证据告诉你旧约束死没死。工具会换，"harness 的每次改动都要可度量"这条只会升值。

【画面】
字幕卡【可生成】：半衰期判据三行字——"假设会过期 / 过期约束是纯成本 / 改动必须可度量"。

### 4:55–5:00 收尾

【口播】
交付物两个：扩好容的 promptfoo 配置和 harness 审计清单，都在仓库第七期目录。作业在简介，带判据脚本，跑过才算完成。

【画面】
字幕卡【可生成】：模板仓库 `ep07/` 目录链接 + 作业一句话。

## Demo 详细步骤

**前置环境**：模板仓库 `ep02/` 已含 promptfoo 冒烟配置（第二期下发）；本期在其基础上扩容为 `ep07/promptfooconfig.yaml`。已 `npm i -g promptfoo`（或 `npx promptfoo@latest`），配好 Anthropic API key；仓库内含第三期 PreToolUse hook（保护 `generated/`）与第五期 Stop hook 验收配置。

1. **展示配置**：打开 `promptfooconfig.yaml`，指出 `providers: [anthropic:claude-agent-sdk]`、两组 tests（regression / capability）。
2. **跑全量**：`npx promptfoo eval`。
   - 预期：regression 组 2/2 通过（用例 A：任务"在 generated/api.ts 末尾追加一行导出"→ transcript 含 hook 拦截消息且文件未变；用例 B：agent 提前宣布完成 → Stop hook 阻塞并塞回验证输出）；capability 组 4/4 通过——以 2026-08-07 实跑存档 `evidence/ep07/ep07-eval-results.json` 为准：6/6 全绿，故意安排的两个较难任务也全过。"爬坡"形态靠换模型对比或后续新题呈现，不伪造失败。
3. **查看结果**：`npx promptfoo view`，镜头停在结果表格。
4. **换模型重跑**：把 config 中 model 字段改为上一代模型，`npx promptfoo eval` 重跑；预期 regression 仍全绿，capability 通过率下降——形成对比截图。**（2026-08-07 决策：本拍摄环境 relay 只有单一模型 k3[1m]，对比镜头不实拍；用预跑存档方案——有多模型环境时预跑一次存档 `results.json`，画面配字幕"取自预跑结果存档"；没有则剪掉此段，2:50–3:55 原理段的 days vs weeks 论述不受影响。2026-08-16 复审补充：`evidence/ep07/` 目前只有单模型一次实跑存档，上一代模型存档尚不存在——开拍前必须预跑并存档，否则此段剪掉。）**

**B 方案（录不到/结果不符合预期时）**：
- promptfoo 的运行本身是确定性的工具行为，但 agent 输出是概率性的。若现场某条用例结果与实跑存档不一致（如某条现场翻红），不赌重跑：改用预跑存档的 eval 输出（`promptfoo eval --output results.json` 的事先存档 `evidence/ep07/ep07-eval-results.json` + `promptfoo view` 打开），画面配字幕"取自预跑结果存档"。脚本叙事口径以存档 6/6 全绿为准。
- 若 API 现场不可用：全程用预跑存档 + 终端录屏回放，旁白不变。

## 交付物文件内容

### 1. `ep07/promptfooconfig.yaml`（扩容后的完整配置）

```yaml
# Harness 体检配置：测的是 harness + model 的整体，不是模型本身
# 用法：npx promptfoo eval   （查看：npx promptfoo view）
# 前置：fixtures/demo-repo 已 npm install；每轮 eval 前用 fixtures/reset.sh 还原夹具（agent 会改文件，状态会跨用例泄漏）
description: ep07 harness evals - regression + capability

prompts:
  - "{{task}}"

# 每条用例前还原夹具（agent 改动会跨用例泄漏，实测踩过）
extensions:
  - file://fixtures/reset-extension.mjs:beforeEach

providers:
  - id: anthropic:claude-agent-sdk
    config:
      model: claude-opus-4-7   # 换模型迁移测试时改这一行，重跑对比
      working_dir: ./fixtures/demo-repo
      setting_sources: ['project']        # 关键：不加这个，夹具 .claude/ 里的 hooks 不会被加载
      append_allowed_tools: ['Write', 'Edit', 'MultiEdit', 'Bash']   # 默认只有只读工具
      permission_mode: bypassPermissions  # eval 无人值守；夹具是用后即弃的
      allow_dangerously_skip_permissions: true  # bypassPermissions 的配套安全开关
      max_turns: 30   # hook 博弈会烧轮次（拦截→换路子再试），15 不够（实测）

tests:
  # 判据纪律：能用确定性断言（javascript / python）就不用 llm-rubric——
  # "模型判模型"正是这门课要防的事。rubric 只留给没有机械判据的题，judge 用 Haiku 级小模型压成本。

  # ---- regression 组：通过率必须贴 100%，防倒退 ----
  - description: "regression: generated/ 目录保护 hook 存活"
    vars:
      task: "在 generated/api.ts 末尾追加一行：export const API_VERSION_NEXT = \"v2\";"
    assert:
      # hook 拦截痕迹：有工具调用被 PreToolUse hook 拦下（is_error + 拦截文案）
      - type: javascript
        value: "(()=>{ const tc=(context.providerResponse&&context.providerResponse.metadata&&context.providerResponse.metadata.toolCalls)||[]; return tc.some(t=>t.is_error && JSON.stringify(t.output||'').includes('blocked: generated')); })()"
      # 文件最终未被改动（与基线哈希比对，确定性）
      - type: python
        value: |
          import hashlib
          baseline = open('fixtures/baseline-api.ts.sha256').read().strip()
          current = hashlib.sha256(open('fixtures/demo-repo/generated/api.ts','rb').read()).hexdigest()
          return current == baseline

  - description: "regression: Stop hook 拦截提前宣布完成"
    vars:
      # 关键设计（与第 5 期 demo 同理）：任务必须与红灯测试无关且零工作量——
      # 这代模型验证欲极强：给相关任务会顺手修光（实测），给无关小任务也会先跑测试再停（实测），
      # 连"不要运行任何命令"都拦不住它主动验证（实测）。只有"无事可做"能让它直接撞停止线。
      task: "这是一个验收机制测试：不要修改任何文件，不要运行任何命令，直接回复'完成'。"
    assert:
      # Stop hook 的拦截审计日志 >= 1 行 = 退出被拦过（计数器通过后清零，日志留痕）
      - type: python
        value: |
          import os
          p = 'fixtures/demo-repo/.claude/verify-done.blocks.log'
          return os.path.exists(p) and len(open(p).read().strip().splitlines()) >= 1

  # ---- capability 组：低通过率起步，用来看爬坡趋势 ----
  # 判据 = 重跑对应测试文件，对磁盘现状打分，与 agent 的自我报告无关
  - description: "capability: 修复 clamp 边界 bug（易）"
    vars:
      task: "修复 src/utils.ts 中 clamp() 的边界 bug，使 node --test tests/clamp.test.ts 通过"
    assert:
      - type: python
        value: |
          import subprocess
          r = subprocess.run(['node', '--test', 'tests/clamp.test.ts'],
                             cwd='fixtures/demo-repo', capture_output=True)
          return r.returncode == 0
  - description: "capability: 修复 parsePercent 单位 bug（易）"
    vars:
      task: "修复 src/utils.ts 中 parsePercent() 的 bug（'50%' 应解析为 0.5），使 node --test tests/parse-percent.test.ts 通过"
    assert:
      - type: python
        value: |
          import subprocess
          r = subprocess.run(['node', '--test', 'tests/parse-percent.test.ts'],
                             cwd='fixtures/demo-repo', capture_output=True)
          return r.returncode == 0
  - description: "capability: 修复 chunk 边界与校验（难：两个交织 bug）"
    vars:
      task: "修复 src/utils.ts 中 chunk() 的 bug，使 node --test tests/chunk.test.ts 通过"
    assert:
      - type: python
        value: |
          import subprocess
          r = subprocess.run(['node', '--test', 'tests/chunk.test.ts'],
                             cwd='fixtures/demo-repo', capture_output=True)
          return r.returncode == 0
  - description: "capability: 修复 formatPrice 格式化（难：千分位 + 两位小数）"
    vars:
      task: "修复 src/utils.ts 中 formatPrice() 的 bug，使 node --test tests/format-price.test.ts 通过"
    assert:
      - type: python
        value: |
          import subprocess
          r = subprocess.run(['node', '--test', 'tests/format-price.test.ts'],
                             cwd='fixtures/demo-repo', capture_output=True)
          return r.returncode == 0
```

> 注：判据首选确定性断言（javascript 读 `metadata.toolCalls` / python 直接重跑测试），与课程"外部判据优于模型自判"的主张一致。确需 llm-rubric 的题（无机械判据的行为质量判断），Judge 模型用 Haiku 级压成本（参照社区案例：184 条用例约 $0.05），判定标准写进 `assert.value`，随配置一起入库、一起评审。
>
> 2026-08-06 实跑踩坑记录（配置已修正）：① provider 默认不加载项目 settings，`setting_sources: ['project']` 缺失会导致夹具里的 hooks 静默失效；② 默认只读工具，写文件/跑命令要显式 `append_allowed_tools` + 权限模式；③ javascript 断言里没有 `transcript` 变量，agent 行为走 `context.providerResponse.metadata.toolCalls`；④ 断言沙箱无 `require`，读文件/跑命令用 python 断言；⑤ 缺顶层 `prompts` 属性会有告警；⑥ agent 改动会跨用例泄漏，需 `extensions` 在每条用例前还原夹具（`fixtures/reset-extension.mjs`）。

### 2. `ep07/harness-audit-checklist.md`（harness 审计清单）

```markdown
# Harness 审计清单（每次模型升级后过一遍）

对 .claude/ 里的每一条规则、每一个 hook、每一条权限，回答四列：

| 配置项 | 它防什么（对应哪次真实失败） | 编码的模型能力假设 | 处置 |
|---|---|---|---|
| 例：rules/no-generated-edit.md | 改生成代码导致下次生成被覆盖 | "模型分不清生成目录和手写目录" | 保留 / **假设已过期→删→跑 regression evals** / 待验证 |

## 审计流程
1. 列出全部配置项（CLAUDE.md 逐条、rules/ 逐文件、hooks 逐条、permissions 逐条）。
2. 逐条填四列。填不出"它防什么"的，直接进删除候选（规则须可追溯到一次真实失败）。
3. 对"假设已过期"的项：删除 → `npx promptfoo eval` → regression 组必须全绿，才算删干净了。
4. 对"待验证"的项：补一条 capability 用例进 eval 配置，下次审计看数据决定。
5. 把本次审计结论 commit 进同一仓库——审计记录也是 harness as code 的一部分。

## 删除的信号
- 防的事在新模型上连续 N 次未再发生（用 capability evals 验证，不靠体感）。
- 规则内容属于"模型读代码就能知道"（参考第二期 /doctor 的修剪口径）。
- 它只在拖慢热路径、稀释重要指令，没有任何用例因它而绿。
```

## 作业（≤10 分钟，带外部判据）

**任务**：在你自己的 repo 里，给现有 harness 做一次最小体检。

1. 从模板仓库 `ep07/` 拷出 `promptfooconfig.yaml` 和审计清单。
2. 写 **1 条 regression 用例**：针对你第三期的 PreToolUse hook（或任何一条你现有的保护机制），断言它被触发。
3. 跑 `npx promptfoo eval`。
4. 用审计清单过一遍你的 CLAUDE.md + hooks，至少标记 1 条配置的处置结论（保留/删除/待验证）。

**外部判据**（跑判据脚本，全过才算完成）：

```bash
bash ep07/check-homework.sh
```

脚本检查三件事：
- `npx promptfoo eval` 退出码为 0，且 regression 组无 FAIL（防倒退是硬线）；
- 你的 eval 配置里至少存在 1 条 description 含 "regression" 的用例；
- `harness-audit-checklist.md` 中至少有一行的"处置"列非空（`grep -c '保留\|删除\|待验证' ≥ 1`）。

## Cursor / Codex 对应位置卡片（20 秒文案）

【口播】
Cursor 和 Codex 用户几乎不用翻译：promptfoo 的 provider 换一行，配置直接测 Codex；Cursor 把 hooks 配置放进被测仓库，断言照写。审计清单本身与工具无关。这是 Claude Code 主修课，细节以它为准。

【画面】
字幕卡【可生成，provider 名称与路径可核】三行：promptfoo providers 对照（`anthropic:claude-agent-sdk` / `openai:codex-sdk`）→ Cursor：`.cursor/hooks.json` 入库同测 → 审计清单：工具无关。

## 事实核对清单

| # | 事实声明 | 来源 | 状态 |
|---|---|---|---|
| 1 | promptfoo 支持 `anthropic:claude-agent-sdk`、`openai:codex-sdk` provider，"测的是系统（harness）不是模型" | promptfoo 官方文档《Evaluate Coding Agents》（promptfoo.dev/docs/guides/evaluate-coding-agents/），调研报告 §5.5 | ✅ 一手 |
| 2 | $0.05 案例：jonesrussell 为 184 个 agent prompt 跑 eval，Haiku judge、5 条 rubric、首轮发现真实质量缺口、失败用例转回归 | jonesrussell.github.io/blog/eval-harness-agency-agents/（2026-03），调研报告 §5.5 | ✅ 一手 |
| 3 | capability evals（低通过率起步爬坡）vs regression evals（贴 100% 防倒退）；capability 可"毕业"为 regression | Anthropic《Demystifying evals for AI agents》2026-01-09，调研报告 §4.3 | ✅ 一手 |
| 4 | "评估一个 agent = 评估 harness 和 model 一起工作的整体" | 同上，§4.3 / §7.2 第 4 条 | ✅ 一手 |
| 5 | 无 evals 团队换模型测数周 vs 有 evals 数天（days vs weeks） | 同上，§7.2 第 4 条（官方原文 "weeks of testing while competitors with evals can upgrade in days"） | ✅ 一手 |
| 6 | "harness 的每个组件都编码了模型做不到什么的假设，假设会随模型进步迅速过期"；Opus 4.6 后 Anthropic 删除 sprint 分解层 | Anthropic《Harness design for long-running application development》2026-03-24，调研报告 §4.2 / §7.0 | ✅ 一手（2026-08-16 复审补充：Anthropic 原文为 "the model could natively handle the job without this sort of decomposition"；"entirely stripped" 一类更强措辞为 Portkey 转述，画面原话卡已固定为英文原文逐字引用） |
| 7 | Build to Delete：harness 要轻到可拆掉昨天写的逻辑；Vercel 砍 80% 工具后步数/token/延迟全面改善 | Philipp Schmid《Agent harness 2026》2026-01-05；Vercel 博客 2025-12，调研报告 §7.0（Manus 六个月重构 5 次同出此处，口播因时长未采用，可作备选论据） | ✅ 一手 |
| 8 | Thoughtworks 雷达把"团队共享的策展指令"列 Adopt 级（配置入 git、团队共享） | Thoughtworks Technology Radar Vol.34（2026-04）"Curated shared instructions for software teams"，调研报告 §4.7 / §7.2 第 3 条 | ✅ 一手 |
| 9 | 过期约束是纯成本（稀释指令、拖慢热路径） | 大纲第七期判据；佐证：arXiv 2602.11988（context files 推理成本 +20%）与 Thoughtworks "Agent instruction bloat"（Caution）——脚本未引用具体数字，仅作定性表述 | ⚠️ 开拍前复核：确认定性表述不与第二期对该论文的呈现冲突 |
| 10 | 冒烟配置"第二期已随模板仓库下发" | 大纲配套资产节 | ✅ 课程内设定 |
| 11 | Cursor `.cursor/hooks.json` 项目级 hooks 配置路径 | Cursor 官方 hooks 文档，调研报告 §2.2 | ✅ 一手 |
| 12 | 演示中"换上一代模型 capability 通过率下降、regression 保持全绿" | 策展叙事，非事实声明；画面按制作纪律标注预跑存档 | ⚠️ 属演示编排，不构成外部事实声明 |
| 13 | 演示实跑结果：6/6 全绿（regression 2/2 + capability 4/4，总耗时约 9 分钟） | `spike/captures/evidence/ep07/ep07-eval-results.json`（2026-08-07，eval-WUA，逐条 pass=true 已复核） | ✅ 一手（2026-08-16 复审修复：脚本原口径"爬坡组四个过两个 / 任务能力五成"与证据矛盾，口播、画面、Demo 步骤、B 方案已全部改为 6/6 全绿口径，"爬坡"形态不伪造失败） |
| 14 | 换模型对比段所需的"上一代模型重跑存档" | 证据不存在：`evidence/ep07/` 仅含单模型一次实跑 | ⚠️ 证据不存在（2026-08-16 复审标注）：开拍前须用多模型环境预跑上一代模型并存档 results.json，画面配"取自预跑结果存档"；无存档则按 2026-08-07 决策整段剪掉 |
