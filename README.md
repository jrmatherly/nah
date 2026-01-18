# nah

[![Go Version](https://img.shields.io/badge/Go-1.24.0-blue.svg)](https://golang.org/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![CI](https://github.com/obot-platform/nah/workflows/test/badge.svg)](https://github.com/obot-platform/nah/actions)

> **Documentation Version:** 1.0.0 | **Last Updated:** 2026-01-14 | **Compatible with:** nah v0.x.x+

**nah** is a Kubernetes controller framework library that provides utilities and abstractions for building custom Kubernetes controllers. It offers a clean, composable API for routing events, managing resources, leader election, and declarative reconciliation.

> **Note:** "nah" is short for "Not Another Handler" - a lightweight, opinionated framework for Kubernetes controller development.

## ✨ Features

- 🎯 **Event Router** - Fluent API for routing Kubernetes events to handlers with middleware support
- 🔄 **Declarative Apply** - `kubectl apply`-like semantics for resource management
- 👑 **Leader Election** - Built-in support for high availability with lease and file-based strategies
- 👀 **Resource Watching** - Efficient resource watching with optimized trigger management
- 🏗️ **Backend Abstraction** - Clean interface over Kubernetes client operations with caching
- 📊 **OpenTelemetry** - Distributed tracing support out of the box
- ⚙️ **GVK-Specific Tuning** - Per-resource threadiness and worker queue configuration
- 🪝 **Webhook Support** - Admission webhook integration

## 📦 Installation

```bash
go get github.com/obot-platform/nah
```

## 🚀 Quick Start

### Basic Controller

```go
package main

import (
    "context"
    "fmt"

    "github.com/obot-platform/nah"
    "github.com/obot-platform/nah/pkg/router"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
)

func main() {
    // Create scheme
    scheme := runtime.NewScheme()
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))

    // Create router with defaults (includes leader election)
    r, err := nah.DefaultRouter("my-controller", scheme)
    if err != nil {
        panic(err)
    }

    // Register handler for ConfigMaps
    r.Type(&corev1.ConfigMap{}).
        Namespace("default").
        HandlerFunc(func(req router.Request, resp router.Response) error {
            cm := req.Object.(*corev1.ConfigMap)
            fmt.Printf("Processing ConfigMap: %s/%s\n", cm.Namespace, cm.Name)
            return nil
        })

    // Start the controller
    ctx := context.Background()
    if err := r.Start(ctx); err != nil {
        panic(err)
    }
}
```

### Custom Configuration

```go
// Create custom options
opts := &nah.Options{
    Scheme:      scheme,
    Namespace:   "default",
    HealthzPort: 9090,
    // Optional: Disable leader election
    ElectionConfig: nil,
    // Optional: Per-resource configuration
    GVKThreadiness: map[schema.GroupVersionKind]int{
        corev1.SchemeGroupVersion.WithKind("ConfigMap"): 10,
    },
}

r, err := nah.NewRouter("my-controller", opts)
```

### Using Filters and Middleware

```go
r.Type(&corev1.Pod{}).
    Namespace("kube-system").
    Selector(labels.Set{"app": "my-app"}).
    Middleware(router.ErrorPrefix("pod-handler")).
    HandlerFunc(func(req router.Request, resp router.Response) error {
        pod := req.Object.(*corev1.Pod)
        // Handle pod...
        return nil
    })
```

### Declarative Resource Management

```go
import (
    "github.com/obot-platform/nah/pkg/apply"
)

func (h *MyHandler) Handle(req router.Request, resp router.Response) error {
    // Get current state
    obj := req.Object

    // Create apply instance
    a := apply.New(resp.Backend(), h.routerName)

    // Define desired state
    desiredResources := []runtime.Object{
        &corev1.ConfigMap{
            ObjectMeta: metav1.ObjectMeta{
                Name:      "my-config",
                Namespace: obj.Namespace,
            },
            Data: map[string]string{
                "key": "value",
            },
        },
    }

    // Apply desired state
    return a.Apply(obj, desiredResources)
}
```

## 📚 Core Concepts

### Router

The `Router` is the central component that:

- Manages the controller lifecycle (start, stop)
- Routes Kubernetes events to registered handlers
- Integrates with leader election
- Provides health check endpoints

```go
// Create router
r, err := nah.DefaultRouter("my-controller", scheme)

// Register handlers
r.Type(&MyResource{}).HandlerFunc(myHandler)

// Start controller
r.Start(ctx)
```

### Handlers

Handlers process Kubernetes resources:

```go
type Handler interface {
    Handle(req Request, resp Response) error
}

// Or use HandlerFunc
func myHandler(req router.Request, resp router.Response) error {
    obj := req.Object
    // Process object...
    return nil
}
```

### Backend

The `Backend` provides a unified interface to Kubernetes operations:

```go
type Backend interface {
    Get(ctx context.Context, key client.ObjectKey, obj client.Object) error
    List(ctx context.Context, list client.ObjectList, opts ...client.ListOption) error
    Create(ctx context.Context, obj client.Object, opts ...client.CreateOption) error
    Update(ctx context.Context, obj client.Object, opts ...client.UpdateOption) error
    Delete(ctx context.Context, obj client.Object, opts ...client.DeleteOption) error
    // ... more methods
}
```

### Apply

The `Apply` interface provides declarative resource management:

```go
type Apply interface {
    Apply(owner runtime.Object, objects []runtime.Object) error
    Ensure(owner runtime.Object, objects []runtime.Object) error
}
```

### Leader Election

Built-in leader election ensures only one controller instance is active:

```go
// Lease-based (default)
config := leader.NewDefaultElectionConfig("", "my-controller", restConfig)

// File-based
config := &leader.ElectionConfig{
    LockType: leader.LockTypeFile,
    LockName: "/tmp/my-controller.lock",
}

opts := &nah.Options{
    ElectionConfig: config,
    // ...
}
```

## 📖 Documentation

### Getting Started

- [Quick Start](#-quick-start) - Get up and running in minutes
- [Examples](docs/examples/) - Complete working examples
- [API Reference](docs/api/) - Detailed API documentation

### Architecture

- [Design Patterns](docs/architecture/patterns.md) - Core design patterns used
- [Component Overview](docs/architecture/components.md) - Architecture deep dive
- [Architecture Decisions](docs/architecture/adr/) - ADRs and design rationale

### Guides

- [Writing Controllers](docs/guides/controllers.md) - Controller development guide
- [Resource Management](docs/guides/apply.md) - Using the Apply package
- [Leader Election](docs/guides/leader-election.md) - HA controller setup
- [Testing](docs/guides/testing.md) - Testing controllers with nah
- [Performance Tuning](docs/guides/performance.md) - Optimization strategies

### Package Documentation

- [router](docs/packages/router.md) - Event routing and handlers
- [backend](docs/packages/backend.md) - Kubernetes client abstraction
- [apply](docs/packages/apply.md) - Declarative resource management
- [runtime](docs/packages/runtime.md) - Runtime configuration
- [leader](docs/packages/leader.md) - Leader election
- [watcher](docs/packages/watcher.md) - Resource watching

### For AI Assistants

- [CLAUDE.md](CLAUDE.md) - Guidance for Claude Code when working in this repository

## 🏗️ Project Structure

```
nah/
├── cmd/deepcopy/       # Code generation tool
├── pkg/
│   ├── router/         # Event routing (15 files)
│   ├── runtime/        # Runtime configuration (10 files)
│   ├── apply/          # Declarative reconciliation (11 files)
│   ├── backend/        # Kubernetes client abstraction
│   ├── watcher/        # Resource watching
│   ├── leader/         # Leader election
│   ├── webhook/        # Admission webhooks
│   └── ...             # Utility packages
├── docs/               # Comprehensive documentation
├── PROJECT_INDEX.md    # Quick reference guide
└── router.go           # Main package API
```

See [PROJECT_INDEX.md](PROJECT_INDEX.md) for a detailed overview.

## 🛠️ Development

### Prerequisites

- Go 1.25.0 or later
- Kubernetes cluster (for testing)
- golangci-lint (for linting)

### Building

```bash
# Install dependencies
go mod download

# Build
go build ./...

# Run code generation
go generate ./...
```

### Testing

```bash
# Run all tests
make test

# Run linter
make validate

# CI validation (checks for uncommitted generated code)
make validate-ci
```

### Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for details on:

- Code style and conventions
- Development workflow
- Pull request process
- Testing requirements

## 🔍 Key Dependencies

- [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) v0.19.0 - Controller framework
- [client-go](https://github.com/kubernetes/client-go) v0.31.1 - Kubernetes client
- [OpenTelemetry](https://opentelemetry.io/) v1.35.0 - Distributed tracing
- [logrus](https://github.com/sirupsen/logrus) v1.9.3 - Structured logging

See [go.mod](go.mod) for complete dependency list.

## 📊 Performance

nah is designed for efficiency:

- **On-demand workers** - Goroutines spawn only when needed
- **sync.Map optimization** - Low-contention trigger management
- **GVK-specific tuning** - Per-resource threadiness and queue splitting
- **Configurable caching** - Fine-grained cache control with `ByObject`

See [Performance Tuning Guide](docs/guides/performance.md) for optimization strategies.

## 🗺️ Roadmap

- [ ] Enhanced testing framework with more test utilities
- [ ] Additional examples and templates
- [ ] Metrics and monitoring integration
- [ ] Advanced caching strategies
- [ ] Multi-cluster support patterns

## 📄 License

Apache License 2.0 - See [LICENSE](LICENSE) for details.

## 🙋 Support

- **Documentation**: [docs/](docs/)
- **Issues**: [GitHub Issues](https://github.com/obot-platform/nah/issues)
- **Discussions**: [GitHub Discussions](https://github.com/obot-platform/nah/discussions)

## 🙏 Acknowledgments

Built on top of the excellent [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) framework by the Kubernetes community.

---

**Made with ❤️ by the Obot Platform Team**
