---
name: x-lab-research
description: 全自动深度调研——给定 GitHub URL 或帖子信息，自动完成 clone/安装/源码阅读/动手实验/评分/报告/飞书推送，零人工干预。重点产出技术原理、能力边界、架构洞察和工作流集成分析。支持断点续研。触发词：调研、试用、evaluate、x-lab、实验、帮我看看这个项目、帮我调研一下。
metadata:
  trigger-hint: 当消息包含"帮我调研一下这个项目"（dispatch 固定格式）、GitHub URL（github.com/xxx/yyy）、"调研"+"项目/仓库/repo"、"试用一下"、"evaluate"、"帮我看看这个项目"、"x-lab research"时触发。注意：如果消息只提到"日报"而没有具体项目 URL，应该由 x-lab-dispatch 处理，不是本 skill。
  openclaw:
    emoji: "🔬"
user-invocable: true
---

# x-lab-research — 全自动实操调研

> **前置依赖**：本 skill 可由 `x-lab-dispatch` 自动触发（从日报中筛选候选后派发），也可手动触发。三个 skill 的协作关系：`daily-x-signal`（生成日报）→ `x-lab-dispatch`（扫描筛选派发）→ `x-lab-research`（执行调研）。飞书文档写入需要配置 `X_LAB_FEISHU_DOC_ID` 环境变量。

你是调研助手。你在本地执行所有调研。

**核心原则：全程自动化，不问人。** 除非遇到无法解决的阻塞（如需要付费 API key），否则不要中途问用户确认任何事情。完成调研后直接推送结果。

---

## 飞书认证配置

所有飞书 API 调用（文档写入、消息推送）需要以下环境变量：

| 环境变量 | 说明 | 示例 |
|---------|------|------|
| `FEISHU_APP_ID` | 飞书应用的 App ID | `cli_a92684ffbc789cd3` |
| `FEISHU_APP_SECRET` | 飞书应用的 App Secret | （不要硬编码到 SKILL.md） |
| `DAILY_X_SIGNAL_FEISHU_RECEIVE_ID` | 消息推送目标的 **open_id**（receive_id_type=open_id） | `ou_9fe2b3d22ed03b23efdeb3afe8d6c60f` |
| `X_LAB_FEISHU_DOC_ID` | 飞书云文档的 document_id（从文档 URL 中获取） | `RcpOdakvOoWB5sxRVVtcPqnHnxb` |

**加载方式**：从 `~/.openclaw/workspace/daily-x-signal/.env.local` 加载（`source .env.local`）。

**重要**：
- `DAILY_X_SIGNAL_FEISHU_RECEIVE_ID` 是 **open_id**（以 `ou_` 开头），发消息时 `receive_id_type` 必须填 `open_id`
- open_id 是**每个飞书应用独立的**——同一个用户在不同 App 下有不同的 open_id，不要混用
- 获取 `tenant_access_token`：`POST https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal`，body `{"app_id": "$FEISHU_APP_ID", "app_secret": "$FEISHU_APP_SECRET"}`

---

## 触发模式

### 模式 A：直接指定

用户发来一个 GitHub URL 或项目名 → 直接执行调研。

**识别方式**：消息中包含 `github.com/`、或包含"调研"/"试试"/"看看"/"evaluate" + 项目名。

### 模式 B：dispatch 触发

x-lab-dispatch 发来调研指令，格式固定：
```
帮我调研一下这个项目：

GitHub: {url}
来源帖子: {post_url}
作者: @{handle}
实验ID: {id}
```

从消息中提取 GitHub URL、帖子 URL、作者、实验 ID。如果消息中包含实验 ID，直接使用；否则自动生成。

### 模式 C：从 pending.json 读取任务

启动时检查 `~/.openclaw/workspace/x-lab/pending.json`，如果存在且有 `"status": "pending"` 的任务，按顺序逐个执行。每个完成后：
1. 更新 pending.json 中该任务的 `"status": "done"`
2. 立即推送报告
3. 继续下一个

### 模式 D：批量调研

用户发来多个 URL 或说"把这几个都调研一下" → 逐个执行调研，每个完成后立即推送报告。

---

## 第一步：解析 GitHub URL（自动，不问人）

如果消息中已包含 GitHub URL，直接使用。

如果只有帖子信息（无 GitHub URL），按以下顺序自动获取：

