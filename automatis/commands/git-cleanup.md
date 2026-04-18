---
description: Clean up local git branches — create PRs for unmerged work, delete merged/gone branches, end on up-to-date main
argument-hint: "[--dry-run] [--no-pr]"
allowed-tools: Bash
---

# Cleanup Local Git Branches

Enumerate local branches, detect which are merged (including squash-merges via GitHub PR status), offer to create PRs for unmerged work, delete merged/gone branches, and finish on an up-to-date main branch.

## When to Use

- After a sprint where several PRs have been merged and local branches have piled up
- When `git branch` output is cluttered and you can't tell what's still active
- Before starting new work, to guarantee a clean slate starting from an updated main
- For the narrower case of purging only branches whose remote was deleted (`[gone]`), the `/commit-commands:clean_gone` skill is a simpler non-interactive alternative

## Shell Safety

See `CLAUDE.md` → **Shell Safety** at the repo root. Copy the bash/Python blocks below exactly — do not rewrite from memory. In particular, multi-field filtering over `gh` output MUST use `python3`, never `jq` with `!=`.

## Arguments

The user can provide:
- `/automatis:git-cleanup` — interactive mode, prompts for confirmation at each decision
- `/automatis:git-cleanup --dry-run` — print the categorization and exit; no mutations
- `/automatis:git-cleanup --no-pr` — skip PR creation for unmerged branches (still categorize and delete where safe)
- `/automatis:git-cleanup --dry-run --no-pr` — flags combine

## Procedure

### Step 1: Preflight — Refuse on Dirty Worktree

**CRITICAL**: Never proceed with a dirty worktree. Auto-stashing hides state and causes confusion when `stash pop` conflicts later.

```bash
if ! git diff --quiet || ! git diff --cached --quiet || [ -n "$(git ls-files --others --exclude-standard)" ]; then
  echo "Working tree is dirty:"
  git status --short
  echo ""
  echo "Stash or commit your changes first, then re-run /automatis:git-cleanup."
  exit 1
fi
```

### Step 2: Detect Main Branch and Repo

The default branch is not always `main`. And `gh` resolves the repo from the current directory, which breaks when run from a worktree or a submodule. Bind explicit values now; reuse everywhere below.

```bash
MAIN=$(git symbolic-ref refs/remotes/origin/HEAD --short 2>/dev/null | sed 's|^origin/||')
MAIN=${MAIN:-main}
OWNER=$(gh repo view --json owner --jq '.owner.login')
REPO=$(gh repo view --json name --jq '.name')
echo "Main branch: $MAIN   Repo: $OWNER/$REPO"
```

### Step 3: Sync With Remote

```bash
git fetch --all --prune
```

`--prune` removes remote-tracking refs for branches deleted on GitHub — this is what makes `[gone]` detection in Step 4 accurate.

### Step 4: Gather Branch State

One Python block: enumerate local branches, capture worktree pinning, compute the merged set, and fetch PR status for every branch via a **single bulk `gh pr list` call** (not per-branch). **Use Python — do NOT rewrite as jq** (per repo shell-safety rule).

