---
name: github-pr-review
description: >-
  Use when reviewing GitHub pull requests with the gh CLI: pending reviews, batched inline comments,
  code suggestions, and event types COMMENT / APPROVE / REQUEST_CHANGES. Trigger on GitHub PR review,
  gh pr review, or pull request feedback.
allowed-tools: AskUserQuestion, Bash
---

# GitHub PR review

## Step 1 — Check `gh` is installed

Run:

```bash
gh --version
```

**If the command fails (not installed):** use **AskUserQuestion** immediately — do not proceed without confirmation:

- Question: "`gh` CLI is not installed. It is required to post the review — there is no fallback. What would you like to do?"
- Options: "Install and authenticate `gh`" / "Abort"

If **Abort** → stop immediately. Do not proceed.

If **Install and authenticate `gh`** → show the instructions below, then stop and wait. Do not proceed until the user explicitly confirms `gh --version` succeeds and they have run `gh auth login`.

> **Install `gh`:**
> - macOS: `brew install gh`
> - Windows: `winget install GitHub.cli`
> - Linux: https://cli.github.com/
>
> After install, authenticate: `gh auth login`

## Step 2 — Fetch PR details and diff

```bash
# Repo slug (needed for API paths)
gh repo view --json nameWithOwner -q .nameWithOwner

# PR metadata
gh pr view <PR_NUMBER> --json title,body,author,baseRefName,headRefName,state,additions,deletions,changedFiles

# Latest commit SHA (required for the review API)
gh pr view <PR_NUMBER> --json commits --jq '.commits[-1].oid'

# Full diff
gh pr diff <PR_NUMBER>
```

## Step 3 — Analyze and draft the review

Read the diff and produce a review using the **standard format** below. Do not post anything yet.

## Step 4 — Show review and ask for approval

Present the complete review to the user, then use **AskUserQuestion**:

- Question: "Post this review to the PR?"
- Options: "Yes, post it" / "No, cancel" / "Edit first"

**Do not call any API until the user selects "Yes, post it".**

## Step 5 — Post the review (only after approval)

Always use the pending review pattern — batch all comments into one review, even for a single comment.

```bash
# 5a: Create a PENDING review with all inline comments
gh api repos/OWNER/REPO/pulls/<PR_NUMBER>/reviews \
  -X POST \
  -f commit_id="<COMMIT_SHA>" \
  -f 'comments[][path]=path/to/file.ts' \
  -F 'comments[][line]=<LINE_NUMBER>' \
  -f 'comments[][side]=RIGHT' \
  -f 'comments[][body]=Inline comment text' \
  --jq '{id, state}'

# Returns: {"id": <REVIEW_ID>, "state": "PENDING"}

# 5b: Submit the pending review
gh api repos/OWNER/REPO/pulls/<PR_NUMBER>/reviews/<REVIEW_ID>/events \
  -X POST \
  -f event="<APPROVE|REQUEST_CHANGES|COMMENT>" \
  -f body="<OVERALL_MESSAGE>"
```

### Parameters

| Parameter | Notes |
|-----------|-------|
| `commit_id` | Latest head SHA from Step 2 |
| `comments[][path]` | File path relative to repo root |
| `comments[][line]` | End line number — use `-F` (integer flag) |
| `comments[][side]` | `RIGHT` for additions, `LEFT` for deletions |
| `comments[][body]` | Markdown; may include a fenced `suggestion` block |

### Syntax rules

- Single-quote keys that contain `[]`: `'comments[][path]=...'`
- Strings: `-f`; integers: `-F`
- Always fetch `commit_id` — never hardcode or skip it

### Code suggestions

````bash
-f 'comments[][body]=Your comment explaining why.

```suggestion
const fixed = "corrected value";
```

Additional context if needed.'
````

Suggestions replace the targeted line(s). Keep the suggested code complete and correct.

---

## Standard review format

Use this structure for **every** review — both what you show the user in Step 4 and what maps to API fields in Step 5.

```
## Summary
<1–3 sentences: what the PR does and why>

## Verdict
APPROVE | REQUEST_CHANGES | COMMENT — <one-line reason>

## Inline comments
- `path/to/file.ts:42` — <comment body>
- `path/to/file.ts:80` — <comment body>
  Suggestion:
  ```
  corrected code here
  ```

## Overall message
<Top-level review body posted to GitHub — plain markdown, no headings>
```

### Verdict → event type

| Verdict | `event` value |
|---------|---------------|
| APPROVE | `APPROVE` |
| REQUEST_CHANGES | `REQUEST_CHANGES` |
| COMMENT | `COMMENT` |

---

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Posting without creating a pending review first | Always use step 5a before 5b |
| Posting after verbal LGTM without AskUserQuestion | Always ask before any POST |
| Wrong quotes on `comments[]` keys | Use single quotes: `'comments[][path]=...'` |
| Missing `commit_id` | Fetch with `gh pr view <N> --json commits --jq '.commits[-1].oid'` |

## Red flags — stop

- Skipping the `gh --version` check
- Posting without showing the full review to the user first
- Assuming `gh` is authenticated
- Submitting inline comments outside a pending review
