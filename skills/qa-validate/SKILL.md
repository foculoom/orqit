---
name: qa-validate
description: QA validation workflow for any product. Use this when asked to validate, test, or capture evidence for a build. Automatically detects platform (Godot, iOS, web) and applies the correct capture procedure.
tier: two-pass
---

# QA Validation

## Model

- **Preferred:** `claude-sonnet-4.6` (default); escalate to `claude-opus-4.8` for Sprite Art Gate (§2.5), Walk-Cycle Receipt Subgate (§2.5.1), and Art Director Visual Review (§3.5) — ADA-class visual judgment matches the `reviewer-qa-gate` Pass 2 standard
- **Cost-tier fallback:** `/model auto` → `claude-sonnet-4.5`; for Opus-escalation steps use `claude-sonnet-4.6 --effort xhigh` + mandatory rubber-duck — see `/fallback-mode`
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

Capture build evidence for any product.

## Step 1: Detect platform

Ask the user or infer from the issue/PR which platform:
- **iOS** — your iOS apps (via Xcode); default to physical-device validation when a reachable device exists
- **Web** — [your static site] (static HTML)
- **macOS** — Cairn

## Step 2: Route to platform-specific skill

- iOS (reachable physical device or "install latest"/"run on my device" request) → build/install on the physical device first using repo testing docs plus `xcrun devicectl`
- iOS (explicit simulator request or no reachable physical device) → use `/qa-capture-ios` skill for simulator capture
- Web → manual browser check + screenshot
- macOS → manual app launch + screenshot

## Step 2.5: Sprite Art Gate (character art / animation deliverables only)

**Applies when:** the QA work validates new, updated, or regenerated character sprite frames or animation cycles (e.g., run cycle, jump, hit, idle frames).

> ⚠️ **Why this gate exists:** A prior REVIEWER miss on a character animation cycle occurred because REVIEWER only evaluated frames at ~60px composite game scale. At that scale, frames with broken poses (wrong silhouette, inconsistent proportions) are indistinguishable from correct ones. Source-frame inspection at native resolution is the only reliable gate.

**Mandatory before any screenshot capture or in-engine validation:**

1. Locate source sprite PNG files at their canonical path in the product repo.
2. **View each frame individually at native resolution** using the `view` tool — do NOT evaluate from a composite gameplay screenshot.
3. For each frame, explicitly verify:
   - [ ] Character silhouette is consistent with all other frames in the set (same body proportions)
   - [ ] Pose reads as the intended action (run → clearly running, jump → clearly jumping)
   - [ ] No unexpected morphing, flattening, or pose regression between frames
   - [ ] Direction is consistent across all frames (all facing the same way)
4. **If any frame fails, STOP HERE.** Do not proceed to in-engine capture. Issue REVISE with each failing frame called out by name and pose defect.

**Critical policy:**
- BUILDER's "gates passed" self-report is NOT sufficient evidence. REVIEWER must independently open and view source files.
- The rubber-duck agent invoked after REVIEWER for sprite art passes must also independently view source frame files — not just the REVIEWER's text report — before confirming the verdict.

### Step 2.5.1 — Walk-Cycle Receipt Subgate (walk animation deliverables only)

**Applies when:** the QA work validates a `walk` animation cycle for sprite-forge output.

**Readiness anchor:** Your project's sprite-forge readiness document (e.g., `docs/releases/sprite-forge-readiness.md`)
- `pinned_commit`: `{YOUR_PINNED_COMMIT}` — the commit of the walk validator used when this contract was established
- `walk_contract_sha256`: `{YOUR_CONTRACT_HASH}` — the SHA256 of the walk contract file at pinned commit

**Required inputs:**

| Input | Expected location | Notes |
|-------|------------------|-------|
| Walk validator receipt JSON | PR-cited path (e.g. `evidence/walk-game/run2/receipt.json`) | Produced by `scripts/validate_walk_cycle.py --out-dir <dir>` at pinned commit |
| REVIEWER sign-off YAML/MD | PR-cited path (e.g. `evidence/1190/workstream-b/reviewer-signoff.md`) | Schema: `agents/templates/sprite-art-reviewer-signoff.md` |
| PR author handle | From PR metadata | Used for anti-self-signoff check |

**Gate procedure — execute each check in order; STOP and report FAIL on first failure:**

**Check A — Receipt file exists**

```
LOAD receipt from PR-cited receipt path
IF receipt file does not exist or is not valid JSON:
  FAIL "WALK-CYCLE-GATE: no walk validator receipt — receipt file missing or invalid JSON at <path>"
```

