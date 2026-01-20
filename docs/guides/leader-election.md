# Leader Election Guide

> **Version**: 1.0.0 | **Created**: 2026-01-19 | **Status**: Production Ready

This guide covers configuring high-availability controller deployments with leader election for production-grade Kubernetes controllers built with nah.

## Table of Contents

1. [Overview](#overview)
2. [Leader Election Fundamentals](#leader-election-fundamentals)
3. [Lease-Based Election](#lease-based-election)
4. [File-Based Election](#file-based-election)
5. [HA Deployment Patterns](#ha-deployment-patterns)
6. [Configuration Options](#configuration-options)
7. [Failure Recovery](#failure-recovery)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)

---

## Overview

### What is Leader Election?

Leader election enables multiple replicas of a controller to run simultaneously while ensuring only one replica (the "leader") actively reconciles resources. This provides high availability without duplicate reconciliation or race conditions.

### Key Benefits

- **High Availability**: Automatic failover if the leader crashes or becomes unreachable
- **Zero Downtime**: Leaders transition seamlessly during rolling updates
- **Split-Brain Prevention**: Only one controller actively reconciles resources
- **Resource Efficiency**: Standby replicas consume minimal resources while waiting

### When to Use Leader Election

| Scenario | Use Leader Election | Reasoning |
|----------|---------------------|-----------|
| **Production deployments** | ✅ Yes | Ensures availability during node failures |
| **Controllers managing cluster-scoped resources** | ✅ Yes | Prevents duplicate reconciliation |
| **Controllers with external side effects** | ✅ Yes | Ensures exactly-once semantics |
| **Development/testing** | ❌ Optional | Single replica is simpler |
| **Read-only controllers** | ❌ Optional | Multiple readers are safe |

### Architecture

```
┌─────────────────────────────────────────────────────┐
│           Kubernetes Cluster                         │
│                                                       │
│  ┌──────────────────┐      ┌──────────────────┐    │
│  │  Controller Pod  │      │  Controller Pod  │    │
│  │   (Leader) ✓     │      │   (Standby)      │    │
│  │                  │      │                  │    │
│  │  Reconciling ──┐ │      │  Waiting...      │    │
│  └────────┬───────┼┘      └──────────────────┘    │
│           │       │                                 │
│           │       │        ┌──────────────────┐    │
│           │       │        │  Controller Pod  │    │
│           │       │        │   (Standby)      │    │
│           │       │        │                  │    │
│           │       │        │  Waiting...      │    │
│           │       │        └──────────────────┘    │
│           │       │                                 │
│           │       └──────────────┐                 │
│           ↓                      ↓                 │
│  ┌─────────────────┐    ┌──────────────────────┐  │
│  │   Resources     │    │  Lease Resource      │  │
│  │   (Reconciled)  │    │  coordination.k8s.io │  │
│  │                 │    │  v1/Lease            │  │
│  └─────────────────┘    │                      │  │
│                          │  HolderIdentity:     │  │
│                          │  controller-pod-1    │  │
│                          │  LeaseDuration: 15s  │  │
│                          │  RenewTime: now-2s   │  │
│                          └──────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## Leader Election Fundamentals

### How It Works

Leader election uses a distributed locking mechanism based on Kubernetes Lease resources:

1. **Lease Acquisition**: Controller attempts to create/acquire a Lease resource
2. **Leader Operations**: Holder of the lease becomes leader and starts reconciliation
3. **Lease Renewal**: Leader periodically renews the lease to maintain leadership
4. **Leader Failover**: If leader fails to renew, standby replicas compete for the lease
5. **Graceful Shutdown**: Leader releases lease on clean shutdown

### Lease Resource

Leader election uses the `coordination.k8s.io/v1` Lease API:

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: my-controller-leader
  namespace: default
spec:
  holderIdentity: "controller-pod-abc123"
  leaseDurationSeconds: 15
  acquireTime: "2026-01-19T10:00:00Z"
  renewTime: "2026-01-19T10:00:12Z"
  leaseTransitions: 3
```

### Timing Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| **LeaseDuration** | 15s | How long lease is valid without renewal |
| **RenewDeadline** | 10s | Leader must renew before this deadline |
| **RetryPeriod** | 2s | How often non-leaders attempt acquisition |

**Formula**: `LeaseDuration > RenewDeadline > RetryPeriod`

---

## Lease-Based Election

### Basic Setup

Standard production deployment using Kubernetes Lease resources:

```go
package main

import (
    "context"
    "os"
    "time"

    "github.com/obot-platform/nah/pkg/router"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
    "k8s.io/client-go/tools/leaderelection"
    "k8s.io/client-go/tools/leaderelection/resourcelock"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func main() {
    ctx := context.Background()

    // Load kubeconfig
    config, err := rest.InClusterConfig()
    if err != nil {
        panic(err)
    }

    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        panic(err)
    }

    // Configure leader election
    lock := &resourcelock.LeaseLock{
        LeaseMeta: metav1.ObjectMeta{
            Name:      "my-controller-leader",
            Namespace: "default",
        },
        Client: clientset.CoordinationV1(),
        LockConfig: resourcelock.ResourceLockConfig{
            Identity: os.Getenv("HOSTNAME"), // Pod name
        },
    }

    // Run leader election
    leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
        Lock:            lock,
        ReleaseOnCancel: true,
        LeaseDuration:   15 * time.Second,
        RenewDeadline:   10 * time.Second,
        RetryPeriod:     2 * time.Second,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(ctx context.Context) {
                // Start controller
                runController(ctx, config)
            },
            OnStoppedLeading: func() {
                // Leader lost, exit to allow restart
                os.Exit(0)
            },
            OnNewLeader: func(identity string) {
                if identity == os.Getenv("HOSTNAME") {
                    log.Printf("I am the new leader")
                } else {
                    log.Printf("Current leader: %s", identity)
                }
            },
        },
    })
}

func runController(ctx context.Context, config *rest.Config) {
    // Create nah router
    r := router.New(ctx, router.Config{
        Namespace: "default",
    })

    // Register handlers
    router.Type(&corev1.ConfigMap{}).
        HandlerFunc(reconcileConfigMap).
        Register(r)

    // Start controller (blocks until context cancelled)
    if err := r.Start(ctx); err != nil {
        panic(err)
    }
}
```

### RBAC Requirements

Leader election requires permissions for Lease resources:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-controller-leader-election
  namespace: default
rules:
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["get", "create", "update"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-controller-leader-election
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: my-controller-leader-election
subjects:
- kind: ServiceAccount
  name: my-controller
  namespace: default
```

### Deployment Configuration

Deploy multiple replicas with leader election:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-controller
  namespace: default
spec:
  replicas: 3  # High availability with 3 replicas
  selector:
    matchLabels:
      app: my-controller
  template:
    metadata:
      labels:
        app: my-controller
    spec:
      serviceAccountName: my-controller
      containers:
      - name: controller
        image: my-controller:v1.0.0
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

---

## File-Based Election

### Development Setup

For local development with kinm, use file-based leader election:

```go
package main

import (
    "context"
    "os"
    "path/filepath"
    "time"

    "github.com/obot-platform/nah/pkg/router"
    "k8s.io/client-go/tools/leaderelection"
    "k8s.io/client-go/tools/leaderelection/resourcelock"
)

func runWithFileElection(ctx context.Context) {
    // Create file-based lock
    lockFile := filepath.Join(os.TempDir(), "my-controller.lock")

    lock := &resourcelock.FileLock{
        Path: lockFile,
        Identity: os.Getenv("HOSTNAME"),
    }

    // Run leader election with file lock
    leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
        Lock:            lock,
        ReleaseOnCancel: true,
        LeaseDuration:   15 * time.Second,
        RenewDeadline:   10 * time.Second,
        RetryPeriod:     2 * time.Second,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(ctx context.Context) {
                runKinmController(ctx)
            },
            OnStoppedLeading: func() {
                os.Exit(0)
            },
        },
    })
}

func runKinmController(ctx context.Context) {
    // Create router with kinm backend
    r := router.New(ctx, router.Config{
        Namespace: "default",
        // kinm-specific configuration
    })

    // Register handlers and start
    if err := r.Start(ctx); err != nil {
        panic(err)
    }
}
```

### File Lock Location

Choose lock file location based on environment:

| Environment | Lock Path | Reasoning |
|-------------|-----------|-----------|
| **Development** | `/tmp/controller.lock` | Simple, auto-cleaned |
| **Docker Compose** | `/shared/controller.lock` | Shared volume for multiple containers |
| **Local Testing** | `./data/controller.lock` | Project-local state |

---

## HA Deployment Patterns

### Active-Standby Pattern

**Default pattern**: One leader, multiple standbys.

```
┌────────────────────────────────────────┐
│  Replica 1 (Leader)                    │
│  • Actively reconciling                │
│  • Renewing lease                      │
│  • Processing all events               │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│  Replica 2 (Standby)                   │
│  • Watching for leader failure         │
│  • Ready to become leader              │
│  • Minimal resource usage              │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│  Replica 3 (Standby)                   │
│  • Watching for leader failure         │
│  • Ready to become leader              │
│  • Minimal resource usage              │
└────────────────────────────────────────┘
```

**Pros**:
- Simple to reason about
- No split-brain risk
- Consistent reconciliation

**Cons**:
- Standby resources are idle
- Limited horizontal scaling

### Sharded Pattern

**Advanced pattern**: Multiple leaders, each responsible for a subset of resources.

```go
import (
    "hash/fnv"
    "github.com/obot-platform/nah/pkg/router"
)

// Create multiple leader election instances with different lease names
func runShardedController(ctx context.Context, shardID int, totalShards int) {
    leaseName := fmt.Sprintf("my-controller-shard-%d", shardID)

    // Leader election for this shard
    leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
        Lock: createLeaseLock(leaseName),
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(ctx context.Context) {
                runShardController(ctx, shardID, totalShards)
            },
        },
    })
}

func runShardController(ctx context.Context, shardID, totalShards int) {
    r := router.New(ctx, router.Config{
        Namespace: "default",
    })

    // Only process resources belonging to this shard
    router.Type(&corev1.ConfigMap{}).
        Middleware(shardFilter(shardID, totalShards)).
        HandlerFunc(reconcileConfigMap).
        Register(r)

    r.Start(ctx)
}

func shardFilter(shardID, totalShards int) router.Middleware {
    return func(obj *corev1.ConfigMap, status corev1.ConfigMapStatus, handler router.Handler) error {
        // Hash resource name to determine shard
        h := fnv.New32a()
        h.Write([]byte(obj.Namespace + "/" + obj.Name))
        resourceShard := int(h.Sum32()) % totalShards

        // Only process if this shard owns the resource
        if resourceShard != shardID {
            return nil
        }

        return handler(obj, status)
    }
}
```

**Deployment**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-controller-shard-0
spec:
  replicas: 2  # HA for shard 0
  template:
    spec:
      containers:
      - name: controller
        env:
        - name: SHARD_ID
          value: "0"
        - name: TOTAL_SHARDS
          value: "3"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-controller-shard-1
spec:
  replicas: 2  # HA for shard 1
  # ... SHARD_ID=1

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-controller-shard-2
spec:
  replicas: 2  # HA for shard 2
  # ... SHARD_ID=2
```

**Pros**:
- Horizontal scaling across shards
- Distributes load across multiple leaders
- Each shard is independently HA

**Cons**:
- More complex deployment
- Requires careful shard distribution
- Rebalancing on shard count changes is complex

---

## Configuration Options

### Timing Tuning

Adjust timing parameters based on requirements:

```go
// Fast failover (aggressive)
leaderelection.LeaderElectionConfig{
    LeaseDuration: 10 * time.Second,  // Shorter lease
    RenewDeadline: 7 * time.Second,   // Faster renewal
    RetryPeriod:   1 * time.Second,   // Frequent retries
}
// Pros: ~10s failover time
// Cons: More API calls, higher CPU usage

// Balanced (default)
leaderelection.LeaderElectionConfig{
    LeaseDuration: 15 * time.Second,
    RenewDeadline: 10 * time.Second,
    RetryPeriod:   2 * time.Second,
}
// Pros: Good balance of responsiveness and efficiency
// Cons: ~15s failover time

// Conservative (gentle)
leaderelection.LeaderElectionConfig{
    LeaseDuration: 30 * time.Second,  // Longer lease
    RenewDeadline: 20 * time.Second,  // Relaxed renewal
    RetryPeriod:   5 * time.Second,   // Infrequent retries
}
// Pros: Lower API load, minimal overhead
// Cons: ~30s failover time
```

### Identity Configuration

Set unique identity for each replica:

```go
// Option 1: Use pod name (Kubernetes)
identity := os.Getenv("HOSTNAME")

// Option 2: Use pod UID (more unique)
identity := os.Getenv("POD_UID")

// Option 3: Generate unique ID
import "github.com/google/uuid"
identity := uuid.New().String()

// Option 4: Combine namespace + pod name
identity := fmt.Sprintf("%s/%s",
    os.Getenv("POD_NAMESPACE"),
    os.Getenv("POD_NAME"),
)
```

**Best practice**: Use pod name for observability, pod UID for uniqueness.

### Namespace Scoping

Leader election can be cluster-scoped or namespace-scoped:

```go
// Namespace-scoped (recommended)
lock := &resourcelock.LeaseLock{
    LeaseMeta: metav1.ObjectMeta{
        Name:      "my-controller",
        Namespace: "my-namespace",  // Scoped to namespace
    },
}

// Cluster-scoped (requires ClusterRole)
lock := &resourcelock.LeaseLock{
    LeaseMeta: metav1.ObjectMeta{
        Name:      "my-controller",
        Namespace: "kube-system",  // Shared namespace
    },
}
```

---

## Failure Recovery

### Leader Crash Recovery

**Scenario**: Leader pod crashes or is deleted.

```
Time 0s:  Leader crashes
Time 0-15s: Standby replicas detect missing lease renewal
Time 15s: Lease expires
Time 15-17s: Standbys compete for lease (RetryPeriod)
Time 17s: New leader elected
Time 17s+: New leader starts reconciliation
```

**Total failover time**: LeaseDuration + RetryPeriod (~17s with defaults)

### Network Partition Recovery

**Scenario**: Leader loses network connectivity.

```go
// Leader loses connectivity but keeps running
// Lease renewal fails -> leader stops reconciling after RenewDeadline
// New leader elected after LeaseDuration
// Old leader reconnects -> sees it's not leader -> exits

leaderelection.LeaderElectionConfig{
    Callbacks: leaderelection.LeaderCallbacks{
        OnStoppedLeading: func() {
            log.Printf("Lost leadership, exiting")
            // Exit to force clean restart
            os.Exit(0)
        },
    },
}
```

**Best practice**: Exit on leadership loss to ensure clean state.

### Graceful Shutdown

Release lease on clean shutdown:

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())

    // Handle shutdown signals
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        <-sigCh
        log.Printf("Shutdown signal received, releasing lease")
        cancel() // Triggers ReleaseOnCancel
    }()

    leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
        ReleaseOnCancel: true,  // Release lease on context cancellation
        // ...
    })
}
```

### Split-Brain Prevention

Leader election prevents split-brain scenarios:

```go
// Anti-pattern: Multiple leaders reconciling simultaneously
// ❌ DON'T DO THIS
func main() {
    // No leader election
    runController(context.Background())
}

