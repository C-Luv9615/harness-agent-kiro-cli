# NuttX/C Testing Patterns

**Load this reference when:** detected framework is cmocka or gtest in a NuttX/Vela project.

## TDD Strategy by Code Type

| Code Type | Strategy | Mock Needs |
|-----------|----------|------------|
| Pure functions (parse, validate, compute) | Direct input→output testing | None |
| Functions with syscall deps (ioctl, open) | `__wrap_` mock syscalls | Mock HAL/OS layer |
| State machines / event-driven | Mock event source, test each transition | Mock HAL + event injection |

## Pure Functions — No Mocks

```c
static void test_parse_config_null(void **state)
{
  assert_int_equal(module_parse_config(NULL), -EINVAL);
}

static void test_parse_config_max_value(void **state)
{
  struct config cfg = { .timeout = INT32_MAX };
  assert_int_equal(module_parse_config(&cfg), 0);
}
```

## With Dependencies — `__wrap_` Mock

Use `__wrap_` prefix + `--wrap` linker flag to mock syscalls/HAL.

```c
int __wrap_ioctl(int fd, int cmd, ...)
{
  check_expected(cmd);
  return mock_type(int);
}

static void test_read_sensor_ioctl_fail(void **state)
{
  expect_value(__wrap_ioctl, cmd, SENSOR_READ);
  will_return(__wrap_ioctl, -EIO);
  assert_int_equal(read_sensor(&ctx, &val), -EIO);
}
```

Linker config in Makefile/CMakeLists.txt:
```cmake
target_link_options(test_module PRIVATE -Wl,--wrap=ioctl,--wrap=open,--wrap=close)
```

## State Machines — Test Each Transition

Each state transition is one test case. Mock HAL layer, inject events, verify state change.

```c
static void test_event_X_in_state_A_goes_to_B(void **state)
{
  struct ctx ctx = { .state = STATE_A };
  event_t ev = { .type = EVENT_X };
  /* mock HAL calls if needed */
  assert_int_equal(state_machine_run(&ctx, &ev), 0);
  assert_int_equal(ctx.state, STATE_B);
}
```

## What Must Be Tested

| Category | Required |
|----------|----------|
| All public API functions | ✅ |
| NULL/invalid parameter handling | ✅ |
| Every error code branch | ✅ |
| Boundary values (0, max, overflow) | ✅ |
| State machine transitions (if any) | ✅ |
| Internal helpers | Only if complex |

## cmocka Test File Structure

```c
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <cmocka.h>

#include "module_under_test.h"

/* --- mocks --- */

int __wrap_ioctl(int fd, int cmd, ...)
{
  check_expected(cmd);
  return mock_type(int);
}

/* --- tests --- */

static void test_init_normal(void **state) { /* ... */ }
static void test_init_null(void **state) { /* ... */ }

/* --- main --- */

int main(void)
{
  const struct CMUnitTest tests[] =
    {
      cmocka_unit_test(test_init_normal),
      cmocka_unit_test(test_init_null),
    };

  return cmocka_run_group_tests(tests, NULL, NULL);
}
```

## Anti-Patterns for NuttX/C

- ❌ Mocking too many syscalls at once — mock only what the test needs
- ❌ Testing `__wrap_` mock behavior instead of module behavior
- ❌ Forgetting `--wrap` linker flag — test links to real syscall, mock never called
- ❌ Sharing mutable state between tests without `setup`/`teardown`
- ❌ Testing implementation details (internal struct layout) instead of API behavior
