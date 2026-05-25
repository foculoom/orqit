---
name: dev-session
description: Unified session lifecycle — start, resume, monitor, and end Copilot CLI sessions. Covers context management, twin sync, issue state, and branch cleanup.
tier: standard
---

# Session Lifecycle

## Model

- **Preferred:** `claude-sonnet-4.6`
- **Cost-tier fallback:** `/model auto` → `claude-sonnet-4.5` — see `/fallback-mode`
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

This skill owns the full session lifecycle referenced by `.github/copilot-instructions.md`. Use `/dev-session` at the start and end of every session.

## Session Start

### Cold-start vs Savepoint-resume path

**Check first:** Does `session/RESUME.md` exist?

```bash
ls session/RESUME.md 2>/dev/null && echo "SAVEPOINT EXISTS" || echo "COLD START"
```

| Path | Condition | Steps |
|---|---|---|
| **Savepoint-resume** | `session/RESUME.md` exists | `view session/RESUME.md` → targeted issue lookup (active issue only) → begin work. **Skip** `skill("dev-session")` and `skill("fallback-mode")` — their operative rules are already in system instructions. |
| **Cold-start** | No RESUME.md | Follow the full Session Start protocol (steps 1–7 below) |

**Skills safe to skip on resume** (operative rules already in system instructions):
- `dev-session` — session lifecycle; rules in custom instructions
- `fallback-mode` — model routing; rules in custom instructions

**Skills that must always be invoked when the task requires them** (procedural detail NOT in system instructions):
- `qa-capture-ios` — exact xcrun/simctl commands
- `brand-asset-pipeline` — exact export pipeline steps
- `qa-validate`, `kids-safety`, `risk-review`, `ship-issue` — when task scope requires them

**Context savings:** Skipping `dev-session` + `fallback-mode` skill invocations saves ~523 lines / ~6–8k tokens per session start.

---

1. **Check repository_memories** — review stored facts in the system prompt. Do NOT re-research topics already covered.

2. **Check if a relevant skill exists** for the current task.
   - Prefer the repo skill catalog in `.github/skills/`
   - Use `/skills list` if you need the runtime list