// Correct pattern: Single active leader
// ✅ DO THIS
func main() {
    leaderelection.RunOrDie(ctx, config)
}
```

**Protection mechanisms**:
- Lease atomicity: Only one holder at a time
- Renewal deadline: Leader steps down if renewal fails
- Standby election: New leader only after lease expiry

---

## Best Practices

### 1. Use Unique Pod Names

Ensure pod names are unique for accurate leader tracking:

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    metadata:
      labels:
        app: my-controller
    spec:
      # StatefulSet-like naming
      hostname: my-controller
      subdomain: my-controller-pods
```

Or use StatefulSet for guaranteed unique names:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-controller
spec:
  serviceName: my-controller
  replicas: 3
  # Pods named: my-controller-0, my-controller-1, my-controller-2
```

### 2. Set Resource Requests

Ensure standbys have minimal resource consumption:

```yaml
spec:
  containers:
  - name: controller
    resources:
      requests:
        cpu: 50m      # Minimal CPU for standby
        memory: 64Mi  # Minimal memory for standby
      limits:
        cpu: 500m     # Burst capacity when leader
        memory: 512Mi
```

### 3. Configure Health Checks

Health checks should consider leadership status:

```go
import "net/http"

var isLeader bool

func healthHandler(w http.ResponseWriter, r *http.Request) {
    // Liveness: Pod is running
    w.WriteHeader(http.StatusOK)
}

