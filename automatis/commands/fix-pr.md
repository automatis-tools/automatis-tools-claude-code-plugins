---
description: Fix open review comments on a GitHub PR
argument-hint: "[pr-url | pr-number] [--review=ID] [--severity=high,medium,low,critical]"
allowed-tools: Bash, Read, Edit, Grep, Glob
---

# Fix PR Review Comments

Fetch PR review comments from GitHub and fix identified issues in the codebase.

## When to Use

- After receiving PR review feedback
- To address automated code review comments (AI reviewers, linters)
- To systematically work through review issues

## Shell Safety

See `CLAUDE.md` → **Shell Safety** at the repo root. Copy the bash/Python blocks below exactly — do not rewrite from memory.

## Arguments

The user can provide:
- `/automatis:fix-pr https://github.com/owner/repo/pull/123` - Full PR URL
- `/automatis:fix-pr 123` - PR number (uses current repo)
- `/automatis:fix-pr` - Prompts for PR URL or number

Optional flags in the argument:
- `--review=ID` - Focus on specific review ID
- `--severity=high,medium` - Filter by severity. Accepted: `critical`, `high`, `medium`, `low`. `critical` is treated as `high`.
- `--unresolved` - Only show comments without replies (default: true)

## Procedure

### Step 1: Parse Input and Bind Shell Variables

Bind three shell variables that every later step uses:

```bash
# Case A — full URL like https://github.com/ACME/webapp/pull/123:
#   OWNER=ACME  REPO=webapp  PR_NUMBER=123
# Case B — bare PR number (uses the current repo):
OWNER=$(gh repo view --json owner --jq '.owner.login')
REPO=$(gh repo view --json name --jq '.name')
PR_NUMBER=<pr-number>
```

Every subsequent `gh api` call uses these explicit variables (`repos/$OWNER/$REPO/pulls/$PR_NUMBER/...`). Do not rely on `gh`'s native `{owner}`/`{repo}` templating — it breaks when the target PR is in a different repo than the current directory.

### Step 2: Comment Schema

Step 4 does the actual filtered fetch. For reference, the comments endpoint is `repos/$OWNER/$REPO/pulls/$PR_NUMBER/comments?per_page=100` (append `&page=2` etc. for >100 comments). The fields Claude cares about per comment: `id`, `pull_request_review_id`, `path`, `line`, `body`, `user.login`, `in_reply_to_id`. **CRITICAL**: never drop `per_page=100` — the default page size hides comments.

### Step 3: Fetch Review Summaries

Get the review summaries to understand context:

```bash
gh api "repos/$OWNER/$REPO/pulls/$PR_NUMBER/reviews" \
  --jq '.[] | {
    id: .id,
    state: .state,
    body: .body,
    user: .user.login,
    submitted_at: .submitted_at
  }'
```

### Step 4: Identify OPEN Comments (No Author Reply)

**CRITICAL**: Only process comments that are truly OPEN (have no reply from the PR author).

**Definition of OPEN comment:**
1. It's a top-level review comment (`in_reply_to_id` is null)
2. There is **NO reply from the PR author** to that comment

**A comment is CLOSED if the PR author has replied to it** - even if the reply says "won't fix" or "not applicable".

**IMPORTANT: Only process comments from the SPECIFIED or LATEST review**

GitHub shows comments from the most recent review. Older reviews' comments are hidden/superseded even if they're on valid lines. The API returns ALL comments from ALL reviews, but users only see the latest.

**ALWAYS filter by review ID:**

```bash
# Set REVIEW_ID from one of:
#   (a) User-provided review URL like .../pull/123#pullrequestreview-3684614038
#       → extract the trailing number, e.g. REVIEW_ID=3684614038
#   (b) Latest review on the PR (default if user gave no review URL):
REVIEW_ID=$(gh api "repos/$OWNER/$REPO/pulls/$PR_NUMBER/reviews" --jq '
  sort_by(.submitted_at) | last | .id // empty
')

# Guard: if PR has no reviews at all, REVIEW_ID is empty — nothing to do.
if [ -z "$REVIEW_ID" ]; then
  echo "PR #$PR_NUMBER has no reviews — nothing to process."
  exit 0
fi
```