1. **正则匹配**：从帖子 `text`、`summary_bullets`、`why_it_matters` 中匹配 `github.com/[\w.-]+/[\w.-]+`
2. **Web 搜索**：用 `web_search` 搜索 `"{author_handle} {关键词} site:github.com"`
3. **短链展开**：对 `text` 中的 t.co 链接，用 `web_fetch` 跟随重定向获取最终 URL
4. **如果以上都找不到**：跳过 clone 阶段，仅基于帖子内容和 web 搜索结果撰写评价报告（标注"未找到仓库，基于公开信息评价"）

**绝对不要问用户要 GitHub URL。** 找不到就基于已有信息完成调研。

---

## 第二步：检查断点续研

```bash
EXP_DIR=~/.openclaw/workspace/x-lab/experiments/$EXPERIMENT_ID
```

如果 `$EXP_DIR/experiment.json` 已存在，读取 `progress.phase`：

| 已完成阶段 | 恢复行为 |
|-----------|---------|
| `info_gathering` | 检查 `repo/` 目录是否存在。存在则跳过 clone，直接进入安装 |
| `installing` | 检查 `install.log` 最后一行是否包含 success/完成。是则跳过安装 |
| `testing` | 检查 `test.log` 是否存在。存在则跳过测试，进入源码分析 |
| `source_analysis` | 检查 `scores.technical_depth` 是否已填充。已填则跳过，进入动手实验 |
| `hands_on` | 检查 `demo.log` 是否存在。存在则跳过，进入评分 |
| `scoring` | 检查 `scores` 字段是否已填充。已填则跳过评分，直接写报告 |
| `reporting` | 检查 `research-report.md` 是否存在。存在则直接进入推送 |
| `done` | 回复"该项目已调研完成"，跳过 |
| `failed` | 如果 `retry_count < 2`，从失败的阶段重试；否则回复"已重试 2 次仍失败" |

如果 experiment.json 不存在，正常从头开始。

---

## 第三步：执行调研

### 工作目录

```
~/.openclaw/workspace/x-lab/
├── experiments/                   # 每个实验一个目录，目录即隔离（无需 git 分支）
│   └── {experiment_id}/
│       ├── experiment.json        # 结构化评分 + 进度追踪（唯一 source of truth）
│       ├── research-report.md     # 可读报告（主要产出）
│       ├── install.log            # 安装日志
│       ├── test.log               # 测试日志
│       ├── demo.log               # 动手实验日志
│       └── repo/                  # clone 的仓库（shallow）
├── EXPERIMENTS.md                 # 汇总表（derived view，从 experiment.json 生成）
├── pending.json                   # 待调研任务队列（dispatch 写入，research 消费）
└── dispatch-log-{date}.json       # 派发日志（dispatch 写入，仅做记录）
```

**隔离策略**：每个实验独立目录，无需 git 分支。如需归档旧实验，按月打包即可。

实验 ID 格式：`YYYY-MM-DD-{x_author}-{repo}`（如 `2026-03-22-epiral-bb-browser`）。
- `x_author`：X 作者 handle（不含 @）
- `repo`：GitHub repo 名称
- 如果没有 GitHub URL，`repo` 取帖子正文前 16 个字母数字字符（小写）

### 进度通知（飞书）

每完成一个主要阶段，发一条短飞书消息通知用户进度。这解决了调研过程中用户无法感知进度的问题（调研可能需要 20-40 分钟）。

**格式**：
```
🔬 {experiment_id} 进度：{phase_cn}完成
{一句话状态}
```

**示例**：
```
🔬 2026-03-23-oragnes-browser-use 进度：安装完成
✅ pip install -e . 成功（23秒），进入源码分析
```

只在阶段 A（信息收集）、B（安装）、D（源码分析）完成时发送。阶段 C（测试）和 E（实验）结果合并到 F（评分）通知中。不要每个小步骤都发消息。

如果飞书消息发送失败，忽略继续——进度通知是辅助功能，不应影响调研主流程。

### 安全约束

- 所有操作限制在 `~/.openclaw/workspace/x-lab/experiments/{id}/` 内
- 不用 sudo
- 安装超时 5 分钟，测试超时 3 分钟
- 不执行 `rm -rf`、`curl | bash`、`curl | sh` 等危险命令
- Python 用 venv 隔离，Node 用局部 node_modules
- 不注册账号，不调用需要 API key 的功能

### 进度追踪

每完成一个阶段，立即更新 experiment.json 的 `progress` 字段：

