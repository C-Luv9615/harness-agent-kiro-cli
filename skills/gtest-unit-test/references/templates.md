# Google Test Code Templates

## License Header

所有生成的文件都需要 ASF License Header，用 `[LICENSE_HEADER]` 占位符表示。C++ 文件（`.cpp`、`.h`）使用 `/* */` 注释风格，Build 文件（`Makefile`、`CMakeLists.txt`、`Make.defs`、`Kconfig`）使用 `#` 注释风格。首行包含文件相对路径。

```
/****************************************************************************
 * [filepath]
 *
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.  The
 * ASF licenses this file to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance with the
 * License.  You may obtain a copy of the License at
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  See the
 * License for the specific language governing permissions and limitations
 * under the License.
 *
 ****************************************************************************/
```

## Test Source Template ([source]_test.cpp)

两种模式共用同一模板，仅文件路径不同：
- **Embedded**: `tests/[submodule]/[source]_test.cpp`
- **Standalone**: `[output_base]/[test_module]/src/[source]_test.cpp`

Standalone 模式下 `#include` 路径与 Embedded 相同（如 `#include "ash/file/file.h"`），因为构建文件中已通过 `-I` 添加了被测模块的 include 路径。

```cpp
[LICENSE_HEADER]

/****************************************************************************
 * Included Files
 ****************************************************************************/

#include "[module]/[submodule]/[source].h"  // 被测产品代码头文件

#include <gtest/gtest.h>
#include <gmock/gmock.h>                    // 仅当需要 mock 时

[ADDITIONAL_INCLUDES]

namespace [module] {
namespace {

/****************************************************************************
 * Mock Classes
 ****************************************************************************/

[MOCK_CLASSES]

/****************************************************************************
 * Test Fixture
 ****************************************************************************/

class [Source]Test : public ::testing::Test {
 protected:
  void SetUp() override {
    // 初始化测试环境
  }

  void TearDown() override {
    // 清理测试环境
  }

  // 测试所需的成员变量
  [MEMBER_VARIABLES]
};

/****************************************************************************
 * Test Cases
 ****************************************************************************/

[TEST_CASES]

}  // namespace
}  // namespace [module]
```

**Placeholders**:

- `[ADDITIONAL_INCLUDES]` — Extra headers needed (e.g., `#include <string>`, `#include <vector>`, `#include <memory>`)

- `[MOCK_CLASSES]` — gmock mock class definitions. Example:
  ```cpp
  class MockExternalAPIs : public ExternalAPIs {
   public:
    MOCK_METHOD(bool, fn1, (const std::string& name), (override));
    MOCK_METHOD(std::string, fn2, (int id), (override));
  };
  ```

- `[MEMBER_VARIABLES]` — Test fixture member variables. Example:
  ```cpp
  std::string temp_dir_;
  MockExternalAPIs mock_apis_;
  std::unique_ptr<File> file_;
  ```

- `[TEST_CASES]` — All test case implementations. Example:
  ```cpp
  TEST_F(FileTest, Open_ValidPath_ReturnsTrue) {
    EXPECT_TRUE(file_->open("/tmp/test.txt", O_RDONLY));
  }

  TEST_F(FileTest, Open_InvalidPath_ReturnsFalse) {
    EXPECT_FALSE(file_->open("", O_RDONLY));
  }

  TEST_F(FileTest, Read_NullBuffer_ReturnsError) {
    EXPECT_EQ(file_->read(nullptr, 100), -1);
  }
  ```

## Entry File Template

两种模式共用同一 main 函数体，仅文件名和 include 行不同：
- **Embedded**: `tests/[module]_unit_test.cpp`，测试文件通过 `#include "[submodule]/[source]_test.cpp"` 拉入
- **Standalone**: `[output_base]/[test_module]/gt_[test_module]_entry.cpp`，测试文件通过 `#include "src/[source]_test.cpp"` 拉入

