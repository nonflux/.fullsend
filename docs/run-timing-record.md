# Agent Run Timing Record

Tracking execution timing for code agent runs to identify bottlenecks,
regressions, and optimization opportunities.

---

## Run #40 — SUCCESS (2026-04-20)

**GitHub Actions:** https://github.com/nonflux/.fullsend/actions/runs/24677918010
**PR Created:** [nonflux/integration-service#37](https://github.com/nonflux/integration-service/pull/37)
**Files Changed:** 3 (`snapshotgc.go`, `snapshotgc_test.go`, `snapshot.go`)
**Timeout Setting:** 20 minutes
**Commit:** pre-commit retry limit + progress markers

| Phase | Start (UTC) | End (UTC) | Duration | Notes |
|-------|-------------|-----------|----------|-------|
| Agent start | 16:54:47 | — | — | Session begins |
| Issue reading + codebase exploration | 16:54:47 | 16:55:10 | ~23s | Read issue, checked for existing branch |
| Implementation (edits) | ~16:55:30 | ~16:56:33 | ~63s | Wrote code changes across 3 files |
| `go build` | 16:56:33 | 16:59:11 | **2m38s** | Downloaded Go 1.25.0 at runtime |
| Secret scan | 16:59:11 | 16:59:14 | 3s | scan-secrets passed |
| Pre-commit run 1 (install + run) | 16:59:14 | 17:01:18 | **2m4s** | Installed 9 hook envs. FAILED: `go-fmt` auto-fixed imports |
| Pre-commit run 2 (cached) | 17:01:23 | 17:02:53 | **1m30s** | Hook envs cached. ALL PASSED (golangci-lint is the slow hook) |
| Tests (`go test`) | 17:02:58 | 17:03:25 | 27s | 43 tests pass in 0.1s + compilation overhead |
| Commit + gitlint | 17:03:40 | 17:03:54 | 14s | Passed |
| **TOTAL** | 16:54:47 | 17:03:54 | **~9m07s** | Well under 20m timeout |

**Pre-commit details:**
- Run 1: `go-fmt` failed (auto-fixed `snapshotgc_test.go` imports). All other hooks passed.
- Run 2: All 14 hooks passed. golangci-lint is the bottleneck (~60-90s of the 1m30s).
- YAML hooks skipped (no YAML files changed): `check yaml`, `yamllint`, `Lint Dockerfile`, `actionlint`

**Key observation:** Agent did NOT pre-format with gofmt before running pre-commit
(this SKILL.md improvement was added after this run). Cost: 1 extra pre-commit cycle.

---

## Run #42 — SUCCESS (2026-04-20)

**GitHub Actions:** https://github.com/nonflux/.fullsend/actions/runs/24682018015
**PR Created:** [nonflux/integration-service#38](https://github.com/nonflux/integration-service/pull/38)
**Files Changed:** 3 (`snapshotgc.go`, `snapshotgc_test.go`, `snapshot.go`)
**Timeout Setting:** 25 minutes
**Commit:** post-script pre-commit gate + per-file→all-at-once + pre-format step

| Phase | Start (UTC) | End (UTC) | Duration | Notes |
|-------|-------------|-----------|----------|-------|
| Agent start | 18:00:02 | — | — | Session begins |
| Issue reading + codebase exploration | 18:00:02 | 18:01:17 | ~75s | Deep exploration: 7 reads, 4 greps, 1 PR view |
| Branch creation | 18:01:17 | 18:01:19 | 2s | `agent/1-fix-plr-finalizer-removal-v8` |
| Implementation (edits) | 18:01:24 | 18:02:26 | **62s** | 6 edits across 3 files |
| `go build` (compile check) | 18:02:29 | 18:04:04 | **95s** | Downloaded Go 1.25.0 at runtime |
| Secret scan | 18:04:07 | 18:04:08 | 1s | scan-secrets passed |
| **Pre-format (gofmt)** | 18:04:11 | 18:04:12 | **<1s** | ✅ NEW: Agent ran gofmt before pre-commit |
| Pre-commit run 1 (install + run) | 18:04:18 | 18:07:54 | **3m36s** | Installed 9 hook envs. ALL PASSED first try |
| Tests (`go test` snapshotgc) | 18:07:58 | 18:08:21 | 23s | 43 tests pass |
| Tests (`go test` gitops) | 18:08:24 | 18:08:27 | 3s | No matching tests (expected) |
| Full build verification | 18:08:31 | 18:08:35 | 4s | `go build ./...` — cached, fast |
| Secret scan (staged) | 18:08:47 | 18:08:47 | <1s | Pre-commit scan on staged files |
| Commit + gitlint | 18:08:54 | 18:08:59 | 5s | Passed |
| **TOTAL** | 18:00:02 | 18:09:11 | **~9m09s** | Well under 25m timeout |

**Pre-commit details:**
- Run 1: ALL 14 hooks passed on first try (zero retries needed)
- Pre-format with gofmt eliminated the auto-fix issue seen in run #40
- golangci-lint: Passed (still the slowest hook, ~60-90s of the 3m36s total)
- Hook env install: ~2m of the 3m36s (first-run cost, unavoidable in sandbox)
- YAML hooks skipped (no YAML files changed)

**Key observation:** Pre-format step (Step A from revised SKILL.md) saved 1 full
pre-commit cycle compared to run #40. Run completed in nearly identical total time
but with only 1 pre-commit invocation instead of 2.

---

## Comparative Analysis

| Metric | Run #40 | Run #42 | Delta |
|--------|---------|---------|-------|
| Total agent time | 9m07s | 9m09s | +2s |
| Pre-commit invocations | 2 | 1 | -1 (saved ~1m30s potential) |
| Pre-commit total time | 3m34s | 3m36s | — |
| Hook env install time | ~2m (first run only) | ~2m (first run only) | — |
| golangci-lint time | ~90s per run | ~90s | — |
| go build time | 2m38s | 1m35s | -63s (partial cache?) |
| Tests time | 27s | 26s | — |
| Files changed | 3 | 3 | — |
| YAML files | 0 | 0 | — |

## Time Budget Breakdown (typical successful 3-file Go run)

```
Codebase reading + planning:   1-2 min
Implementation (edits):        1 min
go build (first compile):      1.5-2.5 min  ← Go download at runtime
Secret scan:                   <5s
Pre-format (gofmt/goimports):  <1s
Pre-commit (first run):        3-4 min      ← 2 min install + 1-2 min hooks
Tests:                         0.5 min
Commit + gitlint:              <10s
─────────────────────────────────────────
TOTAL:                         ~8-10 min
```

## Known Bottlenecks

1. **Pre-commit hook env install (2-3 min):** Every run pays this cost because
   the sandbox is destroyed after each run. Hook environments are not persisted.
   Fix: Could pre-install hook envs in the Containerfile IF sandbox overlay
   issues are resolved (previously attempted, abandoned due to overlay bug).

2. **Go toolchain download (1-2 min):** The sandbox downloads Go 1.25.0 at
   runtime via `go build`. Fix: Pin the exact Go version in the Containerfile
   so it's baked into the image.

3. **golangci-lint (~90s):** This is the slowest pre-commit hook. It compiles
   the entire Go module to perform static analysis. Unavoidable for correctness.
   Running per-file would multiply this cost by N — all-at-once is critical.

4. **YAML file risk (untested):** When the agent touches YAML files
   (e.g., `config/rbac/*.yaml`), yamllint activates. If the agent generates
   YAML with style mismatches, it could trigger retries. Each retry with
   golangci-lint adds ~90s. Worst case with YAML: 9m baseline + 2×90s = ~12 min.

## Run #43 — FAILED / TIMEOUT (2026-04-20)

**GitHub Actions:** https://github.com/nonflux/.fullsend/actions/runs/24683281235
**PR Created:** None (timed out)
**Timeout Setting:** 25 minutes
**Artifacts:** None (output directory empty)

| Phase | Start (UTC) | End (UTC) | Duration | Notes |
|-------|-------------|-----------|----------|-------|
| Fullsend CLI start | 18:26:07 | — | — | |
| Gateway ready | — | 18:27:01 | 54s | |
| Sandbox created | — | 18:28:01 | 60s | |
| Project code copied | — | 18:28:28 | 27s | |
| Agent start | 18:28:28 | — | — | |
| Agent timeout | — | 18:53:28 | **25m0s** | `ssh command timed out after 25m0s` |
| **TOTAL job time** | — | — | **27m37s** | Setup (2m21s) + agent (25m) |

**Context:** PR #38 from Run #42 was still OPEN on branch
`agent/1-fix-plr-finalizer-removal-v8`. There were 10+ existing agent
branches for issue #1. The SKILL.md step 4 tells the agent to check out
existing branches and "work on top" of them.

**Hypothesis — Existing branch confusion:** When the agent finds a branch
with a complete, working implementation, it may:
1. Check out the existing v8 branch with all commits intact
2. Get confused about what more to do (the issue is already solved)
3. Try to "improve" the solution by adding RBAC, more tests, etc.
4. Add YAML files that trigger pre-commit failures → retry loops → timeout

This explains the non-determinism: runs #40 and #42 succeeded because
the agent happened to create a fresh approach with just 3 Go files. Run
#43 may have tried to build on the existing branch and added complexity.

**Action items:**
- Delete old agent branches before triggering new runs (clean slate)
- Or close PR #38 so the agent doesn't see "work already done"
- Or update SKILL.md step 4 to say: "if a PR already exists for this
  branch, the work is done — do not repeat it"

---

## Failed Runs (for comparison)

| Run | Duration | Outcome | Root Cause |
|-----|----------|---------|------------|
| #38 (wf) | 20m (timeout) | No branch | Unknown (no artifacts) |
| #39 (wf) | 20m (timeout) | No branch | Unknown (no artifacts) |
| #41 (wf) | 20m (timeout) | No branch | Unknown (no artifacts) |
| #43 (wf) | 25m (timeout) | No branch | Likely existing branch confusion + RBAC file |

All timeout runs produced zero artifacts, making root cause impossible to
confirm. The successful runs complete in ~9 min when touching only Go files.

**Pattern:** 3 successes out of 6 recent runs (50% success rate). All
successes had 3 Go files. All failures are timeouts with no artifacts.
The same issue (#1) has been solved multiple times — stale branches and
open PRs may be causing the agent to over-engineer on subsequent runs.

---

*Last updated: 2026-04-20 after Run #43*
