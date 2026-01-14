# nah Documentation

Welcome to the **nah** framework documentation! This directory contains comprehensive guides, API references, and architectural documentation.

## 📚 Documentation Index

### Getting Started
- [Quick Start](../README.md#-quick-start) - Get up and running in 5 minutes
- [Installation](../README.md#-installation) - How to install nah
- [Core Concepts](../README.md#-core-concepts) - Understanding nah fundamentals

### API Reference
- [Router Package](packages/router.md) - Event routing and handler management
- [Backend Package](packages/backend.md) - Kubernetes client abstraction
- [Apply Package](packages/apply.md) - Declarative resource management
- [Runtime Package](packages/runtime.md) - Runtime configuration (Coming Soon)
- [Leader Package](packages/leader.md) - Leader election (Coming Soon)

### Architecture
- [Overview](architecture/overview.md) - High-level architecture with diagrams
- [Design Patterns](architecture/patterns.md) - Core design patterns (Coming Soon)
- [Components](architecture/components.md) - Component deep dive (Coming Soon)
- [ADRs](architecture/adr/) - Architecture Decision Records (Coming Soon)

### Guides
- [Writing Controllers](guides/controllers.md) - Controller development guide (Coming Soon)
- [Resource Management](guides/apply.md) - Using the Apply package (Coming Soon)
- [Leader Election](guides/leader-election.md) - High availability setup (Coming Soon)
- [Testing](guides/testing.md) - Testing controllers (Coming Soon)
- [Performance Tuning](guides/performance.md) - Optimization strategies (Coming Soon)

### Examples
- [Basic Controller](examples/basic-controller.md) - Simple controller example (Coming Soon)
- [Multi-Resource](examples/multi-resource.md) - Managing multiple resources (Coming Soon)
- [Custom Middleware](examples/middleware.md) - Writing middleware (Coming Soon)
- [Finalizers](examples/finalizers.md) - Using finalizers (Coming Soon)

### Contributing
- [Contributing Guide](../CONTRIBUTING.md) - How to contribute
- [Code Style](../CONTRIBUTING.md#code-style) - Coding standards
- [Pull Request Process](../CONTRIBUTING.md#pull-request-process) - PR guidelines

### Development Assistance
- [CLAUDE.md](../CLAUDE.md) - Guidance for Claude Code and AI assistants

---

## 🔍 Quick Reference

### Create a Basic Controller

```go
import "github.com/obot-platform/nah"

router, err := nah.DefaultRouter("my-controller", scheme)
if err != nil {
    panic(err)
}

router.Type(&corev1.ConfigMap{}).
    Namespace("default").
    HandlerFunc(func(req router.Request, resp router.Response) error {
        cm := req.Object.(*corev1.ConfigMap)
        fmt.Printf("Processing: %s\n", cm.Name)
        return nil
    })

router.Start(ctx)
```

### Apply Desired State

```go
import "github.com/obot-platform/nah/pkg/apply"

a := apply.New(backend)

desired := []client.Object{
    &corev1.ConfigMap{...},
    &corev1.Service{...},
}

a.Apply(ctx, owner, desired...)
```

### Use Filters and Middleware

```go
router.Type(&corev1.Pod{}).
    Namespace("kube-system").
    Selector(labels.Set{"app": "my-app"}).
    Middleware(router.ErrorPrefix("pod-handler")).
    HandlerFunc(handler)
```

---

## 📖 Documentation Structure

```
docs/
├── README.md                    # This file
├── api/                         # API reference documentation
├── architecture/                # Architecture documentation
│   ├── overview.md             # Architecture overview with diagrams
│   ├── patterns.md             # Design patterns
│   ├── components.md           # Component details
│   └── adr/                    # Architecture Decision Records
├── guides/                      # How-to guides
│   ├── controllers.md          # Writing controllers
│   ├── apply.md                # Resource management
│   ├── leader-election.md      # HA setup
│   ├── testing.md              # Testing strategies
│   └── performance.md          # Performance tuning
├── packages/                    # Per-package documentation
│   ├── router.md               # Router package
│   ├── backend.md              # Backend package
│   ├── apply.md                # Apply package
│   ├── runtime.md              # Runtime package
│   └── leader.md               # Leader package
└── examples/                    # Code examples
    ├── basic-controller.md     # Basic examples
    ├── multi-resource.md       # Advanced examples
    ├── middleware.md           # Middleware examples
    └── finalizers.md           # Finalizer examples
```

---

## 🎯 Learning Path

### Beginner
1. Read [Quick Start](../README.md#-quick-start)
2. Understand [Core Concepts](../README.md#-core-concepts)
3. Review [Basic Controller Example](examples/basic-controller.md)
4. Study [Router Package](packages/router.md) documentation

### Intermediate
1. Learn [Apply Package](packages/apply.md) for resource management
2. Understand [Architecture Overview](architecture/overview.md)
3. Explore [Design Patterns](architecture/patterns.md)
4. Practice with [Multi-Resource Example](examples/multi-resource.md)

### Advanced
1. Study [Backend Package](packages/backend.md) internals
2. Implement [Custom Middleware](examples/middleware.md)
3. Configure [Leader Election](guides/leader-election.md)
4. Optimize with [Performance Guide](guides/performance.md)

---

## 💡 Common Use Cases

### Simple Resource Controller

**Scenario:** Watch ConfigMaps and log changes

**Documentation:**
- [Router Package](packages/router.md)
- [Basic Controller Example](examples/basic-controller.md)

### Multi-Resource Application

**Scenario:** Manage Deployment, Service, and Ingress

**Documentation:**
- [Apply Package](packages/apply.md)
- [Multi-Resource Example](examples/multi-resource.md)

### High Availability Controller

**Scenario:** Multiple controller replicas with leader election

**Documentation:**
- [Leader Election Guide](guides/leader-election.md)
- [Architecture Overview](architecture/overview.md)

### Custom Reconciliation Logic

**Scenario:** Implement complex business logic with finalizers

**Documentation:**
- [Finalizers Example](examples/finalizers.md)
- [Resource Management Guide](guides/apply.md)

---

## 🔗 External Resources

### Kubernetes
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)
- [Controller Patterns](https://kubernetes.io/docs/concepts/architecture/controller/)

### controller-runtime
- [controller-runtime Documentation](https://pkg.go.dev/sigs.k8s.io/controller-runtime)
- [controller-runtime Book](https://book.kubebuilder.io/reference/controller-runtime.html)

### Go
- [Effective Go](https://golang.org/doc/effective_go.html)
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)

---

## 🤝 Contributing to Documentation

Found an error? Want to improve the docs?

1. Check [Contributing Guide](../CONTRIBUTING.md)
2. Edit the relevant markdown file
3. Submit a Pull Request

We welcome documentation improvements!

---

## 📝 Documentation Standards

### Writing Style
- Clear and concise language
- Active voice preferred
- Code examples for all concepts
- Links to related documentation

### Code Examples
- Complete, runnable examples
- Proper error handling
- Comments explaining key concepts
- Follow project code style

### Diagrams
- Use Mermaid for diagrams
- Keep diagrams simple and focused
- Provide text explanations

---

## ❓ Getting Help

- **Documentation Issues**: [Open an issue](https://github.com/obot-platform/nah/issues/new)
- **Questions**: [GitHub Discussions](https://github.com/obot-platform/nah/discussions)
- **API Reference**: [pkg.go.dev](https://pkg.go.dev/github.com/obot-platform/nah)

---

## 📦 Related Projects

- [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) - The foundation for nah
- [kubebuilder](https://github.com/kubernetes-sigs/kubebuilder) - Kubernetes API extension framework
- [operator-sdk](https://github.com/operator-framework/operator-sdk) - Operator development toolkit

---

**Documentation Version:** Matches nah library version
**Last Updated:** 2026-01-14

---

**Made with ❤️ by the Obot Platform Team**
