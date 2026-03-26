# Equalizer Claude Code Plugin — Design Spec

## Overview

A Claude Code plugin for the Equalizer fintech platform that bundles MCP server connections, skills, and agents so clients can interact with their transaction/analytics data through Claude. Published on GitHub for client installation.

## Plugin Structure

```
equalizer-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin metadata
├── .mcp.json                    # All tenant MCP server entries (SSE)
├── skills/
│   └── equalizer-analytics/
│       └── SKILL.md             # Core analytics skill
├── agents/
│   └── analytics-agent.md       # Specialized analytics subagent
├── hooks/
│   ├── hooks.json               # Hook config for session-start
│   └── session-start            # Bash script injecting bootstrap context
├── package.json                 # Minimal metadata
└── README.md                    # Installation & usage for clients
```

## MCP Server Configuration

### Approach

All tenants (6-7) are hardcoded in `.mcp.json` as separate SSE entries. Each tenant produces a namespaced set of tools (e.g., `mcp__equalizer_tenant_alpha__list_spvs`). The analytics skill instructs Claude to ask the user which tenant they're working with before calling any tool, then use only tools with that tenant's prefix for the session.

### Format

```json
{
  "equalizer-<tenant-name>": {
    "type": "sse",
    "url": "https://<tenant>.equalizer.com/mcp"
  }
}
```

One entry per tenant. Tenant names and URLs to be provided during implementation.

### Available Tools Per Tenant (7 tools)

| Tool | Module | Purpose |
|------|--------|---------|
| `list_spvs` | securitizations | Search SPVs by name (case-insensitive, max 15 results) |
| `list_template_analytics` | stratification | Discover curated analytics views for an SPV |
| `get_analytics_data` | stratification | Execute curated analytics with optional filters |
| `list_spv_views` | stratification | List all stratification views assigned to an SPV |
| `get_filterable_columns` | stratification | Discover filterable columns on a view |
| `get_strat_view_data` | stratification | Retrieve computed view data in CSV format |
| `get_strat_url` | stratification | Generate frontend URL for a view |

### Authentication

Keycloak OAuth2/OIDC with JWT verification. Multi-tenant — each MCP endpoint handles its own auth. Clients authenticate through Keycloak; unauthorized tenants simply reject requests.

## Design Note: View Discovery Without Semantic Search

The original Dify agent uses a `knowledge_search` RAG tool for semantic matching of user queries to strat views. The MCP server does not expose this — instead, `list_template_analytics` and `list_spv_views` return structured metadata (display_name, description, aggregations, dimensions). The skill compensates by instructing Claude to match user intent against these structured fields. This is less flexible than semantic search but sufficient given the structured metadata quality. If view discovery proves too rigid in practice, a dedicated search/matching MCP tool could be added later.

## Analytics Skill (`skills/equalizer-analytics/SKILL.md`)

### Audience & Tone

Users are ABF and structured finance professionals — portfolio managers, credit analysts, structurers. The skill instructs Claude to:

- Never explain basic concepts (DPD buckets, CPR/SMM, vintage analysis, roll rates, etc.)
- Lead with data and interpretation, not education
- Every sentence either presents data or interprets it

### Tenant Selection

Before any tool call, Claude must ask the user which tenant they're working with. Once identified, only tools prefixed with that tenant name are used for the rest of the session. If the user switches tenants, acknowledge and switch the prefix.

### Two Operating Modes

#### Single-Query Mode (default)

For specific KPI questions, single-metric lookups, and follow-ups.

**Data Retrieval Workflow:**

1. **Find SPV:** `list_spvs` to locate the SPV by name
2. **Discover views:** `list_template_analytics` or `list_spv_views` to find relevant analytical views
3. **Disambiguate:** If multiple views match, stop and present options with metrics/filters/dimensions. Resolve by exact metric name match, filter match, or dimension match before asking.
4. **Filter (optional):** If user requests filtering, call `get_filterable_columns` first, then pass `table_filters` to the data retrieval call
5. **Retrieve data:** `get_analytics_data` (curated) or `get_strat_view_data` (assigned views)
6. **Source link:** `get_strat_url` to generate a frontend URL

**Context Retention:**
- Reuse the active strat view for follow-up questions
- Only switch views when the user explicitly requests a new, unrelated KPI
- Refinement questions (e.g., "what about California?") re-query the same view with filters

**Response Format:**
1. Headline answer (one sentence)
2. Key figures (2-3 bullet points with exact numbers)
3. Context (optional, one sentence on significance)
4. Source link to platform UI

**KPI Selection Logic:**

| Query Type | Focus On |
|---|---|
| Trend ("how has X changed?") | Delta between first and last visible rows |
| Status ("what is X?") | Latest row value or total from stats |
| Segmentation ("where is X strongest?") | Segment with highest/lowest value |
| Bucket ("what about 90+ days?") | Matching cluster label in the dimension |

#### Comprehensive Analysis Mode

Triggered by: "comprehensive analysis", "full analysis", "transaction review", "portfolio report", "analyze this transaction", "what's the state of this Transaction", or any request for a holistic view.

**Step 1: Discover all views** across analytical domains:

| Domain | Search Terms |
|---|---|
| Portfolio Overview | outstanding balance, loan count, weighted average |
| Credit Performance | delinquency, DPD, days past due, days in delay |
| Losses | default, loss, charge-off, write-off |
| Prepayment | prepayment, CPR, SMM, early repayment |
| Concentration | geographic, state, FICO distribution, concentration |

