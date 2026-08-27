# wayai-skill

The official WayAI skill for filesystem-having AI agents. Drives the canonical onboarding flow at [wayai.pro](https://wayai.pro/).

---

## Copy one prompt into your AI

Both lines, into Claude Code, Codex, Cursor, OpenCode, Cline, GitHub Copilot, or any of 48 more. The first line says what you want; the second installs this skill, which handles the rest — CLI install, login, organization, credentials, OAuth, and your first hub.

```
Set up a WayAI agent for customer support over WhatsApp.
Install the skill first: mkdir -p .claude && npx skills add wayai-pro/wayai-skill -y
```

`mkdir -p .claude` lets Claude Code pick up the skill (harmless on other agents); `-y` skips prompts so it runs unattended. If your agent is already running, ask it to run the install command — it will pick up the skill on its next turn.

No account yet? The free plan includes 3,000 operations per month, no credit card required — the skill walks you through signup, or [create one first](https://app.wayai.pro/login).
