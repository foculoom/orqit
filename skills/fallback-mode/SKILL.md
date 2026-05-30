---
name: fallback-mode
description: Cost-tier step-down mode that reduces per-turn spend while compensating quality loss with extra process discipline. Applies to any model tier — not limited to a specific model name.
tier: standard
---

# Fallback Mode

## Model

- **Preferred:** `claude-sonnet-4.6` (standard tier — the target for most fallback step-downs)
- **Cost-tier fallback:** n/a (already at fallback tier)
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

## What it is

Fallback mode reduces the model cost tier for any skill or agent, stepping down to a cheaper tier while adding quality compensators to recover most of the quality gap. It is **not tied to any specific model** — it applies whenever the active tier is more expensive than needed.

## Step-down table

> ⚠️ Note on tier ordering: the current Matrix numbers tiers 1–4 by routing role, not by cost. Tier 4 (extra-premium, 7.5×) is cheaper per turn than tier 3 (premium, 15×). The table below is ordered by cost (high → low).

| Source tier | Example model | Multiplier | → Fallback tier | Fallback multiplier | Cost reduction | Compensation |
|---|---|---|---|---|---|---|
| premium | `claude-opus-4.8` | 15× | standard | 1× | ~15× per turn | `--effort xhigh` + rubber-duck + GPT-5.3-Codex for code |
| extra-premium | `gpt-5.5` | 7.5× | standard | 1× | ~7.5× per turn | `--effort xhigh` + rubber-duck |
| standard | `claude-sonnet-4.6` | 1× | basic | 0.33× | ~3× per turn | model switch only — **do NOT set `--effort xhigh`** (Haiku rejects reasoning effort) |

### Two-pass and mixed-tier skills

For skills that run premium on one substep and standard on another (e.g., `qa-validate`, `brand-compliance`, `reviewer-qa-gate`): **step down only the premium substeps**. Standard substeps stay at standard. Do not downgrade an entire skill when only one pass uses a premium model.

## When to invoke

- Budget threshold signal (75% / 90%) from `fallback-detector` extension
- Quota-exhausted / HTTP 429 for any premium or extra-premium model
- Founder explicitly wants to reduce spend on a routine or long batch of work
- Starting a long session where most remaining work is at a lower complexity level than the source tier

## Posture: thoroughness over speed

Fallback mode trades latency for accuracy. The founder has explicitly stated: **prefer slower, verified work over fast, plausible-sounding work.**

- It is correct to take +N turns, +1 rubber-duck pass, or to stop and defer rather than guess.
- It is incorrect to compress steps to "save time" when cost is the bottleneck.
- If a task feels rushed, slow down. Add a verification turn instead of skipping one.

## Activation Checklist (self-verify before continuing)

**Do not proceed with any task until all three steps are confirmed.**

