# Testing Strategy for nah

## Testing Philosophy

**nah** is a framework library used by downstream projects. Testing strategy focuses on:

- API correctness and stability
- Framework behavior (routing, apply, backend)
- Integration patterns
- Example usage validation

**Test Pyramid**:

```
       /\
      / Ex \        Example/Integration tests
     /------\
    /  Unit  \      Many focused unit tests
   /----------\
```

## Running Tests

### All Tests

```bash
# Run all tests
make test
go test ./...

# With verbose output
go test -v ./...

# With race detector
go test -race ./...

# With coverage
go test -cover ./...
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### Specific Package Tests

```bash
# Test router package
go test ./pkg/router

# Test apply package
go test ./pkg/apply

# Test with verbose output
go test -v ./pkg/router

# Test specific function
go test -v -run TestRouterStart ./pkg/router
```

### CI Validation

```bash
# Full CI validation (what GitHub Actions runs)
make validate-ci

# This runs:
# 1. go generate - Regenerate code
# 2. go mod tidy - Tidy dependencies
# 3. Git dirty check - Fail if uncommitted changes
```

## Test Organization

### Test Files

Tests live in `*_test.go` files next to source code:

```
pkg/router/
├── router.go
├── router_test.go
├── handler.go
├── handler_test.go
```

### Test Packages

Tests use either:

- **Same package** (`package router`): For testing internal/private functions
- **Test package** (`package router_test`): For testing public API as external user

**Prefer test packages** to verify public API works as documented.

### Test Fixtures

Example from `pkg/apply/`:

```go
// testdata/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
data:
  key: value