**Check B — Required fields present**

```
REQUIRED_FIELDS = [
  "validator_version", "contract_hash", "timestamp_utc",
  "frame_sha256s", "passed", "checks"
]
FOR each field in REQUIRED_FIELDS:
  IF field not in receipt:
    FAIL "WALK-CYCLE-GATE: receipt missing required field '<field>'"
```

> **Schema note (deviation from original handoff):** Actual Workstream A validator (v1.0.0) emits `frame_sha256s` (object keyed by filename), `timestamp_utc`, and does NOT emit `validator_commit`. The handoff used `frames[]`, `generated_at`, and `validator_commit`. Gate aligns to actual. `validator_commit` absence is a residual trust gap documented in `docs/releases/sprite-forge-g3-readiness.md §4`.

**Check C — contract_hash matches readiness doc**

```
READINESS_CONTRACT_HASH = "{YOUR_CONTRACT_HASH}"  # Replace with the SHA256 recorded in your readiness document
IF receipt["contract_hash"] != READINESS_CONTRACT_HASH:
  FAIL "WALK-CYCLE-GATE: receipt.contract_hash mismatch — got '<value>', expected '<READINESS_CONTRACT_HASH>'"
```

**Check D — validator_commit provenance (best-effort)**

```
# validator_commit may not be emitted by the current validator receipt.
# Record this as a residual trust gap; do not FAIL on field absence.
# IF a future validator version adds this field, enforce:
#   IF receipt.get("validator_commit") is present AND receipt["validator_commit"] != "{YOUR_PINNED_COMMIT}":
#     FAIL "WALK-CYCLE-GATE: receipt.validator_commit mismatch"
RECORD "WALK-CYCLE-GATE: validator_commit field absent from receipt; residual trust gap noted — contract_hash check is primary provenance anchor"
```

**Check E — REVIEWER sign-off exists and schema is valid**

```
LOAD signoff from PR-cited sign-off path
IF sign-off file does not exist:
  FAIL "WALK-CYCLE-GATE: REVIEWER sign-off missing — no file at <path>"

REQUIRED_SIGNOFF_FIELDS = [
  "reviewer_handle", "signed_at", "frame_verdicts",
  "native_resolution_confirmed", "display_medium", "source_frame_commit"
]
FOR each field in REQUIRED_SIGNOFF_FIELDS:
  IF field not in signoff:
    FAIL "WALK-CYCLE-GATE: sign-off missing required field '<field>'"

IF signoff["native_resolution_confirmed"] != true:
  FAIL "WALK-CYCLE-GATE: sign-off native_resolution_confirmed is not true"

IF signoff["source_frame_commit"] != "{YOUR_PINNED_COMMIT}":
  FAIL "WALK-CYCLE-GATE: sign-off source_frame_commit mismatch"

IF signoff["frame_verdicts"] is not a list OR len(signoff["frame_verdicts"]) < 6:
  FAIL "WALK-CYCLE-GATE: sign-off frame_verdicts must contain at least 6 entries (one per frame)"

FOR each entry in signoff["frame_verdicts"]:
  IF "frame_index" not in entry OR "verdict" not in entry OR "notes" not in entry:
    FAIL "WALK-CYCLE-GATE: sign-off frame_verdicts entry missing frame_index, verdict, or notes"
```

**Check F — Anti-self-signoff (reviewer independence)**

```
PR_AUTHOR = <handle of PR author>
BUILDER_HANDLE = <handle used by BUILDER agent for this PR>

IF signoff["reviewer_handle"] == PR_AUTHOR:
  FAIL "WALK-CYCLE-GATE: independence violation — reviewer_handle equals PR author '<PR_AUTHOR>'"

IF signoff["reviewer_handle"] == BUILDER_HANDLE:
  FAIL "WALK-CYCLE-GATE: independence violation — reviewer_handle equals BUILDER handle '<BUILDER_HANDLE>'"
```

> **Policy note:** Founder-approved rule: reviewer_handle must differ from PR author AND BUILDER agent handle. It does NOT need to differ from PLANNER/spec author.

**Check G — Receipt passed**

```
IF receipt["passed"] != true:
  FAIL "WALK-CYCLE-GATE: walk validator receipt reports passed=false — walk cycle did not pass automated checks"
```

**If all checks pass:**

```
PASS "WALK-CYCLE-GATE: all 7 checks passed — receipt schema valid, contract_hash matches, REVIEWER sign-off valid and independent"
```

