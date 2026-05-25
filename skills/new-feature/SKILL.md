---
name: new-feature
description: Intake a new feature idea through the PLANNER decision process. Use this when the user has a new idea, feature request, or wants to evaluate whether something should be built.
tier: premium
---

# New Feature Intake (PLANNER)

## Model

- **Preferred:** `claude-opus-4.7`
- **Cost-tier fallback:** `claude-sonnet-4.6` + `--effort xhigh` — see `/fallback-mode`
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

Process a new feature idea through the standard decision framework.

## Step 1: Capture the idea

Ask the user for:
- What they want to build
- Why it matters now
- Which product/repo it affects
- Whether it is customer-facing or internal-only
- Primary launch-platform exception, if any (customer-facing work defaults to iOS-first)

## Step 2: Evaluate build/defer/kill

Apply these criteria:
- **Build now** if: directly advances current sprint goals, has clear acceptance criteria, small scope
- **Defer** if: nice-to-have, large scope, blocked by dependencies
- **Kill** if: off-brand, duplicates existing work, no clear user benefit

## Step 3: Produce handoff packet

Use the exact schema from `agents/handoff-template.md`:

```
PROJECT:
REPO:
PLATFORM:
GITHUB_ISSUE: {YOUR_REPO}#___
GOAL:
SMALLEST_SHIPPABLE_SCOPE:
OUT_OF_SCOPE:
CONSTRAINTS:
ACCEPTANCE_CRITERIA:
TEST_EXPECTATIONS:
RISKS:
NEEDS_FOUNDER_APPROVAL:
```

The handoff packet must be followed by a sources block:

```
## Sources
- [Title](URL) — what constraint/capability/platform claim this anchors
(or: - NONE if the feature has no external platform dependencies)
```

For customer-facing work, set `PLATFORM` to `Apple/iOS-first` unless a
founder-approved exception is explicit in the packet. Internal-only work may
record the approved non-iOS exception directly in `PLATFORM` or `CONSTRAINTS`.

## Step 4: Create tracking issues from spec

After the spec file is saved to `specs/`, run the spec-to-issue helper to create GitHub Issues automatically:

```bash
python3 scripts/spec-to-issues/spec_to_issues.py specs/<spec-filename>.md
```

Use `--dry-run` first to verify what will be created:

```bash
python3 scripts/spec-to-issues/spec_to_issues.py specs/<spec-filename>.md --dry-run
```

The script creates one issue per item in the spec's `## 10. Issue Breakdown` section, applies product labels from the spec filename, and is idempotent (safe to re-run).

## Step 5: Route to next agent

Based on the idea's characteristics:

1. **Sensitive ideas** (health, privacy, children, finance, legal): Output `⏭️ Run /risk-review first.` and stop.
2. **Narrative/content-heavy ideas** (story, cartoon, books, manual-heavy): Output `⏭️ Switch to @planner` and stop.
3. **All other ideas**: Output `⏭️ Switch to @planner` and stop.

Do NOT write specs, code, or create GitHub Issues — those are PLANNER and BUILDER responsibilities.

## Mission alignment check

Verify the feature aligns with your mission and product principles.
- For kids products: keep the experience safe, clear, and age-appropriate
- For adult products: prefer calm, structured, low-friction workflows
- Brand palette and tone should come from your current brand spec
