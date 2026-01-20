# nah Framework Documentation

**Kubernetes Controller Framework** - Developer Guides and Reference
**Last Updated:** 2026-01-19

> **Quick Links:** [Getting Started](#getting-started) | [Core Guides](#core-framework-guides) | [Advanced Topics](#advanced-topics) | [Examples](#examples)

---

## Overview

nah is a Kubernetes controller framework that simplifies building production-grade controllers. This documentation provides comprehensive guides for using nah in your projects.

```
nah/docs/
├── README.md                     <- This file: Guide navigation
└── guides/                       <- Implementation guides
    ├── controllers.md            <- Event routing and handler patterns
    ├── apply.md                  <- Declarative resource management
    ├── leader-election.md        <- HA controller deployments
    ├── testing.md                <- Controller testing patterns
    └── performance.md            <- Optimization strategies
```

---

## Getting Started

### New to nah?

**Follow this path:**

1. **[nah/README.md](../README.md)** - Framework overview and quick start
2. **[controllers.md](guides/controllers.md)** - Learn the core controller patterns
3. **[apply.md](guides/apply.md)** - Understand declarative resource management
4. **Build your first controller** - Use the patterns from the guides

### Quick Reference by Task

| I want to... | Read This |
|--------------|-----------|
| **Build a basic controller** | [controllers.md](guides/controllers.md) |
| **Manage resources declaratively** | [apply.md](guides/apply.md) |
| **Deploy controllers for HA** | [leader-election.md](guides/leader-election.md) |
| **Test my controllers** | [testing.md](guides/testing.md) |
| **Optimize performance** | [performance.md](guides/performance.md) |

---

## Core Framework Guides

### [Controller Development](guides/controllers.md)

Learn how to build Kubernetes controllers using nah's event routing and handler patterns.

**Topics covered:**
- Event Router API
- Handler patterns and middleware
- GVK-specific tuning
- Backend abstraction (Kubernetes, kinm)

**Start here if:** You're building your first nah controller or migrating from controller-runtime.

### [Declarative Apply](guides/apply.md)

Master nah's Apply API for declarative resource management following kubectl apply semantics.

**Topics covered:**
- kubectl apply semantics
- Resource management patterns
- Conflict resolution strategies
- Apply best practices

**Start here if:** You need to manage Kubernetes resources from your controller.

### [Leader Election](guides/leader-election.md)

Configure high-availability controller deployments with leader election.

**Topics covered:**
- Lease-based election
- File-based election
- HA patterns and best practices
- Failure recovery

**Start here if:** You're deploying controllers in production with multiple replicas.

---

## Advanced Topics

### [Testing Strategies](guides/testing.md)

Build confidence in your controllers with comprehensive testing patterns.

**Topics covered:**
- Unit testing controllers
- Integration testing with backends
- Mock backends and test fixtures
- envtest patterns

**Start here if:** You want to test controllers thoroughly before production deployment.

### [Performance Optimization](guides/performance.md)

Optimize controller performance for production workloads.

**Topics covered:**
- Caching strategies
- Worker pool tuning
- Resource watching optimization
- OpenTelemetry tracing
- Profiling techniques

**Start here if:** Your controller needs to scale or you're experiencing performance issues.

---

## Examples

See working examples in the nah repository:

- **[examples/](../examples/)** - Complete controller implementations
- **[pkg/router/](../pkg/router/)** - Router implementation patterns
- **[pkg/apply/](../pkg/apply/)** - Apply implementation patterns

---

## Architecture

### Framework Components

```
┌─────────────────────────────────────────────────────┐
│                  Your Controller                     │
│  ┌────────────┐  ┌─────────┐  ┌─────────────────┐  │
│  │  Handlers  │→ │  Apply  │→ │ Leader Election │  │
│  └────────────┘  └─────────┘  └─────────────────┘  │
└──────────┬──────────────────────────────────────────┘
           │
           ↓
┌─────────────────────────────────────────────────────┐
│              nah Event Router                        │
│  ┌────────────────┐        ┌────────────────────┐  │
│  │  GVK Routing   │        │    Middleware      │  │
│  └────────────────┘        └────────────────────┘  │
└──────────┬──────────────────────────────────────────┘
           │
           ↓
┌─────────────────────────────────────────────────────┐
│           Backend Abstraction                        │
│  ┌─────────────┐           ┌────────────────────┐  │
│  │ Kubernetes  │           │       kinm         │  │
│  └─────────────┘           └────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

See [controllers.md](guides/controllers.md) for detailed architecture documentation.

---

## Contributing

Found an issue or want to improve the documentation?

1. Check existing issues in the nah repository
2. Follow the patterns in [CONTRIBUTING.md](../CONTRIBUTING.md)
3. Submit a pull request with your improvements

---

## Additional Resources

| Resource | Description |
|----------|-------------|
| [nah/README.md](../README.md) | Framework overview and quick start |
| [nah/CLAUDE.md](../CLAUDE.md) | AI assistant development guide |
| [API Reference](../pkg/) | GoDoc API documentation |
| [Examples](../examples/) | Working controller implementations |

---

*For workspace-wide documentation patterns and standards, see [../../documentation/docs/reference/documentation-guide.md](../../documentation/docs/reference/documentation-guide.md).*
