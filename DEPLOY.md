# AI Stack 迁移清单 / DEPLOY.md

把 `ai-daily-scan` + `ai-auto-harness` + `auto-deploy-agent` 搬到新服务器的对照清单。
**用法**:旧机打包 → 新机解压到 `/root/` → 照本文 1→6 节逐步做。

> 三者关系:
> ```
> ai-daily-scan（发现 AI 项目，出报告）
>      └─MCP→ ai-auto-harness（cron 平台：部署+验证+出建议，依赖 ai-daily-scan）
> auto-deploy-agent（旧 Python 部署框架，已弃用，仅留档/参考）
> ```
> 引擎说明:ai-auto-harness **自带** claude-haha 引擎（claude-haha fork），**不需要** `/root/claudecode_sourcecode1`。

---

## 1. 新机器需要满足的条件（前提）

| 项 | 要求 | 说明 |
|---|---|---|
| 系统 | Linux x86_64 | |
| **bun** | 已安装（旧机现用 nvm node v22 自带的 bun） | ai-auto-harness 用。`curl -fsSL https://bun.sh/install \| bash` |
| **Python** | ≥ 3.10 + pip + venv | 两个 Python 项目用 |
| **GPU**（可选） | NVIDIA GPU + CUDA 驱动；显存建议 ≥16–24GB | **只在真跑 self-host 部署重模型时需要**；skill 会跳过已用显存 ≥15GB 的卡。无 GPU 见第 6 节降级用法 |
| **磁盘** | 系统盘几十 G；**另留大数据盘给 workspace/weights** | 单个模型权重 13–24GB，旧机 workspace+runs 占 ~68G。建议预留 ≥300–500GB |
| **网络** | 能访问：① 你的 LLM 端点/代理 ② HuggingFace ③ PyPI/GitHub | 内网环境注意代理设置 |
| **路径** | **强烈建议沿用 `/root/ai-auto-harness` 和 `/root/ai-daily-scan`** | 否则要改几十处硬编码（见第 5 节） |

---

## 2. 需要准备 / 替换的参数（最关键）

新机器上要填的环境变量（写进各项目的 `.env`）：

### 核心：LLM 接入（不配则 harness 起不来）
| 变量 | 用途 | 必填 |
|---|---|---|
| `ANTHROPIC_AUTH_TOKEN` 或 `ANTHROPIC_API_KEY` | LLM 认证 token | ✅ |
| `ANTHROPIC_BASE_URL` | LLM 端点；走自定义/代理时必填，走官方 API 可留空 | 看方案 |
| `ANTHROPIC_MODEL` / `ANTHROPIC_DEFAULT_OPUS_MODEL` / `..._SONNET_MODEL` / `..._HAIKU_MODEL` | 模型名映射（自定义端点时按对端模型名填） | 自定义端点时✅ |

> ⚠️ **头号决策**:旧机用的是**自定义端点**（很可能配套本地代理 `bin/anthropic-qwen-proxy`）。
> 新机器二选一:
> - **(A) 直接用 Anthropic 官方 API** → 填官方 token，`BASE_URL` 留空，模型名用官方名（最省事）；
> - **(B) 沿用自定义端点/代理** → 把 `bin/anthropic-qwen-proxy` 也在新机起起来，`BASE_URL` 指向它。
> 旧 `.env` 里若带的是旧端点地址，**务必改成新机器可达的地址**。

### 模型权重
| 变量 | 用途 | 必填 |
|---|---|---|
| `HF_TOKEN` | 下 HuggingFace（含 gated）模型权重 | 下模型时✅ |

### 路径覆盖（仅当**不**放在 `/root` 时才需要）
| 变量 | 用途 |
|---|---|
| `AI_AUTO_HARNESS_ROOT` | ai-auto-harness 根（cron 脚本读） |
| `ADA_WORKSPACE` | auto-deploy-agent 工作区 |
| `VERIFY_CC_HOME` / `VERIFY_CC_CONFIG_DIR` | auto-deploy-agent verify 模块调用的 CC 安装位置（可指到 ai-auto-harness） |

### 可选 / 调优
| 变量 | 默认 | 用途 |
|---|---|---|
| `FEISHU_WEBHOOK_URL` | — | 报告推送飞书 |
| `TOP_N` / `MAX_ANALYST_CANDIDATES` / `ANALYST_CONCURRENCY` | 3 / 8 / 4 | ai-daily-scan 调优 |
| `ADA_MAX_REPAIR_ITERATIONS` / `ADA_REPAIR_TOTAL_BUDGET_S` | 5 / 1800 | auto-deploy-agent 修复预算 |

---

## 3. 打包（在旧机器上执行）

> ⚠️ **tar 不认 `.gitignore`**：只会排除下面显式列出的项，所以运行时/过程文件要逐个排掉。

