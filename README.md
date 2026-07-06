# skills

My personal collection of agent skills, following the [Agent Skills](https://agentskills.io) specification. Compatible with Claude Code, Codex, Cursor, OpenCode, and any other agent supported by the [skills CLI](https://github.com/vercel-labs/skills).

## Install

```bash
npx skills add lucasrcosta/skills
```

The CLI shows an interactive picker of available skills and asks which agents to install them for. To target specific agents directly:

```bash
npx skills add lucasrcosta/skills -a claude-code
```

To install a single skill:

```bash
npx skills add https://github.com/lucasrcosta/skills/tree/main/skills/rewrite-history
```

To update installed skills later:

```bash
npx skills update
```

## Skills

| Skill | Description |
| --- | --- |
| [rewrite-history](skills/rewrite-history/SKILL.md) | Rewrite commit history on the current branch into a coherent didactic story with step-by-step commits |

## Adding a new skill

Create `skills/<name>/SKILL.md` with `name` and `description` frontmatter. Keep frontmatter to the shared spec fields so skills stay portable across agents.
