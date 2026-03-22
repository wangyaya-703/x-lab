# x-lab — X日报实操调研自动化

从 X 日报中筛选值得动手试的项目，自动完成 clone → 安装 → 测试 → 评分 → 报告，零人工干预。

## Skills

本仓库包含两个协作 skill，运行在 [OpenClaw](https://github.com/anthropics/openclaw) 或 Claude Code 环境中：

| Skill | 职责 | 触发方式 |
|-------|------|---------|
| **x-lab-dispatch** | 扫描日报 → 评分筛选 → 去重 → 派发调研任务 | HEARTBEAT 每天 10:00 / 手动 |
| **x-lab-research** | 接收调研指令 → clone/安装/测试/评分/写报告 | dispatch 自动触发 / 手动 |

## 流程

```
daily-x-signal (生成日报)
        ↓
x-lab-dispatch (扫描筛选派发)
        ↓ 飞书消息触发
x-lab-research (全自动调研)
        ↓
飞书文档 + 飞书消息推送
```

## 前置依赖

- **[daily-x-signal](https://github.com/wangyaya-703/daily-x-signal)**：生成日报 JSON（`output/daily-brief-*.json`），dispatch 从中筛选候选
- **飞书 App**：需要 `FEISHU_APP_ID` + `FEISHU_APP_SECRET` 环境变量（复用 daily-x-signal 的飞书配置）
- **飞书文档**：调研报告写入飞书云文档，需要 `X_LAB_FEISHU_DOC_ID` 环境变量

## 安装

将 `skills/` 目录下的两个 skill 复制到 OpenClaw 或 Claude Code 的 skill 目录中即可。

## 配置

在 daily-x-signal 的 `config/default.yaml` 中已包含 `lab:` 配置段：

```yaml
lab:
  enabled: false
  ssh_host: mac-mini
  ssh_timeout_sec: 60
  remote_base: "~/.openclaw/workspace/x-lab"
  auto_run_setup: false
  max_candidates_display: 10
  min_actionable_score: 2
  feishu_poll_timeout_sec: 1800
  feishu_poll_interval_sec: 30
  feishu_report_doc_id: null
  feishu_report_doc_id_env: X_LAB_FEISHU_DOC_ID
  luna_receive_id: null
  luna_receive_id_type: open_id
```

在 `.env.local` 中设置实际值：

```bash
export FEISHU_APP_ID="your_app_id"
export FEISHU_APP_SECRET="your_app_secret"
export X_LAB_FEISHU_DOC_ID="your_doc_id"
```

## 工作目录

调研结果存储在：

```
~/.openclaw/workspace/x-lab/
├── experiments/
│   └── {experiment_id}/
│       ├── experiment.json      # 结构化评分
│       ├── research-report.md   # 可读报告
│       ├── install.log
│       ├── test.log
│       └── repo/                # clone 的仓库
├── EXPERIMENTS.md               # 汇总表
└── queue.json                   # 派发队列
```

## License

MIT
