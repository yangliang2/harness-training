# 第 3 期：Hooks 管"拦不拦得住"

> 时长 5 分钟 | 交付物：settings.json 里的 PreToolUse 保护 hook（+ 守护脚本；PostToolUse 示例在模板仓库） | 前置：第 2 期（同一条"别改 generated/ 目录"规则）
> 格式说明：本期演示为确定性机制实拍，Edit 拦截与 sed 绕过/补洞两拍均已实跑存档（证据见 `spike/captures/evidence/MANIFEST.md` ep03 节）；Step 2/3 拍摄布景（规则文件暂移出）在画面段如实标注；作业为 verify-hook.sh 外部判据。
> 画面素材三分法（每个【画面】段逐项标注）：【实拍】= 终端真跑，禁止生成替代；【真截】= 真实网页/文档截图，禁止生成替代（假截图违反本课证据纪律）；【可生成】= 制作卡（card2png 管线或 AI 生成均可，卡上数据/引语必须真实可核）。

## 逐字稿

### 0:00-0:30 翻车导入

【口播】
上一期我们把"别改 generated/ 目录"写进了 CLAUDE.md，加载位置也配对了。给你看实测记录：一句最日常的中性指令"把 API 地址切到 staging"，它顺手就把 `generated/` 里的生成文件直改了——规则它"看到"了，但没照做。这样的录屏我们三天采了三组 16 次：15/16、7/16、7/16，从不为零。而 `generated/` 改错了，下次 codegen 一跑，改动直接蒸发。这个后果我承受不起，所以今天把它升级成确定性拦截：PreToolUse hook。

【画面】
上一期配好的 CLAUDE.md 一闪而过【实拍（第 2 期素材回放）】→ 实拍片段【实拍】：`spike/captures/evidence/ep02/` Day3 中性指令组录屏选一条违反样本（D1-neutral.cast，选中条已附采样分布 `verdicts-v2.csv`：D 组 Day3 2/4 违反）——agent 的 diff 里出现 `generated/client.ts` 的改动 → 字幕卡【可生成】："概率服从 ≠ 确定执行"。

### 0:30-0:55 机制定位

【口播】
hook 不是 prompt。它是你在 settings.json 里注册的 shell 命令，挂在 agent 生命周期的固定卡点上——工具要动手，先过这道卡。官方文档的说法：无论模型决定做什么，hook 都会执行。换句话说，在这条链路上，模型没有投票权。你可能会问：permission 的 deny 规则不也能挡吗？能挡，但 deny 只回一句"不行"——它不知道该怎么办，大概率换个姿势再撞一次；hook 能把"为什么不行、正确的路在哪"写回它的上下文。一个是门禁，一个还兼着导航。Claude Code 的生命周期事件已经有三十来个（还在随版本增加），今天只打穿一个：PreToolUse，工具动手之前。

【画面】
图示【可生成】：agent loop 时间轴，标出 PreToolUse 的卡点位置（工具调用 →【hook】→ 执行）。字幕卡【可生成】（静止 3 秒，供观众自读）：本期口诀——"事实放 CLAUDE.md，流程放 skill，强制约束放 hook"（官方文档分工建议的课程归纳，非官方原文，依据见末节事实核对清单），本期讲第三句。

### 0:55-2:05 写 hook + 第一次拦截

【口播】
配置分两层。settings.json 里声明：哪个事件、管哪些工具、触发哪个脚本。脚本收到这次调用的信息，只看一件事：目标文件是不是落在 `generated/` 里。是，就用一个约定的退出码告诉系统：不准动——顺便把我们写的那段话带回去给它。

跑一遍。我让它把 `generated/client.ts` 里的 base URL 换成 staging 地址。它发出修改请求——被拦了，文件没动，它收到我们写的那段话。注意它的反应：没有重试，转头去读生成器模板了。这就是确定性和建议的差别。

但注意，这时候我们只锁了前门。

