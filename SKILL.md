---
name: pr-review
allowed-tools: Bash(gh:*), Bash(git clone:*), Bash(git log:*), Bash(git diff:*), Bash(mkdir:*), Bash(grep:*), Bash(find:*), Bash(wc:*), Bash(jq:*), Read, Write
description: Comprehensive GitHub PR code review — fetches diff/metadata/comments via gh CLI, writes review files, posts only on /send or /send-decline
version: "1.4.0"
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
| **Correctness** | Logic errors? Off-by-one? Null/undef dereferences? Guard clauses used to reduce nesting rather than deep if/else chains? What are the failure modes — do they fail loudly, silently, or dangerously? For changes to shared, base, or widely-used code: is the impact radius understood? Could this break callers in non-obvious ways? Concurrency: race conditions, shared mutable state, or missing locks in concurrent contexts? |
| **Readability** | Meaningful names? DRY? Comments explain *why* not *what*? Magic numbers replaced with named constants? Functions kept to a reasonable length, with abstraction applied where appropriate? Are there one-use trivial helpers or wrapper classes that add indirection without value? Is there a simpler built-in or standard library method that could replace custom logic? |
| **Style** | Consistent with codebase conventions? Follows project patterns? |
| **Performance** | Any O(n²) where O(n) would do? Unnecessary repeated calls? |
| **Security** | Injection? Unvalidated external input? Credential exposure? |
| **Testing** | Tests exist? Cover success and error paths? Assertions meaningful? Does the PR description show evidence the author ran and tested the change locally? Are any existing assertions weakened, removed, or replaced with tautologies to make tests pass? |
| **PR Quality** | Focused scope? Commit messages clear? Description accurate? PR not in draft/WIP without proper designation? For user-facing changes, is a changelog or NEWS entry included? Is this change solving the right problem — not just implementing what was literally asked? |

**Testing — additional checks:**
- Does the stub or mock target the exact symbol production code calls? A stub wired to the wrong namespace or import path never fires (e.g., Perl: `CORE::sleep` vs `ddclient::sleep`; Python: the import path at call time, not the definition path).
- Would this test still fail if the production logic it targets were removed or broken? If not, the test provides no regression coverage.
- Are bare numeric literals in test assertions derived from configurable defaults? A default value is not a correctness invariant — tests should use the configurable parameter directly rather than hardcoding the default's computed value (e.g., asserting jitter is always `< interval * 0.2` when `0.2` is just the default percentage, not a fixed bound).

### Finding Priority Markers

- **Blocker** — must be fixed before merge
- **Important** — should be addressed
- **Nit** — minor style or preference
- **Suggestion** — consider for future
- **Question** — needs clarification from author
- **Praise** — acknowledge excellent work

### Analysis Steps