1. Identify your **source tier** from the step-down table above.
2. Switch to the **fallback tier model** for that source tier (e.g., for premium → standard: `/model claude-sonnet-4.6`).
3. Apply the **compensation** for that tier drop (see step-down table).
4. **Recipe read:** read all 6 steps in [The Recipe](#the-recipe-6-steps--follow-all-of-them) below before writing any code or plan.

## The Recipe (6 steps — follow all of them)

1. **Switch model:** switch to the fallback tier model per the step-down table. For standard-tier fallback: `/model claude-sonnet-4.6`. For basic-tier fallback: `/model claude-haiku-4.5`.
2. **Crank reasoning (standard tier only):** if the fallback tier is standard, start the next CLI invocation with `--effort xhigh`. Sonnet 4.6 accepts this flag. **Skip this step for basic tier** — Haiku rejects reasoning effort and will fail the request.
3. **Mandatory rubber-duck pass (premium and extra-premium source tiers):** call `task(agent_type="rubber-duck", ...)` after each plan **and** after each implementation. This pattern catches ~70% of bugs that a premium model would catch but standard misses. Rubber-duck passes do NOT recursively trigger another rubber-duck.
4. **Decompose aggressively + savepoint-preferred:** break the task into ≤2-task atomic chunks. Between chunks, **prefer the savepoint pattern** (write RESUME.md → `/clear` → reload) over `/compact` — savepoint provides 4–8× more context headroom and standard models degrade faster on residual summaries than premium models. Use `/compact` only when one small step remains or savepoint setup cost exceeds benefit (see `dev-session/SKILL.md` § Savepoint pattern).
5. **Pin code generation to GPT-5.3-Codex (premium source tier only):** when stepping down from a 15× premium model, route all non-trivial code edits through `task(model="gpt-5.3-codex", ...)`.
6. **Defer-don't-substitute** for the list below — stop and wait for quota reset, don't attempt at a lower tier.

## Anti-hallucination amplifiers (fallback-specific)

The universal Anti-Hallucination Rules in `.github/copilot-instructions.md` apply at all times. In fallback mode, these are **tightened** — standard-tier models hallucinate more than premium models on identifier recall, API surface area, and "do I remember this file?" questions. Mitigations:

- **No-cite = no-claim is HARD here**, not soft. For any "must cite" claim (per claim tiers in copilot-instructions.md), if you don't have evidence in this session, run the tool call before responding — do not promise to verify "next turn."
- **Read source files even when you "remember" them.** Standard models' recall of file contents from prior context degrades faster. View the file this turn before quoting it.
- **Grep before naming any symbol, file, or path.** Do not type a path or function name from memory without a `glob` or `grep` confirmation in this session.
- **Verify identifiers via tool, not memory.** Issue numbers, PR numbers, commit SHAs, version pins, dates: re-fetch via `gh` or `view` this session.
- **Run the rubber-duck before any factual assertion in these high-risk classes** (per universal rules): architecture claims, tool/model capability claims, issue closure logic, release state, default-branch assertions, side-effecting founder asks.
- **When in doubt, defer** and add a `Stale-risk:` or `Needs founder input:` marker rather than guessing.

## Defer-don't-substitute categories

Stop and open a follow-up issue instead of attempting at a lower tier:

- **Swift 6 concurrency debugging** (actor isolation, data-race sendable violations)
- **Multi-file refactors touching ≥5 files** (premium lead on multi-file agentic coding is ~12 pts on SWE-bench Verified)
- **Image-input analysis at >2× resolution** (premium-only capability)
- **Deep architectural decisions requiring wide codebase context**
- **Any claim that will drive a side-effecting founder action** (merge, deploy, publish, money/legal) where evidence cannot be gathered with tool calls in this session

## Rules while active

- Do not invoke any model at a higher tier than the fallback tier selected for this session. The `fallback-detector` extension hook enforces this; this skill states the rule so human agents honor it too.
- Do not pair sticky `--effort xhigh` with `/model auto` or `claude-haiku-4.5`. Auto can route to Haiku 4.5, which rejects reasoning effort. If you need Haiku, start a fresh session or clear effort first.
- Do not switch off `--effort xhigh` until `/fallback-mode-off` or the founder explicitly clears it (standard tier only — basic tier never sets it).
- Every handoff produced under this mode MUST prefix its header with:
  `FALLBACK_MODE: active — quality may be reduced, founder review weight ↑`
- Every handoff produced under this mode MUST include an `Evidence:` block (per copilot-instructions.md compact evidence pattern) — no implicit "trust me" claims.

## Worked example (BUILDER task, premium → standard step-down)

Task: "Add a 2-line logging statement to `.github/skills/status/SKILL.md` and update the Matrix row."

- Default (premium): 1 turn, ~3 tool calls at 15× cost.
- Under fallback-mode (premium → standard): `/model claude-sonnet-4.6` + `--effort xhigh` → view target file (don't trust memory) → plan → rubber-duck plan → apply → view diff → rubber-duck diff → verify Matrix row name via grep → commit; handoff prefixed with `FALLBACK_MODE: active` and an Evidence block.

Overhead: +2 rubber-duck turns, +1 verification view, +1 grep. Value: catches a typo, a stale path memory, or a logic miss the standard tier alone would have let through. Cost: ~1× vs 15× — ~93% saving.

## Exit

- `/fallback-mode-off` (founder only) — clears the flag, resumes normal model routing.
- Quota reset — the `fallback-detector` extension auto-surfaces a "restored" notice; founder confirms before clearing.
