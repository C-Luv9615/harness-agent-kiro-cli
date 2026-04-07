---
name: vela-harness
description: >
  Automated test harness: build → run → test → parse results → feedback loop.
  Orchestrates build, run, test skills into a closed-loop verification pipeline.
  Auto-detects test framework from project context (cmocka, gtest, pytest, jest,
  vitest, mocha, cargo test, go test, node test runner, etc.).
  Use when user says "harness", "验证闭环", "跑测试", "run tests",
  "test loop", "自动验证", "build and test", "编译跑测试", or after code generation needs
  automated verification. Also triggers when spec-design tasks need execution with verification.
---

# Harness — Automated Build-Test-Verify Loop

Orchestrate existing skills into a closed-loop: code change → build → run → test → parse → fix/pass.

## Core Concept

```
┌─────────────┐     ┌───────────┐     ┌──────────┐     ┌─────────────┐
│ Code Change  │────▶│   Build   │────▶│   Run    │────▶│ Parse Result│
└─────────────┘     └───────────┘     └──────────┘     └──────┬──────┘
       ▲                                                       │
       │              ┌──────────┐                             │
       └──────────────│ AI Fix   │◀────────────────────────────┘
         if FAIL      └──────────┘         FAIL → feedback
                                           PASS → done
```

Human stays in the loop at two points:
1. **Before fix** — AI proposes fix, user approves
2. **After max retries** — AI reports what it tried, user decides next step

## Input

Required:
- **test_cmd**: Test command or test file/module (e.g., `cmocka_test_sched_timer`, `pytest tests/`, `npx jest`)

Optional:
- **target**: Build target for compiled languages (e.g., `qemu-armv8a:nsh_smp`, `sim:cmocka`)
- **source_files**: Files being modified (for targeted rebuild)
- **max_retries**: Auto-fix retry limit (default: 3)
- **framework**: Override auto-detected framework (e.g., `jest`, `pytest`, `cmocka`)

## Framework Auto-Detection

Delegate to `unit-test-gen` skill for framework detection. Harness receives the detected framework and uses it to determine build, run, and parse strategies.

If `unit-test-gen` cannot detect a framework, prompt the user:
```
⚠️ 未检测到测试框架。请指定：
  1. 项目使用的测试框架（如 jest, pytest, cmocka）
  2. 运行测试的命令
```

## Skill Dependencies

| Phase | Skill Used | When |
|-------|-----------|------|
| Build | `vela-build` | NuttX/Vela targets |
| Run | `vela-run` + `executor` | NuttX targets (QEMU/simulator) |
| Test Gen | `unit-test-gen` | All languages — auto-detects framework, delegates to cmocka-unit-test/gtest-unit-test for NuttX or generates directly |
| TDD Discipline | `test-driven-development` | Always — enforces Red-Green-Refactor |
| Test Run | `executor` | NuttX targets |
| Test Run | `execute_bash` | All other languages (direct command) |
| Fix | (AI direct) | Analyze failure, propose code fix |
| Commit | `git-commit` | Commit passing changes |

## Workflow

### Step 1: Detect Framework & Validate Environment

Call `unit-test-gen` to detect the project's test framework.

If test code doesn't exist yet, prompt with the appropriate skill:
```
⚠️ 未找到测试代码。是否先生成测试？
  检测到框架: {framework}
  推荐: unit-test-gen skill
```

### Step 2: Build (compiled languages only)

**NuttX/Vela** — use `vela-build` skill:
```bash
cmake -Bbuild -GNinja -DBOARD_CONFIG={target} nuttx
ninja -C build
```

**Rust:**
```bash
cargo build --tests
```

**Go:**
```bash
go build ./...
```

**JS/TS/Python** — skip build step (interpreted).

**Build failure handling:** same as before — capture error, propose fix, user approves, retry.

### Step 3: Run Tests

**NuttX/Vela** — via `executor` (interactive NSH shell):
```python
executor_start(command="./build/nuttx")  # or QEMU
executor_send(text="{test_cmd}", wait_time=5.0, full_buffer=true, tail_lines=200)
```

**All other languages** — via `execute_bash`:

| Framework | Command |
|-----------|---------|
| Jest | `npx jest --verbose --no-coverage 2>&1` |
| Vitest | `npx vitest run --reporter=verbose 2>&1` |
| Mocha | `npx mocha --reporter spec 2>&1` |
| Node test | `node --test 2>&1` |
| pytest | `pytest -v 2>&1` |
| unittest | `python -m unittest -v 2>&1` |
| cargo test | `cargo test -- --nocapture 2>&1` |
| go test | `go test -v ./... 2>&1` |

