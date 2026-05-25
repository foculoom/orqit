---
name: kids-safety
description: Per-feature COPPA/KOSA checklist for any kids product. Run before spec on any feature touching user data, IAP, AI output, or social features for minors.
tier: standard
---

# Kids Safety Skill

> **Order of operations:** Run **after** `risk-review` (broad ship/kill/descope decision) and **before** PLANNER spec work on the surviving surface. See `## Order of operations` below.
>
> **Note:** This checklist is not legal advice. Verify requirements with qualified counsel. Last verified: 2026-05-24.

## Model

- **Preferred:** `claude-haiku-4.5` (mechanical checklist mode) — **only** when feature is provably trivial (no data collection AND no AI output AND no IAP AND no UGC/social, all confirmed in invocation context)
- **Default for non-trivial features:** `claude-sonnet-4.6 --effort xhigh` + mandatory rubber-duck pass — COPPA/KOSA carry $50k/violation risk; default to Sonnet when the feature touches user data, AI, IAP, or social/UGC surfaces (i.e., any of trigger surfaces 1–5 below)
- **Escalation (judgment required flag):** `claude-sonnet-4.6 --effort xhigh` — see `## Escalation`
- **Cost-tier fallback:** `claude-sonnet-4.6` + `--effort xhigh` + mandatory rubber-duck pass — see `/fallback-mode`
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

## When to use

Run this skill when a feature (new or modified) touches **any** of the following kids-product trigger surfaces:

1. **User data** — any collection, storage, or transmission of data from a child user (name, device ID, usage analytics, location, voice input, photos)
2. **Account creation** — signup, sign-in, profile management, or parental-account linking for users under 13 (COPPA) or under 17 (KOSA)
3. **IAP visible to child** — any in-app purchase UI, subscription prompt, or premium-unlock flow that a child can see or tap without a parent intercept
4. **Analytics events** — any new or modified analytics/telemetry event sent from a kids-facing surface (even if aggregate or anonymized)
5. **AI-generated content surfaced in UI** — any TTS output, image-gen output, or LLM text rendered inside a kids-product screen

Also run when:
- A feature is added to any product labeled `kids`
- A third-party SDK is added to or updated in a kids-product build
- Social interaction, UGC, external links, or push notifications are introduced

## COPPA

Checklist for Children's Online Privacy Protection Act compliance. Check every box before proceeding.

**Notice & Consent:**
- [ ] The feature does NOT collect personal information from users under 13 without verifiable parental consent
- [ ] If personal information is collected, a COPPA-compliant privacy notice is displayed at or before collection
- [ ] Parental consent mechanism is implemented (direct notice to parent, verified consent before data collection)
- [ ] The app does NOT condition a child's participation on disclosing more personal information than necessary

**Data Minimization:**
- [ ] Only the minimum personal information necessary for the feature is collected
- [ ] No persistent identifiers (device ID, IP address used beyond session, advertising IDs) are linked to child-identifiable data without consent
- [ ] Data retention period is defined and minimized; no indefinite retention

**Third-Party SDK Audit:**
- [ ] Every third-party SDK in the kids build has been reviewed for COPPA compliance (analytics, ads, crash reporting, attribution)
- [ ] SDKs that do not have a Kids-mode or COPPA-compliant operating mode are **removed** or isolated from the kids build
- [ ] No behavioral advertising SDKs are present in a kids-facing build
- [ ] FAL, ElevenLabs, and OpenAI API calls from kids surfaces transmit no child-identifiable parameters (user ID, device fingerprint) unless founder has approved a specific data processing agreement

**Disclosure:**
- [ ] App Store listing for kids-category apps includes COPPA disclosure
- [ ] Privacy policy linked from the app is COPPA-specific (not just a generic policy)

## KOSA (Kids Online Safety Act)

Checklist for Kids Online Safety Act design-feature requirements. KOSA applies when target audience includes users <17.

### (a) Design-feature mitigations (harmful design patterns)