```bash
# 用 python3 更新 progress（避免 jq 依赖）
python3 -c "
import json, datetime
with open('$EXP_DIR/experiment.json', 'r') as f: d = json.load(f)
d['progress'] = {'phase': 'CURRENT_PHASE', 'started_at': d.get('progress',{}).get('started_at','$(date -u +%Y-%m-%dT%H:%M:%S)'), 'completed_at': None, 'error': None, 'retry_count': d.get('progress',{}).get('retry_count',0)}
with open('$EXP_DIR/experiment.json', 'w') as f: json.dump(d, f, ensure_ascii=False, indent=2)
"
```

### 阶段 A：信息收集

**进入时更新 progress.phase = "info_gathering"**

```bash
EXP_DIR=~/.openclaw/workspace/x-lab/experiments/$EXPERIMENT_ID
mkdir -p "$EXP_DIR"

# 初始化 experiment.json（含 progress 字段）
# ... 写入初始结构 ...

# 1. GitHub API 元信息
curl -s "https://api.github.com/repos/{owner}/{repo}" > "$EXP_DIR/github-meta.json"

# 2. Shallow clone（失败时重试 1 次）
git clone --depth 1 {github_url} "$EXP_DIR/repo" 2>&1 | tee "$EXP_DIR/clone.log"
if [ $? -ne 0 ]; then
    sleep 5
    git clone --depth 1 {github_url} "$EXP_DIR/repo" 2>&1 | tee -a "$EXP_DIR/clone.log"
fi

# 3. 读 README（取前 500 行）
head -500 "$EXP_DIR/repo/README.md"

# 4. 项目结构
find "$EXP_DIR/repo" -maxdepth 2 -type f | head -80
```

如果 clone 两次都失败，记录错误，跳过安装和测试阶段，基于 GitHub API 信息和帖子内容评价。

### 阶段 B：安装尝试

**进入时更新 progress.phase = "installing"**

自动检测项目类型并安装：

```bash
cd "$EXP_DIR/repo"

# 检测项目类型
if [ -f "pyproject.toml" ] || [ -f "setup.py" ] || [ -f "requirements.txt" ]; then
    echo "TYPE: Python"
    python3 -m venv "$EXP_DIR/venv"
    source "$EXP_DIR/venv/bin/activate"
    pip install -e . 2>&1 || pip install -r requirements.txt 2>&1
elif [ -f "package.json" ]; then
    echo "TYPE: Node"
    npm install 2>&1 || bun install 2>&1
elif [ -f "go.mod" ]; then
    echo "TYPE: Go"
    go build ./... 2>&1
elif [ -f "Cargo.toml" ]; then
    echo "TYPE: Rust"
    cargo build 2>&1
elif [ -f "Dockerfile" ]; then
    echo "TYPE: Docker（跳过安装，标注需要 Docker 环境）"
elif [ -f "SKILL.md" ]; then
    echo "TYPE: Claude Code Skill"
    # 只检查结构，不安装
else
    echo "TYPE: Unknown/Docs"
fi
```

安装输出保存到 `install.log`。记录 `install_success` 和 `install_seconds`。

**安装失败时**：记录错误信息，尝试不安装也能验证的部分（如读代码结构、跑 --help）。如果有 Dockerfile，标注"需要 Docker 环境"。

### 阶段 C：功能验证（快速冒烟）

**进入时更新 progress.phase = "testing"**

```bash
cd "$EXP_DIR/repo"

# 1. 跑测试（超时 3 分钟，只跑前面的测试）
if [ -d "tests" ] || [ -d "test" ]; then
    # macOS 可能没有 GNU timeout，用 python 替代
python3 -c "
import subprocess, sys
try:
    r = subprocess.run([sys.executable, '-m', 'pytest', 'tests/', '-x', '--tb=short', '-q'], timeout=180, capture_output=False)
except subprocess.TimeoutExpired:
    print('TIMEOUT: 测试超时（3分钟）')
" 2>&1 | tee "$EXP_DIR/test.log"
fi
if [ -f "package.json" ]; then
    python3 -c "import subprocess; subprocess.run(['npm', 'test'], timeout=180)" 2>&1 | tee -a "$EXP_DIR/test.log"
fi

# 2. CLI 检查
# 根据 package.json 的 bin 字段或 setup.py 的 entry_points 确定 CLI 命令
{detected_cli} --help 2>&1 | head -30

# 3. 基本 import 检查（Python）
python3 -c "import {package_name}; print('import OK')" 2>&1
```

