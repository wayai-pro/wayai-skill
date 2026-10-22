# Template: an agent that works on a repository

A hosted agent (on a **Claude Agent (Managed)** connection) that works on one GitHub repository: it reads the repository's `AGENTS.md` first, works on a branch of its own, commits, pushes that branch, never the default branch, and opens its pull request. Use it for long autonomous work on code or documents kept in a repository.

Unlike the [canonical example](../canonical-example/README.md), these files are meant to be copied and edited.

## Files

```
repository-agent/
└── agents/
    ├── repository-agent.yaml   # the agent: its connection, repository, branch, sandbox network, pull request tool
    └── repository-agent.md     # its instructions: read AGENTS.md first, the branch-only rule, the pull request
```

## Before you use it

1. **A Claude Agent (Managed) connection** on the hub, with its webhook and your organization's GitHub App set: [connections.md → Claude Agent (Managed)](../connections.md#claude-agent-managed). Install the App on the repository.
2. **Protect the repository's default branch** (branch protection, or a ruleset that requires a pull request). Each session's token can push to any branch of the repository and open pull requests on it, so the protection is what keeps the agent's changes behind review. The instructions tell the agent never to push the default branch; the protection makes sure.

## Use it

1. Copy `agents/repository-agent.yaml` and `agents/repository-agent.md` into your hub folder's `agents/` (`wayai-ws/hubs/<hub>/agents/`).
2. Edit the YAML:
   - `connection`: the name of your Claude Agent (Managed) connection in `hub.yaml`.
   - `repo_url`: the repository, as `https://github.com/<owner>/<repo>`.
   - `repo_checkout`: the branch, or the full commit hash, each session starts from. Remove it to start from the repository's default branch. A value that is neither a branch name nor a full commit hash is refused when you push.
   - `network_egress_allowlist`: the hosts the repository's build and tests need (package registries, for example). The attached repository stays reachable without being listed. Keep the list short: the sandbox can send anything to a listed host, `git push` included.
   - `tools.native`: keep `open_pull_request`, the tool the agent opens its pull request with.
   - `tools.mcp`: to give the agent a toolset's tools, each tool's name and the name of the hub's [Base Toolset](../connections.md#base-toolset) connection to that toolset; the platform runs each call. The agent is a pilot, so it holds only a toolset a customer-facing agent may hold (*Customer-facing agents* there). Leave the list empty for none. An [MCP Server](../connections.md#mcp-server) connection's tools go in the same list; WayAI runs them with that connection's own credential.
3. Edit the instructions in `repository-agent.md` for your repository and the work you hand the agent. Keep the first section: a repository's `AGENTS.md` is not loaded for the agent, so it reads it because the instructions say so.
4. `wayai push`.

## What it can do

Clone the repository at its branch or commit, read it, change files, run commands in its sandbox, commit, push a branch of its own, and open that branch's pull request with `open_pull_request`. It ends its reply with the pull request's link. Each call of `open_pull_request` uses a token WayAI mints for that call from the connection's GitHub App, for this repository only, with contents and pull request permissions; no personal token is stored or used. The agent never names the repository: the tool opens the pull request on the one in `repo_url`.

Each session gets its own sandbox and its own copy of the repository. A change to `repo_url` or `repo_checkout` starts the conversation's next turn in a new session.
