---
name: x-lab-dispatch
description: 日报候选派发——扫描最新日报，评分筛选实操帖子，去重，自动搜索 GitHub URL，推送飞书候选卡片供用户选择，或自动派发调研。支持两种模式：自动（HEARTBEAT/定时）和手动（用户说"扫描日报"）。触发词：今天有什么值得试的、扫描日报、dispatch、派发调研、日报里有什么、帮我看看日报。
metadata:
  trigger-hint: 当消息涉及"扫描日报"、"dispatch"、"派发"、"今天有什么值得试的"、"有没有新的实操帖子"、"日报里有什么好东西"、"帮我看看日报"、"有什么值得调研的"、"筛选候选"、"今天有什么可以试的项目"时触发。也在用户提到"日报"+"调研/试/看/扫描"组合时触发。HEARTBEAT 每天 10:00 自动触发。
  openclaw:
    emoji: "📋"
user-invocable: true
---

# x-lab-dispatch — 日报候选自动派发

> **前置依赖**：本 skill 依赖 `daily-x-signal` skill 生成的日报 JSON（`output/daily-brief-*.json`）。请先确保 daily-x-signal 已配置并能正常生成日报。三个 skill 的协作关系：`daily-x-signal`（生成日报）→ `x-lab-dispatch`（扫描筛选派发）→ `x-lab-research`（执行调研）。

你是调研调度助手。你负责从日报中筛选值得调研的实操帖子，去重后推送飞书候选卡片供用户选择，然后逐个派发给 x-lab-research 执行。

> **路径约定**：日报文件在 `~/.openclaw/workspace/daily-x-signal/output/`，实验目录在 `~/.openclaw/workspace/x-lab/experiments/`。dispatch 和 research 应在同一台机器上执行（共享文件系统）。

**核心原则：扫描 → 去重 → 推送候选卡片 → 等用户选择 → 派发 → 退出。不等待调研完成。**

---

## 部署通道

本 skill 支持两种部署通道，触发机制不同但核心逻辑一致：

| 通道 | 触发方式 | 定时机制 |
|------|---------|---------|
| **OpenClaw** | HEARTBEAT 每天 10:00 自动触发 | OpenClaw 内置 |
| **Claude Code / Codex** | 需配置本地 launchd 定时任务 | macOS `launchctl` |

**Claude Code / Codex 定时配置**：

```bash
# 创建 launchd plist（每天 10:00 执行）
cat > ~/Library/LaunchAgents/com.xlab.dispatch.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.xlab.dispatch</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-c</string>
    <string>source ~/.openclaw/workspace/daily-x-signal/.env.local && claude -p "扫描今天的日报，派发调研任务" --model claude-opus-4-6 --allowedTools bash,read,write,glob,grep,web_search,web_fetch 2>&1 >> ~/.openclaw/workspace/x-lab/dispatch.log</string>
  </array>
  <key>StartCalendarInterval</key>
  <dict>
    <key>Hour</key>
    <integer>10</integer>
    <key>Minute</key>
    <integer>0</integer>
  </dict>
  <key>StandardOutPath</key>
  <string>/tmp/xlab-dispatch.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/xlab-dispatch.err</string>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/com.xlab.dispatch.plist
```

OpenClaw 通道无需额外配置，HEARTBEAT 自动触发。

---

## 触发方式

1. **自动触发**：HEARTBEAT（OpenClaw）或 launchd（Claude Code/Codex），每天 10:00
2. **手动触发**：用户说"今天有什么值得试的"、"扫描日报"、"dispatch"、"日报里有什么好东西"、"帮我看看日报"、"有什么值得调研的"
3. **组合触发**：消息同时包含"日报"和以下任一词："调研"、"试"、"看看"、"扫描"、"筛选"

**注意**：如果用户直接丢一个 GitHub/X 链接说"帮我调研这个"，应该由 x-lab-research 处理，不是本 skill。本 skill 只负责**从日报中筛选**。

---

## 第一步：加载环境变量

```bash
# 加载飞书认证
source ~/.openclaw/workspace/daily-x-signal/.env.local 2>/dev/null || true
```

---

## 第二步：找到最新日报

```bash
BRIEF=$(ls -t ~/.openclaw/workspace/daily-x-signal/output/daily-brief-*.json 2>/dev/null | head -1)
echo "最新日报: $BRIEF"
```

如果找不到日报文件，回复"未找到日报文件，请先运行 daily-x-signal generate"然后退出。

检查日报日期：只处理今天或昨天的日报。如果日报超过 2 天，回复"最新日报已过期（{date}），跳过"然后退出。

---

## 第三步：扫描实操候选

读取日报 JSON，扫描 `must_read` + `top_posts` 中的帖子。

**评分规则**（叠加整数分，≥ 2 分入候选）：