func readinessHandler(w http.ResponseWriter, r *http.Request) {
    // Readiness: Only leader is ready
    if isLeader {
        w.WriteHeader(http.StatusOK)
    } else {
        w.WriteHeader(http.StatusServiceUnavailable)
    }
}

// Update leadership status in callbacks
Callbacks: leaderelection.LeaderCallbacks{
    OnStartedLeading: func(ctx context.Context) {
        isLeader = true
        runController(ctx)
    },
    OnStoppedLeading: func() {
        isLeader = false
        os.Exit(0)
    },
}
```

Kubernetes probe configuration:

```yaml
spec:
  containers:
  - name: controller
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
    readinessProbe:
      httpGet:
        path: /readyz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
```

### 4. Log Leadership Changes

Track leadership transitions for debugging:

```go
import "log/slog"

Callbacks: leaderelection.LeaderCallbacks{
    OnStartedLeading: func(ctx context.Context) {
        slog.Info("started leading",
            "identity", os.Getenv("HOSTNAME"),
            "lease", "my-controller",
        )
        runController(ctx)
    },
    OnStoppedLeading: func() {
        slog.Warn("stopped leading",
            "identity", os.Getenv("HOSTNAME"),
            "reason", "lease lost",
        )
        os.Exit(0)
    },
    OnNewLeader: func(identity string) {
        if identity == os.Getenv("HOSTNAME") {
            slog.Info("became new leader", "identity", identity)
        } else {
            slog.Info("new leader elected",
                "leader", identity,
                "self", os.Getenv("HOSTNAME"),
            )
        }
    },
}
```

### 5. Test Failover Scenarios

Regularly test leader failover:

```bash
# Delete current leader pod
kubectl delete pod -l app=my-controller --field-selector=status.phase=Running | head -1