新增测试文件时，在入口文件中追加对应的 `#include` 行。

```cpp
[LICENSE_HEADER]

/****************************************************************************
 * Included Files
 ****************************************************************************/

#include <cstring>
#include <gtest/gtest.h>

#include "[submodule]/[source]_test.cpp"    // Embedded 模式
#include "src/[source]_test.cpp"            // Standalone 模式（二选一）

/****************************************************************************
 * Public Functions
 ****************************************************************************/

extern "C" int main(int argc, char** argv)
{
  ::testing::GTEST_FLAG(filter) = "*";
  ::testing::InitGoogleTest(&argc, argv);

  for (int i = 1; i < argc; i++) {
    const char* arg = argv[i];
    if (strncmp(arg, "--gtest_filter=", 15) == 0) {
      ::testing::GTEST_FLAG(filter) = arg + 15;
    } else if (strcmp(arg, "--gtest_filter") == 0 && i + 1 < argc) {
      ::testing::GTEST_FLAG(filter) = argv[++i];
    }
  }

  return RUN_ALL_TESTS();
}
```

**Note**:
- `extern "C"` 是必须的，因为 NuttX 会将 `main` 重命名为 `[PROGNAME]_main`。
- 手动解析 `--gtest_filter` 是因为 NuttX 静态变量在任务调用间持久化，导致 `InitGoogleTest` 在后续运行时跳过 argv 解析。
- gtest 不需要手动注册测试，所有 `TEST_F` / `TEST_P` 宏在链接时自动发现。

## Parameterized Test Template

When a method needs to be tested with many input variations, use `TEST_P`:

```cpp
class [Source]ParamTest
    : public ::testing::TestWithParam<std::pair<InputType, ExpectedType>> {
 protected:
  void SetUp() override {
    // 初始化
  }
};

TEST_P([Source]ParamTest, MethodName_VariousInputs_ReturnsExpected) {
  auto [input, expected] = GetParam();
  EXPECT_EQ(target_function(input), expected);
}

INSTANTIATE_TEST_SUITE_P(
    InstantiationName, [Source]ParamTest,
    ::testing::Values(
        std::make_pair(input1, expected1),
        std::make_pair(input2, expected2),
        std::make_pair(input3, expected3)
    ));
```

## CMakeLists.txt Template — Embedded Mode

**File**: `CMakeLists.txt` (at module root)
**Purpose**: CMake-based build configuration for unit tests
**Note**: 配置和编译开关宏通常已有统一入口。如果模块已有 `*_UNIT_TEST` 配置，直接复用。

```cmake
[LICENSE_HEADER]

if(CONFIG_[MODULE_UPPER]_UNIT_TEST)
  include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/include
  )

  file(GLOB_RECURSE [MODULE_UPPER]_TEST_SRCS
    ${CMAKE_CURRENT_SOURCE_DIR}/tests/*_test.cpp)

  nuttx_add_application(
    NAME
    [module]_unit_test
    SRCS
    ${CMAKE_CURRENT_SOURCE_DIR}/tests/[module]_unit_test.cpp
    ${[MODULE_UPPER]_TEST_SRCS}
    STACKSIZE
    16384
    PRIORITY
    100
    MODULE
    ${CONFIG_[MODULE_UPPER]_UNIT_TEST}
    INCLUDE_DIRECTORIES
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/src)
endif()
```

**Note**: 使用 `GLOB_RECURSE` 自动收集所有 `*_test.cpp` 文件，新增测试文件时无需修改 CMakeLists.txt。

## Makefile Template — Embedded Mode

**File**: `Makefile` (at module root)
**Purpose**: Traditional make-based build configuration

```makefile
[LICENSE_HEADER]

include $(APPDIR)/Make.defs

ifeq ($(CONFIG_[MODULE_UPPER]_UNIT_TEST), y)
  PROGNAME += [module]_unit_test
  PRIORITY += 100
  STACKSIZE += 16384
  MAINSRC += $(APPDIR)/[module_path]/tests/[module]_unit_test.cpp

  CXXSRCS += tests/[submodule]/[source]_test.cpp

  CXXFLAGS += -I$(CURDIR)/include
endif

include $(APPDIR)/Application.mk
```