**Complete filter for OPEN + CURRENT REVIEW comments (Python — do NOT rewrite as jq):**
```bash
# Assumes REVIEW_ID is already set (see block above). PR_AUTHOR fetched fresh.
PR_AUTHOR=$(gh api "repos/$OWNER/$REPO/pulls/$PR_NUMBER" --jq '.user.login')

OPEN_REVIEW_COMMENTS=$(gh api "repos/$OWNER/$REPO/pulls/$PR_NUMBER/comments?per_page=100" | python3 -c "
import sys, json
comments = json.load(sys.stdin)
author = '$PR_AUTHOR'
review_id = $REVIEW_ID
replied_ids = set(
    c['in_reply_to_id'] for c in comments
    if c['user']['login'] == author and c.get('in_reply_to_id')
)
open_comments = [
    c for c in comments
    if c.get('pull_request_review_id') == review_id
    and c.get('in_reply_to_id') is None
    and c['id'] not in replied_ids
]
print(json.dumps(open_comments, indent=2))
")
```

Severity detection patterns:
- **CRITICAL/HIGH**: `:warning:`, `**HIGH**`, `CRITICAL`, `🔴`, `🟠`
- **MEDIUM**: `:large_orange_diamond:`, `**MEDIUM**`, `🟡`
- **LOW**: `:large_blue_diamond:`, `**LOW**`, `🔵`

### Step 5: Group Issues by File

Organize unresolved comments by file path:

```
internal/browser/pool.go:
  - Line 339: [MEDIUM] Proxy credentials logged with insufficient masking
  - Line 346: [LOW] Proxy verification blocks on synchronous network call

pkg/captcha/solver.go:
  - Line 299: [LOW] Misleading invariant comment about error handling
```

### Step 6: Present Issues and Confirm

Show the user:
1. Total number of unresolved issues by severity
2. Grouped list of issues by file
3. Ask which issues to fix (all, by severity, or specific ones)

```
Found 3 unresolved review comments:
- MEDIUM: 1
- LOW: 2

Which issues do you want to fix?
1. All issues
2. Only HIGH/MEDIUM severity
3. Specific files (list them)
4. Let me review each one
```

### Step 7: Fix Each Issue

For each issue to fix:

1. **Read the relevant code** around the line number
2. **Understand the issue** from the review comment
3. **Determine the fix**:
   - If the comment includes a suggestion, apply it
   - If it's a code smell, refactor appropriately
   - If it's unclear, ask the user for clarification
4. **Apply the fix** using Edit tool
5. **Track what was done** — remember the comment ID and a brief description of the fix for Step 8

### Step 8: Reply to Comments

**IMPORTANT**: Reply to ALL fixed comments BEFORE pushing the branch. This ensures the reviewer sees your explanations before being notified of new commits.

For each fixed comment, reply with a brief description of the actual fix applied:

```bash
gh api "repos/$OWNER/$REPO/pulls/$PR_NUMBER/comments" \
  -X POST \
  -f body="Fixed: <brief description of what was changed and why>" \
  -f in_reply_to=$COMMENT_ID  # set per-comment in your loop
```

**Guidelines for reply content:**
- Be specific about what was changed (e.g. "Fixed: replaced raw credential logging with masked output using `sanitize()` helper")
- If you disagreed with the suggestion and took a different approach, explain why
- If a comment was skipped, reply explaining the reason (e.g. "Not applicable because..." or "Skipped — needs clarification on...")

**Before posting replies**, show the user the list of planned replies and ask for confirmation.

### Step 9: Push Changes

After all replies are posted:

1. **Run tests/lint** to verify fixes don't break anything
2. **Commit** the changes with a descriptive message referencing the PR
3. **Push** the branch

**CRITICAL**: Only push AFTER replies are posted. If you push first, the reviewer gets notified of new commits and may start re-reviewing before seeing your reply context.

### Step 10: Summary

Report:
- Number of issues fixed
- Files modified
- Replies posted (count and any notable ones)
- Any issues skipped and why
- Branch push status

## Safety Rules

1. Read the code before applying any suggested fix; ask or skip when intent is unclear.
2. Run tests/lint after fixes when the project has them.
3. Show planned replies for confirmation; post before pushing (Step 8 `IMPORTANT`).
4. Push only with explicit user approval (Step 9 `CRITICAL`).

## Example Session

```
User: /automatis:fix-pr https://github.com/owner/repo/pull/4

Claude: 3 unresolved comments in review #3684614038:
  MEDIUM × 1 (pool.go:339)
  LOW × 2 (pool.go:346, solver.go:299)
Fix all?

User: yes

Claude: [fixes each, tracks reply text per comment]
Planned replies:
  pool.go:339  → "Fixed: masked proxy credentials via sanitize() helper"
  pool.go:346  → "Fixed: moved verify to goroutine with context timeout"
  solver.go:299 → "Fixed: corrected comment about error propagation"
Post replies, then push?

User: yes

Claude: 3 replies posted, branch pushed to origin.
```
