# Analytics Operations

Reference for WayAI analytics via the CLI: `wayai analytics` (summary), `wayai analytics query` (structured), and `wayai analytics sql` (raw SQL, including per-message cost). All hit `/api/analytics/*` with the same auth and grants as the platform UI. See [CLI Access](#cli-access).

## Table of Contents
- [Overview](#overview)
- [Variable Categories](#variable-categories)
- [Variable Types](#variable-types)
- [Filter Operations](#filter-operations)
- [Time Analysis](#time-analysis)
- [Analytics Variables](#analytics-variables)
- [CLI Access](#cli-access)
- [Raw SQL & Cost Analysis](#raw-sql--cost-analysis)
- [Common Workflows](#common-workflows)

## Overview

WayAI Analytics provides insights into hub conversations and agent performance. Data is organized into:

1. **Variables** - Metrics that can be tracked (e.g., message_count, response_time, satisfaction_score)
2. **Categories** - Logical groupings of variables
3. **Filters** - Conditions to narrow down data
4. **Time Intervals** - Date ranges for analysis

### Quick Start

```
1. wayai analytics --list-metrics         → Discover available metrics
2. wayai analytics --period 7d            → Summary block (add --metric for aggregates)
3. wayai conversations --json             → Find specific conversations and their ids
4. wayai conversations <id> observability → Per-turn detail for one conversation
```

---

## Variable Categories

Analytics variables are organized into six categories:

| Category | Description | Example Variables |
|----------|-------------|-------------------|
| `conversation_metrics` | General conversation statistics | message_count, response_time, duration |
| `instruction_following` | How well agents follow instructions | completion_rate, deviation_count |
| `escalation_performance` | Escalation handling metrics | escalation_rate, resolution_time |
| `function_calling` | Tool usage analytics | tool_call_count, success_rate |
| `user_satisfaction` | User feedback metrics | satisfaction_score, feedback_sentiment |
| `annotations` | Post-hoc business outcomes set after a conversation ends (via `wayai conversations <id> annotate`) — a third variable origin (`data.annotations.*`) distinct from agent-set variables | customer_purchased, refund_amount, churned |

### Category Details

**Conversation Metrics**
- Core metrics about conversation flow
- Includes: message counts, response times, conversation duration
- Available for all hubs by default

**Instruction Following**
- Measures agent adherence to defined instructions
- Useful for quality assurance and training
- Origin: Typically from agent evaluations

**Escalation Performance**
- Tracks human handoff efficiency
- Includes: escalation triggers, resolution times, escalation rates
- Critical for support team optimization

**Function Calling**
- Tool and function usage patterns
- Includes: which tools are called, success/failure rates, latency
- Helps optimize tool configurations

**User Satisfaction**
- End-user feedback and sentiment
- Includes: ratings, sentiment analysis, feedback text
- Origin: Collected from user interactions

**Annotations**
- Post-hoc business outcomes labeled AFTER a conversation ends — e.g. customer_purchased, refund_amount, churned
- Set via the CLI (`wayai conversations <id> annotate --set key=value [--type numeric|categorical|text]`), not inferred by the AI
- A third variable origin (`data.annotations.*`), distinct from agent-set variables (`data.variables.*`) and system metrics (`data.system.*`) — use it to correlate agent/evaluator predictions against real outcomes (e.g. predicted purchase intent vs actual purchase)
- The conversation must still be within its hub's retention window so its working copy is annotatable after it ends; values are coerced to the declared `--type` so they aggregate like any other dimension

---

## Variable Types

Each variable has a type that determines how it can be filtered and aggregated:

### Numeric Variables
Quantitative measurements that support mathematical operations.

**Aggregations:**
- Average, Sum, Minimum, Maximum, Median, Count

**Example variables:**
- `message_count` - Number of messages in conversation
- `response_time_seconds` - Time to first response
- `token_count` - Total tokens used

### Categorical Variables
Discrete values from a predefined set of options.

**Aggregations:**
- Distribution (count per category)

**Example variables:**
- `sentiment` - positive, neutral, negative
- `escalation_reason` - timeout, user_request, agent_triggered
- `resolution_status` - resolved, pending, escalated

### Text Variables
Free-form text data for qualitative analysis.

**Aggregations:**
- Word frequency, text search

**Example variables:**
- `feedback_comment` - User feedback text
- `escalation_notes` - Support notes

---

## Filter Operations

Filters narrow down analytics results based on variable values.

### Numeric Filters

| Operator | Description | Example |
|----------|-------------|---------|
| `eq` | Equals | `message_count eq 10` |
| `neq` | Not equals | `message_count neq 0` |
| `gt` | Greater than | `response_time gt 60` |
| `gte` | Greater than or equal | `message_count gte 5` |
| `lt` | Less than | `duration lt 300` |
| `lte` | Less than or equal | `token_count lte 1000` |

### Text Filters

| Operator | Description | Example |
|----------|-------------|---------|
| `equals` | Exact match | `status equals "resolved"` |
| `not_equals` | Not exact match | `status not_equals "pending"` |
| `contains` | Contains substring | `feedback contains "helpful"` |
| `not_contains` | Doesn't contain | `notes not_contains "error"` |
| `starts_with` | Starts with | `name starts_with "John"` |
| `ends_with` | Ends with | `email ends_with "@company.com"` |

### Categorical Filters

| Operator | Description | Example |
|----------|-------------|---------|
| `is` | Equals category | `sentiment is "positive"` |
| `is_not` | Not equals category | `status is_not "pending"` |
| `in` | In list of values | `channel in ["whatsapp", "email"]` |
| `not_in` | Not in list | `reason not_in ["spam", "test"]` |

---

## Time Analysis

Analytics can be viewed as period summaries or trends over time.

### Periodicity Options

| Period | Description | Use Case |
|--------|-------------|----------|
| `daily` | Day-by-day breakdown | Short-term analysis, daily reports |
| `weekly` | Week-by-week | Medium-term trends, weekly reviews |
| `monthly` | Month-by-month | Long-term patterns, monthly reports |
| `yearly` | Year-by-year | Annual comparisons |

### Time Interval Format

Dates use ISO format: `YYYY-MM-DD`

```
start_date: "2024-01-01"
end_date: "2024-01-31"
```

### Trend Analysis

Enable `include_trend: true` to get data points over time:

```
Without trend: { average: 12.5, sum: 1250, min: 1, max: 45 }
With trend: [
  { period: "2024-01-01", value: 10 },
  { period: "2024-01-02", value: 15 },
  ...
]
```

---

## Analytics Variables

Every variable a hub tracks is discoverable from the CLI: `wayai analytics --list-metrics`
names them, and `wayai analytics --metric <name>[,<name>]` aggregates them. A variable
carries an id, name, description, type and origin, and belongs to one of the categories
above. (`wayai analytics query` is a different surface — it takes raw ClickHouse column
paths, as `--metric "col=data.system.tokens_total,agg=avg,alias=avg_tokens"`, never variable
names; see [Quick Reference](#quick-reference).)

```
### Conversation Metrics (5)
- **message_count** (numeric)
  - ID: `cv_abc123`
  - Description: Total messages in conversation

- **response_time** (numeric)
  - ID: `cv_def456`
  - Description: Average response time in seconds
```

Pinning a variable for quick access is a UI action (Analytics tab); it changes presentation
only and has no CLI or config-as-code equivalent.

---


## CLI Access

CLI command reference and flags live in the skill's **Common CLI Commands** section (SKILL.md → Common CLI Commands); this file stays focused on the conceptual model.

## Raw SQL & Cost Analysis

`wayai analytics sql "<SELECT>"` runs a single read-only SELECT. Two tables are readable:

| Table | Grain | Carries |
|-------|-------|---------|
| `conversation` | one row per conversation | everything in [Variable Categories](#variable-categories) — `data.system.*`, `data.variables.*`, `data.meta.*`, `data.annotations.*` |
| `message` | one row per stored message | `data.system.*` usage facts only: tokens, USD cost, speech/container usage, operations |

**Write bare table names and no scope clause.** The server rewrites a bare `conversation` or `message` into a subquery already filtered to your hub (`WHERE hub_id = … AND is_eval = false`, with `FINAL`), so adding `hub_id`, `is_eval`, or `FINAL` yourself is redundant. Statements are also restricted to a single SELECT, no DDL/DML, no `UNION`, and a 10,000-row cap. `conversation` and `message` cannot be rebound in the statement's leading `WITH` list, in either the `name AS (…)` or the `<expr> AS name` form.

Database-qualified names (`wayai.message`) are rejected in `FROM`/`JOIN`, and whitespace or a comment between the segments does not change that.

Treat that as *how to write a query that works*, not as a boundary to probe. Two separate mechanisms keep tenants apart: the endpoint authorizes the hub you asked for, and the injected predicate scopes the rows you get back. Neither is optional, and neither is something your SQL should try to reach around.

Run `wayai analytics sql --schema` for both tables' columns and variable paths.

### What lives at which grain

Message rows carry the **additive** variables — anything you can sum across a conversation's messages: `tokens_total`, `tokens_input`, `tokens_output`, the cache-token variants, `cost_usd_*` (input, output, cache_read, cache_write, stt, tts, container, total), `stt_*` / `tts_*`, `code_execution_requests_total`, `container_sessions_total`, `billable_operations_total`, and `extra_latency_ops` (operations charged for residency beyond the base, not a latency measurement).

`wayai analytics sql --schema` prints the authoritative list with descriptions; the sentence above is a summary of it, not a second source of truth.

Everything non-additive stays conversation-only: counts, durations, response times, outcomes, sentiment, kanban status. A duration has no meaning on a single message.

### Six rules that decide whether your query runs, and whether its numbers are right

1. **Cast every `data.*` path — uncast, the query does not run.** JSON sub-paths resolve to ClickHouse's `Dynamic` type, which aggregates reject (`Illegal type Dynamic of argument for aggregate function sum`) and which `GROUP BY` refuses (`Data types Variant/Dynamic are not allowed in GROUP BY keys`). Two casts, by use:
   - **numeric** (`sum`, `avg`, `min`, `max`, and any arithmetic): `toFloat64OrNull(toString(data.system.cost_usd_total))`
   - **grouping / text**: `toString(data.meta.kanban_status)`

   Top-level columns (`agent_model`, `agent_role`, `connection_id`, `created_at`, …) are typed and need no cast — this applies to `data.*` only.

   Mind which table owns the namespace: `message` carries `data.system.*` **only**. A missing JSON sub-path returns NULL instead of erroring, so selecting `data.meta.*` from `message` does not fail — it silently collapses to one empty group. Reach conversation-owned paths by joining `conversation`, as the last example does.
2. **Rows appear at conversation CLOSE, best-effort.** An open conversation contributes nothing. Emission is less durable than the conversation row, so `message` is analysis-grade — the `conversation` figure remains the customer-visible, billing-reconcilable one.
3. **An unfiltered sum EQUALS the conversation figure** (for conversations closed after the boundaries in 3b). Both count every agent role, background included, so `message` *decomposes* the conversation rollup rather than exceeding it. Use the filter below only to **isolate** the spend a hub's own agents did, splitting it from platform background work (`conversation_evaluator`, `message_evaluator`, `monitor`, `summarizer`). It lands at or below the conversation total — `filtered <= total`, strict only when an excluded row actually contributed to the metric you selected; a conversation that ran no background agent has nothing to exclude, so equality there is correct rather than a fault. Filter **NULL-safely**: a bare `NOT IN` on a nullable column silently drops every user/team/system row. **Keep the outer parentheses:** `AND` binds tighter than `OR`, so an unparenthesized copy combined with any other condition silently re-parses as `agent_role IS NULL OR (… AND …)` and readmits every NULL-role row from all history:
   ```sql
   WHERE (agent_role IS NULL OR agent_role NOT IN ('conversation_evaluator','message_evaluator','monitor','summarizer'))
   ```
   Operations are *unchanged* by this filter — background rows carry zero — so only the cost and token sums move.
   **Historical boundaries — there are two,** both forward-only with no backfill, so a conversation's aggregate reflects whichever rules were live when it **closed**:
   - `summarizer` spend entered **both** tables when the provider-usage carrier shipped. Before that it was recorded nowhere at all, so no query recovers the earlier figures.
   - `conversation_evaluator`/`message_evaluator`/`monitor` spend entered the **`conversation`** aggregate later, when the role exclusion narrowed to shape only. `message` carried those rows all along, so for a window spanning only that second change, summing `message` is the continuous answer.
4. **`connection_id` names the LLM connection only.** TTS/STT/container spend rides the same row, so attributing cost to a credential means netting the media terms out of `cost_usd_total`.
5. **Bucket dates in the hub's timezone,** not raw UTC: `toTimeZone(created_at, 'America/Sao_Paulo')` before `toDate`/`toStartOfWeek`. Raw UTC bucketing misassigns everything near midnight.
6. **Bound `message.created_at`** — and only that one. `message` retains five years at per-message volume, so an unbounded scan is the expensive way to ask every question; every example below carries a window. In a join, do **not** add the matching bound on `conversation`: `conversation.created_at` is when the conversation *opened*, not when the spend happened, and the join is inner — so a chat conversation opened 60 days ago and closed yesterday would have all of its in-window spend dropped from the result, silently. Bounding the message side alone is what "spend in the last 30 days" means, and it is the table whose volume matters.

### Example queries

Every example casts (rule 1) and bounds time (rule 6). Replace the 30-day window and the timezone with the hub's.

Spend by model — where the money actually goes:
```bash
wayai analytics sql "SELECT agent_model, sum(toFloat64OrNull(toString(data.system.cost_usd_total))) AS usd, sum(toFloat64OrNull(toString(data.system.tokens_total))) AS tokens, count() AS messages FROM message WHERE created_at >= now() - INTERVAL 30 DAY GROUP BY agent_model ORDER BY usd DESC"
```

Spend by agent role, limited to the hub's own agents (rule 3 — note the parentheses):
```bash
wayai analytics sql "SELECT agent_role, sum(toFloat64OrNull(toString(data.system.cost_usd_total))) AS usd FROM message WHERE created_at >= now() - INTERVAL 30 DAY AND (agent_role IS NULL OR agent_role NOT IN ('conversation_evaluator','message_evaluator','monitor','summarizer')) GROUP BY agent_role ORDER BY usd DESC"
```

What background work costs — the share of the conversation figure that is evaluation, monitoring and summarization rather than customer-facing replies:
```bash
wayai analytics sql "SELECT agent_role, sum(toFloat64OrNull(toString(data.system.cost_usd_total))) AS usd FROM message WHERE created_at >= now() - INTERVAL 30 DAY AND agent_role IN ('conversation_evaluator','message_evaluator','monitor','summarizer') GROUP BY agent_role ORDER BY usd DESC"
```

Cost per credential, media netted out (rule 4 — each term casts separately):
```bash
wayai analytics sql "SELECT connection_id, sum(toFloat64OrNull(toString(data.system.cost_usd_total)) - toFloat64OrNull(toString(data.system.cost_usd_tts)) - toFloat64OrNull(toString(data.system.cost_usd_stt)) - toFloat64OrNull(toString(data.system.cost_usd_container))) AS llm_usd FROM message WHERE created_at >= now() - INTERVAL 30 DAY GROUP BY connection_id ORDER BY llm_usd DESC"
```

Daily trend in hub time (rule 5):
```bash
wayai analytics sql "SELECT toDate(toTimeZone(created_at, 'America/Sao_Paulo')) AS day, sum(toFloat64OrNull(toString(data.system.cost_usd_total))) AS usd FROM message WHERE created_at >= now() - INTERVAL 30 DAY GROUP BY day ORDER BY day"
```

Cache effectiveness — cache reads are billed at a discount, writes at a premium:
```bash
wayai analytics sql "SELECT agent_model, sum(toFloat64OrNull(toString(data.system.cache_read_input_tokens))) AS cache_reads, sum(toFloat64OrNull(toString(data.system.tokens_input))) AS fresh_input, sum(toFloat64OrNull(toString(data.system.cost_usd_cache_write))) AS write_usd FROM message WHERE created_at >= now() - INTERVAL 30 DAY GROUP BY agent_model ORDER BY write_usd DESC"
```

The most expensive individual conversations:
```bash
wayai analytics sql "SELECT conversation_id, sum(toFloat64OrNull(toString(data.system.cost_usd_total))) AS usd FROM message WHERE created_at >= now() - INTERVAL 30 DAY GROUP BY conversation_id ORDER BY usd DESC LIMIT 20"
```

Spend by workflow stage — both tables in one query, grouping on a `conversation` path (rule 1's text cast):
```bash
wayai analytics sql "SELECT toString(c.data.meta.kanban_status) AS status, sum(toFloat64OrNull(toString(m.data.system.cost_usd_total))) AS usd FROM message m JOIN conversation c ON m.conversation_id = c.conversation_id WHERE m.created_at >= now() - INTERVAL 30 DAY GROUP BY status ORDER BY usd DESC"
```

## Common Workflows

### Workflow 1: Daily Performance Check

```
User: "How did our support hub perform yesterday?"

wayai analytics --period 24h
  → summary stats only: conversations, AI-only rate, response times

wayai analytics --period 24h --metric first_response_time,tokens_total
  → the same summary plus a per-metric aggregate block
```

A bare invocation selects no variables, so only the Summary block prints — pass `--metric`
(names from `--list-metrics`) to get aggregates. `--period` takes `<n><h|d|w|m|y>` (`24h`,
`7d`, `4w`, `3m`, `1y`); for an explicit window use `--from`/`--to` instead.

### Workflow 2: Investigate Response Times

```
User: "Which conversations had slow response times last week?"

1. wayai analytics --period 7d --metric first_response_time \
     --filter "first_response_time gt 300"
     → the slow slice (filters are "<name> <op> <value>", AND'd by repeating --filter)
2. wayai conversations --json
     → find the conversation ids in that slice
3. wayai conversations <id> observability
     → per-turn latency, prompt, tool calls for one slow conversation
```

### Workflow 3: Trend Analysis

```
User: "Show me how escalation rates changed over the last 3 months"

wayai analytics --metric escalation_rate --from 2024-01-01 --to 2024-03-31
  → the aggregate over the window

# Correlate it against another column — `query` takes raw column paths, not metric
# names, and `agg=corr` requires `corr_with`:
wayai analytics query \
  --metric "col=data.variables.escalation_rate,agg=corr,corr_with=data.system.avg_agent_response_time,alias=esc_resp" \
  --from 2024-01-01 --to 2024-03-31

# Break it down by a `data.*` path — `--group-by` casts the grouping key for you, so
# pass the bare path. The grouped column comes back under the underscored form of that
# path (`data_meta_kanban_status`), and `--order-by` takes either a metric alias or a
# grouped path:
wayai analytics query \
  --metric "col=data.variables.escalation_rate,agg=avg,alias=avg_escalation" \
  --group-by data.meta.kanban_status \
  --order-by avg_escalation:desc \
  --from 2024-01-01 --to 2024-03-31

# Drop to sql when the shape above cannot express it — a join, a computed key, several
# grains at once (see Raw SQL & Cost Analysis). There you apply the cast yourself, and
# you bound the window yourself: the server's rewrite scopes the hub, not the time range.
# Rule 6's "bound message.created_at, and only that one" is about the message table in a
# join — this is a conversation-only query, so the bound belongs here:
wayai analytics sql "SELECT toString(data.meta.kanban_status), avg(toFloat64OrNull(toString(data.variables.escalation_rate))) FROM conversation WHERE created_at >= '2024-01-01' AND created_at < '2024-04-01' GROUP BY 1"
```

### Workflow 4: Find High-Value Conversations

```
User: "Show me conversations with positive feedback"

1. wayai analytics --filter "user_sentiment is positive"
     → the positive slice and its size
2. wayai conversations <id> observability
     → read what the agent actually received on a successful interaction
```

For cost and per-message spend, and for anything these fixed shapes cannot express, drop to
`wayai analytics sql` — see [Raw SQL & Cost Analysis](#raw-sql--cost-analysis).

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `wayai analytics` | Summary block; add `--metric` (names from `--list-metrics`) for per-metric aggregates. Also `--filter`, `--period`/`--from`/`--to` |
| `wayai analytics query` | Structured query over raw column paths: `--metric "col=…,agg=…"`, `--group-by`, `--order-by`, correlations. `--metric`, `--group-by`, `--order-by` and `--filter` take `data.*` paths bare — this surface casts those for you, unlike `analytics sql`. On `--filter` the cast follows the operator: `gt`/`gte`/`lt`/`lte` read the path as a number, so a value stored as the JSON string `"7"` compares equal to `7`; `eq`/`neq`/`in`/`not_in`/`like` compare it as text, so a path is matched by its digits whichever way it was stored (`eq 1` matches both `1` and `"1"`). Both work whatever mixture of types the scanned rows store on the path. Give `in`/`not_in` a comma-separated list (`in gold,silver`). A row that never carried the variable matches no filter on it — including a negated one, so `neq gold` returns rows that have a non-gold tier, not rows that merely lack one. A grouped column comes back under the underscored form of its path (`data_meta_kanban_status`) |
| `wayai analytics sql` | Raw single-SELECT over `conversation` and `message` — the cost surface |
| `wayai analytics sql --schema` | Both table catalogs + the message-grain casting rules |
| `wayai conversations` | List/inspect conversations (`--json` to get ids) |
| `wayai conversations <id> observability` | Per-turn record: prompt, completion, tool calls, tokens |
| `wayai conversations <id> annotate` | Record a post-hoc business outcome as a dimension |

Flags and full syntax live in SKILL.md → Common CLI Commands.