# Watch leadership transition
kubectl get lease my-controller-leader -w

# Verify new leader started reconciling
kubectl logs -f deployment/my-controller
```

### 6. Monitor Lease Metrics

Track lease health with metrics:

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    leaderElectionGauge = promauto.NewGauge(prometheus.GaugeOpts{
        Name: "controller_leader_election_status",
        Help: "1 if leader, 0 if standby",
    })

    leaderTransitionsCounter = promauto.NewCounter(prometheus.CounterOpts{
        Name: "controller_leader_transitions_total",
        Help: "Total number of leadership transitions",
    })
)

Callbacks: leaderelection.LeaderCallbacks{
    OnStartedLeading: func(ctx context.Context) {
        leaderElectionGauge.Set(1)
        leaderTransitionsCounter.Inc()
        runController(ctx)
    },
    OnStoppedLeading: func() {
        leaderElectionGauge.Set(0)
        os.Exit(0)
    },
}
```

---

## Troubleshooting

### No Leader Elected

**Symptom**: All replicas are standbys, no leader.

**Solutions**:

1. Check RBAC permissions:
   ```bash
   kubectl auth can-i create leases --namespace=default --as=system:serviceaccount:default:my-controller
   kubectl auth can-i update leases --namespace=default --as=system:serviceaccount:default:my-controller
   ```

