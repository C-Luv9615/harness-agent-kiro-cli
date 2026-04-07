---
name: session-handoff
description: >
  Harness 会话接力：检测项目状态，恢复上下文，定位当前阶段。每次新会话开始时自动触发。
  通过 .harness/ 和 .specify/ 的文件状态机械判断是接力还是全新开始，不依赖 AI 记忆。
---

# Session Handoff — 会话接力

每次新会话开始时，通过文件状态判断是接力已有工作还是全新开始。

## 流程

### Step 1: 检测项目状态

检查以下文件是否存在：

```bash
ls .harness/PROGRESS.md .harness/lessons-learned.md 2>/dev/null
# 检查 SDD 制品
ls .specify/*/spec.md .specify/*/plan.md .specify/*/tasks.md 2>/dev/null
```

### Step 2: 判断会话类型

SDD 制品统一路径：`.specify/{feature}/spec.md`、`plan.md`、`tasks.md`。

| .harness/ | SDD 制品 | 判断 |
|-----------|---------|------|
| 不存在 | — | 全新项目，提示 `init` |
| 存在 | 无 spec.md | 已初始化但未规划，提示 `specify` |
| 存在 | 有 spec.md 无 tasks.md | SDD 进行中，定位到未完成阶段 |
| 存在 | 有 tasks.md | 检查 task 完成状态 → Step 3 |

### Step 3: 定位当前 Task

读取 `.specify/tasks.md`，扫描 checkbox 状态：

- 有 `[ ]`（未完成 task）→ 阶段 2（TDD 执行），定位到第一个未完成 task
- 全部 `[x]` → 阶段 5（收尾），提示 `evolve`
- 有 `[~]` 或标记 `[COMPLETED BUT CHANGED]` → 需要回归的 task

### Step 4: 加载上下文

按 agent.md 上下文加载策略：

1. 读 `.harness/PROGRESS.md` — 了解整体进度和工作区状态（`## 工作区` 区域）
2. 读 `.harness/lessons-learned.md` — 加载历史经验
3. 读当前 task 对应的设计章节（从 spec.md/plan.md 中定位）

若 PROGRESS.md 中记录了隔离工作区（类型非 none），验证路径是否仍存在：
```bash
ls {worktree_path} 2>/dev/null || echo "STALE: worktree 路径不存在"
```
路径不存在时在报告中标注 `⚠️ 工作区路径已失效，可能已被手动清理`。

### Step 5: 验证环境

```bash
cd {项目目录} && bash .harness/verify.sh
```

- GATE PASSED → 环境正常
- GATE FAILED → 报告失败项，可能上次会话中断时留下了未修复的问题

### Step 6: 报告并询问

向用户展示状态摘要：

```
📍 会话接力 — {项目名}
  当前阶段：{阶段 N}
  当前 Task：{Task ID} — {描述}
  工作区：{worktree 类型 + 分支 + 路径 | "当前分支（未隔离）"}
  verify.sh：{PASSED/FAILED}
  经验条目：{N} 条

继续当前任务，还是开启新的工作？
```

- 用户确认继续 → 加载当前 task 设计章节，按 TDD 流程推进
- 用户说"新的" → 询问走哪条路径（fix / specify / brainstorm）
- 用户无明确回复 → 默认继续当前任务

## 全新项目时的输出

```
📍 未检测到 .harness/ 目录 — 这是一个全新项目。
  请先执行 `init` 初始化项目结构。
```

## Key Rules

- **机械判断，不靠记忆** — 只通过文件存在性和 checkbox 状态判断，不猜测
- **默认接力** — 用户未明确说"新的"就认为是接力
- **verify.sh 先行** — 恢复上下文后先跑一次验证，确认环境状态
