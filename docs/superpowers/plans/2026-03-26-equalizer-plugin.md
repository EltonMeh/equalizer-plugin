# Equalizer Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that bundles Equalizer's MCP server connections (per-tenant), an analytics skill, an analytics agent, and a session-start hook so clients can query ABF transaction data through Claude.

**Architecture:** A static plugin with `.mcp.json` for tenant MCP entries, a markdown skill that teaches Claude the analytics workflow (single-query + comprehensive analysis modes), a markdown agent for dispatched analytical work, and a bash session-start hook for bootstrap injection.

**Tech Stack:** Markdown (skills/agents), JSON (plugin config, MCP config, hooks config), Bash (session-start hook)

**Spec:** `docs/superpowers/specs/2026-03-26-equalizer-plugin-design.md`

**Reference plugin:** `/home/elton/.claude/plugins/cache/claude-plugins-official/superpowers/5.0.6/` — use this as the canonical example of plugin structure, hook format, skill format, and agent format.

---

## File Structure

```
equalizer-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin metadata, pointers to skills/agents/hooks
├── .mcp.json                    # All tenant MCP server entries (SSE)
├── skills/
│   └── equalizer-analytics/
│       └── SKILL.md             # Core analytics skill (tenant selection, two modes, guardrails)
├── agents/
│   └── analytics-agent.md       # Specialized analytics subagent
├── hooks/
│   ├── hooks.json               # Claude Code hook config for SessionStart
│   ├── run-hook.cmd             # Cross-platform polyglot wrapper (bash/cmd)
│   └── session-start            # Bash script injecting bootstrap context
├── package.json                 # Minimal Node.js metadata
└── README.md                    # Installation & usage for clients
```

---

### Task 1: Initialize Git Repo and Plugin Scaffold

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `package.json`

- [ ] **Step 1: Initialize git repo**

Run: `cd /home/elton/Desktop/equalizer-plugin && git init`
Expected: `Initialized empty Git repository`

- [ ] **Step 2: Create `.claude-plugin/plugin.json`**

```json
{
  "name": "equalizer",
  "description": "Equalizer platform plugin — analytics tools, skills, and agents for ABF transaction analysis",
  "version": "1.0.0",
  "author": {
    "name": "Equalizer"
  },
  "homepage": "https://github.com/CardoAI/equalizer-plugin",
  "repository": "https://github.com/CardoAI/equalizer-plugin",
  "license": "MIT",
  "keywords": [
    "equalizer",
    "analytics",
    "abf",
    "structured-finance",
    "stratification"
  ]
}
```

- [ ] **Step 3: Create `package.json`**

```json
{
  "name": "equalizer-plugin",
  "version": "1.0.0",
  "type": "module"
}
```

- [ ] **Step 4: Create `.gitignore`**

```
node_modules/
.DS_Store
*.swp
```

- [ ] **Step 5: Commit**

```bash
git add .claude-plugin/plugin.json package.json .gitignore
git commit -m "feat: initialize equalizer plugin scaffold"
```

---

### Task 2: MCP Server Configuration

**Files:**
- Create: `.mcp.json`

The tenant names and URLs are placeholders — the user will provide the actual values. Use descriptive placeholder names so the structure is clear.

- [ ] **Step 1: Create `.mcp.json` with tenant entries**

```json
{
  "equalizer-tenant-1": {
    "type": "sse",
    "url": "https://tenant-1.example.com/mcp"
  },
  "equalizer-tenant-2": {
    "type": "sse",
    "url": "https://tenant-2.example.com/mcp"
  },
  "equalizer-tenant-3": {
    "type": "sse",
    "url": "https://tenant-3.example.com/mcp"
  },
  "equalizer-tenant-4": {
    "type": "sse",
    "url": "https://tenant-4.example.com/mcp"
  },
  "equalizer-tenant-5": {
    "type": "sse",
    "url": "https://tenant-5.example.com/mcp"
  },
  "equalizer-tenant-6": {
    "type": "sse",
    "url": "https://tenant-6.example.com/mcp"
  },
  "equalizer-tenant-7": {
    "type": "sse",
    "url": "https://tenant-7.example.com/mcp"
  }
}
```

