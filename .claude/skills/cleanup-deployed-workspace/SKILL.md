---
name: cleanup-deployed-workspace
description: 部署成功且 runbook 已写后,白名单清理 workspace 的可重建产物(venv/.cache/repo),保留 state/results/logs/output,把磁盘从 ~30GB 降到 ~50MB
allowed-tools: [Read, Bash]
agent: cleanup-agent
---

# cleanup-deployed-workspace

## 落盘约定(必读)

- **日志**:`$WORKSPACE/logs/cleanup.log` — 每条删除/保留 + 总结
- **结果**:`$WORKSPACE/results/cleanup.json` — return schema
- **state 更新**:`$WORKSPACE/state.json` — `phase: "done" → "archived"`,加 `archived_at` + `freed_bytes`
- **决策**:`runs/$RUN_ID/decisions.md` — append 一行

```bash
mkdir -p "$WORKSPACE/logs" "$WORKSPACE/results"
LOG="$WORKSPACE/logs/cleanup.log"
echo "==== cleanup start at $(date -Iseconds) ====" >> "$LOG"
echo "=== PHASE_START phase=cleanup slug=$SLUG run_id=$RUN_ID ts=$(date -Iseconds) ==="
```

注:你**只 Read + Bash**(无 Edit/Write),所以"写日志"也只能通过 `>> "$LOG"` 或 `tee -a`,"写 JSON"用 `bash -c 'cat > $WORKSPACE/results/cleanup.json <<JSON ... JSON'`。

## 你的输入(主 agent 传入)

```json
{
  "slug": "song-generation",
  "workspace_path": "/root/ai-auto-harness/workspace/song-generation",
  "run_id": "songgen-e2e-20260525-143000",
  "verify_passed": true,
  "runbook_path": "reports/runbooks/song-generation-2026-05-25.md",
  "dry_run": false,
  "force_cleanup_incomplete": false
}
```

- `dry_run`:默认 `false`(生产模式真删);可传 `true` 走预演,不真删但 log 输出 "would rm"
- `force_cleanup_incomplete`:仅在 verify_passed=false 时由主 agent 显式传 true 才允许清(默认 false,verify 没过就保留 workspace)

## 工作流(分 5 步)

### 第 1 步:4 道防护(任一不过 = 整个 cleanup 跳过)

#### G1: workspace 路径前缀校验

```bash
case "$WORKSPACE" in
    /root/ai-auto-harness/workspace/*) : ;;
    *) 
        echo "REFUSED G1: workspace_path 不在安全前缀 (/root/ai-auto-harness/workspace/)" >> "$LOG"
        echo "  实际: $WORKSPACE" >> "$LOG"
        # 立刻 raise,绝不动磁盘
        SKIPPED=true
        SKIPPED_REASON="G1_prefix_check_failed"
        ;;
esac

# 进一步:workspace_path 必须包含 slug
case "$WORKSPACE" in
    *"/$SLUG"|*"/$SLUG/") : ;;
    *)
        echo "REFUSED G1: workspace_path 不包含 slug=$SLUG" >> "$LOG"
        SKIPPED=true
        SKIPPED_REASON="G1_slug_mismatch"
        ;;
esac
```

#### G2: trace 完整校验

