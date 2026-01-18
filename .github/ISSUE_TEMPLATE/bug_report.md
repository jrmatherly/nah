---
name: Bug report
about: Create a report to help us improve nah
title: ''
labels: bug
assignees: ''

---

## Describe the Bug

A clear and concise description of what the bug is.

## To Reproduce

Steps to reproduce the behavior:

1. Initialize router with '...'
2. Register handler for '...'
3. Trigger event '...'
4. See error

**Minimal Reproduction Code:**

```go
// Paste minimal code that reproduces the issue
package main

import (
    "github.com/obot-platform/nah"
    // ...
)

func main() {
    // Your reproduction code here
}
```

## Expected Behavior

A clear and concise description of what you expected to happen.

## Actual Behavior

What actually happened? Include error messages, stack traces, or unexpected output.

```
# Paste error messages, logs, or stack traces here
```

## Environment

**nah version:** <!-- e.g., v0.5.2 or commit SHA -->
**Go version:** <!-- e.g., 1.25.0 -->
**Kubernetes version:** <!-- e.g., 1.28.0 -->
**controller-runtime version:** <!-- e.g., v0.19.0 -->
**Operating System:** <!-- e.g., macOS 14, Ubuntu 22.04 -->

## Controller Configuration

<!-- If applicable, describe your controller setup -->

**Resources being watched:**

- <!-- e.g., ConfigMaps in namespace "default" -->

**Leader election:** <!-- enabled/disabled -->
**Custom options:** <!-- any non-default Options passed to nah.NewRouter -->

## Additional Context

Add any other context about the problem here. For example:

- Does this happen consistently or intermittently?
- Did this work in a previous version? If so, which version?
- Are there any workarounds you've found?
- Related upstream issues (controller-runtime, client-go, etc.)

## Logs

<details>
<summary>Click to expand controller logs</summary>

```
# Paste relevant controller logs here
```

</details>

## Possible Solution

<!-- Optional: If you have ideas on how to fix this, share them here -->
