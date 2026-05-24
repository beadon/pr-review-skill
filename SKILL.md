---
allowed-tools: Bash(gh:*), Bash(git clone:*), Bash(git log:*), Bash(git diff:*), Bash(mkdir:*), Bash(grep:*), Bash(find:*), Bash(wc:*), Bash(jq:*), Read, Write
description: Comprehensive GitHub PR code review — fetches diff/metadata/comments via gh CLI, writes review files, posts only on /send or /send-decline
version: "1.0.0"
---

## Overview

This is a **two-stage PR review**. All analysis happens locally first. **Nothing is posted to GitHub until you run `/send` or `/send-decline`.**

---

## Step 1 — Parse PR Reference

`$ARGUMENTS` will be one of:
- A full GitHub PR URL: `https://github.com/owner/repo/pull/123`
- A PR number with repo: `owner/repo 123` or `owner/repo#123`
- A PR number alone (uses the current repo): `123`

Parse `$ARGUMENTS` to extract:
- `REPO` — e.g. `ddclient/ddclient`
- `PR_NUM` — numeric PR number
- `REVIEW_DIR` — `/tmp/pr-review/${REPO}/${PR_NUM}`

If `$ARGUMENTS` is empty or cannot be parsed, ask the user for the PR URL or number before continuing.

---

## Step 2 — Fetch PR Data

Run each command below. Create the workspace first:

```bash
mkdir -p "/tmp/pr-review/${REPO}/${PR_NUM}/pr"
```

**PR metadata** (one call, captures all needed fields):
```bash
gh pr view ${PR_NUM} --repo ${REPO} \
  --json number,title,body,author,state,headRefName,baseRefName,additions,deletions,files,commits,labels,assignees,reviewRequests \
  > "/tmp/pr-review/${REPO}/${PR_NUM}/metadata.json"
```

**Diff**:
```bash
gh pr diff ${PR_NUM} --repo ${REPO} \
  > "/tmp/pr-review/${REPO}/${PR_NUM}/diff.patch"
```

**Inline code review comments** (single call with --jq):
```bash
gh api "/repos/${REPO}/pulls/${PR_NUM}/comments" \
  --jq '[.[] | {path,line,body,user_login:.user.login,created_at}]' \
  > "/tmp/pr-review/${REPO}/${PR_NUM}/inline_comments.json"
```

**PR review summaries** (approvals, request-changes, etc.):
```bash
gh api "/repos/${REPO}/pulls/${PR_NUM}/reviews" \
  --jq '[.[] | {user_login:.user.login,state,body,submitted_at}]' \
  > "/tmp/pr-review/${REPO}/${PR_NUM}/reviews.json"
```

**General issue/PR comments**:
```bash
gh api "/repos/${REPO}/issues/${PR_NUM}/comments" \
  --jq '[.[] | {user_login:.user.login,body,created_at}]' \
  > "/tmp/pr-review/${REPO}/${PR_NUM}/issue_comments.json"
```

**Latest commit SHA** (needed for inline comment posting later):
```bash
gh pr view ${PR_NUM} --repo ${REPO} \
  --json commits --jq '.commits[-1].oid' \
  > "/tmp/pr-review/${REPO}/${PR_NUM}/head_sha.txt"
```

Read and reason over each fetched file before proceeding.

---

## Step 3 — Analyze

Review the fetched data against all categories below. Document findings as you go — you will write them to files in Step 4.

### Review Categories

| Category | Key Questions |
|---|---|
| **Functionality** | Does the code solve the stated problem? Edge cases handled? Regressions possible? |
| **Correctness** | Logic errors? Off-by-one? Null/undef dereferences? |
| **Readability** | Meaningful names? DRY? Comments explain *why* not *what*? |
| **Style** | Consistent with codebase conventions? Follows project patterns? |
| **Performance** | Any O(n²) where O(n) would do? Unnecessary repeated calls? |
| **Security** | Injection? Unvalidated external input? Credential exposure? |
| **Testing** | Tests exist? Cover success and error paths? Assertions meaningful? |
| **PR Quality** | Focused scope? Commit messages clear? Description accurate? |

