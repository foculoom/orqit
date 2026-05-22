---
name: reviewer-qa-gate
description: Default REVIEWER on-device QA pass after BUILDER's TestFlight upload; catches HUD/visual/accessibility regressions. Deviations must be documented in PR or handoff with rationale.
tier: two-pass
---

# REVIEWER QA Gate

## Model

- **Default:** Two-pass — see `## Two-pass execution` below
  - Pass 1 (items 1–12): `claude-sonnet-4.6`
  - Pass 2 (items 13–16): `claude-opus-4.7` (HARD-FAIL legal/IP scope)
- **Premium-exhausted fallback:** `claude-sonnet-4.6` + `--effort xhigh` (all items) — see `/fallback-mode`
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

## When to invoke

- After BUILDER uploads a TestFlight build and before the related issue is marked "validated"
- Before any App Store submission (gate clause in `ship-issue` skill §5.5)
- On founder-requested spot-checks of any installed Foculoom app

## Why this exists

REVIEWER did not catch the Jumpyloo HUD-blocks-character bug because no systematic on-device review ran before validation. `ship-issue` §5.5 covers BUILDER smoke-testing, but BUILDER and REVIEWER have different eyes — BUILDER checks "does it work?", REVIEWER checks "is it shippable?". This skill makes the second pass mandatory.

## Device coverage by release type

Tiered to match `ship-issue` §4.2 soak windows — do not over-test patches or under-test major releases:

| Release type | Minimum devices | Example classes |
|---|---|---|
| Patch (x.y.**Z**) | 1 | iPhone Pro Max (latest) |
| Minor (x.**Y**.0) | 2 | iPhone Pro Max + iPhone SE |
| Major (**X**.0.0) or first App Store submission | 3 | iPhone Pro Max + iPhone SE + iPad |

"Latest iPhone Pro Max" is the mandatory first device for all tiers (highest-complexity screen; catches Dynamic Island + camera-pill issues). Add SE for minor+ (smallest safe-area). Add iPad for major+/first-submit (different layout breakpoint).

## Pass criteria — 20-item checklist

Run checklist on each required device class. Capture screenshots per `qa-capture-ios` skill.

**Layout & framing:**
1. No HUD/UI element occludes character or primary gameplay surface
2. All CTAs reachable; none clipped by safe area or notch
3. Bottom-edge controls clear of home-indicator gesture area
4. Top-edge HUD clear of camera notch / Dynamic Island

**Accessibility:**
5. VoiceOver labels present on every interactive element (smoke-test 1 minute of nav)
6. Dynamic Type at XXXL doesn't break primary screens
7. Reduce Motion: parallax/transitions/idle animations respect it
8. Increase Contrast: primary CTAs remain legible

> If product = Jumpyloo (or any kids product), run the `a11y-godot` skill before continuing. See `.github/skills/a11y-godot/SKILL.md`.

**Polish:**
9. No placeholder strings, Lorem Ipsum, `[String Key]` tokens, or debug overlays; **no cross-product label contamination** (e.g., a sibling-product name appearing on screen — a failing example is "Skiplet" rendered in a Jumpyloo window); brand colors and typography match the product spec. **Art director judgment required:** reviewer must assess overall visual polish with the eye of someone who has won Apple Design Awards — not just "does it pass?" but "would this embarrass us if it appeared in a design publication?". Every qualitative FAIL must cite a public reference that did it right (ADA winner, HIG section, or respected App Store product).
10. Dark Mode parity (or explicit single-mode by spec)
11. Audio: no clipping, no missing sounds at expected events, haptics fire
12. Performance: no obvious frame drops in 60s of typical use (visual judgment; Instruments capture optional)

