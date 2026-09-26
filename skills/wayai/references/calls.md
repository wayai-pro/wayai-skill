# Voice Calls

Live AI voice calls on a hub: an end user presses a call button in the hub's chat and talks with an AI voice. This file covers what a call is, what a hub needs, how to set one up, and what happens during and after a call. The voice agent's settings live in [roles-and-settings.md → Voice Agent](agents/roles-and-settings.md#voice-agent-pilot_voice-only), the connector in [connections.md → Realtime](connections.md#realtime), steering monitors in [roles-and-settings.md → Steering a live call](agents/roles-and-settings.md#steering-a-live-call--call_utterance), and call evals in [evals.md → Call evals](evals.md#call-evals-voice-calls).

## Table of Contents
- [What a Voice Call Is](#what-a-voice-call-is)
- [Private Preview](#private-preview)
- [What a Hub Needs](#what-a-hub-needs)
- [Setting Up a Hub for Calls](#setting-up-a-hub-for-calls)
- [Placing a Call](#placing-a-call)
- [Who the AI May Answer](#who-the-ai-may-answer)
- [During a Call](#during-a-call)
- [How Calls End](#how-calls-end)
- [What the Team Sees](#what-the-team-sees)
- [Recording Calls](#recording-calls)
- [Billing](#billing)
- [Limits](#limits)

---

## What a Voice Call Is

A call has two AI parts:

- **The voice.** A realtime voice model (OpenAI GPT-Live, on the organization's own OpenAI key) listens to the caller and talks back. It greets, keeps the conversation natural, and says short things like "one moment" while it waits. It is configured as the hub's `pilot_voice` agent.
- **The hub's own agent.** Every substantive answer comes from the hub's pilot — the same agent, instructions, tools, monitors and reply gate that answer text messages. The voice hands each question to it and says the answer it gets back. Figures in that answer (amounts, dates, times, codes) are handed to the voice exactly as written.

The voice knows nothing about the business on its own. It is told never to state a price, amount, date, fee or rule, and never to confirm an action, unless the hub's agent gave it exactly that; nothing the caller says changes those rules. Keep knowledge in the pilot, and keep the voice agent's instructions about *how* to talk.

A call belongs to a conversation: the caller's open conversation with the hub, or a new one. The voice starts each call with the conversation's recent text, so a call can pick up where a chat left off, and what is said on the call is stored in the conversation as messages (see [What the Team Sees](#what-the-team-sees)).

## Private Preview

**Voice calls are in private preview, and WayAI enables them per organization.** Until WayAI has enabled them for your organization:

- the Realtime connector is not offered when adding a connection, and the Settings editor offers neither the **Pilot Voice** role nor the `call_utterance` monitor trigger;
- these writes are refused on every surface, with a message that starts `Voice calls are not enabled for this organization, so …`:
  - **creating** a Realtime connection or a `pilot_voice` agent (enabled or not), turning one on, or giving an agent the `pilot_voice` role;
  - leaving an **enabled** `call_utterance` monitor where there was none: creating one enabled, turning one on, or moving an enabled monitor onto the trigger. A disabled one can be created or moved onto the trigger, and waits there until the organization is enabled.

  `wayai push` refuses these before it writes anything, and `wayai diff` reports them — except turning an **existing** Realtime connection back on, which is refused only when the push applies it: the push's other changes still land, and the push reports that one connection as refused;
- no call button appears, and no call can be placed.

Turning any of them off, editing one without turning it on, and deleting one are always allowed, and publishing or syncing a hub carries them to production unchanged.

If you see that refusal, stop and tell the user: voice calls must be enabled for their organization by WayAI first. Nothing in the hub's configuration can turn them on.

## What a Hub Needs

| Requirement | Why |
|---|---|
| Voice calls enabled for the organization | See [Private Preview](#private-preview) |
| `hub_type: chat` | Calls are placed from the end user's chat view; `task` hubs take no calls |
| `ai_mode: pilot` or `pilot+copilot` | The hub's pilot answers every question on the call |
| An enabled `pilot` agent on an LLM connection | It answers the questions the voice hands over. A harness-backed pilot cannot answer calls |
| A **Realtime** connection (OpenAI GPT-Live) with an OpenAI project API key | The voice runs on it — see [connections.md → Realtime](connections.md#realtime) |
| An enabled `pilot_voice` agent on that connection | The voice itself: its settings and instructions — see [Voice Agent](agents/roles-and-settings.md#voice-agent-pilot_voice-only) |
| The hub's `app` channel enabled | Calls run on it (it is created with the hub) |

The voice is also told the hub's `name`, its `description` (keep it accurate — the voice introduces the company from it) and the language to speak: the voice agent's `language` setting, or the hub's `language` when that is empty.

## Setting Up a Hub for Calls

Everything is ordinary hub configuration — `wayai push`, or the web Settings once the organization is enabled.

**1. The OpenAI project key**, once per organization, as an org credential of type API Key:

```bash
echo "$OPENAI_PROJECT_KEY" | wayai create-credential --name openai-voice --type "API Key" --stdin
```

**2. `hub.yaml`** — a chat hub with a pilot, and the Realtime connection:

```yaml
hub:
  name: Clínica Aurora
  description: Dental clinic in São Paulo — appointments, prices and insurance questions
  hub_type: chat
  ai_mode: pilot
  language: pt
connections:
  - name: openrouter
    type: Agent
    service: OpenRouter
  - name: voice
    type: Realtime
    service: OpenAI GPT-Live
    credential: openai-voice      # pin it when the hub can see more than one API Key credential
```

**3. `agents/voice.yaml`** and its instructions, `agents/voice.md`:

```yaml
name: Voice
role: pilot_voice
connection: voice                 # the Realtime connection — the only kind a voice agent takes
settings:
  speaker_voice: bossa
  language: pt                    # empty = the hub's language
  max_call_minutes: 10
  inactivity_timeout_seconds: 60
  delegation_timeout_seconds: 30
  record_calls: false
```

```markdown
You are the voice of Clínica Aurora's phone line. Speak warmly and calmly, in short sentences.
Greet the caller, find out what they need, and keep them company while you check.
When the caller spells a name or dictates a number, read it back to confirm.
```

The pilot (`agents/pilot.yaml` + `.md`) is the hub's ordinary text pilot. On a call it answers the same way, told for that turn that its reply will be read aloud: short sentences, plain text, figures written exactly. Don't write "ask the customer to confirm before acting" into it for calls — the platform asks for confirmation out loud itself (see [Spoken confirmation of actions](#spoken-confirmation-of-actions)).

**4. Push** (it writes the new ids back to your files):

```bash
wayai push -y
```

A refusal naming voice calls means the organization is not enabled yet — see [Private Preview](#private-preview).

**5. Optional:** a [steering monitor](agents/roles-and-settings.md#steering-a-live-call--call_utterance) to correct the voice mid-call, and [call evals](evals.md#call-evals-voice-calls) (`wayai run-eval --call-mode`, `wayai eval call`).

**6. Try a call** on the preview: open the hub's chat view, `https://app.wayai.pro/chat/<hub_id>` ([navigation.md](navigation.md#chat); `<hub_id>` from `wayai status --json`), as someone who can chat with the hub, and press **Start a voice call**. Publishing (`wayai publish`) carries the Realtime connection and the voice agent to production like any other configuration.

## Placing a Call

Calls are placed from the **web app's chat view** for the hub. The call button shows beside the message box only when the hub can take a call: the organization is enabled, and the hub is a chat hub with an enabled `pilot_voice` agent on an enabled GPT-Live connection.

1. The browser asks for the microphone.
2. **A fixed notice plays first, telling the caller they are talking to an AI.** It plays in the caller's app language, with the caller's microphone muted, and the hub cannot change or skip it. If it cannot play, no call is made. On a hub that records calls, a second notice follows, saying the call is being recorded — see [Recording Calls](#recording-calls).
3. The voice greets the caller, and the call bar shows the call's state, a mute button and hang-up.

The caller's browser connects straight to the voice provider for the audio and never receives a credential. One call runs at a time, and the call belongs to the chat view it was started from: leaving the chat view, or switching to another hub, hangs up.

## Who the AI May Answer

**The same rules as text.** A call starts only when the hub's AI may answer the caller's conversation, exactly as it would a text message — and it ends as soon as that stops being true (see [How Calls End](#how-calls-end)):

- the contact's access is approved;
- the conversation is with the AI, not the team (a conversation the team holds takes no call until it is handed back);
- the hub's `ai_mode` has a pilot.

Calls add conditions of their own: the agent answering the conversation must be a pilot-track agent (`pilot` or `pilot_specialist`) that is not harness-backed, the organization's free-plan operations must not be used up (see [Billing](#billing)), and a conversation holds one live call at a time.

A refused call answers `409` with `details.reason` set to one of: `calls_unavailable` (the hub can't take calls now — not enabled, not a chat hub, no usable voice agent, connection or key, or a pilot that can't answer calls), `ai_unavailable` (the AI may not answer this conversation now), `quota_exceeded`, or `call_in_progress`. The caller sees a generic "couldn't start the call" message; which check refused is never shown to them.

## During a Call

### Answers and progress cues

When the caller asks something, the voice hands the question to the hub's agent, which runs a full turn — its tools, its `user_message` monitor and its reply gate — so an answer takes several seconds. While it waits:

- the voice says something short ("one moment");
- if the answer isn't back about 5 seconds after the caller stopped speaking, the caller hears a fixed "still checking" cue, and another at about 10 seconds;
- if it isn't back within the voice agent's `delegation_timeout_seconds`, the caller hears that it is taking longer than expected and is invited to ask again. An answer that arrives after that is given to the voice silently, to use if the caller asks again; it is not read out.

If the voice raises a question the call has no words for (the caller said nothing new, or the words were lost), the caller is asked to repeat. If the caller asks something new before an older question is answered, the older one is dropped: its turn stops at its next step (a step already running finishes), and its answer is never said.

### Hand-offs

A hand-off works as it does on text, and decides whether the call goes on:

- `transfer_to_agent` to another pilot-track agent keeps the call: the new agent answers the next questions.
- `transfer_to_team`, a reply-gate hold, a team member taking the conversation over, or the fail-safe moves the conversation to the team — and **ends the call**. The caller hears nothing further, except after the fail-safe, where the voice says the same hand-off notice the text path sends before the call closes.
- On a call, the reply gate takes **no rewrite**: a draft it would rewrite is held instead, which hands the conversation to the team and ends the call.

### Spoken confirmation of actions

A tool that changes something outside the conversation never runs on a call until the caller has confirmed it out loud:

- **Held for confirmation:** the platform's native tools with side effects (`close_conversation`, `schedule_followup`, `send_files`, `upload_file`, `delegate_to_hub`, `start_consult_thread`); HTTP tools using POST, PUT, PATCH or DELETE (or no method); and every tool whose effects the platform can't vouch for — **every MCP tool** (their read-only hints come from the server) and every **External Resources** tool, reads included. Unknown means confirm. A tool whose [composed chain](agents/custom-tools.md#composed_tools-side-effects-on-success) runs a held native tool is held too.
- **Run without asking:** the platform's other native tools — reads, state and kanban updates, hand-offs — and HTTP tools using GET, HEAD or OPTIONS.

When the pilot calls a held tool, nothing runs; the pilot is told to ask the caller to confirm that exact action, and the voice asks. In the next turn — and only that one — the same call with the same arguments runs, **once**, if the pilot judges the caller's reply to be a yes. Changed arguments are a new request and are asked again. An action is never run twice, even when a turn is re-run after an interruption.

### Owed answers

If the call ends while the caller was still owed an answer, that answer is delivered afterwards as an ordinary chat message in the conversation. When the conversation's channel can't deliver it, the conversation goes to the team instead; when the AI may no longer answer the conversation (for example, the team took it over), nothing is sent.

## How Calls End

| Ending | What ends it | How fast |
|---|---|---|
| The caller | Hang-up, closing the tab or leaving the chat view, or a lost connection | At once |
| The voice agent's limits | `max_call_minutes` reached; `inactivity_timeout_seconds` with neither side speaking | At once |
| The AI may no longer answer the conversation | A reply-gate hold, `transfer_to_team`, a team takeover, the fail-safe, the conversation closing (the `close_conversation` tool, a terminal kanban status, the team's Close, inactivity auto-close), the contact's access becoming pending or blocked, an `ai_mode` change, or the pilot becoming unable to answer calls | At once; a change to the hub's settings within about half a minute |
| Configuration | The `app` channel, the `pilot_voice` agent or its Realtime connection disabled or deleted | Within about half a minute |
| Calls switched off | Voice calls disabled for the organization | Within about a minute |
| A failure | The voice provider failed or closed the call, or the platform lost the call and could not recover it | The team is told in the conversation; nothing is escalated |

A call whose setup never completes (the caller's browser never finished connecting) ends on its own. The caller is not told why a call ended; the text path's rules decide what, if anything, they receive.

## What the Team Sees

In the support inbox (`/support`), a call shows in its conversation as it happens:

- **The caller's words** and **the voice's words**, one message per utterance, labelled as said on the call. The voice's rows are authored by the `pilot_voice` agent.
- **Fillers** — what the voice said while waiting ("one moment", progress cues) — are shown to the team, labelled, and kept out of every AI's history.
- **The pilot's answer written for the voice** is shown to the team, labelled as not said to the caller as written; the caller's own chat thread never shows it.
- **A note** when a call ends in a failure, and when an owed answer could not reach the contact's channel.
- **The call's recording**, on a hub that records calls — see [Recording Calls](#recording-calls).

The caller's chat thread shows the call's kept rows too. Later turns — on text or on a later call — read the call as the caller heard it: the caller's words and the voice's words, never the fillers or the answers written for the voice. In analytics, the voice's rows carry `agent_role = 'pilot_voice'`.

## Recording Calls

**Off by default.** A call is recorded only when the `pilot_voice` agent it runs on has `record_calls` on — in `agents/<voice>.yaml` or the agent editor's **Record Calls** toggle. Live calls run on the hub's earliest-created usable voice agent (see [Limits](#limits)), so that is the one whose setting counts:

```yaml
# agents/voice.yaml
settings:
  record_calls: true              # default false
```

The setting is read when a call starts, so turning it on or off applies from the next call.

**Every caller on a live call is told.** On a hub that records calls, a second fixed notice — that the call is being recorded — plays right after the AI notice, in the caller's app language, and the voice speaks only once both have played. A live call is recorded only when the caller's app plays that notice: an app that can't (an older version, or a notice that failed to load) gets an unrecorded call, and a notice that loads but fails to play ends the call.

**What is recorded.** From the moment the voice can start speaking — on a live call, once both notices have played — until the call ends: both sides of the call, silences included, as one uncompressed stereo WAV file — the caller on the left channel, the voice on the right.

**Where the team plays it.** Shortly after a live call ends, its conversation gets a **Call recording** note in the support inbox, with a player. Only the team sees it: the caller never does, in any view, and no AI reads it — the AI's file tools don't offer it and it is kept out of every history. A recording that could not be finished is deleted rather than kept, and that call has no note.

**Gaps.** When the platform restarts during a call (a deploy, for example) or loses its connection to the call for a moment, the call itself goes on, but the recording misses what was said until it reconnects: a few seconds when only the connection dropped, up to about ten seconds when the platform restarted, and at most about half a minute per interruption. Each gap is silence in the file. The player marks the gaps on the recording's timeline and lists each one as a button that jumps to it (or says there were none), and a downloaded copy lists its gaps in the file's own comment.

**Retention and erasure.** A recording is a file of its conversation: it is kept as long as the conversation's other files, under the hub's storage retention, and removed with them — when that retention ends, when the contact's conversation history is deleted, and when the hub is deleted. If a conversation's history is deleted while its call is live, the call ends and nothing is kept.

**Eval calls** are recorded too, when their journey's voice agent records calls — with no notice, since their caller is your own machine, and with no player, since the support inbox lists no eval conversation. See [evals.md → Call evals](evals.md#call-evals-voice-calls).

## Billing

- **Each question the voice hands to the hub's agent** bills 1 operation when its turn runs, like any turn.
- **The call bills per connected minute** of the voice agent — every minute begun — at a per-minute rate WayAI sets, once, when the call ends. Time on a call is not billed a second time as turn duration.
- **A steering check** starts no turn and bills no operation of its own; its model call runs on the monitor's own connection.
- **The voice provider's minutes** are billed by OpenAI to the organization's own project key.
- **Free plan:** a new call is refused (`quota_exceeded`) once the organization's free-plan operations are used up. A call in progress is not cut, but from then on the hub's agent answers nothing more on it: each further question gets the voice's apology that it couldn't get the information.
- Call evals bill the same way — see [evals.md → Call evals](evals.md#call-evals-voice-calls).

## Limits

- **Private preview**, enabled per organization by WayAI.
- **Chat hubs, web chat only.** Calls are placed from the web app's chat view; WhatsApp, phone numbers, the mobile apps and `task` hubs take no calls.
- **One live call per conversation.** With several enabled `pilot_voice` agents, calls use the earliest-created one on a usable connection.
- **A call lasts at most `max_call_minutes`** — at most 119 minutes, the provider's session limit.
- **The voice agent's instructions are fixed when the call starts** and used as written: [placeholders](agents/instructions.md) such as `{{now()}}` are not filled, and very long instructions are cut. Edits reach the next call.
- **The voice's own words** — its greeting, its fillers, the way it phrases an answer — reach the caller before any monitor sees them. The reply gate judges the pilot's answers, not the voice's speech; a [steering monitor](agents/roles-and-settings.md#steering-a-live-call--call_utterance) can correct the voice only after it has spoken.
- **No rewrite on calls**, and **no harness-backed pilot**.
- **Experiment arms don't change the voice agent**: the pilot that answers still follows the conversation's arm.