```bash
TRACE_DIR="runs/$RUN_ID"
if [ ! -d "$TRACE_DIR" ]; then
    echo "REFUSED G2: trace 目录不存在: $TRACE_DIR" >> "$LOG"
    SKIPPED=true
    SKIPPED_REASON="G2_trace_dir_missing"
fi
if [ ! -f "$TRACE_DIR/harness.stdout.ndjson" ]; then
    echo "REFUSED G2: harness.stdout.ndjson 缺失" >> "$LOG"
    SKIPPED=true
    SKIPPED_REASON="G2_ndjson_missing"
fi
if [ ! -f "$TRACE_DIR/meta.json" ]; then
    echo "REFUSED G2: meta.json 缺失" >> "$LOG"
    SKIPPED=true
    SKIPPED_REASON="G2_meta_missing"
fi
# 若 G2 失败 → 还要写 pending_human,提示 trace 不完整可能 run 出问题
if [ -n "$SKIPPED_REASON" ] && [[ "$SKIPPED_REASON" == G2_* ]]; then
    cat > "pending_human/$SLUG.md" <<EOF
# Cleanup 跳过:$SLUG

**原因**: $SKIPPED_REASON
**Run ID**: $RUN_ID
**时间**: $(date -Iseconds)

trace 不完整,workspace 保留供事后排查。
若确认可清理,手动 \`rm -rf $WORKSPACE\` 即可。
EOF
fi
```

#### G3: runbook 已写校验

```bash
if [ ! -f "$RUNBOOK_PATH" ]; then
    echo "REFUSED G3: runbook 文件不存在: $RUNBOOK_PATH" >> "$LOG"
    SKIPPED=true
    SKIPPED_REASON="G3_runbook_missing"
elif [ $(wc -c < "$RUNBOOK_PATH") -lt 1024 ]; then
    echo "REFUSED G3: runbook 文件 < 1KB,可能是半成品: $RUNBOOK_PATH" >> "$LOG"
    SKIPPED=true
    SKIPPED_REASON="G3_runbook_too_small"
fi
```

**为什么 runbook 必须先在**:cleanup 删了 workspace 就没法重新抽 runbook。runbook 是清理后唯一的"如何复现"依据。

#### G4: verify_passed 或 explicit force

```bash
if [ "$VERIFY_PASSED" != "true" ] && [ "$FORCE_CLEANUP_INCOMPLETE" != "true" ]; then
    echo "REFUSED G4: verify_passed=false 且 force_cleanup_incomplete=false,跳过清理" >> "$LOG"
    SKIPPED=true
    SKIPPED_REASON="G4_verify_not_passed"
fi
```

**任一 G1-G4 不过** → 直接进第 5 步,return `{skipped: true, ...}`,**不动磁盘**。

### 第 2 步:白名单删(严格 bash,不用 `*` 展开 root)

**绝不**用 `rm -rf $WORKSPACE/$VAR/*`(`$VAR` 空就清根)。**必须**显式枚举:

```bash
FREED_BYTES=0
REMOVED=()
SKIPPED_NOT_EXIST=()

# 白名单(只清这 3 个目录)
TARGETS=(venv .cache repo)

for target in "${TARGETS[@]}"; do
    TARGET_PATH="$WORKSPACE/$target"
    if [ -d "$TARGET_PATH" ]; then
        # 抽尺寸(供日志/return)
        SIZE_BYTES=$(du -sb "$TARGET_PATH" 2>/dev/null | awk '{print $1}')
        SIZE_HUMAN=$(du -sh "$TARGET_PATH" 2>/dev/null | awk '{print $1}')
        
        if [ "$DRY_RUN" = "true" ]; then
            echo "[DRY] would rm -rf $TARGET_PATH ($SIZE_HUMAN)" >> "$LOG"
        else
            rm -rf "$TARGET_PATH"
            if [ $? -eq 0 ]; then
                echo "removed $TARGET_PATH ($SIZE_HUMAN, $SIZE_BYTES bytes)" >> "$LOG"
                FREED_BYTES=$((FREED_BYTES + SIZE_BYTES))
                REMOVED+=("$target")
            else
                echo "ERROR removing $TARGET_PATH" >> "$LOG"
            fi
        fi
    else
        echo "skipped $TARGET_PATH (not exist)" >> "$LOG"
        SKIPPED_NOT_EXIST+=("$target")
    fi
done

# 总结
FREED_HUMAN=$(awk -v b=$FREED_BYTES 'BEGIN{
    if (b > 1e9) printf "%.1fGB", b/1e9;
    else if (b > 1e6) printf "%.1fMB", b/1e6;
    else printf "%dKB", b/1e3
}')
echo "[summary] freed $FREED_HUMAN total" >> "$LOG"
```

