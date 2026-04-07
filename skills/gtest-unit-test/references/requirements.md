# Google Test Unit Test Generator — Detailed Requirements

## Project Context

- **Project**: Vela (NuttX-based) embedded system
- **Framework**: Google Test (gtest) + Google Mock (gmock)
- **Language**: C++
- **Reference**: https://google.github.io/googletest/ , https://google.github.io/googletest/gmock_for_dummies.html

## Source & Test Directory Convention

产品代码和单元测试代码的目录结构保持镜像关系：

```
[module_root]/
├── include/
│   └── [module]/
│       └── [submodule]/
│           └── [source].h
├── src/
│   └── [submodule]/
│       └── [source].cpp
└── tests/
    ├── [module]_unit_test.cpp      # 入口文件，包含 main()
    └── [submodule]/
        └── [source]_test.cpp       # 测试文件
```

例如 `src/file/file.cpp` 对应的单元测试文件为 `tests/file/file_test.cpp`。

## Interaction Model

Use natural language dialog. Typical flow:

```
User: Generate tests for frameworks/runtimes/ash/src/file/file.cpp

Agent: 检测到以下内容：

类及其公有方法：
  - File
    - open(const std::string& path, int flags) -> bool
    - close() -> void
    - read(void* buf, size_t count) -> ssize_t

自由函数：
  （无）

确认以下问题：
1. 是否需要生成 Mock 类？（默认：否）
2. 是否有额外需要 mock 的外部依赖？（默认：否）
3. 输出路径 frameworks/runtimes/ash/tests/file/file_test.cpp 是否可以？（默认：是）
4. 是否需要调整测试范围？（默认：测试所有公有方法）

User: 1. 是，mock ExternalAPIs 类  2-4 默认

Agent: [generates files...]
```

## Mock Strategy

### gmock（C++ 接口/抽象类）

当被测代码依赖抽象类或接口（含虚函数）时，使用 gmock 生成 Mock 类：

```cpp
class MockExternalAPIs : public ExternalAPIs {
 public:
  MOCK_METHOD(bool, fn1, (const std::string& name), (override));
  MOCK_METHOD(std::string, fn2, (int id), (override));
};
```

### C 函数 Mock（预处理器重定向）

当被测 C++ 代码调用 C 函数（如 POSIX API、NuttX 系统调用）时，使用预处理器重定向方案，
与 cmocka-unit-test 中的方案类似。详见 [mock.md](mock.md)。

### User-specified

- User can add extra mock classes/functions via dialog
- User can exclude auto-detected mocks

## Test Scenarios

For each function/method, generate at minimum:

1. **Normal path** — valid inputs, correct return
2. **NULL/nullptr parameters** — nullptr for each pointer param
3. **Invalid input** — out-of-range, bad enums
4. **Boundary conditions** — max/min values, empty strings/containers
5. **Error handling** — mock dependencies return errors/throw exceptions

## Naming Conventions

- Test files: `[source]_test.cpp` — mirrors source file name with `_test` suffix
- Test class (fixture): `[Source]Test` — PascalCase of source name + `Test`
- Test cases: `TEST_F([Source]Test, MethodName_Scenario_Expected)` — descriptive naming
- Mock classes: `Mock[ClassName]` — `Mock` prefix + original class name
- Entry file: `tests/[module]_unit_test.cpp` — one per module
- Namespace: `namespace [module] { namespace { ... } }` — anonymous inner namespace
- Follow project coding standard

## Test Case Naming Patterns

使用 `MethodName_Scenario_Expected` 三段式命名：

| 部分 | 说明 | 示例 |
|------|------|------|
| MethodName | 被测方法名 | `Open`, `Read`, `ParseConfig` |
| Scenario | 测试场景 | `ValidPath`, `NullParam`, `EmptyString` |
| Expected | 期望结果 | `ReturnsTrue`, `ThrowsException`, `ReturnsNullptr` |

完整示例：`TEST_F(FileTest, Open_ValidPath_ReturnsTrue)`

对于参数化测试，使用 `TEST_P`：

```cpp
TEST_P(FilePathParamTest, Normalize_VariousPaths_ReturnsExpected) {
  auto [input, expected] = GetParam();
  EXPECT_EQ(FilePath::normalize(input), expected);
}

INSTANTIATE_TEST_SUITE_P(
    PathCases, FilePathParamTest,
    ::testing::Values(
        std::make_pair("/a/b/../c", "/a/c"),
        std::make_pair("/a/./b", "/a/b")
    ));
```

## Build Files

See [templates.md](templates.md) for complete build file templates:
- [CMakeLists.txt Template](templates.md#cmakeliststxt-template)
- [Makefile Template](templates.md#makefile-template)
- [Make.defs Template](templates.md#makedefs-template)
- [Kconfig Template](templates.md#kconfig-template)

### Key Points

- 配置和编译开关宏已有统一入口，通常不需要 AI 工具添加 Kconfig/CMakeLists.txt
- 如果模块已有 `*_UNIT_TEST` 配置项，直接复用，不要重复创建
- 如果模块没有测试配置，参考模板创建
- 使用 `LIB_GOOGLETEST` 作为 gtest 依赖（Kconfig 中 `depends on LIB_GOOGLETEST`）
- Include Apache 2.0 license header on all generated files

## Output Path

- Default: `[module_root]/tests/[submodule]/` — mirrors source directory structure
- User can specify custom path
- Create directories if they don't exist

## Quality Requirements

- Generated code MUST compile with C++17 or later
- Generated tests MUST be runnable (even if assertions fail)
- All files include Apache 2.0 license header
- Code follows project coding conventions
- Use `EXPECT_*` for non-fatal assertions, `ASSERT_*` for fatal assertions
- Prefer `EXPECT_*` unless subsequent assertions depend on the result
