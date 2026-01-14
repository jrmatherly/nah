# Project Index: nah

**Generated:** 2026-01-14
**Type:** Go Library / Kubernetes Controller Framework
**License:** Apache 2.0

> **Documentation Version:** 1.0.0 | **Last Updated:** 2026-01-14 | **Compatible with:** nah v0.x.x+

---

## 📋 Quick Summary

`nah` is a Kubernetes controller framework library by Obot Platform that provides utilities and abstractions for building custom Kubernetes controllers. It offers routing, leader election, resource management, and declarative apply/reconciliation logic.

**Repository:** github.com/obot-platform/nah
**Go Version:** 1.24.0
**Total Files:** 68 Go source files across 24 packages

---

## 📁 Project Structure

```
nah/
├── cmd/
│   └── deepcopy/          # Code generation tool for deep copy methods
├── pkg/
│   ├── router/            # Core controller routing (15 files)
│   ├── runtime/           # Controller runtime configuration (10 files)
│   ├── apply/             # Declarative resource management (11 files)
│   ├── backend/           # Kubernetes client abstraction (1 file)
│   ├── watcher/           # Resource watching and triggers (1 file)
│   ├── leader/            # Leader election (2 files)
│   ├── webhook/           # Admission webhooks (2 files)
│   ├── handlers/          # Request handlers (1 file)
│   ├── conditions/        # Status conditions (1 file)
│   ├── typed/             # Type-safe wrappers (4 files)
│   ├── deepcopy/          # Deep copy utilities (1 file)
│   ├── yaml/              # YAML processing (1 file)
│   ├── data/              # Data manipulation (2 files)
│   ├── restconfig/        # REST configuration (3 files)
│   ├── mapper/            # Resource mapping (1 file)
│   ├── urlbuilder/        # URL construction (1 file)
│   ├── randomtoken/       # Token generation (1 file)
│   ├── ratelimit/         # Rate limiting (1 file)
│   ├── logrus/            # Logrus configuration (1 file)
│   ├── log/               # Logging utilities (1 file)
│   ├── version/           # Version info (1 file)
│   ├── fields/            # Field selectors (1 file)
│   ├── untriggered/       # Skip reconciliation (1 file)
│   ├── name/              # Naming conventions (1 file)
│   └── merr/              # Multi-error handling (1 file)
└── router.go              # Main package entry point
```

---

## 🚀 Entry Points

### Primary API Entry Point
- **File:** `router.go:1`
- **Purpose:** Main package interface for creating and configuring routers
- **Key Functions:**
  - `DefaultRouter(routerName, scheme)` - Create router with default configuration
  - `DefaultOptions(routerName, scheme)` - Generate standard options
  - `NewRouter(handlerName, opts)` - Create router with custom options

### Code Generation Tool
- **File:** `cmd/deepcopy/main.go`
- **Purpose:** Generate deep copy methods for Kubernetes types

---

## 📦 Core Modules

### Router (`pkg/router/`)
**Purpose:** Event routing and handler management for Kubernetes resources

**Key Exports:**
- `Router` - Main controller router
- `RouteBuilder` - Fluent API for route configuration
- `HandlerSet` - Collection of handlers
- Middleware: `ErrorPrefix`, `IgnoreNilHandler`, `IgnoreRemoveHandler`
- Filters: `NameNamespaceFilter`, `SelectorFilter`, `FieldSelectorFilter`

**Sub-packages:**
- `tester/` - Testing utilities for router handlers

**Files:** 15 (handlers.go, router.go, trigger.go, save.go, response_wrapper.go, matcher.go, healthz.go, handler.go, getorcreate.go, finalizer.go, client.go, types.go)

### Backend (`pkg/backend/`)
**Purpose:** Abstraction layer over Kubernetes client operations

**Key Interfaces:**
- `Backend` - Main Kubernetes operations interface
- `Watcher` - Resource watching
- `Trigger` - Event triggering
- `CacheFactory` - Cache management

**Provides:** Unified API for Kubernetes operations with caching

### Apply (`pkg/apply/`)
**Purpose:** Declarative resource management and reconciliation

**Key Exports:**
- `Apply` interface - Apply desired state operations
- `DesiredSet` - Desired vs actual state management
- `New()` - Create apply instance
- `Ensure()` - Ensure resource exists