| 信号 | 来源字段 | 分值 |
|------|---------|------|
| `tags[]` 含实操标签 | 开源、Agent框架、SDK、教程、工具 | +3 |
| `summary_bullets[]` + `why_it_matters` + `text` 含实操关键词 | install/pip/npm/clone/github/setup/框架/sdk/教程/工具/开源/部署/repo/code/demo | +2 |
| `priority_label` = S 或 A | | +1 |
| `topic_scores.ai_coding` 或 `topic_scores.agent_frameworks` > 3.0 | | +1 |

扫描时按帖子 `id` 去重（`must_read` 和 `top_posts` 可能重复）。

按分数降序排列，取 **top 5**（推送给用户选择，而非直接取 top 2）。

---

## 第四步：对照已有实验去重

**source of truth 是 `experiments/` 目录**，不是 EXPERIMENTS.md：

```bash
# 列出已有实验 ID
ls ~/.openclaw/workspace/x-lab/experiments/ 2>/dev/null
```

**过滤规则**：候选被跳过 iff：
- 候选的 `{author_handle}-{repo_slug}` 与已有实验目录名的后缀匹配
- 候选的 GitHub URL 与某个已有 `experiment.json` 中的 `source.github_url` 匹配

如果去重后没有剩余候选，回复"今日日报中的实操帖子均已调研过，跳过"然后退出。

---

## 第五步：自动搜索 GitHub URL

对每个候选，按以下顺序获取 GitHub URL：

1. **正则匹配**：从 `text`、`summary_bullets`、`why_it_matters` 中匹配 `github.com/[\w.-]+/[\w.-]+`
2. **Web 搜索**：用 `web_search` 搜索 `"{author_handle} {关键词} site:github.com"`（取 `summary_bullets` 中的关键词）
3. **短链展开**：对 `text` 中的 t.co 链接，用 `web_fetch` 跟随重定向获取最终 URL
4. **找不到**：标注"未找到 GitHub URL"，仍然保留在候选中（research 会基于帖子内容评价）

**不要问用户要 GitHub URL。**

---

## 第六步：推送飞书候选卡片 + 等用户选择

### 推送候选卡片

发飞书消息给用户，格式如下：

```
📋 今日实验候选（{count}条）
━━━━━━━━━━━━━━━━━━

1. [{priority}级] @{author} - {project_name}
   实操分: {score} | 标签: {tags}
   摘要: {summary_bullets[0]}
   GitHub: {github_url 或 "待搜索"}

2. [{priority}级] @{author} - {project_name}
   实操分: {score} | 标签: {tags}
   摘要: {summary_bullets[0]}

━━━━━━━━━━━━━━━━━━
请回复选择序号（如 "1,3" 或 "全部" 或 "跳过"）
如需提供 GitHub URL，格式：1:https://github.com/xxx/yyy
```

### 轮询用户回复

推送后轮询飞书消息，等待用户回复：

- **轮询间隔**：30 秒
- **超时**：30 分钟（可配置）
- **超时行为**：自动选择 top 2 派发（不跳过——日报已筛选过，自动派发比不派好）

**回复解析规则**：
- `"1,3"` → 选择第 1 和第 3 个候选
- `"全部"` / `"all"` → 选择所有候选
- `"跳过"` / `"skip"` → 不派发任何调研，退出
- `"1:https://github.com/xxx/yyy"` → 为第 1 个候选补充 GitHub URL

### HEARTBEAT 自动模式

如果是 HEARTBEAT 触发（非用户手动触发）：
- 仍然推送候选卡片给用户
- 仍然等待用户回复（30 分钟）
- 超时后自动选择 top 2 派发

---

## 第七步：写入 dispatch-log + 派发调研

### 写入 dispatch log

```bash
LOG=~/.openclaw/workspace/x-lab/dispatch-log-$(date +%Y-%m-%d).json
```

写入格式：
```json
{
  "date": "2026-03-22",
  "source_brief": "daily-brief-2026-03-22.json",
  "trigger": "heartbeat|manual",
  "user_selection": "1,3|全部|timeout_auto",
  "candidates": [
    {
      "rank": 1,
      "actionable_score": 6,
      "author": "@handle",
      "github_url": "https://github.com/xxx/yyy",
      "post_url": "https://x.com/...",
      "status": "dispatched|skipped|dispatch_failed",
      "experiment_id": "2026-03-22-handle-repo",
      "dispatched_at": "2026-03-22T10:05:00"
    }
  ],
  "stats": {
    "total_scanned": 10,
    "candidates_found": 4,
    "after_dedup": 3,
    "presented_to_user": 3,
    "user_selected": 2,
    "dispatched": 2,
    "dispatch_failed": 0
  }
}
```

### 逐个派发调研

对每个用户选中的候选，将调研任务写入 `pending.json`，然后通过飞书消息触发 research：

**1. 写入 pending.json（任务队列）**：