The following patterns are **prohibited** on any kids-facing surface:
- [ ] **No autoplay** — content does not auto-advance to the next item without user initiation (video, audio, story levels)
- [ ] **No infinite scroll** — feed/list does not scroll without a clear stopping point or pagination control
- [ ] **No manipulative dark patterns** — no false urgency ("hurry, only 2 left!"), no misleading UI that tricks children into unintended actions
- [ ] **No engagement-maximization reward/streak loops** — no streak counters, reward chains, or notification sequences designed primarily to maximize daily active use rather than educational/play value
- [ ] **No guilt-trip exit flows** — no "are you sure? You'll lose your progress!" language on optional exit or close actions
- [ ] **No countdown timers on purchases** — no time-limited purchase prompts targeting children

### (b) Parental tools requirement

- [ ] Parental controls are available (time limits, content filters, purchase locks, or account oversight via parent dashboard or device Screen Time integration)
- [ ] Parent account or guardian-link flow exists for the feature (or documented that existing parental oversight covers this feature)
- [ ] Parents can review, export, or delete their child's data on request

### (c) Harmful-content categories

These categories must be **absent** from kids-facing surfaces. Confirm each:
- [ ] **Sexual content** — no suggestive, romantic, or explicit content of any kind
- [ ] **Self-harm** — no content depicting or encouraging self-injury, suicide, or eating disorders
- [ ] **Substances** — no depiction of alcohol, tobacco, drugs, or vaping
- [ ] **Online predator risk** — no unmoderated direct messaging, no public display of child's real name or location, no ability for strangers to contact the child
- [ ] **Marketing manipulation** — no influencer-style endorsements, no deceptive native ads, no sponsored content presented as organic

## AI-Generated Content (kids surfaces)

### (a) Inventory of kids-facing surfaces using AI

List every AI-powered component in the kids product being reviewed:

| Surface | AI provider | Content type | Review gate in place? |
|---------|-------------|--------------|----------------------|
| TTS narration / voiceover | ElevenLabs (example presets — replace with your own preset names) | Audio | Pre-rendered + founder reviewed before ship |
| Image generation | FAL.ai (Flux Pro) or OpenAI (`gpt-image-1`) | Visual | Pre-rendered + REVIEWER sign-off required |
| LLM text (hints, story, dialogue) | OpenAI / Claude | Text | Human-in-the-loop review gate required |

> **New feature checklist:** If the feature adds a new AI surface not listed above, it must be added to this table and a review gate must be defined before the feature can proceed to spec.

### (b) Review gate requirement

Before **any** AI-generated output is shown to a child:

- [ ] AI output is **pre-rendered and reviewed** by a human (founder or REVIEWER) **before** it ships in a build — OR —
- [ ] A **human-in-the-loop gate** exists in the production flow that prevents unreviewed AI output from reaching a child user
- [ ] No real-time, un-reviewed LLM text or image generation is surfaced directly to a child in production
- [ ] Voice cloning is **out of scope** for kids surfaces — requires `/risk-review` with explicit founder sign-off

### (c) Cross-reference to reviewer-qa-gate AI items

`reviewer-qa-gate` items 15-16 own **output verification** of AI-disclosure on kids surfaces. This skill owns the **pre-spec input gate**. Do not duplicate; cross-reference instead.

→ See `.github/skills/reviewer-qa-gate/SKILL.md` items 15-16 (AI-generated content disclosure — HARD-FAIL)

## Escalation

Raise the **"judgment required"** flag when:
- A COPPA or KOSA checklist item cannot be definitively answered `[ ]`/`[x]` without product architecture context not available in the feature spec
- A third-party SDK has no public COPPA compliance documentation
- A new AI surface is proposed that doesn't fit any pre-rendered + reviewed pattern
- Any answer to a harmful-content category is "uncertain" or "partial"

When the flag is raised:
1. Switch to `claude-sonnet-4.6 --effort xhigh` for the remainder of this invocation
2. Annotate every uncertain item with `⚠️ JUDGMENT REQUIRED: <reason>`
3. Route to `risk-review` for net-new sensitive surfaces (see `## Cross-references`)
4. Surface to founder before any spec work proceeds

## Cross-references

The following skills and specs are **related** — cross-reference, do not duplicate their content:

- **`risk-review`** skill (`.github/skills/risk-review/SKILL.md`) — broad ship/kill/descope gate for sensitive features; runs **before** this skill
- **`reviewer-qa-gate`** items 15-16 (`.github/skills/reviewer-qa-gate/SKILL.md`) — AI-generated content disclosure HARD-FAIL; owns **output** verification
- **Your product's trust posture spec (if one exists)** — policy decisions there supersede individual feature-level judgments here

