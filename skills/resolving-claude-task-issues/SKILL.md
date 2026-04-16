---
name: resolving-claude-task-issues
description: Use when the user asks to work on, resolve, pick up, or fix open GitHub issues labeled 'claude-task' — gates each on its Open Questions and dispatches one worktree-isolated subagent per ready issue to produce a PR
---

# Resolving `claude-task` Issues

## Overview

Fetch open issues labeled `claude-task`, check each body for unanswered `## Open Questions`, and dispatch one isolated subagent per **ready** issue to open a PR. Report blocked issues with their unchecked questions verbatim.

**Core principle:** never implement a blocked issue. A single `- [ ]` in `## Open Questions` = human has not committed.

## Step 1 — Fetch

```bash
git fetch origin
gh issue list --state open --label claude-task \
  --json number,title,body,labels,url,comments
```

Empty → tell the user "no open claude-task issues" and stop. On `error connecting to api.github.com`, retry ~1s up to 3×.

## Step 2 — Parse `## Open Questions`

- **No heading** → ready.
- **Heading present** → walk top-level bullets until the next `## `:
  - `- [x]` → answered (bullet = question; indented sub-bullets = answer).
  - `- [ ]` → **blocked**. Any single `- [ ]` blocks the whole issue.

**Do not parse comments for readiness** — they are discussion, not decisions. Comments ARE passed to the agent as context.

## Step 3 — Dispatch (Ready Only)

Call `Agent` with `subagent_type: "general-purpose"` and `isolation: "worktree"` (**mandatory** for parallel safety). **All dispatches in one message** so they run concurrently.

### Agent prompt template

```
You are in an isolated worktree of <owner>/<repo>. Resolve issue #<N>.

## Issue #<N>: <title>
<verbatim body>

## Discussion (non-authoritative)
<comments oldest-first, or "No comments.">

## Repo context
Main: <master|main>. Match commit style from `git log --oneline -10`.

## Deliverable
1. Branch off main (e.g. `fix/issue-<N>-<slug>`).
2. Implement Scope, respect "Out of scope", build/tests green.
3. Commit ending with:
     Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
4. Push, then:
     gh pr create --base <main> --head <branch> \
       --title "<type>: <summary>" \
       --label claude-task <+ every issue label> \
       --body "<body with 'Closes #<N>'>"
   `claude-task` on the PR is MANDATORY (issue→PR traceability).
5. Retry `gh` up to 3× on `error connecting to api.github.com`.

## Constraints
- No files outside scope; no unrelated refactors; no skipped tests.
- If unexpected human input needed: STOP, don't commit, return a
  "needs clarification" summary with specific questions.

## Return (<400 words)
Files · decisions · tests · PR URL · skips.
```

## Step 4 — Report

PR table `| Issue | Branch | PR | Status |`; unchecked questions verbatim for blocked; branch + output for failures; worktree paths (do NOT auto-remove).

## Common Mistakes

- Dispatching a blocked issue — `- [ ]` is a contract.
- Sequential dispatch of independent issues — batch into one message.
- Missing `isolation: "worktree"` — agents trample each other.
- PR without `--label claude-task` — breaks traceability.
- Parsing comments for readiness — only `## Open Questions` counts.
- Silently skipping blocked issues instead of reporting them.