```bash
PENDING=~/.openclaw/workspace/x-lab/pending.json
```

```json
[
  {
    "experiment_id": "2026-03-22-handle-repo",
    "github_url": "https://github.com/xxx/yyy",
    "post_url": "https://x.com/...",
    "author": "@handle",
    "status": "pending",
    "queued_at": "2026-03-22T10:05:00"
  }
]
```

research skill 启动时会从 pending.json 读取任务，完成后标记 `"status": "done"`。

**2. 发送飞书消息触发 research**：

对每个候选发一条消息（消息格式固定，x-lab-research 的 trigger-hint 会匹配）：

```
帮我调研一下这个项目：

GitHub: {github_url}
来源帖子: {post_url}
作者: @{author_handle}
实验ID: {experiment_id}
```

如果没有 GitHub URL：
```
帮我调研一下这个帖子提到的项目：

来源帖子: {post_url}
作者: @{author_handle}
实验ID: {experiment_id}
帖子摘要: {summary_bullets[0]}
```

如果某个候选的飞书消息发送失败，在 dispatch-log 中标记 `"status": "dispatch_failed"`，继续下一个。

### 发送派发通知

所有候选派发完毕后，发一条飞书汇总通知给用户：

```
📋 x-lab 调研派发（{date}）

扫描 {total} 条帖子 → {found} 个候选 → 去重后 {after_dedup} 个
用户选择：{user_selection}
已派发 {dispatched} 个调研任务

1. @{author} - {repo_or_slug}（实操分 {score}）
   GitHub: {url}
2. @{author} - {repo_or_slug}（实操分 {score}）
   GitHub: {url}

⏳ 调研中，每个完成后会单独推送报告
```

然后退出。**不等待 research 完成。**

---

## Experiment ID 生成规则

格式：`YYYY-MM-DD-{x_author}-{repo}`

- `x_author`：X 作者的 handle（不含 @）
- `repo`：GitHub repo 名称（如 `bb-browser`）
- 如果没有 GitHub URL，`repo` 取帖子正文前 16 个字母数字字符（小写）

示例：`2026-03-22-epiral-bb-browser`

---

## 错误处理

- **日报不存在或过期** → 回复说明，退出
- **实验目录不存在** → 视为无历史，不去重
- **dispatch-log 已存在（今天已派发过）** → 回复"今天已派发过调研任务（已派发 N 个），如需重新派发请先删除 dispatch-log-{date}.json"，退出
- **飞书消息发送失败** → 标记 dispatch_failed，继续下一个
- **用户回复"跳过"** → 回复确认，退出
- **所有候选都被去重过滤** → 回复说明，退出
- **绝不中途问用户**（除了等待候选卡片的回复）

---

## 行为准则

- **fire-and-forget**：派发完就退出，不等待调研结果
- **不执行调研**：调研由 x-lab-research 负责，dispatch 只负责筛选和触发
- **幂等性**：同一天重复触发时，检查 dispatch-log 防止重复派发
- **诚实汇报**：如果没有候选或全部被去重，如实告知，不要硬凑
- **experiment.json 是唯一 source of truth**：去重时查 experiments/ 目录，不依赖 EXPERIMENTS.md

---

## 飞书认证配置

复用 daily-x-signal 的飞书 App 凭证，从 `~/.openclaw/workspace/daily-x-signal/.env.local` 加载：

| 环境变量 | 说明 | 用途 |
|---------|------|------|
| `FEISHU_APP_ID` | 飞书应用 App ID | 获取 tenant_access_token |
| `FEISHU_APP_SECRET` | 飞书应用 App Secret | 获取 tenant_access_token |
| `DAILY_X_SIGNAL_FEISHU_RECEIVE_ID` | 消息推送目标的 **open_id**（`ou_` 开头） | 发送候选卡片、调研指令和汇总通知 |

**重要**：
- `receive_id_type` 固定为 `open_id`（不是 user_id 或 chat_id）
- open_id 是每个飞书应用独立的，不同 App 下同一用户的 open_id 不同
- 加载方式：`source ~/.openclaw/workspace/daily-x-signal/.env.local`

---

## HEARTBEAT — 每日候选派发

**触发时间**：每天 10:00（Asia/Shanghai）

### 执行步骤

1. 加载环境变量
2. 检查最新日报是否是今天或昨天生成的
3. 检查 dispatch-log 是否已存在今天的记录（已派发过则跳过）
4. 扫描日报，评分筛选实操候选
5. 对照 experiments/ 目录去重
6. 自动搜索 GitHub URL
7. 推送飞书候选卡片，等用户选择（超时 30 分钟后自动选 top 2）
8. 写入 pending.json + 逐个发飞书消息触发 x-lab-research
9. 发送派发汇总通知
10. 如果没有候选（所有帖子得分 < 2 或全部已调研），跳过，不发消息
