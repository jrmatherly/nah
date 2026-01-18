# Debugging Workflows for nah

## Overview

This guide covers debugging nah framework itself and applications built with nah. Since nah is a library, debugging often involves testing with downstream projects like obot-entraid.

## Debug Logging

### Enable Debug Logging in Applications

Applications using nah can enable debug logging:

```go
import (
    "github.com/sirupsen/logrus"
    _ "github.com/obot-platform/nah/pkg/logrus"  // Auto-configures logrus
)

func main() {
    // Set log level
    logrus.SetLevel(logrus.DebugLevel)
    
    // Or via environment
    // LOG_LEVEL=debug ./myapp
}
```

### Structured Logging

nah uses logrus for structured logging:

```go
import log "github.com/sirupsen/logrus"

log.WithFields(log.Fields{
    "resource": obj.GetName(),
    "gvk":      gvk.String(),
}).Debug("handling resource")
```

## Debugging Router Issues

### Check Router Started

**In application logs**:

```
router <name> started
```

**Debug startup issues**:

```go
router, err := nah.NewRouter("my-controller", opts)
if err != nil {
    log.Fatalf("failed to create router: %v", err)
}

// Add debug logging
log.WithField("router", "my-controller").Info("starting router")

if err := router.Start(ctx); err != nil {
    log.Fatalf("failed to start router: %v", err)
}
```

### Check Handler Registration

**Verify handlers registered**:

```go
// In your application
router.Type(&MyResource{}).HandlerFunc(func(req router.Request, resp router.Response) error {
    log.WithField("resource", req.Object.GetName()).Debug("handler called")
    return nil
})

// Logs should show:
// registered handler for <GVK>
```

### Debug Handler Not Triggering

**Possible causes**:

1. Handler not registered
2. Resource type mismatch (GVK different)
3. Namespace filter excluding resource
4. Selector filter not matching
5. Leader election blocking (not leader)

**Debug steps**:

```go
// Log handler registration
router.Type(&MyResource{}).
    HandlerFunc(func(req router.Request, resp router.Response) error {
        log.Info("handler called!")  // Add this
        return nil
    })

// Check GVK matches
gvk := MyResource{}.GetObjectKind().GroupVersionKind()
log.WithField("gvk", gvk).Info("expecting events for GVK")

// Check filters
router.Type(&MyResource{}).
    Namespace("default").  // Only watches default namespace
    HandlerFunc(handler)

// Try without filters to test
router.Type(&MyResource{}).HandlerFunc(handler)  // No filters
```

### Debug Infinite Reconciliation Loops

**Symptom**: Handler called repeatedly, never reaches steady state

**Common causes**:

1. Handler modifies resource on every call (not idempotent)
2. Handler doesn't check current state before updating
3. Status update triggers reconciliation
4. Another controller conflicting

**Debug**:

```go
func (h *Handler) Handle(req router.Request, resp router.Response) error {
    resource := req.Object.(*MyResource)
    
    // Log entry
    log.WithFields(log.Fields{
        "name":            resource.Name,
        "resourceVersion": resource.ResourceVersion,
        "generation":      resource.Generation,
    }).Debug("reconciling resource")
    
    // Check if work needed
    if resource.Status.Phase == "Ready" {
        log.Debug("already ready, skipping")
        return nil  // Don't update if already done
    }
    
    // Do work...
    
    return nil
}
```

**Make handlers idempotent**:

- Check current state first
- Only update if needed
- Use Apply for declarative updates (automatically idempotent)

## Debugging Apply Issues

### Debug Apply Not Creating Resources

**Check desired resources**:

```go
a := apply.New(backend)

desiredResources := []client.Object{
    &corev1.ConfigMap{...},
}

log.WithField("count", len(desiredResources)).Debug("applying resources")

err := a.Apply(ctx, owner, desiredResources...)
if err != nil {
    log.WithError(err).Error("apply failed")
    return err
}

log.Debug("apply succeeded")
```

**Common issues**:

- Desired resources slice empty
- Owner reference nil
- Namespace not set
- Invalid object (missing required fields)

### Debug Apply Not Pruning

**Pruning enabled by default in Apply()**, disabled in Ensure()

```go
// This prunes
a.Apply(ctx, owner, desiredResources...)

// This doesn't prune
a.Ensure(ctx, desiredResources...)

// Check pruning behavior
log.WithField("prune", "enabled").Debug("using Apply with pruning")
```

**Why not pruning**:

- Using Ensure() instead of Apply()
- Owner reference not set
- WithNoPrune() called
- Resource has different owner
- Subcontext mismatch

### Debug Apply Updates Not Working

**Check resource comparison**:

Apply uses three-way merge:

1. Last applied configuration (annotation)
2. Current state (from API)
3. Desired state (from Apply call)

```go
// Apply adds annotation
// kubectl.kubernetes.io/last-applied-configuration

// Check if annotation present
log.WithField("annotations", obj.GetAnnotations()).Debug("resource annotations")
```

**Common issues**:

- Resource modified out-of-band (without Apply)
- Last applied annotation removed
- Immutable fields changed
- Resource owned by different controller

## Debugging Backend Issues

### Debug Get/List Operations

```go
backend := resp.Backend()

var obj MyResource
key := client.ObjectKey{Namespace: "default", Name: "test"}

log.WithField("key", key).Debug("getting resource")

err := backend.Get(ctx, key, &obj)
if err != nil {
    if apierrors.IsNotFound(err) {
        log.Debug("resource not found")
    } else {
        log.WithError(err).Error("get failed")
    }
    return err
}

log.WithField("resourceVersion", obj.ResourceVersion).Debug("got resource")
```

### Debug Create/Update Operations

```go
obj := &MyResource{...}

log.WithFields(log.Fields{
    "name":      obj.Name,
    "namespace": obj.Namespace,
}).Debug("creating resource")

err := backend.Create(ctx, obj)
if err != nil {
    if apierrors.IsAlreadyExists(err) {
        log.Debug("resource already exists")
        // Try update instead
    } else {
        log.WithError(err).Error("create failed")
    }
    return err
}

log.WithField("resourceVersion", obj.ResourceVersion).Debug("created resource")
```

### Debug Conflict Errors

**Symptom**: `the object has been modified; please apply your changes to the latest version and try again`

**Cause**: Resource modified between Get and Update (optimistic locking)

**Debug**:

```go
var obj MyResource
key := client.ObjectKey{Namespace: "default", Name: "test"}

// Get latest version
err := backend.Get(ctx, key, &obj)
if err != nil {
    return err
}

log.WithField("resourceVersion", obj.ResourceVersion).Debug("got resource")

// Modify
obj.Spec.Field = "new-value"

// Update
err = backend.Update(ctx, &obj)
if err != nil {
    if apierrors.IsConflict(err) {
        log.WithField("resourceVersion", obj.ResourceVersion).Warn("conflict, retrying")
        // Retry from Get
    } else {
        log.WithError(err).Error("update failed")
    }
    return err
}
```

**Solution**: Implement retry with exponential backoff

## Debugging Leader Election

### Check Leader Status

```go
// In application using leader election
if elector != nil {
    // Check if leading (not exposed in current API)
    log.Info("leader election enabled")
} else {
    log.Info("leader election disabled")
}
```

**In logs**:

```
became leader
stopped leading
```

### Debug Leader Election Blocking

**Symptom**: Router starts but handlers never called

**Cause**: Not winning leader election

**Debug**:

1. Check ElectionConfig not nil
2. Check lease not held by another process
3. Check TTL not too long

```go
config := leader.NewDefaultElectionConfig("", "my-controller", restConfig)
config.LeaseDuration = 15 * time.Second  // Default
config.RenewDeadline = 10 * time.Second
config.RetryPeriod = 2 * time.Second

log.WithFields(log.Fields{
    "leaseDuration": config.LeaseDuration,
    "lockName":      config.LockName,
}).Info("starting leader election")
```

