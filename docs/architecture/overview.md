# Architecture Overview

This document provides a comprehensive overview of the nah framework architecture, including component relationships, data flow, and design decisions.

## Table of Contents

- [High-Level Architecture](#high-level-architecture)
- [Core Components](#core-components)
- [Data Flow](#data-flow)
- [Component Interactions](#component-interactions)
- [Design Principles](#design-principles)

---

## High-Level Architecture

```mermaid
graph TB
    subgraph "Controller Application"
        User[User Code]
        Router[Router]
        Handlers[Handlers]
    end

    subgraph "nah Framework"
        RouteBuilder[Route Builder]
        Backend[Backend]
        Apply[Apply]
        Leader[Leader Election]
        Watcher[Watcher]
    end

    subgraph "Kubernetes"
        APIServer[API Server]
        Informers[Informers/Cache]
    end

    User -->|Configure| Router
    Router -->|Build Routes| RouteBuilder
    Router -->|Start| Leader
    Leader -->|Elected| Watcher
    Watcher -->|Watch| Informers
    Informers -->|Sync| APIServer
    Watcher -->|Trigger| Handlers
    Handlers -->|Use| Backend
    Handlers -->|Apply State| Apply
    Backend -->|Read| Informers
    Backend -->|Write| APIServer
    Apply -->|CRUD| Backend
```

**Key Flow:**

1. User configures Router with routes and handlers
2. Router integrates with Leader Election
3. Watcher subscribes to Kubernetes resources via Informers
4. Events trigger appropriate Handlers
5. Handlers use Backend and Apply to reconcile state

---

## Core Components

### Router

The Router is the central orchestrator that:

- Manages controller lifecycle (start, stop)
- Routes Kubernetes events to handlers
- Integrates leader election
- Provides health checks

```mermaid
classDiagram
    class Router {
        +RouteBuilder
        +Backend backend
        +ElectionConfig electionConfig
        +Start(ctx) error
        +Handle(objType, handler)
        +Stopped() channel
    }

    class RouteBuilder {
        +Type(objType) RouteBuilder
        +Namespace(ns) RouteBuilder
        +Selector(sel) RouteBuilder
        +Middleware(...) RouteBuilder
        +Handler(h)
    }

    class HandlerSet {
        -handlers map
        +Register(gvk, handler)
        +Get(gvk) Handler
    }

    Router --> RouteBuilder
    Router --> HandlerSet
    RouteBuilder --> Router
```

**Responsibilities:**

- Lifecycle management
- Event routing
- Handler registration
- Leader election integration

### Backend

Backend provides abstraction over Kubernetes operations:

```mermaid
classDiagram
    class Backend {
        <<interface>>
        +Get(ctx, key, obj) error
        +List(ctx, list, opts) error
        +Create(ctx, obj) error
        +Update(ctx, obj) error
        +Delete(ctx, obj) error
        +Trigger(ctx, gvk, key, delay) error
        +Watcher(ctx, gvk, name, cb) error
    }

    class Trigger {
        <<interface>>
        +Trigger(ctx, gvk, key, delay) error
    }

    class Watcher {
        <<interface>>
        +Watcher(ctx, gvk, name, cb) error
    }

    class CacheFactory {
        <<interface>>
        +GetInformerForKind(ctx, gvk) Informer
    }

    Backend --|> Trigger
    Backend --|> Watcher
    Backend --|> CacheFactory
    Backend --|> "client.WithWatch"
```

**Responsibilities:**

- CRUD operations
- Resource watching
- Cache management
- Trigger management

### Apply

Apply provides declarative resource management:

```mermaid
classDiagram
    class Apply {
        <<interface>>
        +Apply(ctx, owner, objs) error
        +Ensure(ctx, objs) error
        +WithOwnerSubContext(sub) Apply
        +WithNoPrune() Apply
        +FindOwner(ctx, obj) Object
    }

    class DesiredSet {
        -owner Object
        -desired []Object
        -subcontext string
        +process() error
        +compare() Changes
        +apply() error
    }

    class ObjectSet {
        -objects map
        +Add(obj)
        +Contains(key) bool
        +Remove(key)
    }

    Apply ..> DesiredSet
    DesiredSet --> ObjectSet
```

**Responsibilities:**

- Declarative reconciliation
- Owner reference management
- Resource pruning
- Three-way merge patches

---

## Data Flow

### Event Processing Flow

```mermaid
sequenceDiagram
    participant K8s as Kubernetes
    participant Informer
    participant Watcher
    participant Router
    participant Middleware
    participant Handler
    participant Backend
    participant Apply

    K8s->>Informer: Resource Change
    Informer->>Watcher: Event Notification
    Watcher->>Router: Trigger Handler
    Router->>Middleware: Pre-process Request
    Middleware->>Handler: Handle Request
    Handler->>Backend: Get Current State
    Backend->>Handler: Current Object
    Handler->>Handler: Compute Desired State
    Handler->>Apply: Apply Desired State
    Apply->>Backend: Create/Update/Delete
    Backend->>K8s: API Calls
    Handler->>Middleware: Return Result
    Middleware->>Router: Complete
```

**Steps:**

1. Kubernetes resource changes
2. Informer detects change
3. Watcher triggers appropriate handler
4. Middleware pre-processes request
5. Handler computes desired state
6. Apply reconciles to desired state
7. Backend performs Kubernetes operations

### Startup Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Router
    participant Leader as Leader Election
    participant Backend
    participant Watcher
    participant Handlers

    App->>Router: Configure Routes
    App->>Router: Start(ctx)
    Router->>Backend: Preload Caches
    Router->>Leader: Run Election
    Leader->>Leader: Acquire Lock
    Leader-->>Router: Elected
    Router->>Handlers: Start All Handlers
    Handlers->>Backend: Register Watchers
    Backend->>Watcher: Start Watching
    Router-->>App: Running
```

**Steps:**

1. Application configures routes
2. Router starts
3. Backend preloads caches
4. Leader election runs
5. After election, handlers start
6. Watchers begin monitoring resources

---

## Component Interactions

### Handler Request/Response Pattern

```mermaid
graph LR
    subgraph "Handler Context"
        Request[Request<br/>- Object<br/>- Context<br/>- FromController]
        Response[Response<br/>- Backend<br/>- Objects]
    end

    subgraph "Handler"
        Logic[Handler Logic]
    end

    subgraph "Operations"
        Backend[Backend<br/>CRUD Ops]
        Apply[Apply<br/>Declarative]
    end

    Request -->|Input| Logic
    Logic -->|Use| Response
    Response -->|Access| Backend
    Response -->|Access| Apply
```

**Pattern:**

- Request contains resource being processed
- Response provides access to Backend and Apply
- Handler implements business logic
- Middleware wraps handlers

### Apply Workflow

```mermaid
graph TB
    Start[Apply Called] --> GetCurrent[Get Current Resources]
    GetCurrent --> Compare[Compare Desired vs Current]
    Compare --> Create[Create Missing]
    Compare --> Update[Update Changed]
    Compare --> Delete[Delete Removed]
    Create --> SetOwner[Set Owner References]
    Update --> ThreeWay[Three-Way Merge]
    Delete --> CheckPrune{Prune Enabled?}
    CheckPrune -->|Yes| DeleteRes[Delete Resource]
    CheckPrune -->|No| Skip[Skip]
    SetOwner --> Done[Complete]
    ThreeWay --> Done
    DeleteRes --> Done
    Skip --> Done
```

**Workflow:**

1. Retrieve current resources with owner reference
2. Compare desired vs current state
3. Create missing resources
4. Update changed resources (three-way merge)
5. Delete removed resources (if pruning enabled)
6. Set owner references on all managed resources

---

## Design Principles

### 1. Interface-Based Design

Small, focused interfaces enable:

- Easy testing (mocking)
- Flexible implementations
- Clear contracts

```go
type Backend interface {
    // Focused interface
}

type Apply interface {
    // Single responsibility
}
```

### 2. Builder Pattern

Fluent configuration APIs provide:

- Readable code
- Type safety
- Chaining

```go
r.Type(&Pod{}).
    Namespace("default").
    Selector(labels.Set{"app": "myapp"}).
    HandlerFunc(handler)
```

### 3. Declarative Reconciliation

Following Kubernetes principles:

- Declare desired state
- Framework reconciles to desired state
- Idempotent operations
- Level-triggered (not edge-triggered)

### 4. Middleware Pattern

Cross-cutting concerns as middleware:

- Error handling
- Logging
- Metrics
- Authorization

```go
router.Middleware(
    ErrorPrefix("handler"),
    LoggingMiddleware{},
    MetricsMiddleware{},
)
```

### 5. Separation of Concerns

- **Router**: Event routing and lifecycle
- **Backend**: Kubernetes operations
- **Apply**: Declarative state management
- **Leader**: High availability
- **Watcher**: Resource monitoring

Each component has a single, well-defined responsibility.

---

## Layered Architecture

```mermaid
graph TB
    subgraph "Application Layer"
        App[Your Controller]
        Handlers[Your Handlers]
    end

    subgraph "Framework Layer"
        Router[Router<br/>Routing & Lifecycle]
        RouteBuilder[Route Builder<br/>Configuration]
    end

    subgraph "Abstraction Layer"
        Backend[Backend<br/>K8s Operations]
        Apply[Apply<br/>Declarative Mgmt]
        Leader[Leader Election<br/>HA]
    end

    subgraph "Platform Layer"
        Runtime[Runtime<br/>Controller Runtime]
        ClientGo[client-go<br/>K8s Client]
    end

    App --> Router
    Handlers --> Backend
    Handlers --> Apply
    Router --> RouteBuilder
    Router --> Leader
    Backend --> Runtime
    Apply --> Backend
    Runtime --> ClientGo
```

**Layers:**

1. **Application** - Your controller code
2. **Framework** - nah router and builders
3. **Abstraction** - Backend, Apply, Leader
4. **Platform** - controller-runtime, client-go

---

## Performance Optimizations

### 1. On-Demand Workers

Workers spawn only when needed:

- Reduces resource usage
- Scales automatically
- Efficient trigger handling

### 2. Sync.Map for Triggers

Low-contention trigger management:

- Read-heavy workload optimization
- Concurrent access without locks
- Memory-efficient

### 3. Informer Caching

Kubernetes informers provide:

- Local cache for reads
- Watch-based updates
- Reduced API server load

### 4. GVK-Specific Configuration

Per-resource tuning:

- Custom threadiness per GVK
- Queue splitting for high-throughput
- Flexible worker management

---

## Concurrency Model

```mermaid
graph TB
    Main[Main Goroutine] --> Router
    Router --> LeaderElection[Leader Election<br/>Goroutine]
    Router --> HealthCheck[Health Check<br/>Server Goroutine]
    Router --> Handlers[Handler Goroutines<br/>Per GVK]
    Handlers --> Workers[Worker Pool<br/>On-Demand]
    Workers --> Handler1[Handler<br/>Instance]
    Workers --> Handler2[Handler<br/>Instance]
    Workers --> HandlerN[Handler<br/>Instance...]
```

**Goroutine Usage:**

- Main goroutine manages lifecycle
- Leader election runs in separate goroutine
- Health check server runs in separate goroutine
- Each GVK has worker goroutines (configurable count)
- Workers process events concurrently

---

## See Also

- [Design Patterns](patterns.md) - Detailed pattern explanations
- [Component Deep Dive](components.md) - Per-component documentation
- [ADRs](adr/) - Architecture Decision Records

---

**For Questions:** [GitHub Discussions](https://github.com/obot-platform/nah/discussions)
