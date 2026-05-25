---
name: qa-capture-ios
description: Exact simulator-only QA capture procedure for your iOS apps on macOS Simulator. Use this only when physical-device validation is unavailable or simulator evidence is explicitly requested.
tier: basic
---

# QA Capture — iOS Apps (macOS Simulator Fallback)

## Model

- **Preferred:** `claude-haiku-4.5`
- **Cost-tier fallback:** `claude-haiku-4.5` — see `/fallback-mode`
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

This skill handles simulator capture only. Physical-device install and launch
remain the default iOS validation path in BUILDER and `qa-validate`.

## Critical rules — never deviate

- **Screenshots:** `xcrun simctl io "$UDID" screenshot file.png` — NEVER use `screencapture`
- **Video:** `xcrun simctl io "$UDID" recordVideo --codec=h264 raw.mp4 --force &` — stop with `kill -INT <PID>`
- **Compress:** `avconvert -p PresetMediumQuality -s raw.mp4 -o output.mp4` — NEVER use `ffmpeg` (simulator outputs ~600fps, ffmpeg fails)
- **Interaction:** NEVER use CGEvent, Quartz, cliclick, or `osascript click at` for simulator interaction. These are unreliable on macOS Sonoma+ (synthetic events silently dropped without Accessibility permissions, coordinate-based clicking is fragile). Always use XCUITest or `simctl` commands.
- **Always use UDID** — NEVER use `booted` (fails when multiple simulators running)

## Step 1: Get simulator UDID

```bash
UDID=$(xcrun simctl list devices | grep -m1 'Booted' | grep -oE '[0-9A-F-]{36}')
echo "UDID: $UDID"
```

If no simulator is booted, boot one:
```bash
xcrun simctl boot "iPhone 16 Pro Max"
UDID=$(xcrun simctl list devices | grep -m1 'Booted' | grep -oE '[0-9A-F-]{36}')
```

## Step 2: Bypass onboarding (if needed)

```bash
xcrun simctl spawn "$UDID" defaults write <bundleId> hasSeenOnboarding -bool true
```

## Step 3: Interact with the app

Use this hierarchy — try each level in order:

1. **XCUITest** (primary — reliable, Apple-supported): Write a minimal test that taps elements by accessibility identifier, run with `xcodebuild test -scheme AppUITests -destination "id=$UDID" -only-testing:AppUITests/QACaptureTests`
2. **`simctl openurl`** (secondary — deep-link): `xcrun simctl openurl "$UDID" "myapp://screen/gameplay"`
3. **Launch arguments** (tertiary — startup config): `xcrun simctl launch "$UDID" {YOUR_BUNDLE_ID}`

## Step 4: Screenshot + validate

```bash
xcrun simctl io "$UDID" screenshot qa/screenshots/screenshot.png
```

**Mandatory validation** — run after every screenshot:
```bash
FILE="qa/screenshots/screenshot.png"
[ ! -f "$FILE" ] && echo "⚠️ FAIL: screenshot not created at $FILE" && exit 1
W=$(sips -g pixelWidth "$FILE" | grep pixelWidth | awk '{print $2}')
H=$(sips -g pixelHeight "$FILE" | grep pixelHeight | awk '{print $2}')
SIZE=$(stat -f%z "$FILE")
echo "Resolution: ${W}x${H}  Size: ${SIZE} bytes"
# FAIL if file < 100KB (blank/corrupt captures are 40–80KB; flat-color screens like menu
# can legitimately produce ~226KB, so the threshold must stay well below that).
[ "$SIZE" -lt 102400 ] && echo "⚠️ FAIL: file too small — likely blank/corrupt" && exit 1
# Optional: check unique color count if ImageMagick available
if command -v identify &>/dev/null; then
  COLORS=$(identify -format '%k' "$FILE")
  echo "Unique colors: $COLORS"
  [ "$COLORS" -lt 4000 ] && echo "⚠️ FAIL: too few colors — likely blank or wrong frame" && exit 1
fi
echo "✅ Screenshot validated"
```

> **Content check:** Open the screenshot (`open "$FILE"`) and visually confirm
> it shows the expected screen — format validation alone cannot catch wrong-screen captures.

## Step 4.5: Art Director Visual Review

After every screenshot batch, open each image and run a mandatory visual check before accepting as evidence. This step exists because format/size validation cannot detect cross-product label contamination, wrong-character renders, or brand failures.

