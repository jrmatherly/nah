# Declarative Apply Guide

> **Version**: 1.0.0 | **Created**: 2026-01-19 | **Status**: Production Ready

This guide covers declarative resource management using nah's Apply API, implementing kubectl apply semantics for idempotent controller operations.

## Table of Contents

1. [Overview](#overview)
2. [Apply Fundamentals](#apply-fundamentals)
3. [Resource Ownership](#resource-ownership)
4. [Apply Strategies](#apply-strategies)
5. [Conflict Resolution](#conflict-resolution)
6. [Three-Way Merge](#three-way-merge)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)

---

## Overview

### What is Declarative Apply?

The nah Apply API provides kubectl apply semantics for programmatic resource management in controllers. It enables idempotent, declarative operations that merge desired state with existing resources while respecting manual changes.

### Key Features

- **kubectl apply Semantics**: Three-way merge between last-applied, current, and desired state
- **Ownership Tracking**: Automatic owner references and garbage collection
- **Conflict Resolution**: Configurable strategies for handling field conflicts
- **Idempotent Operations**: Safe to call repeatedly with same input
- **Backend Abstraction**: Works with both Kubernetes and kinm
- **Partial Updates**: Preserve fields not managed by the controller

### When to Use Apply

| Scenario | Use Apply | Use Direct Update |
|----------|-----------|-------------------|
| **Managing child resources** | ✅ Yes | ❌ No |
| **Declarative reconciliation** | ✅ Yes | ❌ No |
| **Allow manual edits** | ✅ Yes | ❌ No |
| **Status-only updates** | ❌ No | ✅ Yes |
| **Performance-critical** | ❌ No | ✅ Yes |
| **Full ownership required** | ❌ No | ✅ Yes |

### Architecture

```
┌─────────────────────────────────────────────────────┐
│              Controller Handler                      │
│  ┌────────────────────────────────────────────┐    │
│  │  Desired State (ConfigMap, Service, etc.)  │    │
│  └──────────────────┬─────────────────────────┘    │
│                     │                                │
│                     ↓                                │
│  ┌─────────────────────────────────────────────┐   │
│  │          nah Apply API                      │   │
│  │  • Load last-applied-configuration          │   │
│  │  • Fetch current resource                   │   │
│  │  • Three-way merge                          │   │
│  │  • Set owner references                     │   │
│  │  • Apply with conflict resolution           │   │
│  └──────────────────┬──────────────────────────┘   │
└────────────────────┼────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────┐
│           Kubernetes/kinm Backend                    │
│  ┌─────────────┐         ┌──────────────────────┐  │
│  │  Create     │         │  Update (Patch)      │  │
│  │  (if new)   │         │  (if exists)         │  │
│  └─────────────┘         └──────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## Apply Fundamentals

### Basic Apply

Create or update a resource declaratively:

```go
package main

import (
    "context"
    "github.com/obot-platform/nah/pkg/apply"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func reconcileConfigMap(ctx context.Context, owner *corev1.Pod) error {
    // Define desired state
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "my-config",
            Namespace: owner.Namespace,
            Labels: map[string]string{
                "app": "myapp",
            },
        },
        Data: map[string]string{
            "config.yaml": "key: value",
        },
    }

    // Apply will create or update the ConfigMap
    if err := apply.Apply(ctx, cm); err != nil {
        return err
    }

    return nil
}
```

### Apply with Owner Reference

Establish parent-child relationships for garbage collection:

```go
func reconcileService(ctx context.Context, deployment *appsv1.Deployment) error {
    service := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      deployment.Name,
            Namespace: deployment.Namespace,
        },
        Spec: corev1.ServiceSpec{
            Selector: deployment.Spec.Selector.MatchLabels,
            Ports: []corev1.ServicePort{
                {
                    Port:       80,
                    TargetPort: intstr.FromInt(8080),
                },
            },
        },
    }

    // Set deployment as owner - service will be deleted when deployment is deleted
    if err := apply.Apply(ctx, service, apply.WithOwner(deployment)); err != nil {
        return err
    }

    return nil
}
```

### Apply Multiple Resources

Efficiently apply multiple related resources:

```go
func reconcileApplication(ctx context.Context, app *v1alpha1.Application) error {
    // Create ConfigMap
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      app.Name + "-config",
            Namespace: app.Namespace,
        },
        Data: app.Spec.Config,
    }

    // Create Service
    svc := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      app.Name,
            Namespace: app.Namespace,
        },
        Spec: corev1.ServiceSpec{
            Selector: map[string]string{"app": app.Name},
            Ports:    app.Spec.Ports,
        },
    }

    // Create Deployment
    deploy := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      app.Name,
            Namespace: app.Namespace,
        },
        Spec: app.Spec.DeploymentSpec,
    }

    // Apply all resources with ownership
    resources := []client.Object{cm, svc, deploy}
    for _, resource := range resources {
        if err := apply.Apply(ctx, resource, apply.WithOwner(app)); err != nil {
            return err
        }
    }

    return nil
}
```

---

## Resource Ownership

### Owner References

Owner references enable Kubernetes garbage collection and establish resource hierarchies.

#### Setting Ownership

```go
// Set single owner
apply.Apply(ctx, childResource, apply.WithOwner(parentResource))

