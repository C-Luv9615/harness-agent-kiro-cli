# pytest Testing Patterns

**Load this reference when:** detected framework is pytest in a Python project.

## Project Structure

```
src/
├── mypackage/
│   ├── __init__.py
│   ├── auth.py
│   └── services.py
tests/
├── conftest.py                ← shared fixtures
├── test_auth.py
└── test_services.py
```

**Convention detection:** check for existing `tests/` dir or co-located `test_*.py`. Default: separate `tests/` directory.

## Naming

- Test file: `test_{module}.py`
- Test function: `test_{behavior_description}` — `test_rejects_empty_email`, not `test_validate_1`
- Test class (optional): `TestClassName` — group related tests

## Test Structure

```python
import pytest
from mypackage.auth import AuthService

class TestAuthService:
    def setup_method(self):
        self.service = AuthService()

    def test_login_valid_credentials(self):
        result = self.service.login("user@example.com", "password123")
        assert result.success is True
        assert result.token is not None

    def test_login_invalid_password(self):
        result = self.service.login("user@example.com", "wrong")
        assert result.success is False
        assert result.error == "Invalid credentials"

    def test_login_empty_email_raises(self):
        with pytest.raises(ValueError, match="Email required"):
            self.service.login("", "password123")
```

## Fixtures

```python
# conftest.py — shared across all tests in directory

@pytest.fixture
def db_session():
    """Provide a clean database session."""
    session = create_test_session()
    yield session
    session.rollback()
    session.close()

@pytest.fixture
def auth_service(db_session):
    """AuthService with test database."""
    return AuthService(db=db_session)

@pytest.fixture(params=["sqlite", "postgres"])
def db_backend(request):
    """Parametrize across database backends."""
    return create_backend(request.param)
```

```python
# test_auth.py — uses fixtures by name

def test_create_user(auth_service):
    user = auth_service.create_user("alice@example.com")
    assert user.id is not None

def test_duplicate_user_raises(auth_service):
    auth_service.create_user("alice@example.com")
    with pytest.raises(DuplicateError):
        auth_service.create_user("alice@example.com")
```

## Mock Strategy

### When to mock

| Dependency | Mock? | How |
|-----------|-------|-----|
| External API / HTTP | ✅ Always | `responses`, `httpx_mock`, or `unittest.mock.patch` |
| Database | ✅ Usually | Mock repository, or use test DB with fixture |
| File system | ✅ Usually | `tmp_path` fixture (built-in) or `mock_open` |
| Pure functions | ❌ Never | Use real implementation |
| Time/Date | ✅ When needed | `freezegun` or `time_machine` |
| Environment vars | ✅ When needed | `monkeypatch.setenv` |

### unittest.mock.patch

```python
from unittest.mock import patch, MagicMock

def test_fetch_user_api_failure():
    with patch("mypackage.services.requests.get") as mock_get:
        mock_get.return_value = MagicMock(status_code=500)
        result = fetch_user("123")
        assert result is None
        mock_get.assert_called_once_with("https://api.example.com/users/123")

# Or as decorator
@patch("mypackage.services.requests.get")
def test_fetch_user_success(mock_get):
    mock_get.return_value = MagicMock(
        status_code=200,
        json=lambda: {"id": "123", "name": "Alice"},
    )
    user = fetch_user("123")
    assert user["name"] == "Alice"
```

### monkeypatch (pytest built-in)

```python
def test_config_from_env(monkeypatch):
    monkeypatch.setenv("API_KEY", "test-key-123")
    config = load_config()
    assert config.api_key == "test-key-123"

def test_read_file(tmp_path):
    f = tmp_path / "data.txt"
    f.write_text("hello")
    result = read_file(str(f))
    assert result == "hello"
```

## Parametrize

```python
@pytest.mark.parametrize("email,valid", [
    ("user@example.com", True),
    ("invalid", False),
    ("", False),
    ("a@b.c", True),
])
def test_validate_email(email, valid):
    assert validate_email(email) == valid
```

## Config Integration

### `pyproject.toml`

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
addopts = "-v --strict-markers"
markers = [
    "slow: marks tests as slow",
    "integration: marks integration tests",
]

[tool.coverage.run]
source = ["src/mypackage"]
omit = ["tests/*"]

[tool.coverage.report]
fail_under = 80
```

## Anti-Patterns

- ❌ `assert True` — proves nothing
- ❌ Mocking the function under test — test real code, mock dependencies
- ❌ `@patch` on wrong module path — patch where it's imported, not where it's defined
- ❌ Fixtures that do too much — keep fixtures focused, compose them
- ❌ Tests that depend on execution order — each test must be independent
- ❌ Catching exceptions to assert — use `pytest.raises` context manager
