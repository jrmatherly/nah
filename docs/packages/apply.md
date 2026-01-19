# Apply Package API Reference

**Package:** `github.com/obot-platform/nah/pkg/apply`

The `apply` package provides declarative resource management with `kubectl apply`-like semantics for Kubernetes controllers. It handles owner references, garbage collection, and three-way merge patches.

## Table of Contents

- [Overview](#overview)
- [Interfaces](#interfaces)
  - [Apply](#apply)
- [Functions](#functions)
- [Concepts](#concepts)
  - [Owner References](#owner-references)
  - [Pruning](#pruning)
  - [Subcontexts](#subcontexts)
- [Usage Examples](#usage-examples)

---

## Overview

The apply package enables controllers to declare desired state and automatically reconcile to that state. It:

- **Creates** resources that don't exist
- **Updates** resources that have changed (using three-way merge)
- **Deletes** (prunes) resources no longer in desired state
- **Manages** owner references for garbage collection
- **Handles** conflicts and ownership changes

This approach follows Kubernetes declarative principles and mirrors `kubectl apply` behavior.

---

## Interfaces

### Apply

The main interface for declarative resource management.

```go
type Apply interface {
    Ensure(ctx context.Context, obj ...client.Object) error
    Apply(ctx context.Context, owner client.Object, objs ...client.Object) error
    WithOwnerSubContext(ownerSubContext string) Apply
    WithNamespace(ns string) Apply
    WithPruneGVKs(gvks ...schema.GroupVersionKind) Apply
    WithPruneTypes(types ...client.Object) Apply
    WithNoPrune() Apply
    FindOwner(ctx context.Context, obj client.Object) (client.Object, error)
    PurgeOrphan(ctx context.Context, obj client.Object) error
}
```

#### Methods

##### Apply

```go
func Apply(ctx context.Context, owner client.Object, objs ...client.Object) error
```

Applies desired state for resources, setting the owner reference.

**Parameters:**

- `ctx` - Context for the operation
- `owner` - Owner object (controller resource)
- `objs` - Desired resources to create/update

**Returns:**

- `error` - Error if apply fails

**Behavior:**

- Creates resources that don't exist
- Updates resources with changes
- Prunes resources no longer desired (unless `WithNoPrune()`)
- Sets owner references on all resources
- Uses three-way merge for updates

**Example:**

```go
a := apply.New(client)

desiredResources := []client.Object{
    &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "my-config",
            Namespace: owner.Namespace,
        },
        Data: map[string]string{
            "key": "value",
        },
    },
    &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "my-service",
            Namespace: owner.Namespace,
        },
        Spec: corev1.ServiceSpec{
            Selector: map[string]string{"app": "my-app"},
            Ports: []corev1.ServicePort{
                {Port: 80, TargetPort: intstr.FromInt(8080)},
            },
        },
    },
}

if err := a.Apply(ctx, owner, desiredResources...); err != nil {
    return fmt.Errorf("apply failed: %w", err)
}
```

##### Ensure

```go
func Ensure(ctx context.Context, obj ...client.Object) error
```

Ensures resources exist without setting owner references or pruning.

**Parameters:**

- `ctx` - Context for the operation
- `obj` - Resources to ensure

**Returns:**

- `error` - Error if ensure fails

**Behavior:**

- Creates resources if they don't exist
- Updates resources if they exist
- Does NOT set owner references
- Does NOT prune resources

**Use Cases:**

- Creating cluster-scoped resources
- Ensuring prerequisites
- Resources without lifecycle coupling

**Example:**

```go
a := apply.New(client)

namespace := &corev1.Namespace{
    ObjectMeta: metav1.ObjectMeta{
        Name: "my-namespace",
    },
}

if err := a.Ensure(ctx, namespace); err != nil {
    return err
}
```

##### WithOwnerSubContext

```go
func WithOwnerSubContext(ownerSubContext string) Apply
```

Sets a subcontext for owner references, allowing multiple controllers to manage resources.

**Parameters:**

- `ownerSubContext` - Subcontext identifier

**Returns:**

- `Apply` - Apply instance for chaining

**Use Case:**

- Multiple controllers managing different aspects of same resource
- Separating concerns within a controller

**Example:**

```go
a := apply.New(client).
    WithOwnerSubContext("networking")

if err := a.Apply(ctx, owner, networkResources...); err != nil {
    return err
}
```

##### WithNamespace

```go
func WithNamespace(ns string) Apply
```

Sets the default namespace for resources without explicit namespace.

**Parameters:**

- `ns` - Namespace name

**Returns:**

- `Apply` - Apply instance for chaining

**Example:**

```go
a := apply.New(client).
    WithNamespace("default")

// ConfigMap will be created in "default" namespace
configMap := &corev1.ConfigMap{
    ObjectMeta: metav1.ObjectMeta{
        Name: "my-config",
        // Namespace not set, will use "default"
    },
}

if err := a.Apply(ctx, owner, configMap); err != nil {
    return err
}
```

##### WithPruneGVKs

```go
func WithPruneGVKs(gvks ...schema.GroupVersionKind) Apply
```

Limits pruning to specific GroupVersionKinds.

**Parameters:**

- `gvks` - GVKs to consider for pruning

**Returns:**

- `Apply` - Apply instance for chaining

**Example:**

```go
a := apply.New(client).
    WithPruneGVKs(
        schema.GroupVersionKind{Group: "", Version: "v1", Kind: "ConfigMap"},
        schema.GroupVersionKind{Group: "", Version: "v1", Kind: "Service"},
    )

// Only ConfigMaps and Services will be pruned
if err := a.Apply(ctx, owner, desiredResources...); err != nil {
    return err
}
```

##### WithPruneTypes

```go
func WithPruneTypes(types ...client.Object) Apply
```

Limits pruning to specific resource types (convenience method).

**Parameters:**

- `types` - Resource type examples

**Returns:**

- `Apply` - Apply instance for chaining

**Example:**

```go
a := apply.New(client).
    WithPruneTypes(&corev1.ConfigMap{}, &corev1.Secret{})

// Only ConfigMaps and Secrets will be pruned
if err := a.Apply(ctx, owner, desiredResources...); err != nil {
    return err
}
```

##### WithNoPrune

```go
func WithNoPrune() Apply
```

Disables pruning of resources.

**Returns:**

- `Apply` - Apply instance for chaining

**Use Case:**

- Additive-only resource management
- Preserving manually created resources
- Troubleshooting

**Example:**

```go
a := apply.New(client).WithNoPrune()

// Resources will be created/updated but never deleted
if err := a.Apply(ctx, owner, desiredResources...); err != nil {
    return err
}
```

##### FindOwner

```go
func FindOwner(ctx context.Context, obj client.Object) (client.Object, error)
```

Finds the owner of a resource.

**Parameters:**

- `ctx` - Context for the operation
- `obj` - Resource to find owner for

**Returns:**

- `client.Object` - Owner object
- `error` - Error if owner not found or lookup fails

**Example:**

```go
a := apply.New(client)

owner, err := a.FindOwner(ctx, configMap)
if err != nil {
    return err
}
fmt.Printf("Owner: %s/%s\n", owner.GetNamespace(), owner.GetName())
```

##### PurgeOrphan

```go
func PurgeOrphan(ctx context.Context, obj client.Object) error
```

Deletes a resource that no longer has a valid owner.

**Parameters:**

- `ctx` - Context for the operation
- `obj` - Orphaned resource to delete

**Returns:**

- `error` - Error if purge fails

**Example:**

```go
a := apply.New(client)

if err := a.PurgeOrphan(ctx, orphanedResource); err != nil {
    return err
}
```

---

## Functions

### New

```go
func New(c client.Client) Apply
```

Creates a new Apply instance.

**Parameters:**

- `c` - Kubernetes client

**Returns:**

- `Apply` - Apply interface implementation

**Example:**

```go
a := apply.New(backend.Client())
```

### Ensure (package function)

```go
func Ensure(ctx context.Context, client client.Client, obj ...client.Object) error
```

Convenience function for ensuring resources without creating Apply instance.

**Parameters:**

- `ctx` - Context for the operation
- `client` - Kubernetes client
- `obj` - Resources to ensure

**Returns:**

- `error` - Error if ensure fails

**Example:**

```go
if err := apply.Ensure(ctx, client, namespace, clusterRole); err != nil {
    return err
}
```

### AddValidOwnerChange

```go
func AddValidOwnerChange(oldSubcontext, newSubContext string)
```

Registers a valid owner subcontext migration path.

**Parameters:**

- `oldSubcontext` - Old subcontext identifier
- `newSubContext` - New subcontext identifier

**Use Case:**

- Controller refactoring
- Subcontext migrations
- Ownership handoff

**Example:**

```go
// Allow migration from "old-controller" to "new-controller"
apply.AddValidOwnerChange("old-controller", "new-controller")
```

---

## Concepts

### Owner References

Apply automatically manages Kubernetes owner references:

```yaml
metadata:
  ownerReferences:
  - apiVersion: v1
    kind: MyResource
    name: owner-name
    uid: owner-uid
    controller: true
    blockOwnerDeletion: true
```

**Benefits:**

- Automatic garbage collection
- Ownership tracking
- Cascade deletion

**Best Practices:**

- Use Apply for managed resources
- Use Ensure for independent resources
- Namespace-scoped owners can only own namespace-scoped resources

### Pruning

Pruning automatically deletes resources no longer in desired state:

**Enabled by default:**

```go
// Old resources will be pruned
a.Apply(ctx, owner, currentDesiredResources...)
```

**Selective pruning:**

```go
// Only prune ConfigMaps and Secrets
a.WithPruneTypes(&corev1.ConfigMap{}, &corev1.Secret{}).
    Apply(ctx, owner, desiredResources...)
```

**Disabled:**

```go
// No pruning
a.WithNoPrune().Apply(ctx, owner, desiredResources...)
```

**Pruning Logic:**

1. List resources with owner reference
2. Compare with desired state
3. Delete resources not in desired state

### Subcontexts

Subcontexts enable multiple controllers or concerns to manage resources:

```go
// Networking controller
networkApply := apply.New(client).WithOwnerSubContext("networking")
networkApply.Apply(ctx, owner, services, ingresses...)

// Storage controller
storageApply := apply.New(client).WithOwnerSubContext("storage")
storageApply.Apply(ctx, owner, pvcs, configMaps...)
```

**Owner Reference Format:**

```yaml
metadata:
  annotations:
    objectset.obot.io/owner-gvk: "apps/v1, Kind=Deployment"
    objectset.obot.io/owner-name: "my-deployment"
    objectset.obot.io/owner-namespace: "default"
    objectset.obot.io/id: "networking"  # Subcontext
```

---

## Usage Examples

### Basic Apply Pattern

```go
type MyController struct {
    client client.Client
}

func (c *MyController) Reconcile(ctx context.Context, obj *MyResource) error {
    a := apply.New(c.client)

    // Define desired state
    desired := []client.Object{
        &corev1.ConfigMap{
            ObjectMeta: metav1.ObjectMeta{
                Name:      obj.Name + "-config",
                Namespace: obj.Namespace,
            },
            Data: obj.Spec.ConfigData,
        },
        &appsv1.Deployment{
            ObjectMeta: metav1.ObjectMeta{
                Name:      obj.Name,
                Namespace: obj.Namespace,
            },
            Spec: appsv1.DeploymentSpec{
                // ... deployment spec
            },
        },
    }

    // Apply desired state
    return a.Apply(ctx, obj, desired...)
}
```

### Multi-Resource Application

```go
func (c *Controller) createInfrastructure(ctx context.Context, app *App) error {
    a := apply.New(c.client)

    resources := []client.Object{}

    // Add ConfigMap
    resources = append(resources, &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      app.Name + "-config",
            Namespace: app.Namespace,
        },
        Data: app.Spec.Config,
    })

    // Add Secret
    resources = append(resources, &corev1.Secret{
        ObjectMeta: metav1.ObjectMeta{
            Name:      app.Name + "-secret",
            Namespace: app.Namespace,
        },
        StringData: app.Spec.Secrets,
    })

    // Add Service
    resources = append(resources, &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      app.Name,
            Namespace: app.Namespace,
        },
        Spec: corev1.ServiceSpec{
            Selector: app.Spec.Selector,
            Ports:    app.Spec.Ports,
        },
    })

    // Add Deployment
    resources = append(resources, &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      app.Name,
            Namespace: app.Namespace,
        },
        Spec: buildDeploymentSpec(app),
    })

    return a.Apply(ctx, app, resources...)
}
```

### Selective Pruning

```go
func (c *Controller) reconcileNetworking(ctx context.Context, obj *MyResource) error {
    a := apply.New(c.client).
        WithOwnerSubContext("networking").
        WithPruneTypes(&corev1.Service{}, &networkingv1.Ingress{})

    networkResources := buildNetworkResources(obj)

    // Only Services and Ingresses will be pruned
    // ConfigMaps, Secrets, etc. managed by other subcontexts remain
    return a.Apply(ctx, obj, networkResources...)
}
```

### Ensuring Prerequisites

```go
func (c *Controller) ensurePrerequisites(ctx context.Context) error {
    a := apply.New(c.client)

    prerequisites := []client.Object{
        // Namespace
        &corev1.Namespace{
            ObjectMeta: metav1.ObjectMeta{
                Name: "my-app",
            },
        },
        // ClusterRole
        &rbacv1.ClusterRole{
            ObjectMeta: metav1.ObjectMeta{
                Name: "my-app-role",
            },
            Rules: []rbacv1.PolicyRule{
                // ... rules
            },
        },
    }

    // Ensure without owner references
    return a.Ensure(ctx, prerequisites...)
}
```

### Complex Ownership Example

```go
func (c *Controller) multiTierReconcile(ctx context.Context, obj *MyResource) error {
    // Database resources
    dbApply := apply.New(c.client).
        WithOwnerSubContext("database").
        WithPruneTypes(&corev1.Secret{}, &appsv1.StatefulSet{})

    if err := dbApply.Apply(ctx, obj, c.buildDatabaseResources(obj)...); err != nil {
        return err
    }

    // Application resources
    appApply := apply.New(c.client).
        WithOwnerSubContext("application").
        WithPruneTypes(&corev1.ConfigMap{}, &appsv1.Deployment{}, &corev1.Service{})

    if err := appApply.Apply(ctx, obj, c.buildAppResources(obj)...); err != nil {
        return err
    }

    // Networking resources
    netApply := apply.New(c.client).
        WithOwnerSubContext("networking").
        WithPruneTypes(&networkingv1.Ingress{})

    return netApply.Apply(ctx, obj, c.buildNetworkResources(obj)...)
}
```

---

## Best Practices

### 1. Use Apply for Managed Resources

Resources that should be created, updated, and deleted by your controller:

```go
a.Apply(ctx, owner, managedResources...)
```

### 2. Use Ensure for Independent Resources

Resources that should exist but aren't lifecycle-coupled:

```go
a.Ensure(ctx, namespaces, clusterRoles...)
```

### 3. Use Subcontexts for Separation

Separate concerns within a controller:

```go
dbApply := a.WithOwnerSubContext("database")
appApply := a.WithOwnerSubContext("application")
```

### 4. Use Selective Pruning

Only prune resources you manage:

```go
a.WithPruneTypes(&corev1.ConfigMap{}, &corev1.Secret{})
```

### 5. Handle Errors Appropriately

```go
if err := a.Apply(ctx, owner, resources...); err != nil {
    return fmt.Errorf("failed to apply resources: %w", err)
}
```

---

## See Also

- [Router Package](router.md) - Using Apply in handlers
- [Backend Package](backend.md) - Client operations
- [DesiredSet](desiredset.md) - Lower-level apply implementation

---

**Package Documentation:** [pkg.go.dev/github.com/obot-platform/nah/pkg/apply](https://pkg.go.dev/github.com/obot-platform/nah/pkg/apply)