测试输出保存到 `test.log`。

**测试超时时**：标注"测试超时，基于已执行部分评估"，不影响其他评分。

### 阶段 D：源码深度阅读（核心阶段，不可跳过）

**进入时更新 progress.phase = "source_analysis"**

这是调研报告质量的决定性阶段。不读源码就写不出有价值的报告。你要像一个资深工程师 code review 一样去理解这个项目。

**D.1 理解入口和数据流**

1. 找到入口文件（`main.py`/`index.ts`/`cmd/`/CLI entrypoint），读完整内容
2. 从入口追踪核心调用链：用户输入 → 经过哪些处理 → 最终输出是什么
3. 画出核心数据流：什么数据从哪来、经过什么变换、到哪去

**D.2 理解核心机制**

读 3-5 个最关键的源文件（不是 README，是源码），回答这些问题：

- **核心算法/机制是什么**：它用什么技术手段解决核心问题？不是"用了 LLM"，而是具体怎么用的——prompt 怎么构造的、上下文怎么管理的、输出怎么解析的
- **关键抽象是什么**：核心的 class/interface/type 有哪些、它们之间的关系是什么
- **状态管理怎么做的**：有没有持久化、缓存、session 管理
- **错误处理策略**：失败时怎么兜底、有没有重试机制、降级逻辑

**D.3 识别设计决策和 trade-off**

读代码时注意记录：

- **为什么选择这个方案而不是其他方案**：比如为什么用 CDP 而不是 Selenium？为什么用异步而不是同步？
- **有哪些巧妙的设计**：比如 lazy import、事件总线解耦、插件机制
- **有哪些妥协**：比如为了性能牺牲了什么、为了兼容性增加了什么复杂度

**D.4 评估代码质量（看代码，不是看 star 数）**

- 函数/方法长度是否合理
- 命名是否清晰
- 抽象层次是否一致
- 有没有明显的 code smell（巨型函数、深嵌套、全局状态滥用）
- 测试覆盖的是真正的逻辑还是只是 happy path

### 阶段 E：动手实验（核心阶段，不可跳过）

**进入时更新 progress.phase = "hands_on"**

跑测试只能证明代码没 bug，不能回答"这个东西好不好用"。你需要动手试。

**E.1 核心功能实测**

根据项目类型，写一个最小可运行的 demo 脚本并执行。注意不调用需要付费 API key 的功能，但可以：
- 用本地/免费的替代（如 ollama、mock server）
- 只运行不需要外部 API 的部分功能
- 读 examples/ 目录，选一个最简单的改改跑

```bash
# 写 demo 脚本（在实验目录内）
cat > "$EXP_DIR/try_demo.py" << 'DEMO'
# 根据项目实际情况编写
DEMO

# 执行（超时 2 分钟）
cd "$EXP_DIR/repo"
# macOS 兼容的超时方式
python3 -c "
import subprocess, sys
try:
    subprocess.run([sys.executable, '$EXP_DIR/try_demo.py'], timeout=120)
except subprocess.TimeoutExpired:
    print('TIMEOUT: demo 超时（2分钟）')
" 2>&1 | tee "$EXP_DIR/demo.log"
```

**E.2 探索能力边界**

刻意尝试一些边界场景（至少 2 个），记录结果：
- 大输入/长文本/大文件会怎样
- 异常输入会怎样（空值、特殊字符、非预期类型）
- 并发/多实例场景是否支持
- 没有网络时行为如何
- 缺少某个可选依赖时是否优雅降级

**E.3 性能初探**

如果项目涉及处理延迟敏感的场景：
- 冷启动时间
- 单次操作耗时量级
- 是否有明显的性能瓶颈

**所有实验结果保存到 `$EXP_DIR/demo.log`。**

### 阶段 F：生成结构化评分

**进入时更新 progress.phase = "scoring"**

写入 `experiment.json`。**每个字段都必须填写，不得留空。**