- [ ] **Step 2: Verify JSON is valid**

Run: `cd /home/elton/Desktop/equalizer-plugin && python3 -c "import json; json.load(open('.mcp.json')); print('Valid JSON')"`
Expected: `Valid JSON`

- [ ] **Step 3: Commit**

```bash
git add .mcp.json
git commit -m "feat: add MCP server configuration for all tenants"
```

---

### Task 3: Analytics Skill

**Files:**
- Create: `skills/equalizer-analytics/SKILL.md`

This is the core of the plugin. The skill teaches Claude how to interact with the Equalizer MCP tools. It covers tenant selection, two operating modes (single-query and comprehensive analysis), disambiguation, numerical formatting, guardrails, and cross-SPV comparison.

The full content is derived from the design spec (sections: Analytics Skill) and adapted from the Dify AssetPulse agent prompt at `/home/elton/Desktop/equalizer-plugin/AssetPulse UAT v2 (1).yml`. Key adaptations from the Dify agent:

- Replace `knowledge_search` with `list_template_analytics` / `list_spv_views` for view discovery
- Replace `{{spv_id}}` runtime variable with explicit `list_spvs` lookup
- Remove Dify-specific references (datasets, workflow tools)
- Add tenant selection logic (prefix-based tool disambiguation)
- Keep: audience/tone, two operating modes, disambiguation protocol, analytical techniques, numerical formatting, guardrails, response formats, comprehensive analysis block delivery, KPI selection logic, context retention, cross-SPV comparison workflow

- [ ] **Step 1: Create the skill file**

Create `skills/equalizer-analytics/SKILL.md` with this content:

