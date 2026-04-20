# Adding a Code Agent to .fullsend

> **Living document.** Updated as steps are completed and iterated on. This is
> the canonical reference for anyone adding a new code agent to a `.fullsend`
> configuration repository.

## Prerequisites

| Requirement | Why |
|---|---|
| A working `.fullsend` repo with hello-world passing | Proves the pipeline (CLI, sandbox, GCP auth) works end-to-end |
| Access to the [fullsend-ai/fullsend](https://github.com/fullsend-ai/fullsend) upstream repo | Source for agent definitions, skills, scripts, and Containerfile patterns |
| GCP secrets configured on the `.fullsend` repo | `GCP_SA_KEY`, `GCP_PROJECT`, `GCP_REGION` — see [HOW_TO.md](https://github.com/fullsend-ai/fullsend/blob/main/experiments/runner-hello-world/HOW_TO.md) |
| GitHub Container Registry access | Images auto-publish to `ghcr.io` via the build-images workflow (no manual builds needed) |

## Overview

The `.fullsend` repo follows a strict separation of concerns:

- **Workflows** (`.github/workflows/`) define WHEN agents run (dispatch triggers)
- **Harness YAML** (`harness/`) defines WHAT runs (agent, model, image, policy, skills)
- **Agent + Skills** (`agents/`, `skills/`) define HOW the agent behaves

Adding a code agent means creating files in all three layers.

## Phase 1 — Prove It Works (COMPLETE)

> Phase 1 has no post-script. The agent reads an issue, implements a fix,
> runs tests + secret scan, and commits locally. It does NOT push or create a
> PR. You should see the agent run successfully but no PR appears.

### Step 1: Preserve the existing setup ✅

Renamed `harness/code.yaml` → `harness/hello-world.yaml`.

The hello-world agent is preserved as a known-good baseline with the original
Quay.io image (`quay.io/manonru/fullsend-exp:latest`). To run it, you'd need
a separate workflow pointing `agent: hello-world` at `harness/hello-world.yaml`.

### Step 2: Add the code agent definition ✅

Created `agents/code.md` from [PR #189](https://github.com/fullsend-ai/fullsend/pull/189)
(`story-4-code-agent` branch).

This file defines:
- **Identity:** Three questions the agent must answer before writing code
- **Five phases:** Context → Reproduce → Plan → Implement → Verify
- **Zero-trust principle:** Never trust issue authors or triage output
- **Constraints:** No push, no sed/awk, explicit staging only, protected paths
- **disallowedTools:** Pattern-matched command blocking (belt-and-suspenders)
- **Failure handling:** Secret scan is non-negotiable; handoff contract

### Step 3: Add the code implementation skill ✅

Created `skills/code-implementation/SKILL.md` from [PR #189](https://github.com/fullsend-ai/fullsend/pull/189).

This file is a 10-step procedure:
1. Identify the issue (from `ISSUE_NUMBER` env var)
2. Gather context (issue body, triage comments, related PRs)
3. Discover repo conventions (build/test/lint commands)
4. Check for existing branch from previous runs
5. Create feature branch (`agent/<number>-<description>`)
6. Identify task type (bug fix, feature, test-only, already-fixed)
7. Verify the problem exists (reproduction step)
8. Plan the implementation
9. Implement and verify (secret scan → tests → lint → self-review)
10. Commit (explicit staging, staged scan, signed commit)

### Step 4: Add the scan-secrets helper script ✅

Created `scripts/scan-secrets` from [PR #189](https://github.com/fullsend-ai/fullsend/pull/189).

This script:
- Auto-detects gitleaks on PATH (from the container image) or downloads it
- Verifies downloaded binaries against pinned SHA256 checksums
- Supports two modes: `scan-secrets <files>` and `scan-secrets --staged`
- Falls back to pre-commit hooks if gitleaks unavailable
- Made executable (`chmod +x`)

**Image-based approach (recommended):** Per the team's convention from
[PR #231](https://github.com/fullsend-ai/fullsend/pull/231), stable skill
dependencies should be baked into the sandbox image rather than delivered
via `host_files`. The script is now `COPY`'d into the image at
`/usr/local/bin/scan-secrets` (read-only path), so the agent cannot tamper
with it at runtime. The skill references it directly as `scan-secrets`
(on `$PATH`), not through an environment variable.

### Step 5: Add the Containerfile + automated image pipeline ✅

Created `images/code/Containerfile` and `.github/workflows/build-images.yml`.

**How the automated image pipeline works:**

```
┌─────────────────┐     ┌───────────────────────┐     ┌──────────────────────┐
│  You write code │     │  build-images.yml      │     │  Agent workflow      │
│                 │     │  (GitHub Actions CI)   │     │  (every agent run)   │
│                 │     │                        │     │                      │
│ Create/edit ────────► │ Auto-discovers images/ │     │                      │
│ images/code/    │     │ Builds base sandbox    │     │                      │
│ Containerfile   │     │ Builds each agent img  │     │ fullsend CLI reads   │
│                 │     │ Pushes to ghcr.io ─────────► │ harness/code.yaml    │
│ Push to main    │     │                        │     │ pulls image field    │
│ (or dispatch)   │     │ ghcr.io/nonflux/       │     │ creates sandbox      │
│                 │     │ .fullsend/code:latest  │     │ gitleaks on PATH     │
└─────────────────┘     └───────────────────────┘     └──────────────────────┘
```

**No manual builds.** The workflow handles everything:

1. **Trigger:** Push to main that touches `images/**/Containerfile`, or manual
   `workflow_dispatch` (optionally targeting a single image)
2. **Discover:** Finds all `images/*/Containerfile` directories
3. **Build base:** Checks out `fullsend-ai/fullsend`, builds the base sandbox
   image, pushes to `ghcr.io/nonflux/fullsend-sandbox:latest` (or uses
   `quay.io/manonru/fullsend-exp:latest` as fallback if PR #239 isn't merged)
4. **Build agents (matrix):** For each discovered image, builds it with
   `--build-arg BASE_IMAGE=<base>:latest` and pushes to
   `ghcr.io/nonflux/fullsend-<name>:latest`
5. **Cache:** Uses GitHub Actions cache for fast rebuilds

**Bringing your own agent image:** Create `images/<your-agent>/Containerfile`,
push to main, and the workflow discovers and builds it automatically.

**Why pre-install gitleaks:** The `scan-secrets` script uses `command -v gitleaks`
first. If found on PATH (pre-installed in image), it runs with zero latency
and no network. Otherwise it falls back to downloading at runtime — slower and
requires egress from the sandbox.

### Step 6: Create the code agent harness ✅

Created `harness/code.yaml`:

```yaml
agent: agents/code.md
model: opus
image: ghcr.io/nonflux/fullsend-code:latest
policy: policies/code.yaml

host_files:
  - src: env/gcp-vertex.env
    dest: /tmp/workspace/.env.d/gcp-vertex.env
    expand: true
  - src: ${GOOGLE_APPLICATION_CREDENTIALS}
    dest: /tmp/workspace/.gcp-credentials.json

skills:
  - skills/code-implementation

timeout_minutes: 30
```

Key differences from the hello-world harness:
- **agent:** `agents/code.md` (real code agent, not hello-world)
- **image:** `ghcr.io/nonflux/fullsend-code:latest` (auto-built, includes gitleaks)
- **policy:** `policies/code.yaml` (broader network — GitHub API, Go/npm/PyPI)
- **skills:** `skills/code-implementation` (10-step procedure)
- **timeout_minutes:** 30 (code agent needs more time than hello-world's 5)
- **No validation_loop:** The agent's own verification cycle replaces external validation
- **No post_script:** Phase 1 — agent commits locally only

### Step 7: Create the code agent sandbox policy ✅

Created `policies/code.yaml` extending the hello-world policy with:

- **Vertex AI:** `*.googleapis.com` (model inference)
- **GitHub:** `api.github.com` + `github.com` (gh CLI reads issues/PRs, git operations)
- **Gitleaks releases:** `github.com` + `objects.githubusercontent.com` (fallback download)
- **Package registries:** npm, PyPI, Go module proxy (`proxy.golang.org`, `sum.golang.org`)

### Step 8: Update the workflow ✅

Modified `.github/workflows/code.yml`:

- **Added `ISSUE_NUMBER`** — passed to the agent via both `setup-agent-env.sh`
  (as `CODE_ISSUE_NUMBER`) and directly on the fullsend step. The skill uses
  `gh issue view "${ISSUE_NUMBER}"` in step 1.
- **Added `packages: read`** — permission to pull images from ghcr.io.
- **Changed target repo `fetch-depth: 0`** — full history so the agent can
  create branches and work with git properly.

### Step 9: Test end-to-end

> **Before testing:** The `build-images.yml` workflow must run first to publish
> the container image to ghcr.io. Either push the Containerfile to main and let
> CI build it, or trigger `build-images.yml` manually via workflow_dispatch.

Trigger the workflow manually:

```bash
gh workflow run code.yml \
  --repo nonflux/.fullsend \
  -f event_type=issue \
  -f source_repo=nonflux/<target-repo> \
  -f event_payload='{"issue":{"number":1,"html_url":"https://github.com/nonflux/<target-repo>/issues/1"}}'
```

Watch the run:

```bash
gh run list --repo nonflux/.fullsend --workflow code.yml --limit 1
gh run watch <RUN_ID> --repo nonflux/.fullsend
```

**What to verify:**
- [ ] Harness loads successfully
- [ ] Container image pulls and sandbox starts
- [ ] Agent receives the issue context (`ISSUE_NUMBER`, `GITHUB_ISSUE_URL`)
- [ ] gitleaks is found on PATH (no download fallback message)
- [ ] scan-secrets runs and passes
- [ ] Agent produces a commit on a local feature branch
- [ ] **No PR is created** (this is Phase 1 — commit only)

**What to check in artifacts:**
- Download `fullsend-code` artifact from the Actions run
- Check transcripts for the agent's reasoning and execution
- Verify the agent followed the 10-step procedure from the skill

## Phase 2 — Pre/Post Script Integration (IN PROGRESS)

Phase 2 adds the security-critical pre-script and post-script that turn the
agent's local commit into a pushed branch + PR.

### Step 10: Create the pre-script ✅

Created `scripts/pre-code.sh` — runs on the runner BEFORE sandbox creation.

Validates all three `event_payload` inputs:
- `ISSUE_NUMBER` — must be a positive integer (regex: `^[1-9][0-9]*$`)
- `REPO_FULL_NAME` — must be `owner/repo` format
- `GITHUB_ISSUE_URL` — must match `https://github.com/<owner>/<repo>/issues/<num>`

Fails fast with `::error::` annotations if any input is malformed. This prevents
shell injection and malicious payloads from reaching the sandbox.

### Step 11: Create the post-script ✅

Created `scripts/post-code.sh` — runs on the runner AFTER the sandbox is
destroyed. This is the most security-sensitive component in the pipeline.

**Security gates (executed in order):**

1. **Branch verification** — refuses to push if the agent is on `main`/`master`
   or detached HEAD
2. **Protected-path check** — rejects if the agent modified any of:
   `.github/`, `.claude/`, `agents/`, `harness/`, `policies/`, `scripts/`,
   `api-servers/`, `CODEOWNERS`
3. **Authoritative secret scan** — runs `gitleaks detect` on the agent's
   commit diff (`HEAD~1..HEAD`). Self-bootstraps gitleaks if not on PATH.
   This is the load-bearing security gate: the agent's scan is belt, this
   is suspenders.
4. **Push** — rewrites the git remote to use `PUSH_TOKEN` (which never
   enters the sandbox) and pushes the feature branch
5. **PR creation** — creates a PR that auto-closes the originating issue

### Step 12: Token scoping ✅

**Agent sandbox receives `GH_TOKEN` (read-only intent):**
- Used for `gh issue view`, `gh pr view` (reads only)
- `disallowedTools` blocks `git push`, `gh pr create`, `gh api`
- Sandbox network policy only allows `gh`/`git` binaries to reach GitHub

**Post-script receives `PUSH_TOKEN` (write-scoped):**
- Stored as a repo secret (`PUSH_TOKEN`) — a PAT or fine-grained token
  with `contents: write` + `pull-requests: write` on target repos
- Passed via `runner_env` in the harness — never enters the sandbox
- Falls back to `github.token` for same-repo setups (cross-repo requires
  a separate token since `github.token` is scoped to the workflow repo)

### Step 13: Update harness and workflow ✅

**harness/code.yaml changes:**
```yaml
pre_script: scripts/pre-code.sh
post_script: scripts/post-code.sh

runner_env:
  PUSH_TOKEN: "${PUSH_TOKEN}"
  REPO_FULL_NAME: "${REPO_FULL_NAME}"
  ISSUE_NUMBER: "${ISSUE_NUMBER}"
```

**code.yml changes:**
- Added `Validate inputs` step (runs pre-script before CLI)
- Added `Push branch and create PR` step (fallback if CLI doesn't run post_script)
- Added `PUSH_TOKEN`, `REPO_FULL_NAME` to agent step environment
- Changed target repo `fetch-depth: 0` (full history for push)
- Fallback step detects if CLI post_script already pushed (checks for
  existing PR on the branch) to avoid duplicate PRs

### Step 14: Set up the PUSH_TOKEN secret

Create a fine-grained PAT (or classic PAT) with access to target repos:

1. Go to **Settings → Developer settings → Fine-grained personal access
   tokens → Generate new token**
2. Scope to the org/repos the agent will target
3. Permissions: `Contents: Read and write`, `Pull requests: Read and write`
4. Add as a secret on the `.fullsend` repo:
   ```bash
   gh secret set PUSH_TOKEN --repo nonflux/.fullsend
   ```

### Step 15: Test the full loop

Trigger on a real issue and verify:
1. Pre-script validates inputs (check workflow logs)
2. Agent reads the issue, implements fix, commits locally
3. Post-script validates: protected paths → secret scan → push → PR
4. PR appears on the target repo linking to the issue
5. No secrets leaked (check gitleaks output in logs)

## File Checklist

| Action | Path | Status | Source |
|--------|------|--------|--------|
| RENAME | `harness/code.yaml` → `harness/hello-world.yaml` | ✅ | existing |
| CREATE | `harness/code.yaml` | ✅ | new (Step 6) |
| CREATE | `agents/code.md` | ✅ | PR #189 |
| CREATE | `skills/code-implementation/SKILL.md` | ✅ | PR #189 |
| CREATE | `scripts/scan-secrets` | ✅ | PR #189 (also baked into image) |
| CREATE | `policies/code.yaml` | ✅ | new (Step 7) |
| CREATE | `images/code/Containerfile` | ✅ | pattern from PR #239 |
| CREATE | `.github/workflows/build-images.yml` | ✅ | new (auto-builds images) |
| MODIFY | `.github/workflows/code.yml` | ✅ | added ISSUE_NUMBER, packages:read, fetch-depth:0 |
| KEEP   | `agents/hello-world.md` | — | unchanged |
| KEEP   | `skills/hello-world-summary/SKILL.md` | — | unchanged |
| KEEP   | `policies/hello-world.yaml` | — | unchanged |
| KEEP   | `scripts/validate-output.sh` | — | unchanged |
| CREATE | `scripts/pre-code.sh` | ✅ | Phase 2 — input validation |
| CREATE | `scripts/post-code.sh` | ✅ | Phase 2 — push + PR |
| MODIFY | `harness/code.yaml` | ✅ | Phase 2 — pre/post script, runner_env |
| MODIFY | `.github/workflows/code.yml` | ✅ | Phase 2 — validate step, post-script fallback, PUSH_TOKEN |
| MODIFY | `env/code-agent.env` | ✅ | Phase 2 — GH_TOKEN read-only docs |
| MODIFY | `.github/workflows/code.yml` | ✅ | Phase 2 — GitHub App token generation, fallback chain |
| MODIFY | `harness/code.yaml` | ✅ | Phase 2 — PUSH_TOKEN_SOURCE in runner_env |
| MODIFY | `scripts/post-code.sh` | ✅ | Phase 2 — token source logging, GitHub App support |

## Known Limitations (Phase 1)

1. **No PR creation** — by design. The agent commits locally; nothing pushes.
   This is the Phase 1 constraint. Phase 2 adds the post-script.
2. **Image must be published first** — the `build-images.yml` workflow needs
   to run at least once before the code agent can be triggered. The image
   path `ghcr.io/nonflux/fullsend-code:latest` is hardcoded to this org.
3. **Network policy covers Go/Node/Python** — other ecosystems (Rust, Ruby,
   Java) would need additional endpoints in `policies/code.yaml`.
4. **Agent text references post-script** — `agents/code.md` and the skill
   mention a post-script that handles pushing/PR creation. In Phase 1 this
   doesn't exist, so the agent commits and the sandbox exits cleanly.

## Phase 1 Run #18 — Findings and Fixes

These issues were found by auditing all 50 tool calls in the successful
Run #18 transcript. Each wasted tool call = latency + cost.

| # | Issue | Wasted calls | Severity | Fix |
|---|-------|-------------|----------|-----|
| 1 | **Go not installed in sandbox** — agent tried `go build` (×2), `which go`, `find / -name go` before giving up and doing manual code review only. Could not compile or run tests. | 4 | Critical | Added Go 1.24.13 to `images/code/Containerfile` |
| 2 | **`git fetch origin` SSL failure** — `CAfile: none` because sandbox lacks system CA certificates. Agent fell back to branching off local `main`. Would also block `git push` in Phase 2. | 1 | Medium | Added `ca-certificates` to `images/code/Containerfile` |
| 3 | **gitleaks not pre-installed** — `scan-secrets` had to download gitleaks at runtime because the custom image (which includes gitleaks) isn't being used. GHCR visibility issue means harness falls back to `quay.io/manonru/fullsend-exp:latest`. | 0 (just latency) | Medium | Already in Containerfile; need GHCR image to actually be pulled |
| 4 | **PR #23 not accessible** — agent tried `gh pr view 23` for context on a previous related PR but got "PR not found". Minor; agent adapted. | 1 | Low | Expected (repo is a fork, PR is in upstream) |

**Total wasted tool calls: 6 out of 50 (12%)**

Once the custom image is pulled (GHCR visibility fix), issues 1–3 are
all resolved. Issue 4 is a non-blocking edge case.

## Phase 2 Security TODOs — Status

These items were identified during the Phase 1 security review:

1. ~~**Validate `event_payload` in the pre-script**~~ ✅ Done — `scripts/pre-code.sh`
   validates ISSUE_NUMBER (positive integer), REPO_FULL_NAME (owner/repo),
   and GITHUB_ISSUE_URL (GitHub URL pattern) before the sandbox is created.

2. ~~**Scope down `GH_TOKEN`**~~ ✅ Done — the agent still receives `github.token`
   for reads, but writes are blocked by `disallowedTools` + sandbox policy.
   The write-scoped `PUSH_TOKEN` is passed only to the post-script via
   `runner_env` (never enters the sandbox). Full isolation via a GitHub App
   token remains a future upgrade if needed.

## Phase 2 Improvements — Post-Run #23

These improvements were added after analyzing the first successful end-to-end run:

1. ~~**Bot commit identity**~~ ✅ Done — `GIT_AUTHOR_NAME` / `GIT_COMMITTER_NAME`
   changed from `fullsend[bot]` to `fullsend-code` in `env/code-agent.env`.
   Commits are now attributed to `fullsend-code`.

2. ~~**PR description quality**~~ ✅ Done — `post-code.sh` now extracts the
   agent's commit subject as the PR title and the commit body as the PR
   description. Includes a changed-files list and post-script verification
   checklist. Falls back to a summary if the commit body is empty.

3. ~~**Pre-commit hooks**~~ ✅ Done — `pre-commit` is baked into the sandbox
   image (`images/code/Containerfile`). The skill (`SKILL.md` step 9b) now
   requires the agent to run `pre-commit run --files <changed-files>` if the
   target repo has a `.pre-commit-config.yaml`. Only changed files are checked
   (pre-existing failures are not the agent's responsibility).

4. ~~**PR authorship**~~ ✅ Done — PRs are now authored by the GitHub App
   when configured (see Step 16). The workflow uses `actions/create-github-app-token@v3`
   to generate a short-lived installation token scoped to the target repo.
   This token replaces the PAT as `PUSH_TOKEN`, so both `git push` and
   `gh pr create` authenticate as the App. PRs appear as `fullsend-code[bot]`
   (or whatever the App is named). Falls back to PAT if App is not configured.

### Step 16: GitHub App for bot-authored PRs ✅

When `PUSH_TOKEN` is a PAT, PRs show as authored by the PAT owner (a human
account). To make PRs appear as `fullsend-code[bot]`, the workflow now uses
a GitHub App installation token instead.

**How it works:**

```
┌──────────────────────┐     ┌─────────────────────────┐     ┌───────────────────┐
│  GitHub App          │     │  code.yml workflow       │     │  post-code.sh     │
│  (fullsend-code)     │     │                          │     │                   │
│                      │     │  1. Extract repo owner   │     │                   │
│  APP_ID (variable)   │     │  2. create-github-app-   │     │                   │
│  PRIVATE_KEY (secret)│────►│     token@v3 generates   │────►│  PUSH_TOKEN =     │
│                      │     │     installation token   │     │  App token        │
│  Installed on target │     │  3. Resolve: App token   │     │  git push + gh pr │
│  repos with:         │     │     > PAT > none         │     │  authenticate as  │
│  - contents:write    │     │  4. Pass as PUSH_TOKEN   │     │  the App          │
│  - pull-requests:    │     │                          │     │                   │
│      write           │     │                          │     │  PR appears from  │
│                      │     │                          │     │  fullsend-code    │
│                      │     │                          │     │  [bot]            │
└──────────────────────┘     └─────────────────────────┘     └───────────────────┘
```

**Token resolution priority (defense-in-depth):**

1. **GitHub App token** — if `FULLSEND_CODER_CLIENT_ID` variable is set, the
   workflow generates a scoped installation token via `actions/create-github-app-token@v3`.
   PRs are authored by the App identity.
2. **PAT fallback** — if App is not configured but `PUSH_TOKEN` secret exists,
   uses the PAT. PRs are authored by the PAT owner (with a warning).
3. **No token** — Phase 1 mode. No push, no PR.

**Setup instructions:**

1. **Create the GitHub App** (one-time, org-admin):

   Go to **GitHub → Settings → Developer settings → GitHub Apps → New GitHub App**
   (or use `https://github.com/settings/apps/new`):

   | Field | Value |
   |---|---|
   | App name | `fullsend-code` (or `fullsend-<org>-coder` per [ADR 0007](https://github.com/fullsend-ai/fullsend/blob/main/docs/ADRs/0007-per-role-github-apps.md)) |
   | Homepage URL | `https://github.com/fullsend-ai/fullsend` |
   | Webhook | Unchecked (not needed) |
   | Permissions → Repository → Contents | Read and write |
   | Permissions → Repository → Pull requests | Read and write |
   | Where can this app be installed? | Only on this account (or Any account for multi-org) |

   After creation, note the **Client ID** (shown on the App settings page,
   looks like `Iv23li...`). The numeric App ID is no longer needed — the
   `actions/create-github-app-token@v3` action now uses `client-id`.

2. **Generate a private key:**

   On the App settings page, scroll to **Private keys → Generate a private key**.
   This downloads a `.pem` file. This is your only chance to get this key — if
   lost, you must generate a new one (the old one is invalidated).

3. **Install the App on target repos:**

   Go to `https://github.com/apps/<app-slug>/installations/new` and install
   on the org/repos the agent will target. Grant access to the specific repos
   (e.g., `nonflux/integration-service`) — not "All repositories".

4. **Store credentials on the `.fullsend` repo:**

   ```bash
   # Client ID as a repository variable (not a secret — it's not sensitive)
   gh variable set FULLSEND_CODER_CLIENT_ID \
     --repo nonflux/.fullsend \
     --body "<CLIENT_ID>"

   # Private key as a repository secret
   gh secret set FULLSEND_CODER_APP_PRIVATE_KEY \
     --repo nonflux/.fullsend \
     < /path/to/downloaded-private-key.pem
   ```

   Naming follows the [fullsend normative SPEC](https://github.com/fullsend-ai/fullsend/blob/main/docs/normative/admin-install/v1/adr-0014-github-apps-and-secrets/SPEC.md):
   `FULLSEND_<ROLE>_CLIENT_ID` (variable) and `FULLSEND_<ROLE>_APP_PRIVATE_KEY`
   (secret). The role is `CODER` (not `CODE`) per `config.ValidRoles`.

5. **Verify:**

   Trigger a code agent run. Check the workflow logs:
   - "Generate GitHub App token" step should succeed (no deprecation warning)
   - "Resolve push token" should say "Using GitHub App installation token"
   - The created PR should show as authored by `fullsend-code[bot]`
     (or whatever you named the App)
   - `PUSH_TOKEN_SOURCE` in the post-script output should say `github-app`

**Backward compatibility:** The existing `PUSH_TOKEN` PAT secret continues
to work. If you don't configure the App, the workflow falls back to the PAT
with a warning. You can remove the `PUSH_TOKEN` secret once the App is
verified working.

**Installer path (recommended):** Once [PR #264](https://github.com/fullsend-ai/fullsend/pull/264)
lands, `fullsend admin install` automates all of this. The installer:

1. Creates per-role GitHub Apps via the manifest flow (browser-based)
2. Stores `FULLSEND_CODER_APP_ID` (variable) and `FULLSEND_CODER_APP_PRIVATE_KEY`
   (secret) on the `.fullsend` repo automatically
3. Scaffolds workflow files that already include the `create-github-app-token` step

Run:
```bash
make go-build
./bin/fullsend admin install <your-org> \
  --repo <target-repo> \
  --gcp-region <region> \
  --gcp-project <project> \
  --gcp-credentials-file <path/to/credentials.json>
```

If you've already installed manually (like our `nonflux/.fullsend` setup), re-running
the installer will update the config files to match the scaffold. See Ralph's
[demo on appdumpster](https://github.com/appdumpster/.fullsend) for a working example.

**Architecture difference from upstream scaffold:** The installer's scaffolded
`code.yml` passes the App token directly to the agent as `CODE_GH_TOKEN` —
the agent uses it for both reads and writes (including push/PR creation).
Our architecture is stricter: the App token is used only on the runner
(post-script) and never enters the sandbox. The agent gets a read-only
`github.token` for issue/PR reads, and writes are gated by `disallowedTools`
+ sandbox network policy. This aligns with [ADR 0017](https://github.com/fullsend-ai/fullsend/blob/main/docs/ADRs/0017-credential-isolation-for-sandboxed-agents.md)
("no credentials in sandbox") and provides an additional security layer.

**Security alignment:**

- **[ADR 0007](https://github.com/fullsend-ai/fullsend/blob/main/docs/ADRs/0007-per-role-github-apps.md):**
  Per-role GitHub Apps with least-privilege permissions.
- **[ADR 0017](https://github.com/fullsend-ai/fullsend/blob/main/docs/ADRs/0017-credential-isolation-for-sandboxed-agents.md):**
  No credentials in the sandbox. The App token is generated and consumed
  entirely on the runner — the sandbox never sees it.
- **[Experiment #67](https://github.com/fullsend-ai/fullsend/blob/main/experiments/67-claude-github-app-auth/README.md):**
  Proved the JWT → installation token → agent flow works end-to-end.
- **[PR #119 MVP](https://github.com/fullsend-ai/fullsend/pull/119):**
  Used `fullsend-agent[bot]` and `fullsend-reviewer[bot]` as separate
  identities in the nonflux demo pipeline.

## Phase 2 Improvements — Post-Run #26

Run #26 ([nonflux/.fullsend/actions/runs/24640993160](https://github.com/nonflux/.fullsend/actions/runs/24640993160))
was the first successful bot-authored PR:
[nonflux/integration-service#26](https://github.com/nonflux/integration-service/pull/26),
authored by `fullsend-nonflux-coder[bot]`. The review bot approved it after
a single iteration. Analysis of the run artifacts and CI check results
revealed several improvements needed:

1. ~~**`PUSH_TOKEN_SOURCE` env var missing**~~ ✅ Done — The fullsend CLI
   validates that every `runner_env` reference resolves to an actual env var.
   `PUSH_TOKEN_SOURCE` was in the harness `runner_env` but wasn't passed as
   an `env:` on the workflow's agent step. Added to `code.yml`.

2. ~~**Force-push on existing branches**~~ ✅ Done — `post-code.sh` now uses
   `git push --force-with-lease` instead of plain `git push`. Agent branches
   (`agent/<n>-*`) reuse the same name per issue, so re-runs need to update
   the remote branch. `--force-with-lease` is safe: it fails if the remote
   branch moved since the last fetch (race condition protection).

3. ~~**Duplicate post-script execution**~~ ✅ Done — Removed the redundant
   "Push branch and create PR" step from `code.yml`. The fullsend CLI already
   runs `post_script` from the harness definition. Having a second execution
   in the workflow was confusing and masked errors. Now the harness owns the
   entire post-script lifecycle.

4. ~~**PR-exists handling**~~ ✅ Done — `post-code.sh` now checks for an
   existing PR before calling `gh pr create`. If a PR already exists for the
   branch, it logs "PR #N already exists — branch updated with new commits"
   and exits cleanly (exit 0). GitHub auto-shows new commits on existing PRs.

5. ~~**`client-id` deprecation**~~ ✅ Done — Updated `code.yml` to use
   `client-id` instead of the deprecated `app-id` input for
   `actions/create-github-app-token@v3`. Variable changed from
   `FULLSEND_CODER_APP_ID` to `FULLSEND_CODER_CLIENT_ID`.

6. ~~**Pre-commit not available in sandbox**~~ ✅ Done — Run #26 showed
   `pre-commit: command not found` because the image was built before
   the Containerfile update. Added `gitlint-core` alongside `pre-commit`
   in the Containerfile. SKILL.md step 9b updated to be **MANDATORY** with
   explicit `--install-hooks` and re-run-after-autofix pattern.

7. ~~**Gitlint commit message validation**~~ ✅ Done — SKILL.md step 10c
   now instructs the agent to read `.gitlint`, respect line-length limits
   (commonly 72 chars for title AND body), and validate with
   `gitlint --commit HEAD` after committing.

8. ~~**Test transparency when suites can't run**~~ ✅ Done — Run #26's
   agent couldn't run the gitops envtest suite (envtest binaries blocked
   by sandbox network). It committed anyway without disclosing this. SKILL.md
   step 9c now requires the agent to note unrunnable test suites in the
   commit message so reviewers and the review bot know what wasn't verified.
   Also added Kubernetes-specific test pitfalls (Status subresource handling).

19. **Authoritative pre-commit gate in post-script** — The agent runs
   pre-commit inside the sandbox but may commit with failures disclosed
   (to prevent timeout blackouts). To ensure security hooks always pass,
   `post-code.sh` now runs an authoritative `pre-commit run --files` on
   the agent's changed files on the GitHub Actions runner (full network,
   no sandbox restrictions) **before** pushing the branch or creating a
   PR. If any hook fails, the push is blocked entirely. This is a
   defense-in-depth layer: the agent tries to fix failures (2 retries),
   and the post-script is the hard gate that guarantees compliance.

20. **Pre-commit timeout optimization** — Analysis of PRs #28–#37
   revealed a clear pattern: every run where the agent modified
   `config/rbac/snapshotgc_rbac.yaml` (4 files) timed out, while runs
   with only 3 Go/test files succeeded. Root cause: the agent was
   spending too many cycles retrying pre-commit without a hard cap,
   and linter hooks like golangci-lint compile the entire Go package
   per invocation. SKILL.md now uses a structured 4-step approach:
   (A) pre-format Go files with gofmt/goimports before pre-commit to
   eliminate auto-fix churn, (B) run all files at once (not per-file —
   per-file multiplies golangci-lint compilations by N), (C) distinguish
   auto-fixes from linter errors, (D) hard cap of 3 total pre-commit
   invocations (initial + 2 retries). Also added guidance to match
   existing YAML style before creating new YAML files, and an explicit
   rule against refactoring to satisfy linters (which introduces new
   findings and restarts the cycle).

### Known remaining issues

- **Sandbox image must include repo-specific test tooling** — The sandbox
  network policy may block downloads at runtime. Any test framework binaries
  that the target repo needs (e.g. envtest for K8s controller projects,
  playwright for frontend projects) must be pre-installed in the sandbox
  image. Each target repo maintainer can supply a custom image via the
  harness `image:` field. The Containerfile should bake in everything
  the repo's `make test` needs. If a tool can't be pre-installed, the
  SKILL instructs the agent to disclose unrunnable suites in the commit
  message, and the review bot serves as a safety net.

## Phase 2 Automation TODOs

1. **Auto-generate a run report HTML** — after each agent run, the post-script
   should parse the JSONL transcript artifact and produce an HTML report
   (similar to `docs/phase1-results.html`) that captures: pipeline timing,
   agent thinking/tool-call replay, commit diff, secret scan results, and
   verification status. This report should be uploaded as a GitHub Actions
   artifact alongside the transcript so reviewers can open a single HTML file
   to see exactly what the agent did, thought, and produced — without needing
   to parse raw JSONL. The Phase 1 report template can serve as the base;
   the generation script just needs to hydrate it from the transcript data.

## Reference Links

- [fullsend-ai/fullsend](https://github.com/fullsend-ai/fullsend) — upstream repo with CLI, architecture docs, experiments
- [PR #189](https://github.com/fullsend-ai/fullsend/pull/189) — code agent definition and skill (V8 hybrid, 490+ trials)
- [PR #239](https://github.com/fullsend-ai/fullsend/pull/239) — security scanning + base sandbox Containerfile pattern
- [Architecture doc](https://github.com/fullsend-ai/fullsend/blob/main/docs/architecture.md) — execution stack, harness, sandbox, policy
- [HOW_TO.md](https://github.com/fullsend-ai/fullsend/blob/main/experiments/runner-hello-world/HOW_TO.md) — GCP setup, experiment runner guide