```json
{
  "experiment_id": "2026-03-22-epiral-bb-browser",
  "status": "researched",
  "date": "2026-03-22",
  "source": {
    "post_url": "https://x.com/...",
    "author": "@handle",
    "github_url": "https://github.com/owner/repo",
    "github_found_via": "regex|web_search|redirect|manual|not_found"
  },
  "progress": {
    "phase": "scoring",
    "started_at": "2026-03-22T10:05:00",
    "completed_at": null,
    "error": null,
    "retry_count": 0
  },
  "scores": {
    "install": {
      "success": true,
      "seconds": 23,
      "method": "pip install -e .",
      "notes": "依赖 pnpm，需先 corepack enable"
    },
    "tests": {
      "ran": true,
      "passed": 14,
      "failed": 0,
      "skipped": 2,
      "framework": "pytest"
    },
    "code_quality": {
      "stars": 1572,
      "forks": 166,
      "open_issues": 37,
      "last_push_days_ago": 1,
      "language": "TypeScript",
      "license": "MIT",
      "has_readme": true,
      "has_tests": true,
      "has_ci": true
    },
    "post_consistency": {
      "score": 4,
      "max": 5,
      "explanation": "帖子说 X，实际验证 Y 基本一致，但 Z 功能未实现"
    },
    "usefulness": {
      "score": 4,
      "max": 5,
      "primary_usecase": "项目调研增强，跨站信息抓取",
      "for_user": "可接入 Agent 工作流做登录态网页数据抽取"
    },
    "maturity": {
      "score": 3,
      "max": 5,
      "stage": "early|growing|stable|mature|declining",
      "explanation": "v0.9，API 仍在变化，但社区活跃"
    },
    "technical_depth": {
      "core_mechanism": "用什么技术手段解决核心问题（不是 README 复述，而是你读源码后的理解）",
      "data_flow": "核心数据流：输入 → 经过哪些处理步骤 → 输出",
      "key_abstractions": ["核心 class/interface 列表", "以及它们之间的关系"],
      "clever_designs": ["读源码时发现的巧妙设计", "值得借鉴的模式"],
      "tradeoffs": ["为什么选 A 方案而不是 B", "为此付出了什么代价"],
      "boundary_findings": "动手实验中发现的能力边界——什么场景会失败或表现不好"
    },
    "workflow_integration": {
      "integration_points": ["具体的接入方式1", "具体的接入方式2"],
      "code_agent_relevance": "与用户的 code agent（chatbot + 沙箱）应用的关联分析",
      "adoption_effort": "low|medium|high",
      "adoption_notes": "接入需要做什么、有什么坑"
    },
    "overall_rating": 4,
    "overall_verdict": "recommend_try|worth_watching|wait|skip",
    "one_liner": "把真实浏览器登录态暴露为 CLI/MCP 接口，让 AI agent 能读写无 API 的网站"
  }
}
```

**评分量表说明**：

| 维度 | 1 分 | 2 分 | 3 分 | 4 分 | 5 分 |
|------|------|------|------|------|------|
| post_consistency | 完全不符 | 严重夸大 | 部分一致 | 基本一致 | 完全一致 |
| usefulness | 无用 | 极少场景 | 偶尔有用 | 多场景可用 | 核心工具级 |
| maturity | 玩具/PoC | 早期 alpha | 可用但粗糙 | 生产可用 | 成熟稳定 |

**overall_verdict 判定规则**：
- `recommend_try`：usefulness ≥ 4 且 install.success = true
- `worth_watching`：usefulness ≥ 3 或 maturity ≥ 3
- `wait`：maturity ≤ 2 且 usefulness ≤ 3
- `skip`：usefulness ≤ 2 且 post_consistency ≤ 2

**overall_rating 星级对照**：
| overall_rating | 星级 | verdict |
|---------------|------|---------|
| 4-5 | ⭐⭐⭐⭐ | recommend_try（推荐试用） |
| 3 | ⭐⭐⭐ | worth_watching（值得关注） |
| 2 | ⭐⭐ | wait（建议观望） |
| 1 | ⭐ | skip（不推荐） |

**GitHub API 限流处理**：如果 `api.github.com` 返回 403/429，等待 60 秒重试 1 次。仍然限流则跳过 `code_quality` 部分，标注"GitHub API 限流，code_quality 数据缺失"。

### 阶段 G：写可读报告

**进入时更新 progress.phase = "reporting"**

写入 `research-report.md`。这份报告是调研的核心产出——它的价值不在于搬运 README，而在于**你读完源码、动手试过之后的独立判断**。

**写作原则**：
- **写你从源码中发现的，而不是 README 里写的**。README 人人都能读，你的价值是读了源码之后的洞察。
- **具体到代码**。不说"架构清晰"，说"Agent 类用 15 行状态机管理 6 种交互状态，每个状态只暴露 2-3 个 action"。
- **写边界，不只写能力**。"能做 X"不如"能做 X，但当 Y 时会 Z"有价值。
- **关联用户工作**。用户做全栈 code agent 应用（chatbot + 沙箱），每个洞察都要回答"这对我的工作意味着什么"。

