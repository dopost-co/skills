# dopost-co/skills

The dopost API skill for coding assistants and autonomous agents, including Cursor, Claude Code, Codex, GitHub Copilot, and OpenClaw.

## Install

```sh
npx skills add dopost-co/skills
```

## Available Skills

| Skill | Description |
|-------|-------------|
| [dopost-api](./dopost-api/SKILL.md) | Publish and schedule posts, manage accounts and media, and retrieve account, post, content, and audience analytics via the dopost REST API |

## Usage with specific agents

All commands below install the same `dopost-api` skill. The `-a` option selects which agent receives it; it does not change the skill's instructions.

| Agent | Install command |
|-------|-----------------|
| Cursor | `npx skills add dopost-co/skills -a cursor` |
| Claude Code | `npx skills add dopost-co/skills -a claude-code` |
| Codex | `npx skills add dopost-co/skills -a codex` |
| GitHub Copilot | `npx skills add dopost-co/skills -a github-copilot` |
| OpenClaw | `npx skills add dopost-co/skills -a openclaw` |
| Gemini CLI | `npx skills add dopost-co/skills -a gemini-cli` |
| Kimi Code CLI | `npx skills add dopost-co/skills -a kimi-code-cli` |
| Grok Build | `npx skills add dopost-co/skills -a grok` |
| Hermes Agent | `npx skills add dopost-co/skills -a hermes-agent` |
| OpenCode | `npx skills add dopost-co/skills -a opencode` |
| Cline | `npx skills add dopost-co/skills -a cline` |
| Windsurf | `npx skills add dopost-co/skills -a windsurf` |

Install for multiple agents in one command:

```sh
npx skills add dopost-co/skills -a cursor codex claude-code
```

By default, installation is scoped to the current project. Add `-g` to install for your user across projects. For example, install into OpenClaw's user skill directory:

```sh
npx skills add dopost-co/skills -g -a openclaw
```

For other clients, check the installer's [supported agents and identifiers](https://github.com/vercel-labs/skills#supported-agents). Agents that connect through MCP may use a different setup: see the [dopost agent guides](https://dopost.co/docs/api/ai-agents).

## Configure access

Create a workspace API key in [Settings → API keys](https://dopost.co/dashboard/settings/api-keys) and configure `DOPOST_API_KEY` in the environment where your agent runs. API access requires a Pro or Agency workspace. Select the permissions you need, including `analytics:read` for statistics.
