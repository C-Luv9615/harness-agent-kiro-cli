---
name: gtest-unit-test
description: >
  Generate Google Test (gtest) + Google Mock (gmock) unit tests for Vela/NuttX C++ code.
  Use when the user asks to "generate C++ unit tests", "create gtest tests",
  "write test cases for C++ functions/classes", "generate tests with gmock",
  "create Google Test cases", or any request involving gtest/gmock test generation
  for C++ source files. Handles test code generation, mock classes, build files
  (Makefile, CMakeLists.txt, Make.defs, Kconfig), and directory structure following
  the project's C++ unit test conventions.
---

# Google Test Unit Test Generator for Vela/NuttX C++

Generate Google Test + Google Mock unit tests for C++ code in Vela/NuttX systems.

## Naming Definitions

Throughout this document, the following placeholders are used. **Strictly distinguish** between them:

| Placeholder | Scope | Derivation Rule | Example |
|-------------|-------|-----------------|---------|
| `[module]` | Top-level module name (CMake build unit) | Derived from source tree: the directory containing `CMakeLists.txt`, `Kconfig`, and `src/`+`tests/` | `ash`, `connectivity`, `media` |
| `[submodule]` | Sub-directory under `src/` and `tests/` | Mirrors the sub-directory structure under `src/` | `file`, `net`, `codec` |
| `[source]` | Source file name (without `.cpp` extension) | The basename of the source `.cpp` file being tested | `file`, `file_path`, `codec_manager` |
| `[Source]` | PascalCase form of `[source]` | Convert `[source]` to PascalCase | `File`, `FilePath`, `CodecManager` |
| `[MODULE_UPPER]` | Uppercase form of `[module]` | `[module]` converted to uppercase | `ASH`, `CONNECTIVITY` |
| `[test_module]` | 独立目录模式下的测试模块名 | 由源文件路径推导，取最后两级目录用 `_` 连接。例如 `frameworks/runtimes/ash/src/file/file.cpp` → `ash_file` | `ash_file`, `media_codec` |
| `[TEST_MODULE_UPPER]` | `[test_module]` 的大写形式 | `[test_module]` 转大写 | `ASH_FILE`, `MEDIA_CODEC` |
| `[output_base]` | 独立目录模式下测试输出的基础目录 | 默认为 `tests/velatest`，用户可在 Step 1 确认时自定义 | `tests/velatest`, `tests/mytest` |

## Directory Modes

本 skill 支持两种测试目录模式，由用户在 Step 1 中选择：

### 模式 A：内嵌模式（Embedded，默认）

测试文件放在源码模块内部的 `tests/` 子目录下，与 `src/` 平级。构建配置追加到模块已有的 `CMakeLists.txt`、`Makefile` 等文件中。

```
[module_root]/
├── CMakeLists.txt              (追加 test 编译段)
├── src/
│   └── [submodule]/[source].cpp
└── tests/
    ├── [module]_unit_test.cpp
    └── [submodule]/[source]_test.cpp
```

适用场景：模块自身维护测试，测试与源码紧密耦合。

### 模式 B：独立目录模式（Standalone）

测试文件放在独立的目录中（默认 `tests/velatest/[test_module]/`），拥有独立的构建文件（`CMakeLists.txt`、`Makefile`、`Make.defs`、`Kconfig`）。

```
[output_base]/[test_module]/
├── CMakeLists.txt
├── Kconfig
├── Make.defs
├── Makefile
├── src/
│   └── [source]_test.cpp
└── gt_[test_module]_entry.cpp
```

适用场景：测试与源码分离管理，或需要在统一的测试目录下组织多个模块的测试。

Key rules (common to both modes):
- **One test file per source file**: Each source file gets its own `_test.cpp` file.
- **Test class uses PascalCase**: `[Source]Test` (e.g., `FileTest`, `FilePathTest`)
- **Test cases use descriptive names**: `TEST_F([Source]Test, MethodName_Scenario_Expected)` or `TEST_F([Source]Test, API_Name)`
- **Namespace**: Tests are wrapped in `namespace [module] { namespace { ... } }` (anonymous inner namespace).