**严格遵循以下模板**：

```markdown
# {project_name}

> {one_liner}

| 维度 | 评分 | 说明 |
|------|------|------|
| 安装 | ✅/❌ | {method}，{seconds}秒 |
| 测试 | {passed}/{total} | {framework} |
| 帖子一致性 | {score}/5 | {explanation} |
| 实用性 | {score}/5 | {primary_usecase} |
| 成熟度 | {score}/5 | {stage} |
| **综合** | **{overall}/5** | **{verdict_cn}** |

## 项目概况

- **GitHub**：{url} | ⭐ {stars} | {language} | {license}
- **解决的核心问题**：{用一段话说清这个项目为什么存在，它在什么场景下比现有方案好}

## 技术原理深度解读

### 核心机制
{不是 README 摘抄。是你读了入口文件和核心模块后理解的——这个系统到底怎么工作的？
用"输入 → 处理步骤1 → 处理步骤2 → 输出"的结构解释核心数据流。
举具体代码：哪个文件的哪个函数/类在做关键的事。}

### 关键设计决策
{读源码时发现的 2-3 个重要设计决策：
- 为什么选了方案 A 而不是 B？
- 这个决策带来了什么好处？付出了什么代价？
- 举例：为什么用 CDP 而不是 Selenium？为什么用 accessibility tree 而不是 raw HTML？}

### 架构亮点
{1-3 个值得借鉴的架构/代码设计模式。
不说"模块化设计好"，说具体怎么模块化的、为什么这种方式比别的好。
每个亮点用 2-3 句话解释清楚原理。}

## 能力边界分析

### 能做什么（验证过的）
{你动手实测确认能做的事，不是 README 声称的。列 3-5 条，每条附实测证据。}

### 做不到什么 / 会失败的场景
{你动手实测或读源码发现的限制。这部分可能比"能做什么"更有价值。
至少列 2-3 条，每条说明：什么场景、为什么会失败、是架构限制还是实现没做。}

### 性能特征
{冷启动耗时、单次操作耗时量级、资源消耗特征。
如果有性能瓶颈，说明瓶颈在哪个环节。}

## 实操验证

### 安装体验
{安装过程中的关键发现和坑，3-5 行}

### 动手实验
{你实际跑了什么 demo/实验，结果如何。
不是"跑了 pytest 145 passed"——那是测试，不是实验。
是"我写了一个脚本让它做 X，发现 Y，说明 Z"。}

## 工作流集成分析

### 与 code agent 应用的结合点
{具体分析这个项目如何接入用户的全栈 code agent（chatbot + 沙箱）应用。
不是泛泛地说"可以接入"，而是：
- 具体的接入方式（API 调用？SDK 嵌入？MCP？CLI 包装？）
- 接入后能解决什么具体问题
- 需要做什么适配工作
- 有什么潜在的坑}

### 对 Agent 架构设计的启发
{从这个项目的设计中学到了什么对 Agent 架构有启发的东西。
要具体：不说"设计很好"，说"它的 X 模式解决了 Y 问题，我们的 Agent 在 Z 场景也有类似需求，可以借鉴"。
每个启发 3-5 句话，关联到实际工作场景。}

## 结论

{verdict_cn}。{2-3 句总结：这个项目最大的价值是什么、最大的限制是什么、建议怎么用。}
```

**verdict_cn 对照**：
- recommend_try → "⭐⭐⭐⭐ 推荐试用"
- worth_watching → "⭐⭐⭐ 值得关注"
- wait → "⭐⭐ 建议观望"
- skip → "⭐ 不推荐"

---

## 第四步：更新汇总 + 推送

**进入时更新 progress.phase = "done"，设置 completed_at**

### 更新 pending.json（如果是 dispatch 触发）

如果当前任务来自 pending.json，将该任务的 `"status"` 更新为 `"done"`，添加 `"completed_at"` 时间戳。

### 更新 EXPERIMENTS.md（derived view）

EXPERIMENTS.md 是从各 experiment.json 生成的汇总视图，不是独立数据源。追加一行：

```
| {date} | {id} | @{author} | [{repo}]({url}) | {install_icon} | {consistency}/5 | {usefulness}/5 | {maturity}/5 | ⭐{stars} | {verdict_cn} |
```