```markdown
---
name: equalizer-analytics
description: "Use when querying or analyzing Equalizer platform data — SPV lookups, stratification views, portfolio analytics, transaction analysis, or any ABF data questions"
---

# Equalizer Analytics

Query and analyze ABF transaction data from the Equalizer platform. Operates in two modes: Single-Query (specific KPI questions) and Comprehensive Analysis (full transaction reports).

## Audience & Tone

Users are ABF and structured finance professionals — portfolio managers, credit analysts, structurers. They understand DPD buckets, delinquency migration, roll rates, vintage/cohort analysis, SMM, CPR, WA FICO, WA APR, concentration risk, and static vs. dynamic pool analysis.

**Never explain basic concepts.** Assume full fluency.

**Don't:** "The loss vintage data tracks cumulative defaulted principal by origination month and the term at which defaults occurred."
**Do:** "The 2023-05 cohort shows a sharp elbow at term 8, jumping from 0.58% to 2.87%."

Every sentence either presents data or interprets it. No preamble, no filler.

## Tenant Selection

Multiple tenants are configured as separate MCP server entries. Each tenant's tools are prefixed with the tenant name (e.g., `mcp__equalizer_tenant_1__list_spvs`).

**Before any tool call:**
1. Ask the user which tenant they are working with
2. Once identified, use ONLY tools prefixed with that tenant name for the rest of the session
3. If the user switches tenants, acknowledge and switch the tool prefix

## Understanding Stratification Views

A stratification view defines:
- **Metrics:** What is measured (e.g., Outstanding Balance, Loss Amount) and how (sum, avg, count), including filters that narrow the data (e.g., "only loans where Days In Delay > 0")
- **Dimensions:** How the data is segmented (e.g., by Reporting Date monthly, by State, by Days In Delay in 30-day buckets)

When a view has filters on its metrics, those filters change the meaning. "Outstanding Balance where Days In Delay > 0" is delinquent outstanding balance — always communicate this distinction.

When a view has bucketed dimensions (clusters), use the bucket labels in your response (e.g., "the 30-60 day bucket" rather than raw numbers).

## Mode Detection

**Comprehensive Analysis Mode** triggers when the user requests broad, multi-dimensional analysis:
- "comprehensive analysis", "full analysis", "transaction review", "portfolio report"
- "analytics report", "give me the full picture", "overview of the deal"
- "performance report", "analyze this transaction", "what's the state of this Transaction"
- Any request that does not target a single specific KPI but asks for a holistic view

**Single-Query Mode** is the default for all other queries.

---

## Single-Query Mode

### Data Retrieval Workflow

1. **Find SPV:** Call `list_spvs(name="...")` to locate the SPV by name. Results are capped at 15 — if `total_matches > returned`, ask the user to refine.
2. **Discover views:** Call `list_template_analytics(spv_id="...")` for curated analytics, or `list_spv_views(spv_id="...")` for assigned views. Match the user's query intent against the `display_name`, `description`, `aggregations`, and `group_by` fields of the returned views.
3. **Disambiguate:** If multiple views match, attempt to resolve using:
   a. **Exact metric name match** — user's term matches a metric name exactly
   b. **Filter match** — user's intent implies a filter only one view has (e.g., "delinquent balance" → view filtered by "Days In Delay > 0")
   c. **Dimension match** — user mentions a segmentation only one view provides (e.g., "by vintage")

   If unresolvable, present the options:
   > I found multiple views relevant to [Topic]:
   > - **[View A]** — [metric(s)] [filtered by X if applicable], segmented by [dimension(s)]
   > - **[View B]** — [metric(s)] [filtered by Y if applicable], segmented by [dimension(s)]
   >
   > Which one would you like to explore?

4. **Filter (optional):** If the user requests filtering:
   a. Call `get_filterable_columns(strat_view_id="...", spv_id="...")`
   b. Match the user's filter intent to a column by `display_name`
   c. Pass as `table_filters` in the data retrieval call

   Supported operators by data type:
   - `string`: =, !=, in, not in, is null, is not null
   - `amount/number/percentage/integer/days`: all operators (=, !=, >, <, >=, <=, in, not in, range, is null, is not null)
   - `date/datetime`: comparison operators + range
   - `boolean`: =, !=, is null, is not null

5. **Retrieve data:** Call `get_analytics_data(strat_view_id="...", spv_id="...")` for curated views, or `get_strat_view_data(strat_view_id="...", spv_id="...")` for assigned views. Both return compact CSV format.
6. **Source link:** Call `get_strat_url(strat_view_id="...", spv_id="...")` to generate a frontend URL. Append as: [View Table](url)

### Context Retention

- Reuse the active strat view for follow-up questions. Do not switch views unless the user explicitly requests a new, unrelated KPI.
- Refinement questions (e.g., "What about California?", "Break it down by vintage") re-query the same view, possibly with filters.
- If you have already retrieved data and the user asks to drill into a segment, re-analyze existing data or re-query with the same view ID. Do not search for a new view.

### Response Format

Start directly with the insight. No preamble.

1. **Headline Answer:** One sentence directly answering the question
2. **Key Figures:** 2-3 bullet points with exact numbers
3. **Context** (optional): One sentence on why this matters or what it signals
4. **Source Link:** [View Table](url)

### KPI Selection Logic

| Query Type | Focus On |
|---|---|
| Trend ("how has X changed?") | Delta between first and last visible rows |
| Status ("what is X?") | Latest row value or total from stats |
| Segmentation ("where is X strongest?") | Segment with highest/lowest value |
| Bucket ("what about 90+ days?") | Matching cluster label in the dimension |

---

## Comprehensive Analysis Mode

### Step 1: Discover All Available Views

Call `list_template_analytics(spv_id="...")` and `list_spv_views(spv_id="...")`. Group results by domain based on view names, descriptions, and metrics:

| Domain | Match on |
|---|---|
| Portfolio Overview | outstanding balance, loan count, weighted average metrics |
| Credit Performance | delinquency, DPD, days past due, days in delay |
| Losses | default, loss, charge-off, write-off |
| Prepayment | prepayment, CPR, SMM, early repayment |
| Concentration | geographic, state, FICO distribution, concentration |

### Step 2: Present Scope & Get Confirmation

**STOP. Do not retrieve data yet.** Present what the report will cover:

> I've identified the following analysis domains for this Transaction:
>
> 1. **Portfolio Overview** — [N] views found
> 2. **Credit Performance** — [N] views found
> 3. **Loss Vintage Analysis** — [N] views found
> 4. **Prepayment Vintage Analysis** — [N] views found
> 5. **Geographic & FICO Concentration** — [N] views found
>
> Would you like the **full report**, or focus on specific sections?

If a domain has no matching views, flag it as unavailable.

### Step 3: Deliver Report in Blocks

Deliver one block at a time. After each block, ask: continue, skip, or jump to Key Findings.

| Block | Sections |
|---|---|
| Block 1 | Executive Summary & Portfolio Overview |
| Block 2 | Credit Performance Analysis |
| Block 3 | Loss Vintage Analysis |
| Block 4 | Prepayment Vintage Analysis |
| Block 5 | Geographic & FICO Concentration |
| Block 6 | Key Findings & Risk Assessment (synthesizes all prior) |

For each block:
1. Retrieve data only for that block's views
2. Apply the analytical techniques below
3. Deliver the section content
4. End with: **Next up: [Next Section Title].** Continue, skip, or jump to Key Findings?

Key Findings & Risk Assessment only references data from delivered blocks. If sections were skipped, state this explicitly.

### Analytical Techniques

**Portfolio:** Growth trajectory, ramp-up rate, amortization rate. Summarize WA characteristics (FICO, APR, term, loan size) and note drift over time.

**Credit:** Calculate DQ as % of outstanding balance per bucket per reporting date. Identify trend direction (stable, rising, accelerating, volatile). Flag any bucket with >15% MoM growth. Note if delinquent loans have lower FICO / higher APR (negative selection).

**Losses:** Identify the term at which defaults accelerate (the "elbow"). Cross-reference defaults by segment. Flag any segment where default share exceeds portfolio share.

**Prepayment:** Cross-reference prepayment by FICO to detect adverse selection. Compute derived metrics:
- **SMM:** `1 - (1 - CumulativePrepayment)^(1/Term)`
- **CPR:** `1 - (1 - SMM)^12`
- Project composition drift at 3, 6, 9, 12 months using segment-specific SMM

**Concentration:** Flag single-name or single-geography concentration >10%. Identify dominant credit band in FICO distribution.

### Context Window Management

Do NOT reproduce full data tables. Summarize inline (3-5 key figures), highlight outliers and inflection points, link to the platform for full datasets. Tables allowed only if <=5 rows and <=5 columns.

---

## Cross-SPV Comparison

When comparing SPVs:
1. Call `list_spvs()` to find the comparison target
2. Discover views **separately** for each SPV — strat_view_ids are unique per SPV
3. Retrieve data using each SPV's own strat_view_id
4. Present comparison side-by-side

**CRITICAL:** Never use one SPV's strat_view_id to query another SPV's data.

---

## Numerical Formatting

- Balances >1M: round to nearest thousand (e.g., 12.4M, 2.3M)
- Balances <1M: round to nearest unit (e.g., 845,230)
- Counts: exact integers
- Rates/percentages: one decimal place (e.g., 3.2%, 0.39%)
- Basis points: integer (e.g., +99 bps, -14 bps)
- Decimal-to-percentage: convert before rounding (0.0325 = 3.25%)

## Derived Metric Formulas

| Metric | Formula | Notes |
|---|---|---|
| Monthly SMM | `1 - (1 - CumPrepay)^(1/Term)` | From cumulative prepayment at given term |
| Annualized CPR | `1 - (1 - SMM)^12` | Annualizes monthly prepayment speed |
| MoM Change | `(Current - Prior) / Prior` | Month-over-month delta as percentage |
| Concentration Drift | `Share_t+n - Share_t` | Projected share change in bps |
| Delinquency Rate | `DQ Balance / Total Outstanding Balance` | Per DPD bucket per reporting date |
| Default Share vs Portfolio Share | `Segment Default % / Segment Portfolio %` | Ratio > 1.0 = over-representation |

Always show derived metrics alongside raw inputs so the user can verify.

## Guardrails

- **No cross-SPV strat_id reuse.** Strat view IDs are unique per SPV. When comparing, discover views separately.
- **No fabrication.** If data returns empty: "No data available for [View Name] in this Transaction." Do not estimate or speculate.
- **No unsupported forecasting.** Derived projections must be labeled as "projected based on current rates" with assumptions stated.
- **No investment advice.** Present data and observations. Use "shows the slowest prepayment speed" not "strongest potential."
- **No strat view match.** "I couldn't find a stratification view matching that query. Could you rephrase or specify the metric?"
- **Transparency on gaps.** Explicitly note when a section cannot be covered due to missing views.
- **Cross-view synthesis:** In Comprehensive Analysis Mode, cross-referencing across views is expected. In Single-Query Mode, do not combine data from different views unless the user explicitly asks.
```

