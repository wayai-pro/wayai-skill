# wayai-skill

The official WayAI skill for filesystem-having AI agents — Claude Code, Cursor, Codex CLI, OpenCode, Claude Cowork, and any agent that loads skills from disk. Drives the canonical onboarding flow described at [wayai.pro/docs/get-started](https://wayai.pro/docs/get-started).

## Install

**Claude Code** (two commands):

```
/plugin marketplace add wayai-pro/wayai-skill
/plugin install wayai@wayai-skill
```

**Cursor / cross-tool** (via [skills.sh](https://skills.sh)):

```
npx skills add wayai
```

**Manual** (any AGENTS.md-aware agent):

```
git clone https://github.com/wayai-pro/wayai-skill ~/.claude/skills/wayai
```

## Then ask your agent

> Set up a WayAI hub for customer support over WhatsApp.

The skill walks you through CLI install, login, organization, credentials, OAuth, and your first hub.

## Source of truth

This repository is **mirrored** from the WayAI platform monorepo on every change to the canonical skill path. Do not send PRs here — they will be overwritten by the next sync. Open issues at [github.com/wayai-pro/wayai_platform](https://github.com/wayai-pro/wayai_platform) instead.
