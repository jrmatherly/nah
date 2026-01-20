# Controller Development Guide

> **Version**: 1.0.0 | **Created**: 2026-01-19 | **Status**: Production Ready

This guide covers building Kubernetes controllers using nah's Event Router and handler patterns for production-grade controller development.

## Table of Contents

1. [Overview](#overview)
2. [Event Router Fundamentals](#event-router-fundamentals)
3. [Handler Patterns](#handler-patterns)
4. [Middleware](#middleware)
5. [GVK-Specific Tuning](#gvk-specific-tuning)
6. [Backend Abstraction](#backend-abstraction)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)

---

## Overview

### What is the Event Router?

The nah Event Router provides a fluent API for directing Kubernetes events (Create, Update, Delete) to appropriate handlers. It abstracts the complexity of watching resources, managing informers, and routing events while providing powerful customization options.

### Key Features

- **Fluent API**: Chain method calls to build controller routing logic
- **Type-Safe Handlers**: Strong typing for resource objects
- **Flexible Middleware**: Intercept and modify events before handlers
- **GVK-Specific Tuning**: Per-resource configuration for workers and threadiness
- **Backend Abstraction**: Works with Kubernetes clusters or kinm API server
- **OpenTelemetry Integration**: Built-in distributed tracing support

### Architecture

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
│  │  GVK Routing   │  →     │    Middleware      │  │
│  │  Create/Update │        │    Transform       │  │
│  │  Delete/Status │        │    Filter          │  │
│  └────────────────┘        └────────────────────┘  │
└──────────┬──────────────────────────────────────────┘
           │
           ↓
┌─────────────────────────────────────────────────────┐
│           Backend Abstraction                        │
│  ┌─────────────┐           ┌────────────────────┐  │
│  │ Kubernetes  │           │       kinm         │  │
│  │  Informers  │           │   File-backed      │  │
│  │  Caching    │           │   Development      │  │
│  └─────────────┘           └────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## Event Router Fundamentals

### Basic Setup

```go
package main

import (
    "context"
    "github.com/obot-platform/nah/pkg/router"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
)

func main() {
    ctx := context.Background()

    // Create router with backend
    r := router.New(ctx, router.Config{
        Namespace: "default",
    })

    // Add routes for resources
    router.Type(&corev1.ConfigMap{}).
        HandlerFunc(handleConfigMap).
        Middleware(validateConfigMap).
        Register(r)

    router.Type(&appsv1.Deployment{}).
        HandlerFunc(handleDeployment).
        Register(r)

    // Start the controller
    if err := r.Start(ctx); err != nil {
        panic(err)
    }
}
```

### Event Types

The router handles four event types:

| Event | Trigger | Handler Signature |
|-------|---------|-------------------|
| **Create** | Resource created | `func(obj *T, status T) error` |
| **Update** | Resource modified | `func(obj *T, status T) error` |
| **Delete** | Resource deleted | `func(obj *T, status T) error` |
| **StatusUpdate** | Status subresource changed | `func(obj *T, status T) error` |

**Note**: `Create` and `Update` often use the same handler in reconciliation patterns.

### Route Registration

```go
// Route with specific event handlers
router.Type(&corev1.Pod{}).
    Create(handlePodCreate).
    Update(handlePodUpdate).
    Delete(handlePodDelete).
    Register(r)

// Route with unified handler (common pattern)
router.Type(&corev1.Service{}).
    HandlerFunc(reconcileService).  // Handles Create and Update
    Delete(cleanupService).
    Register(r)

// Route with status handler
router.Type(&appsv1.Deployment{}).
    HandlerFunc(reconcileDeployment).
    StatusHandler(updateDeploymentStatus).
    Register(r)
```

### Selector Filtering

Watch only resources matching a label selector:

```go
// Watch only resources with specific labels
router.Type(&corev1.ConfigMap{}).
    Selector(&metav1.LabelSelector{
        MatchLabels: map[string]string{
            "app.kubernetes.io/managed-by": "mycontroller",
        },
    }).
    HandlerFunc(handleConfigMap).
    Register(r)
```

---

## Handler Patterns

### Reconciliation Handler

The most common pattern: idempotent reconciliation of desired state.

```go
func reconcileDeployment(obj *appsv1.Deployment, status appsv1.DeploymentStatus) error {
    // 1. Validate the resource
    if err := validateDeployment(obj); err != nil {
        return err
    }

    // 2. Ensure dependent resources exist
    if err := ensureServiceAccount(obj); err != nil {
        return err
    }

    if err := ensureConfigMaps(obj); err != nil {
        return err
    }

    // 3. Update status to reflect current state
    obj.Status.ObservedGeneration = obj.Generation
    obj.Status.Conditions = []appsv1.DeploymentCondition{
        {
            Type:   appsv1.DeploymentAvailable,
            Status: corev1.ConditionTrue,
            Reason: "Reconciled",
        },
    }

    return nil
}
```

### Error Handling

Return errors to trigger requeue with exponential backoff:

```go
func handleResource(obj *corev1.Pod, status corev1.PodStatus) error {
    // Transient error - will be retried
    if !isReady(obj) {
        return fmt.Errorf("pod not ready, requeueing")
    }

    // Permanent error - log but don't requeue
    if obj.Annotations["invalid"] == "true" {
        log.Printf("pod %s has invalid annotation, skipping", obj.Name)
        return nil
    }

    // Process the resource
    return processResource(obj)
}
```

### Delete Handler

Clean up external resources when a Kubernetes object is deleted:

```go
func cleanupService(obj *corev1.Service, status corev1.ServiceStatus) error {
    // Check if resource has finalizer
    if !hasFinalizer(obj, "mycontroller.io/cleanup") {
        return nil
    }

    // Perform cleanup
    if err := deleteExternalLoadBalancer(obj); err != nil {
        return fmt.Errorf("failed to delete load balancer: %w", err)
    }

    // Remove finalizer to allow deletion
    obj.Finalizers = removeFinalizer(obj.Finalizers, "mycontroller.io/cleanup")

    return nil
}

func hasFinalizer(obj metav1.Object, finalizer string) bool {
    for _, f := range obj.GetFinalizers() {
        if f == finalizer {
            return true
        }
    }
    return false
}
```

### Status-Only Handler

Update status subresource without triggering full reconciliation:

```go
func updateDeploymentStatus(obj *appsv1.Deployment, status appsv1.DeploymentStatus) error {
    // Calculate ready replicas
    readyReplicas := calculateReadyReplicas(obj)

    // Update status
    obj.Status.ReadyReplicas = readyReplicas
    obj.Status.UpdatedReplicas = obj.Status.Replicas

    // Set conditions based on state
    if readyReplicas >= *obj.Spec.Replicas {
        setCondition(&obj.Status, appsv1.DeploymentAvailable, corev1.ConditionTrue)
    } else {
        setCondition(&obj.Status, appsv1.DeploymentAvailable, corev1.ConditionFalse)
    }

    return nil
}
```

---

## Middleware

Middleware intercepts events before they reach handlers, enabling cross-cutting concerns like validation, transformation, and filtering.

### Validation Middleware

```go
func validateConfigMap(obj *corev1.ConfigMap, status corev1.ConfigMapStatus, handler router.Handler) error {
    // Validate required annotations
    if _, ok := obj.Annotations["app.kubernetes.io/name"]; !ok {
        return fmt.Errorf("missing required annotation: app.kubernetes.io/name")
    }

    // Validate data keys
    if _, ok := obj.Data["config.yaml"]; !ok {
        return fmt.Errorf("missing required key: config.yaml")
    }

    // Pass to next handler
    return handler(obj, status)
}

// Register with middleware
router.Type(&corev1.ConfigMap{}).
    Middleware(validateConfigMap).
    HandlerFunc(reconcileConfigMap).
    Register(r)
```

### Transformation Middleware

```go
func addDefaultLabels(obj *corev1.Service, status corev1.ServiceStatus, handler router.Handler) error {
    // Add default labels if missing
    if obj.Labels == nil {
        obj.Labels = make(map[string]string)
    }

    if _, ok := obj.Labels["managed-by"]; !ok {
        obj.Labels["managed-by"] = "mycontroller"
    }

    return handler(obj, status)
}
```

### Filtering Middleware

```go
func ignoreSystemNamespaces(obj *corev1.Pod, status corev1.PodStatus, handler router.Handler) error {
    // Skip system namespaces
    systemNamespaces := []string{"kube-system", "kube-public", "kube-node-lease"}

    for _, ns := range systemNamespaces {
        if obj.Namespace == ns {
            return nil // Don't call handler
        }
    }

    return handler(obj, status)
}
```

### Middleware Chaining

Multiple middleware functions are executed in order:

```go
router.Type(&corev1.Pod{}).
    Middleware(ignoreSystemNamespaces).  // First
    Middleware(validatePod).              // Second
    Middleware(addDefaultLabels).         // Third
    HandlerFunc(reconcilePod).            // Final handler
    Register(r)
```

---

## GVK-Specific Tuning

Optimize controller performance by tuning workers and threadiness per resource type.

### Configuration Options

```go
// Configure router with GVK-specific settings
r := router.New(ctx, router.Config{
    Namespace: "default",

    // Global defaults
    Workers:     2,
    Threadiness: 5,

    // Per-GVK overrides
    GVKConfig: map[schema.GroupVersionKind]router.GVKConfig{
        // High-volume resources: more workers
        {Group: "", Version: "v1", Kind: "Pod"}: {
            Workers:     4,
            Threadiness: 10,
        },

        // Low-volume resources: fewer workers
        {Group: "apps", Version: "v1", Kind: "Deployment"}: {
            Workers:     1,
            Threadiness: 2,
        },

        // Critical resources: higher priority
        {Group: "", Version: "v1", Kind: "Node"}: {
            Workers:     2,
            Threadiness: 5,
            Priority:    100,
        },
    },
})
```

### Tuning Guidelines

| Resource Characteristics | Workers | Threadiness | Reasoning |
|--------------------------|---------|-------------|-----------|
| **High volume** (Pods, Events) | 4-8 | 10-20 | Maximize throughput |
| **Low volume** (CRDs, Namespaces) | 1-2 | 2-5 | Minimize overhead |
| **Critical** (Nodes, PVs) | 2-4 | 5-10 | Balance responsiveness |
| **Complex reconciliation** | 2-3 | 3-5 | Avoid resource contention |

### Monitoring and Adjustment

```go
// Enable OpenTelemetry tracing to monitor performance
r := router.New(ctx, router.Config{
    Namespace:   "default",
    EnableTrace: true,
    TracingConfig: router.TracingConfig{
        ServiceName: "mycontroller",
        Endpoint:    "localhost:4318",
    },
})

// Metrics to watch:
// - Queue depth: High values indicate need for more workers
// - Processing time: High variance suggests need for threadiness tuning
// - Requeue rate: High rate indicates handler errors or transient issues
```

---

## Backend Abstraction

nah provides a unified interface over Kubernetes and kinm backends, enabling seamless switching between production and development environments.

### Kubernetes Backend

Production deployment using a real Kubernetes cluster:

```go
import (
    "github.com/obot-platform/nah/pkg/router"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
)

func newKubernetesRouter(ctx context.Context) (*router.Router, error) {
    // Load kubeconfig
    config, err := rest.InClusterConfig()
    if err != nil {
        return nil, err
    }

    // Create Kubernetes client
    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        return nil, err
    }

    // Create router with Kubernetes backend
    return router.New(ctx, router.Config{
        Backend:   router.KubernetesBackend(clientset),
        Namespace: "default",
    }), nil
}
```

### kinm Backend

Development environment using file-backed API server:

```go
import (
    "github.com/obot-platform/nah/pkg/router"
    "github.com/obot-platform/kinm/pkg/server"
)

func newKinmRouter(ctx context.Context) (*router.Router, error) {
    // Create kinm server (file-backed)
    kinmServer, err := server.New(ctx, server.Config{
        DataDir: "./kinm-data",
    })
    if err != nil {
        return nil, err
    }

    // Create router with kinm backend
    return router.New(ctx, router.Config{
        Backend:   router.KinmBackend(kinmServer),
        Namespace: "default",
    }), nil
}
```

### Backend-Agnostic Code

Write handlers that work with both backends:

```go
func reconcileDeployment(obj *appsv1.Deployment, status appsv1.DeploymentStatus) error {
    // This code works identically with Kubernetes or kinm backends

    // Use nah's Apply API (backend-agnostic)
    if err := apply.Apply(ctx, obj, apply.WithOwner(obj)); err != nil {
        return err
    }

    return nil
}

// Switch backends via configuration
func NewRouter(ctx context.Context, useKinm bool) (*router.Router, error) {
    if useKinm {
        return newKinmRouter(ctx)
    }
    return newKubernetesRouter(ctx)
}
```

### Backend Features Comparison

| Feature | Kubernetes | kinm |
|---------|------------|------|
| **Caching** | Informer-based | In-memory |
| **Persistence** | etcd | Filesystem |
| **Performance** | Production-grade | Development-optimized |
| **Setup** | Cluster required | Standalone binary |
| **Use Case** | Production deployment | Local development, testing |

---

## Best Practices

### 1. Idempotent Handlers

Ensure handlers can be called multiple times safely:

```go
func reconcileService(obj *corev1.Service, status corev1.ServiceStatus) error {
    // ✅ Good: Check if work is already done
    if obj.Annotations["mycontroller.io/configured"] == "true" {
        return nil
    }

    // Perform configuration
    if err := configureLoadBalancer(obj); err != nil {
        return err
    }

    // Mark as configured
    if obj.Annotations == nil {
        obj.Annotations = make(map[string]string)
    }
    obj.Annotations["mycontroller.io/configured"] = "true"

    return nil
}
```

### 2. Use Finalizers for Cleanup

Ensure cleanup happens before deletion:

```go
const finalizerName = "mycontroller.io/cleanup"

func reconcilePod(obj *corev1.Pod, status corev1.PodStatus) error {
    // Add finalizer on create
    if !hasFinalizer(obj, finalizerName) {
        obj.Finalizers = append(obj.Finalizers, finalizerName)
        return nil
    }

    // Handle deletion
    if !obj.DeletionTimestamp.IsZero() {
        return cleanupPod(obj)
    }

    // Normal reconciliation
    return processResource(obj)
}
```

### 3. Separate Status Updates

Update status separately from spec to avoid conflicts:

```go
router.Type(&appsv1.Deployment{}).
    HandlerFunc(reconcileDeploymentSpec).      // Updates spec
    StatusHandler(reconcileDeploymentStatus).  // Updates status
    Register(r)
```

### 4. Handle Errors Appropriately

```go
func handleResource(obj *corev1.ConfigMap, status corev1.ConfigMapStatus) error {
    // Transient error: return to requeue
    if !isReady() {
        return fmt.Errorf("dependency not ready")
    }

    // Permanent error: log and return nil
    if obj.Data == nil {
        log.Printf("WARNING: ConfigMap %s has no data", obj.Name)
        return nil
    }

    // Fatal error: return to requeue with backoff
    if err := processData(obj.Data); err != nil {
        return fmt.Errorf("failed to process data: %w", err)
    }

    return nil
}
```

### 5. Use Structured Logging

```go
import "log/slog"

func reconcileService(obj *corev1.Service, status corev1.ServiceStatus) error {
    logger := slog.With(
        "namespace", obj.Namespace,
        "name", obj.Name,
        "uid", obj.UID,
    )

    logger.Info("reconciling service")

    if err := processService(obj); err != nil {
        logger.Error("reconciliation failed", "error", err)
        return err
    }

    logger.Info("reconciliation complete")
    return nil
}
```

### 6. Enable OpenTelemetry Tracing

```go
r := router.New(ctx, router.Config{
    Namespace:   "default",
    EnableTrace: true,
    TracingConfig: router.TracingConfig{
        ServiceName: "mycontroller",
        Endpoint:    "localhost:4318",
        SamplingRate: 0.1, // Sample 10% of requests
    },
})
```

---

## Troubleshooting

### Handler Not Called

**Symptom**: Events are received but handler never executes.

**Solutions**:

1. Check selector matches resource labels:
   ```go
   // Ensure labels match selector
   router.Type(&corev1.Pod{}).
       Selector(&metav1.LabelSelector{
           MatchLabels: map[string]string{
               "app": "myapp",  // Verify pod has this label
           },
       }).
       HandlerFunc(handlePod).
       Register(r)
   ```

2. Verify namespace filtering:
   ```go
   // Router namespace must match resource namespace
   r := router.New(ctx, router.Config{
       Namespace: "default",  // Only watches "default" namespace
   })
   ```

3. Check middleware isn't filtering:
   ```go
   func debugMiddleware(obj *corev1.Pod, status corev1.PodStatus, handler router.Handler) error {
       log.Printf("Middleware called for pod: %s/%s", obj.Namespace, obj.Name)
       return handler(obj, status)
   }
   ```

### High Requeue Rate

**Symptom**: Handlers repeatedly fail and requeue.

**Solutions**:

1. Add error handling for transient failures:
   ```go
   func handlePod(obj *corev1.Pod, status corev1.PodStatus) error {
       // Check preconditions before processing
       if !isReady(obj) {
           return fmt.Errorf("pod not ready: %w", err)
       }
       return processResource(obj)
   }
   ```

2. Use exponential backoff awareness:
   ```go
   // Return errors only for retriable failures
   // Return nil for permanent errors
   ```

3. Monitor queue depth:
   ```bash
   # Enable metrics endpoint
   curl http://localhost:9090/metrics | grep workqueue
   ```

### Performance Issues

**Symptom**: Controller lags behind events.

**Solutions**:

1. Increase workers and threadiness:
   ```go
   r := router.New(ctx, router.Config{
       Workers:     4,  // More workers
       Threadiness: 10, // More threads per worker
   })
   ```

2. Tune per-GVK:
   ```go
   GVKConfig: map[schema.GroupVersionKind]router.GVKConfig{
       podGVK: {
           Workers:     8,
           Threadiness: 20,
       },
   }
   ```

3. Profile handler execution:
   ```go
   func timedHandler(obj *corev1.Pod, status corev1.PodStatus) error {
       start := time.Now()
       defer func() {
           log.Printf("Handler took %v", time.Since(start))
       }()
       return processResource(obj)
   }
   ```

### Memory Leaks

**Symptom**: Controller memory usage grows over time.

**Solutions**:

1. Check for goroutine leaks in handlers:
   ```go
   // ❌ Bad: goroutine never exits
   go func() {
       for {
           doWork()
       }
   }()

   // ✅ Good: goroutine respects context
   go func(ctx context.Context) {
       for {
           select {
           case <-ctx.Done():
               return
           default:
               doWork()
           }
       }
   }(ctx)
   ```

2. Avoid caching large objects in handlers:
   ```go
   // ❌ Bad: global cache grows unbounded
   var cache = make(map[string]*corev1.Pod)

   // ✅ Good: use router's built-in caching
   // or implement cache eviction
   ```

### Finalizer Deadlock

**Symptom**: Resources stuck in "Terminating" state.

**Solutions**:

1. Ensure finalizer is removed:
   ```go
   func cleanup(obj *corev1.Service, status corev1.ServiceStatus) error {
       // Perform cleanup
       if err := deleteResources(obj); err != nil {
           return err  // Will retry
       }

       // CRITICAL: Remove finalizer
       obj.Finalizers = removeFinalizer(obj.Finalizers, finalizerName)
       return nil
   }
   ```

2. Add timeout to cleanup:
   ```go
   func cleanup(obj *corev1.Service, status corev1.ServiceStatus) error {
       ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
       defer cancel()

       if err := deleteResourcesWithContext(ctx, obj); err != nil {
           // Log but remove finalizer anyway to prevent deadlock
           log.Printf("cleanup failed: %v", err)
       }

       obj.Finalizers = removeFinalizer(obj.Finalizers, finalizerName)
       return nil
   }
   ```

---

## Additional Resources

| Resource | Description |
|----------|-------------|
| [apply.md](./apply.md) | Declarative resource management with nah's Apply API |
| [leader-election.md](./leader-election.md) | High-availability controller deployments |
| [testing.md](./testing.md) | Controller testing strategies and patterns |
| [performance.md](./performance.md) | Optimization and profiling techniques |
| [nah/README.md](../../README.md) | Framework overview and quick start |
| [nah/pkg/router](../../pkg/router/) | Router implementation and API reference |

---

*For workspace-wide documentation patterns, see [documentation/docs/reference/documentation-guide.md](../../../documentation/docs/reference/documentation-guide.md).*