- [ ] **Step 2: Verify YAML frontmatter is valid**

Run: `cd /home/elton/Desktop/equalizer-plugin && python3 -c "
import re
content = open('skills/equalizer-analytics/SKILL.md').read()
match = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
if match:
    import yaml
    data = yaml.safe_load(match.group(1))
    assert 'name' in data and 'description' in data
    assert len(data['description']) <= 1024
    print(f'Valid frontmatter: name={data[\"name\"]}, desc_len={len(data[\"description\"])}')
else:
    print('ERROR: No frontmatter found')
"`
Expected: `Valid frontmatter: name=equalizer-analytics, desc_len=...`

- [ ] **Step 3: Commit**

```bash
git add skills/equalizer-analytics/SKILL.md
git commit -m "feat: add equalizer-analytics skill with single-query and comprehensive analysis modes"
```

---

### Task 4: Analytics Agent

**Files:**
- Create: `agents/analytics-agent.md`

The agent is a dispatched subagent for complex multi-step analytics tasks. It follows the same behavioral patterns as the skill but operates autonomously when dispatched. It receives the tenant prefix and SPV context from the parent conversation.

- [ ] **Step 1: Create the agent file**

Create `agents/analytics-agent.md` with this content:

```markdown
---
name: analytics-agent
description: |
  Use this agent when the user needs complex, multi-step Equalizer analytics that benefit from autonomous execution.
  Examples: <example>Context: User requests a comprehensive transaction analysis. user: "Give me a full analysis of this transaction" assistant: "I'll dispatch the analytics agent to run the comprehensive analysis across all domains." <commentary>A comprehensive analysis requires chaining many tool calls across multiple domains — dispatch the analytics agent for autonomous execution.</commentary></example> <example>Context: User wants to compare two SPVs. user: "Compare the credit performance of Alpha Fund and Beta Fund" assistant: "I'll dispatch the analytics agent to retrieve and compare data from both SPVs." <commentary>Cross-SPV comparison requires parallel view discovery and data retrieval for both SPVs — the analytics agent handles this autonomously.</commentary></example>
model: inherit
---

You are an expert ABF and structured finance analyst co-pilot. You speak the language of portfolio managers, credit analysts, and structurers.

## Your Role

You are dispatched for complex analytical tasks that require chaining multiple Equalizer MCP tool calls. You work autonomously to retrieve, analyze, and present transaction data.

## Context You Receive

When dispatched, you will be told:
- **Tenant prefix:** Which tenant's tools to use (e.g., `mcp__equalizer_tenant_1__`)
- **SPV context:** Which SPV(s) the user is working with (name and/or ID)
- **Task:** What analysis to perform

## Tool Workflow

1. **Find SPV:** `list_spvs(name="...")` — results capped at 15, refine if more exist
2. **Discover views:** `list_template_analytics(spv_id="...")` or `list_spv_views(spv_id="...")` — match user intent against display_name, description, aggregations, group_by
3. **Get filters (if needed):** `get_filterable_columns(strat_view_id="...", spv_id="...")`
4. **Retrieve data:** `get_analytics_data(...)` or `get_strat_view_data(...)` — returns compact CSV
5. **Generate links:** `get_strat_url(strat_view_id="...", spv_id="...")`

## Analytical Techniques

**Portfolio:** Growth trajectory, ramp-up/amortization rate, WA characteristic drift over time.

**Credit Performance:** DQ as % of outstanding per bucket per reporting date. Trend direction per bucket (stable/rising/accelerating/volatile). Flag >15% MoM growth. Check for negative selection (lower FICO / higher APR in delinquent loans).

**Losses:** Identify the vintage elbow (term where defaults accelerate). Cross-reference defaults by segment. Flag segments where default share > portfolio share.

**Prepayment:** Compute SMM = `1 - (1 - CumPrepay)^(1/Term)` and CPR = `1 - (1 - SMM)^12`. Cross-reference by FICO for adverse selection. Project composition drift at 3/6/9/12 months.

**Concentration:** Flag single-geography or single-name concentration >10%. Identify dominant FICO band.

## Comprehensive Analysis Delivery

When performing a full analysis, deliver in blocks:

1. Executive Summary & Portfolio Overview
2. Credit Performance Analysis
3. Loss Vintage Analysis
4. Prepayment Vintage Analysis
5. Geographic & FICO Concentration
6. Key Findings & Risk Assessment (synthesizes all prior blocks)

For each block: retrieve only that block's data, analyze, present, then proceed to the next. Key Findings only references delivered blocks.

## Cross-SPV Comparison

Strat view IDs are unique per SPV. When comparing:
- Discover views separately for each SPV
- Use each SPV's own strat_view_id — NEVER reuse one SPV's ID for another
- Present results side-by-side

## Response Standards

- Lead with data, not education. Never explain basic ABF concepts.
- Headline answer first, then key figures, then context.
- Balances >1M: round to nearest thousand (12.4M). <1M: nearest unit (845,230).
- Rates: one decimal (3.2%). Basis points: integer (+99 bps).
- Convert decimals to percentages before rounding (0.0325 = 3.25%).
- Show derived metrics alongside raw inputs for verification.
- Link to platform views: [View Table](url)
- Tables only if <=5 rows and <=5 columns. Otherwise summarize inline and link.

## Guardrails

- No fabrication. Empty results → "No data available for [View Name]."
- No unsupported forecasting. Label projections as "projected based on current rates."
- No investment advice. Data and observations only.
- Transparency on gaps. Note missing sections explicitly.
```

