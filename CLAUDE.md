# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Documentation Version:** 1.0.0 | **Last Updated:** 2026-01-14 | **Compatible with:** nah v0.x.x+

## Project Overview

**nah** is a Kubernetes controller framework library providing utilities and abstractions for building custom Kubernetes controllers. It offers event routing, declarative resource management, leader election, and backend abstractions over Kubernetes operations.

**Key Philosophy:** "Not Another Handler" - a lightweight, opinionated framework emphasizing declarative reconciliation patterns similar to `kubectl apply`.

## Development Commands

### Build and Test

```bash
# Run all tests
make test
go test ./...

# Run tests for specific package
go test ./pkg/router
go test -v ./pkg/apply  # verbose output

# Run with race detector
go test -race ./...

# Run with coverage
go test -cover ./...
```

### Code Generation

```bash
# Generate deep copy methods and controller code
go generate

# Regenerate for specific package
go generate ./pkg/router
```

### Linting and Validation

```bash
# Run golangci-lint with project configuration
make validate

# Auto-fix issues where possible
golangci-lint run --fix

# Install linter (CI environment)
make setup-ci-env
```

### Pre-commit Validation

```bash
# Simulates CI pipeline: generate, tidy, check for dirty repo
make validate-ci

# This will fail if generated code is out of sync
# Always run before committing code changes
```

### Dependency Management

```bash
# Tidy and verify dependencies
go mod tidy

# Update specific dependency
go get -u github.com/example/package
go mod tidy
```

## Architecture Overview

### Core Components

**Router** (`pkg/router/`, `router.go`)

- Central orchestrator managing controller lifecycle
- Routes Kubernetes events to appropriate handlers
- Provides fluent `RouteBuilder` API for configuration
- Integrates leader election and health checks
- Embeds `HandlerSet` for handler registration

**Backend** (`pkg/backend/`)

- Abstraction layer over Kubernetes client operations
- Combines `Trigger`, `Watcher`, `CacheFactory` interfaces
- Provides both cached (read) and uncached (write) operations
- Manages informers and event triggering

**Apply** (`pkg/apply/`)

- Declarative resource management with `kubectl apply` semantics
- Uses `DesiredSet` for comparing desired vs current state
- Handles owner references automatically
- Supports selective pruning and subcontexts for multi-controller scenarios
- Three-way merge patches for updates

**Runtime** (`pkg/runtime/`)

- Controller runtime configuration and backend factory
- Configures caching, namespaces, selectors
- Supports GVK-specific threadiness and worker queue splitting

**Leader Election** (`pkg/leader/`)

- Ensures only one active controller instance
- Supports lease-based (default, 15s TTL) and file-based strategies
- Integrates with router startup/shutdown

**Watcher** (`pkg/watcher/`)

- Monitors resource changes via Kubernetes informers
- Uses `sync.Map` for low-contention trigger management
- Supports on-demand worker goroutine spawning

### Design Patterns

**Builder Pattern**

- `RouteBuilder` provides fluent API for route configuration
- Options structs with `complete()` methods for validation

**Factory Pattern**

- `New()` constructors with explicit dependencies
- `Default*()` functions for standard configurations

**Middleware/Filter Pattern**

- Handlers wrapped with middleware for cross-cutting concerns
- Filters (namespace, selector, field selector) for resource selection

**Reconciliation Loop**

- Watch → Queue → Handler → Apply cycle
- Level-triggered (not edge-triggered)
- Idempotent reconciliation

### Key Data Flow

1. **Initialization**: `Scheme → RESTConfig → Runtime → Backend → Router`
2. **Route Registration**: `Router.Type(Resource) → Filters → Middleware → Handler`
3. **Startup**: `Router.Start() → Leader Election → Start Handlers → Start Watchers`
4. **Event Processing**: `Watcher → Trigger → Router → Middleware → Handler → Apply/Backend`
5. **Reconciliation**: `Handler → DesiredSet → Apply → Compare → Patch/Create/Delete`

### Important Interfaces

**Handler** (`pkg/router/`)

```go
type Handler interface {
    Handle(req Request, resp Response) error
}
```

Request contains the Kubernetes object being processed. Response provides access to Backend and Apply (via Objects()).

**Apply** (`pkg/apply/`)

```go
type Apply interface {
    Apply(ctx, owner, objs) error  // With owner references and pruning
    Ensure(ctx, objs) error         // Without owner references or pruning
    WithOwnerSubContext(string) Apply
    WithNoPrune() Apply
}
```

**Backend** (`pkg/backend/`)
Extends `client.WithWatch`, `client.FieldIndexer`, plus:

```go
Trigger(ctx, gvk, key, delay) error
Watcher(ctx, gvk, name, callback) error
GetInformerForKind(ctx, gvk) (Informer, error)
```

### Code Organization

- **`router.go`** - Main package API, top-level router creation functions
- **`pkg/router/`** - Router implementation, handlers, middleware, filters
- **`pkg/apply/`** - Declarative apply logic, DesiredSet, ObjectSet
- **`pkg/backend/`** - Interface definitions (implementations in pkg/runtime/)
- **`pkg/runtime/`** - Backend implementations, controller runtime, caching
- **`pkg/leader/`** - Leader election implementations
- **`pkg/watcher/`** - Resource watching and trigger management
- **`pkg/*/`** - Utility packages (typed, data, yaml, deepcopy, etc.)
- **`cmd/deepcopy/`** - Code generation tool for deep copy methods

