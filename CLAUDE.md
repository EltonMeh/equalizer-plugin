# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Plugin Structure

This is a Claude Code plugin for the Equalizer ABF analytics platform. Key directories:

- `.claude-plugin/` — plugin metadata (`plugin.json`, `marketplace.json`)
- `skills/` — skill definitions (`equalizer-analytics/`, `spv-discovery/`)
- `agents/` — agent definitions (`analytics-agent.md`)
- `hooks/` — hook scripts and configuration (`hooks.json`, `session-start`, `run-hook.cmd`)
- `.mcp.json` — MCP server registrations (staging + local)

## MCP Environments

Two environments are configured in `.mcp.json`:

| Server | URL |
|---|---|
| `equalizer-staging` | `https://abf-backend.staging.cardoaiapps.com/mcp/` |
| `equalizer-local` | `http://localhost:8000/mcp/` |

The active environment is controlled by `enabledMcpjsonServers` in `.claude/settings.local.json`. Staging is the default.

**To test locally:** start the backend on port 8000, then set `"enabledMcpjsonServers": ["equalizer-local"]` in `.claude/settings.local.json`.

## Agent Model

`analytics-agent` is pinned to `claude-sonnet-4-6` — do not change this to `inherit` without understanding the client impact.

## Critical Constraint

`strat_view_id` is unique per SPV. Never reuse a view ID from one SPV when querying another. Cross-SPV analysis requires discovering views separately for each SPV.

## Branch & PR Conventions

- Use `feat/`, `fix/`, `chore/` prefixes for branches
- Open a PR to `main` — no direct commits to main