**Reporting format:**

```
Walk-Cycle Receipt Subgate: PASS ✅ / FAIL ❌
Check A (receipt exists):       PASS / FAIL <reason>
Check B (required fields):      PASS / FAIL missing: <field>
Check C (contract_hash):        PASS / FAIL got: <value>
Check D (validator_commit):     NOTED (residual trust gap) / PASS (if field present)
Check E (sign-off schema):      PASS / FAIL <reason>
Check F (independence):         PASS / FAIL reviewer_handle=<value> conflicts with <role>
Check G (receipt passed):       PASS / FAIL
```

## Step 3: Collect evidence artifacts

Required artifacts for Ship recommendation:
- [ ] Build succeeds (log output)
- [ ] Screenshot(s) of key screens — **each validated** (resolution ✓, file size ✓, correct content ✓)
- [ ] Video of gameplay/interaction (iOS only)
- [ ] No console errors or warnings
- [ ] Performance acceptable (no frame drops, lag)

## Step 3.5: Art Director Visual Review

After screenshots are collected and format-validated, run a visual audit across **all user-facing surfaces** before writing the QA report. Format checks cannot detect brand failures, wrong-product assets, or layout defects. A prior cross-product label contamination incident shows why this step is mandatory.

### Art Director Persona

The reviewer running this step operates as a seasoned art director — someone who has won Apple Design Awards, watched what separates the four-star apps from the ones that make the Editors' Choice list, and can immediately tell when a product doesn't know what it wants to look like. Feedback is direct, opinionated, and grounded in references to public work that got it right.

**Voice:** Confident, precise, not hedge-y. Say "this reads like a placeholder" rather than "this may potentially need attention." Say "Florence (2018 ADA) made every piece of typography carry emotional weight — this label carries none" rather than "typography could be improved."

