---
name: unit-test-gen
description: >
  Generate unit tests for any language/framework. Auto-detects test framework from project
  context (cmocka, gtest, pytest, jest, vitest, mocha, cargo test, go test, node test runner).
  Delegates to specialized skills when available (cmocka-unit-test, gtest-unit-test), otherwise
  generates tests directly per detected framework conventions. Use when user says "generate tests",
  "写测试", "生成测试", "write tests for", or when vela-harness needs test code generated.
---

# Unit Test Generator

Auto-detect project test framework and generate appropriate test code.

## Input

Required:
- **source_files**: Files to generate tests for (paths or module names)

Optional:
- **framework**: Override auto-detected framework
- **coverage_focus**: What to emphasize (e.g., "error paths", "boundary values", "all public API")

## Framework Detection

**Detection order — first match wins.**

### NuttX/Vela (C/C++)

Check: `nuttx/Makefile` or `nuttx/CMakeLists.txt` exists

| Signal | Framework | Action |
|--------|-----------|--------|
| C source + cmocka available | cmocka | Delegate to `cmocka-unit-test` skill, load `references/nuttx-c-testing.md` |
| C++ source + gtest available | gtest | Delegate to `gtest-unit-test` skill |

### JavaScript / TypeScript

Check: `package.json` exists → read `devDependencies` + check config files

| Signal | Framework | Test File Pattern |
|--------|-----------|-------------------|
| `jest` in deps or `jest.config.*` exists | Jest | `*.test.ts` / `*.test.js` |
| `vitest` in deps or `vitest.config.*` exists | Vitest | `*.test.ts` / `*.test.js` |
| `mocha` in deps or `.mocharc.*` exists | Mocha | `*.spec.ts` / `*.spec.js` |
| None + Node ≥ 20 | Node test runner | `*.test.js` |

### Python

Check: `pyproject.toml` or `setup.py` or `requirements*.txt` exists

| Signal | Framework | Test File Pattern |
|--------|-----------|-------------------|
| `[tool.pytest]` in pyproject.toml or `pytest.ini` / `conftest.py` exists | pytest | `test_*.py` |
| Fallback | unittest | `test_*.py` |

### Rust

Check: `Cargo.toml` exists → built-in `#[cfg(test)]` module or `tests/` directory.

### Go

Check: `go.mod` exists → `*_test.go` files.

### Detection Failure

```
⚠️ 未检测到测试框架。请告知：
  1. 项目使用的测试框架
  2. 测试文件的存放位置
```

## Workflow

### Step 1: Detect Framework

Run detection logic above. Report to user:
```
🔍 检测到测试框架: {framework}
   项目类型: {language}
   测试目录: {test_dir}
```

### Step 2: Analyze Source

Read source files to extract:
- Public API (exported functions/classes/methods)
- Parameter types and constraints
- Error paths and return codes
- Dependencies (imports, syscalls, external services)
- State management (if any)

### Step 3: Generate Tests

**Load the framework-specific reference before generating:**

| Detected Framework | Reference to Load | Action |
|---|---|---|
| cmocka (NuttX/C) | `references/nuttx-c-testing.md` | Delegate to `cmocka-unit-test` skill |
| gtest (NuttX/C++) | — | Delegate to `gtest-unit-test` skill |
| Jest / Vitest | `references/jest-testing.md` | Generate directly per reference patterns |
| pytest / unittest | `references/pytest-testing.md` | Generate directly per reference patterns |
| cargo test (Rust) | `references/rust-testing.md` | Generate directly per reference patterns |
| go test (Go) | `references/go-testing.md` | Generate directly per reference patterns |

Always also load `references/testing-anti-patterns.md` for mock quality guidance.

**For NuttX/C (cmocka):** delegate to `cmocka-unit-test` skill with `references/nuttx-c-testing.md` loaded for mock strategy guidance.

**For NuttX/C++ (gtest):** delegate to `gtest-unit-test` skill.

**For all other frameworks:** generate directly following the loaded reference, applying these universal rules:

#### Test Coverage Requirements

| Category | Required | Example |
|----------|----------|---------|
| Normal path | ✅ | Valid inputs → expected output |
| Null/invalid params | ✅ | null, undefined, empty string, wrong type |
| Boundary values | ✅ | 0, -1, MAX, empty array, single element |
| Error paths | ✅ | Network failure, file not found, permission denied |
| Edge cases | ✅ | Concurrent access, unicode, very large input |

#### Generation Rules

- One test per behavior, not per function
- Test names describe the behavior: `test_rejects_empty_email`, not `test_validate_1`
- Use real objects over mocks when possible (see `references/testing-anti-patterns.md`)
- Each test is independent — no shared mutable state
- Arrange-Act-Assert structure

### Step 4: Place Test Files

Follow project conventions first (check existing test files). If no convention detected, follow the framework-specific reference's "Project Structure" section.

### Step 5: Verify Generation

After writing test files:
```
✅ 测试已生成:
  框架: {framework}
  文件: {test_file_paths}
  用例数: {count}
  覆盖: {coverage_summary}

下一步: 运行测试验证（通过 vela-harness 或直接执行）
```

## Framework-Specific Templates

Detailed templates, project structure, mock strategies, and config integration are in the framework-specific references. Load the appropriate reference before generating — do not use inline templates.

## Key Rules

- **Detect, don't assume** — always detect framework from project files before generating
- **Delegate when possible** — use `cmocka-unit-test` / `gtest-unit-test` for NuttX, they have deeper domain knowledge
- **Load references per context** — NuttX/C projects load `references/nuttx-c-testing.md` for mock patterns
- **Follow project conventions** — match existing test file naming, directory structure, import style
- **Never generate tests that pass without implementation** — if source already exists, tests should cover untested paths
- **Anti-patterns awareness** — always consider `references/testing-anti-patterns.md` when generating mocks