### Debug Multiple Leaders

**Symptom**: Conflicting updates, race conditions

**Cause**: Multiple processes winning leadership (shouldn't happen)

**Debug**:

1. Check using same lock name
2. Check using same namespace
3. Check coordination lease resource

```bash
# Check lease
kubectl get lease <controller-name> -n <namespace> -o yaml

# Should have:
# holderIdentity: <pod-name>
# acquireTime: <timestamp>
```

## Debugging Watcher Issues

### Debug Trigger Not Firing

**Symptom**: Resource created but handler not called

**Debug**:

```go
// Check watcher started
log.Info("starting watcher")

// Check trigger called
err := backend.Trigger(ctx, gvk, key, 0)
if err != nil {
    log.WithError(err).Error("trigger failed")
}

// Verify trigger map
// (internal to nah, inspect via logging)
```

### Debug Memory Leak in Triggers

**Symptom**: Memory grows over time

**Cause**: Trigger map entries not cleaned up

**Debug**:

```go
// Add logging to track trigger map size
// (requires modifying nah or adding metrics)

// Workaround: Monitor memory usage
runtime.ReadMemStats(&m)
log.WithField("alloc", m.Alloc).Debug("memory usage")
```

**Solution**: Ensure resources are deleted properly

## Debugging Tests

### Run Single Test with Logging

```bash
# Run specific test with verbose output
go test -v -run TestRouterStart ./pkg/router

# With debug logging
LOG_LEVEL=debug go test -v -run TestRouterStart ./pkg/router
```

### Debug Test Failures

```go
func TestSomething(t *testing.T) {
    // Add debug output
    t.Logf("Starting test")
    
    result := doSomething()
    
    t.Logf("Result: %+v", result)  // Only shows on failure or with -v
    
    if result.Err != nil {
        t.Fatalf("unexpected error: %v", result.Err)
    }
}
```

### Debug Flaky Tests

**Symptoms**: Tests pass sometimes, fail sometimes

**Common causes**:

1. Race conditions (run with `-race`)
2. Timing dependencies (use proper synchronization)
3. Shared state between tests
4. External dependencies (mock them)

**Debug**:

```bash
# Run test multiple times
go test -v -run TestFlaky -count=100 ./pkg/router

# With race detector
go test -race -v -run TestFlaky ./pkg/router

# With shuffle (randomize test order)
go test -shuffle=on -v ./pkg/router
```

## Debugging with Downstream Projects

### Test nah Changes with obot-entraid

**Primary integration test** - see `downstream_consumers` memory:

```bash
# In obot-entraid
echo 'replace github.com/obot-platform/nah => ../nah' >> go.mod
go mod tidy

# Run tests
make test

# Run dev mode
make dev

# Check logs
make dev | grep nah

# Verify controllers work
export KUBECONFIG=tools/devmode-kubeconfig
kubectl get agents
kubectl get mcpservers
```

### Debug nah API Changes

**When making changes to nah that break obot-entraid**:

1. **Identify breakage**:

   ```bash
   cd obot-entraid
   go build ./...
   # Shows compilation errors
   ```

2. **Fix each usage site**:

   ```bash
   # Search for usage
   grep -r "nah/pkg/router" . --include="*.go"
   grep -r "nah/pkg/apply" . --include="*.go"
   ```

3. **Test incrementally**:

   ```bash
   # After fixing package
   go test ./pkg/controller/handlers/mcpserver/
   ```

4. **Verify end-to-end**:

   ```bash
   make dev
   # Test controllers reconcile correctly
   ```

## Performance Debugging

### Profile CPU Usage

```bash
# In application using nah
go build -o myapp
./myapp -cpuprofile=cpu.prof

# Analyze
go tool pprof cpu.prof
(pprof) top10
(pprof) list <function-name>
```

### Profile Memory Usage

```bash
./myapp -memprofile=mem.prof

go tool pprof mem.prof
(pprof) top10
(pprof) list <function-name>
```

### Check Goroutine Leaks

```go
// In application
import "runtime"

func checkGoroutines() {
    log.WithField("goroutines", runtime.NumGoroutine()).Info("goroutine count")
}

// Call periodically
ticker := time.NewTicker(10 * time.Second)
go func() {
    for range ticker.C {
        checkGoroutines()
    }
}()
```

**Expected**: Stable count after startup (< 100 typically)

**Leak**: Count grows continuously

## Using Debugger

### Delve Debugger

```bash
# Install
go install github.com/go-delve/delve/cmd/dlv@latest

# Debug application
dlv exec ./myapp -- --args

# Debug test
dlv test ./pkg/router -- -test.run TestRouterStart

# Set breakpoint
(dlv) break router.go:42
(dlv) continue

# Inspect variables
(dlv) print req
(dlv) print resp.Backend()

# Step through
(dlv) next   # Step over
(dlv) step   # Step into
(dlv) stepout  # Step out
```

### IDE Debugging

**GoLand/IntelliJ**:

1. Set breakpoints (click gutter)
2. Run in Debug mode
3. Inspect variables (hover or Evaluate Expression)
4. Step through code (F7/F8)

**VS Code**:

1. Install Go extension
2. Set breakpoints
3. F5 to debug
4. Use Debug Console to evaluate expressions

## Common Issues and Solutions

### Issue: Import Cycle

**Symptom**: `import cycle not allowed`

**Cause**: Package A imports B, B imports A

**Solution**: Refactor to break cycle (extract interface, move code)

### Issue: Generated Code Out of Sync

**Symptom**: Compilation errors after updating CRDs

**Solution**:

```bash
go generate ./...
go mod tidy
```

### Issue: Dependency Conflicts

**Symptom**: `go mod tidy` fails or wrong versions

**Solution**:

```bash
go mod tidy
go mod verify

# If still broken
rm go.sum
go mod tidy
```

### Issue: Test Data Race

**Symptom**: `go test -race` reports data race

**Solution**: Fix shared state, add proper locking

```go
// Before (data race)
var counter int
go func() { counter++ }()
go func() { counter++ }()

// After (fixed)
var (
    mu      sync.Mutex
    counter int
)
go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()
```

## Quick Reference

### Enable Debug Logging

```go
logrus.SetLevel(logrus.DebugLevel)
// Or: LOG_LEVEL=debug ./myapp
```

### Run Tests

```bash
go test -v ./pkg/router              # Specific package
go test -v -run TestFunc ./...       # Specific test
go test -race ./...                  # With race detector
LOG_LEVEL=debug go test -v ./...     # With debug logs
```

### Test with obot-entraid

```bash
cd obot-entraid
echo 'replace github.com/obot-platform/nah => ../nah' >> go.mod
go mod tidy
make test
make dev
```

### Profile Performance

```bash
./myapp -cpuprofile=cpu.prof
go tool pprof cpu.prof
```

### Debug with Delve

```bash
dlv test ./pkg/router -- -test.run TestFunc
(dlv) break router.go:42
(dlv) continue
```

## Debugging Checklist

When encountering an issue:

- [ ] Enable debug logging
- [ ] Check router started
- [ ] Verify handlers registered
- [ ] Test handler in isolation
- [ ] Check filters and selectors
- [ ] Verify GVK matches
- [ ] Test with race detector
- [ ] Check for goroutine leaks
- [ ] Test with downstream project (obot-entraid)
- [ ] Add debug logging to narrow down issue
- [ ] Use debugger to step through code
- [ ] Check generated code up to date
- [ ] Verify dependencies correct

## Getting Help

If stuck after following these workflows:

1. **Check existing issues**: https://github.com/obot-platform/nah/issues
2. **Review documentation**: CLAUDE.md, README.md, docs/
3. **Test with examples**: Create minimal reproduction
4. **Check downstream**: May be obot-entraid issue, not nah
5. **Ask for help**: Create detailed issue with:
   - Minimal code to reproduce
   - Expected vs actual behavior
   - Relevant logs with debug enabled
   - Go version, OS, etc.
