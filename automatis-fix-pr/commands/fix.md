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
5. **Mark as addressed** (optionally reply to the comment)

### Step 8: Optional - Reply to Comments

After fixing, optionally add replies to mark issues as addressed:

```bash
gh api "repos/{owner}/{repo}/pulls/{pr}/comments" \
  -X POST \
  -f body="Fixed in latest commit" \
  -f in_reply_to={comment_id}
```

### Step 9: Summary

Report:
- Number of issues fixed
- Files modified
- Any issues skipped and why
- Suggest running tests/lint before committing

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
5. Don't auto-commit - let user review changes first

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
Done! Fixed 3 issues in 2 files.
Run `make lint && make test` to verify.
```