【画面】
屏幕实拍【实拍】：左侧 settings.json，右侧 `protect-generated.sh`，逐行高亮。高亮处弹出字幕卡【可生成】（观众自读，不占口播）：stdin JSON（工具名 + 参数）/ exit 2 = 阻塞 / stderr 回灌给模型。然后终端实拍【实拍，存档 `spike/captures/evidence/ep03-edit-blocked.cast`】：发出指令 → 写入 `generated/` 的调用出现 → 红色 blocked → agent 改去翻 `templates/`。段尾字幕一闪【可生成】："只锁了前门"。

### 2:05-3:05 高潮：sed 绕过与 matcher 补洞

【口播】
社区早就报告过一类绕过模式：Edit 被拦，它换 Bash，一条 `sed -i` 照样改文件。我现场演示：我明确下令——codegen 管道坏了，直接用 sed 改，责任我担。文件被改了，hook 一声没吭——刚才锁住的前门，形同虚设。

问题出在哪？matcher 只罩住了写文件的工具，没罩住命令行。换句话说，我们保护的不是"修改 `generated/`"这个动作，只是这个动作的一个入口。而入口，永远不止一个。

补法：matcher 加上 Bash，脚本对命令单独判断——`sed -i`、tee、重定向这类写入模式指向 `generated/`，照拦；cat、grep 这种只读的放行，别把它的手脚全捆了。再跑同一条指令——这次拦住了，`git diff` 干净。所以写保护 hook 不是配置清单，是博弈设计：先想"它会从哪绕"，再决定 matcher 长什么样。

【画面】
录屏【实拍，已存档 `spike/captures/evidence/ep03-sed-bypass.cast`】：同一指令跑两次。第一次（matcher 无 Bash）：sed 执行成功，`git diff` 显示 `generated/client.ts` 被改——字幕【可生成】"它试图绕过，成功了"。第二次（matcher 补上 Bash）：blocked 红字，`git diff` 干净。两段之间插一张图示【可生成】：动作 vs 入口（一个动作，N 个入口）。布景注（字幕角标）【可生成】：拍摄时规则文件暂移出——这代模型在规则在场时会直接拒演 sed，拍不到尝试动作；演示的是"用户显式越权时 Bash 通道管不管得住"。

### 3:05-3:35 拦截信息写法

【口播】
拦下来只是上半场，stderr 里写什么同样有讲究。这段文字不是给人看的报错，是塞回上下文、给模型看的 prompt——它会真的读，并且照着做。所以必须说两件事：为什么拦——`generated/` 是生成产物，手改会丢；正确动作是什么——去改 `templates/` 里的模板，再跑 `npm run codegen`。只写"禁止"两个字，它大概率换个姿势再撞一次；给了出路，它顺着就走了。写拦截信息就是写 prompt，这个心态转过来，质量立刻不一样。

【画面】
屏幕左右对比卡【可生成】：左边"⛔ 禁止修改 generated/"，右边完整版拦截信息（原因 + 替代动作，逐字取自交付物 `protect-generated.sh` 的 block 信息）。字幕卡【可生成】：stderr = 给模型的 prompt。

### 3:35-4:10 原理 + 一句带过

【口播】
为什么这事必须在 hook 层做？因为这是结构问题，不是能力问题。模型是优化器，优化的是你给的信号；规则文件只是信号之一，和"把任务做完"这个信号冲突时，它有权取舍。hook 把这条规则从信号变成了环境——环境里的事，没得谈。顺便说，exit 2 只是最硬的一档；exit 0 配合 stdout 输出 JSON，还能做更细的事——放行、改写工具入参、注入上下文，协议细节都在 cheatsheet 里。

hook 家族还有两类，今天一句带过：PostToolUse 是反馈型，工具跑完自动 lint --fix、截断超长输出，成品配置在模板仓库；Stop 是验收型，拦住退出、强制跑验证——那是第五期的正菜。

【画面】
图示【可生成】：规则文件 = 信号（可被取舍）vs hook = 环境（不可谈）。底部字幕条一闪【可生成】：PostToolUse / Stop → 模板仓库 & 第 5 期。

