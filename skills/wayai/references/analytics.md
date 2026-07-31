# Analytics Operations

Reference for WayAI analytics via the CLI: `wayai analytics` (summary), `wayai analytics query` (structured), and `wayai analytics sql` (raw SQL, including per-message cost). All hit `/api/analytics/*` with the same auth and grants as the platform UI. See [CLI Access](#cli-access).

## Table of Contents
- [Overview](#overview)
- [Variable Categories](#variable-categories)
- [Variable Types](#variable-types)
- [Filter Operations](#filter-operations)
- [Time Analysis](#time-analysis)
- [Analytics Tools](#analytics-tools)
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
1. get_analytics_variables(hub_id)    → Discover available metrics
2. get_analytics_data(hub_id, ...)    → Query aggregated data
3. get_conversations_list(hub_id, ...)→ Find specific conversations
4. get_conversation_messages(...)     → View message details
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

## Analytics Tools

### get_analytics_variables

Discover all available analytics variables for a hub, organized by category.

```
get_analytics_variables(hub_id)
```

**Parameters:**
- `hub_id` (required): Hub ID to query

**Returns:**
- Variables grouped by category
- Each variable includes: id, name, description, type, origin

**Example output:**
```
## Analytics Variables

### Conversation Metrics (5)
- **message_count** (numeric)
  - ID: `cv_abc123`
  - Description: Total messages in conversation

- **response_time** (numeric)
  - ID: `cv_def456`
  - Description: Average response time in seconds
```

---

### get_analytics_data

Query analytics data with aggregations and optional trend analysis.

```
get_analytics_data(
  hub_id,           # Required
  variable_ids,     # Required: list of variable IDs
  start_date,       # Required: YYYY-MM-DD
  end_date,         # Required: YYYY-MM-DD
  periodicity,      # Optional: daily|weekly|monthly|yearly (default: daily)
  include_trend,    # Optional: true for time series data (default: false)
  include_summary,  # Optional: true for conversation summary (default: true)
  filters           # Optional: list of filter conditions
)
```

**Returns:**
- Summary statistics (total conversations, AI-only rate, response times)
- Aggregated metrics per variable
- Trend data if requested

**Example:**
```
get_analytics_data(
  hub_id="abc123",
  variable_ids=["cv_message_count", "cv_response_time"],
  start_date="2024-01-01",
  end_date="2024-01-31",
  periodicity="weekly",
  include_trend=true
)
```

---

### get_conversations_list

List conversations in a hub with optional filtering and pagination.

```
get_conversations_list(
  hub_id,       # Required
  start_date,   # Optional: filter by date range
  end_date,     # Optional
  limit,        # Optional: max results (default: 50)
  offset,       # Optional: pagination offset (default: 0)
  filters       # Optional: variable filters
)
```

**Returns:**
- List of conversations with:
  - conversation_id
  - participant info
  - message count
  - last updated timestamp

**Example:**
```
get_conversations_list(
  hub_id="abc123",
  start_date="2024-01-01",
  end_date="2024-01-31",
  limit=20,
  filters=[
    { variable_id: "cv_message_count", filter_type: "gte", filter_value: 10 }
  ]
)
```

---

### get_conversation_messages

Get full message history for a specific conversation.

```
get_conversation_messages(
  hub_id,           # Required (for access verification)
  conversation_id,  # Required
  limit,            # Optional: max messages (default: 100)
  offset            # Optional: pagination (default: 0)
)
```

**Returns:**
- Conversation metadata
- List of messages with:
  - sender type (user, assistant, support)
  - message text
  - timestamp

**Example:**
```
get_conversation_messages(
  hub_id="abc123",
  conversation_id="conv_xyz789",
  limit=50
)
```

---

### pin_analytics_variable

Pin or unpin a variable for quick access in the UI.

```
pin_analytics_variable(
  hub_id,       # Required
  variable_id,  # Required
  pinned        # Required: true to pin, false to unpin
)
```

**Note:** Requires write access to the hub.

---

## CLI Access

CLI command reference and flags live in the skill's **Common CLI Commands** section (SKILL.md → Common CLI Commands); this file stays focused on the conceptual model.

## Raw SQL & Cost Analysis

`wayai analytics sql "<SELECT>"` runs a single read-only SELECT. Two tables are readable:

| Table | Grain | Carries |
|-------|-------|---------|
| `conversation` | one row per conversation | everything in [Variable Categories](#variable-categories) — `data.system.*`, `data.variables.*`, `data.meta.*`, `data.annotations.*` |
| `message` | one row per stored message | `data.system.*` usage facts only: tokens, USD cost, speech/container usage, operations |

**Write bare table names and no scope clause.** The server rewrites a bare `conversation` or `message` into a subquery already filtered to your hub (`WHERE hub_id = … AND is_eval = false`, with `FINAL`), so adding `hub_id`, `is_eval`, or `FINAL` yourself is redundant. Statements are also restricted to a single SELECT, no DDL/DML, no `UNION`, and a 10,000-row cap; `conversation` and `message` are reserved CTE names.

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
3. **An unfiltered sum EXCEEDS the conversation figure, by design.** Message rows include background agents (`conversation_evaluator`, `message_evaluator`, `monitor`) that every conversation aggregate excludes; their spend is real and reaches no other queryable store. To reproduce the conversation figure, filter **NULL-safely** — a bare `NOT IN` on a nullable column silently drops every user/team/system row. **Keep the outer parentheses:** `AND` binds tighter than `OR`, so an unparenthesized copy combined with any other condition silently re-parses as `agent_role IS NULL OR (… AND …)` and readmits every NULL-role row from all history:
   ```sql
   WHERE (agent_role IS NULL OR agent_role NOT IN ('conversation_evaluator','message_evaluator','monitor'))
   ```
4. **`connection_id` names the LLM connection only.** TTS/STT/container spend rides the same row, so attributing cost to a credential means netting the media terms out of `cost_usd_total`.
5. **Bucket dates in the hub's timezone,** not raw UTC: `toTimeZone(created_at, 'America/Sao_Paulo')` before `toDate`/`toStartOfWeek`. Raw UTC bucketing misassigns everything near midnight.
6. **Bound `message.created_at`** — and only that one. `message` retains five years at per-message volume, so an unbounded scan is the expensive way to ask every question; every example below carries a window. In a join, do **not** add the matching bound on `conversation`: `conversation.created_at` is when the conversation *opened*, not when the spend happened, and the join is inner — so a chat conversation opened 60 days ago and closed yesterday would have all of its in-window spend dropped from the result, silently. Bounding the message side alone is what "spend in the last 30 days" means, and it is the table whose volume matters.

### Example queries

Every example casts (rule 1) and bounds time (rule 6). Replace the 30-day window and the timezone with the hub's.

Spend by model — where the money actually goes:
```bash
wayai analytics sql "SELECT agent_model, sum(toFloat64OrNull(toString(data.system.cost_usd_total))) AS usd, sum(toFloat64OrNull(toString(data.system.tokens_total))) AS tokens, count() AS messages FROM message WHERE created_at >= now() - INTERVAL 30 DAY GROUP BY agent_model ORDER BY usd DESC"
```

Spend by agent role, reconciled to the customer-visible figure (rule 3 — note the parentheses):
```bash
wayai analytics sql "SELECT agent_role, sum(toFloat64OrNull(toString(data.system.cost_usd_total))) AS usd FROM message WHERE created_at >= now() - INTERVAL 30 DAY AND (agent_role IS NULL OR agent_role NOT IN ('conversation_evaluator','message_evaluator','monitor')) GROUP BY agent_role ORDER BY usd DESC"
```

What background evaluation costs you — the spend conversation analytics hides:
```bash
wayai analytics sql "SELECT agent_role, sum(toFloat64OrNull(toString(data.system.cost_usd_total))) AS usd FROM message WHERE created_at >= now() - INTERVAL 30 DAY AND agent_role IN ('conversation_evaluator','message_evaluator','monitor') GROUP BY agent_role ORDER BY usd DESC"
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

1. get_analytics_variables(hub_id) → Find relevant variables
2. get_analytics_data(hub_id, variables, yesterday, today, include_summary=true)
   → Get summary stats and key metrics
```

### Workflow 2: Investigate Response Times

```
User: "Which conversations had slow response times last week?"

1. get_analytics_variables(hub_id) → Find response_time variable ID
2. get_conversations_list(hub_id, last_week, today, filters=[
     { variable_id: "response_time", filter_type: "gt", filter_value: 300 }
   ])
   → List conversations with >5min response time
3. get_conversation_messages(hub_id, conversation_id)
   → Review specific slow conversations
```

### Workflow 3: Trend Analysis

```
User: "Show me how escalation rates changed over the last 3 months"

1. get_analytics_variables(hub_id) → Find escalation_rate variable
2. get_analytics_data(hub_id, [escalation_rate_id],
     start_date="2024-01-01", end_date="2024-03-31",
     periodicity="weekly", include_trend=true)
   → Get weekly trend data
```

### Workflow 4: Find High-Value Conversations

```
User: "Show me conversations with positive feedback"

1. get_analytics_variables(hub_id) → Find satisfaction/feedback variables
2. get_conversations_list(hub_id, filters=[
     { variable_id: "sentiment", filter_type: "is", filter_value: "positive" }
   ])
   → List conversations with positive sentiment
3. get_conversation_messages(hub_id, conversation_id)
   → Learn from successful interactions
```

---

## Quick Reference

| Tool | Purpose |
|------|---------|
| `get_analytics_variables` | Discover available metrics |
| `get_analytics_data` | Query aggregated analytics |
| `get_conversations_list` | List/filter conversations |
| `get_conversation_messages` | View message history |
| `pin_analytics_variable` | Pin variables for quick access |
