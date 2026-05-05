---
name: renovate-shepherd
description: Shepherd open `renovate/*` PRs in open-telemetry/opentelemetry-collector-contrib — refresh stale ones with main, approve when CI is green, and squash-merge approved + green ones. Trigger when the user asks to check, refresh, approve, or merge recent renovate PRs (e.g. "check renovate PRs", "merge approved renovate PRs", "babysit renovate").
---

# Renovate PR shepherd (open-telemetry/opentelemetry-collector-contrib)

Workflow for keeping the queue of `renovate/*` PRs moving in this repo. Default scope is "PRs opened in the last 24 hours" unless the user specifies otherwise.

## Step 1 — list candidate PRs

```bash
CUTOFF=$(date -u -d "24 hours ago" +%Y-%m-%dT%H:%M:%SZ)
gh pr list --repo open-telemetry/opentelemetry-collector-contrib \
  --search "head:renovate created:>=${CUTOFF} is:open draft:false" \
  --json number,headRefName,title,createdAt,author,url --limit 100
```

`date -v-24H` (BSD form) does NOT work in this environment — use `date -d "24 hours ago"` (GNU form).

**Skip draft PRs** — `draft:false` in the search above keeps them out. Renovate opens some updates as drafts intentionally (e.g. when the package is marked unstable or pinned for manual review); leave those alone.

## Step 2 — classify each PR

For each PR, fetch the last commit author, review decision, mergeable state, and check counts:

```bash
gh pr view $PR --repo open-telemetry/opentelemetry-collector-contrib \
  --json commits,reviewDecision,mergeable,mergeStateStatus \
  --jq '{lastCommitAuthors: [.commits[-1].authors[].login], reviewDecision, mergeable, mergeStateStatus}'

gh pr checks $PR --repo open-telemetry/opentelemetry-collector-contrib \
  | awk -F'\t' '{print $2}' | sort | uniq -c
```

Decide based on the **last commit author**:

| Last commit | Action |
|---|---|
| `otelbot[bot]`, branch behind `main` | Merge `main` into the renovate branch and push (Step 3a) |
| `otelbot[bot]`, branch up-to-date with `main` | Push an empty commit to trigger CI (Step 3b) |
| Anyone else (e.g. `songy23`, another reviewer) | Check CI; approve if green (Step 4) |

**Green CI takes precedence over the refresh action regardless of last commit author.** If `gh pr checks` already shows only `pass` and `skipping` (no `pending` or `fail`), skip Step 3 entirely and go straight to Step 4 (approve) + Step 5 (squash-merge), even when otelbot is the last committer. The refresh in Step 3 exists only to coax CI into running when it hasn't; once CI has run and is green, there's nothing to refresh.

**Failed required checks** — when a PR has real CI failures (after filtering the non-required checks in Step 4), handle via Step 3c before giving up.

For PRs that end up **approved + CI green** (`reviewDecision: APPROVED` and `mergeStateStatus: CLEAN`), squash-merge them (Step 5).

## Step 3 — refresh stale renovate branches (otelbot last commit)

First, check whether the branch is actually behind `main`:

```bash
git fetch origin main renovate/<branch>
BEHIND=$(git rev-list --count origin/renovate/<branch>..origin/main)
echo "Behind main by: $BEHIND commit(s)"
```

### Step 3a — branch is behind main (`BEHIND > 0`): merge and push

```bash
git checkout -B renovate/<branch> origin/renovate/<branch>
git merge origin/main --no-edit
git push origin renovate/<branch>
```

If `git merge` fails because of a conflict, see Step 6.

### Step 3b — branch is up-to-date with main (`BEHIND == 0`): push an empty commit

When otelbot's last commit is already on top of an up-to-date base, the PR's CI sometimes never runs (CI counts `1 pass` and nothing else). Pushing an empty commit re-triggers the workflows:

```bash
git checkout -B renovate/<branch> origin/renovate/<branch>
git -c commit.gpgsign=false commit --allow-empty -m "Trigger CI"
git push origin renovate/<branch>
```

### Step 3c — real CI failures: retry once, then triage

When a PR has failing required checks (not in the ignored list), look up the GitHub Actions run attempt number to decide what to do:

```bash
# Extract the run URL from the failing check, then get the run ID and attempt number
FAILING_URL=$(gh pr checks $PR --repo open-telemetry/opentelemetry-collector-contrib \
  --json name,state,link \
  --jq '.[] | select(.state == "FAIL") | .link' | head -1)
# URL shape: https://github.com/<owner>/<repo>/actions/runs/<run-id>/jobs/<job-id>
RUN_ID=$(echo "$FAILING_URL" | grep -oP 'runs/\K[0-9]+')
ATTEMPT=$(gh run view $RUN_ID --repo open-telemetry/opentelemetry-collector-contrib \
  --json attemptNumber --jq '.attemptNumber')
echo "Run $RUN_ID attempt: $ATTEMPT"
```

**If `attemptNumber == 1`** — first failure, retry all failed jobs:

```bash
gh run rerun $RUN_ID --failed --repo open-telemetry/opentelemetry-collector-contrib
```

Then leave the PR alone; it will either go green (approve in the next run) or fail again (handle below).

**If `attemptNumber > 1`** — already retried and still failing; this is likely a real incompatibility introduced by the dependency update. Convert the PR to draft and label it for code-owner attention:

```bash
gh pr ready $PR --undo --repo open-telemetry/opentelemetry-collector-contrib
gh pr edit $PR --add-label "waiting-for-code-owners" --repo open-telemetry/opentelemetry-collector-contrib
```

The `draft:false` filter in Step 1 will then exclude this PR from future shepherd runs automatically.

### Both 3a and 3b

