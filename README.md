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
