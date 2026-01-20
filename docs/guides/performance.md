# Performance Optimization Guide

> **Version**: 1.0.0 | **Created**: 2026-01-19 | **Status**: Production Ready

This guide covers performance optimization strategies for nah-based Kubernetes controllers, from caching and resource management to profiling and monitoring.

## Table of Contents

1. [Overview](#overview)
2. [Performance Fundamentals](#performance-fundamentals)
3. [Caching Strategies](#caching-strategies)
4. [Worker and Threadiness Tuning](#worker-and-threadiness-tuning)
5. [Resource Management](#resource-management)
6. [Profiling and Monitoring](#profiling-and-monitoring)
7. [OpenTelemetry Integration](#opentelemetry-integration)
8. [Memory Optimization](#memory-optimization)
9. [Throughput Optimization](#throughput-optimization)
10. [Best Practices](#best-practices)
11. [Troubleshooting](#troubleshooting)

---

## Overview

### Why Performance Matters

Kubernetes controllers are often the bottleneck in cloud-native systems. Poor controller performance leads to:

- **Slow reconciliation**: Resources take longer to reach desired state
- **Resource contention**: High CPU/memory usage impacts cluster stability
- **Cascading failures**: Delays in one controller affect dependent resources
- **Poor user experience**: Application deployments and updates lag

### Performance Goals

| Metric | Target | Critical |
|--------|--------|----------|
| **Reconciliation latency** | < 1s for simple resources | < 5s |
| **Event processing rate** | > 100 events/second | > 10 events/second |
| **Memory usage** | < 500MB for typical controller | < 2GB |
| **CPU usage** | < 1 core steady state | < 4 cores peak |
| **Queue depth** | < 100 items | < 1000 items |

### Key Optimization Areas

```
┌─────────────────────────────────────────────────────┐
│           Controller Performance Stack              │
├─────────────────────────────────────────────────────┤
│  Handler Logic          │ Algorithm efficiency     │
│                         │ Business logic           │
├─────────────────────────────────────────────────────┤
│  Router Configuration   │ Workers, threadiness     │
│                         │ GVK-specific tuning      │
├─────────────────────────────────────────────────────┤
│  Caching                │ Informer caching         │
│                         │ Custom caches            │
├─────────────────────────────────────────────────────┤
│  API Calls              │ Batching, retries        │
│                         │ Rate limiting            │
├─────────────────────────────────────────────────────┤
│  Resource Management    │ Memory, goroutines       │
│                         │ Connection pooling       │
└─────────────────────────────────────────────────────┘
```

---

## Performance Fundamentals

### Understanding the Event Loop

nah controllers process events through a work queue:

```
1. Informer watches API server
   ↓
2. Events added to work queue
   ↓
3. Workers pull from queue (parallel)
   ↓
4. Handler processes event
   ↓
5. Success: Remove from queue
   Failure: Requeue with backoff
```

### Performance Bottlenecks

| Bottleneck | Symptoms | Impact |
|------------|----------|--------|
| **Slow handlers** | High processing time, low throughput | Queue backlog grows |
| **Too few workers** | Low CPU usage, high queue depth | Underutilized resources |
| **Too many workers** | High CPU, context switching overhead | Inefficient processing |
| **API rate limits** | 429 errors, retries | Cascading delays |
| **Memory leaks** | Growing RSS, OOM kills | Controller crashes |
| **Inefficient caching** | Excessive API calls | API server overload |

### Measuring Performance

```go
import (
    "time"
    "log/slog"
)

func reconcileWithMetrics(obj *corev1.Pod, status corev1.PodStatus) error {
    start := time.Now()
    defer func() {
        duration := time.Since(start)
        slog.Info("reconciliation complete",
            "namespace", obj.Namespace,
            "name", obj.Name,
            "duration_ms", duration.Milliseconds(),
        )
    }()

    return reconcilePod(obj)
}
```

---

## Caching Strategies

### Informer Caching

nah automatically caches watched resources using Kubernetes informers:

```go
import "github.com/obot-platform/nah/pkg/router"

r := router.New(ctx, router.Config{
    Namespace: "default",

    // Informer sync period (default: 10 minutes)
    ResyncPeriod: 5 * time.Minute,
})

// Access cached objects (no API call)
pod, err := r.Get(ctx, "mypod", &corev1.Pod{})
```

### Custom Caching

For frequently accessed data not in informer cache:

```go
import (
    "sync"
    "time"
)

// TTL cache for expensive computations
type Cache struct {
    mu      sync.RWMutex
    data    map[string]*CacheEntry
    ttl     time.Duration
}

type CacheEntry struct {
    value      interface{}
    expiration time.Time
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    entry, exists := c.data[key]
    if !exists || time.Now().After(entry.expiration) {
        return nil, false
    }

    return entry.value, true
}

func (c *Cache) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.data[key] = &CacheEntry{
        value:      value,
        expiration: time.Now().Add(c.ttl),
    }
}

// Use in handler
var configCache = &Cache{
    data: make(map[string]*CacheEntry),
    ttl:  5 * time.Minute,
}

func reconcileDeployment(obj *appsv1.Deployment, status appsv1.DeploymentStatus) error {
    // Check cache first
    if config, ok := configCache.Get(obj.Name); ok {
        return applyConfig(obj, config)
    }

    // Expensive computation
    config := computeConfiguration(obj)
    configCache.Set(obj.Name, config)

    return applyConfig(obj, config)
}
```

### Cache Invalidation

Invalidate cache when watched resources change:

```go
// Watch ConfigMaps and invalidate cache on change
router.Type(&corev1.ConfigMap{}).
    HandlerFunc(func(obj *corev1.ConfigMap, status corev1.ConfigMapStatus) error {
        // Invalidate related cache entries
        configCache.Delete(obj.Name)
        return nil
    }).
    Register(r)
```

### Caching Best Practices

| Pattern | Use Case | TTL |
|---------|----------|-----|
| **Informer cache** | Kubernetes resources | Controller-managed |
| **Computed results** | Expensive transformations | 5-10 minutes |
| **External API data** | Third-party services | 1-5 minutes |
| **Validation results** | Schema checks | 15-30 minutes |
| **Configuration** | Rarely-changing settings | 1 hour |

---

## Worker and Threadiness Tuning

### Understanding Workers vs. Threadiness

- **Workers**: Number of goroutines pulling from work queue
- **Threadiness**: Concurrent handlers per worker

```go
r := router.New(ctx, router.Config{
    // Global defaults
    Workers:     2,  // 2 goroutines pulling from queue
    Threadiness: 5,  // Each worker handles 5 resources concurrently
    // Total concurrency: 2 * 5 = 10 concurrent handlers
})
```

### Tuning Guidelines

#### High-Volume Resources (Pods, Events)

```go
GVKConfig: map[schema.GroupVersionKind]router.GVKConfig{
    {Group: "", Version: "v1", Kind: "Pod"}: {
        Workers:     4,   // More workers
        Threadiness: 10,  // Higher concurrency
    },
}

// Rationale: Maximize throughput for frequent events
// Tradeoff: Higher CPU usage, more memory
```

#### Low-Volume Resources (Namespaces, CRDs)

```go
{Group: "", Version: "v1", Kind: "Namespace"}: {
    Workers:     1,  // Single worker sufficient
    Threadiness: 2,  // Low concurrency
}

// Rationale: Minimize overhead for rare events
// Tradeoff: Slightly higher latency acceptable
```

#### I/O-Bound Handlers (External APIs, Database)

```go
{Group: "", Version: "v1", Kind: "Service"}: {
    Workers:     2,   // Moderate workers
    Threadiness: 20,  // High concurrency (waiting on I/O)
}

// Rationale: High concurrency keeps CPU busy during I/O waits
// Tradeoff: More goroutines, watch for connection limits
```

#### CPU-Bound Handlers (Complex computations)

```go
{Group: "apps", Version: "v1", Kind: "Deployment"}: {
    Workers:     2,  // Moderate workers
    Threadiness: 4,  // Lower concurrency (avoid CPU contention)
}

// Rationale: Match concurrency to available CPU cores
// Tradeoff: Lower throughput, more predictable latency
```

### Dynamic Tuning

Monitor metrics and adjust:

```go
import "runtime"

// Adjust based on available CPUs
numCPU := runtime.NumCPU()

r := router.New(ctx, router.Config{
    // Scale with CPU count
    Workers:     max(2, numCPU/2),
    Threadiness: max(4, numCPU),
})
```

### Tuning Decision Matrix

| Handler Profile | Event Rate | Workers | Threadiness | Example |
|----------------|------------|---------|-------------|---------|
| **Fast + High volume** | > 100/s | 4-8 | 10-20 | Pod events |
| **Fast + Low volume** | < 10/s | 1-2 | 2-5 | Namespace |
| **Slow I/O + High volume** | > 50/s | 2-4 | 15-30 | API calls |
| **Slow CPU + Low volume** | < 20/s | 2-3 | 3-6 | Complex logic |

---

## Resource Management

### Memory Optimization

#### Avoid Object Copying

```go
// ❌ Bad: Creates copy of large object
func handlePod(obj *corev1.Pod, status corev1.PodStatus) error {
    podCopy := *obj  // Unnecessary copy
    return processPod(&podCopy)
}

// ✅ Good: Use pointer directly
func handlePod(obj *corev1.Pod, status corev1.PodStatus) error {
    return processPod(obj)
}
```

#### Limit List Operations

```go
// ❌ Bad: Lists all pods (unbounded memory)
func getPods() ([]*corev1.Pod, error) {
    return client.CoreV1().Pods("").List(ctx, metav1.ListOptions{})
}

// ✅ Good: Use label selector and pagination
func getPods() ([]*corev1.Pod, error) {
    return client.CoreV1().Pods("default").List(ctx, metav1.ListOptions{
        LabelSelector: "app=myapp",
        Limit:         100,
    })
}
```

#### Release Resources Promptly

```go
func reconcileDeployment(obj *appsv1.Deployment, status appsv1.DeploymentStatus) error {
    // Load large config
    config, err := loadConfiguration(obj)
    if err != nil {
        return err
    }

    // Process immediately
    result := processConfig(config)

    // Release reference (eligible for GC)
    config = nil

    return applyResult(obj, result)
}
```

### Goroutine Management

#### Bounded Goroutines

```go
// ❌ Bad: Unbounded goroutines
func handlePod(obj *corev1.Pod, status corev1.PodStatus) error {
    for _, container := range obj.Spec.Containers {
        go processContainer(container)  // Goroutine leak risk
    }
    return nil
}

// ✅ Good: Use worker pool
func handlePod(obj *corev1.Pod, status corev1.PodStatus) error {
    sem := make(chan struct{}, 5)  // Max 5 concurrent
    errCh := make(chan error, len(obj.Spec.Containers))

    for _, container := range obj.Spec.Containers {
        sem <- struct{}{}
        go func(c corev1.Container) {
            defer func() { <-sem }()
            errCh <- processContainer(c)
        }(container)
    }

    // Wait for completion
    for range obj.Spec.Containers {
        if err := <-errCh; err != nil {
            return err
        }
    }

    return nil
}
```

#### Context Cancellation

```go
func handleWithContext(obj *corev1.Pod, status corev1.PodStatus) error {
    // Create cancellable context
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // Pass to long-running operations
    return processWithContext(ctx, obj)
}

func processWithContext(ctx context.Context, obj *corev1.Pod) error {
    select {
    case <-ctx.Done():
        return fmt.Errorf("operation cancelled: %w", ctx.Err())
    case result := <-doWork(obj):
        return handleResult(result)
    }
}
```

### Connection Pooling

```go
import "net/http"

// Reuse HTTP client with connection pooling
var httpClient = &http.Client{
    Timeout: 30 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
    },
}

func callExternalAPI(url string) error {
    resp, err := httpClient.Get(url)  // Reuses connections
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    return processResponse(resp)
}
```

---

## Profiling and Monitoring

### CPU Profiling

```go
import (
    "net/http"
    _ "net/http/pprof"
)

func main() {
    // Enable pprof endpoint
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()

    // Start controller
    startController()
}
```

Access profiles:
```bash
# CPU profile (30 seconds)
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Analyze in browser
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/profile
```

### Memory Profiling

```bash
# Heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Allocation profile
go tool pprof http://localhost:6060/debug/pprof/allocs

# Goroutine profile
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

### Metrics Exposition

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    reconcileLatency = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "controller_reconcile_duration_seconds",
            Help:    "Time spent in reconciliation handler",
            Buckets: prometheus.DefBuckets,
        },
        []string{"resource", "namespace", "name"},
    )

    reconcileErrors = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "controller_reconcile_errors_total",
            Help: "Total number of reconciliation errors",
        },
        []string{"resource", "error_type"},
    )
)

func init() {
    prometheus.MustRegister(reconcileLatency)
    prometheus.MustRegister(reconcileErrors)
}

func reconcileWithMetrics(obj *corev1.Pod, status corev1.PodStatus) error {
    timer := prometheus.NewTimer(reconcileLatency.WithLabelValues(
        "Pod", obj.Namespace, obj.Name,
    ))
    defer timer.ObserveDuration()

    if err := reconcilePod(obj); err != nil {
        reconcileErrors.WithLabelValues("Pod", errorType(err)).Inc()
        return err
    }

    return nil
}

// Expose metrics endpoint
func main() {
    http.Handle("/metrics", promhttp.Handler())
    go http.ListenAndServe(":9090", nil)

    startController()
}
```

### Key Metrics to Monitor

| Metric | Type | Alert Threshold | Meaning |
|--------|------|----------------|---------|
| `workqueue_depth` | Gauge | > 1000 | Queue backlog |
| `workqueue_adds_total` | Counter | - | Event rate |
| `reconcile_duration_seconds` | Histogram | p99 > 10s | Slow handlers |
| `reconcile_errors_total` | Counter | Rate > 10/min | Handler failures |
| `go_goroutines` | Gauge | > 10000 | Goroutine leak |
| `go_memstats_alloc_bytes` | Gauge | > 2GB | Memory leak |

---

## OpenTelemetry Integration

### Basic Setup

```go
import (
    "github.com/obot-platform/nah/pkg/router"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/trace"
)

func setupTracing(ctx context.Context) error {
    // Configure OTLP exporter
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("localhost:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return err
    }

    // Create trace provider
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithSampler(trace.TraceIDRatioBased(0.1)), // 10% sampling
    )
    otel.SetTracerProvider(tp)

    return nil
}

func main() {
    ctx := context.Background()

    // Setup tracing
    if err := setupTracing(ctx); err != nil {
        panic(err)
    }

    // Enable tracing in router
    r := router.New(ctx, router.Config{
        EnableTrace: true,
        TracingConfig: router.TracingConfig{
            ServiceName:  "mycontroller",
            SamplingRate: 0.1,
        },
    })

    startController(r)
}
```

### Custom Spans

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
)

func reconcileDeployment(obj *appsv1.Deployment, status appsv1.DeploymentStatus) error {
    tracer := otel.Tracer("mycontroller")

    ctx, span := tracer.Start(ctx, "reconcileDeployment",
        trace.WithAttributes(
            attribute.String("namespace", obj.Namespace),
            attribute.String("name", obj.Name),
            attribute.Int64("generation", obj.Generation),
        ),
    )
    defer span.End()

    // Nested span for expensive operation
    _, subSpan := tracer.Start(ctx, "computeConfiguration")
    config := computeConfiguration(obj)
    subSpan.End()

    return applyConfiguration(ctx, obj, config)
}
```

### Trace Analysis

Use Jaeger or similar to analyze traces:

```bash
# Run Jaeger locally
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 4317:4317 \
  jaegertracing/all-in-one:latest

# Access UI
open http://localhost:16686
```

Look for:
- Long-running spans (slow operations)
- High span frequency (excessive calls)
- Error spans (failure patterns)

---

## Memory Optimization

### Identifying Memory Leaks

```go
import "runtime"

func printMemStats() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    log.Printf("Alloc = %v MiB", m.Alloc/1024/1024)
    log.Printf("TotalAlloc = %v MiB", m.TotalAlloc/1024/1024)
    log.Printf("Sys = %v MiB", m.Sys/1024/1024)
    log.Printf("NumGC = %v", m.NumGC)
}

// Run periodically
go func() {
    ticker := time.NewTicker(1 * time.Minute)
    for range ticker.C {
        printMemStats()
    }
}()
```

### Common Memory Leaks

#### Global Map Growth

```go
// ❌ Bad: Global map grows unbounded
var cache = make(map[string]*corev1.Pod)

func handlePod(obj *corev1.Pod, status corev1.PodStatus) error {
    cache[obj.Name] = obj  // Never removed!
    return nil
}

// ✅ Good: Implement eviction
type LRUCache struct {
    capacity int
    items    map[string]*list.Element
    order    *list.List
}

func (c *LRUCache) Add(key string, value interface{}) {
    // Evict oldest if at capacity
    if c.order.Len() >= c.capacity {
        oldest := c.order.Back()
        c.order.Remove(oldest)
        delete(c.items, oldest.Value.(string))
    }

    c.items[key] = c.order.PushFront(value)
}
```

#### Goroutine Leaks

```go
// ❌ Bad: Goroutine never exits
func handlePod(obj *corev1.Pod, status corev1.PodStatus) error {
    go func() {
        for {
            checkHealth(obj)
            time.Sleep(10 * time.Second)
        }
    }()  // Never stops!
    return nil
}

// ✅ Good: Respect context cancellation
func handlePod(ctx context.Context, obj *corev1.Pod, status corev1.PodStatus) error {
    go func() {
        ticker := time.NewTicker(10 * time.Second)
        defer ticker.Stop()

        for {
            select {
            case <-ctx.Done():
                return  // Exit when context cancelled
            case <-ticker.C:
                checkHealth(obj)
            }
        }
    }()
    return nil
}
```

### Garbage Collection Tuning

```go
import "runtime/debug"

func init() {
    // Reduce GC frequency for high-throughput controllers
    debug.SetGCPercent(200)  // Default: 100

    // Set memory limit (Go 1.19+)
    debug.SetMemoryLimit(1 * 1024 * 1024 * 1024)  // 1GB
}
```

---

## Throughput Optimization

### Batch Processing

```go
func handlePodBatch(pods []*corev1.Pod) error {
    // Process multiple pods in single handler call
    var errors []error

    for _, pod := range pods {
        if err := processPod(pod); err != nil {
            errors = append(errors, err)
        }
    }

    if len(errors) > 0 {
        return fmt.Errorf("batch errors: %v", errors)
    }

    return nil
}
```

### Parallel API Calls

```go
import "golang.org/x/sync/errgroup"

func reconcileDeployment(obj *appsv1.Deployment, status appsv1.DeploymentStatus) error {
    g, ctx := errgroup.WithContext(context.Background())

    // Create dependencies in parallel
    g.Go(func() error {
        return ensureServiceAccount(ctx, obj)
    })

    g.Go(func() error {
        return ensureConfigMap(ctx, obj)
    })

    g.Go(func() error {
        return ensureSecret(ctx, obj)
    })

    // Wait for all to complete
    return g.Wait()
}
```

### Request Coalescing

```go
type RequestCoalescer struct {
    mu       sync.Mutex
    pending  map[string]chan interface{}
}

func (rc *RequestCoalescer) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
    rc.mu.Lock()

    // Check if request already in flight
    if ch, exists := rc.pending[key]; exists {
        rc.mu.Unlock()
        return <-ch, nil  // Wait for existing request
    }

    // Create channel for this request
    ch := make(chan interface{}, 1)
    rc.pending[key] = ch
    rc.mu.Unlock()

    // Execute request
    result, err := fn()

    // Broadcast result to all waiters
    ch <- result
    close(ch)

    rc.mu.Lock()
    delete(rc.pending, key)
    rc.mu.Unlock()

    return result, err
}

// Use in handler
var coalescer = &RequestCoalescer{
    pending: make(map[string]chan interface{}),
}

func handlePod(obj *corev1.Pod, status corev1.PodStatus) error {
    // Multiple pods may trigger same config load
    config, err := coalescer.Do(obj.Name, func() (interface{}, error) {
        return loadExpensiveConfig(obj)
    })

    return applyConfig(obj, config)
}
```

---

## Best Practices

### 1. Profile Before Optimizing

```go
// Baseline performance before changes
go test -bench=. -benchmem -cpuprofile=cpu.prof -memprofile=mem.prof

// After optimization
go test -bench=. -benchmem -cpuprofile=cpu-new.prof -memprofile=mem-new.prof

// Compare
go tool pprof -base cpu.prof cpu-new.prof
```

### 2. Use Structured Logging with Levels

```go
import "log/slog"

// Set appropriate log level for production
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,  // Don't use Debug in production
}))

func reconcilePod(obj *corev1.Pod, status corev1.PodStatus) error {
    // Info for important events
    logger.Info("reconciling pod", "name", obj.Name)

    // Debug only when enabled (no overhead in production)
    logger.Debug("pod details", "spec", obj.Spec)

    return nil
}
```

### 3. Optimize Handler Fast Path

```go
func reconcileDeployment(obj *appsv1.Deployment, status appsv1.DeploymentStatus) error {
    // Fast path: check if already reconciled
    if obj.Status.ObservedGeneration == obj.Generation &&
       isConditionTrue(obj.Status, appsv1.DeploymentAvailable) {
        return nil  // No work needed
    }

    // Slow path: full reconciliation
    return fullReconcile(obj)
}
```

### 4. Avoid Unnecessary Status Updates

```go
func updateStatus(obj *appsv1.Deployment) error {
    // Check if status actually changed
    newStatus := computeStatus(obj)
    if reflect.DeepEqual(obj.Status, newStatus) {
        return nil  // Skip API call
    }

    obj.Status = newStatus
    return client.Status().Update(ctx, obj)
}
```

### 5. Use Informer Caching

```go
// ❌ Bad: API call on every reconcile
func getPod(name string) (*corev1.Pod, error) {
    return client.CoreV1().Pods("default").Get(ctx, name, metav1.GetOptions{})
}

// ✅ Good: Use informer cache
func getPod(r *router.Router, name string) (*corev1.Pod, error) {
    return r.Get(ctx, name, &corev1.Pod{})  // From cache
}
```

### 6. Set Resource Limits

```yaml
# Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: controller
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 1000m
            memory: 512Mi
```

---

## Troubleshooting

### High CPU Usage

**Symptom**: Controller using > 2 CPU cores constantly.

**Diagnosis**:
```bash
# CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Find hot spots
(pprof) top10
(pprof) list <function_name>
```

**Solutions**:

1. **Reduce log verbosity**:
   ```go
   logger.SetLevel(slog.LevelInfo)  // Not Debug
   ```

2. **Optimize hot path**:
   ```go
   // Cache expensive computations
   var cache = make(map[string]Result)

   func compute(key string) Result {
       if result, ok := cache[key]; ok {
           return result
       }
       result := expensiveComputation(key)
       cache[key] = result
       return result
   }
   ```

3. **Reduce worker count**:
   ```go
   r := router.New(ctx, router.Config{
       Workers:     1,  // Start low, increase as needed
       Threadiness: 2,
   })
   ```

### High Memory Usage

**Symptom**: Controller RSS > 2GB and growing.

**Diagnosis**:
```bash
# Heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Find allocations
(pprof) top10
(pprof) list <function_name>
```

**Solutions**:

1. **Find leak source**:
   ```go
   // Check for goroutine leaks
   runtime.NumGoroutine()  // Should be stable

   // Check for map growth
   log.Printf("Cache size: %d", len(globalCache))
   ```

2. **Implement cache eviction**:
   ```go
   // Use bounded cache
   cache := NewLRUCache(1000)  // Max 1000 entries
   ```

3. **Release resources**:
   ```go
   defer resp.Body.Close()  // Always close
   config = nil  // Release large objects
   ```

### Queue Backlog

**Symptom**: Work queue depth > 1000 items.

**Diagnosis**:
```bash
# Check queue metrics
curl http://localhost:9090/metrics | grep workqueue_depth
```

**Solutions**:

1. **Increase workers**:
   ```go
   r := router.New(ctx, router.Config{
       Workers:     4,  // Double workers
       Threadiness: 10,
   })
   ```

2. **Optimize handler**:
   ```go
   // Profile handler execution
   func timedHandler(obj *corev1.Pod, status corev1.PodStatus) error {
       start := time.Now()
       defer func() {
           duration := time.Since(start)
           if duration > 1*time.Second {
               log.Printf("Slow handler: %v", duration)
           }
       }()
       return processResource(obj)
   }
   ```

3. **Filter events**:
   ```go
   // Ignore resources that don't need processing
   router.Type(&corev1.Pod{}).
       Middleware(filterUnmanaged).
       HandlerFunc(handlePod).
       Register(r)
   ```

### Slow Reconciliation

**Symptom**: Resources take > 10 seconds to reconcile.

**Diagnosis**:
```bash
# Trace single reconciliation
curl http://localhost:6060/debug/pprof/trace?seconds=5 > trace.out
go tool trace trace.out
```

**Solutions**:

1. **Parallel operations**:
   ```go
   // Use errgroup for concurrent API calls
   g, ctx := errgroup.WithContext(ctx)
   g.Go(func() error { return createServiceAccount(ctx) })
   g.Go(func() error { return createConfigMap(ctx) })
   return g.Wait()
   ```

2. **Reduce API calls**:
   ```go
   // Batch operations
   for _, obj := range objects {
       batch.Add(obj)
   }
   return batch.Apply()  // Single API call
   ```

3. **Optimize validation**:
   ```go
   // Cache validation results
   if cached, ok := validationCache.Get(obj.UID); ok {
       return cached.(bool)
   }
   valid := validate(obj)
   validationCache.Set(obj.UID, valid)
   return valid
   ```

---

## Additional Resources

| Resource | Description |
|----------|-------------|
| [controllers.md](./controllers.md) | Event Router and handler patterns |
| [testing.md](./testing.md) | Performance testing strategies |
| [leader-election.md](./leader-election.md) | High-availability patterns |
| [apply.md](./apply.md) | Efficient resource management |
| [Go Performance Wiki](https://github.com/golang/go/wiki/Performance) | Official Go optimization guide |
| [Kubernetes Client-Go Docs](https://github.com/kubernetes/client-go) | Informer and caching patterns |

---

*For workspace-wide documentation patterns, see [documentation/docs/reference/documentation-guide.md](../../../documentation/docs/reference/documentation-guide.md).*
