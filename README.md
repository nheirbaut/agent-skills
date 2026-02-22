# Agent Skills

My preferred [OpenCode](https://opencode.ai) skills — curated for .NET development and professional software engineering.

## Skills

| Skill | Loading | Description |
|-------|---------|-------------|
| [logging-best-practices](logging-best-practices/) | On demand | Structured logging for .NET using `UseSerilogRequestLogging`/`IDiagnosticContext`, wide events, `ILogger<T>`, Serilog, and OpenTelemetry correlation |
| [solid](solid/) | On demand | SOLID principles, TDD, clean code, design patterns, and professional software craftsmanship |
| [agent-conduct](agent-conduct/) | Always on | Agent behavioral rules — questions get answers (not actions), scope discipline, and uncertainty handling |

## Setup

### Option 1 — Global install (all projects)
Copy the skill directories into the OpenCode global skills folder:
```bash
# Clone this repo
git clone https://github.com/nheirbaut/agent-skills.git
cp -r agent-skills/logging-best-practices ~/.config/opencode/skills/
cp -r agent-skills/solid ~/.config/opencode/skills/
cp -r agent-skills/agent-conduct ~/.config/opencode/skills/
```

Then add `agent-conduct` to the `instructions` field in your global config (`~/.config/opencode/opencode.json`) so it is always loaded:

```json
{
  "instructions": [
    "~/.config/opencode/skills/agent-conduct/SKILL.md"
  ]
}
```

### Option 2 — Project-level install
Place skill directories inside your project's `.opencode/skills/` folder:
```bash
mkdir -p .opencode/skills
cp -r /path/to/agent-skills/logging-best-practices .opencode/skills/
cp -r /path/to/agent-skills/solid .opencode/skills/
cp -r /path/to/agent-skills/agent-conduct .opencode/skills/
```

Then add `agent-conduct` to the `instructions` field in your project's `opencode.json`:

```json
{
  "instructions": [
    ".opencode/skills/agent-conduct/SKILL.md"
  ]
}
```

### Option 3 — Reference via `opencode.json`
Point OpenCode to this repo's location on disk by adding a `skills.paths` entry to your `opencode.json` (global or project-level):
```jsonc
{
  "skills": {
    "paths": ["/path/to/agent-skills/logging-best-practices", "/path/to/agent-skills/solid", "/path/to/agent-skills/agent-conduct"]
  },
  "instructions": [
    "/path/to/agent-skills/agent-conduct/SKILL.md"
  ]
}
```

## How skills work
OpenCode discovers skills automatically from several locations. Each skill is a directory containing a `SKILL.md` file with YAML frontmatter (`name`, `description`) and markdown instructions.

There are two loading modes:

- **On demand** — OpenCode lists available skills and the agent activates them when the task matches the skill's description. Domain-specific skills like `logging-best-practices` and `solid` work well this way.
- **Always on** — Behavioral skills like `agent-conduct` have no specific domain to match against and would rarely (if ever) activate on demand. These must be added to the [`instructions`](https://opencode.ai/docs/config/#instructions) field in `opencode.json`, which injects their content into every session regardless of task.

## License

- **logging-best-practices** — MIT. Adapted from [boristane/agent-skills](https://github.com/boristane/agent-skills); original concepts by Boris Tane.
- **solid** — By [ramziddin](https://github.com/ramziddin/solid-skills).
- **agent-conduct** — MIT. By [nheirbaut](https://github.com/nheirbaut).