// Set owner with controller flag
apply.Apply(ctx, childResource,
    apply.WithOwner(parentResource),
    apply.WithController(true),
)

// Multiple owners (rare)
apply.Apply(ctx, resource,
    apply.WithOwner(owner1),
    apply.WithOwner(owner2),
)
```

#### Owner Reference Behavior

| Option | Behavior | Use Case |
|--------|----------|----------|
| **No owner** | Resource persists independently | Shared resources (ConfigMaps, Secrets) |
| **With owner** | Deleted when owner deleted | Dependent resources |
| **Controller=true** | Only one controller owner allowed | Primary controller relationship |
| **Controller=false** | Multiple non-controller owners | Shared ownership scenarios |

### Garbage Collection

Kubernetes automatically deletes owned resources when the owner is deleted:

```go
func reconcileDeployment(ctx context.Context, app *v1alpha1.Application) error {
    deployment := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      app.Name,
            Namespace: app.Namespace,
        },
        Spec: createDeploymentSpec(app),
    }

    // Deployment will be garbage collected when Application is deleted
    return apply.Apply(ctx, deployment,
        apply.WithOwner(app),
        apply.WithController(true),
    )
}
```

### Preventing Cascade Deletion

Use finalizers to perform cleanup before allowing deletion:

```go
const finalizerName = "myapp.io/cleanup"