1. Read `metadata.json` — understand scope, linked issues, existing reviews
2. Check for a project conventions file (`CLAUDE.md`, `CONTRIBUTING.md`, or similar) at the repo root and factor its rules into the review
3. Read the **test changes** in `diff.patch` first, before the implementation — weakened or removed assertions are easiest to spot in isolation, before the implementation anchors your expectations
4. Read the **implementation changes** in `diff.patch` — walk through every changed file
5. Read `inline_comments.json` and `reviews.json` — note what reviewers already flagged (avoid duplicating)
6. Read `issue_comments.json` — any discussion context
7. Check commit messages for clarity and accuracy
8. Apply the category checklist above

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
- Proposed inline comments (file, line, side, comment body; for concrete code fixes include a ` ```suggestion ` block — renders as a one-click Apply button in GitHub)

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

1. Read the commit SHA from: /tmp/pr-review/${REPO}/${PR_NUM}/head_sha.txt
2. Read proposed inline comments from: /tmp/pr-review/${REPO}/${PR_NUM}/pr/review.md
3. Read the review body from: /tmp/pr-review/${REPO}/${PR_NUM}/pr/human.md

Step A — Create a PENDING review, bundling all proposed inline comments:

  gh api "/repos/${REPO}/pulls/${PR_NUM}/reviews" -X POST \
    -f commit_id="HEAD_SHA" \
    -f 'comments[][path]=PATH' -F 'comments[][line]=LINE' -f 'comments[][side]=RIGHT' -f 'comments[][body]=BODY' \
    --jq '{id,state}'

  Repeat the four comments[][] fields for each inline comment.
  Omit all comments[][] fields if there are no inline comments.
  For multi-line ranges: prepend -F 'comments[][start_line]=N' before the line field.
  Use side=LEFT for deleted lines, side=RIGHT for added or context lines.
  For concrete code fixes: include a ```suggestion block in the body.

Step B — Submit the pending review using the id returned from Step A:

  gh api "/repos/${REPO}/pulls/${PR_NUM}/reviews/REVIEW_ID/events" -X POST \
    -f event="APPROVE" \
    -f body="$(cat /tmp/pr-review/${REPO}/${PR_NUM}/pr/human.md)"

Confirm success and show the URL of the posted review.
Then delete ~/.claude/commands/send.md and ~/.claude/commands/send-decline.md so stale commands don't persist.
```

**`~/.claude/commands/send-decline.md`** — content:
```
Post the prepared review for PR #${PR_NUM} in ${REPO} as a **request for changes**.

1. Read the commit SHA from: /tmp/pr-review/${REPO}/${PR_NUM}/head_sha.txt
2. Read proposed inline comments from: /tmp/pr-review/${REPO}/${PR_NUM}/pr/review.md
3. Read the review body from: /tmp/pr-review/${REPO}/${PR_NUM}/pr/human.md

Step A — Create a PENDING review, bundling all proposed inline comments:

  gh api "/repos/${REPO}/pulls/${PR_NUM}/reviews" -X POST \
    -f commit_id="HEAD_SHA" \
    -f 'comments[][path]=PATH' -F 'comments[][line]=LINE' -f 'comments[][side]=RIGHT' -f 'comments[][body]=BODY' \
    --jq '{id,state}'

  Repeat the four comments[][] fields for each inline comment.
  Omit all comments[][] fields if there are no inline comments.
  For multi-line ranges: prepend -F 'comments[][start_line]=N' before the line field.
  Use side=LEFT for deleted lines, side=RIGHT for added or context lines.
  For concrete code fixes: include a ```suggestion block in the body.

Step B — Submit the pending review using the id returned from Step A:

  gh api "/repos/${REPO}/pulls/${PR_NUM}/reviews/REVIEW_ID/events" -X POST \
    -f event="REQUEST_CHANGES" \
    -f body="$(cat /tmp/pr-review/${REPO}/${PR_NUM}/pr/human.md)"

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

## Inline Comment Syntax Reference

Inline comments are bundled into the PENDING review as part of `/send` and `/send-decline` (Step A). Field reference:

| Field | Flag | Notes |
|---|---|---|
| `comments[][path]` | `-f` | File path relative to repo root |
| `comments[][line]` | `-F` | End line number (integer; `-F` for type coercion) |
| `comments[][start_line]` | `-F` | Start line for multi-line ranges (omit for single-line) |
| `comments[][side]` | `-f` | `RIGHT` for added/context lines; `LEFT` for deleted lines |
| `comments[][body]` | `-f` | Comment text; see suggestion block format below |

Use `-f` for string fields, `-F` for integer fields (`line`, `start_line`).

**Single-line comment:**
```bash
-f 'comments[][path]=lib/ddclient.pm' \
-F 'comments[][line]=42' \
-f 'comments[][side]=RIGHT' \
-f 'comments[][body]=Comment text'
```

**Multi-line comment (spans lines 40–42):**
```bash
-f 'comments[][path]=lib/ddclient.pm' \
-F 'comments[][start_line]=40' \
-F 'comments[][line]=42' \
-f 'comments[][side]=RIGHT' \
-f 'comments[][body]=Comment text'
```

**Code suggestion** (renders as one-click Apply button in GitHub):
```bash
-f 'comments[][body]=Prefer int(rand($n)) to avoid silent float truncation passed to sleep:

```suggestion
my $jitter = int(rand($interval * $pct));
```'
```

---

## References

Sources consulted when building and refining this checklist:

- **SpillwaveSolutions/pr-reviewer-skill** — original skill this was adapted from; gh CLI replacements for Python scripts applied during import
  https://github.com/SpillwaveSolutions/pr-reviewer-skill
- **ParadiseSS13/Paradise discussion #21968** — community PR review checklist covering code cleanliness, readability, runtime prevention, and testing practices; BYOND-specific items excluded
  https://github.com/ParadiseSS13/Paradise/discussions/21968
- **r/rails — "How do you approach PR reviews?"** (u/arup_r, 2026) — practitioner discussion; contributed: test diff before implementation diff, tautological assertion detection, failure mode analysis, impact radius for shared code, unnecessary abstraction as tech debt, right-problem check
- **bhserna review skill gist** — Claude Code skill for branch/PR review; contributed: check project conventions file (CLAUDE.md/CONTRIBUTING.md) before reviewing, concurrency as an explicit correctness concern
  https://gist.github.com/bhserna/831bc50ad38378813eee9d9407609cf7
- **ddclient/ddclient PR #888 review** — real-world review where the testing depth checks were first identified: Perl namespace mismatch (`CORE::sleep` vs `ddclient::sleep`), regression sensitivity, and configurable-default-as-invariant in test assertions
- **aidankinzett/claude-git-pr-skill** — contributed: pending review API pattern (bundle inline comments into PENDING review before submitting), code suggestion ` ```suggestion ` block syntax, `-f` vs `-F` flag distinction for string/integer fields, multi-line inline comment ranges via `start_line`, `LEFT`/`RIGHT` side parameter
  https://github.com/aidankinzett/claude-git-pr-skill
