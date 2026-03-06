# Agent Skills

Reusable skills for AI coding agents — [Cursor](https://cursor.com), [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview), [Codex](https://openai.com/index/codex/), and [37+ others](https://skills.sh/docs).

## Skills

| Skill | What it does | Install |
| --- | --- | --- |
| [caddy-docker-proxy-local](skills/caddy-docker-proxy-local/SKILL.md) | Automatic `*.localhost` routing for Docker dev environments | `npx skills add wjessup/agent-skills@caddy-docker-proxy-local` |

## Usage

```bash
npx skills add wjessup/agent-skills@<skill-name>
```

Or install globally so it's available across all projects:

```bash
npx skills add wjessup/agent-skills@<skill-name> -g
```

## Contributing

Each skill lives in `skills/<name>/SKILL.md`. PRs welcome.
