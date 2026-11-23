# wayai-skill

The official WayAI skills for filesystem-having AI agents: `wayai`, which drives the canonical onboarding flow at [wayai.pro](https://wayai.pro/) and every setup after it, and `wayai-cases`, for an agent that works one case.

---

## Copy one prompt into your AI

Both lines, into Claude Code, Codex, Cursor, OpenCode, Cline, GitHub Copilot, or any of 48 more. The first line says what you want; the second installs the `wayai` skill, which handles the rest — CLI install, login, organization, credentials, OAuth, and your first hub.

```
Set up a WayAI agent for customer support over WhatsApp.
Install the skill first: mkdir -p .claude && npx skills add wayai-pro/wayai-skill --skill wayai -y
```

`mkdir -p .claude` lets Claude Code pick up the skill (harmless on other agents); `--skill wayai` installs that skill alone; `-y` skips prompts so it runs unattended. If your agent is already running, ask it to run the install command — it will pick up the skill on its next turn.

No account yet? The free plan includes 3,000 operations per month, no credit card required — the skill walks you through signup, or [create one first](https://app.wayai.pro/login).

## Working one case

An agent that works one case of a case space does it in the case's clone (`wayai cases clone`), with a case token that reaches that case alone. Install the operator skill there, in the clone's folder, instead of `wayai`:

```
mkdir -p .claude && npx skills add wayai-pro/wayai-skill --skill wayai-cases -y
```

`wayai cases clone` prints this command too.