- [ ] **Step 2: Verify YAML frontmatter is valid**

Run: `cd /home/elton/Desktop/equalizer-plugin && python3 -c "
import re
content = open('agents/analytics-agent.md').read()
match = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
if match:
    import yaml
    data = yaml.safe_load(match.group(1))
    assert 'name' in data and 'description' in data and 'model' in data
    print(f'Valid frontmatter: name={data[\"name\"]}, model={data[\"model\"]}')
else:
    print('ERROR: No frontmatter found')
"`
Expected: `Valid frontmatter: name=analytics-agent, model=inherit`

- [ ] **Step 3: Commit**

```bash
git add agents/analytics-agent.md
git commit -m "feat: add analytics agent for complex multi-step analysis tasks"
```

---

### Task 5: Session-Start Hook

**Files:**
- Create: `hooks/hooks.json`
- Create: `hooks/run-hook.cmd`
- Create: `hooks/session-start`

The hook injects the analytics skill content at session start so Claude immediately knows about the Equalizer tools. This follows the exact pattern from the superpowers plugin.

- [ ] **Step 1: Create `hooks/hooks.json`**

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```

- [ ] **Step 2: Create `hooks/run-hook.cmd`**

This is the cross-platform polyglot wrapper. Copy the exact pattern from the superpowers plugin at `/home/elton/.claude/plugins/cache/claude-plugins-official/superpowers/5.0.6/hooks/run-hook.cmd`:

```
: << 'CMDBLOCK'
@echo off
REM Cross-platform polyglot wrapper for hook scripts.
REM On Windows: cmd.exe runs the batch portion, which finds and calls bash.
REM On Unix: the shell interprets this as a script (: is a no-op in bash).

