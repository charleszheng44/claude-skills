---
name: authoring-claude-task-issues
description: Use when the user asks to create, file, open, or track GitHub issues — especially when some are simple enough for a later dispatched agent to implement autonomously
---

# Authoring `claude-task` Issues

## Overview

Structure each issue so a future autonomous agent can pick it up — OR, if human decisions are needed, capture them as blocking `## Open Questions` to be answered before dispatch.

**Core principle:** every issue is either *ready to implement* (passes the rubric, labeled `claude-task`, no unanswered `## Open Questions`) or *needs human input first*.

Run `gh issue list --state open --limit 20` to avoid duplicates. Create the `claude-task` label if missing (color `ae9f80`).

## The `claude-task` Rubric

Apply the label **only** if all hold:

- [ ] Scope is mechanical/well-defined — no design pass
- [ ] Files to touch are identifiable from the codebase
- [ ] No UX/product judgment the user should own
- [ ] No new credentials, infra, or architectural patterns
- [ ] Fits one small PR (~<300 LOC / 6 files)

If any item fails, file the issue **without** `claude-task`.

## The `## Open Questions` Convention

Human decisions go in an `## Open Questions` section near the top of the body as a **GitHub checkbox list**. Humans unblock by editing the body in place, checking each box and writing the answer on indented lines under it — `- [x]` plus indented prose = decision recorded.

An issue is **ready to dispatch** iff there is no `## Open Questions` section, or every `- [ ]` is `- [x]`. A single `- [ ]` blocks the issue — `resolving-claude-task-issues` refuses it. `## Open Questions` is compatible with `claude-task`; use both when the work qualifies but needs a human decision.

## Issue Body Template

```markdown
## Problem
<1–3 sentences: what's wrong, why it matters.>

## Open Questions   <!-- omit when none -->
- [ ] <question>

## Scope
- [ ] <concrete action>
- [ ] `<build/test>` passes

## Out of scope
<guardrails>

## Acceptance criteria
- [ ] <observable outcome>

---
*To unblock: edit this body, check each `- [ ]` under Open Questions,
write your answer on an indented line below. Comments ≠ decisions.*
```

The footer is **required** whenever `## Open Questions` is present.

## Creating the Issue

Pass the body via heredoc. Always include a type label (`bug`, `enhancement`, `refactor`, `docs`, `ci`, `test`) plus `claude-task` when the rubric passes:

```bash
gh issue create --title "fix: <summary>" \
  --label bug --label claude-task \
  --body "$(cat <<'EOF'
...
EOF
)"
```

On `error connecting to api.github.com`, retry ~1s up to 3×.

## Common Mistakes

- `claude-task` on vibes — run the rubric.
- Design doc in the body. Needs a spec? Make it an Open Question or skip `claude-task`.
- Missing unblock footer when `## Open Questions` exists.
- `- [ ]` in `## Problem` / `## Out of scope` — harmless to the parser, but confuses humans about what to check.