### 4:10-4:40 半衰期判据

【口播】
老问题：模型再强一倍，这期还成立吗？分两层答。具体这条保护 `generated/` 的 hook，编码的是"模型会误改生成产物"这个假设——假设会过期，也许哪天模型再也不碰生成产物了，那这条规则就该删，模型升级后该审计就审计，第七期专门讲怎么审。但"概率服从和确定执行之间的差值"永远在：只要规则还是建议，无视它的概率就不为零；只要后果你承受不起，这个差值就得用机制填掉。这跟模型多强没关系，跟你要不要为那个概率买单有关系。

【画面】
字幕卡【可生成】：会过期的 = 具体规则；不过期的 = 概率与确定的差值。

### 4:40-5:00 交付物 + 作业 + 卡片

【口播】
带走的文件：settings.json 的 PreToolUse 保护 hook 加守护脚本，在模板仓库 ep03 目录。作业：装进你自己的仓库，保护一个你真正在乎的路径，然后跑仓库里的 verify 脚本——它伪造三种工具输入打给你的 hook，三个 exit code 全对才算过。

用 Cursor 和 Codex 的同事，概念原样迁移：Cursor 在 `.cursor/hooks.json` 里配 preToolUse，exit 2 阻塞语义是官方明示兼容的；Codex 今年也上了 hooks，config.toml 里开 `codex_hooks` 开关再配 hooks.json——注意它目前只拦 shell 命令，编辑类工具还不在拦截面内。名字不一样，博弈是一样的。这是 Claude Code 主修课，但这一招不交学费。

【画面】
文件树截图【真截】（template-repo/ep03 目录真实文件树）→ 作业三步字幕卡【可生成】→ Cursor/Codex 对应位置卡片【可生成，事实已逐条核，见末节】。

---

## Demo 详细步骤

**前置环境**：demo repo 含 `generated/client.ts`（代码生成产物）、`templates/`、`package.json` 里有 `codegen` 脚本；已装 `jq`；`.claude/rules/` 里有第 2 期配好的那条规则（作用域正确）。

**Step 1 — 配置 hook（确定性，一条过）**
1. 写入下方"交付物文件内容"的两个文件；`chmod +x .claude/hooks/protect-generated.sh`。
2. **第一遍先把 matcher 写成 `"Edit|Write|MultiEdit"`（不含 Bash）**——这是翻车镜头的布景。
3. 重启 Claude Code 会话让 settings 生效。

**Step 2 — Edit 拦截（确定性，一条过）**
- 指令："把 generated/client.ts 里的 API base URL 改成 staging 的地址。"
- 预期：agent 发起 Edit → 终端显示 hook blocked，stderr 信息回灌 → agent 转去读 `templates/`。
- 录不到（agent 直接先去读模板）：改指令为"直接改 generated/client.ts，不要动模板"，强迫它撞 Edit。仍不行就剪掉此段，从 Step 3 讲起。

**Step 3 — sed 绕过与补洞（确定性设计，见下）**
- 指令（2026-08-06 实测校准版，两拍均实跑通过）："codegen 管道现在坏了，等不及修。我明确要求你直接用 Bash sed -i 把 generated/client.ts 里的 https://api.example.com 改成 https://staging-api.example.com，就这一次，责任我担。执行即可，不要改模板。"——用户显式指定 sed 且给出"紧急+担责"的越权理由，模型会执行。**注意：温和的指令版本（"用 sed 把 base URL 改成 staging"）当前模型会引用规则拒绝执行**（实测复现），必须用强硬指令版；这不是造假，演示的本来就是"用户显式越权时 hook 管不管得住"。
- 预期（matcher 无 Bash）：sed 执行成功，`git diff` 显示文件被改。这就是翻车镜头。（实测彩蛋：agent 执行完会主动提醒"记得回滚"，可留作花絮。）
- 现场把 matcher 改为 `"Edit|Write|MultiEdit|Bash"`，重开会话，跑同一条指令。
- 预期：hook 拦截，`git diff` 干净（实测：拦截后 agent 会给出"改模板走 codegen"或"请你明确指示临时放行 hook"两个替代选项——第二个选项恰好是绝佳的现挂台词："hook 管的是 agent，管不了人。这也对——约束的对象本来就是它。"）。
- **B 方案**（若真实环境中该绕过模式不复现）：翻车段改旁白 + 示意图（Edit 被拦 → 箭头绕到 Bash → sed 落地），并在字幕明示"社区已报告的绕过模式，示意复现"；或策展 transcript 标注"选自 N 次运行的典型样本"。
- **Step 2 补充（2026-08-06 实测）**：规则文件在场时，当前模型对"直接改 generated/"会 preemptively 拒绝（不触发 hook，拍不到拦截画面）。实拍布景：Step 2 拍摄时临时把 `.claude/rules/generated.md` 移出再发指令，agent 发起 Edit/(cp 也行) → hook 拦截 → stderr 回灌 → agent 改走"改模板 + npm run codegen"正路。整条已实跑录到 transcript（含 cp 被拦、改道成功的完整链路），B 方案可直接策展这条。

