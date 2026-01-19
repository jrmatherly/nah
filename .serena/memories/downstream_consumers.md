# Downstream Consumers of nah

## Overview

**nah** is a Kubernetes controller framework library with downstream projects that depend on it. Changes to nah's API can impact these consumers, so coordination and testing are essential.

## Known Downstream Projects

### obot-entraid

**Repository**: https://github.com/jrmatherly/obot-entraid

**Relationship**: Primary consumer of nah framework

**Usage**:

- **Controllers**: 40+ Kubernetes controllers managing MCP servers, workflows, agents, knowledge management
- **Core imports**:
  - `github.com/obot-platform/nah/pkg/router` - Event routing
  - `github.com/obot-platform/nah/pkg/apply` - Declarative resource management
  - `github.com/obot-platform/nah/pkg/backend` - Kubernetes client abstraction
  - `github.com/obot-platform/nah/pkg/leader` - Leader election
  - `github.com/obot-platform/nah/pkg/handlers` - Common handlers

**Current Version**: Pseudo-version from main branch (e.g., `v0.0.0-20250418220644-1b9278409317`)

**Integration Points**:

- Router initialization in `pkg/services/config.go`
- Route registration in `pkg/controller/routes.go`
- Handler implementations in `pkg/controller/handlers/`
- Apply usage throughout controllers for declarative reconciliation

**Maintainer**: Same as nah (jrmatherly)

### Other Potential Consumers

**obot-platform/obot** (upstream): The upstream Obot project also uses nah and should be considered when making changes.

## Breaking Change Policy

### API Stability Guidelines

Since nah is a framework library with downstream consumers, follow these principles:

1. **Avoid Breaking Changes**: Minimize API changes that break existing code
2. **Deprecation Period**: Deprecate APIs before removal (at least one release cycle)
3. **Communication**: Document breaking changes clearly in commit messages and releases
4. **Coordination**: Test with obot-entraid before releasing breaking changes

### What Constitutes a Breaking Change

**Breaking changes include**:

- Removing exported functions, types, or methods
- Changing function signatures (parameters, return types)
- Renaming exported identifiers
- Changing behavior that consumers depend on
- Modifying interface requirements

**Non-breaking changes include**:

- Adding new functions, types, or methods
- Adding new optional parameters (with defaults)
- Internal implementation changes
- Performance improvements
- Bug fixes that restore documented behavior

### Before Making Breaking Changes

1. **Check downstream impact**:
   - Search obot-entraid codebase for usage
   - Identify all locations that would break
   - Estimate migration effort

2. **Consider alternatives**:
   - Can the change be additive instead?
   - Can old API be deprecated instead of removed?
   - Can migration be automated?

3. **Document the change**:
   - Update CHANGELOG.md (if exists)
   - Add migration guide
   - Include examples of old vs new usage

4. **Coordinate with downstream maintainers**:
   - Since you maintain both projects, schedule updates together
   - Update nah first, then obot-entraid

## Testing with Downstream Projects

### Local Testing Workflow

**When modifying nah, always test with obot-entraid before pushing**:

1. **Make changes in nah**:

```bash
cd /path/to/nah
# Make code changes
go generate
go mod tidy
```

1. **Run nah tests**:

```bash
make validate-ci   # Generate, tidy, check for uncommitted changes
make validate      # Run linters
make test          # Run all tests
```

1. **Test with obot-entraid using replace directive**:

```bash
cd /path/to/obot-entraid

# Add replace directive to go.mod
echo 'replace github.com/obot-platform/nah => ../nah' >> go.mod

# Update dependencies
go mod tidy

# Run tests
make test

# Test manually
make dev

# Verify controllers reconcile correctly
export KUBECONFIG=tools/devmode-kubeconfig
kubectl get agents
kubectl get mcpservers
kubectl get threads
```

1. **Verify integration**:

```bash
# Create test resources
kubectl --kubeconfig tools/devmode-kubeconfig apply -f - <<EOF
apiVersion: obot.obot.ai/v1
kind: Agent
metadata:
  name: test-agent
  namespace: default
spec:
  manifest:
    name: test-agent
EOF

# Check reconciliation
kubectl --kubeconfig tools/devmode-kubeconfig get agent test-agent -o yaml

# Watch logs for errors
# (make dev outputs to terminal)
```

1. **Clean up replace directive**:

```bash
# Remove replace directive from go.mod
# (Never commit replace directives!)
```

### CI/CD Considerations

**nah CI** (`.github/workflows/test.yaml`):

