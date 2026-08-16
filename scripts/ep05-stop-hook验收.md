# 第 5 期：验证一级——"完成"的定义权

> 时长 5 分钟 | 交付物：Stop hook 验收配置 | 前置：第 1 期（优化信号而非意图、自我验收不可信）、第 3 期（PreToolUse 机制）
> 格式说明：开场回放第 1 期已定版实拍素材；Demo 为确定性机制实拍（exit 2 是进程级语义），实录证据 `evidence/ep05-stop-hook.cast`（15 轮 106 秒完整闭环），三连拦兜底证据为脱机判定记录 `evidence/ep05-blocks-三连拦.log`（非逐条录屏，如实标注）；作业为 verify-done.sh 直调判定，返回码是外部判据。
> 画面素材三分法（每个【画面】段逐项标注）：【实拍】= 终端真跑，禁止生成替代；【真截】= 真实网页/文档截图，禁止生成替代（假截图违反本课证据纪律）；【可生成】= 制作卡（card2png 管线或 AI 生成均可，卡上数据/引语必须真实可核）。

## 逐字稿

### 0:00–0:30 翻车导入

【口播】
第一期你见过那个现场：规则写得明明白白，它自作主张给绕了过去——它没撒谎，也没人逼它，它只是优化你给的信号。而"做完了"这句话，一直以来也是它自己说、自己算。如果你当时的解法是往 CLAUDE.md 里加一句"完成前必须跑测试"——那是第二期讲的东西：建议性约束，它遵守的概率很高，但违反的概率不为零。今天换个思路：让"完成"这个词，从它嘴里失效。

【画面】
前 5 秒回放第 1 期已定版开场实拍（中性指令三拍，`evidence/ep01-neutral-violation.cast`，打"第 1 期"角标）【实拍（回放）】→ 切黑底字幕卡【可生成】："完成的定义权，归谁？"

### 0:30–0:55 机制定位

【口播】
Claude Code 有个 Stop 事件：agent 每次想结束这一轮，都要经过它。挂一个 hook 上去，它说"我做完了"的那一刻，先跑你定义的验证；验证不过，脚本用一个约定的退出码告诉它：不准停，顺便把失败原因带给它，它就得接着干。宣布完成的权力，从此在 harness 手里。这不是野路子——官方插件 ralph-wiggum 干的就是这个，第六期细讲。

【画面】
官方 hooks 参考文档 Stop 事件段落【真截】→ 字幕卡【可生成】（静止 3 秒，观众自读，不占口播）：exit 2 = 拦截，stderr 失败信息回灌模型上下文 → 字幕卡【可生成】："Stop hook = 出口处的检票口"

### 0:55–2:35 Demo：配置 + 拦截实录

【口播】
看配置。settings.json 里挂 Stop hook，指向这个脚本。脚本干三件事：第一，跑一个快速检查——这里是 typecheck 加单测；第二，过了就放行；第三，不过就拦下，把失败摘要塞回给它。注意这段摘要的写法：它是写给模型看的指令，不是写给人看的报错——和第三期拦截信息同一个原则。
注意中间这段计数器：每拦一次记一笔，拦到第三次放行。没有它，一个修不好的检查就是死循环。
实录。仓库里已经埋了 4 处 bug，6 个测试红着，我给它的任务是写一个新函数，跟那些失败毫不相干。它写完，说"完成了"——看，Stop 被拦下，6 个失败用例的摘要塞了回去，它转去修。第二次想停，测试 10/10 通过，放行。注意，它从头到尾都没觉得自己在撒谎，它只是用了它手里唯一的信号。
整个机制里，没有任何一步取决于它自不自觉。

【画面】
1. 配置特写【实拍】：`settings.json` 配置特写（语法高亮，逐行放大）→ `verify-done.sh` 脚本滚动展示，计数器段落打框；配置高亮时弹出字幕卡【可生成】（观众自读，不占口播）：exit 0 = 放行 / exit 2 = 拦截，stderr 摘要回灌模型上下文（同第 3 期协议）
2. 实录【实拍】（证据 `evidence/ep05-stop-hook.cast`，15 轮 106 秒完整闭环）：terminal 演示——agent 宣布完成 → Stop hook 拦截（stderr 红字失败摘要出现在会话中）→ agent 修复 → 第二次放行。等待片段加"已剪辑"字幕【可生成】
3. 结尾字幕卡【可生成】："确定性：测试不过 = 停不了"

