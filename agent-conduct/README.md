# Agent Conduct

An OpenCode skill for predictable, trustworthy agent behavior.

## What it covers

- **Questions vs. Instructions** — when to evaluate and report vs. when to take action
- **Edge case classification** — handling ambiguous phrasing like "Make sure X is correct" or "Can you update X?"
- **Transparency on uncertainty** — state interpretation, ask for clarification, default to less action
- **Scope discipline** — do only what was asked, report unrelated findings separately, propose before expanding

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Skill definition and behavioral rules |

## Installation

This skill is designed for **global installation** so it applies across all projects:

```bash
cp -r agent-conduct ~/.config/opencode/skills/
```

See the [repo README](../README.md) for all installation options.