```bash
tar czf ai-stack.tar.gz -C /root \
  `# 大文件 / 可重建` \
  --exclude='*/workspace' --exclude='*/runs' \
  --exclude='*/node_modules' --exclude='*/.venv' --exclude='*/.git' \
  `# 缓存 / 构建产物` \
  --exclude='*/__pycache__' --exclude='*/.pytest_cache' --exclude='*/.mypy_cache' \
  --exclude='*.egg-info' --exclude='*.pyc' --exclude='*.log' \
  `# 运行时 / 过程数据` \
  --exclude='ai-auto-harness/reports' --exclude='ai-auto-harness/pending_human' \
  --exclude='ai-auto-harness/git-learning-remote.git' \
  --exclude='ai-auto-harness/git-learning-teammate' \
  --exclude='auto-deploy-agent/modules/verify/logs' \
  --exclude='ai-daily-scan/*_report.md' \
  `# —— 运行历史/记忆:想"接着上次跑"就删掉下面两行保留它们 ——` \
  --exclude='ai-auto-harness/memory' \
  --exclude='ai-daily-scan/state' \
  ai-daily-scan ai-auto-harness auto-deploy-agent
```
> - `.env` **会被一起打包**（git 忽略它，tar 不忽略）—— 配置跟着走，但**端点/token 必须按第 2 节改成新机适用值**。
> - `memory/`（harness 记忆）和 `ai-daily-scan/state/`（已扫描指纹/产出记录）是"运行历史"：上面默认排除=**全新开始**；若想新机**接着上次的进度跑**，把那两行 `--exclude` 删掉即可保留。
> - **想要绝对干净**（只带 git 认可的文件 + `.env`），见下方备选命令。

**备选:按 git 跟踪清单打包（保证零过程文件）**
```bash
cd /root
( for d in ai-daily-scan ai-auto-harness auto-deploy-agent; do
    git -C "$d" ls-files --cached --others --exclude-standard | sed "s|^|$d/|"
    [ -f "$d/.env" ] && echo "$d/.env"
  done ) > /tmp/pack.list
tar czf ai-stack.tar.gz -C /root -T /tmp/pack.list
```
> 这条用 git 的忽略规则筛文件，`.gitignore` 里的运行时数据一律不进包（最干净）；缺点是不含 `memory/`、`state/` 等历史。

---

## 4. 新机器上的步骤（按顺序）

```bash
# 1) 解压到 /root（保持同路径）
cd /root && tar xzf ai-stack.tar.gz

# 2) 装运行时（若没有）
#    bun:  curl -fsSL https://bun.sh/install | bash
#    python>=3.10 + venv

# 3) ai-auto-harness：重建依赖（不要用旧 node_modules）
cd /root/ai-auto-harness && bun install

# 4) ai-daily-scan：建 venv 装依赖
cd /root/ai-daily-scan
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt

# 5) auto-deploy-agent（可选，已弃用）
cd /root/auto-deploy-agent
python3 -m venv .venv && . .venv/bin/activate
pip install -e ".[llm]"

# 6) 配 .env（按第 2 节填）—— 至少 ai-auto-harness/.env 和 ai-daily-scan 的环境变量
#    若走自定义端点：先把代理起起来
#    cd /root/ai-auto-harness && ./bin/anthropic-qwen-proxy   （按需）
```

---

## 5. 验证（怎么算搬成功）

```bash
# A. harness 引擎能跑（最基本）
cd /root/ai-auto-harness && ./bin/claude-haha -p "回复 OK 两个字"
#   → 能正常返回 = LLM 端点/token 通了

# B. ai-daily-scan MCP server 能起
python /root/ai-daily-scan/mcp_server.py   # 不报缺包/缺 key 即可

# C.（有 GPU）跑一次部署链，看能否下权重+起服务
#   触发 ai-auto-harness 的 intake/auto-deploy skill，或 cron/daily.sh

# D. 挂定时任务（可选）：参考 cron/crontab.example
```

---

## 6. 常见坑 & 无 GPU 降级

**常见坑**
1. **LLM 端点连不上 → harness 起不来**（最高频）。先单独验证 A 步。
2. **没放在 `/root`** → 改这些硬编码：`.claude/settings.json`（hook 路径 + ai-daily-scan MCP 路径）、`.claude/skills/*/SKILL.md`、`cron/*.sh` 里的 `/root/ai-auto-harness`、`/root/ai-daily-scan`。或尽量用 `*_ROOT` 环境变量覆盖。
3. **直接拷了旧 `node_modules`** → 原生模块可能不匹配，删掉重新 `bun install`。
4. **磁盘不够** → 权重下不全 / workspace 写满。
5. **`.env` 带的是旧端点** → 必须改成新机可达地址。
6. auto-deploy-agent 的 verify 默认找 `/root/claudecode_sourcecode1` 的 CC → 没有就用 `VERIFY_CC_HOME` 指到 `/root/ai-auto-harness`（它自带 CC 引擎）。

**无 GPU 降级**:没 GPU 时,“发现 AI 项目 → 出公司视角建议”这条链照常跑;只有 `self_host_5090` 这条“下载+跑重模型验证”的路线会失败/跳过——可改走 `api` 路线（只生成 API 骨架，不本地跑）。

---

## 7. 一句话总结

代码和流程都是可移植的;**真正决定能不能跑起来的就两件事:① LLM 怎么接(token+端点)② 有没有 GPU。** 这两个在新机器定下来,照本清单走即可。