### 2:35–3:30 三个设计决策

【口播】
配置照抄容易，难的是三个设计决策。
第一，拦什么。只在验证失败时拦。通过了还拦，等于不许它下班，它会学着编造理由绕过你的检查。
第二，跑什么。你的全量测试要跑四十分钟，就不能进这条路——Stop hook 在热路径上，它每次想停都会触发。选快的代理信号：类型检查、受影响文件的测试、lint。信号可以弱，不能慢，慢了你三天后就会亲手把它关掉。
第三，兜底。最大拦截轮次。Cursor 干脆把这个上限做成了内置参数。防的不是模型，是你自己写了一个永远过不了的检查。

【画面】
三张字幕卡依次弹出【可生成】："只在失败时拦 / 快代理信号进热路径 / 最大轮次兜底"；讲第三点时在角落叠 Cursor hooks 文档 `loop_limit` 字段截图【真截】（字幕卡标注【可生成，数字须与文档逐字一致】：内置参数，默认 5 次）

### 3:30–4:25 原理：为什么必须 harness 来判

【口播】
为什么这件事必须 harness 做，不能靠模型自觉？因为它是优化器，优化的是你给的信号，不是你的意图。让它自己判自己，生成和评估出自同一个分布——放水不是它坏，是激励结构摆在那。这是结构问题，不是能力问题：新模型的自我验证确实在变强，但"自己给自己打分"的激励错配，不会随能力消失。Anthropic 自己的长任务实验就踩过这个坑：agent 评价自己的产出，会"自信地夸赞平庸之作"；他们的解法是把评估者拆成独立角色，调成怀疑论者。没有外部检查，"看着像做完了"就是它手里唯一的信号，而你就成了那条人肉验证循环。
但注意，到这一步我们只解决了"谁来打分"。验证信号还得独立于它的实现——不然它会删掉那个失败的测试，让验证通过。所以测试目录的保护不能少：第三期讲过的 PreToolUse 保护 hook，换个路径就是，不展开。

【画面】
1. 讲"自信地夸赞平庸之作"时叠 Anthropic《Harness design for long-running application development》博文截图【真截】（日期与作者标注【可生成】）
2. 讲测试保护时画面一角复现第 3 期 PreToolUse 配置【实拍（复用第 3 期素材）】，路径从 `generated/` 改成 `tests/`，打"第 3 期"角标【可生成】

### 4:25–4:45 信号保真度升级：Playwright MCP

【口播】
验证信号的保真度还能再升一档。左边是 agent 交上来的代码，它说"页面做完了"；右边是 Playwright MCP 实际渲染出来的样子。从"代码看着对"到"渲染结果对"——让评估者像用户一样去点你的应用，这是 Anthropic 那篇长任务 harness 文章里验收环节的标准做法。配置过程不占时间，这里不展开。

【画面】
左右对比卡【可生成·示意，必须打"示意图"角标，不得冒充实拍】：代码 diff（看着合理） vs 渲染结果（布局明显崩坏）。不打攻击/失败过程，只放成品对比。（2026-08-16 核查：本段无实拍证据，由"对比截图"降级为示意卡；若要实拍需另行补录。）

### 4:45–5:10 半衰期判据 + 交付 + 作业

【口播】
半衰期判据：这期内容模型再强也吸收不了，因为 ground truth 在外部世界，不在权重里。会过期的只是信号的形式——今天是单测，明天可能是渲染对比、端到端录制。
交付物就是这套 Stop hook 验收配置，在模板仓库第五期目录。作业：把它装进你自己的仓库，跑作业里附带的判定命令，两个返回码都对才算完成。
用 Cursor 的同事：`.cursor/hooks.json`，`stop` 事件，协议一样，还多一个内置的 loop limit。用 Codex 的：config.toml 里打开 hooks feature，思路相同。这门课主修 Claude Code，但验证独立这件事，到哪儿都不变。

【画面】
字幕卡【可生成】：判据"ground truth 在外部世界"→ 模板仓库目录树【真截】（高亮 `ep05/`；发布前补拍，画面里须能看到交付物文件）→ 作业三步骤卡【可生成】→ Cursor/Codex 对应位置双栏卡【可生成】

