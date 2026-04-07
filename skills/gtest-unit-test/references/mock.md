# Mock 规则（Google Mock）

本文档描述 C++ 单元测试中的 mock 实现规则。

## NuttX Flat Build 约束（最高优先级）

NuttX 使用 flat build 模型：同一个 Makefile 中的多个 `PROGNAME` 共享 `CXXSRCS`，所有 `nuttx_add_application` 最终链接到同一个二进制中。这对 mock 策略有决定性影响：

| Mock 方式 | NuttX flat build 下可用？ | 说明 |
|-----------|--------------------------|------|
| gmock（虚函数接口，方案 A） | **可以** | Mock 类在 anonymous namespace 内，不涉及符号重定向，不会污染其他测试 |
| 预处理器重定向（方案 B） | **不行** | `_under_test.cpp` 中重定向后的符号和库中真实符号在同一链接空间冲突；独立 PROGNAME 的 `RUN_ALL_TESTS()` 也会跑到其他测试的用例 |
| 直接用真实依赖 | **可以** | 如果依赖足够轻量（如 `MessageLoop::Create()`），优先选择，无需 mock |

### 决策优先级

1. **优先直接用真实依赖** — 如果依赖可以在测试中轻量构造（如 `MessageLoop::Create()`），直接用真实对象，不 mock
2. **其次用 gmock** — 如果依赖是 C++ 抽象类/接口且通过依赖注入传入，使用 gmock `MOCK_METHOD`
3. **禁止预处理器重定向** — 在 NuttX flat build 下不要使用方案 B，会导致符号冲突和测试用例串扰

### 为什么预处理器重定向在 flat build 下不可行

1. **符号冲突**：`_under_test.cpp` 中通过 `#include "[source].cpp"` 拉入的被测代码，其符号（函数、全局变量）会和 `ash` 库中真实编译的同名符号在同一链接空间冲突
2. **测试用例串扰**：如果创建独立 PROGNAME，`RUN_ALL_TESTS()` 会发现并执行所有链接进来的 `TEST_F`（包括其他测试文件的用例），无法实现真正的隔离
3. **假类实现冲突**：如果在测试文件中提供假的类成员函数实现（如假 `MessageLoop::Current()`），会和库中真实的实现冲突

## 方案 A：gmock（C++ 接口 Mock）— 推荐

### 适用场景

- 被测代码通过依赖注入使用抽象类/接口
- 被测类的构造函数或方法接受接口指针/引用
- 需要验证方法调用次数、参数、顺序

### A1. Mock 类定义

使用 `MOCK_METHOD` 宏为接口中的每个虚函数生成 mock 实现：

```cpp
class MockFileSystem : public IFileSystem {
 public:
  MOCK_METHOD(bool, exists, (const std::string& path), (const, override));
  MOCK_METHOD(int, open, (const std::string& path, int flags), (override));
  MOCK_METHOD(ssize_t, read, (int fd, void* buf, size_t count), (override));
  MOCK_METHOD(int, close, (int fd), (override));
};
```

`MOCK_METHOD` 语法：
```cpp
MOCK_METHOD(return_type, method_name, (arg_types...), (qualifiers...));
```

qualifiers 包括：`const`, `override`, `noexcept`, `final` 等。

### A2. 设置期望（Expectations）

在测试用例中使用 `EXPECT_CALL` 设置期望：

```cpp
TEST_F(FileTest, Read_ValidFd_ReturnsData) {
  MockFileSystem mock_fs;

  EXPECT_CALL(mock_fs, open("/tmp/test.txt", O_RDONLY))
      .Times(1)
      .WillOnce(::testing::Return(3));

  EXPECT_CALL(mock_fs, read(3, ::testing::_, 64))
      .WillOnce(::testing::Return(64));

  EXPECT_CALL(mock_fs, close(3))
      .WillOnce(::testing::Return(0));

  FileReader reader(&mock_fs);
  auto data = reader.readFile("/tmp/test.txt", 64);
  EXPECT_EQ(data.size(), 64);
}
```

### A3. 常用 Matchers