## Make.defs Template — Embedded Mode

**File**: `Make.defs`
**Purpose**: Build system integration

```makefile
[LICENSE_HEADER]

ifneq ($(CONFIG_[MODULE_UPPER]_UNIT_TEST),)
CONFIGURED_APPS += $(APPDIR)/[module_path]
endif
```

## Kconfig Template — Embedded Mode

**File**: `Kconfig`
**Purpose**: Configuration menu definition
**Note**: 如果模块已有 Kconfig 且包含 `*_UNIT_TEST` 配置项，不要重复创建。

```kconfig
#
# For a description of the syntax of this configuration file,
# see the file kconfig-language.txt in the NuttX tools repository.
#

config [MODULE_UPPER]_UNIT_TEST
	depends on LIB_GOOGLETEST && [MODULE_UPPER_DEP]
	bool "Enable [module] unit test"
	default n
	help
		Enables unit tests for the [module] module.
		Requires GoogleTest library.
```

**Placeholders**:
- `[MODULE_UPPER]` — Module name in uppercase (e.g., `ASH`)
- `[MODULE_UPPER_DEP]` — Module's own config dependency (e.g., `LIBASH`)
- `[module]` — Module name in lowercase (e.g., `ash`)
- `[module_path]` — Relative path from apps root (e.g., `frameworks/runtimes/ash`)

---

## Standalone Mode Build Templates (模式 B)

以下模板用于独立目录模式，所有构建文件位于 `[output_base]/[test_module]/` 目录下。

### CMakeLists.txt Template — Standalone Mode

**File**: `[output_base]/[test_module]/CMakeLists.txt`

```cmake
[LICENSE_HEADER]

if(CONFIG_GT_[TEST_MODULE_UPPER]_TEST)
  set(GT_[TEST_MODULE_UPPER]_INCDIR
    ${CMAKE_CURRENT_LIST_DIR}/src
    [module_root_abs]/include
    [module_root_abs]/src)

  file(GLOB GT_[TEST_MODULE_UPPER]_CSRC ${CMAKE_CURRENT_LIST_DIR}/src/*_test.cpp)

  nuttx_add_application(
    NAME
    gtest_[test_module]
    PRIORITY
    ${CONFIG_GT_[TEST_MODULE_UPPER]_TEST_PRIORITY}
    STACKSIZE
    ${CONFIG_GT_[TEST_MODULE_UPPER]_TEST_STACKSIZE}
    MODULE
    ${CONFIG_GT_[TEST_MODULE_UPPER]_TEST}
    SRCS
    ${CMAKE_CURRENT_LIST_DIR}/gt_[test_module]_entry.cpp
    ${GT_[TEST_MODULE_UPPER]_CSRC}
    INCLUDE_DIRECTORIES
    ${GT_[TEST_MODULE_UPPER]_INCDIR})
endif()
```

**Placeholders**:
- `[module_root_abs]` — 被测模块根目录的绝对路径表达式，例如 `${APPDIR}/frameworks/runtimes/ash`

### Makefile Template — Standalone Mode

**File**: `[output_base]/[test_module]/Makefile`

```makefile
[LICENSE_HEADER]

include $(APPDIR)/Make.defs

PROGNAME  = gtest_[test_module]
PRIORITY  = $(CONFIG_GT_[TEST_MODULE_UPPER]_TEST_PRIORITY)
STACKSIZE = $(CONFIG_GT_[TEST_MODULE_UPPER]_TEST_STACKSIZE)
MODULE    = $(CONFIG_GT_[TEST_MODULE_UPPER]_TEST)

MAINSRC = $(CURDIR)/gt_[test_module]_entry.cpp

CXXSRCS += src/[source]_test.cpp

CXXFLAGS += -I$(CURDIR)/src
CXXFLAGS += -I$(APPDIR)/[module_path]/include
CXXFLAGS += -I$(APPDIR)/[module_path]/src

include $(APPDIR)/Application.mk
```