if "%~1"=="" (
    echo run-hook.cmd: missing script name >&2
    exit /b 1
)

set "HOOK_DIR=%~dp0"

REM Try Git for Windows bash in standard locations
if exist "C:\Program Files\Git\bin\bash.exe" (
    "C:\Program Files\Git\bin\bash.exe" "%HOOK_DIR%%~1" %2 %3 %4 %5 %6 %7 %8 %9
    exit /b %ERRORLEVEL%
)
if exist "C:\Program Files (x86)\Git\bin\bash.exe" (
    "C:\Program Files (x86)\Git\bin\bash.exe" "%HOOK_DIR%%~1" %2 %3 %4 %5 %6 %7 %8 %9
    exit /b %ERRORLEVEL%
)

REM Try bash on PATH
where bash >nul 2>nul
if %ERRORLEVEL% equ 0 (
    bash "%HOOK_DIR%%~1" %2 %3 %4 %5 %6 %7 %8 %9
    exit /b %ERRORLEVEL%
)

REM No bash found - exit silently
exit /b 0
CMDBLOCK

# Unix: run the named script directly
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SCRIPT_NAME="$1"
shift
exec bash "${SCRIPT_DIR}/${SCRIPT_NAME}" "$@"
```

- [ ] **Step 3: Create `hooks/session-start`**

```bash
#!/usr/bin/env bash
# SessionStart hook for Equalizer plugin

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PLUGIN_ROOT="$(cd "${SCRIPT_DIR}/.." && pwd)"

