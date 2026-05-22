---
name: status
description: Unified pipeline audit and founder digest. Quick mode scans open issues and recommends next action. Full mode produces a cross-repo daily brief with stale PRs, failing CI, and priority routing.
tier: basic
---

# Status

## Model

- **Preferred:** `claude-haiku-4.5` for Quick mode (JSON scan + label classification, mechanical); `claude-sonnet-4.6` for Full mode (cross-repo daily brief synthesis)
- **Premium-exhausted fallback:** `claude-haiku-4.5` for both modes — see `/fallback-mode`
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

Pipeline audit and founder digest in one skill. Runs in Quick mode by default; use Full mode for a cross-repo daily brief.

## Quick Mode (Pipeline Check)

Default when the user runs `/status`.

### Step 1: Get open issues

Use the `pipeline_status` tool when available. It already summarizes open issues, PR state, and suggested next action.

If the tool is unavailable, fall back to:

```bash
gh issue list --repo {your-github-org}/{your-main-repo} --state open --json number,title,labels,assignees --limit 50
```

### Step 2: Determine pipeline state for each issue

Check in order:

1. **Labels** — Read the `labels` field from each issue.
2. **Has PR?** — Check for linked PRs using the full cross-repo issue reference.
    ```bash
    gh pr list --repo {your-github-org}/<target-repo> --search "{your-main-repo}#<N>" --json number,title,state
    ```
    Use `pipeline_status` output when available. If the target repo is unclear, infer it from the issue body, spec, labels, or existing branch/PR context.
3. **PR state** — Merged, Open, Closed, or None.

### Step 3: Classify each issue

Evaluate states in this priority order (first match wins):

| State | Meaning | Next Agent |
|-------|---------|------------|
| Has `validated` label | QA passed, ready to close | Close issue |
| PR merged, issue still open | Needs QA validation | @builder → validate → close |
| PR open | In review | Founder review |
| Has `idea` label, no PR | Needs triage | @planner |
| Not in active focus set (bucket check) | Strategically next — not yet | ⚠️ @founder: strategy-fit gate |
| No PR (default) | Ready for implementation | @builder |

**"Ready for implementation" vs "strategically next":** An issue with all readiness signals (merged prerequisites, open `@builder` label, no PR) is only **ready for implementation** when it is also in the active focus set of the governing strategy issue. If a ratified strategy issue is active (e.g., {your-ratified-strategy-issue}), check the issue's bucket before recommending `@builder`. Issues in `deferred-*`, provisional, or product-coupled buckets are **strategically next** — surface them to the founder with a strategy-fit gate prompt rather than routing to `@builder`. See `docs/policy/strategy-fit-gate.md`.

### Step 4: Output status table

```
#  Title                              Labels       PR       Next Action
-- ---------------------------------- ------------ -------  -------------------------
90 Create prompt file library          idea         None     @planner
91 Create skills library                            None     @builder
92 Validate game build                 validated    Merged   Close issue
```

### Step 5: Recommend next action

Suggest the single highest-priority item the founder should work on, with the exact agent to use.

Priority order: P0 (security) > P1 (high impact) > P2 (medium) > P3 (lower) > product issues.

---

## Full Mode (Founder Brief)

Run when the user asks for a full brief, daily digest, or cross-repo overview.

### Repos to scan

```bash
REPOS=(
  "{your-github-org}/{your-main-repo}"
  "<companion-repo-1>"   # substitute your companion repos (e.g., owner/my-ios-app)
  "<companion-repo-2>"
  "{your-github-org}/{your-additional-repo}"
)
```

Add repos as needed.

### Step 1: Stale PRs (open > 3 days)

For each repo, find PRs open longer than 3 days:

```bash
for REPO in "${REPOS[@]}"; do
  gh pr list --repo "$REPO" --state open \
    --json number,title,createdAt,url --limit 20
done
```

Flag any PR where `createdAt` is more than 3 days ago. These need founder review or a decision to close.

### Step 2: Stale issues (no activity > 7 days)

For each repo, find open issues with no recent updates:

```bash
for REPO in "${REPOS[@]}"; do
  gh issue list --repo "$REPO" --state open \
    --json number,title,updatedAt,labels --limit 30
done
```

Flag any issue where `updatedAt` is more than 7 days ago. These may be orphaned or blocked.

### Step 3: Failing CI runs

For repos with GitHub Actions workflows, check the most recent run on the default branch:

```bash
for REPO in "${REPOS[@]}"; do
  gh run list --repo "$REPO" --branch master --limit 3 \
    --json databaseId,conclusion,name,url 2>/dev/null
done
```

Flag any run where `conclusion` is `failure`.

### Step 4: Recently shipped companion lanes

Surface recent merged PRs from companion repos for founder awareness. Start with your primary companion repo (substitute `<companion-repo>` with your actual repo, e.g., `owner/my-ios-app`).

```bash
gh pr list --repo <companion-repo> --state merged \
  --json number,title,mergedAt,url --limit 10
```

Include only PRs merged within the last 7 days. Show at most 3–5 items. If none, omit the section or note the lane is quiet.

### Step 5: Open issue summary on {your-main-repo}

Pull the current pipeline state from the tracking repo:

```bash
gh issue list --repo {your-github-org}/{your-main-repo} --state open \
  --json number,title,labels,updatedAt --limit 50
```

Classify each by label priority:
- **P0** (security/critical) — immediate attention
- **P1** (high impact) — this session
- **P2/P3** (medium/lower) — when bandwidth allows
- **No priority label** — needs triage by @planner

### Step 6: Output the Founder Brief

Compile findings into a single digest:

```
# 🗒️ Founder Brief — <today's date>

## 🔴 Needs Immediate Attention
<P0 items, failing CI, stale PRs awaiting your review>

## 🟡 This Session
<P1 items, issues ready for implementation>

## 🟢 On Deck
<P2/P3 items, backlog>

## 🚀 Recently Shipped / Active Lane
<merged PRs in <companion-repo> within the last 7 days>

## 📊 Pipeline Summary
| Repo                   | Open Issues | Open PRs | Stale PRs | Failing CI |
|------------------------|-------------|----------|-----------|------------|
| {your-main-repo}       | N           | N        | N         | N          |
| <companion-repo-1>     | N           | N        | N         | N          |
| <companion-repo-2>     | N           | N        | N         | N          |
| {your-additional-repo} | N           | N        | N         | N          |

## 🎯 Recommended Next Action
<highest-priority single item + which agent to invoke>
```

Keep `🔴`, `🟡`, and `🟢` sections limited to open work only. Use the shipped lane for awareness context, not as a competing priority.

### Routing suggestions

| Situation | Agent |
|-----------|-------|
| Idea needs triage | @planner |
| Issue needs a spec | @planner |
| Issue ready for implementation | @builder |
| PR needs review | Founder review |
| PR merged, needs validation | @builder |
| Stale issue, unclear status | `/status` |

---

## Notes

- Quick mode is the default — use it for routine "what should I work on next?" checks.
- Full mode is for start-of-session awareness or when the founder asks for a daily digest.
- This skill is for awareness only — it does not take action.
- For issues in repos not listed above, add them to the `REPOS` array.