Key rules (Embedded mode):
- **Test file mirrors source file**: `src/[submodule]/[source].cpp` → `tests/[submodule]/[source]_test.cpp`
- **Build files use `[module]`**: `CMakeLists.txt`, `Makefile`, `Make.defs`, `Kconfig` are at the module level.
- **Entry file**: `tests/[module]_unit_test.cpp` — contains `main()` that runs all tests.

Key rules (Standalone mode):
- **Test file path**: `[output_base]/[test_module]/src/[source]_test.cpp`
- **Build files are standalone**: 独立目录下有完整的 `CMakeLists.txt`、`Makefile`、`Make.defs`、`Kconfig`。
- **Entry file**: `gt_[test_module]_entry.cpp` — contains `main()` that runs all tests.
- **Config prefix**: 使用 `GT_[TEST_MODULE_UPPER]_TEST` 作为 Kconfig 配置项前缀（区别于内嵌模式的 `[MODULE_UPPER]_UNIT_TEST`）。

## References

- [Google Test documentation](https://google.github.io/googletest/)
- [Google Mock documentation](https://google.github.io/googletest/gmock_for_dummies.html)
- For detailed requirements and examples, read [references/requirements.md](references/requirements.md)
- For code templates, read [references/templates.md](references/templates.md)
- For mock implementation rules (gmock), read [references/mock.md](references/mock.md)

## Workflow

### Step 1: Collect Parameters and Confirm

When the user provides a source file path, first read the source file to:
1. Auto-detect the module name and derive the test output path.
2. List all classes, public methods, and free functions found in the source file.

**MUST display the function/class list in the following format:**

```
检测到以下内容：

类及其公有方法：
  - ClassName
    - method1(param_type1 param1) -> return_type
    - method2(void) -> void

自由函数（非成员函数）：
  - function_name_1(param_type1 param1) -> return_type
  - function_name_2(void) -> void

（如果源文件包含 main 函数，需标注"已排除 main 函数"）
```

Then **MUST present the following 3 questions to the user and wait for explicit confirmation before proceeding**:

1. **是否需要 mock 依赖？**（默认：否。如需 mock，请指定需要 mock 的类、接口或函数，例如："mock IFileSystem 接口"、"mock ExternalAPIs 类"。AI 将根据 [references/mock.md](references/mock.md) 的规则自动选择 mock 策略：C++ 虚函数接口用 gmock，C 函数/系统调用封装为接口类后用 gmock，轻量依赖直接使用真实对象）
2. **测试文件放在哪里？**（必须让用户选择，展示时将占位符替换为实际路径）
   - **A) 内嵌模式**：放在源码模块内部 → 显示实际路径如 `frameworks/runtimes/ash/tests/threading/thread_test.cpp`（路径固定，不可自定义）
   - **B) 独立目录模式**：放在独立测试目录 → 显示实际路径如 `tests/velatest/ash_threading/src/thread_test.cpp`（可自定义目录，例如："B，但放在 tests/mytest/ 下"）
3. **是否需要调整测试范围？**（默认：测试上方列出的所有公有方法和自由函数。用户可以指定只测试某些方法，或排除某些方法。例如："只测试 method_a 和 method_b"、"不测试 method_c"）

**BLOCKING**: 必须等待用户明确回复确认后，才能继续执行 Step 2。禁止跳过确认直接生成代码。

### Step 2: Analyze Source Code

Read the source file and extract for each target function/method:
- Function/method signature (name, return type, parameters, const/virtual qualifiers)
- Class membership and access level (public/protected/private)
- Internal dependencies (to determine mock needs)
- Branching complexity (if/else/switch for test scenario planning)
- Whether the class has virtual methods (candidates for gmock)

**过滤规则**：如果源文件中包含 `main` 函数，则 `main` 函数必须从测试目标中排除。

### Step 3: Determine Mock Strategy

**如果用户在 Step 1 中对 Mock 问题回答了"否"，则跳过本步骤。**

仅当用户明确需要 mock 时，才执行 mock 逻辑。

**MUST read [references/mock.md](references/mock.md) for mock strategy selection, implementation rules, and NuttX flat build constraints.**

### Step 4: Generate Test Scenarios