---

## Demo 详细步骤

### 准备（开拍前）

1. 示例仓库用 `template-repo/ep07/fixtures/demo-repo`（拍摄手册指定的 ep05 布景，与实录 `evidence/ep05-stop-hook.cast` 同款；拍前用旁边的 `demo-repo.baseline` 还原到 6 红基线）：含 `npm run typecheck`（tsc）和 `npm test`（node:test），全套通过耗时 ≤ 20 秒（热路径要求）。
2. 基线已埋好 4 处带注释标记的 bug（`src/utils.ts` 的 clamp/parsePercent/chunk/formatPrice），10 个测试中 6 个失败——与实录证据口径一致，不要改成"只埋一个失败测试"。
3. 交付物 hook 配置已随布景自带（`.claude/`，verify-done.sh 与下节内嵌版逐字一致，2026-08-16 已 diff 核对），`chmod +x .claude/hooks/verify-done.sh`。
4. 删除计数器与审计日志：`rm -f "${TMPDIR:-/tmp}/claude-stop-hook-count" .claude/verify-done.blocks.log`。

### 拍摄流程

1. **配置特写**（30s）：`code .claude/settings.json` → `code .claude/hooks/verify-done.sh`，旁白按逐字稿 0:55–2:35 段落走。
2. **拦截实录**（60–90s）：
   - 启动 `claude`，发任务："在 `src/` 里新增一个 `greet.ts`，导出 `greet(name: string): string`，返回 `Hello, ${name}!`。做完就结束，不要做别的。"（与实录证据同一条指令）
   - 预期（`evidence/ep05-stop-hook.cast` 同款，15 轮 106 秒）：agent 完成编码后宣布完成 → Stop hook 触发 → 6 个失败用例的摘要经 stderr 塞回会话 → agent 转去修 `src/utils.ts` 的 4 处 bug → 再次宣布完成 → 测试 10/10 通过 → 放行。
   - **确定性论证**：无论 agent 是否主动跑测试，只要测试处于失败状态，Stop 必被拦（exit 2 是进程级语义，不经过模型判断）。任务本身足够小，agent 一定会尝试结束，不存在"等不到它收尾"的风险。
3. **兜底演示（可选补拍，10s）**：手动往计数器文件写 `3`，再触发一次 Stop，验证直接放行。已有兜底证据 `evidence/ep05-blocks-三连拦.log` 为脱机判定记录（夹具 6 红基线下直调 hook：3 行 blocked round + 第 4 次 exit 0），非逐条录屏——引用时如实标注。

### B 方案（录不到时）

- agent 行为部分（它是否老老实实去修测试）若现场表现不符合叙事：改用策展 transcript，明示"选自 N 次运行中的典型样本"，拦截瞬间的 stderr 塞回画面必须保留实拍（这是确定性部分，一定能拍到）。
- 若 terminal 手机端可读性差：拦截瞬间的 stderr 文本单独做一张放大字幕卡。

---

## 交付物文件内容

以下两个文件即模板仓库 `ep05/` 目录内容（2026-08-16 复核：实体文件在 `ep05/.claude/` 下——`settings.json` + `hooks/verify-done.sh`，git 已跟踪，与本文内嵌版一致）。

### `.claude/settings.json`（增量片段）

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/verify-done.sh"
          }
        ]
      }
    ]
  }
}
```

### `.claude/hooks/verify-done.sh`

```bash
#!/usr/bin/env bash
# Stop hook：把"完成"的定义权收归 harness。
# exit 0 = 放行；exit 2 = 拦截，stderr 原样塞回模型上下文。
set -u
cd "${CLAUDE_PROJECT_DIR:-.}" || exit 0

# --- 决策三：兜底。拦满 3 次放行，防死循环 ---
COUNTER="${TMPDIR:-/tmp}/claude-stop-hook-count"
n=$(cat "$COUNTER" 2>/dev/null || echo 0)
[ "$n" -ge 3 ] && exit 0

# --- 决策二：快代理信号进热路径（换掉这两行适配你的仓库）---
# 要求：全套 ≤ 20 秒。40 分钟的全量测试不许进这里。
out=$(npm run typecheck --silent 2>&1 && npm test --silent 2>&1)