```bash
BRANCH_STATE=$(MAIN="$MAIN" OWNER="$OWNER" REPO="$REPO" python3 <<'PY'
import subprocess, json, re, os, sys, shlex

MAIN = os.environ['MAIN']
REPO_SLUG = f"{os.environ['OWNER']}/{os.environ['REPO']}"

def run(cmd):
    r = subprocess.run(cmd, capture_output=True, text=True)
    if r.returncode != 0:
        sys.exit(f"Command failed: {shlex.join(cmd)}\n{r.stderr.strip()}")
    return r.stdout

main_wt = run(['git', 'rev-parse', '--show-toplevel']).strip()
worktrees = {}
path = None
for line in run(['git', 'worktree', 'list', '--porcelain']).splitlines():
    if line.startswith('worktree '):
        path = line[len('worktree '):].strip()
    elif line.startswith('branch refs/heads/') and path and path != main_wt:
        worktrees[line[len('branch refs/heads/'):].strip()] = path

merged_set = set()
for line in run(['git', 'branch', '--merged', MAIN]).splitlines():
    name = line.lstrip('*').strip()
    if name and name != MAIN:
        merged_set.add(name)

def count(kind, track):
    m = re.search(rf'{kind} (\d+)', track)
    return int(m.group(1)) if m else 0

branches = []
raw = run(['git', 'for-each-ref', '--format=%(refname:short)%00%(upstream:track)', 'refs/heads'])
for record in raw.split('\n'):
    if not record:
        continue
    name, _, track = record.partition('\x00')
    if name == MAIN:
        continue
    branches.append({
        'name': name,
        'ahead': count('ahead', track),
        'gone': '[gone]' in track,
        'worktree_path': worktrees.get(name),
        'pr_state': None, 'pr_url': None, 'pr_number': None,
    })

if not branches:
    print(json.dumps({'branches': [], 'merged': []}))
    sys.exit(0)

# Single bulk PR lookup — one HTTP round-trip for the whole repo,
# vs. one per branch. --limit 500 covers all but the oldest repos.
pr_raw = run(['gh', 'pr', 'list', '--repo', REPO_SLUG,
              '--state', 'all', '--limit', '500',
              '--json', 'number,state,url,headRefName'])
by_head = {}
order = {'MERGED': 0, 'OPEN': 1, 'CLOSED': 2}
for pr in json.loads(pr_raw or '[]'):
    by_head.setdefault(pr['headRefName'], []).append(pr)
for head, prs in by_head.items():
    prs.sort(key=lambda p: order.get(p['state'], 9))

for b in branches:
    prs = by_head.get(b['name'])
    if prs:
        b['pr_state'] = prs[0]['state']
        b['pr_url'] = prs[0]['url']
        b['pr_number'] = prs[0]['number']

print(json.dumps({
    'branches': branches,
    'merged': sorted(merged_set),
}, indent=2))
PY
)

# Early exit: nothing to do.
if [ "$(printf '%s' "$BRANCH_STATE" | python3 -c 'import sys,json; d=json.load(sys.stdin); print(len(d["branches"]))')" = "0" ]; then
  echo "No non-main local branches. Nothing to clean up."
  exit 0
fi
```

### Step 5: Categorize Each Branch

Read the JSON in `$BRANCH_STATE` and assign each branch to exactly one bucket. **Evaluate rules in this exact order** — `WORKTREE_PINNED` must come first because a pinned branch cannot be deleted regardless of merge state.

| Order | Bucket | Criteria | Action |
|-------|--------|----------|--------|
| 1 | `WORKTREE_PINNED` | `worktree_path` is set | Never delete — report path only |
| 2 | `MERGED_GIT` | `name` in the `merged` set | Delete with `git branch -d` (safe) |
| 3 | `MERGED_PR` | `pr_state == "MERGED"` | Delete with `git branch -D` (squash-merge SHAs diverge) |
| 4 | `GONE` | `gone == true` AND `pr_state` not in {`MERGED`, `OPEN`} | Confirm batch, delete with `-D` |
| 5 | `OPEN_PR` | `pr_state == "OPEN"` | Report URL, skip |
| 6 | `CLOSED_PR` | `pr_state == "CLOSED"` | Report URL, ask per-branch whether to force-delete |
| 7 | `NO_PR` | `ahead > 0` AND `pr_state is None` | Offer to push + `gh pr create --fill` |

### Step 6: Present Plan and Gather Confirmations

Print a grouped summary. `MERGED_GIT` and `MERGED_PR` are collapsed under a single `MERGED` header in the display (both delete cleanly); the per-row annotation identifies which sub-bucket each branch came from.

```
Main: main (0 commits behind origin)

MERGED (5) — will delete:
  - feat/x          (merged via PR #421)
  - fix/y           (merged by git)
  - chore/z         (merged via PR #418)
  ...

GONE (2) — remote deleted:
  - legacy/abc
  - old/experiment

OPEN_PR (1) — skip:
  - feat/new-thing  → https://github.com/o/r/pull/440

CLOSED_PR (1) — manual review:
  - experiment/xyz  → https://github.com/o/r/pull/399 (closed, not merged)

NO_PR (2) — create PR?
  - feat/draft-1    (3 commits ahead of main)
  - fix/wip-2       (1 commit ahead of main)

WORKTREE_PINNED (1) — will NOT touch:
  - feat/other      → /Users/mmykola87/work/wt-other
```

Then prompt:

- **For each `NO_PR` branch**: "Create PR for `<branch>`? [y / N / skip-all]". `skip-all` ends all NO_PR prompts.
- **For each `CLOSED_PR` branch**: "Force-delete `<branch>` (PR #XXX was closed without merging)? [y / N]".
- **For `GONE` bucket**: single batch prompt "Delete all GONE branches? [y / N]" (the remote already signaled intent by deleting).

**IMPORTANT**: If `--dry-run` is set, print the summary and exit **before** any prompts or mutations. If `--no-pr` is set, skip the NO_PR prompt loop entirely (still delete merged/gone).

### Step 7: Execute Mutations

**CRITICAL**: Order matters. Do NOT combine `git add`, `git commit`, `git push` in a single compound command (repo-wide rule).

**7a. Create PRs for confirmed NO_PR branches** (one branch at a time):

```bash
git push -u origin "$BRANCH"
gh pr create --fill --base "$MAIN" --head "$BRANCH"
```

**CRITICAL**: Never pass `--no-verify`. Pre-push hooks are a hard gate on this machine (global rule). If a hook blocks, log the failure, skip this branch, and continue — do not attempt to bypass.

**7b. Switch to main before deleting any branch that is currently checked out:**

```bash
git checkout "$MAIN"
git pull --ff-only
```

**7c. Delete branches** (in this order):

```bash
# MERGED_GIT — safe delete (fails if the branch isn't actually merged, which catches bugs).
git branch -d "$BRANCH"

# MERGED_PR, GONE, confirmed CLOSED_PR — force delete (diverged SHAs are expected here).
git branch -D "$BRANCH"
```

Log each deletion with its bucket name.

**7d. Final main update** (in case PRs merged while we worked):

```bash
git pull --ff-only
```

### Step 8: Summary

Report:
- **Deleted**: count + names grouped by bucket (`MERGED_GIT`, `MERGED_PR`, `GONE`, `CLOSED_PR`)
- **PRs created**: count + URLs
- **Skipped**: count + reason (open PR, worktree-pinned, closed-PR declined, push hook failed)
- **Current state**: `git branch --show-current` (should be `$MAIN`) and `git status -sb` (should be clean, up-to-date with origin)

## Safety Rules

1. Refuse to run on a dirty worktree — never auto-stash, never auto-discard.
2. Never pass `--no-verify` to `git push`. If a pre-push hook blocks, report the failure and skip that branch; do not bypass.
3. Never delete a branch that is checked out in another worktree — only report its path.
4. Use `git branch -d` (safe) for `MERGED_GIT`; `-D` (force) is allowed only for buckets where divergence is expected (`MERGED_PR`, `GONE`, confirmed `CLOSED_PR`).
5. `--dry-run` is non-destructive: no push, no PR create, no branch delete, no checkout.
6. Ask per-branch before creating each PR and before deleting any `CLOSED_PR` branch. Batch confirmation is allowed only for the `GONE` bucket.
7. Never combine `git add`, `git commit`, and `git push` in one compound command.
8. Only switch away from the current branch after all confirmations are collected — an aborted run should leave the user on the branch they started on.

## Example Session

```
User: /automatis:git-cleanup

Claude: Main branch: main
Fetching origin with --prune...

Summary:
  MERGED (3) — will delete:
    - feat/login-cleanup  (merged via PR #412)
    - fix/typo-in-readme  (merged by git)
    - chore/bump-deps     (merged via PR #410)
  GONE (1) — remote deleted:
    - legacy/old-api
  NO_PR (1) — create PR?
    - feat/draft-profile  (2 commits ahead of main)
  OPEN_PR (1) — skip:
    - feat/new-billing    → https://github.com/acme/app/pull/415

Create PR for feat/draft-profile? [y / N / skip-all]
User: y

Claude: Pushing feat/draft-profile...
  Pre-push hook passed. Opening PR...
  → https://github.com/acme/app/pull/416

Delete all GONE branches? [y / N]
User: y

Claude: Switching to main...
Deleting 3 MERGED branches:
  ✓ git branch -d feat/login-cleanup
  ✓ git branch -d fix/typo-in-readme
  ✓ git branch -d chore/bump-deps
Deleting 1 GONE branch:
  ✓ git branch -D legacy/old-api
Pulling main...

Summary:
  Deleted:     4 branches (3 MERGED, 1 GONE)
  PRs created: 1 (https://github.com/acme/app/pull/416)
  Skipped:     1 (feat/new-billing — open PR #415)
  Current:     main (up-to-date with origin/main)
```
