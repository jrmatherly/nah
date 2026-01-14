# Contributing to nah

> **Documentation Version:** 1.0.0 | **Last Updated:** 2026-01-14 | **Compatible with:** nah v0.x.x+

Thank you for your interest in contributing to **nah**! This document provides guidelines and instructions for contributing to the project.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Setup](#development-setup)
- [Making Changes](#making-changes)
- [Code Style](#code-style)
- [Testing](#testing)
- [Commit Messages](#commit-messages)
- [Pull Request Process](#pull-request-process)
- [Documentation](#documentation)

---

## Code of Conduct

This project adheres to a Code of Conduct that all contributors are expected to follow. Please be respectful and professional in all interactions.

---

## Getting Started

### Prerequisites

- **Go 1.24.0 or later**
- **git**
- **kubectl** (for testing with real clusters)
- **golangci-lint v1.64.6** (for linting)

### Fork and Clone

1. Fork the repository on GitHub
2. Clone your fork:
   ```bash
   git clone https://github.com/YOUR_USERNAME/nah.git
   cd nah
   ```
3. Add upstream remote:
   ```bash
   git remote add upstream https://github.com/obot-platform/nah.git
   ```

---

## Development Setup

### Install Dependencies

```bash
# Download Go dependencies
go mod download

# Install golangci-lint
make setup-ci-env
```

### Build

```bash
# Build all packages
go build ./...

# Run code generation
go generate ./...
```

### Run Tests

```bash
# Run all tests
make test

# Run specific package tests
go test ./pkg/router

# Run with verbose output
go test -v ./...
```

### Run Linter

```bash
# Run golangci-lint
make validate

# Fix auto-fixable issues
golangci-lint run --fix
```

---

## Making Changes

### Create a Branch

Create a feature branch from `main`:

```bash
git checkout main
git pull upstream main
git checkout -b feature/your-feature-name
```

Branch naming conventions:
- `feature/` - New features
- `fix/` - Bug fixes
- `docs/` - Documentation changes
- `refactor/` - Code refactoring
- `test/` - Test additions or improvements

### Make Your Changes

1. **Write code** following the [Code Style](#code-style) guidelines
2. **Add tests** for new functionality
3. **Update documentation** for API changes
4. **Run tests and linter** before committing:
   ```bash
   make test
   make validate
   ```

---

## Code Style

### General Guidelines

Follow standard Go conventions and idioms:
- Use `gofmt` for formatting (automatically applied)
- Follow [Effective Go](https://golang.org/doc/effective_go.html)
- Use meaningful variable and function names
- Keep functions focused and concise

### Specific Conventions

#### Package Names

- Lowercase, single-word names preferred
- No underscores or mixed caps
- Examples: `router`, `backend`, `apply`

#### Type Names

- PascalCase for exported types: `Router`, `Handler`, `Apply`
- camelCase for unexported types: `errorPrefix`, `handlerSet`

#### Function Names

- PascalCase for exported: `New`, `DefaultRouter`, `Start`
- camelCase for unexported: `complete`, `startHandlers`

#### Imports

Group imports in three sections:
1. Standard library
2. External packages
3. Internal packages

```go
import (
    "context"
    "fmt"

    "k8s.io/apimachinery/pkg/runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    "github.com/obot-platform/nah/pkg/backend"
    "github.com/obot-platform/nah/pkg/router"
)
```

#### Documentation

- Document all exported types, functions, and methods
- Start doc comments with the name of the item:
  ```go
  // Router manages the controller lifecycle and routes events.
  type Router struct { ... }

  // Start begins watching resources and processing events.
  func (r *Router) Start(ctx context.Context) error { ... }
  ```

#### Error Handling

- Always handle errors explicitly
- Wrap errors with context:
  ```go
  if err != nil {
      return fmt.Errorf("failed to start router: %w", err)
  }
  ```
- Don't panic in library code (return errors instead)

#### Configuration Patterns

Use Options structs with `complete()` methods:

```go
type Options struct {
    Backend backend.Backend
    Scheme  *runtime.Scheme
    // ...
}

func (o *Options) complete() (*Options, error) {
    // Validate and set defaults
    if o.Scheme == nil {
        return nil, fmt.Errorf("scheme is required")
    }
    // ...
    return o, nil
}
```

#### Builder Pattern

Use fluent interfaces for configuration:

```go
r.Type(&corev1.Pod{}).
    Namespace("default").
    Selector(labels.Set{"app": "myapp"}).
    HandlerFunc(handler)
```

### Design Patterns

Follow these established patterns in the codebase:

1. **Interface-based design** - Small, focused interfaces
2. **Builder pattern** - Fluent configuration APIs
3. **Factory functions** - `New()` and `Default*()` constructors
4. **Middleware pattern** - For cross-cutting concerns
5. **Options pattern** - For configuration with validation

See [docs/architecture/patterns.md](docs/architecture/patterns.md) for detailed examples.

---

## Testing

### Writing Tests

- Place tests in `*_test.go` files
- Use table-driven tests when appropriate
- Use `testify` for assertions
- Use `autogold` for snapshot testing

Example test structure:

```go
package router_test

import (
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"

    "github.com/obot-platform/nah/pkg/router"
)

func TestRouter(t *testing.T) {
    tests := []struct {
        name    string
        input   interface{}
        want    interface{}
        wantErr bool
    }{
        {
            name:  "valid input",
            input: validInput,
            want:  expectedOutput,
        },
        {
            name:    "invalid input",
            input:   invalidInput,
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := router.Process(tt.input)
            if tt.wantErr {
                require.Error(t, err)
                return
            }
            require.NoError(t, err)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

### Running Tests

```bash
# All tests
make test

# Specific package
go test ./pkg/router

# With coverage
go test -cover ./...

# With race detector
go test -race ./...

# Verbose output
go test -v ./...
```

---

## Commit Messages

### Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types

- `feat` - New feature
- `fix` - Bug fix
- `docs` - Documentation changes
- `style` - Formatting, missing semi-colons, etc; no code change
- `refactor` - Refactoring production code
- `test` - Adding tests, refactoring tests
- `chore` - Updating build tasks, package manager configs, etc

### Examples

```
feat(router): add support for field selectors

Add FieldSelector method to RouteBuilder to enable filtering
resources by field selectors. This is useful for watching
resources on specific nodes or with specific statuses.

Closes #123
```

```
fix(apply): handle nil owner references correctly

Owner reference comparison was failing when comparing nil
values. Added nil check before dereferencing pointers.

Fixes #456
```

```
docs(readme): add quick start guide

Add comprehensive quick start section with code examples
for common use cases.
```

### Guidelines

- Use present tense ("add feature" not "added feature")
- Use imperative mood ("move cursor to..." not "moves cursor to...")
- First line should be 50 characters or less
- Reference issues and pull requests when relevant
- Explain *what* and *why*, not *how*

---

## Pull Request Process

### Before Submitting

1. **Sync with upstream**:
   ```bash
   git checkout main
   git pull upstream main
   git checkout your-feature-branch
   git rebase main
   ```

2. **Run all checks**:
   ```bash
   make test
   make validate
   go generate ./...
   go mod tidy
   ```

3. **Ensure no uncommitted generated files**:
   ```bash
   make validate-ci
   ```

### Submitting

1. Push your branch:
   ```bash
   git push origin your-feature-branch
   ```

2. Create a Pull Request on GitHub

3. Fill out the PR template with:
   - Description of changes
   - Related issues
   - Testing performed
   - Documentation updates

### PR Guidelines

- **Title**: Clear, concise description
- **Description**: Explain the problem and solution
- **Tests**: Add tests for new functionality
- **Documentation**: Update docs for API changes
- **Size**: Keep PRs focused and reasonably sized
- **Commits**: Squash commits if requested

### Review Process

1. Automated checks must pass (tests, linting, etc.)
2. At least one maintainer approval required
3. Address review feedback
4. Maintainer will merge once approved

---

## Documentation

### API Documentation

- Document all exported APIs
- Include examples in godoc comments
- Update package documentation in `docs/packages/`

### README Updates

Update README.md for:
- New features
- Changed APIs
- New dependencies

### Architecture Documentation

Document significant changes in:
- `docs/architecture/` - Architecture decisions
- `docs/architecture/adr/` - Architecture Decision Records

### Writing Documentation

- Use clear, concise language
- Include code examples
- Link to related documentation
- Keep examples runnable and up-to-date

---

## Release Process

(For maintainers)

1. Update version in relevant files
2. Update CHANGELOG.md
3. Create and push tag:
   ```bash
   git tag -a v1.2.3 -m "Release v1.2.3"
   git push upstream v1.2.3
   ```
4. Create GitHub release with changelog

---

## Questions?

- **Issues**: [GitHub Issues](https://github.com/obot-platform/nah/issues)
- **Discussions**: [GitHub Discussions](https://github.com/obot-platform/nah/discussions)
- **Documentation**: [docs/](docs/)

---

## License

By contributing to nah, you agree that your contributions will be licensed under the Apache License 2.0.

---

**Thank you for contributing to nah!** 🎉