**Critical: commit message must NOT contain `Co-Authored-By:`** — the EasyCLA check fails when a Co-Authored-By trailer references an account without CLA. The default `git merge` message and a plain `"Trigger CI"` empty-commit message are both fine; do not let the Claude trailer be injected. After committing, verify with `git log -1` before pushing.

## Step 4 — approve PRs with green CI

```bash
gh pr review $PR --repo open-telemetry/opentelemetry-collector-contrib --approve
```

"Green" here means `gh pr checks` returns only `pass` and `skipping` — no `pending` or `fail` — **after filtering out non-required checks**. The repo runs several optional/experimental jobs that are not part of branch protection and can fail or stay pending without blocking the merge:

- `build-and-test-experimental / exp-cross-compile-affected` (experimental cross-compile, often fails on `aix`/`ppc64`)
- `e2e-tests / kubernetes-test-matrix` (e2e tests against k8s versions, often slow to finish)
- `kubernetes-test` (kubernetes integration tests; not required, can fail without blocking)
- `supervisor-test` (supervisor integration tests; not required, can fail without blocking)

Treat these as informational: ignore their state when deciding whether to approve. A quick way to see the non-passing checks that actually matter:

```bash
gh pr checks $PR --repo open-telemetry/opentelemetry-collector-contrib \
  | awk -F'\t' '$2 != "pass" && $2 != "skipping" {print $1 "\t" $2}' \
  | grep -Ev '^(exp-cross-compile-affected|kubernetes-test-matrix|kubernetes-test|supervisor-test)'
```

If that prints nothing, the PR is effectively green. `mergeStateStatus: CLEAN` is the most reliable shortcut once review is also green — and after you approve, if the only blockers were these optional checks, BLOCKED should clear and the squash-merge will succeed.

## Step 5 — squash-merge approved + green PRs

```bash
gh pr merge $PR --repo open-telemetry/opentelemetry-collector-contrib --squash
```

**Do NOT pass `--auto`** — this repo does not enable auto-merge and the call returns `GraphQL: Auto merge is not allowed for this repository`.

### Step 5b — clean up local renovate branches after merging

Local `renovate/*` branches that you checked out for Step 3 refresh stick around after their PRs are merged. Squash-merge changes the SHA, so git sees them as "unmerged" — use `-D` (capital D):

```bash
git checkout main
git branch -D $(git branch | grep '^  renovate/' | tr -d ' ')
```

Or one at a time:

```bash
git branch -D renovate/<branch>
```

Skip any non-`renovate/*` branches (e.g. your own work branches). Do this at the end of each shepherd run so the local branch list stays scannable.

## Step 6 — resolving merge conflicts

The common conflict in renovate PRs is `go.mod` / `go.sum`, usually because two different module updates touched related lines (e.g. one PR bumped `google.golang.org/grpc`, another bumped `google.golang.org/api`).

### Resolution recipe for go.mod conflicts

1. **Take both higher versions** from each side. Renovate's branch has the newer version of the module the PR is updating; `main` typically has the newer version of whatever else has been merged in the meantime.
2. Edit each conflicted `go.mod` to remove the `<<<<<<<` / `=======` / `>>>>>>>` markers, keeping the higher version on each line.
3. For `go.sum` conflicts, take main's side as a starting point, then run `go mod tidy` in the affected module:

```bash
git checkout --theirs <path>/go.sum
git add <conflicted go.mod files>
(cd <module-dir> && go mod tidy)
git add <module-dir>/go.sum
```

4. **Watch the `go` directive.** This repo pins `go 1.25.0` in every `go.mod`. If `go mod tidy` (or any local Go toolchain) bumps it (e.g. to `go 1.26`), revert that line manually before committing — a higher `go` directive will fail CI and is unrelated to the renovate update.

5. Verify no markers remain and the version layout is sane:

```bash
grep -c "<<<<<<\|>>>>>>\|=======" $(git diff --name-only --diff-filter=U)
grep -H "^go " $(git diff --name-only | grep go.mod) | grep -v "go 1.25.0"  # should be empty
```

6. Commit with a clean merge message — **no `Co-Authored-By` trailer**:

```bash
git commit -m "Merge remote-tracking branch 'origin/main' into renovate/<branch>"
```

If a hook injects `Co-Authored-By:`, use `-c commit.gpgsign=false` only if needed for signing; do not skip CLA-relevant hooks. If the trailer still appears, edit the message via `git commit --amend` (without `--no-verify`).

7. Push: `git push origin renovate/<branch>`.

If the conflict is non-trivial (more than the go.mod/go.sum pattern above — e.g. real source code conflicts), abort with `git merge --abort`, leave the PR alone, and let Renovate regenerate it on its next run.

## Notes & gotchas

- **PR author vs. last commit author**: every renovate PR is opened by `app/renovate`. The "last commit by otelbot[bot]" pattern is the auto-generated `go mod tidy, make genotelcontribcol and make genoteltestbedcol` commit otelbot pushes to renovate branches.
- **Approving your own merge commit is OK**: when a maintainer (e.g. `songy23`) has a merge-from-main as the last commit, GitHub still allows them to approve since the PR author is `app/renovate`, not them. But you cannot approve a PR you authored yourself.
- **`mergeStateStatus` values**: `CLEAN` = ready to merge; `BLOCKED` = needs review or required check; `UNKNOWN` = GitHub still computing (common right after a push) — re-check after a few seconds.
- **PR #48134-style precedent**: previous shepherd runs from this repo are merged quickly by upstream maintainers; don't be surprised if a PR you saw 30 minutes ago is gone.
- **No Co-Authored-By, ever, in any commit on a renovate branch.** EasyCLA blocks the PR otherwise. If you accidentally commit one, push an amended commit before opening or refreshing the PR.