For each function/method, generate these test types:

| Scenario | Naming | Description |
|----------|--------|-------------|
| Normal path | `MethodName_NormalInput_ReturnsExpected` | Valid inputs, verify correct return |
| NULL/nullptr params | `MethodName_NullParam_ReturnsError` | nullptr for each pointer param |
| Invalid input | `MethodName_InvalidInput_ReturnsError` | Out-of-range values, bad enums |
| Boundary | `MethodName_BoundaryValue_HandlesCorrectly` | Max/min values, empty strings |
| Error handling | `MethodName_DependencyFails_HandlesError` | Mock dependencies return errors |

Minimum 3 scenarios per function/method.

### Step 5: Determine Generation Mode

Check whether the test directory already exists.

**Full mode** (directory does not exist or is empty):
- Generate all files: build files, entry file, test source

**Incremental mode** (directory already exists with build files):
- Only create new test source file
- Modify existing files (entry file, build files)
- Do NOT regenerate existing build files from scratch

### Step 5a: Generate Files (Full Mode — Embedded)

**仅当用户在 Step 1 中选择了模式 A（内嵌模式）时使用。**

Generate the following directory structure (mirrors the source tree):

```
[module_root]/
├── CMakeLists.txt              (生成或追加 test 编译段)
├── Kconfig                     (生成或追加 UNIT_TEST 配置项)
├── Make.defs                   (生成或追加 test 构建入口)
├── Makefile                    (生成或追加 test 编译段)
├── include/
│   └── [module]/...            (existing headers)
├── src/
│   └── [submodule]/
│       └── [source].cpp        (existing source)
└── tests/
    ├── [module]_unit_test.cpp  (entry point with main)
    └── [submodule]/
        └── [source]_test.cpp   (test file)
```

| Action | File | Description |
|--------|------|-------------|
| 生成/追加 | `CMakeLists.txt` | 添加 `if(CONFIG_[MODULE_UPPER]_UNIT_TEST)` 编译段，使用 `GLOB_RECURSE` 收集 `tests/*_test.cpp` |
| 生成/追加 | `Kconfig` | 添加 `[MODULE_UPPER]_UNIT_TEST` 配置项，依赖 `LIB_GOOGLETEST` |
| 生成/追加 | `Make.defs` | 添加 `ifneq ($(CONFIG_[MODULE_UPPER]_UNIT_TEST),)` 段 |
| 生成/追加 | `Makefile` | 添加 `ifeq ($(CONFIG_[MODULE_UPPER]_UNIT_TEST), y)` 段，列出 MAINSRC 和 CXXSRCS |
| 生成 | `tests/[module]_unit_test.cpp` | 入口文件，包含 `main()` 调用 `RUN_ALL_TESTS()` |
| 生成 | `tests/[submodule]/[source]_test.cpp` | 测试文件，包含 test fixture 和 test cases |

**注意**：如果模块已有 `CMakeLists.txt`、`Kconfig`、`Makefile`、`Make.defs`，则在现有文件中追加 test 相关段落，不要覆盖已有内容。如果模块尚无这些文件，则按模板完整生成。

### Step 5a-S: Generate Files (Full Mode — Standalone)

**仅当用户在 Step 1 中选择了模式 B（独立目录模式）时使用。**

Generate the following directory structure:

```
[output_base]/[test_module]/
├── CMakeLists.txt
├── Kconfig
├── Make.defs
├── Makefile
├── src/
│   └── [source]_test.cpp       (test file)
└── gt_[test_module]_entry.cpp  (entry point with main)
```

| Action | File | Description |
|--------|------|-------------|
| 生成 | `CMakeLists.txt` | 完整的 CMake 构建文件，使用 `GT_[TEST_MODULE_UPPER]_TEST` 配置项，`GLOB` 收集 `src/*_test.cpp` |
| 生成 | `Kconfig` | `GT_[TEST_MODULE_UPPER]_TEST` 配置项，依赖 `LIB_GOOGLETEST` |
| 生成 | `Make.defs` | `ifneq ($(CONFIG_GT_[TEST_MODULE_UPPER]_TEST),)` 段，`CONFIGURED_APPS` 指向 `$(APPDIR)/[output_base]/[test_module]` |
| 生成 | `Makefile` | 完整的 Makefile，`PROGNAME = gtest_[test_module]`，列出 MAINSRC 和 CXXSRCS，`CXXFLAGS` 包含被测模块的 include 和 src 路径 |
| 生成 | `gt_[test_module]_entry.cpp` | 入口文件，包含 `main()` 调用 `RUN_ALL_TESTS()` |
| 生成 | `src/[source]_test.cpp` | 测试文件，包含 test fixture 和 test cases |

