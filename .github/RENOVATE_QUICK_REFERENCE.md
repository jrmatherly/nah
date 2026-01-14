# Renovate Configuration - Quick Reference

**Project:** nah
**Last Updated:** 2026-01-14

---

## 🚀 Quick Commands

### View Dashboard
```bash
# Dashboard will appear as GitHub issue after first Renovate run
# Title: "Renovate Dashboard :robot:"
```

### Validate Configuration
```bash
cd /Users/jason/dev/AI/nah
cat .github/renovate.json | jq empty  # Syntax check
```

### Test Configuration Locally (if renovate-cli installed)
```bash
npx renovate --dry-run --platform=github
```

---

## 📋 Configuration Overview

### Rate Limits
- **Concurrent PRs:** 3
- **Hourly Limit:** 1 PR/hour
- **Schedule:** 10pm - 6am EST (non-security)
- **Security Updates:** Any time (bypasses schedule)

### Automerge Policy

| Package Type | Patch | Minor | Major | Digest |
| -------------- | ------- | ------- | ------- | -------- |
| Go packages | ✅ 3d | ❌ | ❌ | ✅ 3d |
| GitHub Actions | ✅ 3d | ✅ 3d | ❌ | ✅ 3d |
| Kubernetes | ❌ | ❌ | ❌ | ❌ |
| controller-runtime | ❌ | ❌ | ❌ | ❌ |
| Security updates | ✅ | ✅ | ✅ | ✅ |

Legend: ✅ = Auto-merge enabled, ❌ = Manual review required, 3d = 3 days minimum age

### Package Groups
1. **golang** - Go version updates
2. **kubernetes packages** - k8s.io + sigs.k8s.io
3. **opentelemetry packages** - go.opentelemetry.io
4. **testing packages** - testify, autogold
5. **logging packages** - logrus
6. **github-actions** - All GHA updates

---

## 🔧 Common Modifications

### Adjust Rate Limits
```json
"prConcurrentLimit": 5,  // Change from 3
"prHourlyLimit": 2        // Change from 1
```

### Change Schedule Window
```json
"schedule": [
  "after 8pm",   // Earlier start
  "before 8am"   // Later end
]
```

### Add New Package Group
```json
{
  "description": "Group new-package-family",
  "groupName": "new-package packages",
  "matchPackageNames": ["/^github.com/new-org//"]
}
```

### Block Specific Package Updates
```json
{
  "description": "Block problematic package",
  "matchPackageNames": ["github.com/org/package"],
  "enabled": false
}
```

### Disable Automerge for Package
```json
{
  "description": "Require manual review for package",
  "matchPackageNames": ["github.com/org/package"],
  "automerge": false
}
```

---

## 🏷️ Label Reference

### Type Labels (Automatic)
- `type/major` - Major version updates (breaking changes)
- `type/minor` - Minor version updates (new features)
- `type/patch` - Patch updates (bug fixes)
- `type/digest` - Digest/SHA updates

### Priority Labels
- `security` - Security vulnerability fixes
- `priority/high` - High priority updates
- `breaking-change` - Requires coordination
- `requires-coordination` - Needs downstream testing

### Other Labels
- `renovate` - All Renovate PRs (automatic)

---

## 🔒 Security Features

### Enabled Security Scanning
- ✅ OSV Vulnerability Alerts
- ✅ Renovate Vulnerability Alerts
- ✅ GitHub Action Digest Pinning
- ✅ Immediate Security Update PRs
- ✅ Auto-merge for Security Fixes

### Security Update Flow
1. Vulnerability detected by OSV/Renovate
2. PR created immediately (bypasses schedule)
3. Labeled: `security`, `priority/high`
4. Tests run automatically
5. **Auto-merged if tests pass**
6. Manual review if tests fail

---

## 🚨 Manual Review Required

### Always Requires Review
1. **Kubernetes major updates** (k8s.io, sigs.k8s.io)
   - Label: `breaking-change`, `requires-coordination`
   - Reason: Affects downstream consumers (obot-entraid)

2. **controller-runtime major updates**
   - Label: `breaking-change`, `requires-coordination`
   - Reason: Tied to K8s API versions

3. **Any major version bump** (X.0.0)
   - Label: `type/major`
   - Reason: Potential breaking changes

### Review Process
1. Check PR description for breaking changes
2. Review changelog/release notes
3. Test with obot-entraid (use replace directive)
4. Coordinate with downstream consumers
5. Merge or close with comment

---

## 📊 Dashboard Guide

### Finding the Dashboard
- Navigate to GitHub Issues
- Look for: "Renovate Dashboard :robot:"
- Contains: All open PRs, pending updates, rate limit status

### Dashboard Sections
1. **Rate Limited** - PRs blocked by rate limits
2. **Open** - Active PRs awaiting merge
3. **Pending Approval** - PRs awaiting review
4. **Pending Updates** - Detected but not yet PR'd
5. **Error** - Failed dependency updates

### Dashboard Actions
- Click PR link to review
- Use "Rebase" button to update PR
- Use "Retry" for failed updates
- Check "Rate Limit" status

---

## 🔄 Weekly Maintenance Tasks