| Matcher | 说明 | 示例 |
|---------|------|------|
| `::testing::_` | 匹配任意值 | `EXPECT_CALL(mock, fn(::testing::_))` |
| `::testing::Eq(v)` | 等于 v | `EXPECT_CALL(mock, fn(Eq(42)))` |
| `::testing::Ne(v)` | 不等于 v | `EXPECT_CALL(mock, fn(Ne(0)))` |
| `::testing::Gt(v)` | 大于 v | `EXPECT_CALL(mock, fn(Gt(0)))` |
| `::testing::Lt(v)` | 小于 v | `EXPECT_CALL(mock, fn(Lt(100)))` |
| `::testing::IsNull()` | 为 nullptr | `EXPECT_CALL(mock, fn(IsNull()))` |
| `::testing::NotNull()` | 非 nullptr | `EXPECT_CALL(mock, fn(NotNull()))` |
| `::testing::HasSubstr(s)` | 包含子串 | `EXPECT_CALL(mock, fn(HasSubstr("error")))` |
| `::testing::StrEq(s)` | 字符串相等 | `EXPECT_CALL(mock, fn(StrEq("hello")))` |

### A4. 常用 Actions

| Action | 说明 | 示例 |
|--------|------|------|
| `Return(v)` | 返回值 v | `.WillOnce(Return(42))` |
| `ReturnRef(v)` | 返回引用 | `.WillOnce(ReturnRef(obj))` |
| `ReturnNull()` | 返回 nullptr | `.WillOnce(ReturnNull())` |
| `Throw(e)` | 抛出异常 | `.WillOnce(Throw(std::runtime_error("err")))` |
| `DoDefault()` | 执行默认行为 | `.WillOnce(DoDefault())` |
| `Invoke(f)` | 调用函数 f | `.WillOnce(Invoke(myFunc))` |
| `SetArgPointee<N>(v)` | 设置第 N 个指针参数指向的值 | `.WillOnce(SetArgPointee<1>(data))` |
| `DoAll(a1, a2)` | 依次执行多个 action | `.WillOnce(DoAll(SetArgPointee<1>(buf), Return(64)))` |

### A5. 调用次数控制

```cpp
EXPECT_CALL(mock, fn(_))
    .Times(1);                    // 恰好 1 次

EXPECT_CALL(mock, fn(_))
    .Times(::testing::AtLeast(1)); // 至少 1 次

EXPECT_CALL(mock, fn(_))
    .Times(::testing::AtMost(3));  // 至多 3 次

EXPECT_CALL(mock, fn(_))
    .Times(::testing::AnyNumber()); // 任意次数
```

### A6. 多次调用返回不同值

```cpp
EXPECT_CALL(mock, read(_, _, _))
    .WillOnce(Return(64))         // 第一次返回 64
    .WillOnce(Return(32))         // 第二次返回 32
    .WillOnce(Return(-1));        // 第三次返回 -1

// 或使用 WillRepeatedly 设置默认返回值
EXPECT_CALL(mock, read(_, _, _))
    .WillOnce(Return(64))
    .WillRepeatedly(Return(0));   // 后续调用都返回 0
```

### A7. Mock 类放置位置

Mock 类定义在测试文件中，位于 anonymous namespace 内：

```cpp
namespace [module] {
namespace {

// Mock classes
class MockExternalAPIs : public ExternalAPIs {
 public:
  MOCK_METHOD(bool, fn1, (const std::string& name), (override));
};

// Test fixture
class [Source]Test : public ::testing::Test {
 protected:
  MockExternalAPIs mock_apis_;
};

// Test cases...

}  // namespace
}  // namespace [module]
```

如果多个测试文件共享同一个 Mock 类，可以提取到独立的头文件 `tests/mocks/mock_[class].h`。


## 不可 Mock 的场景及替代方案

某些依赖无法通过 gmock 进行 mock，需要使用替代策略：

### 静态方法 / 工厂方法

静态方法（如 `MessageLoop::Create()`、`Timer::Create()`）不是虚函数，无法用 gmock 的 `MOCK_METHOD` 拦截。

**替代方案**：直接使用真实依赖。如果静态方法足够轻量且无副作用（如创建一个内存对象），在测试中直接调用真实实现：

```cpp
class TimerTest : public ::testing::Test {
 protected:
  void SetUp() override {
    loop_ = MessageLoop::Create();  // 直接用真实依赖
  }
  void TearDown() override {
    loop_.reset();
  }
  std::unique_ptr<MessageLoop> loop_;
};
```

### 具体类（无虚函数）

如果依赖是具体类且没有虚函数，无法直接用 gmock。

**替代方案**（按优先级）：
1. 直接使用真实依赖（如果足够轻量）
2. 提取接口：将依赖的公有方法抽象为接口类，被测代码改为依赖接口
3. 模板注入：将依赖类型作为模板参数，测试时传入 mock 类型

### 全局函数 / C 函数

