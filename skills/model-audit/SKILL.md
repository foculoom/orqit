---
name: model-audit
description: Weekly refresh trigger for the Model Routing Matrix. Compares current Copilot CLI version to the Matrix's "Last verified" pin and produces a draft refresh issue when they diverge. Founder reviews the draft and merges matrix updates; this skill never auto-creates issues or edits the matrix.
tier: basic
---

# Model Audit

## Model

- **Preferred:** `claude-haiku-4.5`
- **Cost-tier fallback:** `claude-haiku-4.5` — see `/fallback-mode`
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

## Cadence

Run every **7 days** or immediately after observing any of:

- `copilot update` succeeded with a minor- **or patch-version** bump (CLI roster has shifted within a single patch in the past, e.g. `claude-sonnet-4` removal at v1.0.36)
- A new model ID appears in the `task` tool's `model` enum that isn't in the Matrix
- A model ID in the Matrix disappears (deprecation)
- A PLANNER session is about to pin a **non-default model** in a spec or agent profile — re-verify the pin is still valid before the spec lands
- A [GitHub Copilot changelog](https://github.blog/changelog/) entry mentions any of: `model`, `auto`, `deprecat`, `pricing`
- **GitHub announces a change to the Auto model pool** (Auto's pool can change without a CLI version bump — Auto-pool drift is **decoupled from CLI version** and requires its own weekly changelog scrape regardless of `copilot --version` output)
- **Any commit to `github/docs:data/tables/copilot/*.yml`** — these YAML files are the authoritative source for `cli: true/false` flags, pool membership, and retirement dates, and update independently of CLI version
- **`gpt-5.5` promo multiplier re-verification** — the 7.5× promo rate is subject to change per GitHub's pricing footnote; re-verify on every audit while gpt-5.5 holds a Matrix Preferred slot

## Steps

### 1. Capture current CLI version

```bash
copilot --version
# → "GitHub Copilot CLI 1.0.32." (example)
```

### 2. Read the Matrix's pinned version

```bash
grep -n 'Last verified' .github/skills/dev-session/SKILL.md
```

Extract the `vX.Y.Z` value.

### 3. Compare

- If they match → **no-op**. Print: `✅ Matrix verified against current CLI vX.Y.Z. No refresh needed.`
- If they differ → proceed to step 4.

### 4. Inspect the task tool's model enum (founder-reviewed)

In any Copilot CLI agent session, the `task` tool's `model` parameter enumerates the full available roster in its JSON schema. Have the founder open a session and confirm:

- Are all Matrix model IDs still present?
- Are there new IDs not yet in the Matrix?

### 4b. Audit the Auto model pool

Auto's pool is maintained by GitHub and can change without a CLI version bump. Check:

1. Visit the [GitHub Copilot changelog](https://github.blog/changelog/) and search "auto model selection" for any posts since the last audit date.
2. Compare listed pool models against the note in `.github/skills/dev-session/SKILL.md` → "How to use" → Auto bullet.
3. If the pool changed, add it to the draft in Step 5:
   - New pool models to add to the note
   - Removed pool models to strike from the note

### 5. Produce a draft refresh issue (dry-run — DO NOT create)

Write the draft to `tmp/model-audit-draft.md` (or print to terminal). **Do not invoke `gh issue create`** — founder reviews first.

Draft template:

```markdown
**Title:** model-audit: refresh Model Routing Matrix for CLI vX.Y.Z

**Trigger:** `copilot --version` reports vX.Y.Z; Matrix last verified against vA.B.C on YYYY-MM-DD.

**Proposed changes:**
- [ ] Bump "Last verified" line in `dev-session/SKILL.md` to vX.Y.Z / <today>
- [ ] Add rows for any new model IDs:
  - `<id>` (multiplier: ?, tier: ?)
- [ ] Remove/deprecate rows for any missing model IDs:
  - `<id>`
- [ ] Re-verify preferred/fallback selections for each skill if multipliers shifted

**Closes:** (assign after creation)
```

### 6. Hand to founder

Report: `⏭️ Draft produced at tmp/model-audit-draft.md. Founder: review and create via gh issue create, then route to @builder.`

### 7. Validate tier field presence and consistency

The Model Routing Matrix declares a canonical **Tier Taxonomy** (basic/standard/premium/extra-premium + two-pass). Every agent profile and skill must declare a `tier:` value that maps to a Preferred model consistent with the taxonomy.

**Declaration form (split by file type — Copilot CLI rejects unknown front-matter fields on `.agent.md` files):**

- `.github/skills/*/SKILL.md` — declare in YAML front-matter as `tier: <value>`.
- `.github/agents/*.agent.md` — declare as an HTML comment immediately after the front-matter close: `<!-- tier: <value> -->`. Front-matter on these files must contain only CLI-recognized fields (`name`, `description`, `model`).

Run these checks as part of every audit:

```bash
# Every SKILL.md must have a tier: front-matter line
missing_skills=$(grep -L "^tier:" .github/skills/*/SKILL.md || true)
# Every .agent.md must have an HTML-comment tier marker
missing_agents=$(grep -L "^<!-- tier:" .github/agents/*.agent.md || true)
missing="${missing_skills}${missing_agents:+$'\n'}${missing_agents}"
if [[ -n "${missing// /}" ]]; then
  echo "❌ Files missing tier marker:"
  echo "$missing"
  exit 1
fi

# Tier values must be one of: basic, standard, premium, extra-premium, two-pass
allowed_skill='^tier: (basic|standard|premium|extra-premium|two-pass)$'
allowed_agent='^<!-- tier: (basic|standard|premium|extra-premium) -->$'
invalid_skill=$(grep -hE "^tier:" .github/skills/*/SKILL.md | grep -vE "$allowed_skill" || true)
invalid_agent=$(grep -hE "^<!-- tier:" .github/agents/*.agent.md | grep -vE "$allowed_agent" || true)
if [[ -n "$invalid_skill$invalid_agent" ]]; then
  echo "❌ Invalid tier values found:"
  echo "$invalid_skill"
  echo "$invalid_agent"
  exit 1
fi
```

For each skill, spot-check that its declared `tier:` value matches its **Preferred** model line:

| Declared tier | Expected Preferred model line |
|---|---|
| `basic` | `claude-haiku-4.5` |
| `standard` | `claude-sonnet-4.6` |
| `premium` | `claude-opus-4.7` |
| `extra-premium` | `gpt-5.5` |
| `two-pass` | two model lines (e.g., Sonnet 4.6 + Opus 4.7) |

Drift here goes into the Step 5 draft as a proposed correction.

## Non-goals

- **Never** auto-create the issue (founder review required).
- **Never** auto-edit the Matrix (that's a separate BUILDER task once founder approves).
- **Never** parse undocumented cache paths (`~/Library/Caches/copilot/...`) — fragile and version-unstable.

## When NOT to run

- Mid-session during critical work (skill is low-priority).
- Within **24h** of the last successful audit **AND** `copilot --version` is unchanged since that run. A CLI version change always re-triggers the audit regardless of recency.
