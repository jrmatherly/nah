# Controller Testing Guide

> **Version**: 1.0.0 | **Created**: 2026-01-19 | **Status**: Production Ready

This guide covers testing patterns for nah-based Kubernetes controllers, from unit tests to integration testing with real backends.

## Table of Contents

1. [Overview](#overview)
2. [Testing Strategy](#testing-strategy)
3. [Unit Testing](#unit-testing)
4. [Integration Testing](#integration-testing)
5. [Mock Backends](#mock-backends)
6. [Test Fixtures](#test-fixtures)
7. [envtest Patterns](#envtest-patterns)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)

---

## Overview

### Why Test Controllers?

Kubernetes controllers manage critical infrastructure. Comprehensive testing ensures:

- **Correctness**: Controllers reconcile resources to desired state
- **Reliability**: Failures are handled gracefully
- **Maintainability**: Refactoring doesn't break functionality
- **Confidence**: Changes can be deployed safely to production

### Testing Pyramid

```
         ┌─────────────┐
         │  E2E Tests  │  (Few - Slow - Real cluster)
         │   ~5-10%    │
         └─────────────┘
        ┌───────────────┐
        │ Integration   │  (Some - Medium - envtest/kinm)
        │  Tests ~30%   │
        └───────────────┘
       ┌─────────────────┐
       │  Unit Tests     │  (Many - Fast - Mocks)
       │    ~60-65%      │
       └─────────────────┘
```

### Testing Levels

| Level | Speed | Scope | Backend | Use For |
|-------|-------|-------|---------|---------|
| **Unit** | ⚡ Fast | Function/method | Mocks | Handler logic, validation |
| **Integration** | 🐢 Medium | Component | kinm/envtest | Router behavior, Apply operations |
| **E2E** | 🐌 Slow | System | Real cluster | Full reconciliation loops |

### Key Testing Concepts

- **Table-Driven Tests**: Parameterized test cases for comprehensive coverage
- **Test Fixtures**: Reusable test data and Kubernetes objects
- **Mock Backends**: In-memory implementations for fast unit tests
- **envtest**: Kubernetes control plane for integration testing
- **kinm**: Lightweight file-backed API server for development

---

## Testing Strategy

### What to Test

#### Handler Logic (Unit Tests)

Test the core reconciliation logic in isolation:

```go
func TestReconcileDeployment(t *testing.T) {
    tests := []struct {
        name        string
        deployment  *appsv1.Deployment
        wantReplicas int32
        wantErr     bool
    }{
        {
            name: "scale up deployment",
            deployment: &appsv1.Deployment{
                ObjectMeta: metav1.ObjectMeta{
                    Name: "test-deploy",
                    Annotations: map[string]string{
                        "autoscale.example.com/target": "5",
                    },
                },
                Spec: appsv1.DeploymentSpec{
                    Replicas: ptr.To[int32](1),
                },
            },
            wantReplicas: 5,
            wantErr:     false,
        },
        {
            name: "invalid target annotation",
            deployment: &appsv1.Deployment{
                ObjectMeta: metav1.ObjectMeta{
                    Name: "test-deploy",
                    Annotations: map[string]string{
                        "autoscale.example.com/target": "invalid",
                    },
                },
            },
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := reconcileDeployment(tt.deployment)

            if (err != nil) != tt.wantErr {
                t.Errorf("reconcileDeployment() error = %v, wantErr %v", err, tt.wantErr)
                return
            }

            if !tt.wantErr && result.Spec.Replicas != nil {
                if *result.Spec.Replicas != tt.wantReplicas {
                    t.Errorf("replicas = %d, want %d", *result.Spec.Replicas, tt.wantReplicas)
                }
            }
        })
    }
}
```

#### Event Router Integration (Integration Tests)

Test that events are routed correctly:

```go
func TestRouterIntegration(t *testing.T) {
    ctx := context.Background()

    // Use kinm for fast integration testing
    backend := kinmtest.NewTestBackend(t)

    r := router.New(ctx, router.Config{
        Backend: backend,
    })

    var handled atomic.Bool
    router.Type(&corev1.ConfigMap{}).
        HandlerFunc(func(req router.Request, resp router.Response) error {
            handled.Store(true)
            return nil
        }).
        Register(r)

    // Start router
    go r.Start(ctx)

    // Create resource
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-cm",
            Namespace: "default",
        },
    }
    backend.Create(ctx, cm)

    // Wait for handler to be called
    assert.Eventually(t, func() bool {
        return handled.Load()
    }, 5*time.Second, 100*time.Millisecond)
}
```

#### Apply Operations (Integration Tests)

Test declarative resource management:

```go
func TestApplyConfigMap(t *testing.T) {
    ctx := context.Background()
    backend := kinmtest.NewTestBackend(t)

    applier := apply.New(backend)

    owner := &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "owner",
            Namespace: "default",
            UID:       "owner-uid",
        },
    }

    // First apply - create
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-cm",
            Namespace: "default",
        },
        Data: map[string]string{
            "key": "value1",
        },
    }

    err := applier.WithOwner(owner).Apply(ctx, cm)
    require.NoError(t, err)

    // Verify created
    var got corev1.ConfigMap
    err = backend.Get(ctx, client.ObjectKey{Name: "test-cm", Namespace: "default"}, &got)
    require.NoError(t, err)
    assert.Equal(t, "value1", got.Data["key"])

    // Second apply - update
    cm.Data["key"] = "value2"
    err = applier.WithOwner(owner).Apply(ctx, cm)
    require.NoError(t, err)

    // Verify updated
    err = backend.Get(ctx, client.ObjectKey{Name: "test-cm", Namespace: "default"}, &got)
    require.NoError(t, err)
    assert.Equal(t, "value2", got.Data["key"])
}
```

### Test Organization

```
your-controller/
├── pkg/
│   ├── controller/
│   │   ├── controller.go
│   │   ├── controller_test.go       # Unit tests for handlers
│   │   └── controller_integration_test.go  # Integration tests
│   ├── reconcile/
│   │   ├── deployment.go
│   │   └── deployment_test.go       # Unit tests for reconcile logic
│   └── testutil/                    # Shared test utilities
│       ├── fixtures.go              # Test fixtures
│       ├── mock_backend.go          # Mock backend implementation
│       └── assert.go                # Custom assertions
└── test/
    ├── integration/                 # Integration test suites
    │   ├── suite_test.go
    │   └── controller_test.go
    └── e2e/                         # End-to-end tests
        └── e2e_test.go
```

---

## Unit Testing

### Handler Unit Tests

Test handler logic without starting the router:

```go
package controller

import (
    "context"
    "testing"
    "github.com/obot-platform/nah/pkg/router"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func TestConfigMapHandler(t *testing.T) {
    tests := []struct {
        name    string
        cm      *corev1.ConfigMap
        wantErr bool
        check   func(t *testing.T, resp router.Response)
    }{
        {
            name: "valid configmap creates service",
            cm: &corev1.ConfigMap{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "app-config",
                    Namespace: "default",
                    Annotations: map[string]string{
                        "create-service": "true",
                    },
                },
            },
            wantErr: false,
            check: func(t *testing.T, resp router.Response) {
                objects := resp.Objects()
                require.Len(t, objects, 1)
                svc, ok := objects[0].(*corev1.Service)
                require.True(t, ok)
                assert.Equal(t, "app-config-svc", svc.Name)
            },
        },
        {
            name: "missing annotation skips service creation",
            cm: &corev1.ConfigMap{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "app-config",
                    Namespace: "default",
                },
            },
            wantErr: false,
            check: func(t *testing.T, resp router.Response) {
                assert.Empty(t, resp.Objects())
            },
        },
        {
            name: "invalid namespace returns error",
            cm: &corev1.ConfigMap{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "app-config",
                    Namespace: "",
                },
            },
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Create test request
            req := router.Request{
                Object: tt.cm,
            }
            resp := router.NewResponse()

            // Call handler
            err := handleConfigMap(req, resp)

            // Check error
            if tt.wantErr {
                require.Error(t, err)
                return
            }
            require.NoError(t, err)

            // Run custom checks
            if tt.check != nil {
                tt.check(t, resp)
            }
        })
    }
}

// Handler function to test
func handleConfigMap(req router.Request, resp router.Response) error {
    cm := req.Object.(*corev1.ConfigMap)

    if cm.Namespace == "" {
        return fmt.Errorf("namespace is required")
    }

    if cm.Annotations["create-service"] == "true" {
        svc := &corev1.Service{
            ObjectMeta: metav1.ObjectMeta{
                Name:      cm.Name + "-svc",
                Namespace: cm.Namespace,
            },
            Spec: corev1.ServiceSpec{
                Type: corev1.ServiceTypeClusterIP,
            },
        }
        resp.Objects(svc)
    }

    return nil
}
```

### Middleware Unit Tests

Test middleware transformation and filtering:

```go
func TestValidationMiddleware(t *testing.T) {
    tests := []struct {
        name       string
        cm         *corev1.ConfigMap
        wantCalled bool
        wantErr    bool
    }{
        {
            name: "valid configmap passes through",
            cm: &corev1.ConfigMap{
                ObjectMeta: metav1.ObjectMeta{
                    Name: "valid-cm",
                    Labels: map[string]string{
                        "app": "test",
                    },
                },
            },
            wantCalled: true,
            wantErr:    false,
        },
        {
            name: "missing label is rejected",
            cm: &corev1.ConfigMap{
                ObjectMeta: metav1.ObjectMeta{
                    Name: "invalid-cm",
                },
            },
            wantCalled: false,
            wantErr:    true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            var handlerCalled bool

            // Create middleware
            mw := validateLabels(func(req router.Request, resp router.Response) error {
                handlerCalled = true
                return nil
            })

            // Create test request
            req := router.Request{
                Object: tt.cm,
            }
            resp := router.NewResponse()

            // Call middleware
            err := mw(req, resp)

            // Verify
            assert.Equal(t, tt.wantCalled, handlerCalled)
            if tt.wantErr {
                require.Error(t, err)
            } else {
                require.NoError(t, err)
            }
        })
    }
}

// Middleware to test
func validateLabels(next router.Handler) router.Handler {
    return router.HandlerFunc(func(req router.Request, resp router.Response) error {
        obj := req.Object.(metav1.Object)
        if obj.GetLabels()["app"] == "" {
            return fmt.Errorf("missing required label: app")
        }
        return next.Handle(req, resp)
    })
}
```

### Testing Reconciliation Logic

Break down complex reconciliation into testable functions:

```go
package reconcile

import (
    "testing"
    "github.com/stretchr/testify/assert"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// Testable reconciliation function
func computeDesiredReplicas(deploy *appsv1.Deployment, metrics *Metrics) (int32, error) {
    if deploy.Annotations["autoscale"] != "true" {
        return *deploy.Spec.Replicas, nil
    }

    target := metrics.CPUUtilization
    if target > 80 {
        return *deploy.Spec.Replicas + 1, nil
    }
    if target < 20 && *deploy.Spec.Replicas > 1 {
        return *deploy.Spec.Replicas - 1, nil
    }

    return *deploy.Spec.Replicas, nil
}

func TestComputeDesiredReplicas(t *testing.T) {
    tests := []struct {
        name     string
        deploy   *appsv1.Deployment
        metrics  *Metrics
        want     int32
        wantErr  bool
    }{
        {
            name: "scale up on high CPU",
            deploy: &appsv1.Deployment{
                ObjectMeta: metav1.ObjectMeta{
                    Annotations: map[string]string{
                        "autoscale": "true",
                    },
                },
                Spec: appsv1.DeploymentSpec{
                    Replicas: ptr.To[int32](2),
                },
            },
            metrics: &Metrics{CPUUtilization: 85},
            want:    3,
        },
        {
            name: "scale down on low CPU",
            deploy: &appsv1.Deployment{
                ObjectMeta: metav1.ObjectMeta{
                    Annotations: map[string]string{
                        "autoscale": "true",
                    },
                },
                Spec: appsv1.DeploymentSpec{
                    Replicas: ptr.To[int32](3),
                },
            },
            metrics: &Metrics{CPUUtilization: 15},
            want:    2,
        },
        {
            name: "no scaling when autoscale disabled",
            deploy: &appsv1.Deployment{
                Spec: appsv1.DeploymentSpec{
                    Replicas: ptr.To[int32](5),
                },
            },
            metrics: &Metrics{CPUUtilization: 90},
            want:    5,
        },
        {
            name: "maintain replicas in normal range",
            deploy: &appsv1.Deployment{
                ObjectMeta: metav1.ObjectMeta{
                    Annotations: map[string]string{
                        "autoscale": "true",
                    },
                },
                Spec: appsv1.DeploymentSpec{
                    Replicas: ptr.To[int32](3),
                },
            },
            metrics: &Metrics{CPUUtilization: 50},
            want:    3,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := computeDesiredReplicas(tt.deploy, tt.metrics)

            if tt.wantErr {
                assert.Error(t, err)
                return
            }

            assert.NoError(t, err)
            assert.Equal(t, tt.want, got)
        })
    }
}

type Metrics struct {
    CPUUtilization float64
}
```

---

## Integration Testing

### Testing with kinm

kinm provides a lightweight file-backed API server for fast integration tests:

```go
package controller_test

import (
    "context"
    "testing"
    "time"

    "github.com/obot-platform/nah/pkg/router"
    "github.com/obot-platform/kinm/pkg/kinmtest"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

func TestControllerIntegration(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // Create test backend
    backend := kinmtest.NewTestBackend(t)

    // Create router with backend
    r := router.New(ctx, router.Config{
        Backend:   backend,
        Namespace: "default",
    })

    // Register handler
    var reconcileCount int
    router.Type(&corev1.ConfigMap{}).
        HandlerFunc(func(req router.Request, resp router.Response) error {
            reconcileCount++
            cm := req.Object.(*corev1.ConfigMap)

            // Create a service for each configmap
            svc := &corev1.Service{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      cm.Name + "-svc",
                    Namespace: cm.Namespace,
                },
                Spec: corev1.ServiceSpec{
                    Type: corev1.ServiceTypeClusterIP,
                    Ports: []corev1.ServicePort{
                        {Port: 80},
                    },
                },
            }
            resp.Objects(svc)
            return nil
        }).
        Register(r)

    // Start router in background
    go func() {
        if err := r.Start(ctx); err != nil {
            t.Errorf("router failed: %v", err)
        }
    }()

    // Wait for router to be ready
    time.Sleep(100 * time.Millisecond)

    // Create ConfigMap
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-config",
            Namespace: "default",
        },
    }
    err := backend.Create(ctx, cm)
    require.NoError(t, err)

    // Wait for service to be created
    var svc corev1.Service
    assert.Eventually(t, func() bool {
        err := backend.Get(ctx, client.ObjectKey{
            Name:      "test-config-svc",
            Namespace: "default",
        }, &svc)
        return err == nil
    }, 5*time.Second, 100*time.Millisecond, "service should be created")

    // Verify service
    assert.Equal(t, "test-config-svc", svc.Name)
    assert.Equal(t, corev1.ServiceTypeClusterIP, svc.Spec.Type)

    // Verify reconciliation happened
    assert.Greater(t, reconcileCount, 0)
}
```

### Testing Apply Operations

```go
func TestApplyIntegration(t *testing.T) {
    ctx := context.Background()
    backend := kinmtest.NewTestBackend(t)
    applier := apply.New(backend)

    t.Run("creates new resource", func(t *testing.T) {
        cm := &corev1.ConfigMap{
            ObjectMeta: metav1.ObjectMeta{
                Name:      "new-cm",
                Namespace: "default",
            },
            Data: map[string]string{
                "key": "value",
            },
        }

        err := applier.Apply(ctx, cm)
        require.NoError(t, err)

        // Verify creation
        var got corev1.ConfigMap
        err = backend.Get(ctx, client.ObjectKey{Name: "new-cm", Namespace: "default"}, &got)
        require.NoError(t, err)
        assert.Equal(t, "value", got.Data["key"])
    })

    t.Run("updates existing resource", func(t *testing.T) {
        // Create initial resource
        cm := &corev1.ConfigMap{
            ObjectMeta: metav1.ObjectMeta{
                Name:      "existing-cm",
                Namespace: "default",
            },
            Data: map[string]string{
                "key": "old-value",
            },
        }
        err := backend.Create(ctx, cm)
        require.NoError(t, err)

        // Apply update
        cm.Data["key"] = "new-value"
        cm.Data["new-key"] = "new-data"
        err = applier.Apply(ctx, cm)
        require.NoError(t, err)

        // Verify update
        var got corev1.ConfigMap
        err = backend.Get(ctx, client.ObjectKey{Name: "existing-cm", Namespace: "default"}, &got)
        require.NoError(t, err)
        assert.Equal(t, "new-value", got.Data["key"])
        assert.Equal(t, "new-data", got.Data["new-key"])
    })

    t.Run("preserves manual changes", func(t *testing.T) {
        // Create resource with Apply
        cm := &corev1.ConfigMap{
            ObjectMeta: metav1.ObjectMeta{
                Name:      "manual-cm",
                Namespace: "default",
            },
            Data: map[string]string{
                "managed-key": "managed-value",
            },
        }
        err := applier.Apply(ctx, cm)
        require.NoError(t, err)

        // Manually add field
        var got corev1.ConfigMap
        err = backend.Get(ctx, client.ObjectKey{Name: "manual-cm", Namespace: "default"}, &got)
        require.NoError(t, err)
        got.Data["manual-key"] = "manual-value"
        err = backend.Update(ctx, &got)
        require.NoError(t, err)

        // Apply again
        err = applier.Apply(ctx, cm)
        require.NoError(t, err)

        // Verify manual change preserved
        err = backend.Get(ctx, client.ObjectKey{Name: "manual-cm", Namespace: "default"}, &got)
        require.NoError(t, err)
        assert.Equal(t, "managed-value", got.Data["managed-key"])
        assert.Equal(t, "manual-value", got.Data["manual-key"], "manual change should be preserved")
    })
}
```

### Testing Leader Election

```go
func TestLeaderElection(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    backend := kinmtest.NewTestBackend(t)

    // Start two controller instances
    var leader1, leader2 atomic.Bool

    // Instance 1
    r1 := router.New(ctx, router.Config{
        Backend:   backend,
        Namespace: "default",
        LeaderElection: &router.LeaderElectionConfig{
            Enabled:  true,
            Identity: "instance-1",
            LeaseName: "test-controller",
            Namespace: "default",
        },
    })
    r1.OnLeaderStart(func() {
        leader1.Store(true)
    })
    r1.OnLeaderStop(func() {
        leader1.Store(false)
    })

    // Instance 2
    r2 := router.New(ctx, router.Config{
        Backend:   backend,
        Namespace: "default",
        LeaderElection: &router.LeaderElectionConfig{
            Enabled:  true,
            Identity: "instance-2",
            LeaseName: "test-controller",
            Namespace: "default",
        },
    })
    r2.OnLeaderStart(func() {
        leader2.Store(true)
    })
    r2.OnLeaderStop(func() {
        leader2.Store(false)
    })

    // Start both instances
    go r1.Start(ctx)
    go r2.Start(ctx)

    // Wait for leader election
    time.Sleep(2 * time.Second)

    // Verify only one is leader
    is1Leader := leader1.Load()
    is2Leader := leader2.Load()

    assert.True(t, is1Leader != is2Leader, "exactly one instance should be leader")

    // Stop leader
    if is1Leader {
        cancel()
        time.Sleep(5 * time.Second)
        // instance-2 should become leader
        assert.True(t, leader2.Load(), "instance-2 should become leader after instance-1 stops")
    }
}
```

---

## Mock Backends

### Creating a Mock Backend

For fast unit tests, create a simple in-memory backend:

```go
package testutil

import (
    "context"
    "fmt"
    "sync"

    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

// MockBackend is a simple in-memory backend for testing
type MockBackend struct {
    mu      sync.RWMutex
    objects map[client.ObjectKey]client.Object
    scheme  *runtime.Scheme
}

func NewMockBackend(scheme *runtime.Scheme) *MockBackend {
    return &MockBackend{
        objects: make(map[client.ObjectKey]client.Object),
        scheme:  scheme,
    }
}

func (m *MockBackend) Get(ctx context.Context, key client.ObjectKey, obj client.Object) error {
    m.mu.RLock()
    defer m.mu.RUnlock()

    stored, ok := m.objects[key]
    if !ok {
        return fmt.Errorf("object not found: %v", key)
    }

    // Copy stored object to obj
    stored.DeepCopyInto(obj)
    return nil
}

func (m *MockBackend) Create(ctx context.Context, obj client.Object) error {
    m.mu.Lock()
    defer m.mu.Unlock()

    key := client.ObjectKeyFromObject(obj)
    if _, exists := m.objects[key]; exists {
        return fmt.Errorf("object already exists: %v", key)
    }

    m.objects[key] = obj.DeepCopyObject().(client.Object)
    return nil
}

func (m *MockBackend) Update(ctx context.Context, obj client.Object) error {
    m.mu.Lock()
    defer m.mu.Unlock()

    key := client.ObjectKeyFromObject(obj)
    if _, exists := m.objects[key]; !exists {
        return fmt.Errorf("object not found: %v", key)
    }

    m.objects[key] = obj.DeepCopyObject().(client.Object)
    return nil
}

func (m *MockBackend) Delete(ctx context.Context, obj client.Object) error {
    m.mu.Lock()
    defer m.mu.Unlock()

    key := client.ObjectKeyFromObject(obj)
    if _, exists := m.objects[key]; !exists {
        return fmt.Errorf("object not found: %v", key)
    }

    delete(m.objects, key)
    return nil
}

func (m *MockBackend) List(ctx context.Context, list client.ObjectList, opts ...client.ListOption) error {
    m.mu.RLock()
    defer m.mu.RUnlock()

    // Simple list implementation - real implementation would filter by namespace, labels, etc.
    items := make([]runtime.Object, 0, len(m.objects))
    for _, obj := range m.objects {
        items = append(items, obj)
    }

    return m.scheme.Convert(&items, list, nil)
}

// Helper methods for testing

func (m *MockBackend) ObjectCount() int {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return len(m.objects)
}

func (m *MockBackend) HasObject(key client.ObjectKey) bool {
    m.mu.RLock()
    defer m.mu.RUnlock()
    _, exists := m.objects[key]
    return exists
}

func (m *MockBackend) Reset() {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.objects = make(map[client.ObjectKey]client.Object)
}
```

### Using Mock Backend in Tests

```go
func TestWithMockBackend(t *testing.T) {
    scheme := runtime.NewScheme()
    _ = corev1.AddToScheme(scheme)

    backend := testutil.NewMockBackend(scheme)

    // Create test object
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-cm",
            Namespace: "default",
        },
        Data: map[string]string{
            "key": "value",
        },
    }

    err := backend.Create(context.Background(), cm)
    require.NoError(t, err)

    // Verify created
    assert.Equal(t, 1, backend.ObjectCount())
    assert.True(t, backend.HasObject(client.ObjectKeyFromObject(cm)))

    // Test retrieval
    var got corev1.ConfigMap
    err = backend.Get(context.Background(), client.ObjectKeyFromObject(cm), &got)
    require.NoError(t, err)
    assert.Equal(t, "value", got.Data["key"])
}
```

### Mock Client for External Dependencies

Mock external services that your controller depends on:

```go
package testutil

type MockMetricsClient struct {
    CPUUtilization float64
    MemoryUsage    float64
    Err            error
}

func (m *MockMetricsClient) GetMetrics(namespace, name string) (*Metrics, error) {
    if m.Err != nil {
        return nil, m.Err
    }
    return &Metrics{
        CPU:    m.CPUUtilization,
        Memory: m.MemoryUsage,
    }, nil
}

// Use in tests
func TestReconcileWithMetrics(t *testing.T) {
    mockMetrics := &MockMetricsClient{
        CPUUtilization: 85.0,
        MemoryUsage:    70.0,
    }

    controller := &Controller{
        metricsClient: mockMetrics,
    }

    // Test reconciliation with mocked metrics
    result, err := controller.Reconcile(context.Background(), deploy)
    require.NoError(t, err)
    assert.Equal(t, expectedReplicas, result.Spec.Replicas)
}
```

---

## Test Fixtures

### Creating Reusable Fixtures

```go
package testutil

import (
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/utils/ptr"
)

// DeploymentFixture creates a test deployment with sensible defaults
func DeploymentFixture(opts ...DeploymentOption) *appsv1.Deployment {
    deploy := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-deploy",
            Namespace: "default",
            Labels: map[string]string{
                "app": "test",
            },
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: ptr.To[int32](1),
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{
                    "app": "test",
                },
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{
                        "app": "test",
                    },
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "app",
                            Image: "nginx:latest",
                        },
                    },
                },
            },
        },
    }

    for _, opt := range opts {
        opt(deploy)
    }

    return deploy
}

// DeploymentOption is a functional option for customizing deployments
type DeploymentOption func(*appsv1.Deployment)

func WithName(name string) DeploymentOption {
    return func(d *appsv1.Deployment) {
        d.Name = name
    }
}

func WithNamespace(namespace string) DeploymentOption {
    return func(d *appsv1.Deployment) {
        d.Namespace = namespace
    }
}

func WithReplicas(replicas int32) DeploymentOption {
    return func(d *appsv1.Deployment) {
        d.Spec.Replicas = ptr.To(replicas)
    }
}

func WithAnnotations(annotations map[string]string) DeploymentOption {
    return func(d *appsv1.Deployment) {
        if d.Annotations == nil {
            d.Annotations = make(map[string]string)
        }
        for k, v := range annotations {
            d.Annotations[k] = v
        }
    }
}

func WithLabels(labels map[string]string) DeploymentOption {
    return func(d *appsv1.Deployment) {
        if d.Labels == nil {
            d.Labels = make(map[string]string)
        }
        for k, v := range labels {
            d.Labels[k] = v
        }
    }
}

// ConfigMapFixture creates a test configmap
func ConfigMapFixture(opts ...ConfigMapOption) *corev1.ConfigMap {
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-cm",
            Namespace: "default",
        },
        Data: map[string]string{},
    }

    for _, opt := range opts {
        opt(cm)
    }

    return cm
}

type ConfigMapOption func(*corev1.ConfigMap)

func WithConfigMapName(name string) ConfigMapOption {
    return func(cm *corev1.ConfigMap) {
        cm.Name = name
    }
}

func WithData(data map[string]string) ConfigMapOption {
    return func(cm *corev1.ConfigMap) {
        if cm.Data == nil {
            cm.Data = make(map[string]string)
        }
        for k, v := range data {
            cm.Data[k] = v
        }
    }
}
```

### Using Fixtures in Tests

```go
func TestDeploymentScaling(t *testing.T) {
    tests := []struct {
        name   string
        deploy *appsv1.Deployment
        want   int32
    }{
        {
            name: "scale deployment with autoscale annotation",
            deploy: testutil.DeploymentFixture(
                testutil.WithName("auto-deploy"),
                testutil.WithReplicas(1),
                testutil.WithAnnotations(map[string]string{
                    "autoscale": "true",
                    "target-cpu": "80",
                }),
            ),
            want: 3,
        },
        {
            name: "no scaling without annotation",
            deploy: testutil.DeploymentFixture(
                testutil.WithName("static-deploy"),
                testutil.WithReplicas(2),
            ),
            want: 2,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := scaleDeployment(tt.deploy, 85.0)
            assert.Equal(t, tt.want, *result.Spec.Replicas)
        })
    }
}
```

---

## envtest Patterns

### Setting Up envtest

envtest runs a real Kubernetes API server for integration testing:

```go
package controller_test

import (
    "context"
    "path/filepath"
    "testing"
    "time"

    "github.com/obot-platform/nah/pkg/router"
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"

    corev1 "k8s.io/api/core/v1"
    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/rest"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/envtest"
)

var (
    cfg       *rest.Config
    k8sClient client.Client
    testEnv   *envtest.Environment
    ctx       context.Context
    cancel    context.CancelFunc
)

func TestControllers(t *testing.T) {
    RegisterFailHandler(Fail)
    RunSpecs(t, "Controller Suite")
}

var _ = BeforeSuite(func() {
    ctx, cancel = context.WithCancel(context.Background())

    By("bootstrapping test environment")
    testEnv = &envtest.Environment{
        CRDDirectoryPaths:     []string{filepath.Join("..", "config", "crd", "bases")},
        ErrorIfCRDPathMissing: false,
    }

    var err error
    cfg, err = testEnv.Start()
    Expect(err).NotTo(HaveOccurred())
    Expect(cfg).NotTo(BeNil())

    // Create client
    k8sClient, err = client.New(cfg, client.Options{Scheme: scheme.Scheme})
    Expect(err).NotTo(HaveOccurred())
    Expect(k8sClient).NotTo(BeNil())
})

var _ = AfterSuite(func() {
    cancel()
    By("tearing down the test environment")
    err := testEnv.Stop()
    Expect(err).NotTo(HaveOccurred())
})
```

### Writing envtest Tests

```go
var _ = Describe("ConfigMap Controller", func() {
    var (
        namespace string
        r         *router.Router
    )

    BeforeEach(func() {
        // Create test namespace
        namespace = "test-" + randString(5)
        ns := &corev1.Namespace{
            ObjectMeta: metav1.ObjectMeta{
                Name: namespace,
            },
        }
        Expect(k8sClient.Create(ctx, ns)).To(Succeed())

        // Create router
        r = router.New(ctx, router.Config{
            Client:    k8sClient,
            Namespace: namespace,
        })

        // Register handlers
        router.Type(&corev1.ConfigMap{}).
            HandlerFunc(handleConfigMap).
            Register(r)

        // Start router
        go func() {
            defer GinkgoRecover()
            Expect(r.Start(ctx)).To(Succeed())
        }()
    })

    AfterEach(func() {
        // Clean up namespace
        ns := &corev1.Namespace{
            ObjectMeta: metav1.ObjectMeta{
                Name: namespace,
            },
        }
        Expect(k8sClient.Delete(ctx, ns)).To(Succeed())
    })

    It("should create a service for configmap with annotation", func() {
        By("creating a configmap")
        cm := &corev1.ConfigMap{
            ObjectMeta: metav1.ObjectMeta{
                Name:      "test-cm",
                Namespace: namespace,
                Annotations: map[string]string{
                    "create-service": "true",
                },
            },
        }
        Expect(k8sClient.Create(ctx, cm)).To(Succeed())

        By("verifying service was created")
        svc := &corev1.Service{}
        Eventually(func() error {
            return k8sClient.Get(ctx, client.ObjectKey{
                Name:      "test-cm-svc",
                Namespace: namespace,
            }, svc)
        }, 10*time.Second, 500*time.Millisecond).Should(Succeed())

        Expect(svc.Spec.Type).To(Equal(corev1.ServiceTypeClusterIP))
    })

    It("should update service when configmap changes", func() {
        By("creating a configmap")
        cm := &corev1.ConfigMap{
            ObjectMeta: metav1.ObjectMeta{
                Name:      "test-cm",
                Namespace: namespace,
                Annotations: map[string]string{
                    "create-service": "true",
                },
            },
            Data: map[string]string{
                "port": "8080",
            },
        }
        Expect(k8sClient.Create(ctx, cm)).To(Succeed())

        By("waiting for service")
        svc := &corev1.Service{}
        Eventually(func() error {
            return k8sClient.Get(ctx, client.ObjectKey{
                Name:      "test-cm-svc",
                Namespace: namespace,
            }, svc)
        }).Should(Succeed())

        By("updating configmap")
        cm.Data["port"] = "9090"
        Expect(k8sClient.Update(ctx, cm)).To(Succeed())

        By("verifying service was updated")
        Eventually(func() string {
            _ = k8sClient.Get(ctx, client.ObjectKeyFromObject(svc), svc)
            return svc.Annotations["port"]
        }).Should(Equal("9090"))
    })
})
```

### Custom Assertions

```go
package testutil

import (
    "context"
    "fmt"
    "time"

    "k8s.io/apimachinery/pkg/api/errors"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

// EventuallyExists waits for an object to exist
func EventuallyExists(ctx context.Context, c client.Client, obj client.Object, timeout time.Duration) error {
    key := client.ObjectKeyFromObject(obj)
    deadline := time.Now().Add(timeout)

    for time.Now().Before(deadline) {
        if err := c.Get(ctx, key, obj); err == nil {
            return nil
        } else if !errors.IsNotFound(err) {
            return err
        }
        time.Sleep(100 * time.Millisecond)
    }

    return fmt.Errorf("object %v did not exist within %v", key, timeout)
}

// EventuallyDeleted waits for an object to be deleted
func EventuallyDeleted(ctx context.Context, c client.Client, obj client.Object, timeout time.Duration) error {
    key := client.ObjectKeyFromObject(obj)
    deadline := time.Now().Add(timeout)

    for time.Now().Before(deadline) {
        if err := c.Get(ctx, key, obj); errors.IsNotFound(err) {
            return nil
        } else if err != nil {
            return err
        }
        time.Sleep(100 * time.Millisecond)
    }

    return fmt.Errorf("object %v still exists after %v", key, timeout)
}

// EventuallyHasLabel waits for an object to have a specific label
func EventuallyHasLabel(ctx context.Context, c client.Client, obj client.Object, key, value string, timeout time.Duration) error {
    objKey := client.ObjectKeyFromObject(obj)
    deadline := time.Now().Add(timeout)

    for time.Now().Before(deadline) {
        if err := c.Get(ctx, objKey, obj); err != nil {
            return err
        }

        if v, ok := obj.GetLabels()[key]; ok && v == value {
            return nil
        }

        time.Sleep(100 * time.Millisecond)
    }

    return fmt.Errorf("object %v did not have label %s=%s within %v", objKey, key, value, timeout)
}
```

---

## Best Practices

### Test Organization

**✅ DO:**

```go
// Group related tests using subtests
func TestDeploymentController(t *testing.T) {
    t.Run("scaling", func(t *testing.T) {
        t.Run("scales up on high CPU", func(t *testing.T) { ... })
        t.Run("scales down on low CPU", func(t *testing.T) { ... })
    })

    t.Run("labels", func(t *testing.T) {
        t.Run("adds default labels", func(t *testing.T) { ... })
        t.Run("preserves custom labels", func(t *testing.T) { ... })
    })
}
```

**❌ DON'T:**

```go
// Flat test structure without organization
func TestDeploymentControllerScaleUp(t *testing.T) { ... }
func TestDeploymentControllerScaleDown(t *testing.T) { ... }
func TestDeploymentControllerAddLabels(t *testing.T) { ... }
func TestDeploymentControllerPreserveLabels(t *testing.T) { ... }
```

### Table-Driven Tests

**✅ DO:**

```go
func TestValidateConfig(t *testing.T) {
    tests := []struct {
        name    string
        config  *Config
        wantErr bool
    }{
        {name: "valid config", config: &Config{Port: 8080}, wantErr: false},
        {name: "missing port", config: &Config{}, wantErr: true},
        {name: "invalid port", config: &Config{Port: -1}, wantErr: true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := validateConfig(tt.config)
            if (err != nil) != tt.wantErr {
                t.Errorf("validateConfig() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

**❌ DON'T:**

```go
// Separate test for each case
func TestValidConfigDoesNotError(t *testing.T) { ... }
func TestMissingPortErrors(t *testing.T) { ... }
func TestInvalidPortErrors(t *testing.T) { ... }
```

### Test Isolation

**✅ DO:**

```go
func TestControllerIsolation(t *testing.T) {
    tests := []struct {
        name string
        test func(t *testing.T)
    }{
        {
            name: "test 1",
            test: func(t *testing.T) {
                backend := kinmtest.NewTestBackend(t) // Fresh backend per test
                // ... test logic
            },
        },
        {
            name: "test 2",
            test: func(t *testing.T) {
                backend := kinmtest.NewTestBackend(t) // Fresh backend per test
                // ... test logic
            },
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, tt.test)
    }
}
```

**❌ DON'T:**

```go
// Shared state between tests
var sharedBackend *kinmtest.TestBackend

func TestSomething1(t *testing.T) {
    // Uses sharedBackend - may fail if TestSomething2 runs first
}

func TestSomething2(t *testing.T) {
    // Uses sharedBackend - may fail if TestSomething1 runs first
}
```

### Async Testing

**✅ DO:**

```go
func TestAsyncOperation(t *testing.T) {
    done := make(chan bool)

    go func() {
        // Async operation
        performAsyncWork()
        done <- true
    }()

    select {
    case <-done:
        // Success
    case <-time.After(5 * time.Second):
        t.Fatal("timeout waiting for async operation")
    }
}
```

**✅ BETTER (using testify):**

```go
func TestAsyncOperation(t *testing.T) {
    var completed atomic.Bool

    go func() {
        performAsyncWork()
        completed.Store(true)
    }()

    assert.Eventually(t, func() bool {
        return completed.Load()
    }, 5*time.Second, 100*time.Millisecond)
}
```

### Test Naming

**✅ DO:**

```go
func TestDeploymentController_ReconcileScalesUpOnHighCPU(t *testing.T) { ... }
func TestApply_CreatesNewResource(t *testing.T) { ... }
func TestMiddleware_FiltersInvalidObjects(t *testing.T) { ... }
```

**❌ DON'T:**

```go
func TestDeployment(t *testing.T) { ... }  // Too vague
func Test1(t *testing.T) { ... }            // Not descriptive
func TestStuff(t *testing.T) { ... }        // Not specific
```

### Error Messages

**✅ DO:**

```go
if got != want {
    t.Errorf("replicas mismatch: got %d, want %d (input: %+v)", got, want, input)
}
```

**❌ DON'T:**

```go
if got != want {
    t.Error("replicas wrong")  // Not helpful for debugging
}
```

---

## Troubleshooting

### Common Issues

#### Tests Hang or Timeout

**Problem:** Tests never complete or timeout

**Solutions:**

1. **Check context cancellation:**
   ```go
   ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
   defer cancel() // Always defer cancel
   ```

2. **Use Eventually with timeout:**
   ```go
   assert.Eventually(t, func() bool {
       return condition()
   }, 10*time.Second, 100*time.Millisecond)
   ```

3. **Stop router properly:**
   ```go
   ctx, cancel := context.WithCancel(context.Background())
   go router.Start(ctx)
   // Later
   cancel() // Stops router gracefully
   ```

#### Flaky Tests

**Problem:** Tests pass sometimes, fail others

**Solutions:**

1. **Increase timeouts for async operations:**
   ```go
   // Too aggressive
   time.Sleep(100 * time.Millisecond)

   // Better - wait for condition
   assert.Eventually(t, condition, 5*time.Second, 100*time.Millisecond)
   ```

2. **Ensure test isolation:**
   ```go
   // Create fresh backend for each test
   t.Run("test case", func(t *testing.T) {
       backend := kinmtest.NewTestBackend(t)
       // Test with isolated backend
   })
   ```

3. **Avoid race conditions:**
   ```go
   // Use atomic for concurrent access
   var counter atomic.Int32
   counter.Add(1)
   ```

#### Resource Leaks

**Problem:** Tests leave resources behind

**Solutions:**

1. **Use defer for cleanup:**
   ```go
   func TestWithCleanup(t *testing.T) {
       backend := kinmtest.NewTestBackend(t)
       defer backend.Reset() // Clean up after test

       // Test logic
   }
   ```

2. **Use t.Cleanup:**
   ```go
   func TestWithCleanup(t *testing.T) {
       backend := kinmtest.NewTestBackend(t)
       t.Cleanup(func() {
           backend.Reset()
       })
   }
   ```

3. **envtest namespace cleanup:**
   ```go
   AfterEach(func() {
       ns := &corev1.Namespace{
           ObjectMeta: metav1.ObjectMeta{Name: testNamespace},
       }
       Expect(k8sClient.Delete(ctx, ns)).To(Succeed())
   })
   ```

#### envtest Fails to Start

**Problem:** envtest.Environment.Start() fails

**Solutions:**

1. **Install envtest binaries:**
   ```bash
   make envtest
   ./bin/setup-envtest use
   ```

2. **Set KUBEBUILDER_ASSETS:**
   ```bash
   export KUBEBUILDER_ASSETS=$(./bin/setup-envtest use -p path)
   go test ./...
   ```

3. **Check CRD paths:**
   ```go
   testEnv = &envtest.Environment{
       CRDDirectoryPaths:     []string{filepath.Join("..", "config", "crd")},
       ErrorIfCRDPathMissing: true, // Helps identify missing CRDs
   }
   ```

### Debugging Tests

#### Enable Verbose Output

```bash
# Run tests with verbose output
go test -v ./...

# Run specific test
go test -v -run TestDeploymentController ./pkg/controller

# Show full output
go test -v -count=1 ./... 2>&1 | tee test.log
```

#### Debug with Delve

```bash
# Debug a specific test
dlv test ./pkg/controller -- -test.run TestDeploymentController

# In delve
(dlv) break controller_test.go:42
(dlv) continue
```

#### Add Debug Logging

```go
func TestWithDebug(t *testing.T) {
    // Enable debug logging
    logger := log.New(os.Stdout, "TEST: ", log.LstdFlags)

    logger.Println("Creating backend")
    backend := kinmtest.NewTestBackend(t)

    logger.Println("Starting router")
    // ... test logic
}
```

#### Inspect Test Objects

```go
func TestDebugObjects(t *testing.T) {
    cm := testutil.ConfigMapFixture()

    // Pretty-print object for debugging
    t.Logf("ConfigMap: %+v", cm)

    // JSON marshal for detailed view
    data, _ := json.MarshalIndent(cm, "", "  ")
    t.Logf("ConfigMap JSON:\n%s", string(data))
}
```

---

## Summary

### Testing Checklist

Before committing controller code:

- [ ] **Unit tests** for all handler logic
- [ ] **Integration tests** for router behavior
- [ ] **Apply tests** for resource management
- [ ] **Table-driven tests** for comprehensive coverage
- [ ] **Test fixtures** for reusable test data
- [ ] **Mock backends** for fast unit tests
- [ ] **Proper cleanup** with defer or t.Cleanup
- [ ] **Async handling** with Eventually or timeouts
- [ ] **Test isolation** - no shared state
- [ ] **Clear naming** - TestComponent_Method_Scenario
- [ ] **Good error messages** with context

### Quick Reference

| Task | Command |
|------|---------|
| Run all tests | `go test ./...` |
| Run with coverage | `go test -cover ./...` |
| Run specific test | `go test -run TestName ./pkg/...` |
| Run with race detector | `go test -race ./...` |
| Verbose output | `go test -v ./...` |
| Generate coverage report | `go test -coverprofile=coverage.out ./...` |
| View coverage HTML | `go tool cover -html=coverage.out` |
| Run integration tests only | `go test -tags=integration ./...` |
| Skip integration tests | `go test -short ./...` |

### Testing Strategy Summary

```
┌──────────────────────────────────────────────────┐
│           Write Tests First (TDD)                │
│                      ↓                           │
│  ┌────────────┐  ┌────────────┐  ┌───────────┐ │
│  │ Unit Tests │→ │ Integration│→ │ E2E Tests │ │
│  │  (Mocks)   │  │  (kinm)    │  │ (Cluster) │ │
│  └────────────┘  └────────────┘  └───────────┘ │
│                      ↓                           │
│           Refactor with Confidence               │
└──────────────────────────────────────────────────┘
```

---

*For more information on building controllers, see [controllers.md](./controllers.md). For production deployment, see [leader-election.md](./leader-election.md).*