2. Verify lease resource:
   ```bash
   kubectl get lease my-controller-leader -o yaml
   ```

3. Check pod logs:
   ```bash
   kubectl logs deployment/my-controller | grep -i "leader"
   ```

### Multiple Leaders

**Symptom**: Multiple replicas think they're leader (split-brain).

**Causes**:
- Clock skew between nodes
- Network partition
- Lease resource deleted externally

**Solutions**:

1. Check node clock synchronization:
   ```bash
   kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].lastHeartbeatTime}{"\n"}{end}'
   ```

2. Verify single lease holder:
   ```bash
   kubectl get lease my-controller-leader -o jsonpath='{.spec.holderIdentity}'
   ```

3. Force lease recreation:
   ```bash
   kubectl delete lease my-controller-leader
   ```

### Frequent Leadership Changes

**Symptom**: Leadership flaps between replicas.

**Solutions**:

1. Check pod stability:
   ```bash
   kubectl get pods -l app=my-controller -w
   ```

2. Increase lease duration:
   ```go
   LeaseDuration: 30 * time.Second,  // More stable
   ```

3. Check resource constraints:
   ```bash
   kubectl top pods -l app=my-controller
   ```

4. Review pod eviction logs:
   ```bash
   kubectl get events --sort-by='.lastTimestamp' | grep -i evict
   ```

