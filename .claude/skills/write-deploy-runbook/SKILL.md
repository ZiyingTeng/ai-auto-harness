---
name: write-deploy-runbook
description: 从本次部署的 trace 抽 AI 友好的 markdown 部署 runbook,保存到 reports/runbooks/<slug>-<date>.md,让人或其他 AI 30-60min 复现本次部署
allowed-tools: [Read, Bash, Write]
agent: runbook-agent
---

# write-deploy-runbook

## 落盘约定(必读)

- **runbook 文件**:`reports/runbooks/<slug>-<YYYY-MM-DD>.md` — AI 可消费的部署手册
- **日志**:`$WORKSPACE/logs/runbook.log` — 抽取过程的 bash stdout/stderr
- **结果**:`$WORKSPACE/results/runbook.json` — return schema
- **决策**:`runs/$RUN_ID/decisions.md` — append 一行(类似 verify-agent 的模式)

```bash
mkdir -p "$WORKSPACE/logs" "$WORKSPACE/results" reports/runbooks
LOG="$WORKSPACE/logs/runbook.log"
echo "==== runbook start at $(date -Iseconds) ====" >> "$LOG"
echo "=== PHASE_START phase=runbook slug=$SLUG run_id=$RUN_ID ts=$(date -Iseconds) ==="
```

## 你的输入(主 agent 传入)

```json
{
  "slug": "<project-slug>",
  "workspace_path": "/root/ai-auto-harness/workspace/<slug>",
  "run_id": "<from 主 agent>",
  "verify_passed": true,
  "verify_result": { "passed": true, "failed_at": null, "evidence": {...}, ... },
  "github_url": "https://github.com/.../...",
  "force_status": null
}
```

`force_status` 可选,用于失败 case 主 agent 显式指定 `paused_at_<phase>` / `blocked_<reason>` 时使用。默认从 `verify_result` 自动推导。

## 你的工作 = "模板 + LLM 双层抽取"

不要纯 LLM 自由写 runbook(每次结构都不一样,AI 消费不稳定)。
不要纯模板硬填(死板,丢失 trace 里的关键洞察)。

**做法**:

```
模板(固定骨架)
  ├── 1. frontmatter      ← 纯字段映射,从 state.json + results/ 抽
  ├── 3. 前置要求          ← 纯映射(GPU 大小 / 磁盘 / token 需求)
  ├── 6. 成本摘要          ← 纯映射(ndjson result event)
  └── 7. trace 指针        ← 纯字符串

LLM 抽取(语义浓缩,带模板约束)
  ├── 2. AI prompt        ← 你按模板生成,必须含 5 stage,每 stage 必含命令+成功标志+预计耗时
  ├── 4. 5 stage 指令     ← 你从 logs/<phase>.log 提取实际跑过的命令,套模板渲染
  └── 5. 错误速查         ← 你从 fixes.log + decisions.md 抽 error → fix 二元结构
```

## 工作流(分 5 步)

### 第 0 步:读取 _template.md

```bash
TEMPLATE=$(cat /root/ai-auto-harness/.claude/skills/write-deploy-runbook/_template.md)
```

把模板放到自己的上下文里,后面填空式生成,不让自己自由发挥。

### 第 1 步:聚合数据源(纯 bash + jq)

```bash
SLUG="$1"        # 从主 agent 传入
WORKSPACE="$2"
RUN_ID="$3"

# 抽 frontmatter 字段
STATE_JSON="$WORKSPACE/state.json"
INTAKE_JSON="$WORKSPACE/results/intake.json"
FETCH_JSON="$WORKSPACE/results/fetch.json"     # 或 fetch-weights.json,看实际
INSTALL_JSON="$WORKSPACE/results/install.json"  # 或 install-env.json
RUN_JSON="$WORKSPACE/results/run.json"          # 或 run-and-repair.json
VERIFY_JSON="$WORKSPACE/results/verify.json"
NDJSON="runs/$RUN_ID/harness.stdout.ndjson"
FIXES_LOG="$WORKSPACE/logs/fixes.log"
DECISIONS_MD="runs/$RUN_ID/decisions.md"

# 检查关键文件存在(缺失就降级)
for f in "$STATE_JSON" "$NDJSON"; do
    if [ ! -f "$f" ]; then
        echo "WARN: $f 缺失,部分字段无法填" >> "$LOG"
    fi
done
```

### 第 2 步:生成 frontmatter + 前置要求 + 成本摘要(纯填空)

