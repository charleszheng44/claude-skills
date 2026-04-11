# claude-skills

Personal Claude Code skills for an issue-driven, autonomous-agent development workflow.

## Skills

| Skill | Purpose |
|---|---|
| [`authoring-claude-task-issues`](skills/authoring-claude-task-issues/SKILL.md) | Create GitHub issues structured for autonomous resolution, with a rubric for the `claude-task` label and an `## Open Questions` convention for gating on human input. |
| [`resolving-claude-task-issues`](skills/resolving-claude-task-issues/SKILL.md) | Fetch open `claude-task` issues on the current repo, gate each on its `## Open Questions` section, and dispatch one isolated subagent per ready issue to produce a PR. |

The two skills share one contract:

1. The `claude-task` label — "this issue can be implemented by an autonomous agent."
2. The `## Open Questions` GitHub-checkbox convention — any `- [ ]` remaining means the issue is blocked on human input.

Issues authored with the first skill can be resolved with the second. Either skill works on its own against issues that already follow the convention.

## Installation

### Option 1 — symlink into `~/.claude/skills/` (personal skills)

```bash
mkdir -p ~/.claude/skills
ln -s "$PWD/skills/authoring-claude-task-issues"  ~/.claude/skills/authoring-claude-task-issues
ln -s "$PWD/skills/resolving-claude-task-issues"  ~/.claude/skills/resolving-claude-task-issues
```

Claude Code picks them up automatically — restart the session to load them into the skill registry.

### Option 2 — clone and symlink anywhere

```bash
git clone <this-repo> ~/Work/claude-skills
# then the symlink commands from Option 1
```

## Verifying the skills loaded

After restarting Claude Code, the skills should appear in the available-skills list. Trigger phrases:

- **authoring:** "create an issue for X", "file a bug about Y", "open issues for these: 1) ..., 2) ..."
- **resolving:** "work on open claude-task issues", "resolve any claude-task issue", "pick up the claude-task issues"

## Editing the skills

The skills are plain markdown with YAML frontmatter. Edit `SKILL.md` in place; because `~/.claude/skills/*` are symlinks, changes take effect on the next session start without reinstallation.

## Conventions enforced by the skills

- `gh` CLI hits `api.github.com`. Both skills retry on `error connecting to api.github.com` up to 3 times rather than falling back to browser workflows — this is a documented transient flap in environments using certain VPN DNS resolvers.
- `Agent` subagent dispatch always uses `isolation: "worktree"` and is batched into a single message for parallelism.
- PRs opened by dispatched agents carry the `claude-task` label plus any labels inherited from the source issue, and include `Closes #<N>` in the body for GitHub's auto-close wiring.
