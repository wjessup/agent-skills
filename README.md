# caddy-docker-proxy-local

An agent skill that sets up automatic `*.localhost` routing for Docker dev environments using [caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy).

Works with [Cursor](https://cursor.com), [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview), [Codex](https://openai.com/index/codex/), and [37+ other agents](https://skills.sh/docs).

## The Problem

When AI agents control your dev environment, you lose track of what's running where. An agent starts `npm run dev` on port 3000 in one terminal. You open a second project — another agent grabs port 3000, fails, falls back to 3001. A third project takes 3002. Now you have three apps on random ports across terminals you didn't open, and nobody knows which URL belongs to which project.

This gets worse with fullstack apps. Frontend, API, database — that's 3+ ports per project. Four projects running? You're juggling 12 port assignments that change every time something restarts.

## The Fix

Move everything into Docker and put a shared reverse proxy in front. Each project gets a stable, deterministic URL like `http://my-project.localhost`. No port conflicts. No guessing. Multiple projects run simultaneously because the proxy routes by hostname, not port number.

Each project opts in with ~6 lines added to `docker-compose.yml`. One shared Caddy proxy runs on port 80 — that's the only port on your machine anything binds to.

## Install

```bash
npx skills add wjessup/agent-skills@caddy-docker-proxy-local
```

See [SKILL.md](skills/caddy-docker-proxy-local/SKILL.md) for the full setup guide.
