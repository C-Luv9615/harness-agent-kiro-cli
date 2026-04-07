# Go Testing Patterns

**Load this reference when:** detected framework is go test in a Go project.

## Project Structure

```
mypackage/
├── auth.go
├── auth_test.go               ← co-located (mandatory in Go)
├── service.go
├── service_test.go
└── testdata/                   ← test fixtures
    └── valid_config.json
```

Go enforces co-located test files. No separate `tests/` directory.

## Naming

- Test file: `{source}_test.go` (same package)
- Test function: `func Test{FunctionName}_{Scenario}(t *testing.T)`
- Example: `TestValidateEmail_EmptyString`, `TestParseConfig_MissingKey`
- Benchmark: `func Benchmark{Name}(b *testing.B)`

## Test Structure — Table-Driven Tests (idiomatic)

```go
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name  string
        email string
        want  bool
    }{
        {"valid email", "user@example.com", true},
        {"missing @", "invalid", false},
        {"empty string", "", false},
        {"minimal valid", "a@b.c", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := ValidateEmail(tt.email)
            if got != tt.want {
                t.Errorf("ValidateEmail(%q) = %v, want %v", tt.email, got, tt.want)
            }
        })
    }
}
```

## Error Testing

```go
func TestParseConfig_MissingFile(t *testing.T) {
    _, err := ParseConfig("/nonexistent")
    if err == nil {
        t.Fatal("expected error for missing file")
    }
    if !errors.Is(err, os.ErrNotExist) {
        t.Errorf("got %v, want os.ErrNotExist", err)
    }
}

func TestDivide_ByZero(t *testing.T) {
    _, err := Divide(10, 0)
    if err == nil {
        t.Fatal("expected error for division by zero")
    }
    if !strings.Contains(err.Error(), "division by zero") {
        t.Errorf("unexpected error message: %v", err)
    }
}
```

## Mock Strategy

### When to mock

| Dependency | Mock? | How |
|-----------|-------|-----|
| External API / HTTP | ✅ Always | `httptest.NewServer` |
| Database | ✅ Usually | Interface + mock impl |
| File system | ✅ Usually | `t.TempDir()` or `os.CreateTemp` |
| Pure functions | ❌ Never | Use real implementation |
| Time | ✅ When needed | Inject clock interface |

### Interface-based mocking (idiomatic Go)

```go
// Define interface for dependency
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
}

// Test mock
type mockUserRepo struct {
    users map[string]*User
}

func (m *mockUserRepo) FindByID(_ context.Context, id string) (*User, error) {
    u, ok := m.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return u, nil
}

func TestGetUser_Found(t *testing.T) {
    repo := &mockUserRepo{
        users: map[string]*User{"1": {Name: "Alice"}},
    }
    svc := NewUserService(repo)

    user, err := svc.GetUser(context.Background(), "1")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("got %q, want Alice", user.Name)
    }
}
```

### httptest for HTTP dependencies

```go
func TestFetchUser(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path != "/users/123" {
            t.Errorf("unexpected path: %s", r.URL.Path)
        }
        w.Header().Set("Content-Type", "application/json")
        fmt.Fprint(w, `{"id":"123","name":"Alice"}`)
    }))
    defer srv.Close()

    client := NewAPIClient(srv.URL)
    user, err := client.FetchUser("123")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("got %q, want Alice", user.Name)
    }
}
```

## Test Helpers

```go
// t.Helper() marks function as helper — errors report caller's line
func assertNoError(t *testing.T, err error) {
    t.Helper()
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
}

func assertEqualUser(t *testing.T, got, want *User) {
    t.Helper()
    if got.Name != want.Name || got.ID != want.ID {
        t.Errorf("got %+v, want %+v", got, want)
    }
}
```

## Testdata & TempDir

```go
func TestParseConfig(t *testing.T) {
    // Read from testdata/ (committed fixtures)
    data, err := os.ReadFile("testdata/valid_config.json")
    assertNoError(t, err)

    config, err := ParseConfig(data)
    assertNoError(t, err)
    if config.Port != 8080 {
        t.Errorf("got port %d, want 8080", config.Port)
    }
}

func TestWriteOutput(t *testing.T) {
    // Use t.TempDir() for writable temp space (auto-cleaned)
    dir := t.TempDir()
    path := filepath.Join(dir, "output.txt")

    err := WriteOutput(path, "hello")
    assertNoError(t, err)

    got, _ := os.ReadFile(path)
    if string(got) != "hello" {
        t.Errorf("got %q, want hello", got)
    }
}
```

## Config Integration

No config file needed. Go test tooling is built-in:

```bash
go test ./...                    # run all tests
go test -v ./...                 # verbose
go test -run TestValidate ./...  # filter by name
go test -cover ./...             # coverage summary
go test -coverprofile=c.out ./...  # coverage file
go tool cover -html=c.out       # HTML report
```

## Anti-Patterns

- ❌ `if err != nil { t.Log(err) }` — use `t.Fatal` or `t.Fatalf`, not `t.Log`
- ❌ Testing unexported functions from `_test.go` in different package — test through public API
- ❌ `t.Parallel()` with shared mutable state — parallel tests must be independent
- ❌ Skipping `t.Helper()` in helper functions — error messages point to wrong line
- ❌ Single giant test function — use table-driven subtests with `t.Run`
- ❌ `testify` assertions everywhere — standard library is sufficient for most cases; use testify only if project already depends on it
