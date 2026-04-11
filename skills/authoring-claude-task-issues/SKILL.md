---
name: authoring-claude-task-issues
description: Use when the user asks to create, file, open, or track one or more GitHub issues for a repo — especially when some of those issues are simple enough to be implemented autonomously later by a dispatched agent
---

# Authoring `claude-task` Issues

## Overview

When the user wants GitHub issues filed, structure each one so a future autonomous agent can pick it up and implement it without further human input — OR, if human decisions are needed, capture those decisions as blocking `## Open Questions` that must be answered before dispatch.

**Core principle:** Every issue is either *ready to implement* (passes the rubric, labeled `claude-task`, no unanswered `Open Questions`) or *needs human input first* (labeled `claude-task` with an `## Open Questions` section, OR not labeled `claude-task` at all because it is too large/ambiguous for autonomous work).

## When to Use

- User says "create an issue for X"
- User lists multiple things: "open issues for these: 1) X, 2) Y, 3) Z"
- User asks to "file a bug" or "track this as an issue"
- User asks you to convert a conversation thread into tracked work

Do NOT use for in-conversation to-do tracking (use `TaskCreate` instead) or for ad-hoc GitHub comments.

## Before You Start

- Run `gh issue list --state open --limit 20` to see existing issues. Avoid duplicates.
- Run `gh label list` to check available labels. If `claude-task` does not exist, create it first:
  ```bash
  gh label create claude-task \
    --description "Issue can be resolved autonomously by a dispatched agent" \
    --color ae9f80
  ```

## The `claude-task` Rubric

Apply the `claude-task` label **only** if **all** of these are true:

- [ ] Scope is mechanical or well-defined — no design pass required
- [ ] Files to touch are identifiable from a quick look at the current codebase
- [ ] No UX or product judgment calls the user should own
- [ ] No new credentials, infrastructure, or external accounts needed
- [ ] No new architectural patterns introduced
- [ ] Fits in one small PR — roughly under 300 LOC or 6 files

If any item fails but the issue is still valid, file it **without** `claude-task` and note the reason in the body (e.g., "Not marked claude-task: needs a brainstorming pass first").

## The `## Open Questions` Convention

When there are decisions the user should make before implementation proceeds, add a `## Open Questions` section near the top of the issue body as a **GitHub checkbox list**:

```markdown
## Open Questions

- [ ] Should the left pane scroll independently or lock-step with the right?
- [ ] What happens if one provider errors mid-stream?
```

**How humans unblock the issue:** they edit the issue body in place, check each box, and write the answer on indented lines directly under the bullet:

```markdown
## Open Questions

- [x] Should the left pane scroll independently or lock-step with the right?
  - Independent. Each pane maintains its own scroll position.
- [x] What happens if one provider errors mid-stream?
  - Show an error card in the failing pane. The other keeps streaming. No auto-retry.
```

An issue is **ready to dispatch** when either:
1. There is no `## Open Questions` section at all, OR
2. Every top-level checkbox under that heading is `- [x]`

A single remaining `- [ ]` means the issue is **blocked** — the companion skill `resolving-claude-task-issues` will refuse to dispatch it.

**Adding `## Open Questions` is compatible with the `claude-task` label.** Use `claude-task` when the underlying work qualifies per the rubric; use `## Open Questions` to gate it on human decisions. Both can coexist on the same issue.

## Standard Issue Body Template

```markdown
## Problem

<One to three sentences: what is wrong or missing, and why it matters.>

## Open Questions  <!-- omit this section when there are none -->

- [ ] <question 1>
- [ ] <question 2>

## Proposed approach

<Only when the approach is obvious. Skip otherwise.>

## Scope

- [ ] <concrete action 1>
- [ ] <concrete action 2>
- [ ] `go test ./...` passes (or repo-equivalent)

## Out of scope

<Things NOT to do. Guardrails for the implementing agent.>

## Acceptance criteria

- [ ] <observable outcome 1>
- [ ] <observable outcome 2>

---
*To unblock this issue for automated implementation: edit this issue body,
check each `- [ ]` under Open Questions, and write your answer on an
indented line below. Comments are welcome for discussion but are not
parsed as decisions.*
```

The footer is **required** whenever the issue has an `## Open Questions` section — it teaches the human how to unblock it.

## Creating the Issues

Use `gh issue create`. Always pass the body via a heredoc to preserve formatting:

```bash
gh issue create \
  --title "fix: /retry cannot retry a message whose streaming failed" \
  --label bug --label claude-task \
  --body "$(cat <<'EOF'
## Problem
...
EOF
)"
```

**Labels:** always include a type label (`bug`, `enhancement`, `refactor`, `documentation`, `ci`, `test`) AND `claude-task` when the rubric passes.

**Network resilience:** `gh` CLI hits `api.github.com` over HTTPS. If it fails with `error connecting to api.github.com`, retry after ~1 second — this can be a transient DNS flap (especially through VPN resolvers). Retry up to 3 times before surfacing the error to the user. Do NOT fall back to a manual browser workflow; retrying is almost always enough.

## Quick Reference

| Step | Command |
|---|---|
| See existing open issues | `gh issue list --state open --limit 20 --json number,title,labels` |
| Check labels | `gh label list` |
| Create `claude-task` label | `gh label create claude-task --description "..." --color ae9f80` |
| Create issue | `gh issue create --title "..." --label <type> --label claude-task --body "<heredoc>"` |
| Verify | Returned URL, or `gh issue view <n>` |

## Common Mistakes

- **Applying `claude-task` on vibes.** Run the rubric checklist. If any item fails, do not apply.
- **Writing a design document in the issue body.** Issues are scope + acceptance criteria, not specs. If it needs a spec, mark that fact as an Open Question or do not apply `claude-task`.
- **Omitting the unblock-footer when there are Open Questions.** Humans do not know they need to edit the body. The footer teaches them.
- **Using `- [ ]` outside `## Open Questions`, `## Scope`, and `## Acceptance criteria`.** Mixing checkboxes elsewhere confuses the dispatcher's parser on the resolving side.
- **Skipping `gh issue list` before creating.** Duplicates are avoidable; check first.
- **Giving up on the first DNS error from `gh`.** Retry; it is usually a flap.
- **Filing three issues in one.** One issue, one concern. Split if scope spans unrelated changes.

## Red Flags — STOP and rethink

- About to apply `claude-task` without running the rubric → run the rubric
- About to file an issue with no acceptance criteria → write them first
- Adding 3+ `Open Questions` → consider whether this should be a design doc or brainstorming request instead of an issue
- About to skip `gh issue list` because "it's probably fine" → run it

## See Also

- **`resolving-claude-task-issues`** — the companion skill that fetches and dispatches agents on issues authored with this convention. The `## Open Questions` parsing rules defined here are what that skill enforces.