**Citation standard:** Every qualitative finding must reference at least one public work that solved the same problem well. Acceptable sources:
- Named Apple Design Award winners (e.g., Alto's Odyssey 2018, Florence 2018, Pok Pok 2022, Darkroom 2022, What the Golf 2020, Monument Valley 2014, Halide 2019, Bear 2017)
- Named Apple Editors' Choice or Featured apps with a clear design reputation
- Named games with well-documented visual direction (Sayonara Wild Hearts, Toca Boca series, Patterned by BRDG)
- HIG guidelines cited by section (e.g., "HIG §Typography — SF Pro Semibold at 13pt minimum for secondary labels")

**What the art director is NOT doing:** rubber-stamping a checklist. They are deciding whether this product looks like it belongs on the App Store next to the competition, or whether it would embarrass the brand if a design journalist screenshot it.

**Reference library for kids products:**
- Toca Boca apps — exemplary tactile delight; every tap feels considered; nothing condescends to children
- Pok Pok (2022 ADA) — restrained palette, maximum expressiveness within that restraint; no element on screen that doesn't contribute to joy
- Alto's Odyssey (2018 ADA) — atmospheric cohesion; the sky gradient, the particle behavior, the typography — all from the same visual world
- Florence (2018 ADA) — typography as storytelling; color shift as emotional arc; proof that simple doesn't mean cheap

### Surface A — App / Simulator / Godot build

> **For sprite/animation deliverables:** Confirm Step 2.5 Sprite Art Gate was completed and each source frame passed before evaluating in-engine screenshots.

```bash
open qa/screenshots/*.png    # iOS / macOS / Godot combo-QA tier images
```

Check every screenshot for:

- **Product name:** Correct product name everywhere — no cross-product label contamination
- **Character / mascot:** Correct product character; no wrong-product sprite bleed
- **Brand colors:** Match product palette in `docs/brand/`
- **Typography:** Correct font; no Lorem Ipsum, no `[String Key]` tokens
- **No debug overlays:** No FPS counter, collision shapes, debug quads (unless DEBUG-only QA build)
- **No z-order / alpha artifacts:** All layers composited; no ghost elements
- **Layout integrity:** Safe area respected; HUD doesn't occlude gameplay; nothing clipped

Any failure blocks the screenshot — file as a separate issue even if pre-existing.

### Surface B — Website

For any release that touches or accompanies a website update (your product website and landing pages), run `/brand-compliance` on the website surface. Check additionally:

- [ ] Product page shows the correct product name and character (not a sibling product)
- [ ] OG image / share card reflects the current product (no stale or wrong-product card)
- [ ] Metadata (`<title>`, og:title, og:image) updated to match the release version

### Surface C — Social Media Assets

For any release with a social post, announcement image, or story asset:

- [ ] Asset file name and alt-text match the correct product name
- [ ] Character / mascot is correct for this product
- [ ] Color palette matches the product's brand spec
- [ ] No draft / placeholder copy ("TBD", "Lorem", "CHANGE ME") in the final export
- [ ] Image dimensions match the target platform slot (see table below)
- [ ] `{YOUR_BRAND_ASSET_PATH}/release/<product>/<version>/manifest.json` updated (run your asset pipeline, e.g. `release-asset-fanout` or equivalent, if not)

| Platform | Recommended size | Notes |
|---|---|---|
| Twitter / X post | 1200×675 | 16:9 landscape |
| Instagram square | 1080×1080 | Square |
| Instagram story | 1080×1920 | 9:16 portrait |
| App Store release hero | 1320×2868 | 6.9" portrait |
| OG share card | 1200×630 | Standard web |

Record findings in the QA report. **Every FAIL must include a citation** of a public work that did it correctly and a specific directive for the fix:
```
Art Director Visual Review: PASS / FAIL
Surface A — App:
- [✅/❌] Product name correct on all screens
- [✅/❌] Character / mascot correct
- [✅/❌] Brand colors match spec
- [✅/❌] No placeholder / debug text
- [✅/❌] No debug overlays
- [✅/❌] No z-order or alpha artifacts
- [✅/❌] Layout / safe area correct
Qualitative notes: <direct, opinionated feedback with citations>
  Example: "The score label weight feels arbitrary. Halide (2019 ADA) proves that
  SF Pro Semibold at 15pt on a dark surface reads at a glance under motion —
  we're using Regular at 13pt and it disappears during gameplay."

Surface B — Website: [PASS / FAIL / N/A — no website update this release]
- brand-compliance result: <link or inline>
- Qualitative notes: <cited feedback>

Surface C — Social assets: [PASS / FAIL / N/A — no social post this release]
- [✅/❌] Correct product name + character
- [✅/❌] No placeholder copy
- [✅/❌] Dimensions match target slots
- [✅/❌] manifest.json updated
- Qualitative notes: <cited feedback>

Issues found: <describe or link to filed issue>
```

## Step 4: Validate captures

Every screenshot MUST pass validation before being accepted as evidence:

```bash
# Verify file exists
[ ! -f "$FILE" ] && echo "FAIL: screenshot not created" && exit 1

# Resolution check
sips -g pixelWidth -g pixelHeight "$FILE"

# File size check (bad captures are 180-255KB; good ones are 3.5+ MB)
SIZE=$(stat -f%z "$FILE")
[ "$SIZE" -lt 512000 ] && echo "FAIL: file < 500KB — likely blank or corrupt" && exit 1
```

If any screenshot fails: do NOT accept it as evidence. Re-capture with corrected
parameters and investigate the cause (wrong screen? blank render? timing issue?).

Allow 1–2 seconds after UI actions before capturing — animations and transitions
may produce mid-frame artifacts that pass size checks but show incorrect content.

### App Store screenshot validation

When capturing for App Store submission, also verify:
- Device matches the required slot (6.9" = iPhone 16/17 Pro Max at 1320×2868)
- Only 6.9" screenshots are strictly required — Apple auto-scales to smaller slots
- ⚠️ iPhone 17 (non-Plus) is 6.3" (1206×2622) — does NOT match the 6.7" slot
- 6.5" slot = iPhone 13 Pro Max (1284×2778), NOT iPhone 14 Pro Max
- Captured image resolution matches the expected resolution for the device

## Step 5: Output QA report

Format:

```
## QA Report — [Product] [Issue #N]

**Platform:** iOS / Web / macOS
**Build:** ✅ Success / ❌ Failed
**Evidence:**
- Screenshot: qa/screenshots/...
- Video: qa/videos/...

**Issues Found:**
- (list any defects)

**Recommendation:** Ship ✅ / Revise ❌
```

## Step 6: Knowledge Checkpoint

After completing validation, store any learnings:

```
For each of these, call store_memory if a new fact was discovered:
- Capture procedure refinements (coordinates, timing, flags)
- Device-specific gotchas (simulator quirks, window offsets)
- Build steps that weren't documented
- Error resolutions during QA
```

## Step 7: Store in artifacts directory

Save evidence to `qa/screenshots/` and `qa/videos/` in the product repo.
Review-only captures should stay uncommitted by default, so product repos should
ignore those paths unless they explicitly document durable tracked QA artifacts.