# --- 决策一：只在失败时拦 ---
if [ $? -eq 0 ]; then
  rm -f "$COUNTER"  # 通过即清零，防上一轮的拦截计数残留到下一轮
  exit 0
fi

echo $((n + 1)) > "$COUNTER"
# 审计日志：拦截留痕（计数器通过即清零，日志不清——eval/审计要能事后查证拦截发生过）
echo "$(date -u +%FT%TZ) blocked round $((n + 1))" >> .claude/verify-done.blocks.log
# stderr 是写给模型看的指令：说明为什么被拦 + 正确的下一步
echo "结束被拦截：验证未通过，任务不算完成。" >&2
echo "请根据以下失败摘要修复后，再宣布完成：" >&2
echo "$out" | tail -30 >&2
exit 2
```

> 配套（一句话，不重复教学）：测试目录保护复用第 3 期 PreToolUse 保护 hook，把路径从 `generated/` 换成 `tests/` 即可——防止它用"删测试"让验证通过。现成的一份就在本期布景里：`.claude/hooks/protect-generated.sh`。

---

## 作业（≤10 分钟，带外部判据）

1. 把交付物两个文件复制进你自己的仓库，把脚本里的检查命令换成你仓库的快速检查（typecheck / 受影响测试 / lint，任选，要求 ≤ 20 秒）。
2. 制造一个已知失败：任选一个测试，把断言改错（或临时在源码里引入一个类型错误）。
3. 跑判定：

```bash
echo '{}' | bash .claude/hooks/verify-done.sh; echo "exit=$?"
```

4. 还原失败，再跑一次同样的命令。

**外部判据（两条都满足才算完成）：**

- 失败状态下返回码为 `2`，且 stderr 里能看到失败摘要；
- 还原后返回码为 `0`，无输出。

（不靠自我报告：返回码是进程给的，不是你说了算。）

---

## Cursor / Codex 对应位置卡片（20 秒文案）

【口播】
用 Cursor 的：`.cursor/hooks.json`，`stop` 事件，stdio JSON 协议和 exit 2 语义与 Claude Code 一致——这是官方明示的兼容设计——还内置了 `loop_limit`，默认五次，兜底不用自己写。用 Codex 的：2026 年起 `config.toml` 里 `[features] hooks` 打开，配 hooks.json，思路相同。事件命名一个是 PascalCase 一个是 camelCase，别抄串了。

【画面】
三栏对照卡【可生成，字段名与默认值必须逐字可核】：Claude Code `settings.json → Stop` | Cursor `.cursor/hooks.json → stop（loop_limit 默认 5）` | Codex `config.toml → [features] hooks`

---

## 事实核对清单

| # | 事实声明 | 来源 | 状态 |
|---|---|---|---|
| 1 | Stop 事件存在；Stop/SubagentStop hook 可阻塞退出、强制 agent 继续 | docs.claude.com/en/docs/claude-code/hooks（调研 §1.2） | ✅ 一手 |
| 2 | hook 通信协议：exit 2 = 阻塞，stderr 回给 Claude | 同上 | ✅ 一手 |
| 3 | 官方 ralph-wiggum 插件用 Stop hook 拦截退出重喂 prompt | anthropics/claude-code 仓库 plugins/ 目录（调研 §5.2） | ✅ 一手 |
| 4 | Anthropic 长任务实验：self-evaluation 失灵，agent"自信地夸赞平庸之作"；解法是 generator/evaluator 分工，evaluator 用 Playwright MCP 像用户一样点应用 | anthropic.com/engineering/harness-design-long-running-apps（2026-03-24，调研 §4.2） | ✅ 一手 |
| 5 | 新模型自我验证在增强（"不是能力问题"的限定依据） | anthropic.com/news/claude-opus-4-6（"better code review and debugging skills to catch its own mistakes"，调研 §7.1） | ✅ 一手 |
| 6 | Cursor stop hook 内置 `loop_limit`，默认 5；Claude Code 默认不限；exit 2 语义一致为官方兼容设计 | cursor.com/docs/hooks（调研 §2.2） | ✅ 一手 |
| 7 | Codex 2026 新增 hooks（`[features] hooks`，hooks.json） | developers.openai.com/codex/config-basic（调研 §3.2） | ✅ 一手 |
| 8 | "优化的是你给的信号而不是你的意图"为课程统一表述 | 大纲"事实纪律"节 | ✅ 课程定稿表述 |
| 9 | 交付物脚本中 settings.json 的 Stop hook 具体字段语法、`$CLAUDE_PROJECT_DIR` 变量、stdin JSON 输入格式 | 官方 hooks 参考文档（2026-08-06 复核：code.claude.com/docs/en/hooks） | ✅ 已复核：`$CLAUDE_PROJECT_DIR`、settings 语法、stdin JSON 均与官方一致。原生防循环字段 `stop_hook_active` **存在**（Stop input 官方字段），但语义是"被拦后的续跑轮直接放行"=只拦一轮；本课设计要"最多拦 3 轮"，原生字段给不了，故保留自实现计数器（harness 自身有连续拦截上限强制结束，社区报告约 9 次，死循环有双保险）。计数器已补"通过即清零"修复 |
| 10 | "40 分钟全量测试"为举例数字，非引用 | — | ✅ 示例，不构成事实声明 |
| 11 | 实录口径：任务指令为"新增 greet.ts"；宣布完成 → 拦截（6 个失败用例摘要塞回）→ 修 `src/utils.ts` 4 处 bug → 10/10 放行；15 轮 106 秒、成本 $0.294 | `evidence/ep05-stop-hook.cast` + MANIFEST ep05 节 | ✅ 一手（2026-08-16 逐条复核 cast，口播与拍摄流程已对齐；旧稿"calc.ts 埋一个失败测试""formatPrice 任务""两行输出"均与证据不符，已改） |
| 12 | 三连拦 + 兜底放行（3 行 blocked round + 第 4 次 exit 0）为脱机判定记录（夹具 6 红基线下直调 hook），非逐条录屏 | `evidence/ep05-blocks-三连拦.log` | ✅ 一手，覆盖范围已在脚本如实标注 |
| 13 | 第 1 期终版开场为"中性指令三拍"规则违反实拍，全片无"谎报完成"画面（谎报诱发 11 次全诚实，拍不到） | `scripts/ep01-立论.md` + MANIFEST ep01 采样史 | ✅ 2026-08-16 修复：开场口播与"前置"表述原称"第 1 期你见过谎报完成现场"，与第 1 期终版矛盾，已改为规则违反现场 + 自我验收论点 |
| 14 | 交付物实体文件位置：`template-repo/ep05/.claude/`（`settings.json` + `hooks/verify-done.sh`，git 已跟踪，2026-08-16 复核确认存在；首轮复审误报"仅有 NOTES.md"系 ls 未显示隐藏目录所致，已更正）。verify-done.sh 与脚本内嵌版 diff 逐字一致（`spike/ep05-live` 里的副本为旧版，缺审计日志两行，以脚本内嵌版为准）；内嵌版 `bash -n` 通过 | 2026-08-16 diff / bash -n / git ls-files 核查 | ✅ 已核实；作业判定命令与外部判据（返回码 2/0）不依赖 agent 自我报告；配套注原指向 `template-repo/ep03/`（仅有 NOTES.md），已改指本期布景自带的 `protect-generated.sh` |
| 15 | Playwright MCP 对比段无实拍证据，"成品在模板仓库"亦不成立（`template-repo/ep05/` 无相关文件） | 2026-08-16 核查 | ⚠️ 已降级：画面改为示意卡（须打"示意图"角标），口播删去"成品在模板仓库"；若要实拍需另行补录 |

## 写作中发现的大纲问题（如实记录）

1. ~~**调研报告未覆盖 Stop hook 的 stdin 输入字段**~~（2026-08-06 已闭环）：官方 hooks 参考文档确认 `stop_hook_active` 存在，但语义是"只拦一轮"；本脚本保留自实现计数器以支持"最多拦 3 轮"，计数器已补"通过即清零"。harness 另有连续拦截上限（社区报告约 9 次）兜底。
2. 大纲给本期排的时长偏紧：Stop hook demo + 三个设计决策 + verifier 分离原理 + Playwright 30s + 判据/作业/Cursor 卡片，逐字稿已压到约 1400 字的上限边缘。若实拍时 demo 实录段落超时，建议优先压缩 0:30–0:55 机制定位段（ralph-wiggum 一句可删，第六期还会讲 Stop hook 与 /goal 的关系），不要动三个设计决策。
