---
name: commit
description: >-
  Orchestrates the full commit workflow: runs code review on staged/unstaged changes, iterates on fixes, suggests a conventional commit message, optionally updates READMEs, then commits on approval. Use when the user wants to commit, asks for a commit message, wants to review changes before committing, or says "what should I commit?".
---

# Commit Flow

This skill orchestrates a full commit workflow: code review → iterative fixes → commit message → README update → commit.

## Step 1 — Gather the diff

```bash
git diff HEAD
```

If the output is empty, also try:

```bash
git diff --cached
```

If both are empty, tell the user there are no changes to commit and stop.

## Step 2 — Run code review

Invoke the `code-review` skill on the current diff. This will evaluate the changes against the standards defined in `code-standards.json` and surface any issues.

## Step 3 — Iterative fix loop

After the review, present the user with a numbered list of issues found (grouped by severity: errors first, then warnings, then suggestions).

Ask: **"Which of these would you like to fix before committing? List the numbers, or say 'none' to skip."**

For each selected issue, work through them one at a time:
1. Explain what needs to change and why (referencing the relevant standard)
2. Make the fix
3. Confirm with the user before moving to the next item

Repeat until the user is satisfied or has skipped all remaining items.

## Step 4 — Suggest a commit message

Re-read the final diff (after any fixes) and produce a short summary (2–4 sentences):
- What files changed and why
- What was added, removed, or modified
- Any notable side-effects or deletions

Then suggest a commit message using **Conventional Commits** format:

```
<type>(<optional scope>): <short imperative description>

<optional body: what changed and why, wrap at 72 chars>
```

**Type rules:**
- `feat` — new feature or behaviour
- `fix` — bug fix
- `refactor` — restructuring without behaviour change
- `test` — adding or updating tests
- `chore` — tooling, config, deps, scripts
- `docs` — documentation only
- `style` — formatting, whitespace (no logic change)
- `perf` — performance improvement
- `ci` — CI/CD pipeline changes

**Message rules:**
- Subject line ≤ 72 characters, imperative mood ("add X", not "added X")
- No period at end of subject line
- Body only if changes need explanation beyond the subject
- If multiple unrelated concerns exist, flag them and suggest splitting commits

Present output in this order:
1. **Summary** — 2–4 sentence prose description of what changed
2. **Suggested commit message** — inside a fenced code block
3. **Alternative** — one shorter or differently-scoped variant if relevant

Ask the user: **"Does this commit message look good, or would you like to adjust it?"**

- If the user requests changes: apply them, show the revised message, and ask again.
- Once approved, continue to Step 5.

## Step 5 — README update

**REQUIRED — do not skip.** After message approval, always ask:

**"Would you like to update README.md files to reflect these changes?"**

- If yes: invoke the `update-readme` skill, then continue to Step 6.
- If no: continue to Step 6.

Do not proceed to Step 6 until the user has answered this question.

## Step 6 — Commit

Run `git status` to collect all files to stage: modified tracked files, deleted files, **and new untracked files** that are part of the change. Stage them explicitly by name (do not use `git add .` or `git add -A`):

```bash
git add <file1> <file2> ...   # list every changed + new file individually
git commit -m "<approved message here>"
```

Do not ask for confirmation again — the message was already approved in Step 4.
