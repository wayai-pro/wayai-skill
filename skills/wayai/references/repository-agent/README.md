# Template: an agent that works on a repository

A hosted agent (on a **Claude Agent (Managed)** connection) that works on one GitHub repository: it reads the repository's `AGENTS.md` first, works on a branch of its own, commits, and pushes that branch, never the default branch. Use it for long autonomous work on code or documents kept in a repository.

Unlike the [canonical example](../canonical-example/README.md), these files are meant to be copied and edited.

## Files

```
repository-agent/
└── agents/
    ├── repository-agent.yaml   # the agent: its connection, repository, branch, sandbox network
    └── repository-agent.md     # its instructions: read AGENTS.md first, the branch-only rule
```

## Before you use it

1. **A Claude Agent (Managed) connection** on the hub, with its webhook and your organization's GitHub App set: [connections.md → Claude Agent (Managed)](../connections.md#claude-agent-managed). Install the App on the repository.
2. **Protect the repository's default branch** (branch protection, or a ruleset that requires a pull request). Each session's token can push to any branch of the repository, so the protection is what keeps the agent's changes behind review. The instructions tell the agent never to push the default branch; the protection makes sure.

## Use it

1. Copy `agents/repository-agent.yaml` and `agents/repository-agent.md` into your hub folder's `agents/` (`wayai-ws/hubs/<hub>/agents/`).
2. Edit the YAML:
   - `connection`: the name of your Claude Agent (Managed) connection in `hub.yaml`.
   - `repo_url`: the repository, as `https://github.com/<owner>/<repo>`.
   - `repo_checkout`: the branch, or the full commit hash, each session starts from. Remove it to start from the repository's default branch. A value that is neither a branch name nor a full commit hash is refused when you push.
   - `network_egress_allowlist`: the hosts the repository's build and tests need (package registries, for example). The attached repository stays reachable without being listed. Keep the list short: the sandbox can send anything to a listed host, `git push` included.
   - `mcp_servers`: to give the agent a toolset's tools, the connection id of an [MCP Server](../connections.md#mcp-server) connection pointed at the toolset's MCP URL ([toolsets.md](../bases/toolsets.md#the-mcp-connection-url)). Leave it empty for none.
3. Edit the instructions in `repository-agent.md` for your repository and the work you hand the agent. Keep the first section: a repository's `AGENTS.md` is not loaded for the agent, so it reads it because the instructions say so.
4. `wayai push`.

## What it can do now, and what comes later

- **Now:** clone the repository at its branch or commit, read it, change files, run commands in its sandbox, commit, and push a branch of its own. It ends its reply with the branch name, what changed and how it was tested.
- **Later:** opening the pull request itself. Until then, a person opens it from the branch the agent names.

Each session gets its own sandbox and its own copy of the repository. A change to `repo_url` or `repo_checkout` starts the conversation's next turn in a new session.