**独立目录模式的 include 路径**：由于测试文件不在源码模块内部，需要在构建文件中显式添加被测模块的 include 路径：
- `CXXFLAGS += -I[module_root]/include -I[module_root]/src`（Makefile）
- `INCLUDE_DIRECTORIES` 中添加 `[module_root]/include` 和 `[module_root]/src`（CMakeLists.txt）

其中 `[module_root]` 是被测源码模块的根目录（包含 `src/` 和 `include/` 的目录）。

### Step 5b: Generate Files (Incremental Mode — Embedded)

**仅当用户选择了模式 A（内嵌模式）且目录已存在时使用。**

Only create and modify:

| Action | File | Description |
|--------|------|-------------|
| Create | `tests/[submodule]/[source]_test.cpp` | New test source with test fixture and test cases |
| Modify | `CMakeLists.txt` | Add new test source if not using glob |
| Modify | `Makefile` | Append new CXXSRCS if needed |

### Step 5b-S: Generate Files (Incremental Mode — Standalone)

**仅当用户选择了模式 B（独立目录模式）且目录已存在时使用。**

Only create and modify:

| Action | File | Description |
|--------|------|-------------|
| Create | `src/[source]_test.cpp` | New test source with test fixture and test cases |
| Modify | `gt_[test_module]_entry.cpp` | 追加 `#include "src/[source]_test.cpp"` |
| Modify | `Makefile` | Append `CXXSRCS += src/[source]_test.cpp` if not present |

### Step 5c: Compilation Mode and main() Rules

NuttX 使用 flat build 模型，所有 `nuttx_add_application` / `PROGNAME` 最终链接到同一个二进制中。因此：

#### 内嵌模式（Embedded）

**所有测试文件统一使用 Include 模式：**

- 测试文件通过 `#include` 被拉入 `tests/[module]_unit_test.cpp` 统一编译
- 测试文件中 **不需要** `main()` 函数（`main()` 已在入口文件中定义）
- 需要在 `tests/[module]_unit_test.cpp` 中追加 `#include "[submodule]/[source]_test.cpp"`

**禁止创建独立的测试 PROGNAME**：独立 PROGNAME 的 `RUN_ALL_TESTS()` 会跑到所有链接进来的 TEST_F（包括其他测试文件的用例），无法实现真正的隔离。

#### 独立目录模式（Standalone）

**所有测试文件统一使用 Include 模式：**

- 测试文件通过 `#include` 被拉入 `gt_[test_module]_entry.cpp` 统一编译
- 测试文件中 **不需要** `main()` 函数（`main()` 已在入口文件中定义）
- 需要在 `gt_[test_module]_entry.cpp` 中追加 `#include "src/[source]_test.cpp"`

**禁止创建独立的测试 PROGNAME**：原因同内嵌模式。

### Step 6: Output Summary

After generation, display:
- List of generated files
- Number of test functions per file
- Mock classes used
- Build instructions

**注意：本步骤仅是文件生成的摘要，不是整个工作流的结束。禁止输出"工作完成"之类的结束语，必须立即继续执行 Step 7。**

### Step 7: Update defconfig and Build

**BLOCKING — 必须立即读取并执行 [references/build.md](references/build.md) 中的全部步骤，直到构建完成为止。**

构建过程中和构建完成后的统计数据追踪、最终报告生成、JSON 输出和 HTTP 上传，**MUST read [../cmocka-unit-test/references/statistics.md](../cmocka-unit-test/references/statistics.md) for detailed rules and execute all steps described therein.**