- Runs on every push
- Lints, tests, validates generated code
- Does NOT test with downstream projects

**Recommendation**: Consider adding a downstream integration test job:

```yaml
downstream-test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
      with:
        path: nah
    - uses: actions/checkout@v4
      with:
        repository: jrmatherly/obot-entraid
        path: obot-entraid
    - name: Test integration
      run: |
        cd obot-entraid
        echo 'replace github.com/obot-platform/nah => ../nah' >> go.mod
        go mod tidy
        make test
```

## Version Management

### Versioning Strategy

**Current approach**: Pseudo-versions (commit-based)

```
github.com/obot-platform/nah v0.0.0-20250418220644-1b9278409317
```

**Future consideration**: Semantic versioning

- `v0.x.y` for pre-1.0 (breaking changes allowed)
- `v1.x.y` for stable API (follow semver)

### Updating Downstream Projects

When nah is updated, obot-entraid must be updated:

```bash
cd /path/to/obot-entraid

# Update to latest nah commit
go get github.com/obot-platform/nah@latest

# Or specific commit
go get github.com/obot-platform/nah@<commit-hash>

# Tidy dependencies
go mod tidy

# Test
make test
make dev

# Commit
git add go.mod go.sum
git commit -m "chore: update nah to <commit-hash>"
```

## Communication Channels

### Commit Messages

Use conventional commits to indicate breaking changes:

```
feat: add new router option (non-breaking)
fix: correct apply behavior (non-breaking)
feat!: change handler signature (BREAKING CHANGE)
refactor!: rename Backend methods (BREAKING CHANGE)
```

### Documentation

When making significant changes:

1. Update CLAUDE.md with new patterns
2. Update PROJECT_INDEX.md if structure changes
3. Add examples in docs/ for new features
4. Update relevant memories (codebase_architecture, design_patterns_and_guidelines)

## Coordination Process

### For Non-Breaking Changes

1. Make changes in nah
2. Test locally with obot-entraid (replace directive)
3. Commit and push nah changes
4. Update obot-entraid dependency
5. Test and commit obot-entraid

### For Breaking Changes

1. **Plan the change**:
   - Document what breaks
   - Design migration path
   - Estimate effort

2. **Implement in nah**:
   - Make the breaking change
   - Update all examples in nah
   - Test nah standalone

3. **Test with obot-entraid**:
   - Use replace directive
   - Fix all compilation errors
   - Update all usage sites
   - Test thoroughly

4. **Coordinate releases**:
   - Commit nah changes with clear breaking change notice
   - Push nah changes
   - Update obot-entraid immediately
   - Push obot-entraid changes

5. **Document**:
   - Add migration guide
   - Update both projects' documentation
   - Note breaking change in commit/PR

## Best Practices for nah Development

### DO

✅ Test all changes with obot-entraid before pushing
✅ Use conventional commits to indicate breaking changes
✅ Document new APIs with examples
✅ Maintain backward compatibility when possible
✅ Add deprecation notices before removing APIs
✅ Update CLAUDE.md and memories after significant changes

### DON'T

❌ Push breaking changes without testing downstream
❌ Remove APIs without deprecation period
❌ Change behavior without documentation
❌ Commit replace directives
❌ Ignore downstream compilation errors
❌ Skip integration testing

## Quick Reference

### Test with obot-entraid

```bash
# In obot-entraid
echo 'replace github.com/obot-platform/nah => ../nah' >> go.mod
go mod tidy
make test
make dev
# Remember to remove replace directive!
```

### Update obot-entraid dependency

```bash
cd /path/to/obot-entraid
go get github.com/obot-platform/nah@latest
go mod tidy
make test
```

### Check for downstream usage

```bash
cd /path/to/obot-entraid
grep -r "nah/pkg" . --include="*.go"
```

### Verify no replace directives

```bash
grep -r "replace.*nah" go.mod
# Should return nothing before committing
```

## Future Considerations

### Potential Improvements

1. **Semantic versioning**: Move to v0.x.y tags for clearer version tracking
2. **Automated testing**: CI job that tests with obot-entraid
3. **Changelog**: Maintain CHANGELOG.md for release notes
4. **Deprecation policy**: Formal policy for API deprecation
5. **Migration guides**: Document for major version changes

### Community Growth

If nah gains more users outside the Obot ecosystem:

- Establish formal versioning and release process
- Create a CONTRIBUTING.md with API stability guarantees
- Set up issue templates for bug reports and feature requests
- Consider a roadmap for v1.0 stable API
