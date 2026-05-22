---
name: fallback-mode
description: Degraded-but-disciplined mode for when Claude Opus 4.7 premium requests are exhausted or must be conserved. Codifies the recipe that recovers ~85-92% of Opus quality on standard-tier models via extra process discipline.
tier: standard
---

# Fallback Mode

## Model

- **Preferred:** `claude-sonnet-4.6` (explicit effort-capable fallback lane for sticky `--effort xhigh`)
- **Premium-exhausted fallback:** n/a (already at fallback tier)
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

## Posture: thoroughness over speed

Fallback mode trades latency for accuracy. The founder has explicitly stated: **prefer slower, verified work over fast, plausible-sounding work.**

- It is correct to take +N turns, +1 rubber-duck pass, or to stop and defer rather than guess.
- It is incorrect to compress steps to "save time" when premium quota is the bottleneck.
- If a task feels rushed, slow down. Add a verification turn instead of skipping one.

## When to invoke

- Session received a quota-exhausted / HTTP 429 response (auto-surfaced by the `fallback-detector` extension)
- Founder explicitly wants to conserve premium requests
- Starting a long batch of routine work where Opus is overkill

## Activation Checklist (self-verify before continuing)

**Do not proceed with any task until all three steps are confirmed.**

- [ ] **Model switched:** typed `/model claude-sonnet-4.6` and the session confirmed the switch
- [ ] **Effort set:** will start the next CLI invocation with `--effort xhigh`; if `/model` cannot set effort directly, pass `--effort xhigh` on the command line when launching the CLI
- [ ] **Recipe read:** read all 6 steps in [The Recipe](#the-recipe-6-steps--follow-all-of-them) below before writing any code or plan

If you cannot check step 1 right now, stop here and type `/model claude-sonnet-4.6` before continuing.

## The Recipe (6 steps — follow all of them)

1. **Switch model:** `/model claude-sonnet-4.6` for reasoning tasks. Use `/model gpt-5.3-codex` explicitly for pure code edits.
2. **Crank reasoning:** start the next CLI invocation with `--effort xhigh`. Sonnet 4.6 accepts this flag and gains ~3-6 points on agentic benchmarks.
3. **Mandatory rubber-duck pass:** call `task(agent_type="rubber-duck", ...)` after each plan **and** after each implementation. This pattern catches ~70% of bugs that Sonnet misses but Opus would catch. Rubber-duck passes do NOT recursively trigger another rubber-duck.
4. **Decompose aggressively + savepoint-preferred:** break the task into ≤2-task atomic chunks. Between chunks, **prefer the savepoint pattern** (write RESUME.md → `/clear` → reload) over `/compact` — savepoint provides 4–8× more context headroom and Sonnet degrades faster on residual summaries than Opus does. Use `/compact` only when one small step remains or savepoint setup cost exceeds benefit (see `dev-session/SKILL.md` § Savepoint pattern for the full decision rule).
5. **Pin code generation to GPT-5.3-Codex:** during fallback, route all non-trivial code edits through `task(model="gpt-5.3-codex", ...)`.
6. **Defer-don't-substitute** for the list below — stop and wait for quota reset, don't attempt on Sonnet.

## Anti-hallucination amplifiers (fallback-specific)

The universal Anti-Hallucination Rules in `.github/copilot-instructions.md` apply at all times. In fallback mode, these are **tightened** — Sonnet hallucinates more than Opus on identifier recall, API surface area, and "do I remember this file?" questions. Mitigations:

- **No-cite = no-claim is HARD here**, not soft. For any "must cite" claim (per claim tiers in copilot-instructions.md), if you don't have evidence in this session, run the tool call before responding — do not promise to verify "next turn."
- **Read source files even when you "remember" them.** Sonnet's recall of file contents from prior context degrades faster than Opus's. View the file this turn before quoting it.
- **Grep before naming any symbol, file, or path.** Do not type a path or function name from memory in fallback mode without a `glob` or `grep` confirmation in this session.
- **Verify identifiers via tool, not memory.** Issue numbers, PR numbers, commit SHAs, version pins, dates: re-fetch via `gh` or `view` this session, even if a memory states the value.
- **Run the rubber-duck before any factual assertion in these high-risk classes** (per universal rules): architecture claims, tool/model capability claims, issue closure logic, release state, default-branch assertions, side-effecting founder asks.
- **When in doubt, defer** to the next premium-quota window and add a `Stale-risk:` or `Needs founder input:` marker rather than guessing.

## Defer-don't-substitute categories

Stop and open a follow-up issue instead of attempting on standard-tier:

- **Swift 6 concurrency debugging** (actor isolation, data-race sendable violations)
- **Multi-file refactors touching ≥5 files** (Opus lead on multi-file agentic coding is ~12 pts on SWE-bench Verified)
- **Image-input analysis at >2× resolution** (Opus-only capability)
- **Deep architectural decisions requiring wide codebase context**
- **Any claim that will drive a side-effecting founder action** (merge, deploy, publish, money/legal) where evidence cannot be gathered with tool calls in this session

## Rules while active

- Do not invoke `task(model="claude-opus-*")` — the extension hook will reject it; this skill states the rule so human agents honor it too.
- Do not pair sticky `--effort xhigh` with `/model auto` or `claude-haiku-4.5`. Auto can route to Haiku 4.5, which rejects reasoning effort. If you need Haiku, start a fresh session or clear effort first.
- Do not switch off `--effort xhigh` until `/fallback-mode-off` or the founder explicitly clears it.
- Every handoff produced under this mode MUST prefix its header with:
  `FALLBACK_MODE: active — quality may be reduced, founder review weight ↑`
- Every handoff produced under this mode MUST include an `Evidence:` block (per copilot-instructions.md compact evidence pattern) — no implicit "trust me" claims.

## Worked example (BUILDER task)

Task: "Add a 2-line logging statement to `.github/skills/status/SKILL.md` and update the Matrix row."

- Default (Sonnet 4.6): 1 turn, ~3 tool calls.
- Under fallback-mode: `/model claude-sonnet-4.6` → view target file (don't trust memory) → plan → rubber-duck plan → apply → view diff → rubber-duck diff → verify Matrix row name via grep → commit; handoff prefixed with `FALLBACK_MODE: active` and an Evidence block.

Overhead: +2 rubber-duck turns, +1 verification view, +1 grep. Value: catches a typo, a stale path memory, or a logic miss Sonnet alone would have let through.

## Exit

- `/fallback-mode-off` (founder only) — clears the flag, resumes normal model routing.
- Quota reset — the `fallback-detector` extension auto-surfaces a "premium restored" notice; founder confirms before clearing.