注意：gtest 的 `file_list` 传入 `.cpp` 文件，脚本会自动检测并使用 gtest 计数模式（统计 `TEST_F`/`TEST` 宏数量）。人工修改追踪的文件范围也对应为 `.cpp` 和 `.h` 文件。

**注意：Step 7 构建成功后，禁止输出统计报告或"工作完成"之类的结束语，必须立即继续执行 Step 8。**

### Step 8: Run Tests on NuttX

**BLOCKING — 构建成功后，必须立即读取 [vela-run skill](../../vela-run/SKILL.md) 启动 NuttX 并执行测试用例，直到所有用例通过为止。**

#### 8a: 环境准备与启动

1. **安装 tmux**（如果不可用）：`which tmux || apt-get install -y tmux`
2. **按照 vela-run skill 启动 NuttX**：使用 Step 7 中用户选择的 lunch config 对应的目标类型，参照 vela-run skill 中的启动方式
3. **SIM 启动失败回退**：当 lunch config 为 SIM 类型（路径包含 `sim/`）时，如果直接运行 `./out/<config>/nuttx` 启动后卡住（超过 10 秒无 `ap>` 或 `nsh>` 提示符输出），执行以下回退步骤：
   - 先清理残留共享内存：`rm -f /dev/shm/ap-remote`
   - 将二进制复制到 `./nuttx/` 目录：`cp out/<config>/nuttx ./nuttx/nuttx`
   - 改用 `./nuttx/nuttx` 启动

#### 8b: 确定测试命令名称并执行

**测试命令名称确定规则**：
- 优先从 `CMakeLists.txt` 中的 `NAME` 字段获取（CMake 构建系统使用此名称注册命令）
- 如果没有 CMakeLists.txt，则使用 `Makefile` 中的 `PROGNAME`
- 可通过 `strings out/<config>/nuttx | grep "gt_[test_module]\|gtest_[module]"` 验证实际注册的命令名

**使用 `--gtest_filter` 只执行本次生成的用例**：
- 从本次生成的 `[source]_test.cpp` 文件中提取测试类名（`[Source]Test`、`[Source]ParamTest` 等）
- 构造 filter 表达式：`--gtest_filter=[Source1]Test.*:[Source2]Test.*`（多个类用 `:` 分隔）
- 完整命令示例：`gtest_ash_file --gtest_filter=FileTest.*:FilePathTest.*`

NSH 就绪后，发送测试命令并捕获 gtest 输出。

#### 8c: 分析测试结果

解析 gtest 输出，判断测试是否全部通过：

- **全部通过**（输出包含 `[  PASSED  ]` 且无 `[  FAILED  ]`）：进入 Step 9
- **存在失败**（输出包含 `[  FAILED  ]`）：进入 8d 修复流程

#### 8d: 修复失败用例并重试

当测试用例执行失败时：

1. **分析失败原因**：根据 gtest 输出的失败信息（断言失败、信号异常、超时等）定位问题
2. **修复代码**：修改测试代码（`src/[source]_test.cpp`）或相关文件修复问题
3. **重新编译**：执行 build.md 中的 Step 3 构建流程（`m`），确保编译通过
4. **重新启动并执行**：重复 8a～8b 的流程，启动 NuttX 并再次执行测试用例
5. **循环直到通过**：重复 8c → 8d 直到所有用例通过，或达到 3 轮修复仍未通过时，将失败信息输出给用户并请求协助

**注意**：
- 修复代码时只修改测试代码，不修改被测源文件
- 每轮修复后需重新编译，编译失败也需要修复直到编译通过
- 修复轮次计入 Step 7 的编译修复轮次统计
- 必须追踪用例修复轮次（每完成一轮 8d 的"修复 → 编译通过 → 重新运行测试"循环计为 1 轮），详见 [../cmocka-unit-test/references/statistics.md](../cmocka-unit-test/references/statistics.md) 中的第 1.1 节

### Step 9: Statistics Report, JSON Generation and Upload

**BLOCKING — 测试全部通过后（或用户确认终止时），必须立即读取并执行 [../cmocka-unit-test/references/statistics.md](../cmocka-unit-test/references/statistics.md) 中第 3 节的最终统计报告、JSON 生成与上传流程。**