3. **If resuming prior work**, follow the [Resuming Prior Work](#resuming-prior-work) section below.

4. **Check session store** for recent work:
   ```sql
   SELECT s.id, s.summary, s.updated_at
   FROM sessions s
   WHERE s.repository LIKE '%{YOUR_REPOSITORY_PATTERN}%'
   ORDER BY s.updated_at DESC LIMIT 5;
   ```

4b. **Check recent mistake patterns (last 7 days)**:
   ```sql
   SELECT si.content, si.session_id, si.source_type
   FROM search_index si
   JOIN sessions s ON s.id = si.session_id
   WHERE si.search_index MATCH 'FIVE_WHYS_POSTMORTEM OR "severity gate" OR postmortem'
     AND datetime(s.updated_at) >= datetime('now', '-7 days')
   ORDER BY rank
   LIMIT 5;
   ```
   - If results exist, surface one line per hit before work:
     `⚠️ Known mistake patterns (last 7 days): <root cause> [session: <id>]`
   - If no results, skip silently.

5. **dtwin rule surfacing (session-start, cold-start recommended):** Run `python3 -m dtwin list-rules --status pending`. BUILDER adds this to the session-start block; whether to gate on cold-start vs every-resume is BUILDER's discretion (document the choice inline if changing defaults). Apply threshold:
   - Pending count ≥ 10: surface top-5 titles + rule-ids to founder for approve/reject before proceeding.
   - Pending count 5–9: add a one-line note to the session digest ("N dtwin rules pending — review via `list-rules` when convenient") without blocking.
   - Pending count < 5: no action needed.

   Use `python3 -m dtwin get-candidate-rules` for the legacy view if needed. (Note: `list-memories` is **not** a dtwin subcommand — verified 2026-05-18 against CLI surface.)

   **Advisory drift check:** Run `python3 -m dtwin list-rules --status approved | wc -l` and compare roughly to the count of Automatable rows in `automation-registry/SKILL.md`. If the registry is materially longer, remind founder: recent automations may need a paired `python3 -m dtwin add-rule --approve --rule-type workflow "title" "body"`.

6. **Usage check (optional, at session start):** If `COPILOT_PLAN_LIMIT` is set, run `scripts/check-copilot-usage.sh --model opus --threshold 70` to check premium model burn before dispatching REVIEWER/PLANNER subagents. Exit 2 = drop to standard tier (see `/fallback-mode`); exit 3 = block premium models. For extra-premium (gpt-5.5) sessions, also check `--model gpt-5.5 --threshold 70`. See `.github/skills/usage/SKILL.md`.

7. **Do NOT re-run setup** that was completed in prior sessions:
   - Do not re-clone repos
   - Do not re-install dependencies unless there's evidence they changed
   - Do not re-validate repo structure

## Resuming Prior Work

Follow these steps in order to resume work efficiently.

### Step 1: Load prior context from digital twin

Call `tom-continuationPrompt` with the topic or issue number. This returns a context pack from all prior sessions.

If no topic is provided, call `tom-continuationPrompt` without a topic to get the most recent work context.

### Step 2: Query session store for recent sessions

```sql
SELECT s.id, s.branch, s.summary, s.updated_at
FROM sessions s
WHERE s.repository LIKE '%{YOUR_REPOSITORY_PATTERN}%'
ORDER BY s.updated_at DESC
LIMIT 5;
```

### Step 3: Check GitHub Issue state

If an issue number is known:

```bash
gh issue view <N> --repo {YOUR_REPO} --json state,title,body,labels
```

If no issue is specified, check for open issues:

```bash
gh issue list --repo {YOUR_REPO} --state open --limit 10
```

### Step 4: Determine what's done and what remains

Compare the digital twin context, session store, and issue state. Identify:
- What was completed in prior sessions
- What remains to be done
- Any blockers or dependencies

### Step 5: Begin work from the current state

Do NOT redo completed steps. Start from where the prior session left off.

## During Session

### Context budget: context-health checkpoints

Context % used (from `/context`) is the **primary** lifecycle signal. Turn count is a safety cap only — not the primary estimate. A heavy REVIEWER pass loads ~8 KB; a lightweight status check loads ~0.5 KB. Evaluate context health at fixed checkpoints instead of auto-stopping at a fixed turn count.

> **Cloud-agent scope note:** `/context` is unavailable in cloud-agent sessions (`copilot-instructions.md:263`). Cloud-agent sessions retain turn-based checkpoints as primary signal; %-context rules apply to local-CLI sessions only.

**When to run `/context` (by turn — safety-cap triggers):**

| Turn | Trigger |
|---|---|
| **Turn 5** | Always — compact if context > 50% used (> 40% in fallback) |
| **Turn 10** | Always — apply context threshold table below |
| **Turn 15** | Hard ceiling (unconditional safety cap) — wind down regardless of context % |

**Context threshold table (apply at Turn 10 reading):**

| Context used | Action |
|---|---|
| **< 60%** | Extend wind-down to turn 14; continue dispatching |
| **60–75%** | Compact + extend to turn 12; flag quality risk |
| **> 75%** | Wind-down mandatory regardless of turns remaining |

**Pre-compaction reading is authoritative.** If you compact at turn 10 and context drops from 65% to 30%, the extension decision is still "compact + extend to turn 12" (based on the 65% pre-compaction reading). Post-compaction reading informs quality risk only.

**Missed checkpoints:** if turn 10 passed without a `/context` check, run it immediately at the current turn and apply the strictest applicable rule. If the current turn is ≥ 15, wind down unconditionally.

**Fallback mode** tightens all thresholds by 10 percentage points:

| Context threshold | Normal | Fallback |
|---|---|---|
| Turn 5 compact threshold | > 50% | > 40% |
| Turn 10: extend to turn 14 | < 60% | < 50% |
| Turn 10: compact + extend to turn 12 | 60–75% | 50–65% |
| Turn 10: mandatory wind-down | > 75% | > 65% |
| Turn 15 hard ceiling (safety cap) | always | always |

**Multi-signal stop predicates** — wind-down or fan-out triggers on any of:

| Signal | Threshold | Action |
|---|---|---|
| (a) Token budget hard ceiling | > 75% used (> 65% fallback) | Halt — wind-down mandatory |
| (b) Token budget soft threshold | > 60% used | Compact (per checkpoint table); if no progress after compact → wind-down. Low-progress signal (≤ 1 atomic task remaining or no AC delta since last compact) escalates to wind-down sooner. |
| (c) Wall-clock warning | Approaching session time limit | Wind-down and checkpoint |
| (d) Cost cap | `/usage` exits 2 (≥ 70% premium model burn) or exit 3 (over quota) | Drop one cost tier (see `/fallback-mode`) or block premium model |
| (e) Repetition | ≥ 3 identical tool calls | Wind-down or fan-out to fresh subagent |
| (f) Error rate | > 5% of tool calls failing | Wind-down or fan-out |

See `fallback-mode` skill and `.github/agents/conductor.agent.md` § Subagent spawning rules → Wind-down buffer for CONDUCTOR enforcement of these checkpoints.

### Proactive compaction

Do NOT wait for auto-compaction at 95%. In normal mode, compact proactively around 50–60%. In fallback mode, compact at 40–50%. The checkpoint table above defines when compaction is mandatory.

### Savepoint pattern (preferred over /compact)

A savepoint fully resets context by writing a structured handoff file, running `/clear`, and reloading. This is **4–8× more context-efficient than `/compact`** because `/compact` leaves 20–40k tokens of residual summary whereas a savepoint starts from a clean slate with only the explicitly reloaded content.

#### RESUME.md format

Keep RESUME.md **≤ 1 500 words / ≤ 8 000 tokens**. Use exactly these four named fields (additional fields are allowed but these four are required):

```markdown
# RESUME.md — <issue or task title>

## Completed items
- <bullet per completed atomic task, past-tense, one line each>

## Next items
- <bullet per remaining atomic task in priority order>

## Open blockers
- <bullet per unresolved dependency or blocker; "None" if clear>

## Context-critical facts
- <bullet per non-obvious fact that a fresh context would not know: file paths,
  decisions made, rejected approaches, environment quirks, branch name, PR URL>
```

#### Command sequence

> **Scratch state:** RESUME.md is ephemeral session state — do not stage or commit it. Preferred path is `session/RESUME.md` (already gitignored). A root-level `RESUME.md` is also gitignored as a belt-and-braces guard.

1. **Before `/clear`:** complete any required session-end obligations for this repo (e.g., `tom-syncTwin` if applicable). If obligations cannot be completed before the reset, capture them as bullets in **Context-critical facts** so the fresh context knows to run them.
2. Write `session/RESUME.md`: use `edit` or `create` at path `session/RESUME.md`.
3. Run `/clear` — this resets the context window completely.
4. At the top of the new session, reload: `view session/RESUME.md`.
5. Re-read every file listed under **Context-critical facts** before resuming work.
6. Continue work from **Next items**.

#### Prefer savepoint vs. /compact decision rule

| Condition | Action |
|---|---|
| Remaining work > 2 atomic tasks **or** fresh-context benefit outweighs compaction residual (e.g., complex multi-file change, premium model swap) | **Use savepoint** — write RESUME.md → `/clear` → reload |
| One small remaining task (≤ 1 atomic step) **or** savepoint setup cost > benefit (e.g., mid-turn emergency compaction, context spike from a single large read) | **Use `/compact`** — faster, lower overhead for short remaining work |
| Approaching Turn 15 hard ceiling with multiple tasks left | **Use savepoint** — do not compress into a lossy summary when significant work remains |
| Fallback mode active (cost-tier step-down) | **Prefer savepoint** — standard-tier models degrade more on residual summaries; clean context recovers more quality |

> **Rule of thumb:** if you would need to compact more than once to finish, use savepoint instead.

### Parallel-execution worktree guard

When running parallel BUILDER work in the same repo, each BUILDER MUST run in its own linked worktree. Single-checkout parallel dispatch is FORBIDDEN.

Pre-dispatch checklist:

1. Run `git worktree list`.
2. If the target issue branch already has a linked worktree, reuse it.
3. Otherwise create one:
   `git worktree add ../<repo>-issue-<N> -b feature/issue-<N>-<slug>`
4. Pin the absolute `WORKTREE_PATH` in the spawn prompt.
5. If an absolute `WORKTREE_PATH` cannot be determined, stop and do not dispatch.

Anchor-validation clause: before inserting this guard, verify `### Proactive compaction` and `### Wind-down protocol` anchors exist. If future restructuring changes those headings, place this guard in the equivalent `## During Session` lifecycle section and do not duplicate it.

### Wind-down protocol (context-health triggers)

Wind-down begins when any of these conditions is met (using pre-compaction context reading):
- Turn 10 check: context > 75% used (> 65% in fallback mode)
- Turn 15: hard ceiling (safety cap) — wind down unconditionally regardless of context %

When wind-down triggers, **stop accepting new work** and begin the wind-down sequence:

1. **Stop dispatching** — do not spawn new subagents or start new tasks.
2. **Let in-flight work finish** — wait for any background subagents to reach a safe checkpoint.
3. **Store memories** — call `store_memory` for every significant discovery this session.
4. **Update plan.md / issue state** — capture remaining work and blockers in the GitHub issue.
5. **Return to default branch** — follow the § Session End checklist below.
6. **Checkpoint** — call `tom-syncTwin` to persist context for the next session.

**CONDUCTOR ownership:** when CONDUCTOR is driving, it owns this sequence. CONDUCTOR must surface `⚠️ WIND-DOWN: context-health checkpoint triggered — beginning end-of-session sequence` (appending the reason, e.g., `turn 10 >75%` or `turn 15 hard ceiling`) before wind-down starts, so the founder can redirect if needed. Do NOT start a new issue during wind-down.

### Model selection

Use the Model Routing Matrix below as the single source of truth for picking a model per skill or agent. It supersedes the prior bullet list.

<!-- Last verified against Copilot CLI v1.0.54 on 2026-05-24 -->
**Last verified:** Copilot CLI **v1.0.54** on 2026-05-24 (issue #1382; pin bump only; CONDUCTOR tier moved from extra-premium to standard per PR #1383 (2026-05-24); claude-opus-4.6 and claude-opus-4.5 now exposed as pin-able task enum IDs (were preview/not exposed at v1.0.51 — see #1231); gpt-5.2/gpt-5.2-codex/gpt-4.1 still in enum pre-June-1 deprecation (confirm removal post-June-1); GPT-5.5 multiplier confirmed 7.5× — re-verify post-June-1; all other routing unchanged). Refresh via the `model-audit` skill (#464) every 7 days (or after any `copilot update` bump, or on any GitHub Copilot changelog entry mentioning model/auto/deprecat/pricing) — see #928 for rationale. Matrix updated with `5-whys` row; no new CLI model-enum verification claimed in this edit.

#### Tier Taxonomy (canonical)

Every agent profile and skill SKILL.md declares its tier as follows: skill `SKILL.md` files use a `tier:` YAML front-matter field; agent `.agent.md` files use an HTML-comment marker `<!-- tier: <value> -->` immediately after the front-matter close (the Copilot CLI custom-agent loader rejects unknown front-matter fields, so `tier:` cannot live in `.agent.md` front-matter; a prior loader regression established this split declaration form). The Matrix's "Preferred Model" column is the authoritative routing; the tier label is a declarative summary for cost-guard gating, allowlisting, and audit. Two-pass skills (`reviewer-qa-gate`, `qa-validate`, `brand-compliance`) declare `tier: two-pass` and use multiple models per the Matrix row.

| Tier | Label | Default model | Multiplier | Used by |
|---|---|---|---|---|
| 1 | **basic** | `claude-haiku-4.5` | 0.33× | usage, model-audit, automation-registry, llc-ops, qa-capture-ios, 5-whys, release-asset-fanout, a11y-godot (OUTLINE), status (Quick mode default) |
| 2 | **standard** | `claude-sonnet-4.6` | 1× | BUILDER, CONDUCTOR, dev-session, ship-issue, brand-asset-pipeline, release-post, fallback-mode, kids-safety, status (Full mode) |
| 3 | **premium** | `claude-opus-4.7` | 15× | PLANNER, REVIEWER, new-feature, risk-review |
| 4 | **extra-premium** | `gpt-5.5` | 7.5× | market-scan |
| — | **two-pass** | per Matrix row | mixed | reviewer-qa-gate (Sonnet+Opus), qa-validate (Sonnet+Opus), brand-compliance (Sonnet+Opus) |

> **Cost-guard mapping** (`scripts/check-copilot-usage.sh --tier <label>`): basic → `*haiku*`, standard → `*sonnet*`, premium → `*opus*`, extra-premium → `gpt-5.5`. Backward-compatible with `--model <glob>`.

#### Model Routing Matrix

| Skill / Agent | Preferred Model | Reason | Cost-Tier Fallback |
|---|---|---|---|
| **agent: planner** | `claude-opus-4.7` | Strategy, spec writing, content judgment | premium → standard: `claude-sonnet-4.6` + `--effort xhigh` + mandatory rubber-duck |
| **agent: builder** | `claude-sonnet-4.6` | Implementation and build/test at standard tier | standard → basic: `/model claude-haiku-4.5` (no effort flag); or `/model auto` → `claude-sonnet-4.5` |
| **agent: reviewer** | `claude-opus-4.7` | Visual/business judgment, deep audits | premium → standard: `claude-sonnet-4.6` + `--effort xhigh` + rubber-duck before every quality verdict |
| **agent: conductor** | `claude-sonnet-4.6` | Orchestration, gate reasoning, subagent dispatch — deliberately pinned to standard tier by founder 2026-05-24 (prev: gpt-5.5) | standard → basic: `claude-haiku-4.5` (no effort flag); REVIEWER/PLANNER subagents: premium → standard per fallback-mode SKILL |
| `a11y-godot` | `claude-haiku-4.5` (while OUTLINE per #947); upgrade to `claude-sonnet-4.6` post-spike | Placeholder checklist; minimal judgment required pending SPIKE-A11Y closure | `claude-haiku-4.5` (while OUTLINE); `/model auto` → `claude-sonnet-4.5` post-#947 |
| `automation-registry` | `claude-haiku-4.5` | Reference lookup, mechanical | `claude-haiku-4.5` (already cheap) |
| `brand-asset-pipeline` | `claude-sonnet-4.6` | Visual judgment + tool orchestration | `/model auto` → `claude-sonnet-4.5` |
| `brand-compliance` | `claude-sonnet-4.6` (checklists); `claude-opus-4.7` for Social Media Art-Director Review (§8) | Visual/brand review; qualitative Art-Director judgment for social | `/model auto` → `claude-sonnet-4.5`; `claude-sonnet-4.6 --effort xhigh` + rubber-duck for Opus-escalation |
| `dev-session` | `claude-sonnet-4.6` | Session orchestration | `/model auto` → `claude-sonnet-4.5` |
| `fallback-mode` | `claude-sonnet-4.6` | Self-referential: the skill IS the fallback | n/a (already at fallback tier) |
| `kids-safety` | `claude-haiku-4.5` (trivial features only — no data/AI/IAP/UGC); `claude-sonnet-4.6 --effort xhigh` default for non-trivial features | COPPA/KOSA per-feature checklist; $50k/violation risk warrants Sonnet default when feature touches any trigger surface | `claude-sonnet-4.6` + `--effort xhigh` + rubber-duck |
| `llc-ops` | `claude-haiku-4.5` | Checklist / date lookup | `claude-haiku-4.5` (already cheap) |
| `market-scan` | `gpt-5.5` | Research + competitive analysis | extra-premium → standard: `claude-sonnet-4.6` + `--effort xhigh` + rubber-duck |
| `model-audit` | `claude-haiku-4.5` | Version diff + template generation | `claude-haiku-4.5` (already cheap) |
| `new-feature` | `claude-opus-4.7` | PLANNER intake decisioning | premium → standard: `claude-sonnet-4.6` + `--effort xhigh` |
| `qa-capture-ios` | `claude-haiku-4.5` | Mechanical `xcrun` commands | `claude-haiku-4.5` (already cheap) |
| `qa-validate` | `claude-sonnet-4.6` (CLI/platform steps); `claude-opus-4.7` for Sprite Art Gate (§2.5), Walk-Cycle Receipt Subgate (§2.5.1), Art Director Visual Review (§3.5) | Platform detection + test runs; ADA-class visual judgment for art gates | `/model auto` → `claude-sonnet-4.5`; `claude-sonnet-4.6 --effort xhigh` + rubber-duck for Opus-escalation steps |
| `risk-review` | `claude-opus-4.7` | Sensitive judgment (health/legal/kids) | premium → standard: `claude-sonnet-4.6` + `--effort xhigh` + rubber-duck |
| `5-whys` | `claude-haiku-4.5` | Mechanical root-cause template; escalate when root cause is disputed | `claude-sonnet-4.6` for severity gates (i)-(iii) or disputed root cause |
| `release-asset-fanout` | `claude-haiku-4.5` | Pure script runner (Pillow resize + manifest JSON write); zero judgment | `claude-haiku-4.5` (already cheap) |
| `release-post` | `claude-sonnet-4.6` | Content gen + tool orchestration | `/model auto` |
| `reviewer-qa-gate` | Two-pass: `claude-sonnet-4.6` (items 1–12, 17–20) + `claude-opus-4.7` (items 13–16 HARD-FAIL legal/IP) | Cost-optimized: Sonnet for layout/a11y/polish/kids-product gates; Opus only for mascot visual/voice + legal/IP gate | `claude-sonnet-4.6` + `--effort xhigh` (all items); founder MAY opt into single-Opus pass for operational simplicity |
| `ship-issue` | `claude-sonnet-4.6` | BUILDER workflow | `/model auto` → `claude-sonnet-4.5` |
| `status` | `claude-haiku-4.5` (Quick mode); `claude-sonnet-4.6` (Full mode) | Quick = JSON scan + label classification (mechanical); Full = cross-repo daily brief synthesis | `claude-haiku-4.5` for both modes |
| `usage` | `claude-haiku-4.5` | Mechanical API call — billing usage check | `claude-haiku-4.5` (already cheap) |

> ⚠️ **Deprecation notice (effective June 1, 2026):** `gpt-5.2`, `gpt-5.2-codex`, and `gpt-4.1` are scheduled for removal from most Copilot experiences. None of these IDs are routed by any Matrix row (they are in the intentionally-unlisted set). No action required before June 1; run `/model-audit` after June 1 to confirm removal from the `task` tool enum and remove these from the audit note count.

#### MCP Tool Inventory (audio/image/TTS)

Registered in `~/.copilot/mcp-config.json`. All output paths are under `~/{YOUR_ORG}/infra/{YOUR_BRAND_ASSET_PATH}`.

| MCP Server | Tool(s) | Use for | Output path |
|---|---|---|---|
| `openai-image` | `openai-image-create-image`, `openai-image-edit-image` | Brand marks, OG images, concept art | `assets/generated/` |
| `fal-ai` | `fal-ai-flux-pro-kontext-max-*` | Multi-frame sprites, reference-consistent characters | `assets/generated/` |
| `suno` | `suno-*` | Music, jingles, game audio suites | `assets/audio/generated/` |
| `elevenlabs` | `elevenlabs-text-to-speech`, `elevenlabs-list-presets` | TTS voice assets (onboarding, readback, narration) | `assets/audio/generated/elevenlabs/{preset}/` |

**ElevenLabs presets → products:** map your preset names to your own product lines in your MCP README or ops docs. Soft monthly cap $5/mo; every generated file must pass REVIEWER before entering a product repo. Voice cloning is out of scope — requires `/risk-review`. See your ElevenLabs MCP README for the full registry.

**How to use:**
- Switch with `/model <id>` or pass `model: "<id>"` when invoking a sub-agent via the `task` tool.
- "Cost-Tier Fallback" applies when spend on the current model tier needs to be reduced; full recipe lives in the `fallback-mode` skill (#463).
- Per-skill `## Model` blocks (#462) repeat the same values inline; on conflict, this matrix wins.
- **Audit note (2026-05-20 / #1231):** All explicit task-tool model IDs referenced in the matrix are still present in the current `task` tool roster (v1.0.51 enum: 14 IDs — unchanged from v1.0.48; of which 3 — `gpt-5.2`, `gpt-5.2-codex`, `gpt-4.1` — are deprecated effective June 1, 2026; post-June-1 audit will confirm their removal and reduce the count to 11). Additional exposed IDs — `gpt-5.4`, `gpt-5.4-mini`, `gpt-5-mini` (plus the 3 deprecated above) — remain intentionally unlisted here because no skill currently recommends them as the preferred or fallback route. Auto pool unchanged at 6 models. Raptor mini appeared in Auto-pool YAML with cli:false — no task enum entry, no Matrix action needed. Claude Opus 4.6 (fast mode) (preview) appears in `model-supported-clients.yml` as cli:true but is not exposed as a pin-able task enum ID; no Matrix row will be added until a stable ID is exposed in the task tool enum. GPT-5.5 multiplier confirmed at 7.5× via `model-multipliers.yml` (fetched 2026-05-18); must be re-verified post-June-1 because GitHub docs state multipliers/costs are subject to change.
- **Audit note (2026-05-24 / #1382):** `claude-opus-4.6` and `claude-opus-4.5` are now pin-able task enum IDs as of CLI v1.0.54 (no longer preview/unexposed). Both are intentionally unlisted in this Matrix — no skill currently recommends them as a preferred or fallback route; all Opus-tier routing uses `claude-opus-4.7`.
- **Auto model selection (GA 2026-04-17; server-side routing since v1.0.43):** `/model auto` is a viable alternative for most Standard-tier and Mechanical-tier skills. It routes dynamically among models with ≤1× multipliers, chosen in real time by GitHub server-side logic — pool composition can change without a CLI version bump. As of 2026-05-06 (`github/docs` commit [`98afef20`](https://github.com/github/docs/blob/main/data/tables/copilot/auto-model-selection.yml)) the CLI Auto pool contains 6 models: Claude Sonnet 4.6, Claude Haiku 4.5, GPT-5.4, GPT-5.3-Codex, **GPT-5.4 mini**, **GPT-5 mini**. Authoritative live list: [`auto-model-selection.yml`](https://github.com/github/docs/blob/main/data/tables/copilot/auto-model-selection.yml). Paid plans receive a 10% multiplier discount on Auto responses. Auto **cannot reach Opus 4.7** — all Opus-tier skills must stay on explicit model IDs. Keep explicit `claude-haiku-4.5` for any task where zero-premium spend is required.
- **Guardrail:** do not pair sticky `--effort xhigh` with `/model auto` (notably inside `/fallback-mode`). Auto may route to `claude-haiku-4.5`, which rejects reasoning effort and can fail the request.

## Session End

Run these steps before ending any session. Each step takes < 1 minute.

### 1. Store new learnings

For each significant discovery this session, call `store_memory`:

- **Error resolutions** — what broke, what fixed it, why
- **Build/QA commands** — exact commands that worked (or didn't)
- **Procedures refined** — capture steps, deployment gotchas
- **Brand/design decisions** — confirmed choices, rejected alternatives
- **MCP/tool configuration** — env vars, paths, flags that matter

Skip if the session was purely research with no new actionable facts.

### 2. Sync digital twin

```
Call tom-syncTwin with mode: "incremental"
```

This ensures the next session can retrieve context from this one via `tom-continuationPrompt`.

### 3. Return to default branch

Before branch cleanup, the repo must be back on its real default branch or fail
with an explicit blocker:

- `origin/HEAD` must resolve to a stable default branch, not a feature or WIP branch
- detached HEAD state is non-compliant
- a dirty non-default branch is non-compliant because session-end cannot switch safely
- if the default branch is checked out in another linked worktree, stop and surface it
- in your tracking repo, session-end is also non-compliant when the default branch is ahead of, behind, or diverged from upstream
- in your tracking repo, local-only spec/workflow commits that reference already-closed issues must land through a reviewable path or remain attached to an explicit open landing step before issue closeout

### 4. Clean merged local branches

If you merged a feature branch during this session, clean it up locally before
stopping:

```bash
scripts/cleanup-redundant-local-branches.sh feature/issue-N-short-description
scripts/cleanup-redundant-local-branches.sh --apply feature/issue-N-short-description
```

- Run the dry-run first.
- Only apply once the branch is confirmed redundant and any linked worktree is
  clean.
- If the current repo does not have this helper, use the repo-native equivalent
  cleanup flow before ending the session.

### 5. Update issue state

If working on a GitHub Issue, add a progress comment:

```bash
gh issue comment <N> --repo {YOUR_REPO} --body "Progress: <what was done, what remains>"
```

### 6. Note incomplete work

If work is unfinished, ensure:
- The session summary captures what remains
- Any blocking issues are documented
- The GitHub Issue reflects current state
- Any tracking-repo issue with local-only spec/workflow commits is either still attached to an explicit open landing step or not closed yet
- Any hook warning about a dirty worktree is addressed by either committing the intended changes, handing the work off to BUILDER or `/delegate`, or explicitly calling out why the files remain uncommitted

### 6b. Run severity-gate completion check

Scan current session turns for correction markers: `wrong`, `mistake`, `I should have`, `apologies`, `bad premise`, `fixed`, `root cause`, `severity gate`.

- If candidates are found, surface:
  `⚠️ Candidate severity-gate events detected this session — please confirm 5-whys is filed on the related issue, or mark as excluded (typo / retried tool call / pre-founder-visible self-correction).`
- If none are found, state:
  `No severity-gate events detected this session.`

**Untracked-file disposition protocol:**
For each untracked file shown by `git status --short`:
1. Identify the linked issue (from file content, filename, or session context).
2. If the linked issue is **OPEN** and the approach is still live → label "needs commit or issue + PR".
3. If the linked issue is **CLOSED** or the approach was abandoned → default to **delete-confirm**: surface to founder, then delete if approved. Do NOT create a new tracking issue.
4. If no linked issue can be found → surface to founder for disposition. Do not assume it should be committed.

### `.dtwin/` protected artifact protocol

Treat `.dtwin/` as protected AI-infra state, not generic scratch/residue.

Before any cleanup/disposition claim about `.dtwin/`, verify from the repo root:
1. **dtwin CLI behavior** (both forms are valid):  
   `python3 -m dtwin list-rules --status pending`  
   `python3 -m dtwin --repo-root "$PWD" list-rules --status pending`
2. **Local `.dtwin/dtwin.db` state** — confirm schema/tables and row counts are readable.
3. **Canonical dtwin DB comparison** — check your canonical shared dtwin database path (for example, `{YOUR_DTWIN_DB_PATH}`) and compare state.
4. **Git visibility/ignore state** — run `git status --untracked-files=all .dtwin` and `git check-ignore` for `.dtwin` artifacts.

Do not delete `.dtwin/`, `dtwin.db`, `dtwin.db-shm`, or `dtwin.db-wal` without explicit founder approval and a replacement ignore/state policy.

## Choosing CONDUCTOR vs individual agents

For multi-role work, prefer CONDUCTOR (`.github/agents/conductor.agent.md`). It spawns PLANNER/BUILDER/REVIEWER as subagents and stops only at named founder gates. Use individual agents only for single-role asks.

CONDUCTOR runs the mandatory plan-critique checkpoint (REVIEWER critiques plans) and pre-PR diff review automatically — these are blind-spot catchers, not nice-to-haves.

## Anti-patterns

- Sessions longer than 15 turns (quality drops sharply)
- Waiting for auto-compaction at 95% (loses critical early context)
- Not syncing twin at end of session (next session starts blind)
- Using Opus 4.7 for routine work (15× vs 1× Sonnet 4.6; Opus 4.5/4.6 are 3×; Opus 4.6 fast mode preview is 30× and is not task-enum pin-able as of v1.0.51). Re-verify after June 1, 2026 because model multipliers and costs are subject to change.
- Re-discovering facts already in repository_memories
- Re-cloning repos that already exist locally
- Re-running setup completed in prior sessions
- Re-discovering procedures encoded in skills (QA capture, brand pipeline, etc.)
- Investigating from scratch without checking `repository_memories` first