func reconcileWithFinalizer(ctx context.Context, obj *corev1.Service) error {
    // Add finalizer on create
    if !hasFinalizer(obj, finalizerName) {
        obj.Finalizers = append(obj.Finalizers, finalizerName)
        return apply.Apply(ctx, obj)
    }

    // Handle deletion
    if !obj.DeletionTimestamp.IsZero() {
        // Perform cleanup (e.g., delete external load balancer)
        if err := cleanupExternalResources(ctx, obj); err != nil {
            return err
        }

        // Remove finalizer to allow deletion
        obj.Finalizers = removeFinalizer(obj.Finalizers, finalizerName)
        return apply.Apply(ctx, obj)
    }

    // Normal reconciliation
    return nil
}
```

---

## Apply Strategies

### Force Apply

Overwrite all fields, discarding manual changes:

```go
func forceApply(ctx context.Context, resource client.Object) error {
    // Force apply ignores three-way merge and overwrites
    return apply.Apply(ctx, resource, apply.WithForce(true))
}
```

⚠️ **Warning**: Force apply discards all manual edits and field manager conflicts.

### Dry Run

Preview changes without applying them:

```go
func previewChanges(ctx context.Context, resource client.Object) error {
    // Dry run shows what would change without modifying the resource
    return apply.Apply(ctx, resource, apply.WithDryRun(true))
}
```

### Field Manager

Specify the field manager for server-side apply:

```go
func applyWithFieldManager(ctx context.Context, resource client.Object) error {
    return apply.Apply(ctx, resource,
        apply.WithFieldManager("mycontroller"),
    )
}
```

### Apply If Not Exists

Create resource only if it doesn't exist (don't update):

```go
func createOnce(ctx context.Context, resource client.Object) error {
    return apply.Apply(ctx, resource, apply.WithCreateOnly(true))
}
```

### Conditional Apply

Apply based on generation or conditions:

```go
func conditionalApply(ctx context.Context, deploy *appsv1.Deployment) error {
    // Only apply if deployment spec changed
    if deploy.Generation == deploy.Status.ObservedGeneration {
        return nil // Already reconciled
    }

    return apply.Apply(ctx, deploy)
}
```

---

## Conflict Resolution

### Understanding Conflicts

Conflicts occur when:
- Multiple controllers manage the same resource
- Manual edits conflict with controller-managed fields
- Field managers disagree on field ownership

### Conflict Strategies

#### 1. Server-Side Apply (Recommended)

Use field management to track ownership:

```go
func serverSideApply(ctx context.Context, cm *corev1.ConfigMap) error {
    return apply.Apply(ctx, cm,
        apply.WithFieldManager("mycontroller"),
        apply.WithForceConflicts(false), // Respect other managers
    )
}
```

#### 2. Force Ownership

Take ownership of all fields:

```go
func forceOwnership(ctx context.Context, svc *corev1.Service) error {
    return apply.Apply(ctx, svc,
        apply.WithFieldManager("mycontroller"),
        apply.WithForceConflicts(true), // Override other managers
    )
}
```

⚠️ **Warning**: Forcing conflicts can cause other controllers to fight over the resource.

#### 3. Selective Field Management

Manage only specific fields:

```go
func manageLabelsOnly(ctx context.Context, pod *corev1.Pod) error {
    // Only manage labels, leave other fields alone
    pod.Labels = map[string]string{
        "managed-by": "mycontroller",
        "app":        "myapp",
    }

    return apply.Apply(ctx, pod,
        apply.WithFieldManager("mycontroller"),
        apply.WithManagedFields([]string{"metadata.labels"}),
    )
}
```

### Handling Conflict Errors

```go
func applyWithRetry(ctx context.Context, resource client.Object) error {
    err := apply.Apply(ctx, resource)

    // Check for conflict
    if errors.IsConflict(err) {
        // Fetch latest version
        key := client.ObjectKeyFromObject(resource)
        if err := client.Get(ctx, key, resource); err != nil {
            return err
        }

        // Retry with latest version
        return apply.Apply(ctx, resource)
    }

    return err
}
```

### Conflict Resolution Pattern

```go
func reconcileWithConflictResolution(ctx context.Context, cm *corev1.ConfigMap) error {
    const maxRetries = 3

    for i := 0; i < maxRetries; i++ {
        // Fetch current state
        current := &corev1.ConfigMap{}
        key := client.ObjectKeyFromObject(cm)
        if err := client.Get(ctx, key, current); err != nil {
            if errors.IsNotFound(err) {
                // Create new resource
                return apply.Apply(ctx, cm)
            }
            return err
        }

        // Merge desired changes into current version
        current.Data = cm.Data
        current.Labels = cm.Labels

        // Apply with current ResourceVersion
        err := apply.Apply(ctx, current)
        if !errors.IsConflict(err) {
            return err
        }

        // Retry on conflict
        time.Sleep(time.Millisecond * 100 * time.Duration(i+1))
    }

    return fmt.Errorf("failed to apply after %d retries", maxRetries)
}
```

---

## Three-Way Merge

### How It Works

The Apply API performs a three-way merge between:

1. **Last Applied**: Stored in `kubectl.kubernetes.io/last-applied-configuration` annotation
2. **Current State**: Resource as it exists in the cluster
3. **Desired State**: Resource as specified by the controller

```
┌────────────────┐   ┌────────────────┐   ┌────────────────┐
│  Last Applied  │   │ Current State  │   │ Desired State  │
│  (annotation)  │   │  (in cluster)  │   │ (from code)    │
└────────┬───────┘   └────────┬───────┘   └────────┬───────┘
         │                    │                     │
         └────────────────────┼─────────────────────┘
                              ↓
                    ┌──────────────────┐
                    │   Three-Way      │
                    │     Merge        │
                    └────────┬─────────┘
                             │
                             ↓
                    ┌──────────────────┐
                    │  Applied Result  │
                    └──────────────────┘