### 第 3 步:保留清单审计(查不该删的没删)

```bash
KEPT=()
PRESERVE=(state.json results logs output progress.md)

for item in "${PRESERVE[@]}"; do
    KEEP_PATH="$WORKSPACE/$item"
    if [ -e "$KEEP_PATH" ]; then
        KEPT+=("$item")
    fi
done

echo "[summary] kept: ${KEPT[*]}" >> "$LOG"
```

### 第 4 步:更新 state.json → archived(dry_run 模式跳过这步)

```bash
if [ "$DRY_RUN" != "true" ]; then
    NOW=$(date -Iseconds)
    jq --arg p "archived" \
       --arg ts "$NOW" \
       --argjson fb $FREED_BYTES \
       '.phase = $p | .status = $p | .archived_at = $ts | .freed_bytes = $fb | .updated_at = $ts' \
       "$WORKSPACE/state.json" > /tmp/state-new && mv /tmp/state-new "$WORKSPACE/state.json"
    echo "state.json: phase=done → archived" >> "$LOG"
fi
```

### 第 5 步:写 results/cleanup.json + 返回

```bash
# 写 cleanup.json
bash -c "cat > '$WORKSPACE/results/cleanup.json' <<JSON
{
  \"slug\": \"$SLUG\",
  \"removed\": [$(printf '\"%s\",' "${REMOVED[@]}" | sed 's/,$//')],
  \"skipped_not_exist\": [$(printf '\"%s\",' "${SKIPPED_NOT_EXIST[@]}" | sed 's/,$//')],
  \"kept\": [$(printf '\"%s\",' "${KEPT[@]}" | sed 's/,$//')],
  \"freed_bytes\": $FREED_BYTES,
  \"freed_human\": \"$FREED_HUMAN\",
  \"dry_run\": $DRY_RUN,
  \"skipped\": ${SKIPPED:-false},
  \"skipped_reason\": ${SKIPPED_REASON:+\"$SKIPPED_REASON\"},
  \"runbook_path\": \"$RUNBOOK_PATH\",
  \"completed_at\": \"$(date -Iseconds)\"
}
JSON"

# decisions.md append
if [ "$DRY_RUN" = "true" ]; then
    echo "- $(date -Iseconds) by cleanup-agent: DRY-RUN would free $FREED_HUMAN (${REMOVED[*]})" >> "runs/$RUN_ID/decisions.md"
else
    echo "- $(date -Iseconds) by cleanup-agent: freed $FREED_HUMAN, archived $SLUG" >> "runs/$RUN_ID/decisions.md"
fi

echo "==== cleanup end at $(date -Iseconds) ====" >> "$LOG"
echo "=== PHASE_END   phase=cleanup slug=$SLUG status=done ts=$(date -Iseconds) ==="
```

## 返回 schema

成功 case:

```json
{
  "slug": "song-generation",
  "removed": ["venv", ".cache", "repo"],
  "skipped_not_exist": [],
  "kept": ["state.json", "results", "logs", "output"],
  "freed_bytes": 34567890123,
  "freed_human": "32.2GB",
  "dry_run": false,
  "skipped": false,
  "skipped_reason": null,
  "runbook_path": "reports/runbooks/song-generation-2026-05-25.md",
  "completed_at": "2026-05-25T14:35:00+08:00"
}
```

跳过 case(任一 G1-G4 失败):

```json
{
  "slug": "song-generation",
  "removed": [],
  "skipped_not_exist": [],
  "kept": [],
  "freed_bytes": 0,
  "freed_human": "0KB",
  "dry_run": false,
  "skipped": true,
  "skipped_reason": "G3_runbook_missing",
  "runbook_path": "reports/runbooks/song-generation-2026-05-25.md",
  "completed_at": "2026-05-25T14:35:00+08:00"
}
```

