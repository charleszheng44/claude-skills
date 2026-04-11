---
name: reviewing-claude-task-prs
description: Use when the user asks to review open PRs — one-shot review on every PR plus a bounded review→address loop on 'claude-task' PRs, with fresh worktree-isolated subagents per iteration
---

# Reviewing `claude-task` PRs

## Overview

Review every open PR. `claude-task` PRs enter a review→address loop run as **waves**, each dispatching **fresh** subagents. Default `MAX_ITER=3`; parse override from request ("max 5 tries" → 5). Every subagent uses `general-purpose` + `isolation: "worktree"` (**mandatory**); all dispatches in a phase go in **one message**.

## Step 1 — Fetch & partition

```bash
gh pr list --state open \
  --json number,title,headRefName,labels,body,url,isCrossRepository
```

Retry ~1s up to 3× on `error connecting to api.github.com`. Empty → stop. Partition: `isCrossRepository:true` → **skipped** (forks); no `claude-task` → Step 2; else Step 3. First run: `gh label create need-human-attention --color d93f0b 2>/dev/null || true`.

## Step 2 — One-shot reviews (non-`claude-task`)

One `Agent` per PR. Checks out head; reads `gh pr diff <N>`; pulls any `Closes #<N>` issue for context; posts **exactly one** `gh pr review <N> --approve|--request-changes|--comment -b "<body>"`; returns verdict.

## Step 3 — Wave loop (`claude-task` only)

Parent tracks `pending` / `approved` / `stuck` / `iter=1`.

### Phase A — Review wave

Fresh `Agent` per pending PR. Checks out head; `gh pr diff <N>`; resolves `Closes #<N>` / `Fixes #<N>` and reads the issue's `## Scope` / `## Out of scope` / `## Acceptance criteria` (none → diff + repo conventions, note it). Judges scope, out-of-scope violations, acceptance criteria, tests, unrelated refactors. Posts **exactly one** of `gh pr review <N> --approve -b "<summary>"` or `--request-changes -b "<items pinned to file:line>"`. Returns verdict.

`approved` exits `pending`; `changes_requested` stays. Empty `pending` → exit loop.

### Phase B — Address wave

Dispatched **after** Phase A returns. Fresh `Agent` per remaining PR. Checks out head; pulls **only** the latest `CHANGES_REQUESTED` review body + line comments (`gh pr view <N> --json reviews`, `gh api repos/<o>/<r>/pulls/<N>/comments`); fixes them **respecting `## Out of scope`**; build/tests **must be green**; commits with trailer `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>`; `git push origin HEAD` — **never `--force`**. Non-fast-forward → one `git pull --rebase` retry; still conflicted → PR → `stuck`.

**Hard constraint:** needs human input OR failing tests → STOP, don't commit, PR → `stuck`.

`iter++`. `iter > MAX_ITER` → remaining → `stuck`; else loop to Phase A.

## Step 4 — Stuck PRs

`gh pr edit <N> --add-label need-human-attention`. The request-changes review is already on the PR.

## Step 5 — Report

`| PR | Title | Label | Iterations | Verdict | Link |` plus `skipped (fork)` rows; include persisted worktree paths — **do not auto-remove**.

## Common Mistakes

- Reviewer + address-agent dispatched together — Phase A must finish first.
- Missing `isolation: "worktree"` — parallel PRs trample.
- Reusing an agent across iterations — violates the fresh-agent rule.
- `git push --force` — overwrites human edits.
- Phase B touching `## Out of scope` — drift.
- Reading older reviews — only the latest `CHANGES_REQUESTED` counts.