```

### Merge Behavior

| Field State | Last Applied | Current | Desired | Result |
|-------------|--------------|---------|---------|--------|
| **Unchanged** | `value` | `value` | `value` | `value` |
| **User modified** | `value` | `manual` | `value` | `manual` (preserve) |
| **Controller updated** | `old` | `old` | `new` | `new` |
| **Both modified** | `value` | `manual` | `new` | `new` (conflict) |
| **User added field** | - | `manual` | - | `manual` (preserve) |
| **Controller added** | - | - | `new` | `new` |

### Example: Three-Way Merge

```go
func demonstrateThreeWayMerge(ctx context.Context) error {
    // Initial apply
    cm1 := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "config",
            Namespace: "default",
        },
        Data: map[string]string{
            "key1": "value1",
            "key2": "value2",
        },
    }
    apply.Apply(ctx, cm1) // Last-applied stored

    // User manually edits in cluster:
    // - Changes key1: "value1" -> "manual-edit"
    // - Adds key3: "manual-addition"

    // Controller updates desired state
    cm2 := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "config",
            Namespace: "default",
        },
        Data: map[string]string{
            "key1": "value1",      // Unchanged by controller
            "key2": "updated",     // Updated by controller
        },
    }
    apply.Apply(ctx, cm2)

    // Result after three-way merge:
    // Data: {
    //   "key1": "manual-edit",      // Preserved (user changed)
    //   "key2": "updated",          // Updated (controller changed)
    //   "key3": "manual-addition",  // Preserved (user added)
    // }

    return nil
}
```

### Disabling Last-Applied Tracking

For resources that don't need three-way merge:

```go
func applyWithoutTracking(ctx context.Context, resource client.Object) error {
    return apply.Apply(ctx, resource,
        apply.WithoutLastApplied(true),
    )
}
```

---

## Best Practices

### 1. Always Set Owner References

Enable automatic garbage collection:

```go
func createDependentResources(ctx context.Context, parent client.Object) error {
    resources := generateChildResources(parent)

    for _, resource := range resources {
        // ✅ Good: Set owner reference
        if err := apply.Apply(ctx, resource, apply.WithOwner(parent)); err != nil {
            return err
        }
    }

    return nil
}
```

### 2. Use Consistent Field Managers

Prevent conflicts between controller versions:

```go
const fieldManager = "mycontroller.io"

func applyConsistently(ctx context.Context, resource client.Object) error {
    return apply.Apply(ctx, resource,
        apply.WithFieldManager(fieldManager),
    )
}
```

### 3. Handle Not Found Errors

Resources may be deleted between reads and applies:

```go
func safeApply(ctx context.Context, resource client.Object) error {
    err := apply.Apply(ctx, resource)

    if errors.IsNotFound(err) {
        // Parent resource was deleted, stop reconciliation
        return nil
    }

    return err
}
```

### 4. Avoid Force Apply

Preserve user customizations:

```go
// ❌ Bad: Overwrites all manual changes
apply.Apply(ctx, resource, apply.WithForce(true))

// ✅ Good: Respects manual edits
apply.Apply(ctx, resource)
```

### 5. Apply Immutable Resources Carefully

Some fields cannot be changed after creation:

```go
func applyService(ctx context.Context, svc *corev1.Service) error {
    // Check if service exists
    existing := &corev1.Service{}
    key := client.ObjectKeyFromObject(svc)
    err := client.Get(ctx, key, existing)

    if err == nil {
        // Service exists - preserve immutable fields
        svc.Spec.ClusterIP = existing.Spec.ClusterIP
        svc.Spec.IPFamilies = existing.Spec.IPFamilies
    }

    return apply.Apply(ctx, svc)
}
```

### 6. Batch Applies for Efficiency

Apply related resources together:

```go
func applyBatch(ctx context.Context, resources []client.Object, owner client.Object) error {
    // Group applies to minimize API calls
    for _, resource := range resources {
        if err := apply.Apply(ctx, resource, apply.WithOwner(owner)); err != nil {
            return fmt.Errorf("failed to apply %s: %w",
                resource.GetName(), err)
        }
    }

    return nil
}
```

### 7. Separate Spec and Status Updates

Use Apply for spec, direct update for status:

```go
func reconcile(ctx context.Context, obj *v1alpha1.MyResource) error {
    // Apply spec changes
    if err := apply.Apply(ctx, obj); err != nil {
        return err
    }

    // Update status separately
    obj.Status.Phase = "Ready"
    return client.Status().Update(ctx, obj)
}
```

### 8. Use Structured Logging

Track apply operations for debugging:

```go
import "log/slog"