### Monday Morning (4am EST)
- ✅ Lock file maintenance PR auto-created
- ✅ Auto-merged if no conflicts
- ✅ Check for conflicts requiring manual resolution

### Daily (10pm-6am EST)
- ✅ Dependency update PRs created
- ✅ Review major update PRs
- ✅ Monitor automerge status

### As Needed
- ✅ Review "breaking-change" PRs
- ✅ Coordinate K8s updates with obot-entraid
- ✅ Check security alerts

---

## 🐛 Troubleshooting

### Dashboard Not Appearing
**Symptom:** No Dashboard issue after 24 hours
**Fix:** Check repository settings → Allow issues
**Verify:** `:dependencyDashboard` in extends array

### Too Many PRs
**Symptom:** Overwhelmed by PRs
**Fix 1:** Reduce `prConcurrentLimit` (e.g., 2)
**Fix 2:** Reduce `prHourlyLimit` (e.g., 0)
**Fix 3:** Close unwanted PRs (Dashboard shows "ignored")

### PRs Outside Schedule
**Symptom:** PRs arriving during day
**Check 1:** Is it a security update? (bypasses schedule)
**Check 2:** Verify timezone is correct
**Check 3:** Check schedule syntax

### Automerge Not Working
**Symptom:** Patches not auto-merging
**Check 1:** Is package in K8s/controller-runtime block?
**Check 2:** Has 3-day minimum age passed?
**Check 3:** Are tests passing?
**Check 4:** Check automerge configuration

### Tests Failing on Update
**Symptom:** PR tests fail
**Fix 1:** Review breaking changes in update
**Fix 2:** Update code to handle changes
**Fix 3:** Close PR if incompatible
**Fix 4:** Add to block list if needed

---

## 📝 Commit Message Format

### Automatic Format
```
{type}({scope}): {message} ( {old} → {new} )
```

### Examples
```
feat(go): update module k8s.io/client-go ( v0.31.1 → v0.31.2 )
fix(go): update module github.com/stretchr/testify ( v1.8.0 → v1.8.1 )
feat(go)!: update module k8s.io/client-go ( v0.31.1 → v0.32.0 )
ci(github-action): update action actions/checkout ( v3 → v4 )
chore(go): update module hash ( abc123 → def456 )
```

### Type Mapping
- `major` → `feat(scope)!:`
- `minor` → `feat(scope):`
- `patch` → `fix(scope):`
- `digest` → `chore(scope):`
- `github-actions` → `ci(github-action):`

---

## 🔗 Coordination with obot-entraid

### When K8s Update Available
1. Create issue in obot-entraid: "nah K8s update to vX.Y.Z"
2. Test nah update locally with replace directive
3. Update obot-entraid's go.mod after nah merges
4. Coordinate timing for simultaneous updates

### Communication Template
```markdown
## nah Kubernetes Update

**Package:** k8s.io/client-go
**Version:** v0.31.1 → v0.32.0
**nah PR:** [link]
**Breaking Changes:** [list]

### Impact on obot-entraid
- [ ] Tested with replace directive
- [ ] Breaking changes identified
- [ ] Code changes required: [Yes/No]
- [ ] Timeline: [date]

### Action Items
- [ ] nah: Review and merge PR
- [ ] obot-entraid: Update dependency
- [ ] obot-entraid: Update code if needed
- [ ] Both: Run integration tests
```

---

## 📚 Additional Resources

### Documentation
- [Enhancement Summary](.github/RENOVATE_ENHANCEMENT_SUMMARY.md)
- [Validation Report](.github/RENOVATE_VALIDATION_REPORT.md)
- [Renovate Docs](https://docs.renovatebot.com/)

### Configuration Files
- Main config: `.github/renovate.json`
- Workflow: `.github/workflows/test.yaml`

### Related Projects
- obot-entraid: `../obot-entraid/renovate.json` (reference)

---

## 💡 Tips

### Best Practices
1. **Review Dashboard weekly** - Stay on top of updates
2. **Test major updates** - Use replace directive in obot-entraid
3. **Monitor security labels** - Prioritize security PRs
4. **Keep config documented** - Add comments for custom rules
5. **Communicate breaking changes** - Coordinate with downstream

### Performance
- Group related packages to reduce PR volume
- Use automerge for safe updates (patches)
- Schedule updates for off-hours
- Rate limit to prevent overwhelm

### Safety
- Disable automerge for critical packages
- Require manual review for majors
- Test with downstream before merge
- Monitor for regression

---

## 🆘 Getting Help

### Issues with Configuration
1. Check syntax: `cat .github/renovate.json | jq empty`
2. Review [Validation Report](.github/RENOVATE_VALIDATION_REPORT.md)
3. Check Renovate logs in Dashboard
4. Review [Renovate Docs](https://docs.renovatebot.com/)

### Issues with Updates
1. Review PR description and changelog
2. Check breaking changes
3. Test locally with replace directive
4. Ask in obot-entraid discussions

### Emergency Rollback
```bash
# Disable Renovate temporarily
git revert <commit-hash>
git push

# Or disable in GitHub Settings → Integrations → Renovate
```

---

*For comprehensive details, see [RENOVATE_ENHANCEMENT_SUMMARY.md](.github/RENOVATE_ENHANCEMENT_SUMMARY.md)*
