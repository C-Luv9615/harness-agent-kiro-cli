# Rust Testing Patterns

**Load this reference when:** detected framework is cargo test in a Rust project.

## Project Structure

### Unit tests (inline)

```rust
// src/auth.rs

pub fn validate_token(token: &str) -> Result<Claims, AuthError> {
    // implementation
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_token() {
        let claims = validate_token("valid.jwt.token").unwrap();
        assert_eq!(claims.user_id, "123");
    }

    #[test]
    fn expired_token() {
        let err = validate_token("expired.jwt.token").unwrap_err();
        assert!(matches!(err, AuthError::Expired));
    }
}
```

### Integration tests (separate)

```
src/
├── lib.rs
├── auth.rs
tests/
├── auth_integration.rs       ← each file is a separate test binary
└── common/
    └── mod.rs                 ← shared test helpers
```

```rust
// tests/auth_integration.rs
use mylib::auth::validate_token;

#[test]
fn full_auth_flow() {
    let token = create_test_token();
    let claims = validate_token(&token).unwrap();
    assert_eq!(claims.role, "admin");
}
```

## Naming

- Unit test module: `#[cfg(test)] mod tests` inside source file
- Test function: `fn {behavior_description}()` — `fn rejects_empty_input()`, not `fn test1()`
- Integration test file: `tests/{feature}_integration.rs`

## Test Patterns

### Result assertions

```rust
#[test]
fn parse_valid_config() {
    let config = parse_config("key=value").unwrap();
    assert_eq!(config.key, "value");
}

#[test]
fn parse_invalid_config() {
    let err = parse_config("invalid").unwrap_err();
    assert_eq!(err.to_string(), "Missing '=' delimiter");
}

#[test]
#[should_panic(expected = "index out of bounds")]
fn access_out_of_bounds() {
    let v = vec![1, 2, 3];
    let _ = v[10];
}
```

### Parameterized (via macro or rstest)

```rust
// With rstest crate
use rstest::rstest;

#[rstest]
#[case("user@example.com", true)]
#[case("invalid", false)]
#[case("", false)]
fn test_validate_email(#[case] email: &str, #[case] expected: bool) {
    assert_eq!(validate_email(email), expected);
}
```

### Async tests

```rust
// With tokio
#[tokio::test]
async fn fetch_user_success() {
    let user = fetch_user("123").await.unwrap();
    assert_eq!(user.name, "Alice");
}
```

## Mock Strategy

### When to mock

| Dependency | Mock? | How |
|-----------|-------|-----|
| External API / HTTP | ✅ Always | `mockito`, `wiremock` |
| Database | ✅ Usually | Trait-based injection + mock impl |
| File system | ✅ Usually | `tempfile` crate |
| Pure functions | ❌ Never | Use real implementation |
| Time | ✅ When needed | Inject clock trait |

### Trait-based mocking (idiomatic Rust)

```rust
// Define trait for dependency
pub trait UserRepository {
    fn find_by_id(&self, id: &str) -> Option<User>;
}

// Production implementation
pub struct PgUserRepository { /* ... */ }
impl UserRepository for PgUserRepository { /* ... */ }

// Test mock
#[cfg(test)]
struct MockUserRepository {
    users: HashMap<String, User>,
}

#[cfg(test)]
impl UserRepository for MockUserRepository {
    fn find_by_id(&self, id: &str) -> Option<User> {
        self.users.get(id).cloned()
    }
}

#[test]
fn service_returns_user() {
    let mut repo = MockUserRepository { users: HashMap::new() };
    repo.users.insert("1".into(), User { name: "Alice".into() });

    let service = UserService::new(Box::new(repo));
    let user = service.get_user("1").unwrap();
    assert_eq!(user.name, "Alice");
}
```

### mockall crate (when trait-based is too verbose)

```rust
use mockall::automock;

#[automock]
pub trait Cache {
    fn get(&self, key: &str) -> Option<String>;
    fn set(&mut self, key: &str, value: &str);
}

#[test]
fn uses_cache() {
    let mut mock = MockCache::new();
    mock.expect_get()
        .with(eq("key"))
        .returning(|_| Some("cached".into()));

    let result = lookup(&mock, "key");
    assert_eq!(result, "cached");
}
```

## Config Integration

### `Cargo.toml`

```toml
[dev-dependencies]
rstest = "0.18"
mockall = "0.12"
tempfile = "3"
tokio = { version = "1", features = ["test-util", "macros"] }
wiremock = "0.6"
```

## Anti-Patterns

- ❌ `#[test] fn test_it_works()` — vague name
- ❌ `unwrap()` in tests without context — use `unwrap()` only when failure is the test failing; for expected errors use `unwrap_err()`
- ❌ Testing private functions directly — test through public API
- ❌ `#[ignore]` without reason — always annotate why
- ❌ Large test functions — split into focused tests, one assertion per behavior