func applyWithLogging(ctx context.Context, resource client.Object) error {
    logger := slog.With(
        "resource", fmt.Sprintf("%s/%s",
            resource.GetNamespace(), resource.GetName()),
        "kind", resource.GetObjectKind().GroupVersionKind().Kind,
    )

    logger.Info("applying resource")

    err := apply.Apply(ctx, resource)
    if err != nil {
        logger.Error("apply failed", "error", err)
        return err
    }

    logger.Info("apply succeeded")
    return nil
}
```

---

## Troubleshooting

### Apply Fails with "Field Managed Conflict"

**Symptom**: Error message about field manager conflicts.

**Solutions**:

1. **Use consistent field manager**:
   ```go
   apply.Apply(ctx, resource,
       apply.WithFieldManager("mycontroller.io"),
   )
   ```

2. **Force take ownership** (use carefully):
   ```go
   apply.Apply(ctx, resource,
       apply.WithFieldManager("mycontroller.io"),
       apply.WithForceConflicts(true),
   )
   ```

3. **Check who owns the field**:
   ```bash
   kubectl get configmap my-config -o yaml | grep -A20 managedFields
   ```

### Resource Not Garbage Collected

**Symptom**: Child resources persist after owner deletion.

**Solutions**:

1. **Verify owner reference is set**:
   ```go
   // Check that owner reference exists
   if len(resource.GetOwnerReferences()) == 0 {
       log.Printf("WARNING: No owner references set")
   }
   ```

2. **Check owner UID matches**:
   ```bash
   # Owner UID must match exactly
   kubectl get pod my-pod -o yaml | grep ownerReferences -A5
   ```

3. **Verify controller flag** (optional):
   ```go
   apply.Apply(ctx, resource,
       apply.WithOwner(owner),
       apply.WithController(true), // Required for some GC scenarios
   )
   ```

### Apply Repeatedly Updates Resource

**Symptom**: Resource shows constant updates in watch stream.

**Solutions**:

1. **Check for unstable defaults**:
   ```go
   // ❌ Bad: Generates new timestamp each reconciliation
   cm.Data["timestamp"] = time.Now().String()

   // ✅ Good: Use stable values
   if _, exists := cm.Data["created"]; !exists {
       cm.Data["created"] = time.Now().String()
   }
   ```

2. **Compare generations**:
   ```go
   if resource.Generation == resource.Status.ObservedGeneration {
       return nil // No spec changes
   }
   ```

3. **Use deep equality check**:
   ```go
   import "reflect"

   if reflect.DeepEqual(desired.Spec, current.Spec) {
       return nil // No changes needed
   }
   ```

### Manual Changes Overwritten

**Symptom**: User edits are lost after controller reconciliation.

**Solutions**:

1. **Don't use force apply**:
   ```go
   // ❌ Bad: Overwrites manual changes
   apply.Apply(ctx, resource, apply.WithForce(true))

   // ✅ Good: Respects three-way merge
   apply.Apply(ctx, resource)
   ```

2. **Fetch before apply**:
   ```go
   // Get current state
   current := &corev1.ConfigMap{}
   key := client.ObjectKeyFromObject(desired)
   client.Get(ctx, key, current)

   // Merge manual changes
   for k, v := range current.Data {
       if _, managed := desired.Data[k]; !managed {
           desired.Data[k] = v // Preserve unmanaged fields
       }
   }

   apply.Apply(ctx, desired)
   ```

### Apply Performance Issues

**Symptom**: Apply operations are slow.

**Solutions**:

1. **Batch applies**:
   ```go
   // Apply related resources together
   errgroup := new(errgroup.Group)
   for _, resource := range resources {
       resource := resource
       errgroup.Go(func() error {
           return apply.Apply(ctx, resource)
       })
   }
   return errgroup.Wait()
   ```

2. **Skip unnecessary applies**:
   ```go
   // Check if apply is needed
   if !needsUpdate(current, desired) {
       return nil
   }
   ```

3. **Use apply without tracking** for performance-critical paths:
   ```go
   apply.Apply(ctx, resource, apply.WithoutLastApplied(true))
   ```

### Owner Reference Validation Failed

**Symptom**: Error about cross-namespace owner references.

**Solutions**:

1. **Ensure same namespace**:
   ```go
   func applyWithNamespaceCheck(ctx context.Context, child, parent client.Object) error {
       if child.GetNamespace() != parent.GetNamespace() {
           return fmt.Errorf("cross-namespace owner references not allowed")
       }

       return apply.Apply(ctx, child, apply.WithOwner(parent))
   }
   ```

2. **Use cluster-scoped owners** for namespace-scoped children:
   ```go
   // Cluster-scoped resources (e.g., Node) can own namespace-scoped resources
   apply.Apply(ctx, pod, apply.WithOwner(node))
   ```

---

## Additional Resources

| Resource | Description |
|----------|-------------|
| [controllers.md](./controllers.md) | Event Router and handler patterns |
| [leader-election.md](./leader-election.md) | High-availability controller deployments |
| [testing.md](./testing.md) | Testing Apply operations and reconciliation |
| [performance.md](./performance.md) | Optimizing Apply performance |
| [nah/pkg/apply](../../pkg/apply/) | Apply API implementation and reference |

---

*For workspace-wide documentation patterns, see [documentation/docs/reference/documentation-guide.md](../../../documentation/docs/reference/documentation-guide.md).*