### Configuration Patterns

**Options Pattern with Validation**

```go
type Options struct {
    Backend        backend.Backend
    Scheme         *runtime.Scheme
    ElectionConfig *leader.ElectionConfig
    GVKThreadiness map[schema.GroupVersionKind]int
}

func (o *Options) complete() (*Options, error) {
    // Validate required fields, set defaults
}
```

**GVK-Specific Configuration**
Use `GVKThreadiness` and `GVKQueueSplitters` in Options for per-resource tuning.

**Subcontexts**
Use `Apply.WithOwnerSubContext()` when multiple controllers or concerns manage different aspects of the same resources.

## Code Style

### Follow Standard Go Conventions

- Use `gofmt` for formatting (automatic)
- Exported types/functions: PascalCase
- Unexported types/functions: camelCase
- Package names: lowercase, single-word

### Error Handling

- Always wrap errors with context: `fmt.Errorf("failed to X: %w", err)`
- Never panic in library code (return errors)
- Use error prefix middleware for handler errors

### Documentation

- Document all exported APIs with godoc comments
- Start comments with the name of the item: `// Router manages...`

### Imports Organization

1. Standard library
2. External packages
3. Internal packages (github.com/obot-platform/nah/...)

### Testing

- Place tests in `*_test.go` files
- Use table-driven tests
- Use `testify` for assertions
- Use `autogold` for snapshot testing

## Important Implementation Details

### Owner References

Apply automatically manages owner references with annotations:

- `objectset.obot.io/owner-gvk`
- `objectset.obot.io/owner-name`
- `objectset.obot.io/owner-namespace`
- `objectset.obot.io/id` (subcontext)

### Pruning

- Enabled by default in `Apply.Apply()`
- Disabled in `Apply.Ensure()`
- Can be scoped with `WithPruneGVKs()` or `WithPruneTypes()`

### Concurrency

- `sync.Map` used for triggers (low contention, read-heavy)
- Mutexes for critical sections (always defer unlock)
- Workers spawn on-demand per GVK

### Leader Election

- Nil `ElectionConfig` disables leader election
- Lease-based default with 15s TTL
- File-based available via `LockTypeFile`

### Generated Code

- Files matching `*generated*.go` excluded from linting
- Commit generated code to repository
- Run `go generate` before committing changes

## CI Pipeline

GitHub Actions runs (`.github/workflows/test.yaml`):

1. `make setup-ci-env` - Install golangci-lint v1.64.6
2. `make validate-ci` - Generate, tidy, check for uncommitted changes
3. `make validate` - Run linters
4. `make test` - Run all tests

**Always run `make validate-ci` before committing** to catch issues the CI will catch.

## Documentation

- **README.md** - Project overview, quick start, features
- **CONTRIBUTING.md** - Contribution guidelines, code style, PR process
- **PROJECT_INDEX.md** - Quick reference, package overview (94% token savings)
- **docs/packages/** - Detailed API documentation for router, backend, apply
- **docs/architecture/** - Architecture overview with Mermaid diagrams

## Common Patterns

### Basic Controller

```go
router, _ := nah.DefaultRouter("my-controller", scheme)
router.Type(&Resource{}).
    Namespace("default").
    HandlerFunc(func(req router.Request, resp router.Response) error {
        // Get current state from req.Object
        // Compute desired state
        // Apply using resp.Backend() or Apply
        return nil
    })
router.Start(ctx)
```

### Using Apply for Declarative Management

```go
a := apply.New(client)
desired := []client.Object{configMap, deployment, service}
a.Apply(ctx, owner, desired...)  // Creates/updates/prunes
```

### Middleware and Filters

```go
router.Type(&Pod{}).
    Namespace("kube-system").
    Selector(labels.Set{"app": "myapp"}).
    Middleware(router.ErrorPrefix("handler")).
    HandlerFunc(handler)
```

### Finalizers

```go
router.Type(&Resource{}).
    FinalizeFunc("my-finalizer", func(req router.Request, resp router.Response) error {
        // Cleanup logic - finalizer removed automatically on success
        return nil
    })
```

## Quick Reference for Common Tasks

**Run full validation before commit:**

```bash
make validate-ci && make validate && make test
```

**Fix most linting issues automatically:**

```bash
golangci-lint run --fix
```

**Test specific functionality:**

```bash
go test -v -run TestRouterStart ./pkg/router
```

**Check generated code is up to date:**

```bash
go generate && go mod tidy && git diff --exit-code
```

## Workspace Integration

This project is part of the AI workspace. Additional resources:

- **Claude Code commands**: `AI/.claude/commands/` (expert-mode, etc.)
- **Shared agents**: `AI/.claude/agents/` (explore, security-audit, etc.)
- **SuperClaude skills**: `/sc:analyze`, `/sc:test`, `/sc:git`, etc.
- **Serena memories**: `AI/.serena/memories/` (task_completion_checklist, etc.)
- **GitHub Actions**: Workspace-level PR review and issue triage

For session initialization with full context, run `/expert-mode` from the workspace root.