NuttX flat build 下，预处理器重定向（方案 B）不可用。如果被测代码调用了全局 C 函数（如 `open()`、`read()`）：

**替代方案**：
1. 直接使用真实系统调用（如果测试环境允许）
2. 将 C 函数调用封装到一个 C++ 接口类中，通过依赖注入传入，然后用 gmock mock 该接口

```cpp
// 封装 C 函数为接口
class IFileOps {
 public:
  virtual ~IFileOps() = default;
  virtual int open(const char* path, int flags) = 0;
  virtual ssize_t read(int fd, void* buf, size_t count) = 0;
  virtual int close(int fd) = 0;
};

// Mock 实现
class MockFileOps : public IFileOps {
 public:
  MOCK_METHOD(int, open, (const char* path, int flags), (override));
  MOCK_METHOD(ssize_t, read, (int fd, void* buf, size_t count), (override));
  MOCK_METHOD(int, close, (int fd), (override));
};
```

## 注意事项

1. **Mock 类必须在 anonymous namespace 内定义**：避免不同测试文件中的同名 Mock 类在链接时冲突。这在 NuttX flat build 下尤为重要，因为所有测试文件最终链接到同一个二进制中。

2. **类成员函数的 mock 实现不得放在 anonymous namespace 内**：如果需要提供类成员函数的假实现（fake），该实现必须在 `namespace [module]` 内但在 anonymous namespace 外，否则链接器找不到符号。

3. **EXPECT_CALL 必须在调用被测函数之前设置**：gmock 的期望是"先声明后验证"模式，如果在被测函数调用之后才设置 `EXPECT_CALL`，期望不会生效。

4. **避免过度 mock**：只 mock 真正需要隔离的外部依赖。如果依赖足够轻量（创建成本低、无副作用、不依赖外部资源），直接使用真实依赖比 mock 更简单可靠。

## 选择指南

根据被测代码的依赖类型，按以下流程选择 mock 策略：

```
依赖是否足够轻量（可直接构造、无副作用）？
  ├── 是 → 直接使用真实依赖（推荐）
  └── 否 → 依赖是否有虚函数接口？
              ├── 是 → 使用 gmock MOCK_METHOD（方案 A）
              └── 否 → 依赖是 C++ 具体类？
                          ├── 是 → 提取接口 + gmock，或使用模板注入
                          └── 否（C 函数）→ 封装为接口类 + gmock
```

| 依赖类型 | NuttX flat build 下推荐方案 | 说明 |
|----------|---------------------------|------|
| 轻量 C++ 对象（如 `MessageLoop`） | 直接用真实依赖 | 最简单，无 mock 开销 |
| C++ 抽象类/接口 | gmock `MOCK_METHOD` | 标准 gmock 用法 |
| C++ 具体类（无虚函数） | 提取接口 + gmock | 需要重构被测代码 |
| C 函数/系统调用 | 封装为接口类 + gmock | 需要重构被测代码 |
| 静态方法/工厂方法 | 直接用真实依赖 | 无法 mock，直接调用 |

## 方案 B 参考（预处理器重定向）— NuttX flat build 下不可用

> ⚠️ 以下内容仅作为参考存档。在 NuttX flat build 环境下，方案 B **不可使用**，原因见本文档顶部"NuttX Flat Build 约束"一节。

### 原理

通过预处理器宏将被测代码中调用的外部函数重定向到测试文件中的假实现：

1. `[source]_redirects.h` — 定义 `#define real_func fake_func` 宏替换表
2. `[source]_under_test.cpp` — 桥接文件，先 `#include` 重定向头文件，再 `#include` 被测源码
3. 测试文件中提供 `fake_func` 的实现

### 不可用原因（NuttX flat build）

1. `_under_test.cpp` 通过 `#include "[submodule]/[source].cpp"` 拉入被测代码，其符号和库中真实编译的同名符号在同一链接空间冲突
2. 独立 PROGNAME 的 `RUN_ALL_TESTS()` 会发现并执行所有链接进来的 `TEST_F`，无法隔离
3. 假类成员函数实现和库中真实实现冲突

### 为什么 `_ut` 后缀重命名在 C++ 下不可行

- 类成员函数名（如 `Create`）太通用，`#define Create Create_ut` 会波及所有类的同名方法（`MessageLoop::Create`、`Timer::Create` 等），范围不可控
- 构造函数名必须与类名一致，无法通过 `#define` 重命名
- 如果 `#define Timer Timer_ut` 重命名整个类，头文件中的类定义、前向声明、友元声明、模板特化等全部被破坏
