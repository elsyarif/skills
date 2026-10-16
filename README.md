# Agent Skills

Custom agent skills for Claude Code. Each folder = one skill.

## Install

Skills follow the open [Agent Skills](https://agentskills.io) format — the same folder works across agents.

Recommended (any agent — [skills.sh](https://www.skills.sh) CLI):

```bash
npx skills add elsyarif/skills
```

Manual (clone, then copy a skill folder into your agent's skills directory):

```bash
git clone https://github.com/elsyarif/skills.git
cp -r skills/architecture-ddd ~/.claude/skills/   # Claude Code
```

| Agent | Skills directory | Docs |
|---|---|---|
| Claude Code | `~/.claude/skills/` | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) |
| Cursor | `~/.cursor/skills/` | [cursor.com/docs/context/skills](https://cursor.com/docs/context/skills) |
| Codex (ChatGPT) | `~/.codex/skills/` | [developers.openai.com/codex/skills](https://developers.openai.com/codex/skills/) |
| Gemini CLI | `~/.gemini/skills/` | [geminicli.com/docs/cli/skills](https://geminicli.com/docs/cli/skills/) |
| Goose | `~/.config/goose/skills/` | [goose docs](https://block.github.io/goose/docs/guides/context-engineering/using-skills/) |
| OpenCode | `~/.config/opencode/skills/` | [opencode.ai/docs/skills](https://opencode.ai/docs/skills/) |

Full client list: [agentskills.io](https://agentskills.io). After installing, reload/restart the agent.

## Usage

Skill auto-loads when its description matches your request. Trigger examples:

- "scaffold Go DDD project untuk feature order"
- "tambah external system payment ke infrastructure"
- "buat aggregate untuk Order dengan invariants"

Or invoke explicitly: `/architecture-ddd`

## Verify

`/reload-skills` (Claude Code) or restart the agent → check skill appears in the list → test a trigger phrase in a fresh session.

## Skills

| Skill                                 | Purpose                                                                                                                                                                                                                                                                                                                                                  | Structure |
|---------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---|
| [architecture-ddd](architecture-ddd/) | Generic DDD & Clean Architecture contract (Eric Evans tactical patterns: Entity, Value Object, Aggregate, Domain Event, Repository, Feature Flags, Error Taxonomy, Testing Standards). Use when adding external systems, DB repositories, message brokers (RMQ/Kafka), usecases, domain services, aggregates, feature flags, or Cache Decorator Pattern. | `SKILL.md` + `references/go-ddd-structure.md` + `templates/domain-tree.txt` |

## Skill Structure Convention

```
<skill-name>/
  SKILL.md              # Required. YAML frontmatter (name, description) + core contract
  references/           # Optional. Heavy reference loaded on-demand (100+ lines)
  templates/            # Optional. Copy-paste scaffolds
```

Rules:
- `name`/`description` in frontmatter = trigger. Never rename without retesting.
- Reference supporting files from SKILL.md with plain relative paths (`` `references/x.md` ``), never `@` (force-loads context).
- SKILL.md stays lean; detail lives in `references/`, loaded only when needed (progressive disclosure).
- After adding a new skill folder: add a row to the table above, run `/reload-skills`, verify trigger in a fresh session.

## License

MIT
