# Router Package API Reference

**Package:** `github.com/obot-platform/nah/pkg/router`

The `router` package provides the core event routing and handler management for Kubernetes controllers. It routes Kubernetes events to registered handlers with support for filters, middleware, and finalization.

## Table of Contents

- [Core Types](#core-types)
  - [Router](#router)
  - [RouteBuilder](#routebuilder)
  - [Handler](#handler)
  - [Request](#request)
  - [Response](#response)
- [Middleware](#middleware)
- [Filters](#filters)
- [Finalizers](#finalizers)
- [Examples](#examples)

---

## Core Types

### Router

The `Router` is the central controller component that manages the lifecycle and routes events to handlers.

```go
type Router struct {
    RouteBuilder

    OnErrorHandler ErrorHandler
    handlers       *HandlerSet
    electionConfig *leader.ElectionConfig
    startLock      sync.Mutex
    postStarts     []func(context.Context, client.Client)
    signalStopped  chan struct{}
}
```

#### Constructor

```go
func New(handlerSet *HandlerSet, electionConfig *leader.ElectionConfig, healthzPort int) *Router
```

Creates a new Router with the given handler set and election configuration.

**Parameters:**

- `handlerSet` - Collection of handlers for different resource types
- `electionConfig` - Leader election configuration (nil for no election)
- `healthzPort` - Port for health check endpoint (default: 8888)

**Example:**

```go
handlerSet := router.NewHandlerSet("my-controller", scheme, backend)
r := router.New(handlerSet, electionConfig, 8888)
```

#### Methods

##### Start

```go
func (r *Router) Start(ctx context.Context) error
```

Starts the controller and begins watching resources. This method blocks until the context is cancelled or an error occurs.

**Parameters:**

- `ctx` - Context for lifecycle management

**Returns:**

- `error` - Error if startup fails

**Example:**

```go
ctx := context.Background()
if err := r.Start(ctx); err != nil {
    log.Fatal(err)
}
```

##### Backend

```go
func (r *Router) Backend() backend.Backend
```

Returns the Backend interface for Kubernetes operations.

**Returns:**

- `backend.Backend` - Backend for client operations

##### Stopped

```go
func (r *Router) Stopped() <-chan struct{}
```

Returns a channel that is closed when the router has stopped.

**Returns:**

- `<-chan struct{}` - Channel closed on stop

**Example:**

```go
<-r.Stopped()
fmt.Println("Router stopped")
```

##### Handle

```go
func (r *Router) Handle(objType client.Object, h Handler)
```

Registers a handler for a specific object type without filters.

**Parameters:**

- `objType` - Kubernetes object type (e.g., `&corev1.Pod{}`)
- `h` - Handler implementation

##### HandleFunc

```go
func (r *Router) HandleFunc(objType client.Object, h HandlerFunc)
```

Registers a handler function for a specific object type without filters.

**Parameters:**

- `objType` - Kubernetes object type
- `h` - Handler function

##### PosStart

```go
func (r *Router) PosStart(f func(context.Context, client.Client))
```

Registers a function to be called after the controller starts.

**Parameters:**

- `f` - Function to call post-start

##### DumpTriggers

```go
func (r *Router) DumpTriggers(indent bool) ([]byte, error)
```

Returns a JSON representation of active triggers for debugging.

**Parameters:**

- `indent` - Whether to indent the JSON output

**Returns:**

- `[]byte` - JSON-encoded trigger data
- `error` - Error if serialization fails

---

### RouteBuilder

The `RouteBuilder` provides a fluent API for configuring resource routes with filters, middleware, and handlers.

```go
type RouteBuilder struct {
    includeRemove     bool
    includeFinalizing bool
    finalizeID        string
    router            *Router
    objType           client.Object
    name              string
    namespace         string
    routeName         string
    middleware        []Middleware
    sel               labels.Selector
    fieldSelector     fields.Selector
}
```

#### Methods

##### Type

```go
func (r RouteBuilder) Type(objType client.Object) RouteBuilder
```

Specifies the Kubernetes object type to watch.

**Parameters:**

- `objType` - Object type (e.g., `&corev1.ConfigMap{}`)

**Returns:**

- `RouteBuilder` - Builder for chaining

**Example:**

```go
r.Type(&corev1.Pod{}).HandlerFunc(handler)
```

##### Namespace

```go
func (r RouteBuilder) Namespace(namespace string) RouteBuilder
```

Filters resources to a specific namespace.

**Parameters:**

- `namespace` - Namespace name

**Returns:**

- `RouteBuilder` - Builder for chaining

**Example:**

```go
r.Type(&corev1.Pod{}).
    Namespace("default").
    HandlerFunc(handler)
```

##### Name

```go
func (r RouteBuilder) Name(name string) RouteBuilder
```

Filters resources to a specific name.

**Parameters:**

- `name` - Resource name

**Returns:**

- `RouteBuilder` - Builder for chaining

##### Selector

```go
func (r RouteBuilder) Selector(sel labels.Selector) RouteBuilder
```

Filters resources by label selector.

**Parameters:**

- `sel` - Label selector

**Returns:**

- `RouteBuilder` - Builder for chaining

**Example:**

```go
r.Type(&corev1.Pod{}).
    Selector(labels.Set{"app": "my-app"}).
    HandlerFunc(handler)
```

##### FieldSelector

```go
func (r RouteBuilder) FieldSelector(sel fields.Selector) RouteBuilder
```

Filters resources by field selector.

**Parameters:**

- `sel` - Field selector

**Returns:**

- `RouteBuilder` - Builder for chaining

**Example:**

```go
r.Type(&corev1.Pod{}).
    FieldSelector(fields.OneTermEqualSelector("spec.nodeName", "node-1")).
    HandlerFunc(handler)
```

##### Middleware

```go
func (r RouteBuilder) Middleware(m ...Middleware) RouteBuilder
```

Adds middleware to the handler chain.

**Parameters:**

- `m` - One or more middleware instances

**Returns:**

- `RouteBuilder` - Builder for chaining

**Example:**

```go
r.Type(&corev1.Pod{}).
    Middleware(ErrorPrefix("pod-handler")).
    HandlerFunc(handler)
```

##### IncludeRemoved

```go
func (r RouteBuilder) IncludeRemoved() RouteBuilder
```

Includes resources marked for deletion (with deletion timestamp).

**Returns:**

- `RouteBuilder` - Builder for chaining

##### IncludeFinalizing

```go
func (r RouteBuilder) IncludeFinalizing() RouteBuilder
```

Includes resources in the finalizing state.

**Returns:**

- `RouteBuilder` - Builder for chaining

##### Handler

```go
func (r RouteBuilder) Handler(h Handler)
```

Registers a Handler implementation.

**Parameters:**

- `h` - Handler instance

##### HandlerFunc

```go
func (r RouteBuilder) HandlerFunc(h HandlerFunc)
```

Registers a handler function.

**Parameters:**

- `h` - Handler function

**Example:**

```go
r.Type(&corev1.ConfigMap{}).
    Namespace("default").
    HandlerFunc(func(req Request, resp Response) error {
        cm := req.Object.(*corev1.ConfigMap)
        fmt.Printf("Processing: %s/%s\n", cm.Namespace, cm.Name)
        return nil
    })
```

##### Finalize

```go
func (r RouteBuilder) Finalize(finalizerID string, h Handler)
```

Registers a finalizer handler for cleanup on deletion.

**Parameters:**

- `finalizerID` - Unique finalizer identifier
- `h` - Handler for finalization

##### FinalizeFunc

```go
func (r RouteBuilder) FinalizeFunc(finalizerID string, h HandlerFunc)
```

Registers a finalizer handler function.

**Parameters:**

- `finalizerID` - Unique finalizer identifier
- `h` - Handler function for finalization

**Example:**

```go
r.Type(&MyResource{}).
    FinalizeFunc("my-finalizer", func(req Request, resp Response) error {
        // Cleanup logic
        return nil
    })
```

---

### Handler

Interface for processing Kubernetes resources.

```go
type Handler interface {
    Handle(req Request, resp Response) error
}
```

#### HandlerFunc

Function type that implements the Handler interface.

```go
type HandlerFunc func(Request, Response) error

func (h HandlerFunc) Handle(req Request, resp Response) error {
    return h(req, resp)
}
```

---

### Request

Contains information about the resource being processed.

```go
type Request struct {
    Object         client.Object
    Ctx            context.Context
    FromController bool
}
```

**Fields:**

- `Object` - The Kubernetes object being processed
- `Ctx` - Request context
- `FromController` - Whether event originated from controller

---

### Response

Provides methods for interacting with Kubernetes during request handling.

```go
type Response interface {
    Backend() backend.Backend
    Objects() *apply.ObjectSet
}
```

**Methods:**

- `Backend()` - Returns backend for Kubernetes operations
- `Objects()` - Returns object set for applying resources

---

## Middleware

Middleware intercepts and processes requests before they reach handlers.

### Built-in Middleware

#### ErrorPrefix

Prefixes handler errors with context.

```go
type ErrorPrefix string

func (e ErrorPrefix) Handle(req Request, resp Response, next Handler) error
```

**Example:**

```go
r.Type(&corev1.Pod{}).
    Middleware(ErrorPrefix("pod-controller")).
    HandlerFunc(handler)
```

#### IgnoreNilHandler

Skips handler if object is nil.

```go
type IgnoreNilHandler struct{}

func (i IgnoreNilHandler) Handle(req Request, resp Response, next Handler) error
```

#### IgnoreRemoveHandler

Skips handler for resources marked for deletion.

```go
type IgnoreRemoveHandler struct{}

func (i IgnoreRemoveHandler) Handle(req Request, resp Response, next Handler) error
```

### Custom Middleware

Implement the `Middleware` interface:

```go
type Middleware interface {
    Handle(req Request, resp Response, next Handler) error
}

type LoggingMiddleware struct{}

func (m LoggingMiddleware) Handle(req Request, resp Response, next Handler) error {
    log.Printf("Processing: %s", req.Object.GetName())
    err := next.Handle(req, resp)
    if err != nil {
        log.Printf("Error: %v", err)
    }
    return err
}
```

---

## Filters

Filters select which resources trigger handlers.

### Built-in Filters

#### NameNamespaceFilter

Filters by exact name and namespace match.

```go
type NameNamespaceFilter struct {
    Name      string
    Namespace string
}

func (n NameNamespaceFilter) Handle(req Request, resp Response, next Handler) error
```

#### SelectorFilter

Filters by label selector.

```go
type SelectorFilter struct {
    Selector labels.Selector
}

func (s SelectorFilter) Handle(req Request, resp Response, next Handler) error
```

#### FieldSelectorFilter

Filters by field selector.

```go
type FieldSelectorFilter struct {
    FieldSelector fields.Selector
}

func (f FieldSelectorFilter) Handle(req Request, resp Response, next Handler) error
```

---

## Finalizers

Finalizers enable cleanup logic when resources are deleted.

### Usage

```go
r.Type(&MyResource{}).
    FinalizeFunc("my-finalizer", func(req Request, resp Response) error {
        obj := req.Object.(*MyResource)

        // Cleanup external resources
        if err := cleanupExternal(obj); err != nil {
            return err
        }

        // Finalizer will be removed automatically
        return nil
    })
```

**Important:**

- Finalizer is added automatically on first reconciliation
- Finalizer is removed automatically after successful cleanup
- Handler is called when resource has deletion timestamp

---

## Examples

### Complete Controller Example

```go
package main

import (
    "context"
    "fmt"

    "github.com/obot-platform/nah"
    "github.com/obot-platform/nah/pkg/router"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/labels"
    "k8s.io/apimachinery/pkg/runtime"
)

func main() {
    scheme := runtime.NewScheme()
    corev1.AddToScheme(scheme)

    r, err := nah.DefaultRouter("my-controller", scheme)
    if err != nil {
        panic(err)
    }

    // Simple handler
    r.Type(&corev1.ConfigMap{}).
        Namespace("default").
        HandlerFunc(handleConfigMap)

    // Handler with middleware and filters
    r.Type(&corev1.Pod{}).
        Namespace("kube-system").
        Selector(labels.Set{"app": "my-app"}).
        Middleware(router.ErrorPrefix("pod-handler")).
        HandlerFunc(handlePod)

    // Handler with finalizer
    r.Type(&corev1.Service{}).
        FinalizeFunc("service-finalizer", cleanupService)

    if err := r.Start(context.Background()); err != nil {
        panic(err)
    }
}

func handleConfigMap(req router.Request, resp router.Response) error {
    cm := req.Object.(*corev1.ConfigMap)
    fmt.Printf("ConfigMap: %s/%s\n", cm.Namespace, cm.Name)
    return nil
}

func handlePod(req router.Request, resp router.Response) error {
    pod := req.Object.(*corev1.Pod)
    fmt.Printf("Pod: %s/%s Phase: %s\n",
        pod.Namespace, pod.Name, pod.Status.Phase)
    return nil
}

func cleanupService(req router.Request, resp router.Response) error {
    svc := req.Object.(*corev1.Service)
    fmt.Printf("Cleaning up service: %s/%s\n", svc.Namespace, svc.Name)
    // Cleanup logic here
    return nil
}
```

---

## See Also

- [Backend Package](backend.md) - Kubernetes client operations
- [Apply Package](apply.md) - Declarative resource management
- [Leader Package](leader.md) - Leader election
- [Main Package API](../../router.go) - Top-level router creation

---

**Package Documentation:** [pkg.go.dev/github.com/obot-platform/nah/pkg/router](https://pkg.go.dev/github.com/obot-platform/nah/pkg/router)