```yaml
---
slug: <SLUG>
github: <github_url from intake.json>
hf_repos: <list from intake.json>
date: <today>
status: <success | incomplete_verify_failed | paused_at_<phase> | blocked_<reason>>
duration_min: <int>
turns: <int>
total_cost_usd: <float, from ndjson result event 的 total_cost_usd>
gpu_required_gb: <from intake.json.gpu_picks 推算>
disk_required_gb: <from fetch.json.bytes_total 估算 + 50GB safety>
needs_hf_token: <bool, intake.json.gated_repos 非空 = true>
audio_output_sec: <from verify.json.evidence.audio_info.duration_seconds,若有>
---
```

`status` 推导规则:

| verify_passed | run-and-repair 是否走完 | status |
|---|---|---|
| true | yes | `success` |
| false | yes(5 stage 都跑了但 verify 失败) | `incomplete_verify_failed` |
| 任何 | no(卡在 fetch/install/run 某 stage) | `paused_at_<phase>` |
| - | 任一资源 preflight 拒(GPU/磁盘) | `blocked_<reason>` |

### 第 3 步:生成 5 stage 指令(LLM 抽取,套模板)

读 5 个 phase 的 log + results,转成 stage 指令。每 stage 必须含:

- **执行命令**(从 logs/<phase>.log 抽实际跑过的成功命令,不是失败的)
- **成功标志**(用 grep / python -c assert / ls 之一)
- **预计耗时**(分钟数,从 ndjson timestamp 或 result event 推算)
- **已知踩坑**(若该 stage 有 fix,append "已知踩坑 N:" 块)

每 stage 抽取规则:

| Phase | 抽 entry_script | 抽 成功标志 | 抽 已知踩坑 |
|---|---|---|---|
| intake | `git clone <github>`(从 logs/intake.log) | `ls -la <repo>` 列举关键文件 | 若有 preflight 失败重试,记 |
| fetch-weights | `hf download <repo> --local-dir <path>`(从 results/fetch.json.weights_done) | `du -sh <path>` 至少 X GB | gated repo / resume / hf vs huggingface-cli |
| install-env | `pip install -r requirements.txt + 修复`(从 results/install.json.fixes_applied) | `python -c "import torch; assert ..."` | sm_120 / numpy / setuptools 冲突 |
| run-and-repair | 最后一次成功的 `python <entry>`(从 results/run.json.invoked_command) | `ls output/` 看到输出文件 | flash_attn / torchcodec / third_party symlink |
| verify | smoke test 命令(从 results/verify.json.evidence) | GPU util > 10% + 输出文件存在 | (verify 不修问题,通常无踩坑) |

### 第 4 步:生成"已知错误速查"(LLM 抽 fixes.log)

`fixes.log` 每行一条修复记录,格式约定:`<phase> <timestamp> <error> -> <fix>`(或类似)。

把每行转成下面的 4 字段块(严格按格式,不许散文):

```markdown
### 已知踩坑 N: <一句话现象>

**触发条件**: <grep-able 的 error message 或前置情况>
**根因**: <一句话>
**修复**:
\`\`\`bash
<复制可执行的命令>
\`\`\`
**验证修复成功**: <grep / python -c assert / ls 之一>
```

例(SongGen 实际):

```markdown
### 已知踩坑 1: torch 安装后 sm_120 不在 arch_list

**触发条件**: `python -c "import torch; print(torch.cuda.get_arch_list())"` 输出不含 `sm_120` 或 `compute_120`
**根因**: requirements.txt pin 了 `torch==2.6.0+cu126`,该 wheel 未编 sm_12 内核(RTX 5090)
**修复**:
\`\`\`bash
sed -i -E "/^(torch|torchvision|torchaudio)([=<>!~]|$)/d" requirements.txt
pip install --pre --index-url https://download.pytorch.org/whl/nightly/cu128 torch torchvision torchaudio
\`\`\`
**验证修复成功**: `python -c "import torch; assert any('120' in a for a in torch.cuda.get_arch_list())"`
```

### 第 5 步:生成"给 AI 的部署 prompt"(节 2,核心)

这是整份 runbook 最有价值的一节:**一段可以整体复制给 Claude / ChatGPT 让它按 runbook 执行的中文 prompt**。

模板见 `_template.md` 第 2 节,要确保:
- 标明强制规则(每 stage 等成功标志才进下一)
- 列出前置(GPU / 磁盘 / token)
- 5 stage 的命令 + 成功标志 + 已知踩坑全部嵌入
- 末尾标注"如遇此 runbook 未列的新错误,先 grep `fixes.log` 再问人"

### 第 6 步:写 runbook 文件

```bash
RUNBOOK_PATH="reports/runbooks/${SLUG}-$(date +%Y-%m-%d).md"
cat > "$RUNBOOK_PATH" <<RUNBOOK
<填好的完整 7 节内容>
RUNBOOK

echo "wrote runbook: $RUNBOOK_PATH" >> "$LOG"
```

### 第 7 步:敏感信息扫描(硬性最后一步)

写完后 grep 一次,确保**没泄露**:

```bash
# 任一返回非空 = 出错
grep -E "hf_[a-zA-Z0-9]{30,}" "$RUNBOOK_PATH"      # HF_TOKEN 真实值
grep -E "sk-ant-[a-zA-Z0-9_-]{20,}" "$RUNBOOK_PATH" # Anthropic key
grep -F "/root/ai-auto-harness/" "$RUNBOOK_PATH"   # 绝对路径

# 若有任一命中,raise + 重写时把真实值替换为占位符:
#   HF_xxx → ${HF_TOKEN}
#   sk-ant-... → ${ANTHROPIC_API_KEY}
#   /root/ai-auto-harness/ → ${HARNESS_ROOT}/
```

通过 → 进第 8 步;不通过 → 修后重 grep,直到通过。

### 第 8 步:写 results/runbook.json + 返回

```bash
cat > "$WORKSPACE/results/runbook.json" <<JSON
{
  "slug": "$SLUG",
  "runbook_path": "$RUNBOOK_PATH",
  "runbook_bytes": $(wc -c < "$RUNBOOK_PATH"),
  "status": "<success|incomplete_verify_failed|paused_at_X|blocked_Y>",
  "ai_prompt_word_count": <int, 从 节 2 字数>,
  "stage_count": 5,
  "traps_documented": <int, 已知踩坑 N 的最大 N>,
  "completed_at": "$(date -Iseconds)"
}
JSON

echo "- $(date -Iseconds) by runbook-agent: wrote $RUNBOOK_PATH ($status, $traps_documented traps)" >> "runs/$RUN_ID/decisions.md"
echo "==== runbook end at $(date -Iseconds) ====" >> "$LOG"
echo "=== PHASE_END   phase=runbook slug=$SLUG status=done ts=$(date -Iseconds) ==="
```

## 返回 schema

```json
{
  "slug": "song-generation",
  "runbook_path": "reports/runbooks/song-generation-2026-05-25.md",
  "runbook_bytes": 8234,
  "status": "success",
  "ai_prompt_word_count": 1234,
  "stage_count": 5,
  "traps_documented": 7,
  "completed_at": "2026-05-25T14:34:00+08:00"
}
```

**主 agent** 把这个 return 用于:
1. 写 `runs/$RUN_ID/runbook.json`(本次快照)
2. 更新 `state.json.runbook_path = <path>`
3. 传给 `write-recommendation` 的 `runbook_paths` 参数,日报里加链接
4. 传给 `cleanup-agent` 做 G3 防护(verify runbook 已写)

## 失败 case 的处理

主 agent 即使 verify_passed=false 也会 dispatch 你,这时:

| 输入 verify_passed | 输入 verify_result.failed_at | runbook status | 内容差异 |
|---|---|---|---|
| true | (null) | `success` | 完整 5 stage |
| false | "startup" / "smoke_test" / "gpu_utilization" | `incomplete_verify_failed` | 完整 5 stage,Stage 5 标 ⚠️ + 引 verify_result.evidence |
| false | (没到 verify,主 agent 显式传 force_status="paused_at_X") | `paused_at_<phase>` | 只到该 phase,后续标 "⚠️ 未到达,见 pending_human/<slug>.md" |
| false | (没到任何 phase,资源 preflight 拒) | `blocked_<reason>` | 只到 intake,主要说"为什么没跑成" |

理由:就算没跑通,记下"踩到 stage 3 装环境就崩了"对后人也有 ROI。

## 反模式

- ❌ 不要纯 LLM 自由写 — 必须套 _template.md 骨架
- ❌ 不要散文式写"已知踩坑" — 必须 `error_pattern → fix_command` 二元结构(4 字段:触发条件/根因/修复/验证)
- ❌ 不要泄露 HF_TOKEN / ANTHROPIC_API_KEY / 绝对路径 `/root/ai-auto-harness/` — 第 7 步必扫
- ❌ 不要"凑"5 stage(若实际没跑到 run-and-repair,Stage 4-5 标 ⚠️ 而不是编命令)
- ❌ 不要主动改 state.json — 主 agent 负责更新 `state.json.runbook_path`
- ❌ 不要触动 workspace 其他文件 — 你只写 `reports/runbooks/<slug>-<date>.md` + `logs/runbook.log` + `results/runbook.json`

## 我做错了什么?常见诱惑

- ❌ "fixes.log 里某条修复看起来很冗长,我简化一下表述" — **不**.原文照转更准确,你只把它套成 4 字段结构,不要主观重写
- ❌ "intake / fetch 都没踩坑,我可以不写这两个 stage" — **不**.每个 stage 都要有命令 + 成功标志,即使没踩坑
- ❌ "我看 verify_passed=true,fixes.log 里有的修复应该不重要" — **不**.该项目下次跑同样会撞那些坑,必须列