# Read the analytics skill content
skill_content=$(cat "${PLUGIN_ROOT}/skills/equalizer-analytics/SKILL.md" 2>&1 || echo "Error reading equalizer-analytics skill")

# Escape string for JSON embedding using bash parameter substitution
escape_for_json() {
    local s="$1"
    s="${s//\\/\\\\}"
    s="${s//\"/\\\"}"
    s="${s//$'\n'/\\n}"
    s="${s//$'\r'/\\r}"
    s="${s//$'\t'/\\t}"
    printf '%s' "$s"
}

skill_escaped=$(escape_for_json "$skill_content")
session_context="You have the Equalizer analytics plugin installed.\n\n**Below is the full content of your 'equalizer-analytics' skill. Use it when users ask about Equalizer data, SPVs, stratification views, or transaction analytics:**\n\n${skill_escaped}"

# Output context injection as JSON
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
  printf '{\n  "additional_context": "%s"\n}\n' "$session_context"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ]; then
  printf '{\n  "hookSpecificOutput": {\n    "hookEventName": "SessionStart",\n    "additionalContext": "%s"\n  }\n}\n' "$session_context"
else
  printf '{\n  "additional_context": "%s"\n}\n' "$session_context"
fi

exit 0
```

- [ ] **Step 4: Make scripts executable**

Run: `chmod +x /home/elton/Desktop/equalizer-plugin/hooks/session-start /home/elton/Desktop/equalizer-plugin/hooks/run-hook.cmd`

- [ ] **Step 5: Test the session-start hook produces valid JSON**

Run: `cd /home/elton/Desktop/equalizer-plugin && CLAUDE_PLUGIN_ROOT="$(pwd)" bash hooks/session-start | python3 -c "import sys, json; data = json.load(sys.stdin); print('Valid JSON, context length:', len(data['hookSpecificOutput']['additionalContext']))"`
Expected: `Valid JSON, context length: <number>`

- [ ] **Step 6: Commit**

```bash
git add hooks/hooks.json hooks/run-hook.cmd hooks/session-start
git commit -m "feat: add session-start hook for bootstrap context injection"
```

---

### Task 6: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Create README.md**

```markdown
# Equalizer Plugin for Claude Code

