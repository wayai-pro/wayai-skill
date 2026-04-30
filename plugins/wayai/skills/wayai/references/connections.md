# Connections

Setup guide for WayAI hub connections. Non-OAuth connections are **auto-created by `wayai push`** from organization credentials. OAuth connections require **UI** setup.

For the canonical, programmatic source of connector definitions (auth schemas, settings schemas, model lists), inspect [`workers/shared/src/catalog/connectors.ts`](../../../../../workers/shared/src/catalog/connectors.ts) — this reference reflects that catalog and is updated alongside it.

## Table of Contents
- [Organization Credentials](#organization-credentials)
- [Creating Connections via CLI](#creating-connections-via-cli)
- [Connector Types](#connector-types)
- [Agent](#agent)
- [Channel](#channel)
- [Tool - Native](#tool---native)
- [Tool - Custom](#tool---custom)
- [Tool - MCP](#tool---mcp)
- [STT](#stt)
- [TTS](#tts)
- [Quick Reference](#quick-reference)

---

## Organization Credentials

Organization credentials store API keys, tokens, and passwords at the organization level. Instead of entering credentials per-connection, you create a credential once and reference it when adding connections to any hub.

**Supported auth types:** API Key, Bearer Token, Basic Auth

**Setup (UI only):**
1. Settings → Organization → Credentials tab
2. Click "+ Add Credential"
3. Enter name, description, auth type, and credential fields
4. Save — credentials are stored securely in vault

**Using with CLI (`wayai push`):**

Add connections to `hub.yaml` and push — `wayai push` auto-creates them using matching org credentials:
```yaml
connections:
  - name: anthropic
    type: Agent
    service: Anthropic
  - name: my-api
    type: Tool - Custom
    service: User Tool
```

**Benefits:**
- **No duplication** — same API key used across multiple hubs
- **CLI-compatible** — agents can create connections via files + push (no raw secrets needed)
- **Easy rotation** — update one credential, all connections use the new value

**Supported connectors (non-OAuth):**
- All Agent connectors (OpenAI, Anthropic, Google AI Studio, OpenRouter, OpenCode)
- Channel connectors with API Key auth (Resend, Telegram)
- Tool - Custom (User Tool, OpenCode Tool)
- Tool - MCP (MCP Server — API Key auth)
- Tool - Native (External Resources)
- STT (Groq STT, OpenAI STT)
- TTS (OpenAI TTS, Groq TTS, ElevenLabs TTS)

**Not supported (OAuth — requires UI):**
- Channel connectors (WhatsApp, Instagram)
- Tool - Native (Google Calendar)

> **Note:** MCP Server supports both API Key and OAuth authentication types. Only OAuth requires UI setup — API Key connections are auto-created from org credentials.

---

## Creating Connections via CLI

Non-OAuth connections are auto-created by `wayai push` using matching organization credentials. Add them to the `connections` section of `hub.yaml`:

### YAML Format

```yaml
connections:
  - name: anthropic           # Display name for the connection
    type: Agent               # Connector type (see Quick Reference)
    service: Anthropic        # Connector service name
  - name: my-custom-api
    type: Tool - Custom
    service: User Tool
  - name: mcp-server
    type: Tool - MCP
    service: MCP Server
```

### How Auto-Creation Works

1. `wayai push` reads the `connections` section from `hub.yaml`
2. For each connection not yet on the hub, it finds a matching connector by `type` + `service`
3. It looks up a matching organization credential (by auth type)
4. Creates the connection automatically — no secrets needed in the repo

### Requirements

- Organization credential must exist (create in UI: Settings → Organization → Credentials)
- The credential's auth type must match one of the connector's `authentication_types` (e.g., API Key for OpenAI)
- OAuth connections (WhatsApp, Instagram, Google Calendar) cannot be auto-created — use UI

### Example: Full Hub Setup via CLI

```
1. User creates hub in platform UI
2. wayai init --hub <hub-id>                    → scope repo to the hub
3. wayai pull -y                                → sync local files
4. Edit hub.yaml and agents/*.yaml — add connections, agents, tools
5. wayai push -y                                → auto-creates connections, agents, tools
```

---

## Connector Types

| Type | Description |
|------|-------------|
| `Agent` | LLM providers for AI agents (OpenAI, Anthropic, Google AI Studio, OpenRouter, OpenCode) |
| `Channel` | Messaging channels (WhatsApp, Instagram, Resend, Telegram) |
| `Tool - Native` | Platform-provided tool integrations (Wayai, Google Calendar, External Resources) |
| `Tool - Custom` | Custom API integrations you create (User Tool, OpenCode Tool) |
| `Tool - MCP` | External MCP server connections |
| `STT` | Speech-to-text services (Groq STT, OpenAI STT) |
| `TTS` | Text-to-speech services (OpenAI TTS, Groq TTS, ElevenLabs TTS) |

---

## Agent

LLM providers for AI functionality. **At least one Agent connection required before creating agents.**

### Available Connectors

| Connector | connector_id | Auth | Description |
|-----------|--------------|------|-------------|
| OpenAI | `0cd6a292-895b-4667-b89e-dd298628c272` | API Key | LLM provider for OpenAI GPT models. |
| Anthropic | `b3c4d5e6-f7a8-9012-bcde-f12345678902` | API Key | LLM provider for Anthropic Claude models. |
| Google AI Studio | `c4d5e6f7-a8b9-0123-cdef-234567890123` | API Key | LLM provider for Google Gemini models. |
| OpenRouter | `4d7e9f23-1a2b-4c3d-9e8f-5a6b7c8d9e0f` | API Key | Multi-provider LLM gateway with access to OpenAI, Anthropic, Google, xAI, DeepSeek, MiniMax, Kimi, GLM, Qwen, and Xiaomi models. |
| OpenCode | `e6f7a8b9-c0d1-2345-ef01-234567890abc` | Basic Auth | Connect to a deployed OpenCode server for self-hosted agentic loops with custom tools and instructions. |

### OpenAI

**Prerequisites:** OpenAI API key from [platform.openai.com](https://platform.openai.com)

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Agent** group, click the **OpenAI** card
3. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **API Key** (required): Your OpenAI API key
4. Click Save

### Anthropic

**Prerequisites:** Anthropic API key from [console.anthropic.com](https://console.anthropic.com)

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Agent** group, click the **Anthropic** card
3. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **API Key** (required): Your Anthropic API key
4. Click Save

### Google AI Studio

**Prerequisites:** Google AI Studio API key from [aistudio.google.com](https://aistudio.google.com)

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Agent** group, click the **Google AI Studio** card
3. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **API Key** (required): Your Google AI Studio API key
4. Click Save

### OpenRouter

**Prerequisites:** OpenRouter API key from [openrouter.ai](https://openrouter.ai)

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Agent** group, click the **OpenRouter** card
3. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **API Key** (required): Your OpenRouter API key
4. Click Save

### OpenCode

Connect to a self-hosted OpenCode server. Unlike other Agent connectors, OpenCode runs its own agentic loop on the server side — tools, MCP servers, and custom instructions are configured in `AGENTS.md` on the OpenCode server, not in the WayAI agent settings.

**Prerequisites:**
- A deployed OpenCode server (e.g., on Railway) with `OPENCODE_SERVER_PASSWORD` set
- The server's base URL

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Agent** group, click the **OpenCode** card
3. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **Server URL** (required): Base URL of the OpenCode server (e.g., `https://your-opencode-server.up.railway.app`)
   - **Username**: Basic auth username (defaults to `opencode`)
   - **Password** (required): Basic auth password (`OPENCODE_SERVER_PASSWORD` on the server)
4. Click Save

**Agent settings:**
- `model: opencode` (single fixed value — actual model is configured server-side)
- `agent_name`: Optional. Name of an OpenCode agent defined in `AGENTS.md` on the server (e.g. `Build`, `Plan`). Leave empty for the default agent.

---

## Channel

Messaging channels for customer communication.

### Available Connectors

| Connector | connector_id | Auth | Description |
|-----------|--------------|------|-------------|
| WhatsApp | `5fb214cb-aaa8-4b3d-8c65-c9370b3e7c85` | OAuth | WhatsApp via Meta Business API with embedded signup. |
| Instagram | `f9e8d7c6-5b4a-3210-9876-543210fedcba` | OAuth | Instagram Direct Messages via Meta Business API. |
| Resend | `a1b2c3d4-e5f6-4a89-b012-3e5e0d000001` | API Key | Send and receive emails via Resend API with email forwarding. |
| Telegram | `a1b2c3d4-e5f6-4a89-b012-3e5e0d000002` | API Key | Send and receive Telegram messages via Bot API. |

### Instagram

**Prerequisites:**
- Instagram Business or Creator account
- Connected to Facebook Page
- Meta Business account

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Channel** group, click the **Instagram** card
3. Click "Connect with Meta"
4. Authorize Instagram messaging permissions
5. Select Instagram account
6. Connection created automatically

**Features:** Instagram DMs, auto-refresh (7 days).

### WhatsApp

**Prerequisites:**
- Meta Business account (or create one during signup)
- Phone number for WhatsApp Business (can be new or existing)

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Channel** group, click the **WhatsApp** card
3. Click "Connect with Meta" → Meta embedded signup opens

**Step 1 - Welcome Screen:**
- Review the connection benefits (Cloud API for messaging at scale)
- Click **Continue**

**Step 2 - Select Business Portfolio:**
- Choose an existing Business portfolio from the dropdown, or
- Click **Create a Business portfolio** if you don't have one
- Click **Next**

**Step 3 - Select WhatsApp Business Account:**
- Choose an existing WhatsApp Business account, or
- Click **Create a WhatsApp Business account** to create new, or
- Click **Connect a WhatsApp Business App** to migrate existing app
- Click **Next**

**Step 4 - Select Facebook Page:**
- Choose an existing Facebook Page from your portfolio, or
- Click **Create a Facebook Page** to create new
- Click **Next**

**Step 5 - Select Pixel (optional):**
- Choose an existing Meta Pixel or create new
- Select an Ad Account for conversion tracking
- Click **Next**

**Step 6 - Add WhatsApp Phone Number:**
- **Use a display name only**: Send messages showing only your business name
- **Use a new or existing WhatsApp number**: Register a phone number (verification required)
- **Add a phone number later**: Skip and configure in Meta Business Settings
- If adding a number: enter phone number and choose verification method (Text message or Phone call)
- Click **Next**

**Step 7 - Review Permissions:**
- Review what WayAI will access (WhatsApp Business account, Ad account, Facebook page, Dataset)
- Click **Confirm**

**Step 8 - Complete:**
- "Your account is connected to Wayai" confirmation appears
- Optionally click **Add payment method** for WhatsApp conversations billing
- Click **Finish**

**After Setup:** Connection created automatically in WayAI.

**Features:** Automatic token refresh (7 days), CTWA + Conversions API.

### Resend

Email send + receive via [Resend](https://resend.com). Resend uses an API key tied to your verified domain and supports email forwarding for inbound.

**Prerequisites:**
- Resend account with a verified domain
- Resend API key from [resend.com/api-keys](https://resend.com/api-keys)
- A `from` email address on your verified domain (e.g., `support@yourdomain.com`)

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Channel** group, click the **Resend** card
3. Fill the form:
   - **Account ID** (required): Your `from` email address (e.g., `support@yourdomain.com`)
   - **Connection Name** (required): A name to identify this connection
   - **Resend API Key** (required): Format `re_xxxxxxxxxx` from resend.com/api-keys
4. Click Save

**Features:** Outbound email send via Resend; inbound email handling via Resend webhooks (Svix-signed).

### Telegram

Send and receive Telegram messages via the [Bot API](https://core.telegram.org/bots/api).

**Prerequisites:**
- A Telegram bot created via [@BotFather](https://t.me/BotFather)
- The bot's API token (from `/newbot` in BotFather)

**Setup:**
1. In Telegram, message [@BotFather](https://t.me/BotFather), send `/newbot`, follow the prompts, and copy the resulting token
2. Settings → Organizations → Project → Hub → Connections
3. In the **Channel** group, click the **Telegram** card
4. Fill the form:
   - **Account ID** (required): Bot username (e.g., `@my_bot`)
   - **Connection Name** (required): A name to identify this connection
   - **Bot Token** (required): The token from BotFather
5. Click Save

**Features:** Outbound and inbound messages, secret-token webhook signature verification (`X-Telegram-Bot-Api-Secret-Token`).

---

## Tool - Native

Platform-provided tool integrations that extend agent capabilities.

See [agents/native-tools.md](agents/native-tools.md) for available tools and their parameters.

### Available Connectors

| Connector | connector_id | Auth | Status | Description |
|-----------|--------------|------|--------|-------------|
| **Wayai** | `b17d9f3a-4e1b-46c9-b648-a2f0c3611aa4` | None | **Auto-enabled** | Native tools for conversation management, file handling, skills, and tool orchestration. |
| Google Calendar | `189c2e74-2275-43b6-8dac-0fb3b782e9de` | OAuth | Enabled | Manage Google Calendar events and check availability. |
| External Resources | `e8f9a0b1-2c3d-4e5f-6789-0abcdef12345` | API Key | Enabled | Connect to external file storage services. |

### Wayai (Auto-enabled)

> **Important:** The Wayai connection is **automatically created and enabled** when a hub is created. No manual setup required.

This is the core native toolset providing:
- **Conversation management:** Close conversations, transfer to team, update kanban status, schedule followups
- **Agent orchestration:** Transfer to agent, consult agent
- **File handling:** Get files, send files from conversations
- **Resource access:** List resource folders, list resource files
- **Tool orchestration:** Get tool schema, execute tool dynamically
- **Skills:** Read skill, read skill file

**Agent tools:** `close_conversation`, `transfer_to_team`, `update_kanban_status`, `schedule_followup`, `transfer_to_agent`, `consult_agent`, `get_files`, `send_files`, `list_resource_folders`, `list_resource_files`, `get_tool_schema`, `execute_tool`, `read_skill`, `read_skill_file`

### Google Calendar

**Prerequisites:** Google account with Calendar access

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Tool - Native** group, click the **Google Calendar** card
3. Click "Connect with Google"
4. Authorize calendar access
5. Connection created automatically

**Agent tools:** `list_events`, `create_event`, `update_event`, `delete_event`, `check_availability`

### External Resources

Connect to external file storage services for agent file access.

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Tool - Native** group, click the **External Resources** card
3. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **Storage API URL** (required): External storage service API endpoint (e.g., `https://storage.example.com/api`)
   - **API Key** (required): Your storage service API key
4. Click Save

**Agent tools:** `get_external_files`, `send_external_files`

---

## Tool - Custom

Custom API integrations you create. Connect agents to your own APIs or third-party services. A Tool - Custom connection is **required** before creating custom tools — each tool must reference a connection for authentication.

See [agents/custom-tools.md](agents/custom-tools.md) for how to create custom tools.

### Available Connectors

| Connector | connector_id | Auth | Description |
|-----------|--------------|------|-------------|
| User Tool | `b15fb991-63e1-4a79-a174-d10aa66f4414` | API Key, Basic Auth, Bearer Token | Connect custom REST APIs using API key, basic auth, or bearer token authentication. |
| OpenCode Tool | `f7a8b9c0-d1e2-3456-f012-345678901bcd` | Basic Auth | Send tasks to a deployed OpenCode server and retrieve results. |

### User Tool

Connect to any REST API using API key, basic auth, or bearer token authentication.

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Tool - Custom** group, click the **User Tool** card
3. Choose an authentication type:
   - **API Key** — provide an API key (and optional access token)
   - **Basic Auth** — provide username and password
   - **Bearer Token** — provide a bearer token
4. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **Base URL** (optional): Base URL for the API (e.g., `https://api.example.com`)
   - Auth credentials (varies by type — see above)
   - **Custom Headers** (optional): Additional headers to include in all requests
5. Click Save

**Usage:** A Tool - Custom connection is required before creating custom tools. Each custom tool must reference a connection for authentication.

### OpenCode Tool

Connect custom tools to a deployed OpenCode server. Use this when you want a specific tool (e.g., a code-review or build endpoint) to dispatch to OpenCode without making the entire agent OpenCode-driven.

**Prerequisites:**
- A deployed OpenCode server with `OPENCODE_SERVER_PASSWORD` set
- The server's base URL

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Tool - Custom** group, click the **OpenCode Tool** card
3. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **Server URL** (required): Base URL of the OpenCode server (e.g., `https://your-opencode-server.up.railway.app`)
   - **Username**: Basic auth username (defaults to `opencode`)
   - **Password** (required): Basic auth password (`OPENCODE_SERVER_PASSWORD` on the server)
   - **Connection Headers** (optional): Additional headers
4. Click Save

**Usage:** Reference this connection from custom tools in `agents/<slug>.yaml` to call OpenCode endpoints. See [agents/custom-tools.md](agents/custom-tools.md).

---

## Tool - MCP

Connect external MCP (Model Context Protocol) servers to extend agent capabilities.

### Available Connectors

| Connector | connector_id | Auth | Description |
|-----------|--------------|------|-------------|
| MCP Server | `f1a2b3c4-d5e6-7890-abcd-ef1234567890` | API Key, OAuth | Connect to external MCP servers with optional API key or OAuth 2.0 authentication. |

### MCP Server

Connect to external MCP servers with optional API key or OAuth 2.0 authentication.

**Prerequisites:** MCP server URL (Streamable HTTP endpoint)

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **Tool - MCP** group, click the **MCP Server** card
3. Choose an authentication type:
   - **API Key** — provide a bearer token (or leave empty for no auth)
   - **OAuth** — complete OAuth 2.0 authorization flow (RFC 9728)
4. Fill the form:
   - **Connection Name** (required): A friendly name for this MCP connection
   - **MCP Server URL** (required): The Streamable HTTP endpoint (e.g., `https://mcp.example.com/mcp`)
   - For **API Key**: **Bearer Token** (optional), **Custom Headers** (optional)
   - For **OAuth**: **OAuth Client ID** (optional), **OAuth Client Secret** (optional)
5. Click Save → Tools auto-discovered (OAuth connections complete authorization flow first)

**After setup:** Use the Sync button to refresh available tools when the MCP server is updated.

**Features:** OAuth connections include automatic token refresh (1 hour).

---

## STT

Speech-to-text services for transcribing voice messages.

### Available Connectors

| Connector | connector_id | Auth | Description |
|-----------|--------------|------|-------------|
| Groq STT | `78328cbf-19d5-4310-9c37-fea2d792f356` | API Key | Fast speech-to-text transcription using Groq's Whisper implementation. |
| OpenAI STT | `c3d4e5f6-7a8b-4c9d-0e1f-2a3b4c5d6e7f` | API Key | Speech-to-text transcription using OpenAI Whisper API. |

### Groq STT

**Prerequisites:** Groq API key from [console.groq.com](https://console.groq.com)

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **STT** group, click the **Groq STT** card
3. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **API Key** (required): Your Groq API key
4. Click Save

**Usage:** Transcribes voice messages to text using Whisper models.

### OpenAI STT

**Prerequisites:** OpenAI API key

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **STT** group, click the **OpenAI STT** card
3. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **API Key** (required): Your OpenAI API key
4. Click Save

**Usage:** Transcribes voice messages to text.

---

## TTS

Text-to-speech services for generating voice responses.

### Available Connectors

| Connector | connector_id | Auth | Description |
|-----------|--------------|------|-------------|
| OpenAI TTS | `b2c3d4e5-f6a7-4b89-c012-3456789abcdf` | API Key | Text-to-speech synthesis using OpenAI voices. |
| Groq TTS | `d7e8f9a0-1b2c-4d3e-8f40-5a6b7c8d9e0f` | API Key | Expressive text-to-speech using Canopy Labs Orpheus models via Groq (Preview). |
| ElevenLabs TTS | `a1b2c3d4-e5f6-4789-a012-3456789abcde` | API Key | High-quality text-to-speech with custom voice cloning. |

### OpenAI TTS

**Prerequisites:** OpenAI API key

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **TTS** group, click the **OpenAI TTS** card
3. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **API Key** (required): Your OpenAI API key
4. Click Save

**Voices available:** alloy, ash, ballad, coral, echo, fable, nova, onyx, sage, shimmer, verse

**Models:** `gpt-4o-mini-tts` (default — supports `instructions` for custom speaking style), `tts-1`, `tts-1-hd`

### Groq TTS

Expressive text-to-speech via Groq using Canopy Labs Orpheus models. **Preview** — model availability may change.

**Prerequisites:** Groq API key from [console.groq.com](https://console.groq.com)

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **TTS** group, click the **Groq TTS** card
3. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **API Key** (required): Your Groq API key
4. Click Save

**Models:** `canopylabs/orpheus-v1-english` (default), `canopylabs/orpheus-arabic-saudi`

**Voices:** English: `troy`, `hannah`, `austin`. Arabic: see [Groq docs](https://console.groq.com/docs/text-to-speech). Default voice: `hannah`.

### ElevenLabs TTS

**Prerequisites:** ElevenLabs API key from [elevenlabs.io](https://elevenlabs.io)

**Setup:**
1. Settings → Organizations → Project → Hub → Connections
2. In the **TTS** group, click the **ElevenLabs TTS** card
3. Fill the form:
   - **Connection Name** (required): A name to identify this connection
   - **API Key** (required): Your ElevenLabs API key
4. Click Save

**Usage:** High-quality voice synthesis with custom voices.

---

## Quick Reference

| Connector | connector_id | Type | Auth |
|-----------|--------------|------|------|
| OpenAI | `0cd6a292-895b-4667-b89e-dd298628c272` | Agent | API Key |
| Anthropic | `b3c4d5e6-f7a8-9012-bcde-f12345678902` | Agent | API Key |
| Google AI Studio | `c4d5e6f7-a8b9-0123-cdef-234567890123` | Agent | API Key |
| OpenRouter | `4d7e9f23-1a2b-4c3d-9e8f-5a6b7c8d9e0f` | Agent | API Key |
| OpenCode | `e6f7a8b9-c0d1-2345-ef01-234567890abc` | Agent | Basic Auth |
| WhatsApp | `5fb214cb-aaa8-4b3d-8c65-c9370b3e7c85` | Channel | OAuth |
| Instagram | `f9e8d7c6-5b4a-3210-9876-543210fedcba` | Channel | OAuth |
| Resend | `a1b2c3d4-e5f6-4a89-b012-3e5e0d000001` | Channel | API Key |
| Telegram | `a1b2c3d4-e5f6-4a89-b012-3e5e0d000002` | Channel | API Key |
| **Wayai** | `b17d9f3a-4e1b-46c9-b648-a2f0c3611aa4` | Tool - Native | None (auto-created) |
| Google Calendar | `189c2e74-2275-43b6-8dac-0fb3b782e9de` | Tool - Native | OAuth |
| External Resources | `e8f9a0b1-2c3d-4e5f-6789-0abcdef12345` | Tool - Native | API Key |
| User Tool | `b15fb991-63e1-4a79-a174-d10aa66f4414` | Tool - Custom | API Key, Basic Auth, Bearer Token |
| OpenCode Tool | `f7a8b9c0-d1e2-3456-f012-345678901bcd` | Tool - Custom | Basic Auth |
| MCP Server | `f1a2b3c4-d5e6-7890-abcd-ef1234567890` | Tool - MCP | API Key, OAuth |
| Groq STT | `78328cbf-19d5-4310-9c37-fea2d792f356` | STT | API Key |
| OpenAI STT | `c3d4e5f6-7a8b-4c9d-0e1f-2a3b4c5d6e7f` | STT | API Key |
| OpenAI TTS | `b2c3d4e5-f6a7-4b89-c012-3456789abcdf` | TTS | API Key |
| Groq TTS | `d7e8f9a0-1b2c-4d3e-8f40-5a6b7c8d9e0f` | TTS | API Key |
| ElevenLabs TTS | `a1b2c3d4-e5f6-4789-a012-3456789abcde` | TTS | API Key |