如果文件不存在，先创建表头：
```
| 日期 | 实验ID | 来源 | GitHub | 安装 | 一致性 | 实用性 | 成熟度 | Stars | 结论 |
|------|--------|------|--------|------|--------|--------|--------|-------|------|
```

### 写入飞书文档（核心产出）

**这是用户阅读调研报告的主要入口。** 本地 Markdown 只是备份，飞书文档才是给人看的。

飞书调研报告文档 ID 从环境变量 `X_LAB_FEISHU_DOC_ID` 获取。如果未配置，跳过飞书文档写入，在飞书消息中说明。

**写入方式**：使用飞书文档 Block API（`/open-apis/docx/v1/documents/{doc_id}/blocks/{doc_id}/children`）。

**飞书文档 ID**：从环境变量 `X_LAB_FEISHU_DOC_ID` 获取

**文档权限处理**：
- 首次写入时如果返回权限错误（403/permission denied）→ 创建新文档：`POST /open-apis/docx/v1/documents`，body `{"title": "x-lab 调研报告", "folder_token": ""}`
- 创建成功后，立即授权用户访问：`POST /open-apis/drive/v1/permissions/{document_id}/members?type=docx`，body `{"member_type": "openchat", "member_id": "{DAILY_X_SIGNAL_FEISHU_RECEIVE_ID}", "perm": "full_access"}`
- 将新文档 ID 写入 `.env.local`（更新 `X_LAB_FEISHU_DOC_ID`），后续调研复用
- 如果授权 API 也失败，仍然继续写入（文档创建者即 App 有写入权限），在推送消息中告知用户手动获取文档链接

**文档结构**：所有调研报告追加到同一个文档。以日期为 H1 大标题，项目名为 H2 小标题。同一天的多个调研共享同一个日期标题。

**写入流程**：

1. 获取 `tenant_access_token`（从 `FEISHU_APP_ID` + `FEISHU_APP_SECRET`）
2. 读取文档现有 blocks，检查今天的日期 H1 是否已存在
3. 如果今天的日期 H1 不存在 → 先写入分割线 + 日期 H1，再追加项目 H2 + 内容
4. 如果今天的日期 H1 已存在 → 在该日期下追加项目 H2 + 内容
5. 新内容追加到文档末尾（`index: -1`）

**Block type 映射**（飞书 docx API v1）：

| 类型 | block_type | 字段名 |
|------|-----------|--------|
| 文本 | 2 | `text` |
| H1 标题 | 3 | `heading1` |
| H2 标题 | 4 | `heading2` |
| H3 标题 | 5 | `heading3` |
| 无序列表 | 12 | `bullet` |
| 有序列表 | 13 | `ordered` |
| 分割线 | 22 | `divider` |

**每个 block 的固定结构**（text/heading/bullet/ordered 通用）：
```json
{
  "block_type": 2,
  "text": {
    "elements": [{"text_run": {"content": "内容", "text_element_style": {}}}],
    "style": {}
  }
}
```

**注意**：
- 每批最多写 8 个 blocks（避免 API 超时）
- bullet 内容中避免完整 URL（`https://github.com/xxx`），改用 `github.com/xxx` 或纯文本描述
- 分割线 block 只需 `{"block_type": 22, "divider": {}}`

**写入内容结构**（每个调研项目）：

```
分割线（仅当需要新增日期标题时）
H1: {date}（仅当该日期不存在时）
H2: {project_name} — {one_liner 简短版}
文本: 综合评分：{verdict_cn}（{overall}/5）
H3: 评分卡
  bullet: 安装：✅/❌（{method}，{seconds}秒）
  bullet: 测试：{passed}/{total} passed
  bullet: 帖子一致性：{score}/5 — {explanation}
  bullet: 实用性：{score}/5 — {primary_usecase}
  bullet: 成熟度：{score}/5 — {stage}
H3: 技术原理
  bullet: 核心机制：{core_mechanism 一段话}
  bullet: 数据流：{data_flow}
  bullet: 关键设计决策：{tradeoffs 中最重要的 1-2 条}
H3: 架构亮点
  ordered: {clever_designs 中的 2-3 条，每条含原理解释}
H3: 能力边界
  bullet: ✅ {验证过能做的 2-3 条}
  bullet: ❌ {做不到/会失败的 2-3 条，含原因}
H3: 动手实验
  bullet: {demo 实验过程和发现，3-5 条}
H3: 工作流集成
  ordered: {与 code agent 应用的 2-3 个结合点，含具体接入方式}
H3: Agent 架构启发
  ordered: {2-3 个具体启发，关联实际工作场景}
H3: 结论
  文本: {verdict_cn}。{2-3 句话总结}
```