dry_run case:

```json
{
  "slug": "song-generation",
  "removed": [],
  "skipped_not_exist": [],
  "kept": ["state.json", "results", "logs", "output"],
  "freed_bytes": 34567890123,
  "freed_human": "32.2GB (dry_run, not actually freed)",
  "dry_run": true,
  "skipped": false,
  "skipped_reason": null,
  "runbook_path": "...",
  "completed_at": "..."
}
```

注:dry_run 模式下 `freed_bytes` 是**估算的 would-be 释放量**,不是实际释放。

## 反模式

- ❌ **绝不**用 `rm -rf $WORKSPACE/$VAR/*` 或 `rm -rf "$WORKSPACE"/*`(变量空就清根)— 必须显式枚举 `for target in venv .cache repo`
- ❌ **绝不**清 G1-G4 任一不过的 workspace — return skipped
- ❌ **绝不**删 PRESERVE 清单内的项(state.json / results / logs / output / progress.md)
- ❌ **绝不**触动其他 workspace(R1 隔离)— `$WORKSPACE` 必须以 `/root/ai-auto-harness/workspace/` 起头
- ❌ **绝不**在 dry_run=true 时真删 — 只写 "would rm" 日志
- ❌ **绝不**修问题或重跑 — cleanup 只清不修(出错就写 pending_human,主 agent 处理)

## 我做错了什么?常见诱惑

- ❌ "为了快,用 `rm -rf $WORKSPACE/{venv,.cache,repo}` 一行搞定" — **不**.bash brace expansion 在某些 shell 有兼容问题,且不易扩展(加 `output` 进白名单时容易写错). 显式 for 循环更安全
- ❌ "看到 venv 大,先 rm 几个 site-packages 试试" — **不**.白名单原则就是要么整目录删,要么不动. 部分删可能破坏依赖关系导致诊断混乱
- ❌ "verify_passed=false 但我看 runbook 写了 success,应该可以清" — **不**.verify_passed 是主 agent 传的事实,你无权重判. 严格按 G4 防护走
- ❌ "我顺便把 pending_human/$SLUG.md 也清了,反正部署成功了" — **不**.pending_human 是另一个 skill 的产物,不在你白名单内
- ❌ "为了省 log,把 cleanup.log 写成单行" — **不**.每个 target 单独一行,审计需要明细
- ❌ "我看 dry_run=true,顺手再 du -sh 一遍 workspace 总大小" — **可以**.dry_run 模式下额外查询是 OK 的,因为不真删. 但要把这些 query 也 echo 到 log

## 与 R 规则的关系

- **R1 workspace 隔离**:G1 防护就是 R1 在 cleanup 场景的硬化实现
- **R8 PHASE 标记**:进出 echo PHASE_START/END 必须有
- **R9 主 agent 严禁亲自 rm -rf**:你的存在就是让主 agent 不必自己干这事

## 测试时怎么用

### L1 单测(用 song-generation-run2 现成 workspace)

```bash
./bin/claude-haha -p "
按 cleanup-deployed-workspace skill 跑:
- slug: song-generation-run2
- workspace_path: /root/ai-auto-harness/workspace/song-generation-run2
- run_id: songgen-e2e-run3-resume-20260522-094040
- verify_passed: true
- runbook_path: reports/runbooks/song-generation-run2-2026-05-25.md
- dry_run: true
"
```

期望:
- 4 道防护全过(G1 prefix ✓ / G2 trace ✓ / G3 runbook ✓ / G4 verify ✓)
- log 输出 "[DRY] would rm venv (9.2GB), .cache (6.1GB), repo (180MB)"
- state.json 不更新(dry_run 模式)
- 磁盘大小不变
- return `dry_run: true, freed_bytes: ~16GB(estimated)`