### Finding Priority Markers

- **Blocker** — must be fixed before merge
- **Important** — should be addressed
- **Nit** — minor style or preference
- **Suggestion** — consider for future
- **Question** — needs clarification from author
- **Praise** — acknowledge excellent work

### Analysis Steps

1. Read `metadata.json` — understand scope, linked issues, existing reviews
2. Read `diff.patch` — walk through every changed file
3. Read `inline_comments.json` and `reviews.json` — note what reviewers already flagged (avoid duplicating)
4. Read `issue_comments.json` — any discussion context
5. Check commit messages for clarity and accuracy
6. Apply the category checklist above

---

## Step 4 — Write Review Files

Write two files using the Write tool:

### `review.md` — Internal detailed review

Path: `/tmp/pr-review/${REPO}/${PR_NUM}/pr/review.md`

Include:
- PR metadata summary (title, author, branch, size)
- Overall assessment
- Findings grouped by priority (Blocker → Important → Nit → Suggestion → Question → Praise)
- Each finding: category, file:line if applicable, issue, recommendation
- Proposed inline comments (file, line, comment text)

Use whatever formatting is useful for your own analysis — this file is not posted.

### `human.md` — Review text to post to GitHub

Path: `/tmp/pr-review/${REPO}/${PR_NUM}/pr/human.md`

Rules for this file:
- No emojis
- No em-dashes — use commas or semicolons instead
- No line number references (GitHub renders diffs, not line numbers in comment bodies)
- No internal analysis notes
- Direct, professional tone — consistent with project communication style
- Lead with a brief summary paragraph, then findings grouped by priority
- End with overall recommendation: approve / request changes / comment

---

## Step 5 — Create /send and /send-decline Commands

Write these two command files so the user can post the review when ready.

**`~/.claude/commands/send.md`** — content:
```
Post the prepared review for PR #${PR_NUM} in ${REPO} as an **approval**.

Read the review body from: /tmp/pr-review/${REPO}/${PR_NUM}/pr/human.md

Post it with:
  gh pr review ${PR_NUM} --repo ${REPO} --approve --body "$(cat /tmp/pr-review/${REPO}/${PR_NUM}/pr/human.md)"

Confirm success and show the URL of the posted review.
Then delete ~/.claude/commands/send.md and ~/.claude/commands/send-decline.md so stale commands don't persist.
```

**`~/.claude/commands/send-decline.md`** — content:
```
Post the prepared review for PR #${PR_NUM} in ${REPO} as a **request for changes**.

Read the review body from: /tmp/pr-review/${REPO}/${PR_NUM}/pr/human.md

Post it with:
  gh pr review ${PR_NUM} --repo ${REPO} --request-changes --body "$(cat /tmp/pr-review/${REPO}/${PR_NUM}/pr/human.md)"

Confirm success and show the URL of the posted review.
Then delete ~/.claude/commands/send.md and ~/.claude/commands/send-decline.md so stale commands don't persist.
```

Replace `${PR_NUM}` and `${REPO}` with the actual values (not shell variables) when writing these files.

---

## Step 6 — Present to User

Show the user:
1. A brief summary of findings (counts by priority level)
2. The path to `review.md` for detailed reading
3. The path to `human.md` for the draft post
4. Instructions: "Edit `human.md` if needed, then run `/send` to approve or `/send-decline` to request changes"

**Do not post anything to GitHub until the user runs `/send` or `/send-decline`.**

---

## Inline Comments (Optional, Post-Review)

After `/send` or `/send-decline`, individual inline comments can be added with:

```bash
HEAD_SHA=$(cat /tmp/pr-review/${REPO}/${PR_NUM}/head_sha.txt)

gh api "/repos/${REPO}/pulls/${PR_NUM}/comments" \
  --method POST \
  --field body="COMMENT_TEXT" \
  --field commit_id="${HEAD_SHA}" \
  --field path="path/to/file" \
  --field line=LINE_NUMBER \
  --field side="RIGHT"
```

List proposed inline comments from `review.md` and ask the user which (if any) to post before running.