### Slow Failover

**Symptom**: Long delay before new leader starts.

**Solutions**:

1. Reduce lease duration:
   ```go
   LeaseDuration: 10 * time.Second,  // Faster failover
   RenewDeadline: 7 * time.Second,
   RetryPeriod:   1 * time.Second,
   ```

2. Check API server latency:
   ```bash
   kubectl get --raw /metrics | grep apiserver_request_duration
   ```

3. Monitor lease renewal:
   ```bash
   kubectl get lease my-controller-leader -w
   ```

### Leader Stuck

**Symptom**: Leader holds lease but doesn't reconcile.

**Solutions**:

1. Check leader pod logs:
   ```bash
   kubectl logs -l app=my-controller --tail=100
   ```

2. Verify controller started:
   ```go
   OnStartedLeading: func(ctx context.Context) {
       log.Printf("Starting controller...")
       runController(ctx)
       log.Printf("Controller stopped")
   }
   ```

3. Force new election:
   ```bash
   # Delete leader pod
   LEADER=$(kubectl get lease my-controller-leader -o jsonpath='{.spec.holderIdentity}')
   kubectl delete pod $LEADER
   ```

### Lease Permission Errors

**Symptom**: Errors like "cannot create resource leases".

**Solution**: Create RBAC resources:

```bash
# Create Role
kubectl create role my-controller-leader-election \
  --verb=get,create,update \
  --resource=leases \
  --resource-name=my-controller-leader

# Create RoleBinding
kubectl create rolebinding my-controller-leader-election \
  --role=my-controller-leader-election \
  --serviceaccount=default:my-controller
```

---

## Additional Resources

| Resource | Description |
|----------|-------------|
| [controllers.md](./controllers.md) | Event routing and handler patterns |
| [apply.md](./apply.md) | Declarative resource management |
| [testing.md](./testing.md) | Controller testing strategies |
| [performance.md](./performance.md) | Optimization and profiling |
| [Kubernetes Leader Election](https://pkg.go.dev/k8s.io/client-go/tools/leaderelection) | Official client-go documentation |
| [coordination.k8s.io/v1](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/lease-v1/) | Lease API reference |

---

*For workspace-wide documentation patterns, see [documentation/docs/reference/documentation-guide.md](../../../documentation/docs/reference/documentation-guide.md).*
