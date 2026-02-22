# Agent Conduct

An OpenCode skill for predictable, trustworthy agent behavior.

## What it covers

- **Questions vs. Instructions** — when to evaluate and report vs. when to take action
- **Edge case classification** — handling ambiguous phrasing like "Make sure X is correct" or "Can you update X?"
- **Transparency on uncertainty** — state interpretation, ask for clarification, default to less action
- **Scope discipline** — do only what was asked, report unrelated findings separately, propose before expanding
- **System safety** — never install, update, or remove software without explicit permission
- **Work items** — describe the problem and desired outcome, not the implementation
- **Version control safety** — never commit or push without explicit permission; editing ≠ committing

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Skill definition and behavioral rules |

## Installation
This skill is designed for **global installation** so it applies across all projects.

### Step 1 — Copy the skill
```bash
cp -r agent-conduct ~/.config/opencode/skills/
```

### Step 2 — Enable always-on loading

Unlike domain-specific skills (which activate on demand), behavioral skills must be **explicitly included** in your `opencode.json` via the `instructions` field. Without this step, the skill relies on on-demand matching and may never activate — it has no specific domain to match against.

Add this to your global config (`~/.config/opencode/opencode.json`):

```json
{
  "instructions": [
    "~/.config/opencode/skills/agent-conduct/SKILL.md"
  ]
}
```

This injects the skill's rules into every session, regardless of task domain.
See the [repo README](../README.md) for all installation options.