**Step 2: Present scope and get user confirmation.** List discovered domains with view counts. Flag any domains with no data. Ask if user wants full report or specific sections. Do not retrieve data until confirmed.

**Step 3: Deliver report in blocks with checkpoints:**

| Block | Sections |
|---|---|
| Block 1 | Executive Summary & Portfolio Overview |
| Block 2 | Credit Performance Analysis |
| Block 3 | Loss Vintage Analysis |
| Block 4 | Prepayment Vintage Analysis |
| Block 5 | Geographic & FICO Concentration |
| Block 6 | Key Findings & Risk Assessment (synthesizes all prior) |

After each block, offer: continue, skip, or jump to Key Findings. Key Findings only references data from delivered blocks.

**Analytical Techniques:**

- **Portfolio:** Growth trajectory, ramp-up/amortization rate, WA characteristic drift
- **Credit:** DQ rate per bucket, trend direction (stable/rising/accelerating), negative selection detection (lower FICO/higher APR in delinquent loans), flag >15% MoM growth
- **Losses:** Vintage elbow identification, cross-reference defaults by segment, flag segments where default share exceeds portfolio share
- **Prepayment:** Derived SMM (`1 - (1 - CumPrepay)^(1/Term)`) and CPR (`1 - (1 - SMM)^12`), adverse selection detection, composition drift projection (3/6/9/12 months)
- **Concentration:** Flag single-name or single-geography concentration >10%, identify dominant credit band

**Context Window Management:**
- Do not reproduce full data tables — summarize inline (3-5 key figures)
- Highlight outliers, inflection points, key rows
- Link to platform for full datasets
- Tables allowed only if <=5 rows and <=5 columns

### Cross-SPV Comparison

When comparing SPVs:
1. `list_spvs` to find the comparison target
2. Discover views separately for each SPV (strat_view_ids are unique per SPV)
3. Retrieve data using each SPV's own strat_view_id
4. Present comparison side-by-side

### Numerical Formatting

- Balances >1M: round to nearest thousand (e.g., 12.4M)
- Balances <1M: round to nearest unit (e.g., 845,230)
- Counts: exact integers
- Rates/percentages: one decimal place (e.g., 3.2%)
- Basis points: integer (e.g., +99 bps)
- Decimal-to-percentage: convert before rounding (0.0325 = 3.25%)

### Guardrails

- **No cross-SPV strat_id reuse.** Strat view IDs are unique per SPV. When comparing, discover views separately for each SPV.
- **No fabrication.** If data returns empty, state "No data available for [View Name] in this Transaction." Do not estimate.
- **No unsupported forecasting.** Derived projections (SMM-based drift) must be labeled as projected and state assumptions.
- **No investment advice.** Present data and observations only. Use "shows the slowest prepayment speed" not "best investment."
- **No strat view match.** Ask the user to rephrase. Do not guess.
- **Transparency on gaps.** Explicitly note when a section cannot be covered.

## Analytics Agent (`agents/analytics-agent.md`)

### Role

Expert ABF/structured finance analyst co-pilot. Speaks the language of portfolio managers, credit analysts, and structurers.

### When Dispatched

- Comprehensive analysis requests (multi-block reports)
- Cross-SPV comparisons (parallel data retrieval)
- Complex filtered queries requiring multiple tool chains

### Inherited Behavior

The agent follows the same two operating modes, disambiguation protocol, guardrails, numerical formatting, and response formats defined in the analytics skill. It receives the tenant prefix and SPV context from the parent conversation.

### Distinct Purpose

The skill teaches Claude how to use the tools in the main conversation. The agent is a dispatched worker that autonomously chains tool calls for complex tasks and returns a formatted report.

### Analytical Capabilities

All techniques from the skill's Comprehensive Analysis Mode:
- Portfolio growth trajectory and WA drift analysis
- Credit performance DQ bucketing, trend direction, negative selection
- Loss vintage elbow identification and segment over-representation
- Prepayment SMM/CPR derivation, adverse selection, composition drift
- Geographic/FICO concentration flagging

### Report Delivery

Block-by-block with user checkpoints. Context-window-aware: summarize inline, link to platform for full tables. Key Findings cross-references only delivered blocks.

## Session-Start Hook

### Purpose

Injects bootstrap context when the plugin loads so Claude knows the skill system exists from the start.

### Implementation

- `hooks/hooks.json` registers a `SessionStart` hook
- `hooks/session-start` is an executable bash script that:
  1. Reads the `equalizer-analytics/SKILL.md` content
  2. Escapes it for JSON
  3. Outputs `{ "hookSpecificOutput": { "additionalContext": "<skill content>" } }`

Same pattern as the superpowers plugin.

## Plugin Metadata

### `.claude-plugin/plugin.json`

```json
{
  "name": "equalizer",
  "description": "Equalizer platform plugin — analytics tools, skills, and agents for ABF transaction analysis",
  "version": "1.0.0",
  "author": { "name": "Equalizer" },
  "skills": "./skills/",
  "agents": "./agents/",
  "hooks": "./hooks/hooks.json"
}
```

### `package.json`

```json
{
  "name": "equalizer-plugin",
  "version": "1.0.0",
  "type": "module"
}
```

## Installation

Clients install via Claude Code CLI:
```
claude plugin install <github-url>
```

The plugin registers:
- MCP server connections for all tenants
- The analytics skill (available via `/equalizer-analytics`)
- The analytics agent (dispatchable for complex tasks)
- Session-start hook for bootstrap context