## Boundary

This skill is a per-feature **input gate** (pre-spec COPPA/KOSA/AI-content checklist). It does **not** own:
- Final output verification of shipped builds → owned by `reviewer-qa-gate`
- Broad product-level ship/kill/descope decisions → owned by `risk-review`
- Legal review or regulatory filings → escalate to founder + outside counsel

## Order of operations

When both `risk-review` AND `kids-safety` apply to a feature:

1. **`risk-review` FIRST** — runs the broad scope decision: ship as-is / kill / descope / restructure. This decision determines what surface (if any) proceeds to spec.
2. **`kids-safety` SECOND** — runs on whatever scope **survives** `risk-review`. Applies the per-feature COPPA/KOSA/AI-content checklist to the surviving surface.

Do not run `kids-safety` on a surface that `risk-review` has killed or descoped — it creates false confidence that a rejected surface has passed compliance review.

## Output verdicts

Every invocation of this skill **must** emit exactly one of the following terminal verdicts at the end of its output:

### `BLOCK`
**Do not proceed.** One or more of the following is unresolved:
- Any KOSA harmful-content category item is unchecked or uncertain
- Any COPPA consent gap exists (data collected without verifiable parental consent)
- A real-time, un-reviewed AI output is being surfaced to a child in production
- Founder must escalate to outside counsel, descope the feature, or explicitly accept the legal risk before any spec work begins.

### `WARN`
**Proceed only after founder explicitly acknowledges the listed risks.** Conditions:
- A kids-facing AI surface lacks a documented review gate, but no harmful-content category or COPPA consent gap exists
- A parental-tools gap exists but the feature is read-only and collects no personal information
- A third-party SDK has incomplete COPPA documentation but is not transmitting child-identifiable data
- Founder acknowledgment must be logged (comment on the issue or a handoff note) before PLANNER begins spec work.

### `ADVISE`
**Proceed.** All checklist items pass. Minor refinements are suggested (listed inline) but no gate is raised. PLANNER may begin spec work immediately.

## Example invocation

```
/kids-safety

Feature: {YOUR_APP} — "{YOUR_FEATURE_NAME}" system
Description: {YOUR_FEATURE_DESCRIPTION}
Audience: {YOUR_TARGET_AUDIENCE}
Data handling: {YOUR_DATA_HANDLING_SUMMARY}
AI usage: {YOUR_AI_USAGE_SUMMARY}
```

> **Template note:** The example verdict below assumes the described feature is a streak-style engagement loop so the BLOCK logic stays visible in the template. Replace the reasoning with your feature's actual risks when you run the checklist.

**Expected checklist output skeleton:**

```
## kids-safety checklist — {YOUR_APP} {YOUR_FEATURE_NAME}

### COPPA
[x] No personal information collected — badge state is device-local only
[x] No parental consent required — no data leaves device
[x] No third-party SDKs added
[x] No advertising IDs or persistent identifiers involved

### KOSA
[ ] ⚠️ JUDGMENT REQUIRED: Streak counter is an engagement-maximization reward loop (AC KOSA §(a) — no streak loops designed to maximize daily active use)
[x] No autoplay, no infinite scroll
[x] No countdown timers on purchases
[x] No dark-pattern exit flows
[x] No harmful-content categories

### AI-Generated Content
[x] No AI-generated content in this feature

### Verdict
**BLOCK**

Reason: The described feature is a streak-engagement loop — a design pattern explicitly prohibited under KOSA §(a) for kids surfaces. The feature cannot proceed as described.

Suggested remediation: Replace the streak counter with a non-streak achievement system (for example, cumulative play milestone badges that do not reset or create "don't break the streak" pressure). Route the revised design back through /kids-safety before spec.
```

## Anti-patterns

- Running this skill **after** PLANNER has already written the spec (it must run **before** spec, as a pre-spec input gate)
- Treating `ADVISE` as blanket approval — read the inline suggestions; some may become `WARN` or `BLOCK` if the feature scope changes
- Skipping this skill because the feature "seems harmless" — COPPA and KOSA apply to the **product** audience, not the feature content
- Running this skill on a surface that `risk-review` has already killed (creates false compliance confidence)
- Using this skill to perform the `reviewer-qa-gate` output verification — that is a separate, post-build step