```

## Testing Patterns

### Table-Driven Tests

**Recommended pattern** for testing multiple scenarios:

```go
func TestRouterRouting(t *testing.T) {
    tests := []struct {
        name     string
        gvk      schema.GroupVersionKind
        want     bool
    }{
        {
            name: "matches registered type",
            gvk:  corev1.SchemeGroupVersion.WithKind("ConfigMap"),
            want: true,
        },
        {
            name: "no match for unregistered type",
            gvk:  appsv1.SchemeGroupVersion.WithKind("Deployment"),
            want: false,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            router := setupTestRouter(t)
            got := router.HasHandler(tt.gvk)
            if got != tt.want {
                t.Errorf("HasHandler() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

### Testify Assertions

Use testify for cleaner assertions:

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestSomething(t *testing.T) {
    result, err := doSomething()
    
    // require: Fails test immediately if false
    require.NoError(t, err)
    require.NotNil(t, result)
    
    // assert: Continues test even if false
    assert.Equal(t, "expected", result.Value)
    assert.Contains(t, result.List, "item")
}
```

### Autogold Snapshot Testing

Use autogold for snapshot/golden file testing:

```go
import (
    "testing"
    "github.com/hexops/autogold/v2"
)

func TestYAMLOutput(t *testing.T) {
    output := generateYAML()
    
    // First run creates snapshot, subsequent runs compare
    autogold.ExpectFile(t, output)
}
```

**Update snapshots**:

```bash
go test ./... -update
```

### Test Helpers

Mark helper functions with `t.Helper()`:

```go
func setupTestRouter(t *testing.T) *router.Router {
    t.Helper()  // Makes test failures point to caller, not this function
    
    scheme := runtime.NewScheme()
    _ = corev1.AddToScheme(scheme)
    
    r, err := NewRouter("test", &Options{
        Scheme: scheme,
    })
    if err != nil {
        t.Fatalf("failed to create router: %v", err)
    }
    
    return r
}
```

## Testing Specific Components

### Testing Router

**Test route registration**:

```go
func TestTypeRegistration(t *testing.T) {
    router := setupTestRouter(t)
    
    // Register handler
    router.Type(&corev1.ConfigMap{}).HandlerFunc(func(req Request, resp Response) error {
        return nil
    })
    
    // Verify registered
    assert.True(t, router.HasHandler(corev1.SchemeGroupVersion.WithKind("ConfigMap")))
}
```

**Test middleware**:

```go
func TestMiddleware(t *testing.T) {
    var called bool
    middleware := func(req Request, resp Response, next Handler) error {
        called = true
        return next.Handle(req, resp)
    }
    
    router := setupTestRouter(t)
    router.Type(&corev1.ConfigMap{}).
        Middleware(middleware).
        HandlerFunc(func(req Request, resp Response) error {
            return nil
        })
    
    // Trigger handler
    // ... invoke router ...
    
    assert.True(t, called, "middleware should be called")
}
```

### Testing Apply

**Test declarative reconciliation**:

```go
func TestApply(t *testing.T) {
    ctx := context.Background()
    client := fake.NewClientBuilder().WithScheme(scheme.Scheme).Build()
    
    apply := apply.New(client)
    
    owner := &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "owner",
            Namespace: "default",
        },
    }
    
    desired := []client.Object{
        &corev1.ConfigMap{
            ObjectMeta: metav1.ObjectMeta{
                Name:      "config",
                Namespace: "default",
            },
            Data: map[string]string{"key": "value"},
        },
    }
    
    // Apply desired state
    err := apply.Apply(ctx, owner, desired...)
    require.NoError(t, err)
    
    // Verify created
    var cm corev1.ConfigMap
    err = client.Get(ctx, client.ObjectKey{
        Namespace: "default",
        Name:      "config",
    }, &cm)
    require.NoError(t, err)
    assert.Equal(t, "value", cm.Data["key"])
}
```

**Test pruning**:

```go
func TestApplyPruning(t *testing.T) {
    ctx := context.Background()
    client := fake.NewClientBuilder().WithScheme(scheme.Scheme).Build()
    
    apply := apply.New(client)
    owner := testOwner()
    
    // First apply - create resource
    apply.Apply(ctx, owner, testConfigMap("cm1"))
    
    // Second apply - remove resource (should prune)
    apply.Apply(ctx, owner) // No resources
    
    // Verify pruned
    var cm corev1.ConfigMap
    err := client.Get(ctx, client.ObjectKey{Name: "cm1", Namespace: "default"}, &cm)
    assert.True(t, apierrors.IsNotFound(err), "resource should be pruned")
}
```

### Testing Backend

**Test with fake client**:

```go
func TestBackendGet(t *testing.T) {
    ctx := context.Background()
    
    // Create fake client with initial data
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test",
            Namespace: "default",
        },
    }
    
    fakeClient := fake.NewClientBuilder().
        WithScheme(scheme.Scheme).
        WithObjects(cm).
        Build()
    
    backend := runtime.NewBackend(fakeClient, scheme.Scheme, ...)
    
    // Test Get
    var result corev1.ConfigMap
    err := backend.Get(ctx, client.ObjectKey{
        Name:      "test",
        Namespace: "default",
    }, &result)
    
    require.NoError(t, err)
    assert.Equal(t, "test", result.Name)
}
```

### Testing Leader Election

**Test lease-based election** (requires integration test):

```go
// +build integration

func TestLeaderElection(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    config := leader.NewDefaultElectionConfig("", "test-controller", restConfig)
    
    elector, err := leader.NewElector(config)
    require.NoError(t, err)
    
    elected := make(chan bool, 1)
    
    go func() {
        err := elector.Run(ctx, func(ctx context.Context) {
            elected <- true
            <-ctx.Done()
        })
        if err != nil {
            t.Logf("elector error: %v", err)
        }
    }()
    
    // Wait for election
    select {
    case <-elected:
        // Success
    case <-time.After(10 * time.Second):
        t.Fatal("leader election timeout")
    }
}
```

## Integration Testing

### Testing with Downstream Projects

**Primary integration test**: obot-entraid

See `downstream_consumers` memory for testing nah changes with obot-entraid:

```bash
# In obot-entraid
echo 'replace github.com/obot-platform/nah => ../nah' >> go.mod
go mod tidy
make test
make dev
```

### Testing Examples

Create example applications in `examples/` directory:

```
examples/
├── basic-controller/
│   ├── main.go
│   └── README.md
└── with-apply/
    ├── main.go
    └── README.md
```

**Test examples compile**:

```bash
# In CI or make target
for example in examples/*/; do
    echo "Building $example"
    (cd "$example" && go build .)
done
```

## CI/CD Integration

### GitHub Actions

File: `.github/workflows/test.yaml`

```yaml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-go@v5
        with:
          go-version: '1.24.0'
      
      - name: Setup CI environment
        run: make setup-ci-env
      
      - name: Validate CI
        run: make validate-ci
      
      - name: Lint
        run: make validate
      
      - name: Test
        run: make test
```

### Makefile Targets

```makefile
.PHONY: test
test:
\tgo test ./...

.PHONY: test-race
test-race:
\tgo test -race ./...

.PHONY: test-cover
test-cover:
\tgo test -coverprofile=coverage.out ./...
\tgo tool cover -html=coverage.out

.PHONY: validate-ci
validate-ci:
\tgo generate
\tgo mod tidy
\t@if [ -n "$$(git status --porcelain)" ]; then \
\t\techo "Encountered dirty repo!"; \
\t\tgit status --porcelain; \
\t\tgit diff; \
\t\texit 1; \
\tfi
```

## Test Coverage

### Generating Coverage Reports

```bash
# Generate coverage
go test -coverprofile=coverage.out ./...

# View in terminal
go tool cover -func=coverage.out

# View in browser
go tool cover -html=coverage.out

# Coverage by package
go test -coverprofile=coverage.out ./... && \
  go tool cover -func=coverage.out | grep -E '^total:|pkg/'
```

### Coverage Goals

**Target coverage** (framework library):

- Core packages (router, apply, backend): 70%+ coverage
- Utility packages: 60%+ coverage
- Overall: 65%+ coverage

**Focus on**:

- Public API functions
- Critical paths (Apply, Router reconciliation)
- Error handling
- Edge cases

**Don't chase 100%**:

- Generated code (deepcopy)
- Trivial getters/setters
- Panic paths (framework should handle)

## Benchmarking

### Writing Benchmarks

```go
func BenchmarkRouterDispatch(b *testing.B) {
    router := setupBenchRouter(b)
    req := testRequest()
    resp := testResponse()
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _ = router.Handle(req, resp)
    }
}

func BenchmarkApply(b *testing.B) {
    ctx := context.Background()
    client := fake.NewClientBuilder().WithScheme(scheme.Scheme).Build()
    apply := apply.New(client)
    
    owner := testOwner()
    desired := testConfigMap("test")
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _ = apply.Apply(ctx, owner, desired)
    }
}
```

### Running Benchmarks

```bash
# Run all benchmarks
go test -bench=. ./...

# Specific package
go test -bench=. ./pkg/router

# With memory allocation stats
go test -bench=. -benchmem ./...

# Compare before/after
go test -bench=. ./... > old.txt
# Make changes
go test -bench=. ./... > new.txt
benchcmp old.txt new.txt
```

## Debugging Tests

### Running Single Test

```bash
# Specific test function
go test -v -run TestRouterStart ./pkg/router

# Pattern matching
go test -v -run TestRouter.* ./pkg/router

# With timeout
go test -v -run TestRouterStart -timeout 30s ./pkg/router
```

### Debug Logging in Tests

```go
func TestSomething(t *testing.T) {
    // Debug output (only shown with -v or on failure)
    t.Logf("Debug info: %+v", someVar)
    
    result := doSomething()
    
    t.Logf("Result: %+v", result)
    
    assert.NoError(t, result.Err)
}
```

### Using Delve Debugger

```bash
# Install delve
go install github.com/go-delve/delve/cmd/dlv@latest

# Debug test
dlv test ./pkg/router -- -test.run TestRouterStart

# In delve:
(dlv) break router.go:42
(dlv) continue
(dlv) print req
(dlv) step
```

## Best Practices

### DO

✅ Write tests for all public APIs
✅ Use table-driven tests for multiple scenarios
✅ Test error cases and edge conditions
✅ Use testify for assertions
✅ Mark helper functions with `t.Helper()`
✅ Test with fake Kubernetes client
✅ Keep tests fast (< 100ms each)
✅ Clean up resources in tests
✅ Test backward compatibility

### DON'T

❌ Skip tests to "save time"
❌ Test internal implementation details
❌ Write flaky tests (timing-dependent without proper synchronization)
❌ Share state between tests
❌ Ignore test failures
❌ Write tests requiring manual setup
❌ Test external dependencies (mock them)
❌ Make breaking API changes without tests verifying backward compatibility

## Testing Checklist

Before releasing changes:

- [ ] All tests pass: `make test`
- [ ] Linting passes: `make validate`
- [ ] Generated code up to date: `make validate-ci`
- [ ] No race conditions: `go test -race ./...`
- [ ] Coverage maintained or improved
- [ ] Integration tested with obot-entraid
- [ ] Examples still compile
- [ ] Breaking changes documented
- [ ] Backward compatibility tests added

## Quick Reference

### Run Tests

```bash
make test                    # All tests
make test-race              # With race detector
make test-cover             # With coverage
go test -v ./pkg/router     # Specific package
go test -run TestFunc ./... # Specific test
```

### CI Validation

```bash
make validate-ci            # Full CI check
make validate               # Linting only
```

### Coverage

```bash
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### Benchmarks

```bash
go test -bench=. -benchmem ./...
```

### Integration with obot-entraid

```bash
cd ../obot-entraid
echo 'replace github.com/obot-platform/nah => ../nah' >> go.mod
go mod tidy
make test
```

## Future Improvements

Potential testing enhancements:

1. **Increase coverage**: Target 75%+ for critical packages
2. **Integration test suite**: Automated tests with real Kubernetes
3. **Example validation**: CI job that builds and runs examples
4. **Downstream testing**: Automated testing with obot-entraid in CI
5. **Performance benchmarks**: Track performance regressions
6. **Mutation testing**: Verify test quality with mutation testing tools