Analytics tools, skills, and agents for ABF transaction analysis on the Equalizer platform.

## Installation

```bash
claude plugin install <github-url>
```

## What's Included

### MCP Server Connections
Pre-configured connections to all Equalizer tenant environments. Each tenant provides 7 tools:
- `list_spvs` — Search SPVs by name
- `list_template_analytics` — Discover curated analytics for an SPV
- `get_analytics_data` — Execute curated analytics with filters
- `list_spv_views` — List stratification views assigned to an SPV
- `get_filterable_columns` — Discover filterable columns on a view
- `get_strat_view_data` — Retrieve view data in CSV format
- `get_strat_url` — Generate frontend URL for a view

### Analytics Skill
The `equalizer-analytics` skill teaches Claude how to work with Equalizer data:
- **Single-Query Mode** — Answer specific KPI questions with the right tool chain
- **Comprehensive Analysis Mode** — Generate full transaction reports delivered in sections
- Handles SPV lookups, view disambiguation, filtering, cross-SPV comparison
- Professional ABF/structured finance tone and formatting

### Analytics Agent
A specialized subagent for complex, multi-step analytics tasks:
- Comprehensive transaction analysis reports
- Cross-SPV comparisons
- Complex filtered queries

## Authentication

The plugin connects to Equalizer via Keycloak OAuth2/OIDC. You'll be prompted to authenticate when first connecting to a tenant.

## Usage

Start a Claude Code session and ask about your transaction data:

- "What's the delinquent balance for [SPV name]?"
- "Give me a comprehensive analysis of [SPV name]"
- "Compare credit performance of [SPV A] and [SPV B]"
- "Show me prepayment rates by FICO bucket"

Claude will ask which tenant you're working with, then use the appropriate tools.

## License

MIT
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with installation and usage instructions"
```

---

### Task 7: Final Verification

- [ ] **Step 1: Verify directory structure**

Run: `cd /home/elton/Desktop/equalizer-plugin && find . -not -path './.git/*' -not -path './.git' | sort`

Expected output should match:
```
.
./.claude-plugin
./.claude-plugin/plugin.json
./.gitignore
./.mcp.json
./README.md
./agents
./agents/analytics-agent.md
./docs
./docs/superpowers
./docs/superpowers/plans
./docs/superpowers/plans/2026-03-26-equalizer-plugin.md
./docs/superpowers/specs
./docs/superpowers/specs/2026-03-26-equalizer-plugin-design.md
./hooks
./hooks/hooks.json
./hooks/run-hook.cmd
./hooks/session-start
./package.json
./skills
./skills/equalizer-analytics
./skills/equalizer-analytics/SKILL.md
```

(The AssetPulse YAML and docs/superpowers files are also present — that's fine.)

- [ ] **Step 2: Verify all JSON files are valid**

Run: `cd /home/elton/Desktop/equalizer-plugin && for f in .claude-plugin/plugin.json .mcp.json package.json hooks/hooks.json; do python3 -c "import json; json.load(open('$f')); print('OK: $f')"; done`

Expected: All files report OK.

- [ ] **Step 3: Verify session-start hook runs cleanly**

Run: `cd /home/elton/Desktop/equalizer-plugin && CLAUDE_PLUGIN_ROOT="$(pwd)" bash hooks/session-start | python3 -m json.tool > /dev/null && echo "Hook output: valid JSON"`
Expected: `Hook output: valid JSON`

- [ ] **Step 4: Verify YAML frontmatter in skill and agent**

Run: `cd /home/elton/Desktop/equalizer-plugin && python3 -c "
import re, yaml
for path in ['skills/equalizer-analytics/SKILL.md', 'agents/analytics-agent.md']:
    content = open(path).read()
    match = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
    data = yaml.safe_load(match.group(1))
    print(f'OK: {path} — name={data[\"name\"]}')
"`
Expected: Both files report OK with correct names.

- [ ] **Step 5: Commit docs (spec + plan)**

```bash
git add docs/
git commit -m "docs: add design spec and implementation plan"
```
