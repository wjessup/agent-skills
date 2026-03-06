# caddy-docker-proxy-local

An agent skill that sets up automatic `*.localhost` routing for Docker dev environments using [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy).

Works with [Cursor](https://cursor.com), [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview), [Codex](https://openai.com/index/codex/), and [37+ other agents](https://skills.sh/docs).

## Install

```bash
npx skills add wjessup/agent-skills@caddy-docker-proxy-local
```

## What it does

Every Docker Compose project gets a clean `http://<project-name>.localhost` URL. One shared Caddy proxy runs on port 80 and routes by hostname — multiple projects run simultaneously with zero port conflicts. Each project opts in with ~6 lines added to `docker-compose.yml`.

See [SKILL.md](skills/caddy-docker-proxy-local/SKILL.md) for the full setup guide.