**Brand & Mascot (when product spec declares a mascot — mark "N/A — no mascot" otherwise):**
13. **Mascot visual presence:** mascot renders at expected screens per product spec; eyes/face/silhouette readable at iPhone Pro Max + iPhone SE display sizes; no z-order or alpha bugs. (Cross-ref the product's mascot-personality spec, e.g. `specs/2026-05-jumpyloo-mascot-personality-v1-spec.md` §10 visual brief.)
14. **Mascot voice playback:** voicelines fire at all spec-defined trigger events (cross-ref the product's meta-loop or equivalent spec, e.g. `specs/2026-05-jumpyloo-meta-loop-v1-spec.md` §4 item 6); no missing audio assets; volume per spec (e.g. "moderate" per Jumpyloo founder selection 2026-05-03).
15. **AI-disclosure copy parity ⛔ HARD-FAIL:** the literal AI-disclosure sentence required by trust-posture v2 §11 (or product equivalent) appears verbatim in **(a)** in-app About / Settings, **(b)** App Store description body, **(c)** product page on Foculoom website. For Jumpyloo the required literal sentence is: `"Mascot voice generated with AI."` (source: `specs/2026-05-jumpyloo-mascot-personality-v1-spec.md` §7 + `specs/2026-05-jumpyloo-trust-posture-v2-spec.md` §11 items 3–4). REVIEWER greps all three surfaces before marking pass. Mismatch on any surface is a HARD-FAIL regardless of defect count.
16. **Mascot voice license artifacts retained ⛔ HARD-FAIL:** voice ID + license screenshot present in `docs/trust/<product>-trust-posture-v2.md` evidence section per trust-posture v2 §11 item 5 (or product equivalent). For Jumpyloo: `docs/trust/jumpyloo-trust-posture-v2.md`. Absence blocks the release; no exception. (Source: `specs/2026-05-jumpyloo-trust-posture-v2-spec.md` §6 item 6 + §11 item 5.)

**Kids product gates** (items 17-20 — run for Jumpyloo and any kids-category product; cross-ref `.github/skills/kids-safety/SKILL.md`):

17. Tap targets ≥44×44pt on all interactive elements (SE-class device check)
18. Primary CTA affordance identifiable without reading the label (pre-reader usability)
19. Onboarding flow completable without reading text (4-year-old baseline)
20. No dark patterns: no countdown timers on purchases, no "miss out" language, no guilt-trip exit flows

## Two-pass execution (default)

Use a two-pass split by default to reduce routine QA cost while preserving high-scrutiny review for mascot/legal/IP-sensitive checks:

- **Pass 1 — `claude-sonnet-4.6` (items 1–12, 17–20):** layout, accessibility, polish, and kids-product gates (tap targets, pre-reader affordance, no dark patterns).
- **Pass 2 — `claude-opus-4.7` (items 13–16):** mascot visual/voice checks plus **HARD-FAIL legal/IP scope** (especially items 15–16).

### Pass 1 ambiguity escalation rule

If any Pass 1 result is ambiguous, borderline, or uncertain, escalate that **specific item** to `claude-opus-4.7` before deciding pass/fail for that item.

Worked example: if item **7** returns borderline accessibility contrast, escalate that single item to Opus before deciding pass/fail.

## Fallback: single-Opus pass

**Founder MAY elect** to run a single `claude-opus-4.7` pass across all items instead of the two-pass split when operational simplicity is more important than cost optimization. This is a deliberate opt-in fallback, not the default path.

## Capture rule

Per required device class: **3 screenshots minimum** (Home, mid-task, post-task) using `xcrun simctl io booted screenshot` or physical-device screenshot. Save under `qa/screenshots/<product>/<build>/` (gitignored unless durable QA spec says otherwise).

If a defect is observed, capture: (a) the broken screen, (b) the HIG reference for what it should look like (link is sufficient).

## Verdict format

Post a single comment on the tracking issue:

```
## REVIEWER QA Gate — <product> build <N>
**Verdict:** PASS / REVISE / FAIL

**Devices tested:** iPhone <model> (<iOS>), iPhone SE (<iOS>), iPad <model>

**Checklist:**
1. [✅/❌] No HUD occlusion — <note if ❌>
2. [✅/❌] CTAs reachable
3. [✅/❌] Bottom-edge controls clear
4. [✅/❌] Top-edge HUD clear
5. [✅/❌] VoiceOver labels present
6. [✅/❌] Dynamic Type XXXL OK
7. [✅/❌] Reduce Motion respected
8. [✅/❌] Increase Contrast legible
9. [✅/❌] No placeholder strings / debug overlays / cross-product labels; brand colors + typography match spec
10. [✅/❌] Dark Mode parity (or N/A — single-mode by spec)
11. [✅/❌] Audio: no clipping, no missing sounds, haptics fire
12. [✅/❌] Performance: no obvious frame drops in 60s

**Brand & Mascot (items 13–16):**
_If this product has no mascot per its product spec, mark all four as "N/A — no mascot" and skip to Kids gates._
13. [✅/❌/N/A] Mascot visual presence — <note>
14. [✅/❌/N/A] Mascot voice playback (all trigger events) — <note>
15. [✅/❌/N/A ⛔ HARD-FAIL] AI-disclosure copy parity — <cite surfaces checked>
16. [✅/❌/N/A ⛔ HARD-FAIL] Mascot voice license artifacts retained — <cite doc path>

**Kids product gates (items 17–20):**
_Apply for Jumpyloo and any kids-category product; mark "N/A" otherwise._
17. [✅/❌/N/A] Tap targets ≥44×44pt on all interactive elements
18. [✅/❌/N/A] Primary CTA affordance identifiable without reading label
19. [✅/❌/N/A] Onboarding flow completable without reading text
20. [✅/❌/N/A] No dark patterns (no countdown timers on purchases, no "miss out" language, no guilt-trip exit flows)

**Defects found:** (file as separate issues, link here)
- <#NNN> <title>

**Screenshots:** <repo-relative paths or attached>
```

## Pass / Revise / Fail rules

- **PASS:** All applicable items ✅. Items 13–16 are required only when the product spec declares a mascot; mark "N/A — no mascot" otherwise. Items 17–20 apply to kids-category products; mark "N/A" otherwise. Build proceeds to store-submission step.
- **REVISE:** 1–2 items ❌ among items 1–14, none of which are accessibility (#5–#8) or layout (#1–#4). BUILDER fixes, REVIEWER re-runs gate.
- **FAIL:** ≥3 items ❌ (items 1–14), OR any layout/accessibility ❌, OR any kids gate ❌ (items 17–20 for kids products). BUILDER blocks; PLANNER may need to scope a follow-up issue.
- **HARD-FAIL (items 15 and 16, when mascot is declared):** Any ❌ on item 15 (AI-disclosure copy parity) or item 16 (mascot voice license artifacts) is an unconditional HARD-FAIL regardless of all other items passing. Rationale: item 15 is a legal/compliance mandate per trust-posture v2 §11 items 3–4; item 16 creates IP exposure if the voice license is unretained. Neither may be waived by BUILDER, REVIEWER, or founder in a release comment — only a new trust-posture spec can change the requirement.

## Hooks into existing skills

- **`ship-issue` §5.5:** "TestFlight smoke-test" step now requires `reviewer-qa-gate` PASS before the build is considered validated.
- **`qa-capture-ios`:** continues to own *how* to capture; this skill owns *what* to verify.
- **`brand-compliance`:** runs on assets; this skill runs on the running build. Different layers; both required.
- **Mascot-personality spec** (e.g. `specs/2026-05-jumpyloo-mascot-personality-v1-spec.md`): source of truth for mascot name, visual brief, 26-line voiceline script, AI-disclosure literal sentence, and voice license requirements. Cross-ref for items 13–15.
- **Meta-loop spec** (e.g. `specs/2026-05-jumpyloo-meta-loop-v1-spec.md`): source of truth for the 6 mascot voiceline trigger events (§4 item 6) and Phase B ship-block conditions (AC #7). Cross-ref for item 14.
- **Trust-posture v2** (e.g. `specs/2026-05-jumpyloo-trust-posture-v2-spec.md`): authorizes mascot voice with disclosure (§6 item 6) and mandates AI-disclosure parity and voice license retention as hard requirements (§11 items 3–5). Cross-ref for items 15–16.

## Cross-references

- **`kids-safety`** skill (`.github/skills/kids-safety/SKILL.md`) — per-feature COPPA/KOSA input gate; runs **before** spec, not after build. Items 17-20 here are the QA-output counterpart to that pre-spec checklist. See also kids-safety `## AI-Generated Content` section for the pre-render review gate that this skill verifies at build time.
- **`a11y-godot`** skill (`.github/skills/a11y-godot/SKILL.md`) — Godot 4.6 iOS accessibility deep-dive; invoked from item 8 for Jumpyloo builds.

## Anti-patterns

- Running this skill on the simulator instead of a real device (founder explicitly requested device-first)
- Treating PASS as a rubber-stamp without screenshots
- Letting BUILDER run this skill on their own work (REVIEWER must be a different context to provide independent eyes)
- Skipping the gate "just this once" to hit a deadline (the deadline cost of a FAIL after submission is far higher than the QA-pass cost)
- **Marking the mascot section (items 13–16) as "N/A — no mascot" when the product spec actually declares a mascot.** Always cross-check against the product's mascot-personality spec before applying N/A. For Jumpyloo specifically, Hoppin is declared (`specs/2026-05-jumpyloo-mascot-personality-v1-spec.md` §14, approved 2026-05-03) — items 13–16 are never N/A for Jumpyloo builds.