**Pattern:** Similar to `kubectl apply` with owner reference tracking

**Sub-packages:**
- `objectset/` - Object set management utilities

**Files:** 11 (apply.go, desiredset*.go, patch_style.go, reconcilers.go)

### Runtime (`pkg/runtime/`)
**Purpose:** Controller runtime configuration and lifecycle

**Key Exports:**
- `Config` - Runtime configuration
- `NewRuntimeWithConfig()` - Create configured runtime
- `NewRuntimeForNamespace()` - Namespace-scoped runtime
- GVK-specific threadiness and queue splitting support

**Files:** 10 (backend.go, cached.go, clients.go, controller.go, copy.go, errorcontroller.go, sharedcontroller.go, sharedcontrollerfactory.go, sharedhandler.go, transaction.go)

### Leader Election (`pkg/leader/`)
**Purpose:** High availability through leader election

**Key Exports:**
- `ElectionConfig` - Leader election configuration
- `NewDefaultElectionConfig()` - Default lease-based election (15s TTL)

**Strategies:**
- Lease-based (default) - Uses Kubernetes lease objects
- File-based - Uses file locks (`locks/file.go`)

**Files:** leader.go, locks/file.go

### Watcher (`pkg/watcher/`)
**Purpose:** Monitor resources and trigger handlers

**Features:**
- Watch resource changes (create, update, delete)
- Optimized trigger management with `sync.Map`
- GVK-based threadiness control
- Multiple worker queues per GVK
- On-demand worker goroutine spawning

---

## 🧩 Supporting Packages

