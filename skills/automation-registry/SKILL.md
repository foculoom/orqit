---
name: automation-registry
description: Living registry of platform operations that are known-automatable (API/CLI with session-store receipts) vs. truly manual (with evidence of why). Consult before labeling any step "manual" or "founder web-UI" in a plan, spec, or handoff.
---

# Automation Registry

A living registry of platform operations with evidence of whether they are API/CLI-automatable or genuinely manual. Consult this before labeling any step "manual" in a plan, spec, or handoff.

## Rule

Before labeling a step "manual," cite one of:
- A receipt row in this registry showing the operation was successfully automated, **or**
- Official platform docs that explicitly state the operation has no public API, **or**
- A prior failure with the exact error surfaced when attempting automation.

If no evidence exists, draft the step as an API attempt with a run-time fallback to manual — **never** as manual upfront.

## Registry

| Platform | Operation | Status | Receipt | Notes |
|----------|-----------|--------|---------|-------|
| GitHub | Create repo | Automatable | `gh repo create` | Supports --public/--private/--org flags |
| GitHub | Open PR | Automatable | `gh pr create` | Full field support |
| GitHub | Close PR | Automatable | `gh pr close` | Supports --comment |
| GitHub | Create/close issue | Automatable | `gh issue create` / `gh issue close` | |
| GitHub | Add label | Automatable | `gh label create` / `gh issue edit --add-label` | |
| GitHub | Trigger workflow | Automatable | `gh workflow run` | Requires workflow_dispatch trigger |

_Add rows as you automate new operations. Format: Platform → Operation → Automatable/Manual → Receipt (command, session ID, or doc URL) → Notes._

## Adding a new row

When you successfully automate a new operation:
1. Add a row to the table above with the receipt.
2. Commit the update to keep the registry current.

When you discover a genuinely manual operation:
1. Add a row with Status: Manual and cite the official doc or failure evidence.
2. Revisit on next platform API release cycle.
