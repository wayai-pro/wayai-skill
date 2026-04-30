# Analytics Operations

Reference for WayAI analytics via the CLI: `wayai analytics` (summary) and `wayai analytics query` (structured). Both hit `/api/analytics/*` with the same auth and grants as the platform UI. See [CLI Access](#cli-access).

## Table of Contents
- [Overview](#overview)
- [Variable Categories](#variable-categories)
- [Variable Types](#variable-types)
- [Filter Operations](#filter-operations)
- [Time Analysis](#time-analysis)
- [Analytics Tools](#analytics-tools)
- [CLI Access](#cli-access)
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

Analytics variables are organized into five categories:

| Category | Description | Example Variables |
|----------|-------------|-------------------|
| `conversation_metrics` | General conversation statistics | message_count, response_time, duration |
| `instruction_following` | How well agents follow instructions | completion_rate, deviation_count |
| `escalation_performance` | Escalation handling metrics | escalation_rate, resolution_time |
| `function_calling` | Tool usage analytics | tool_call_count, success_rate |
| `user_satisfaction` | User feedback metrics | satisfaction_score, feedback_sentiment |

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

CLI command reference and flags live in [`AGENTS.md`](../../../../AGENTS.md) under "Files + CLI" / Analytics; this file stays focused on the conceptual model.

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
