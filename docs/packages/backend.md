# Backend Package API Reference

**Package:** `github.com/obot-platform/nah/pkg/backend`

The `backend` package provides an abstraction layer over Kubernetes client operations with caching, watching, and triggering capabilities.

## Table of Contents

- [Interfaces](#interfaces)
  - [Backend](#backend)
  - [Trigger](#trigger)
  - [Watcher](#watcher)
  - [CacheFactory](#cachefactory)
- [Types](#types)
  - [Callback](#callback)
- [Usage Examples](#usage-examples)

---

## Interfaces

### Backend

The main interface that combines all backend capabilities for Kubernetes operations.

```go
type Backend interface {
    Trigger
    CacheFactory
    Watcher
    client.WithWatch
    client.FieldIndexer

    Preload(ctx context.Context) error
    Start(ctx context.Context) error
    GVKForObject(obj runtime.Object, scheme *runtime.Scheme) (schema.GroupVersionKind, error)
}
```

**Embedded Interfaces:**

- `Trigger` - Trigger reconciliation for resources
- `CacheFactory` - Access informers and caches
- `Watcher` - Watch resource changes
- `client.WithWatch` - Standard Kubernetes client operations (Get, List, Create, Update, Delete, Patch, Watch)
- `client.FieldIndexer` - Field indexing for efficient lookups

#### Methods

##### Preload

```go
func Preload(ctx context.Context) error
```

Preloads the cache before starting the backend.

**Parameters:**

- `ctx` - Context for the operation

**Returns:**

- `error` - Error if preload fails

##### Start

```go
func Start(ctx context.Context) error
```

Starts the backend and begins cache synchronization.

**Parameters:**

- `ctx` - Context for lifecycle management

**Returns:**

- `error` - Error if start fails

**Example:**

```go
if err := backend.Start(ctx); err != nil {
    return fmt.Errorf("failed to start backend: %w", err)
}
```

##### GVKForObject

```go
func GVKForObject(obj runtime.Object, scheme *runtime.Scheme) (schema.GroupVersionKind, error)
```

Returns the GroupVersionKind for a given object.

**Parameters:**

- `obj` - Runtime object
- `scheme` - Runtime scheme

**Returns:**

- `schema.GroupVersionKind` - The GVK for the object
- `error` - Error if GVK cannot be determined

**Example:**

```go
gvk, err := backend.GVKForObject(pod, scheme)
if err != nil {
    return err
}
fmt.Printf("GVK: %s\n", gvk.String())
```

---

### Trigger

Interface for triggering resource reconciliation.

```go
type Trigger interface {
    Trigger(ctx context.Context, gvk schema.GroupVersionKind, key string, delay time.Duration) error
}
```

#### Methods

##### Trigger

```go
func Trigger(ctx context.Context, gvk schema.GroupVersionKind, key string, delay time.Duration) error
```

Triggers reconciliation for a specific resource after an optional delay.

**Parameters:**

- `ctx` - Context for the operation
- `gvk` - GroupVersionKind of the resource
- `key` - Resource key in format "namespace/name"
- `delay` - Delay before triggering (0 for immediate)

**Returns:**

- `error` - Error if trigger fails

**Example:**

```go
// Immediate trigger
err := backend.Trigger(ctx, gvk, "default/my-pod", 0)

// Delayed trigger (5 seconds)
err := backend.Trigger(ctx, gvk, "default/my-pod", 5*time.Second)
```

**Use Cases:**

- Re-queue resources after transient failures
- Implement exponential backoff
- Schedule periodic reconciliation
- Cross-resource dependencies

---

### Watcher

Interface for watching resource changes with callbacks.

```go
type Watcher interface {
    Watcher(ctx context.Context, gvk schema.GroupVersionKind, name string, cb Callback) error
}
```

#### Methods

##### Watcher

```go
func Watcher(ctx context.Context, gvk schema.GroupVersionKind, name string, cb Callback) error
```

Registers a callback to be invoked when resources change.

**Parameters:**

- `ctx` - Context for the watcher lifecycle
- `gvk` - GroupVersionKind to watch
- `name` - Unique name for this watcher
- `cb` - Callback function to invoke on changes

**Returns:**

- `error` - Error if watcher registration fails

**Example:**

```go
err := backend.Watcher(ctx, podGVK, "pod-watcher", func(
    ctx context.Context,
    gvk schema.GroupVersionKind,
    key string,
    obj runtime.Object,
) (runtime.Object, error) {
    pod := obj.(*corev1.Pod)
    fmt.Printf("Pod changed: %s\n", pod.Name)
    return obj, nil
})
```

---

### CacheFactory

Interface for accessing Kubernetes informers and caches.

```go
type CacheFactory interface {
    GetInformerForKind(ctx context.Context, gvk schema.GroupVersionKind) (cache.SharedIndexInformer, error)
}
```

#### Methods

##### GetInformerForKind

```go
func GetInformerForKind(ctx context.Context, gvk schema.GroupVersionKind) (cache.SharedIndexInformer, error)
```

Returns the informer for a specific GVK.

**Parameters:**

- `ctx` - Context for the operation
- `gvk` - GroupVersionKind to get informer for

**Returns:**

- `cache.SharedIndexInformer` - Informer for the GVK
- `error` - Error if informer cannot be retrieved

**Example:**

```go
informer, err := backend.GetInformerForKind(ctx, podGVK)
if err != nil {
    return err
}

// Add event handler
informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        fmt.Println("Object added")
    },
})
```

---

## Types

### Callback

Function type for watcher callbacks.

```go
type Callback func(
    ctx context.Context,
    gvk schema.GroupVersionKind,
    key string,
    obj runtime.Object,
) (runtime.Object, error)
```

**Parameters:**

- `ctx` - Context for the callback
- `gvk` - GroupVersionKind of the changed resource
- `key` - Resource key ("namespace/name")
- `obj` - The resource object

**Returns:**

- `runtime.Object` - Potentially modified object
- `error` - Error if callback processing fails

**Example:**

```go
myCallback := func(
    ctx context.Context,
    gvk schema.GroupVersionKind,
    key string,
    obj runtime.Object,
) (runtime.Object, error) {
    // Process the object
    pod := obj.(*corev1.Pod)
    log.Printf("Processing pod: %s", pod.Name)

    // Return object (possibly modified)
    return obj, nil
}
```

---

## Usage Examples

### Basic Operations

```go
package main

import (
    "context"

    "github.com/obot-platform/nah/pkg/backend"
    "github.com/obot-platform/nah/pkg/runtime"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

func example(b backend.Backend) error {
    ctx := context.Background()

    // Get a resource
    pod := &corev1.Pod{}
    key := client.ObjectKey{Namespace: "default", Name: "my-pod"}
    if err := b.Get(ctx, key, pod); err != nil {
        return err
    }

    // List resources
    podList := &corev1.PodList{}
    if err := b.List(ctx, podList, client.InNamespace("default")); err != nil {
        return err
    }

    // Create a resource
    newPod := &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "new-pod",
            Namespace: "default",
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{
                {
                    Name:  "nginx",
                    Image: "nginx:latest",
                },
            },
        },
    }
    if err := b.Create(ctx, newPod); err != nil {
        return err
    }

    // Update a resource
    pod.Labels = map[string]string{"updated": "true"}
    if err := b.Update(ctx, pod); err != nil {
        return err
    }

    // Delete a resource
    if err := b.Delete(ctx, pod); err != nil {
        return err
    }

    return nil
}
```

### Triggering Reconciliation

```go
func triggerExample(b backend.Backend) error {
    ctx := context.Background()
    gvk := schema.GroupVersionKind{
        Group:   "",
        Version: "v1",
        Kind:    "Pod",
    }

    // Immediate trigger
    if err := b.Trigger(ctx, gvk, "default/my-pod", 0); err != nil {
        return err
    }

    // Delayed trigger (exponential backoff)
    delays := []time.Duration{
        1 * time.Second,
        2 * time.Second,
        4 * time.Second,
        8 * time.Second,
    }

    for _, delay := range delays {
        if err := b.Trigger(ctx, gvk, "default/my-pod", delay); err != nil {
            return err
        }
    }

    return nil
}
```

### Watching Resources

```go
func watchExample(b backend.Backend) error {
    ctx := context.Background()
    gvk := schema.GroupVersionKind{
        Group:   "",
        Version: "v1",
        Kind:    "ConfigMap",
    }

    // Register watcher
    err := b.Watcher(ctx, gvk, "config-watcher", func(
        ctx context.Context,
        gvk schema.GroupVersionKind,
        key string,
        obj runtime.Object,
    ) (runtime.Object, error) {
        cm := obj.(*corev1.ConfigMap)

        // React to ConfigMap changes
        log.Printf("ConfigMap changed: %s/%s", cm.Namespace, cm.Name)

        // Trigger dependent resources
        if err := b.Trigger(ctx, podGVK, key, 0); err != nil {
            return nil, err
        }

        return obj, nil
    })

    return err
}
```

### Using Informers

```go
func informerExample(b backend.Backend) error {
    ctx := context.Background()
    gvk := schema.GroupVersionKind{
        Group:   "apps",
        Version: "v1",
        Kind:    "Deployment",
    }

    // Get informer
    informer, err := b.GetInformerForKind(ctx, gvk)
    if err != nil {
        return err
    }

    // Add event handlers
    informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            log.Println("Deployment added")
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            log.Println("Deployment updated")
        },
        DeleteFunc: func(obj interface{}) {
            log.Println("Deployment deleted")
        },
    })

    return nil
}
```

### Field Indexing

```go
func indexingExample(b backend.Backend) error {
    ctx := context.Background()

    // Create field index for efficient lookups
    err := b.IndexField(ctx, &corev1.Pod{}, "spec.nodeName", func(obj client.Object) []string {
        pod := obj.(*corev1.Pod)
        return []string{pod.Spec.NodeName}
    })
    if err != nil {
        return err
    }

    // Query using field index
    podList := &corev1.PodList{}
    err = b.List(ctx, podList, client.MatchingFields{"spec.nodeName": "node-1"})
    if err != nil {
        return err
    }

    fmt.Printf("Found %d pods on node-1\n", len(podList.Items))
    return nil
}
```

---

## Performance Considerations

### Caching

The backend uses Kubernetes informers for caching:

- **List operations** are served from cache (fast)
- **Get operations** are served from cache (fast)
- **Write operations** (Create, Update, Delete) hit the API server

### Triggering Strategy

Use appropriate delays:

- **0 delay**: Immediate processing
- **Short delays (1-5s)**: Quick retry on transient errors
- **Longer delays (30s+)**: Rate limiting, periodic sync

### Watchers

- Watchers use informer callbacks (efficient)
- Callbacks run synchronously (keep them fast)
- Heavy processing should trigger separate reconciliation

---

## See Also

- [Router Package](router.md) - Event routing and handlers
- [Apply Package](apply.md) - Declarative resource management
- [Runtime Package](runtime.md) - Backend configuration
- [Watcher Package](watcher.md) - Low-level watching

---

**Package Documentation:** [pkg.go.dev/github.com/obot-platform/nah/pkg/backend](https://pkg.go.dev/github.com/obot-platform/nah/pkg/backend)