### Art Director Persona

The reviewer running this step operates as a seasoned art director — someone who has won Apple Design Awards and knows immediately when a product doesn't know what it wants to look like. Feedback is direct, opinionated, and grounded in references to public work that got it right. Say "this reads like a placeholder" not "this may need attention." Every qualitative finding must cite at least one public work that solved the same problem well (ADA winners, HIG sections, Editors' Choice apps, or well-documented design references).

**Reference library for kids products (if your product targets children):**
- Toca Boca apps — every tap feels considered; nothing condescends to children
- Pok Pok (2022 ADA) — restrained palette, maximum expressiveness within that restraint
- Alto's Odyssey (2018 ADA) — atmospheric cohesion across particles, color, and typography
- Florence (2018 ADA) — typography as storytelling; color shift as emotional arc

```bash
open qa/screenshots/*.png   # open all captures for review
```

Check every screenshot for:

| Check | Pass | Fail example |
|---|---|---|
| **Product name** | Correct product name displayed everywhere | One product's name displayed in a sibling product window |
| **Character / mascot** | Correct product character renders; no wrong-product asset | A sibling product mascot rendered in the current product |
| **Brand colors** | Match product spec (see `docs/brand/`) | Wrong palette imported from sibling product |
| **Typography** | Correct font, no Lorem Ipsum, no `[String Key]` tokens | Placeholder keys visible |
| **No debug overlays** | No FPS counter, no collision shapes, no red/green debug quads unless in DEBUG build for QA only | Debug overlay ships to store |
| **No z-order / alpha bugs** | All layers composited correctly; no ghost elements | Invisible UI element partially visible |
| **Spacing / layout** | Safe-area respected; nothing clipped by notch or home indicator | Score label partially under notch |

**Any FAIL = block the screenshot** — re-capture or file a separate issue before proceeding.

Record findings in the QA report under **Visual Review**. Every FAIL must include a citation:
```
Visual Review: PASS / FAIL
- [✅/❌] Product name correct
- [✅/❌] Character / mascot correct
- [✅/❌] Brand colors match spec
- [✅/❌] No placeholder / debug text
- [✅/❌] No debug overlays
- [✅/❌] No z-order or alpha bugs
- [✅/❌] Layout: safe area respected
Qualitative notes: <direct, cited feedback — e.g. "The HUD label sits 4pt inside
  the Dynamic Island dead zone. Halide (2019 ADA) handles this by anchoring all
  critical read-at-a-glance values to the safe area bottom. Move it there.">
Issues found: <describe or file as issues>
```

## Step 5: Verification loop

Every UI interaction must be verified before proceeding:

1. Perform the action (tap, navigate, launch).
2. Wait 1–2 seconds for animations and transitions to settle.
3. Take a screenshot.
4. Run the validation snippet from Step 4.
5. Confirm the screen actually changed (don't assume a tap worked).
6. Only proceed to the next step after validation passes.

If validation fails, retry the action (max 3 attempts) before escalating.

## Step 6: Video recording

Start recording:
```bash
xcrun simctl io "$UDID" recordVideo --codec=h264 qa/videos/raw.mp4 --force &
RECORD_PID=$!
```

Stop and compress (REQUIRED — simulator outputs ~600fps):
```bash
kill -INT $RECORD_PID && wait $RECORD_PID 2>/dev/null
avconvert -p PresetMediumQuality -s qa/videos/raw.mp4 -o qa/videos/gameplay.mp4
rm qa/videos/raw.mp4
```

## App Store device-to-slot reference

| Slot | Resolution (Portrait) | Simulator Device |
|---|---|---|
| 6.9" | 1320×2868 | iPhone 16/17 Pro Max |
| 6.7" | 1290×2796 | iPhone 15/16 Plus |
| 6.5" | 1284×2778 | iPhone 13 Pro Max |
| 6.3" | 1179×2556 | iPhone 15 Pro |

⚠️ iPhone 17 (non-Plus) is 6.3" (1206×2622) — NOT 6.7". Apple auto-scales from 6.9" to all smaller slots, so only 6.9" (Pro Max) screenshots are strictly required.
For landscape screenshots, swap width and height (e.g., 6.9" landscape = 2868×1320).

## Output locations

- Screenshots: `qa/screenshots/`
- Videos: `qa/videos/gameplay.mp4`
- Never save to `.specify/validation/`
