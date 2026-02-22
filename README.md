# Agent Skills

My preferred [OpenCode](https://opencode.ai) skills — curated for .NET development and professional software engineering.

## Skills

| Skill | Description |
|-------|-------------|
| [logging-best-practices](logging-best-practices/) | Structured logging for .NET using `UseSerilogRequestLogging`/`IDiagnosticContext`, wide events, `ILogger<T>`, Serilog, and OpenTelemetry correlation |
| [solid](solid/) | SOLID principles, TDD, clean code, design patterns, and professional software craftsmanship |

## Setup

### Option 1 — Global install (all projects)

Copy the skill directories into the OpenCode global skills folder:

```bash
# Clone this repo
git clone https://github.com/nheirbaut/agent-skills.git

# Copy skills to OpenCode's global skills directory
cp -r agent-skills/logging-best-practices ~/.config/opencode/skills/
cp -r agent-skills/solid ~/.config/opencode/skills/
```

### Option 2 — Project-level install

Place skill directories inside your project's `.opencode/skills/` folder:

```bash
mkdir -p .opencode/skills
cp -r /path/to/agent-skills/logging-best-practices .opencode/skills/
cp -r /path/to/agent-skills/solid .opencode/skills/
```

### Option 3 — Reference via `opencode.json`

Point OpenCode to this repo's location on disk by adding a `skills.paths` entry to your `opencode.json` (global or project-level):

```jsonc
{
  "skills": {
    "paths": ["/path/to/agent-skills/logging-best-practices", "/path/to/agent-skills/solid"]
  }
}
```

## How skills work

OpenCode discovers skills automatically from several locations. Each skill is a directory containing a `SKILL.md` file with YAML frontmatter (`name`, `description`) and markdown instructions.

Skills are loaded **on demand** — the agent sees which skills are available and activates them when the task matches the skill's description. No manual activation is needed.

## License

- **logging-best-practices** — MIT. Adapted from [boristane/agent-skills](https://github.com/boristane/agent-skills); original concepts by Boris Tane.
- **solid** — By [ramziddin](https://github.com/ramziddin/solid-skills).