**Placeholders**:
- `[module_path]` — 被测模块相对于 apps 根目录的路径（e.g., `frameworks/runtimes/ash`）

### Make.defs Template — Standalone Mode

**File**: `[output_base]/[test_module]/Make.defs`

```makefile
[LICENSE_HEADER]

ifneq ($(CONFIG_GT_[TEST_MODULE_UPPER]_TEST),)
CONFIGURED_APPS += $(APPDIR)/[output_base]/[test_module]
endif
```

### Kconfig Template — Standalone Mode

**File**: `[output_base]/[test_module]/Kconfig`

```kconfig
#
# For a description of the syntax of this configuration file,
# see the file kconfig-language.txt in the NuttX tools repository.
#

config GT_[TEST_MODULE_UPPER]_TEST
	tristate "gtest [test_module]"
	default n
	depends on LIB_GOOGLETEST
	---help---
		Enable Google Test unit tests for [test_module]

if GT_[TEST_MODULE_UPPER]_TEST

config GT_[TEST_MODULE_UPPER]_TEST_PRIORITY
	int "Task priority"
	default 100

config GT_[TEST_MODULE_UPPER]_TEST_STACKSIZE
	int "Stack size"
	default 16384

endif
```

**Placeholders**:
- `[TEST_MODULE_UPPER]` — Test module name in uppercase (e.g., `ASH_FILE`)
- `[test_module]` — Test module name in lowercase (e.g., `ash_file`)
- `[output_base]` — Test output base directory (e.g., `tests/velatest`)

## Complete Example: Embedded Mode (ash module)

测试 `frameworks/runtimes/ash/src/file/file.cpp` 和 `src/file/file_path.cpp`。

```
frameworks/runtimes/ash/
├── CMakeLists.txt          (追加 if(CONFIG_ASH_UNIT_TEST) 段)
├── Kconfig                 (追加 ASH_UNIT_TEST 配置项)
├── Make.defs
├── Makefile                (追加 ifeq ($(CONFIG_ASH_UNIT_TEST), y) 段)
├── include/ash/file/
│   ├── file.h
│   └── file_path.h
├── src/file/
│   ├── file.cpp
│   └── file_path.cpp
└── tests/
    ├── ash_unit_test.cpp   (入口，#include "file/file_test.cpp" 等)
    └── file/
        ├── file_test.cpp
        └── file_path_test.cpp
```

关键占位符替换：`[module]`=ash, `[MODULE_UPPER]`=ASH, `[submodule]`=file, `[source]`=file/file_path, `[MODULE_UPPER_DEP]`=LIBASH

## Complete Example: Standalone Mode (ash_file module)

测试 `frameworks/runtimes/ash/src/file/file.cpp`，测试放在 `tests/velatest/ash_file/`。

```
tests/velatest/ash_file/
├── CMakeLists.txt          (CONFIG_GT_ASH_FILE_TEST)
├── Kconfig                 (GT_ASH_FILE_TEST + PRIORITY + STACKSIZE)
├── Make.defs               (CONFIGURED_APPS += .../tests/velatest/ash_file)
├── Makefile                (PROGNAME=gtest_ash_file, CXXFLAGS -I ash/include -I ash/src)
├── src/
│   └── file_test.cpp
└── gt_ash_file_entry.cpp   (入口，#include "src/file_test.cpp")
```

关键占位符替换：`[test_module]`=ash_file, `[TEST_MODULE_UPPER]`=ASH_FILE, `[module_path]`=frameworks/runtimes/ash, `[module_root_abs]`=`${APPDIR}/frameworks/runtimes/ash`, `[output_base]`=tests/velatest
