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
1. If the user has not identified a tenant, ask which tenant they are working with. If the tenant is already clear from context, confirm it briefly and proceed.
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
| Loss Vintage Analysis | default, loss, charge-off, write-off |
| Prepayment Vintage Analysis | prepayment, CPR, SMM, early repayment |
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
- **SMM:** `1 - (1 - CumPrepay)^(1/Term)`
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
