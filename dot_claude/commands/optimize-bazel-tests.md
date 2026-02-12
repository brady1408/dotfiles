# Bazel Test Selection Optimization

## Analysis Summary (Completed Oct 21, 2025)

### Current State
- **Bazel version**: 8.2.1 (Bzlmod)
- **Remote cache**: S3-backed via bazel-remote (quay.io/bazel-remote/bazel-remote:v2.6.0)
- **Cache backend**: `s3://gitlab-bazel-build-cache` (us-east-1, 100GB limit)
- **Services**: 43 independent microservices in monorepo
- **Total test targets**: ~13,526 processes per full run

### Key Finding: The Real Bottleneck

**Your cache is world-class, but you're testing everything on every PR.**

Even with 99% cache hits and 4.5 minute builds:
- All 43 services tested regardless of what changed
- 13,526 targets processed every time

**Optimization opportunity: Selective testing, not cache tuning**

### Goal
Reduce CI time from **4.5 minutes** to **30-60 seconds** for typical single-service changes by only testing affected services.

### Strategy

Test selection based on changed files:
1. Detect which files changed in MR (`git diff`)
2. Map files to service directories
3. Use `bazel query "rdeps(//..., //service/...)"` to find dependents
4. Run tests only for affected targets
5. Keep full `//...` testing for shared library changes

### Safety Guards

Always run full `//...` tests for:
- Main branch builds
- Tag builds
- Changes to `libraries/` (shared code)
- Changes to `.gitlab/`, `scripts/`, `Makefile`, `MODULE.bazel`
- Changes affecting >5 services
- Manual pipelines
- Renovate dependency update PRs

### Expected Outcomes

| Scenario | Current (cached) | With Selection | Savings |
|----------|------------------|----------------|---------|
| Single service change | 4.5 min | 30-60 sec | 83-89% |
| 2-3 service changes | 4.5 min | 1-2 min | 56-78% |
| Shared library change | 4.5 min | 4.5 min | 0% (by design) |

**Average CI time reduction: 65-75%**

$ARGUMENTS
