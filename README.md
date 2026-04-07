# Harness Engineering

AI-driven development workflow for embedded C/NuttX projects. Combines SDD (Spec-Driven Design), TDD (Test-Driven Development), structured code review, and knowledge compounding into a closed-loop engineering process.

## Directory Structure

```
harness-agent/
├── install.sh              ← 一键安装（支持 kiro-cli / claude）
├── skills/                 ← 共享 skill（两个平台通用，19 个）
│   ├── session-handoff/
│   ├── vela-harness/
│   ├── harness-lessons/
│   └── ...
├── kiro/                   ← Kiro CLI 配置
│   ├── harness.agent.md    # Agent 规则（宪法）
│   └── harness.json        # Agent 配置（resources、MCP、tools）
├── claude-code/            ← Claude Code 插件
│   ├── .claude-plugin/
│   │   └── plugin.json     # 插件元数据
│   ├── CLAUDE.md           # 核心规则（等价于 harness.agent.md）
│   ├── AGENTS.md
│   ├── commands/           # 斜杠命令（/fix, /specify, /brainstorm...）
│   └── LICENSE
└── README.md
```

## Quick Start

```bash
git clone <repo> harness-agent
cd harness-agent

# Kiro CLI 用户
bash install.sh --target kiro-cli

# Claude Code 用户
bash install.sh --target claude
```

更新到最新版：

```bash
bash install.sh --target <kiro-cli|claude> --update
```

`--update` 会 `git pull` 最新代码并重新安装，kiro 的飞书 MCP URL 会自动保留。

Kiro 安装：symlink skills 到 `~/.kiro/skills/`，复制 agent 配置到 `~/.kiro/agents/`，交互式配置飞书 MCP。

Claude 安装：symlink 共享 skills 到 `claude-code/skills`，自动注册为全局插件，之后直接启动 `claude` 即可。

两个平台共享同一份 skills，改一处两边生效。

## Philosophy

**structural**（永久）：
1. **确定性 > 概率性** — 能用工具约束的不用 prompt
2. **地图 > 百科全书** — 只给索引按需加载
3. **只解决已发生的问题** — 只记录实际踩过的坑
4. **知识复利** — 每次踩坑沉淀，每次开工检索
5. **只改实现不改测试** — 测试是真理
6. **证据 > 断言** — 每个完成声明必须附带命令输出

**compensatory**（随模型进步可移除）：
7. **人在环中** — 每阶段暂停确认
8. **卡住就停** — 同一问题改 3 次不过就停

## Workflow

```
                         ┌─── fix ──────────────────────┐
                         │                               │
session-handoff → init → ├─── specify → SDD 规划 ───────┤
                         │    (spec→[clarify]→plan→tasks)│
                         └─── brainstorm → specify ──────┘
                                                         │
                                                    [worktree]
                                                         │
                              ┌───────────────────────────┘
                              ↓
                    ┌── TDD 执行（每个 task）──┐
                    │  经验检索 → RED → GREEN   │
                    │  → 重构 → 回顾验证        │←─ 卡住 → 回到设计文档
                    └──────────┬───────────────┘
                               ↓
                    verify（build→run→test→parse→fix）
                               ↓
                    review → commit
                               ↓
                         还有 task？─→ 是 → 回到 TDD
                               ↓ 否
                    evolve（沉淀→刷新→审视规则→分支收尾）
```

## Three Paths

| Command | When | Flow |
|---------|------|------|
| `fix <desc>` | Small change, clear goal | experience search → TDD → verify → review → commit → lessons |
| `specify <desc>` | Large change, clear goal | SDD → worktree → TDD → verify → review → commit → evolve |
| `brainstorm <desc>` | Large change, unclear goal | explore design → transition to specify |

## Commands (14)

| Command | Description |
|---------|-------------|
| `init` | Initialize `.harness/` and `.specify/` |
| `fix` | Quick fix path |
| `specify` | Full SDD path |
| `brainstorm` | Design exploration |
| `clarify` | Requirement clarification (optional, before plan) |
| `plan` | Technical implementation plan |
| `tasks` | Task breakdown |
| `implement` | TDD implementation (with worktree check) |
| `review` | Code review |
| `verify` | Build + test gate |
| `evolve` | Wrap up + knowledge compounding + branch cleanup |
| `search` | Search lessons-learned |
| `status` | Show progress |
| `loop` | Full workflow with pause at each stage |

## Skills (19)

| Skill | Description |
|-------|-------------|
| `session-handoff` | Session continuity — detect project state, resume or start fresh |
| `vela-harness` | Automated build→run→test→parse→fix verification loop |
| `vela-build` | NuttX/Vela build system (CMake+Ninja, Makefile) |
| `vela-run` | Run on QEMU, FVP, simulator, or real hardware |
| `vela-review` | Multi-dimension code review |
| `harness-lessons` | Structured experience capture with staleness detection and refresh |
| `unit-test-gen` | Auto-detect framework, generate unit tests for any language |
| `test-driven-development` | RED-GREEN-REFACTOR discipline for any language |
| `cmocka-unit-test` | Generate cmocka unit tests for NuttX/Vela C code |
| `gtest-unit-test` | Generate gtest + gmock unit tests for NuttX/Vela C++ code |
| `interactive-form` | Browser-based rich form for structured user input |
| `executor` | Persistent interactive CLI process management |
| `git-commit` | NuttX-style commit with checkpatch validation |
| `git-worktree` | Git worktree for isolated parallel development |
| `repoworktree` | Worktree for Google repo multi-repo projects |
| `spec-design` | Requirements → structured design doc with task breakdown |
| `experience-summarize` | Distill bug-solving into reusable experience documents |
| `ce-brainstorm` | Collaborative design exploration |
| `mispec.change-propagation` | Cascade requirement changes across spec/plan/tasks |

## Prerequisites

- **mispec CLI**: `pip install mispec` (for SDD planning)
- **executor-mcp**: MCP server for persistent process management
- **cmocka**: C unit testing framework (for NuttX targets)

## License

MIT
