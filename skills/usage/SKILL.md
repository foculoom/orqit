---
name: usage
description: Check current GitHub Copilot premium request burn. Queries billing API, outputs stable JSON with machine-readable exit codes for CONDUCTOR cost-guard gating.
tier: basic
---

# Usage Check

## Model

- **Preferred:** `claude-haiku-4.5`
- **Premium-exhausted fallback:** `claude-haiku-4.5` (already cheap — no fallback needed)
- **Source of truth:** Model Routing Matrix in your plugin's `dev-session/SKILL.md`

Mechanical skill — occasional API call only. No judgment required; haiku is appropriate.

## When to invoke

Run at session start when CONDUCTOR cost-guard triggers, or manually via `/usage` to check current Opus burn before spawning REVIEWER/PLANNER subagents.

```bash
scripts/check-copilot-usage.sh --model opus --threshold 70
```

## Prerequisites

**Environment variables:**

| Variable | Required | Description |
|---|---|---|
| `COPILOT_PLAN_LIMIT` | **Required** | Monthly request limit (integer). Without this, status is `unknown` and exit is 0. |
| `COPILOT_PLAN_TIER` | Optional | Display label (e.g., `"Business"`). Defaults to `"unknown"` if unset. |
| `COPILOT_USAGE_SCOPE` | Optional | `"user"` (default) or `"org"` |
| `COPILOT_USAGE_ORG` | Conditional | Org slug — required when `COPILOT_USAGE_SCOPE=org` |

**Tools required:**
- `jq` — JSON parsing (`brew install jq`)
- `gh` CLI — authenticated (`gh auth login`)
- `awk` — float arithmetic (standard on macOS/Linux)

## Org-managed setup

Your Copilot plan may be org-managed. The **user endpoint returns 404** for org-managed plans — this is expected GitHub API behavior.

To get real usage data for `{your-github-org}` org:

1. Set `COPILOT_USAGE_SCOPE=org`
2. Set `COPILOT_USAGE_ORG={your-github-org}`
3. Use a PAT with **org Administration: read** permission (user-scope PAT with Plan: read is insufficient for org endpoints)

```bash
export COPILOT_USAGE_SCOPE=org
export COPILOT_USAGE_ORG={your-github-org}
export COPILOT_PLAN_LIMIT=300
export COPILOT_PLAN_TIER=Business
scripts/check-copilot-usage.sh --model opus --threshold 70
```

If you see a 403/404, the script prints a detailed diagnostic to stderr with the required token permissions.

## Usage

### Environment variable table

| Variable | Default | Notes |
|---|---|---|
| `COPILOT_PLAN_LIMIT` | _(unset)_ | Monthly cap; without it, burn % is unknown |
| `COPILOT_PLAN_TIER` | `"unknown"` | Label only — not inferred from API |
| `COPILOT_USAGE_SCOPE` | `"user"` | `user` or `org` |
| `COPILOT_USAGE_ORG` | _(unset)_ | Required if scope=org |

### Flag reference

| Flag | Default | Description |
|---|---|---|
| `--model <name>` | `all` | Filter `usageItems` by case-insensitive substring match on model field. E.g., `--model opus` matches any model containing "opus". |
| `--threshold <pct>` | `70` | Integer 0-100. Exit 2 when `pct_burn >= threshold`. |
| `--format json\|text` | `json` | Output format. |

### Exit code table

| Code | Meaning | CONDUCTOR action |
|---|---|---|
| `0` | `ok` or `unknown` (limit unset) | No action — proceed with default models |
| `1` | `error` | API call failed — check stderr for diagnostic |
| `2` | `warning` — burn ≥ threshold | Drop REVIEWER to `claude-sonnet-4.6 --effort xhigh` |
| `3` | `over_quota` — burn ≥ 100% | Block Opus entirely |

## Example output

**JSON (default):**
```json
{
  "scope": "org",
  "model_filter": "opus",
  "plan_tier": "Business",
  "limit": 300,
  "used": 42,
  "pct_burn": 14.0,
  "remaining": 258,
  "threshold": 70,
  "status": "ok"
}
```

**Text (`--format text`):**
```
Copilot Usage (scope: org, filter: opus)
Plan tier: Business | Limit: 300 req/mo
Used: 42 | Remaining: 258 | Burn: 14.0%
Status: OK (threshold: 70%)
```

**When `COPILOT_PLAN_LIMIT` is unset (exits 0):**
```json
{
  "scope": "user",
  "model_filter": "all",
  "plan_tier": "unknown",
  "limit": "unknown",
  "used": 0,
  "pct_burn": null,
  "remaining": null,
  "threshold": 70,
  "status": "unknown"
}
```
_(stderr: `Warning: COPILOT_PLAN_LIMIT is not set. Cannot compute burn percentage.`)_

## CONDUCTOR integration

CONDUCTOR invokes `scripts/check-copilot-usage.sh --model opus --threshold 70` and gates on exit code:

- **Exit 0** — ok/unknown: proceed with default model routing
- **Exit 2** — warning (≥70% Opus burn): drop REVIEWER to `claude-sonnet-4.6 --effort xhigh`
- **Exit 3** — over quota (≥100% burn): block Opus entirely

See `.github/agents/conductor.agent.md` § Cost guard for the full gating rule, including the ≥3 REVIEWER passes trigger and fallback-mode override.

## Script reference

Full implementation: `scripts/check-copilot-usage.sh`

The script:
- Resolves username via `gh api user --jq '.login'` (never hardcoded)
- Aggregates `usageItems[].grossQuantity` (NOT `netQuantity` — stays 0 inside included quota)
- Warns on empty/missing `usageItems`
- Prints actionable 403/404 diagnostic to stderr
- v1 scope: no TTL cache, no auto org-detect, no plan-tier inference from API