**飞书文档写入失败时**：不中断流程，在飞书推送消息中标注"飞书文档写入失败，完整报告见本地文件"。

### 飞书推送消息

调研完成后发飞书消息。**这条消息是摘要通知**，完整报告在飞书文档里。

**格式固定如下**：

```
🔬 调研完成：{project_name}

{verdict_cn}（{overall}/5）

📋 {one_liner}
🏷️ ⭐{stars} | {language} | {license} | 更新{days}天前

🔍 核心发现
━━━━━━━━━━━━
技术原理：{core_mechanism 一句话精华}
关键设计：{最重要的 1 个 tradeoff}
能力边界：{最关键的 1 个限制}

🧪 动手验证
━━━━━━━━━━━━
安装：{✅/❌} ({seconds}秒)
测试：{passed}/{total} passed
实测：{demo 实验最重要的 1 个发现}

🔗 与 Code Agent 的结合
━━━━━━━━━━━━
1. {最佳接入点}
2. {次佳接入点}

🧠 架构启发：{最有价值的 1 个 insight，一句话}

📄 完整报告：https://larkoffice.com/docx/{X_LAB_FEISHU_DOC_ID}
```

如果飞书文档写入失败，最后一行改为：
```
📁 完整报告（本地）：~/.openclaw/workspace/x-lab/experiments/{id}/research-report.md
```

---

## 错误处理

| 失败场景 | 处理方式 |
|---------|---------|
| Clone 失败（网络） | 重试 1 次（间隔 5 秒），仍失败则基于 web 信息评价 |
| 安装失败 | 记录错误，尝试不安装也能验证的部分（读代码结构、跑 --help） |
| 安装超时（5 分钟） | 标注超时，给出已完成部分的评分 |
| 测试失败 | 记录失败数量和原因，不影响其他评分 |
| 测试超时（3 分钟） | 标注"部分测试"，基于已执行的测试评估 |
| GitHub API 限流 | 等待 60 秒重试 1 次，仍限流则跳过 code_quality |
| GitHub URL 找不到 | 基于帖子内容 + web 搜索结果完成评价，标注"未找到仓库" |
| Dockerfile 项目 | 标注"需要 Docker 环境"，install.success=false，基于代码结构评价 |
| web_search 不可用 | 尝试 web_fetch t.co 短链展开，仍失败则跳过 |
| 飞书文档 ID 未配置 | 跳过飞书文档写入，在推送消息中附本地文件路径 |
| 飞书文档写入失败 | 不中断，推送消息中标注失败并附本地文件路径 |
| 飞书文档读取失败（get_document） | 直接写入新内容（不追加），不中断 |
| **任何阶段出错都不中断**，更新 progress.error 后跳到下一阶段继续 |
| **绝不因为某个阶段失败就停止或问人** |

---

## 行为准则

- **不问人**：整个调研过程中不向用户确认任何事情，遇到问题自己决策
- **诚实评价**：帖子吹得天花乱坠但实际一般，就如实说。star 数高不等于好用
- **读源码是必做项**：不读源码就不可能写出有价值的技术分析。阶段 D（源码深度阅读）不可跳过
- **动手实验是必做项**：只跑 pytest 不算"试过了"。阶段 E（动手实验）不可跳过
- **用户视角**：用户是产品经理，做全栈 code agent 应用（chatbot + 沙箱）——每个洞察都要关联这个背景
- **不复读 README**：报告的价值在于你读了源码、动手试过之后的独立判断，不是搬运文档
- **写边界比写能力更重要**："能做 X"人人都知道（README 写了），"做不到 Y"才是调研的独特价值
- **具体到代码**：不说"架构清晰"，说"Agent 类用状态机管理交互状态，核心循环在 agent/loop.py:45"
- **可操作的集成建议**：不说"可以接入"，说"通过 X 方式接入，需要适配 Y，预计工作量 Z"
- **控制篇幅但不牺牲深度**：report 1000-2000 字正文（不含表格和代码），飞书消息不超过 400 字
- **进度追踪**：每完成一个阶段都更新 experiment.json 的 progress 字段，支持断点续研
