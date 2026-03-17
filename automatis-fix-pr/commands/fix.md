# Fix PR Review

Fetch PR review comments from GitHub and fix identified issues in the codebase.

## When to Use

- After receiving PR review feedback
- To address automated code review comments (AI reviewers, linters)
- To systematically work through review issues

## Shell Safety Rules (MANDATORY)

When generating bash commands for this workflow:
- **NEVER** use `!= null` in jq expressions inside bash — bash history expansion corrupts `!` even in double quotes
- **NEVER** chain `--argjson` with shell variables — empty/invalid JSON breaks silently
- **USE Python** (`python3 -c`) for all multi-step comment filtering logic (open/closed detection, review-ID filtering)
- **COPY the exact code blocks** from this file — do NOT rewrite jq/Python from memory or training data
- Simple single-field `--jq` extractions are safe (e.g., `--jq '.user.login'`)

## Arguments

The user can provide:
- `/automatis-fix-pr:fix https://github.com/owner/repo/pull/123` - Full PR URL
- `/automatis-fix-pr:fix 123` - PR number (uses current repo)
- `/automatis-fix-pr:fix` - Prompts for PR URL or number

Optional flags in the argument:
- `--review=ID` - Focus on specific review ID
- `--severity=high,medium` - Filter by severity (high, medium, low)
- `--unresolved` - Only show comments without replies (default: true)

## Procedure

### Step 1: Parse Input and Determine Repository

Extract owner, repo, and PR number from the input:

```bash
# If full URL provided, parse it
# Example: https://github.com/owner/repo/pull/123
# Extract: owner=owner, repo=repo, pr=123

# If only PR number, get repo from current directory
gh repo view --json owner,name --jq '"\(.owner.login)/\(.name)"'
```

### Step 2: Fetch All Review Comments (WITH PAGINATION)

**CRITICAL**: Always use `per_page=100` to avoid missing comments.

```bash
# Get all review comments with pagination
gh api "repos/{owner}/{repo}/pulls/{pr}/comments?per_page=100" \
  --jq '.[] | {
    id: .id,
    review_id: .pull_request_review_id,
    path: .path,
    line: .line,
    body: .body,
    user: .user.login,
    created_at: .created_at,
    in_reply_to_id: .in_reply_to_id
  }'
```

If more than 100 comments exist, paginate:
```bash
gh api "repos/{owner}/{repo}/pulls/{pr}/comments?per_page=100&page=2"
```

### Step 3: Fetch Review Summaries

Get the review summaries to understand context:

```bash
gh api "repos/{owner}/{repo}/pulls/{pr}/reviews" \
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

**Algorithm (use Python for filtering — do NOT rewrite as jq):**
```bash
# 1. Get PR author
PR_AUTHOR=$(gh api "repos/{owner}/{repo}/pulls/{pr}" --jq '.user.login')

# 2. Fetch all comments and filter open ones via Python
OPEN_COMMENTS=$(gh api "repos/{owner}/{repo}/pulls/{pr}/comments?per_page=100" | python3 -c "
import sys, json
comments = json.load(sys.stdin)
author = '$PR_AUTHOR'
replied_ids = set(
    c['in_reply_to_id'] for c in comments
    if c['user']['login'] == author and c.get('in_reply_to_id')
)
open_comments = [
    c for c in comments
    if c.get('in_reply_to_id') is None and c['id'] not in replied_ids
]
print(json.dumps(open_comments, indent=2))
")
```

**A comment is CLOSED if the PR author has replied to it** - even if the reply says "won't fix" or "not applicable".

**IMPORTANT: Only process comments from the SPECIFIED or LATEST review**

GitHub shows comments from the most recent review. Older reviews' comments are hidden/superseded even if they're on valid lines. The API returns ALL comments from ALL reviews, but users only see the latest.

**ALWAYS filter by review ID:**

```bash
# If user provides review URL like #pullrequestreview-3684614038
# Extract review ID: 3684614038
REVIEW_ID=3684614038

# If no review ID provided, get the LATEST review:
REVIEW_ID=$(gh api "repos/{owner}/{repo}/pulls/{pr}/reviews" --jq '
  sort_by(.submitted_at) | last | .id
')

# Filter comments by review ID + open status (use Python — do NOT rewrite as jq):
```

**Complete filter for OPEN + CURRENT REVIEW comments (Python — do NOT rewrite as jq):**
```bash
PR_AUTHOR=$(gh api "repos/{owner}/{repo}/pulls/{pr}" --jq '.user.login')
REVIEW_ID=3684614038  # or get latest via jq above

OPEN_REVIEW_COMMENTS=$(gh api "repos/{owner}/{repo}/pulls/{pr}/comments?per_page=100" | python3 -c "
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
gh api "repos/{owner}/{repo}/pulls/{pr}/comments" \
  -X POST \
  -f body="Fixed: <brief description of what was changed and why>" \
  -f in_reply_to={comment_id}
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

## Common Severity Patterns in AI Reviews

| Pattern | Severity |
|---------|----------|
| `🔴 CRITICAL`, `:warning: HIGH` | HIGH |
| `🟠 HIGH`, `**HIGH**` | HIGH |
| `🟡 MEDIUM`, `:large_orange_diamond:` | MEDIUM |
| `🔵 LOW`, `:large_blue_diamond:` | LOW |

## Safety Rules

1. Always read the code before making changes
2. Never blindly apply suggestions without understanding context
3. Ask for clarification on ambiguous issues
4. Run tests after fixes if possible
5. Always reply to comments BEFORE pushing the branch
6. Show planned replies to user for confirmation before posting
7. Don't push without explicit user approval

## Example Session

```
User: /automatis-fix-pr:fix https://github.com/newsmill-ai/llm-site-parser/pull/4

Claude: Fetching PR #4 from newsmill-ai/llm-site-parser...

Found 3 unresolved review comments:
- MEDIUM: 1 (internal/browser/pool.go:339)
- LOW: 2 (internal/browser/pool.go:346, pkg/captcha/solver.go:299)

Which issues would you like me to fix?

User: All of them

Claude: Starting with internal/browser/pool.go:339...
[reads file, applies fix, moves to next]
...
All 3 issues fixed locally in 2 files.

Here are the replies I'll post to PR comments:

1. pool.go:339 → "Fixed: replaced raw proxy credential logging with masked output using sanitize() helper"
2. pool.go:346 → "Fixed: moved proxy verification to a goroutine with context timeout"
3. solver.go:299 → "Fixed: updated comment to accurately reflect the error propagation behavior"

Post these replies?

User: Yes

Claude: Replies posted. Ready to push — run `make lint && make test` first?

User: Tests pass, push it

Claude: Pushed. Summary:
- 3 issues fixed in 2 files
- 3 replies posted
- Branch pushed to origin
```
