---
name: gitlab-mr-review
description: >-
  Use when reviewing GitLab merge requests (MRs) with the glab CLI — draft notes for batched inline feedback, then publish after explicit user approval. Use for MR review, merge request review, GitLab code review, or glab mr workflows.
allowed-tools: AskUserQuestion, Bash
---

# GitLab MR review

## Step 1 — Check `glab` is installed

Run:

```bash
glab version
```

**If the command fails (not installed):** use **AskUserQuestion** immediately — do not proceed without confirmation:

- Question: "`glab` CLI is not installed. It is required to post the review — there is no fallback. What would you like to do?"
- Options: "Install and authenticate `glab`" / "Abort"

If **Abort** → stop immediately. Do not proceed.

If **Install and authenticate `glab`** → show the instructions below, then stop and wait. Do not proceed until the user explicitly confirms `glab version` succeeds and they have run `glab auth login`.

> **Install `glab`:**
> - macOS: `brew install glab`
> - Linux/Windows: https://gitlab.com/gitlab-org/cli
>
> After install, authenticate: `glab auth login`
> Use a token with at least **`api`** scope.

## Step 2 — Fetch MR details and diff

**Terminology:** GitLab uses *MR IID* — the small integer in the MR URL (`/-/merge_requests/42` → IID `42`).

```bash
# Infer namespace/project from remote (when possible)
git remote -v

# MR metadata
glab mr view <MR_IID> --json title,body,author,source_branch,target_branch,state

# Diff SHAs needed for inline comments
glab api "projects/<PROJECT_ID>/merge_requests/<MR_IID>" --jq '.diff_refs'

# Full diff
glab mr diff <MR_IID>
```

## Step 3 — Analyze and draft the review

Read the diff and produce a review using the **standard format** below. Do not post anything yet.

## Step 4 — Show review and ask for approval

Present the complete review to the user, then use **AskUserQuestion**:

- Question: "Post this review to the MR?"
- Options: "Yes, post it" / "No, cancel" / "Edit first"

**Do not call any API until the user selects "Yes, post it".**

## Step 5 — Post the review (only after approval)

Always use the draft notes pattern — create all notes in draft first, then publish in one step.

```bash
# 5a: Create a draft note (general MR comment — no position)
glab api --method POST "projects/<PROJECT_ID>/merge_requests/<MR_IID>/draft_notes" \
  -f "note=Comment body (markdown supported)"

# 5a (inline): Create an inline draft note
glab api --method POST "projects/<PROJECT_ID>/merge_requests/<MR_IID>/draft_notes" \
  --input - <<'JSON'
{
  "note": "Inline comment body",
  "position": {
    "base_sha": "<BASE_SHA>",
    "start_sha": "<START_SHA>",
    "head_sha": "<HEAD_SHA>",
    "old_path": "path/to/file.ts",
    "new_path": "path/to/file.ts",
    "position_type": "text",
    "new_line": <LINE_NUMBER>
  }
}
JSON

# 5b: Verify draft notes before publishing
glab api "projects/<PROJECT_ID>/merge_requests/<MR_IID>/draft_notes"

# 5c: Publish all draft notes at once
glab api --method POST "projects/<PROJECT_ID>/merge_requests/<MR_IID>/draft_notes/bulk_publish"

# 5d: Approve the MR (only if verdict is APPROVE and user confirmed)
glab mr approve <MR_IID>
```

### Parameters

| Parameter | Notes |
|-----------|-------|
| `PROJECT_ID` | Numeric project ID or URL-encoded path, e.g. `mygroup%2Fmyproject` |
| `MR_IID` | Small integer from the MR URL, not the global ID |
| `base_sha` / `start_sha` / `head_sha` | From `diff_refs` fetched in Step 2 |
| `new_line` | Target line number in the new file version |
| `position_type` | Always `"text"` for source code comments |

### Outcome mapping

| Verdict | Action |
|---------|--------|
| APPROVE | `bulk_publish` drafts + `glab mr approve <IID>` (with user confirmation) |
| REQUEST_CHANGES | `bulk_publish` drafts; state blocking issues in the overall message |
| COMMENT | `bulk_publish` drafts with neutral summary |

### Code suggestions

GitLab uses Markdown for comments. Format suggestions so the author can copy/paste:

````markdown
**Suggestion**

```ts
const fixed = "corrected value";
```

Replace line 42 — original has a type mismatch.
````

---

## Standard review format

Use this structure for **every** review — both what you show the user in Step 4 and what maps to API fields in Step 5.

```
## Summary
<1–3 sentences: what the MR does and why>

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
<Top-level summary posted to GitLab — plain markdown, no headings>
```

### Verdict → action

| Verdict | Step 5 action |
|---------|---------------|
| APPROVE | `bulk_publish` + `glab mr approve` |
| REQUEST_CHANGES | `bulk_publish` only |
| COMMENT | `bulk_publish` only |

---

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Posting notes before showing full review to user | Always AskUserQuestion before any POST |
| Mixing up project ID and MR IID | Project ID is numeric/encoded slug; MR IID is the small URL integer |
| Guessing `position` fields | Read `diff_refs` from the MR API first |
| Publishing individual notes instead of bulk | Always `bulk_publish` after all drafts are created |
| Missing `api` scope on token | Regenerate token and re-auth with `glab auth login` |

## Red flags — stop

- Skipping the `glab version` check
- Posting without showing the full review to the user first
- Assuming `glab` is authenticated
- Publishing draft notes before listing and verifying them (step 5b)
- Approving the MR without a separate explicit user confirmation
