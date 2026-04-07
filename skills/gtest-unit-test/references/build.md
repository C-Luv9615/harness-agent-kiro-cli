# Build Workflow

This document describes the steps to update defconfig and trigger a build after test case generation.

**CRITICAL**: 生成完所有测试文件后，必须立即继续执行本文档的 Step 1～Step 3，不得直接结束对话或输出总结后停止。Step 7 是完整工作流的必要组成部分。

## Step 1: Ask for Lunch Config

**BLOCKING**: 生成完所有文件后，向用户展示以下预置的 lunch config 列表供选择：

```
请选择要编译的 lunch config：

  1. vendor/sim/boards/vela/configs/vela
  2. vendor/sim/boards/miwear/configs/miwear
  3. vendor/sim/boards/miot/configs/miot
  4. vendor/qemu/boards/vela/configs/goldfish-armeabi-v7a-ap
  5. vendor/qemu/boards/vela/configs/goldfish-arm64-v8a-ap
  6. vendor/qemu/boards/vela/configs/qemu-armeabi-v7a-ap
  7. vendor/qemu/boards/vela/configs/qemu-armeabi-v8m-ap
  8. vendor/qemu/boards/miwear/configs/miwear

请选择编号，或手动输入其他 lunch config 路径：
```

## Step 2: Analyze Dependencies and Update defconfig

用户输入后，根据输入路径定位 defconfig 文件：`[lunch_config_path]/defconfig`

例如用户输入 `vendor/sim/boards/vela/configs/vela`，则 defconfig 路径为：
`vendor/sim/boards/vela/configs/vela/defconfig`

### Step 2a: 检查被测函数的编译依赖

分析被测源文件所在目录的构建文件（`Makefile`、`CMakeLists.txt`、`Make.defs`），找出编译该源文件所需的 `CONFIG_*` 宏。常见方式：

- `Makefile` / `Make.defs` 中的 `ifeq ($(CONFIG_XXX),y)` 或 `ifneq ($(CONFIG_XXX),)` 条件
- `CMakeLists.txt` 中的 `if(CONFIG_XXX)` 条件
- 源文件中的 `#ifdef CONFIG_XXX` / `#if defined(CONFIG_XXX)` 守卫

将这些依赖配置项记录下来，在 Step 2b 中一并检查和追加。

### Step 2b: 追加配置项

在 defconfig 文件末尾追加以下配置项（如果已存在则跳过）：

**内嵌模式（Embedded）**：
```
CONFIG_LIB_GOOGLETEST=y
CONFIG_[MODULE_UPPER]_UNIT_TEST=y
```

**独立目录模式（Standalone）**：
```
CONFIG_LIB_GOOGLETEST=y
CONFIG_GT_[TEST_MODULE_UPPER]_TEST=y
CONFIG_GT_[TEST_MODULE_UPPER]_TEST_PRIORITY=100
CONFIG_GT_[TEST_MODULE_UPPER]_TEST_STACKSIZE=16384
```

以及 Step 2a 中发现的所有依赖配置项（如果 defconfig 中尚未启用）。

其中 `[MODULE_UPPER]` / `[TEST_MODULE_UPPER]` 为本次生成用例对应的模块名大写形式（与 Kconfig 中的配置项一致）。

追加完成后告知用户已更新的 defconfig 路径和追加的所有配置项内容。

## Step 3: Build

**BLOCKING**: 更新 defconfig 后，**不要合并命令**，依次使用 `executeBash` 工具执行以下命令：

1. ```bash
   source build/envsetup.sh
   ```

2. ```bash
   lunch [lunch_config_path]
   ```

3. 执行 `m` 前，先检查 out 目录下的 `.config` 是否包含本次新增的配置项。

   out 目录规则：将 `[lunch_config_path]` 最后三段路径（vendor/board/config）用 `_` 连接，例如 `vendor/sim/boards/vela/configs/vela` → `out/sim_vela_vela`。

   检查命令（根据目录模式选择对应的配置项名称）：

   **内嵌模式**：
   ```bash
   grep "CONFIG_[MODULE_UPPER]_UNIT_TEST" out/[out_dir]/.config
   ```

   **独立目录模式**：
   ```bash
   grep "CONFIG_GT_[TEST_MODULE_UPPER]_TEST" out/[out_dir]/.config
   ```

   - **包含**：直接执行 `m`
   - **不包含**（`.config` 存在但缺少配置项）：直接在 `out/[out_dir]/.config` 末尾追加缺少的配置项（与 Step 2b 追加到 defconfig 的内容一致），然后执行 `m`。CMake 构建系统会检测到 `.config` 变化并触发增量编译，无需 `distclean`
   - **`.config` 不存在**（out 目录不存在或为空）：直接执行 `m`，构建系统会自动从 defconfig 生成 `.config`

4. **独立目录模式（Standalone）且为新建测试目录时，必须在执行 `m` 前 touch 测试输出目录的上层 CMakeLists.txt**，强制 CMake 重新扫描子目录：

   ```bash
   touch [output_base]/CMakeLists.txt
   ```
   **内嵌模式或增量模式（目录已存在）时不需要此步骤**。

5. ```bash
   m > build.log 2>&1
   ```

其中 `[lunch_config_path]` 替换为用户输入的路径。

第五条命令设置 `ignoreWarning: true`，`timeout: 3600000`（60 分钟），将输出重定向到 `build.log`（覆盖写入，不追加）。

构建完成后，读取 build.log 中的 error 和 warning：

```bash
grep -iE "error|warning" build.log
```

显示构建结果（成功或失败信息）。

**如果构建输出中包含 error 或 warning**：

1. 判断是否由本次生成的代码引入（文件路径包含本次生成的测试目录）：
   - **是**：分析并修复问题，修复完成后重新依次执行上述命令，直到构建成功为止
   - **否**：将完整的 error/warning 信息输出给用户，提示用户自行解决，不自动修复

2. 以下为已知可忽略的链接器 warning，无论来源均无需处理：
   - `warning: relocation in read-only section`
   - `warning: creating DT_TEXTREL in a PIE`