### Data & Conversion
- **typed/** - Type-safe wrappers (chan.go, funcs.go, map.go, slice.go)
- **data/** - Data manipulation utilities (generic.go, map.go)
- **yaml/** - YAML processing
- **deepcopy/** - Deep copy generation and utilities

### Kubernetes Utilities
- **restconfig/** - REST config helpers (restconfig.go, scheme.go, wait.go)
- **mapper/** - Resource mapping
- **urlbuilder/** - URL construction for K8s API
- **conditions/** - Status condition management
- **fields/** - Field selector utilities
- **name/** - Resource naming conventions

### Infrastructure
- **webhook/** - Admission webhook support (router.go, match.go)
- **handlers/** - Common request handlers
- **log/** - Logging utilities
- **logrus/** - Logrus configuration
- **version/** - Version information
- **merr/** - Multi-error handling
- **randomtoken/** - Secure token generation
- **ratelimit/** - Rate limiting (none.go)
- **untriggered/** - Skip reconciliation triggers

---

## 🔧 Configuration Files

- **go.mod** - Go module definition with dependencies
- **.golangci.yml** - Linter configuration
- **.github/workflows/test.yaml** - CI/CD pipeline
- **.serena/project.yml** - Serena project configuration
- **Makefile** - Build targets (setup-ci-env, validate-ci, validate, test)

---

## 🧪 Test Coverage

- **Total Go Files:** 68
- **Test Files:** 0 (no `*_test.go` files found)
- **Test Framework:** github.com/stretchr/testify, github.com/hexops/autogold/v2
- **Test Utilities:** `pkg/router/tester/` - Router handler testing framework
- **CI:** GitHub Actions with golangci-lint and `go test ./...`

---

## 🔗 Key Dependencies

### Kubernetes Ecosystem
- **k8s.io/client-go** v0.31.1 - Kubernetes client library
- **k8s.io/apimachinery** v0.31.1 - Kubernetes API machinery
- **k8s.io/api** v0.31.1 - Kubernetes API types
- **sigs.k8s.io/controller-runtime** v0.19.0 - Controller runtime
- **sigs.k8s.io/controller-tools** v0.16.4 - Code generation
- **k8s.io/utils** v0.0.0-20240921022957 - Kubernetes utilities

### Observability
- **go.opentelemetry.io/otel** v1.35.0 - OpenTelemetry tracing
- **go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp** v0.60.0 - HTTP instrumentation

### Logging
- **github.com/sirupsen/logrus** v1.9.3 - Structured logging
- **github.com/bombsimon/logrusr/v4** v4.1.0 - Logrus to logr adapter

### Utilities
- **github.com/google/uuid** v1.6.0 - UUID generation
- **gomodules.xyz/jsonpatch/v2** v2.4.0 - JSON patch operations
- **golang.org/x/time** v0.7.0 - Time utilities (rate limiting)
- **github.com/moby/locker** v1.0.1 - Lock management

---

## 📚 Documentation

### Code Documentation
- Inline godoc comments throughout packages
- Memory files in `.serena/memories/`:
  - `project_overview.md` - High-level project description
  - `codebase_architecture.md` - Detailed architecture
  - `design_patterns_and_guidelines.md` - Design patterns
  - `code_style_and_conventions.md` - Coding standards
  - `task_completion_workflow.md` - Workflow documentation
  - `suggested_commands.md` - Common commands

### External Documentation
- Apache License 2.0 in `LICENSE` file

### Development Assistance
- **CLAUDE.md** - Guidance for Claude Code AI assistant
  - Development commands reference
  - Architecture overview for AI context
  - Code patterns and best practices
  - Quick reference for common tasks

---

## 📝 Quick Start

### Installation
```bash
go get github.com/obot-platform/nah
```

### Basic Usage
```go
import "github.com/obot-platform/nah"

// Create router with defaults
router, err := nah.DefaultRouter("my-controller", scheme)

// Configure routes
router.Type(&MyResource{}).
    Namespace("default").
    HandlerFunc(func(req router.Request, resp router.Response) error {
        // Handle resource
        return nil
    })

// Start controller
router.Start(ctx)
```

### Build Commands
```bash
make setup-ci-env   # Install golangci-lint
make validate       # Run linter
make test           # Run tests
make validate-ci    # Check for dirty repo
```

---

## 🏗️ Architecture Patterns

### Builder Pattern
- `RouteBuilder` for fluent route configuration
- Options structs with `complete()` methods

### Factory Pattern
- `New()` constructors with explicit dependencies
- `Default*()` functions for standard configurations

### Interface Segregation
- Small, focused interfaces (Backend, Apply, Trigger, Watcher)
- Easy mocking and testing

### Middleware/Filter Pattern
- Handler middleware for cross-cutting concerns
- Filters for resource selection

### Reconciliation Loop
- Standard Kubernetes controller pattern
- Watch → Queue → Handler → Apply cycle

---

## 🎯 Key Design Decisions

1. **Declarative State Management** - Apply package mirrors `kubectl apply` semantics
2. **Leader Election by Default** - 15-second TTL lease-based election for HA
3. **Flexible Backend Abstraction** - Supports both cached and uncached operations
4. **GVK-Specific Configuration** - Per-resource threadiness and queue splitting
5. **OpenTelemetry Integration** - Built-in distributed tracing support
6. **On-Demand Workers** - Spawn goroutines only when needed for efficiency
7. **Trigger Optimization** - `sync.Map` for low-contention trigger management

---

## 🔄 Typical Usage Flow

```
1. Initialize:
   Scheme → RESTConfig → Runtime → Backend → Router

2. Register Routes:
   Router.Type(MyResource) → Filters → Handler

3. Start:
   Router.Start() → Leader Election → Handlers → Watchers

4. Event Processing:
   Watcher → Trigger → Router → Middleware → Handler → Apply

5. Reconciliation:
   Handler → DesiredSet → Apply → Compare → Patch/Create/Delete
```

---

## 📊 Package Statistics

| Package | Files | Primary Purpose |
| --------- | ------- | ---------------- |
| router | 15 | Event routing and handler management |
| runtime | 10 | Runtime configuration and lifecycle |
| apply | 11 | Declarative state reconciliation |
| typed | 4 | Type-safe wrappers |
| restconfig | 3 | REST configuration helpers |
| leader | 2 | Leader election |
| webhook | 2 | Admission webhooks |
| data | 2 | Data manipulation |
| *others* | 1 each | Specialized utilities |

**Total:** 24 packages, 68 source files

---

## 🔍 Search Keywords

Kubernetes, controller, operator, reconciliation, apply, leader election, router, watcher, backend, cache, webhook, OpenTelemetry, tracing, handler, middleware, filter, GVK, CRD, custom resource

---

*This index was generated automatically to provide fast context loading (3KB vs 58KB full codebase read). Token savings: ~94% per session.*
