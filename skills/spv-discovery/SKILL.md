---
name: spv-discovery
description: "Use when users want to explore available SPVs, browse asset classes, discover what analytics or views exist for a transaction, or understand the data model (filterable columns, column types)"
---

# SPV Discovery

Help users navigate the Equalizer platform — find SPVs, understand what analytics are available, and explore the data model. This skill is for exploration and orientation, not data retrieval.

## When to Use

- "What SPVs do I have?"
- "What asset classes are available?"
- "What analytics can I run on [SPV]?"
- "What views are assigned to [SPV]?"
- "What columns/fields can I filter on?"
- "What data does this transaction have?"
- Any exploratory question that isn't asking for a specific metric or KPI

For actual data queries ("what's the delinquent balance?"), use the **equalizer-analytics** skill instead.

## Tenant Selection

Same as equalizer-analytics: identify the tenant before any tool call. Tools are prefixed with the server name (e.g., `mcp__plugin_equalizer_equalizer-staging__list_spvs`).

## Discovery Workflows

### Browse SPVs

1. Call `list_spvs(name="")` for a broad listing, or `list_spvs(name="...")` to search
2. Results are capped at 15. If `total_matches > returned`, tell the user and suggest narrowing by name
3. Group results by asset class in your response

**Response format:**

> **[N] SPVs found** ([M] total if capped)
>
> **Automobile Loan or Lease** ([count])
> - [SPV Name] — `[id]`
>
> **Corporate Loan** ([count])
> - [SPV Name] — `[id]`
>
> [etc.]

### Discover Available Analytics

1. Call `list_template_analytics(spv_id="...")` for curated analytics
2. Call `list_spv_views(spv_id="...")` for assigned views
3. If both return empty, tell the user: "No analytics or views are configured for this SPV."

**Response format:**

> **Curated Analytics** — [N] views available
> | View | Metrics | Segmented By |
> |---|---|---|
> | [display_name] | [metric(s) with operator] | [dimension(s)] |
>
> **Assigned Views** — [N] views available
> | View | Metrics | Segmented By | Chart Type |
> |---|---|---|---|
> | [display_name] | [metric(s)] | [dimension(s)] | [chart_types] |

If views exist, end with: "Ask me about any of these to retrieve the data."

### Explore Data Model

1. Pick a view (ask user which one if ambiguous, or use the first available)
2. Call `get_filterable_columns(strat_view_id="...", spv_id="...")`
3. Present columns grouped by data type

**Response format:**

> **[View Name]** — [N] filterable columns
>
> **Amounts** ([count])
> - [Display Name] — [description if available]
>
> **Dates** ([count])
> - [Display Name] — [description if available]
>
> **Strings** ([count])
> - [Display Name] — [description if available]
>
> **Other** ([count])
> - [Display Name] ([data_type]) — [description if available]

Include filter operator guidance:
- Strings: =, !=, in, not in, is null, is not null
- Numbers/amounts: all operators including range
- Dates: comparison operators + range
- Booleans: =, !=, is null, is not null

## Tone

Same professional ABF tone as equalizer-analytics. Present structured results, no filler. If the user seems new to the platform, briefly orient them ("This SPV has 3 views covering portfolio balance, delinquency, and vintage loss data") but don't over-explain.