### Step 4: Parse Results

Each framework has its own output format. Parse by detected framework:

#### cmocka
```
[  PASSED  ] X test(s).
[  FAILED  ] Y test(s).
```
Match: `[       OK ]`, `[  FAILED  ]`, `[  PASSED  ]`

#### gtest
```
[  PASSED  ] X tests.
[  FAILED  ] Y tests.
```
Match: `[       OK ]`, `[  FAILED  ]`, `[  PASSED  ]` (same as cmocka)

#### Jest / Vitest
```
Tests:       2 failed, 15 passed, 17 total
```
Match: `Tests:` summary line. Failed tests prefixed with `✕` or `FAIL`.

#### Mocha
```
  3 passing (10ms)
  1 failing
```
Match: `passing`, `failing`. Failed tests indented with error details.

#### pytest
```
2 passed, 1 failed in 0.12s
```
Match: `passed`, `failed` in summary. Failed tests under `FAILURES` section.

#### cargo test
```
test result: FAILED. 1 passed; 1 failed; 0 ignored
```
Match: `test result:` line.

#### go test
```
ok   package  0.003s
FAIL package  0.003s
```
Match: `ok` / `FAIL` per package. `--- FAIL:` per test.

#### Node test runner
```
# tests 5
# pass 4
# fail 1
```
Match: TAP format `# pass`, `# fail`.

#### Universal failure patterns (all frameworks)

| Pattern | Meaning |
|---------|---------|
| `Segmentation fault` / `SIGSEGV` | Crash |
| `Timeout` / no output for 30s | Hang/deadlock |
| `command not found` / `MODULE_NOT_FOUND` | Missing dependency |
| Non-zero exit code with no parseable output | Unknown failure — show raw output |

### Step 5: Report & Decide

**All tests pass:**
```
✅ Harness PASS — {total} tests, all passed.
  Framework: {framework}
  Command: {test_cmd}
  Time: {elapsed}s

是否提交？(使用 git-commit skill)
```

**Some tests fail:**
```
❌ Harness FAIL — {passed}/{total} passed, {failed} failed.
  Framework: {framework}
  Failed tests:
    - {test_name}: {brief reason}

Retry {current}/{max_retries}. AI 分析失败原因并提出修复方案：

[AI analysis and proposed fix here]

是否应用修复？[y/n]
```

**Max retries reached:**
```
⚠️ 已达到最大重试次数 ({max_retries})。
  尝试过的修复：
    1. [fix description]
    2. [fix description]
    3. [fix description]

建议手动检查以下文件：
  - {source_file}:{line} — {issue}
```

### Step 6: Cleanup

For NuttX/executor-based runs:
```python
executor_stop(process_id="{pid}")
```

Always stop executor processes, even on failure. Bash-based runs need no cleanup.

## Batch Mode

Run multiple test modules/files in sequence:

```
harness batch tests="cmocka_test_a cmocka_test_b cmocka_test_c"
harness batch tests="tests/test_auth.py tests/test_api.py"
```

Workflow:
1. Build once (if compiled)
2. Run each test sequentially
3. Collect all results
4. Report summary:

```
📊 Batch Results — 3 modules, 47 tests total

  ✅ module_a: 15/15 passed
  ❌ module_b: 12/14 passed (2 failed)
  ✅ module_c: 18/18 passed

Overall: 45/47 passed (95.7%)
Failed: module_b → test_boundary, test_double_free
```

## Integration with SDD Workflow

When used after `spec-design` skill generates a task list:

```
spec-design → tasks[] → for each task:
  1. Generate tests (via detected framework's skill or AI)
  2. vela-harness → build + run tests (expect RED)
  3. Implement code
  4. vela-harness → build + run tests (expect GREEN)
  5. git-commit → commit
```

## Key Rules

- **TDD test phase is mandatory by default** — always generate and run tests unless the user explicitly says to skip testing (e.g., "跳过测试", "不用写测试", "skip tests")
- **Auto-detect framework, don't assume** — always detect from project files before running; never hardcode a framework assumption
- **Never auto-apply fixes without user approval** — always show proposed fix first
- **Always cleanup executor processes** — even on error/timeout
- **Parse output strictly** — use the detected framework's parser, don't guess test results
- **Preserve test code** — harness never modifies test files, only implementation files
- **One retry = one build-run cycle** — don't count parse/analysis as retry
- **Timeout is a failure** — treat hangs same as test failures