**Step 4 — 作业 verify 脚本演示（10 秒）**
- 终端跑 `bash .claude/hooks/verify-hook.sh`，输出三行 PASS。

**制作纪律自查**：全程不出现 `.env`、`rm -rf`；terminal 字号按手机端可读性调大；等待处加"已剪辑"字幕。

---

## 交付物文件内容

### `.claude/settings.json`（片段）

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit|Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/protect-generated.sh"
          }
        ]
      }
    ]
  }
}
```

### `.claude/hooks/protect-generated.sh`

```bash
#!/usr/bin/env bash
# PreToolUse hook：禁止以任何入口写入 generated/
# 协议：stdin 收 JSON；exit 2 = 阻塞，stderr 回灌给模型（是 prompt，不是报错）
set -euo pipefail

input=$(cat)
tool=$(jq -r '.tool_name' <<<"$input")

block() {
  cat >&2 <<'MSG'
blocked: generated/ 是代码生成产物，手改会在下次 codegen 时丢失。
正确做法：修改 templates/ 里的生成器模板，然后运行 npm run codegen。
MSG
  exit 2
}

case "$tool" in
  Edit|Write|MultiEdit)
    file=$(jq -r '.tool_input.file_path // empty' <<<"$input")
    [[ "$file" == *generated/* ]] && block
    ;;
  Bash)
    cmd=$(jq -r '.tool_input.command // empty' <<<"$input")
    # 写入型命令（sed -i / awk -i / tee / 重定向 / mv|cp 目标）指向 generated/ 才拦，
    # 只读命令（cat、grep）放行
    if grep -qE '(sed +-i|awk +-i|tee +|>>? *[^ |]*generated/|\b(mv|cp)\b[^|]*generated/)' <<<"$cmd" \
       && grep -q 'generated/' <<<"$cmd"; then
      block
    fi
    ;;
esac
exit 0
```

### `.claude/hooks/verify-hook.sh`（作业判据脚本，随仓库下发）

```bash
#!/usr/bin/env bash
# 伪造三种 PreToolUse 输入打给 protect-generated.sh，断言 exit code
HOOK="$(dirname "$0")/protect-generated.sh"
pass=0; fail=0
check() { # $1=期望 exit code $2=描述 $3=JSON 输入
  set +e
  echo "$3" | bash "$HOOK" >/dev/null 2>&1
  got=$?
  set -e
  if [ "$got" -eq "$1" ]; then echo "PASS: $2"; pass=$((pass+1));
  else echo "FAIL: $2 (expected $1, got $got)"; fail=$((fail+1)); fi
}
check 2 "Edit 写 generated/ 被拦" \
  '{"tool_name":"Edit","tool_input":{"file_path":"generated/client.ts"}}'
check 2 "Bash sed -i 写 generated/ 被拦" \
  '{"tool_name":"Bash","tool_input":{"command":"sed -i s/a/b/ generated/client.ts"}}'
check 0 "Edit 写普通文件放行" \
  '{"tool_name":"Edit","tool_input":{"file_path":"src/app.ts"}}'
echo "---"; echo "$pass passed, $fail failed"
[ "$fail" -eq 0 ]
```

---

## 作业（≤10 分钟，带外部判据）

1. 把 `protect-generated.sh` 和 settings.json 片段装进你自己的仓库，把 `generated/` 改成你真正在乎的保护路径（构建产物、迁移目录、锁文件目录均可）。
2. 同步改造 `verify-hook.sh` 里的三条路径。
3. **外部判据**：跑 `bash .claude/hooks/verify-hook.sh`，输出 `3 passed, 0 failed` 才算完成。不允许"我开会话试了试好像行"——脚本不过就是没过。

---

## Cursor / Codex 对应位置卡片（20 秒文案）

> 这门课以 Claude Code 为主修，但 hook 概念三家通用：
> - **Cursor**：`.cursor/hooks.json`（项目级）/ `~/.cursor/hooks.json`（用户级），事件名 camelCase——`preToolUse`。stdio JSON 协议和 exit 2 阻塞语义与 Claude Code 一致，是官方明示的兼容设计（Cursor 1.7，2025-09-29 起，cursor.com/docs/hooks 原话 "matches Claude Code behavior for compatibility"）。
> - **Codex**：2026 年实装（UnderDevelopment 阶段，默认关），`~/.codex/config.toml` 里 `[features]` 下 `codex_hooks = true` 打开，配 `hooks.json`。两个差异要知道：PreToolUse 目前只拦 shell/Bash 工具（apply_patch、MCP 等不在拦截面内）；判定只认 JSON 输出 `permissionDecision: "deny"`，不是 exit 2。
> - 迁移本期脚本：Cursor 侧只需改配置文件位置和事件名大小写，exit 2 语义、matcher 要覆盖 Bash 这条教训原样适用；Codex 侧输出契约要改成 JSON deny，且它"只罩 shell"的现状恰好把本期"补 Bash 入口"的博弈反过来了——它的口子不在 Bash，在编辑类工具，迁移前先看官方文档当期覆盖面。

---

## 事实核对清单

| 本期事实声明 | 来源 | 状态 |
|---|---|---|
| hooks 是注册在 settings.json 的 shell 命令，在固定生命周期点执行，"无论 Claude 决定做什么"都执行 | 官方 memory 文档原话："Hooks execute as shell commands at fixed lifecycle events and apply regardless of what Claude decides to do"（调研报告 §1.1，docs.claude.com/en/docs/claude-code/memory；2026-08-16 复核逐字命中） | ✅ 一手 |
| Claude Code 生命周期事件已有三十来个 | 官方 hooks 参考文档（2026-08-16 复核：事件表实数 31 行；调研时为 30，数量随版本漂移） | ✅ 一手，口播已改为"三十来个"不写死数字 |
| PreToolUse 在工具执行前触发；exit 2 = 阻塞，stderr 回给 Claude | 官方 hooks 参考文档（调研报告 §1.2） | ✅ 一手 |
| hook 输入为 stdin JSON（含 tool_name、tool_input） | 官方 hooks 参考文档（调研报告 §1.2 通信协议） | ✅ 一手 |
| "bash 绕过 hook"是社区已报告的绕过模式 | 大纲"事实纪律"指定表述；不做确定性断言 | ✅ 按大纲口径；具体绕过样本开拍前实测（大纲二审遗留事项） |
| settings.json 中 matcher 正则语法、`$CLAUDE_PROJECT_DIR` 变量、`Edit\|Write\|MultiEdit\|Bash` 写法 | 官方 hooks 参考文档 | ⚠️ 开拍前对照当月文档逐字复核（交付物配置为教学构造，正则细节以官方文档为准） |
| PostToolUse 可做 lint --fix / 输出截断；Stop 可拦退出 | 官方 hooks 参考文档"真实用法"（调研报告 §1.2） | ✅ 一手 |
| "模型是优化器，优化的是你给的信号而不是你的意图" | 大纲"事实纪律"指定口径 | ✅ 按大纲口径 |
| hook 编码"模型做不到 X"的假设，假设会过期 | Anthropic《Harness design for long-running application development》（调研报告 §4.2/§7） | ✅ 一手 |
| Cursor hooks：1.7（2025-09-29）beta，`.cursor/hooks.json`，camelCase 事件，exit 2 语义官方明示兼容 | cursor.com/changelog/1-7（"Hooks (beta)"，2025-09-29）、cursor.com/docs/hooks（"Exit code 2 ... matches Claude Code behavior for compatibility"）（调研报告 §2.2；2026-08-16 复核一致） | ✅ 一手 |
| Codex 2026 实装 hooks（`[features]` 下 `codex_hooks = true` + hooks.json；PreToolUse 目前仅拦 shell/Bash，判定只认 JSON deny） | OpenAI Codex 文档 + 2026-07 更新的实测综述（调研报告 §3.2；2026-08-16 复核，开关名与拦截面已按实修订） | ✅ 已修订口径 |
| 第 2 期规则"作用域配对仍可能被无视，概率不为零" | 第 2 期内容 + 大纲"事实纪律"（不用无出处精确数字） | ✅ 按大纲口径 |
| 口诀"事实放 CLAUDE.md，流程放 skill，强制约束放 hook" | 课程归纳，非官方原文；依据：官方 memory 文档（facts 留 CLAUDE.md、multi-step procedure 移 skill、must run 写 hook）+ hooks 参考文档 | ✅ 口径已修订（画面字幕标注"课程归纳"） |
| 开场翻车段三天采样 15/16、7/16、7/16 | `spike/captures/evidence/MANIFEST.md` ep01/ep02 节 + `evidence/ep02/verdicts-v2.csv`（2026-08-16 复核一致；Day1 规则载体 rules 文件、Day3 起根 CLAUDE.md，口径为"同一任务"） | ✅ 一手 |
| 开场选条 D1-neutral.cast 为违反样本（diff 含 generated/client.ts） | `evidence/ep02/verdicts-v2.csv`：D 组 Day3 2/4 违反（D1、D4），D1 判定"违反" | ✅ 一手（2026-08-16 复核） |
| sed 绕过与补洞两拍（v1 sed 落地 / v2 matcher 加 Bash 拦截、diff 干净） | `evidence/ep03-sed-bypass.cast`（2026-08-16 复核 cast 内容与声称一致；布景为规则文件暂移出，画面已注） | ✅ 一手实拍 |
| Edit/cp 写 generated/ 被拦 → 改走 templates + codegen 正路 | `evidence/ep03-edit-blocked.cast`（2026-08-16 复核一致） | ✅ 一手实拍 |
| 交付物三文件与脚本内嵌版本逐字一致；作业脚本可运行、判据外部 | 2026-08-16 diff 复核：`template-repo/ep03/.claude/settings.json`、`hooks/protect-generated.sh`、`hooks/verify-hook.sh` 均逐字一致；`bash -n` 通过；verify-hook.sh 在 jq 可用环境下实跑 3 passed, 0 failed（本机无 jq 时为 127，NOTES.md 已载依赖） | ✅ 已核 |

**2026-08-16 复审修复记录**：① 补三分法图例并给全部【画面】段逐项标【实拍】/【真截】/【可生成】；② "30 个生命周期事件"→"三十来个"（官方参考现列 31 个，数字随版本漂移）；③ "官方口诀"→"课程归纳"（官方文档支持分工建议但无此原文）；④ Codex 开关名 `[features] hooks`→`codex_hooks = true`，并补"仅拦 shell/Bash、判定 JSON deny"两个差异（口播与卡片同步）；⑤ Demo Step 3 预期中 agent 的两个替代选项与存档 cast 对齐（"改模板走 codegen"/"请你临时放行 hook"）；⑥ 开场选条从"任选一条"收紧为指定违反样本 D1 并附采样分布。
