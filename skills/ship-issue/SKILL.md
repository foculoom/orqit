---
name: ship-issue
description: Implement a single scoped GitHub Issue and prepare a PR. Use this when an issue has acceptance criteria defined and is ready for implementation. Follows the BUILDER workflow.
tier: standard
---

# Ship Issue (BUILDER)

## Model

- **Preferred:** `claude-sonnet-4.6`
- **Cost-tier fallback:** `/model auto` → `claude-sonnet-4.5`; `gpt-5.3-codex` for pure-code edits — see `/fallback-mode`
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

Implement exactly one GitHub Issue and prepare a reviewable PR.

## BUILDER workflow guard — parallel worktrees

When CONDUCTOR runs ≥2 BUILDER subagents in the same repo, each BUILDER must run in its own linked worktree before implementation starts (prevents silent overwrite collisions).
Primary vs backstop: CONDUCTOR's pre-dispatch worktree creation and `WORKTREE_PATH` dispatch rejection rule is the primary guard. The BUILDER-side collision check below is a backstop only and does not replace CONDUCTOR pre-dispatch worktree creation.

Canonical command pattern:

```bash
git worktree add ../<repo>-issue-<N> -b feature/issue-<N>-<slug>
```

Runnable example:

```bash
repo_name="$(basename "$PWD")"
git worktree add "../${repo_name}-issue-904" -b "feature/issue-904-worktree-guard"
```

Collision enforcement (backstop):
- After `git worktree prune`, if `git worktree list` reveals a sibling worktree on `feature/issue-<other-issue>-...` in this repo (excluding this BUILDER's own `WORKTREE_PATH` / current issue branch), BUILDER is FORBIDDEN from using `git checkout -b` in the current checkout.
- HARD STOP and either create/use the correct linked worktree with `git worktree add` or route back to CONDUCTOR.

Track-C-style references: https://github.com/{YOUR_REPO}/issues/889 and https://github.com/{YOUR_REPO}/issues/890

## New-App First-Ship Checklist

**Only applies when the issue is the first TestFlight upload for a new iOS app.**

Before writing any code, verify every item in `docs/ios-app-onboarding.md` is satisfied in the target repo's `project.yml`. Missing items are the leading cause of avoidable build blockers and post-upload ASC web-UI prompts.

Quick reference (full details + receipts in the onboarding doc):

```
[ ] DEVELOPMENT_TEAM: H8MJMBTFVP present in settings.base
[ ] TARGETED_DEVICE_FAMILY: "1" (iPhone-only) or "1,2" (universal) — documented
[ ] CURRENT_PROJECT_VERSION: YYYYMMDD.1 convention
[ ] ITSAppUsesNonExemptEncryption set (INFOPLIST_KEY_ or info.properties)
[ ] Assets.xcassets listed under sources: not resources:
[ ] AppIcon asset set: 120/180/1024 px sizes present
[ ] CI workflow has DerivedData clean + simctl erase before test step
[ ] Bundle IDs registered in Developer Portal (automatable via POST /v1/bundleIds)
[ ] ASC app record exists (manual — web UI only, POST /v1/apps returns 403)
```

For encryption usage assessment: grep `Sources/` and `*Core/` for `import CryptoKit`,
`import CommonCrypto`, `import Security` — see onboarding doc §3 for exemption criteria.

## Step 1: Load the issue

```bash
gh issue view <N> --repo {YOUR_REPO} --json title,body,labels
```

Verify:
- Acceptance criteria exist and are testable
- Scope is clear (if not, stop and ask for clarification)
- Target repo is identified
- The handoff points at a real `{YOUR_REPO}#N`; if it still says
  `GITHUB_ISSUE: TBD`, stop and route back to PLANNER

## Step 1.5: Repo-state preflight

Before creating a feature branch, inspect the current checkout:

```bash
git status --short --branch
git worktree list
git worktree prune
# replace <N> with this issue's number
git worktree list --porcelain | grep "^branch" | grep "feature/issue-" | grep -v "feature/issue-<N>-" || true
git remote show origin | sed -n 's/.*HEAD branch: //p'
```

Stop before implementation if:

- the repo is dirty and the issue is not explicitly a repo-hygiene / landing task
- acceptance criteria are still unclear or untestable
- branch cleanup is in scope and linked worktrees for old branches still exist
- the branch flow does not start from the current default branch
- after `git worktree prune`, a different-issue `feature/issue-...` branch exists in a sibling worktree and this BUILDER does not have a `WORKTREE_PATH` for this issue (HARD STOP; route back to CONDUCTOR)

If branch cleanup is in scope, run the local helper in dry-run mode first:

```bash
scripts/cleanup-redundant-local-branches.sh branch-name [branch-name ...]
```

## Step 1.75: Cross-Repo Target Detection

If the tracking issue lives in `{YOUR_REPO}` but implementation belongs in a different repo:

**Target repos:**
- `{YOUR_APP_REPO}` — iOS app
- `{YOUR_IOS_COMPANION_REPO}` — iOS companion
- `{YOUR_BRAND_ASSET_REPO}` — brand assets
- `{YOUR_WEBSITE_REPO}` — website
- `{YOUR_INFRA_REPO}` — LLC infrastructure

**Check out the target repo:**

```bash
cd /path/to/target-repo
default_branch="$(git remote show origin | sed -n 's/.*HEAD branch: //p')"
git checkout "$default_branch" && git pull origin "$default_branch"
git checkout -b feature/issue-<N>-short-description
```

Then proceed to Step 2 with the **target repo** as your working directory.

## Step 2: Create feature branch

```bash
default_branch="$(git remote show origin | sed -n 's/.*HEAD branch: //p')"
git checkout "$default_branch"
git pull origin "$default_branch"
git checkout -b feature/issue-<N>-short-description
```

Branch naming: `feature/issue-N-short-description`

## Step 2.5: Brand Asset Check

If the issue involves brand assets (fonts, icons, images, logos, audio):

1. Check `{YOUR_BRAND_ASSET_PATH}` for the required assets
2. Verify each asset exists in the canonical source
3. If an asset is referenced in brand-kit.md but missing from {YOUR_BRAND_ASSET_REPO}:
   - **STOP implementation**
   - Report: "Asset X is defined in brand-kit.md but not present in {YOUR_BRAND_ASSET_REPO}"
   - Route to brand-asset-pipeline for onboarding
4. Only source brand assets from {YOUR_BRAND_ASSET_REPO} — never from external URLs

## Step 3: Implement changes

- Map every change to explicit acceptance criteria
- Keep changes minimal and directly tied to scope
- Do not broaden scope to adjacent improvements

## Step 3.25: Optional `/fleet` execution

Use `/fleet` only when the current issue naturally splits into parallel tracks.

Rules:

- Run it only after Step 1.5 preflight and Step 2 branch creation.
- Give each track explicit files or directories to own.
- Declare dependencies and validation in the prompt.
- Do **not** use `/fleet` if multiple tracks need the same file, the work is strictly linear, or the prompt would mix multiple issues or roles.

`/fleet` accelerates Step 3 execution only; it does not change the one-issue scope, review requirements, or later Verify Loop.

## Step 3.5: Verify Loop (Ralph Loop)

Run an autonomous test→iterate→ship cycle before committing:

1. Run the project's lint and test commands.
2. If all pass → proceed to Step 4 (commit).
3. If any fail → read the error output, fix the root cause, re-run.
4. Repeat up to **3 cycles** (fix → re-test → fix → re-test → fix → re-test).
5. If the 4th run still fails → **STOP**. Do not commit. Report to the founder:
   - The failing test/lint output
   - What was attempted in each cycle
   - A recommendation to investigate manually

```bash
# Example cycle (adapt commands to the target project)
npm run lint && npm run test   # or: godot --headless --script ..., swift build, etc.
```

Only proceed to Step 4 when all tests and lints pass.

## Step 4: Commit with Conventional Commits

```bash
git add -A
git commit -m "feat(scope): description (#N)

Closes {YOUR_REPO}#N

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

## Step 5: Create PR

```bash
gh pr create \
  --repo <target-repo> \
  --title "feat(scope): description" \
  --body "## Summary
<what changed and why>

## Acceptance Criteria
- [ ] <from issue>

## Test Evidence
<build output, screenshots, etc.>

## Risk Notes
<any risks>

Closes {YOUR_REPO}#N"
```

For cross-repo PRs (e.g., in {YOUR_PRODUCT_SLUG}, {YOUR_BRAND_ASSET_REPO}), use the full reference:
`Closes {YOUR_REPO}#N`

## Step 5.25: App Reach gate

**Applies to every iOS app submission (new app and every subsequent version).**

This gate is required by the App Reach policy (`docs/policy/app-reach.md`, {YOUR_REPO}#716). Founder directive: 2026-04-26.

Before uploading to TestFlight, verify all three checks. Each check must either confirm the policy default is met **or** cite an exception in the spec's `Platform & Technical Notes` section:

- [ ] **Deployment target** — `project.yml` deployment target is the lowest-safe target for the current SDK (today: iOS 17.0), **or** the spec's `Platform & Technical Notes` contains a one-line cited exception (e.g., `exception: FoundationModels requires 26.0`).
- [ ] **Device family** — `TARGETED_DEVICE_FAMILY` includes both `1` and `2` (iPhone + iPad) and App Store Connect has "Make this app available on Mac" enabled, **or** the spec's `Platform & Technical Notes` contains a cited exception.
- [ ] **Hard dependency fallback** — if the app uses a hardware/OS feature not available on all devices at the declared minimum OS (Apple Intelligence, LiDAR, Action Button, etc.), the spec links to a fallback/empty-state spec; **or** the spec records `N/A — <FeatureName> available on all devices at iOS <version>`.

Failing any check **blocks the upload**. Passing requires either the default state or a cited exception — no uncited exceptions are permitted.

See `docs/policy/app-reach.md` for the full policy, citation format, and worked examples.

## Step 5.5: iOS release gate — TestFlight first

**Applies to every iOS release of any {YOUR_ORG_NAME} app (1.0 and every subsequent version).**

This gate is required by the TestFlight-first release policy (spec `specs/2026-04-testflight-first-release-policy-v1-spec.md`, {YOUR_REPO}#700). Founder directive: 2026-04-26.

### §4.1 Mandatory gate

For every iOS release:

1. Build is uploaded to App Store Connect (via `xcodebuild -exportArchive … destination: upload` or `xcrun altool`).
2. Build is enabled for **internal TestFlight** distribution (founder + team).
3. Build soaks for at least the minimum window (§4.2 table below).
4. Smoke-test checklist (§4.3) is run against the TestFlight build on a **real device**.
5. **`reviewer-qa-gate` skill PASS** (mandatory; see `.github/skills/reviewer-qa-gate/SKILL.md`). Catches HUD/visual/accessibility regressions that smoke-test misses.
6. Only after smoke-test passes AND `reviewer-qa-gate` returns PASS may the build be promoted to App Store Review via the per-app submit script (`{YOUR_PRODUCT_SLUG}_submit.py`, `veilsort_submit.py`, etc.).

### §4.2 Minimum soak window

| Release size | Minimum soak | Notes |
|---|---|---|
| Patch (x.y.Z) | 24 hours | Catches obvious regressions across one natural-use cycle |
| Minor (x.Y.0) | 48 hours | Surface new-feature interactions across multiple sessions |
| Major (X.0.0) | 72 hours | Maximum risk; covers a weekend of natural-use testing |
| First submission of a new app | 72 hours | First TestFlight upload exposes onboarding/icon/privacy bugs |

Release size is determined by `MARKETING_VERSION` SemVer change. Record the soak-start timestamp and release size in the PR body.

### §4.3 Smoke-test checklist (real device, not simulator)

Run before promoting to App Store Review. Record pass/fail in the tracking issue.

```
[ ] Install on real device via TestFlight (not simulator)
[ ] Cold-launch from Springboard succeeds without crash
[ ] First-run onboarding completes (or skips correctly on repeat launch)
[ ] Core happy-path flow (product-specific — see repo QA doc)
[ ] App returns from background without crash or state loss
[ ] Settings screen reachable and key toggles persist
[ ] No new console errors of severity Error or higher
[ ] Privacy/permissions prompts appear at expected moments only
[ ] No external network calls beyond what PrivacyInfo.xcprivacy declares
[ ] Quit and re-launch: persistence works (history, settings, progress)
```

Each iOS repo MAY add product-specific items; the above is the minimum.

If any item fails: reject promotion, fix → new build → soak window restarts.

### §4.5 Hotfix override

For genuine emergencies (e.g., live-app crash on launch affecting all users), the founder may waive the soak window. Override must be:

- **Explicit in writing** in the tracking issue before promotion.
- Limited to that single release.
- Recorded in the PR body and automation-registry receipt.

No agent may waive the soak window without explicit founder approval.

### Godot release variant (iOS, macOS, web)

**Applies to every Godot-engine release of any {YOUR_ORG_NAME} product on iOS, macOS, or web.**

This gate is required by the Godot soak policy (`specs/2026-05-godot-testflight-soak-policy-v1-spec.md`, {YOUR_REPO}#943). The SwiftUI/SpriteKit gate above continues to govern non-Godot iOS releases unchanged.

**Soak windows** — follow `specs/2026-05-godot-testflight-soak-policy-v1-spec.md §4.2`. Windows apply uniformly across all three platforms.

**{YOUR_PRODUCT} pre-upload automated gate (iOS)**

Before exporting and uploading any {YOUR_PRODUCT} iOS build to TestFlight, run from the {YOUR_PRODUCT_SLUG} repo root:

```bash
cd /path/to/{YOUR_PRODUCT_SLUG} && godot --path godot/ -- --combo-qa
```

The command must **exit 0** (all 3 tier screenshots captured: TIER_2, TIER_3, TIER_4). A non-zero exit **blocks the upload**. This verifies the combo state machine is functional end-to-end in the exact source being shipped. Typical runtime: ~2 min on macOS with a display (requires GUI — do not pass `--headless`).

**Distribution mechanism per platform:**

- **iOS:** upload to App Store Connect; enable internal TestFlight; record soak-start timestamp in PR body.
- **macOS (MAS):** upload to App Store Connect; enable internal TestFlight; record soak-start timestamp in PR body.
- **macOS (DMG):** provide a signed, notarized DMG to the founder-internal channel; record soak-start timestamp in PR body.
- **web:** deploy to the direct-host preview URL; share with founder for internal soak; record soak-start timestamp in PR body.

**Smoke-test checklists** — run the applicable checklist before promotion:

- **iOS:** `specs/2026-05-godot-testflight-soak-policy-v1-spec.md §4.3.A`
- **macOS (MAS or DMG):** `specs/2026-05-godot-testflight-soak-policy-v1-spec.md §4.3.B`
- **web:** `specs/2026-05-godot-testflight-soak-policy-v1-spec.md §4.3.C`

Record pass/fail for every checklist item in the tracking issue. Any failed item rejects promotion; fix → new build → soak window restarts. Hotfix override follows the same clause as §4.5 above (founder explicit approval required).

---

## Step 6: Knowledge Checkpoint

Before writing the Ship/Revise recommendation, store any learnings from this implementation:

```
For each of these, call store_memory if a new fact was discovered:
- Build commands or flags needed
- Error resolutions (what broke, what fixed it)
- CI/test gotchas specific to this repo
- Procedures that were refined during implementation
```

This ensures the next session doesn't re-discover the same things.

## Step 6.5: Rubber-duck review (mandatory)

**This step is blocking — do not proceed to Step 7 without it.**

Run a rubber-duck review of the implementation before declaring done. The rubber duck is an independent critic in a separate context window and catches blind spots the author misses.

```
task(agent_type="rubber-duck", description="Pre-ship review", prompt="
Review the implementation of foculoom-project#{issue_number} before it is presented to the founder for sign-off.

Context:
- Issue: {issue_title}
- PR: #{pr_number} in {repo}
- Acceptance criteria: {acceptance_criteria}

Scope of changes: {brief_description_of_changes}

Please check:
1. Do the changes actually satisfy each acceptance criterion?
2. Are there any logic errors, edge cases, or off-by-one issues?
3. Are there any missing file updates (e.g., stale path refs, missed call sites)?
4. Any security or data-loss risks introduced?

Bugs and logic errors only — no style comments.
")
```

**Exit rules:**
- **No blocking findings** → proceed to Step 7 (`Ship ✅`)
- **Blocking findings** → fix them, re-run affected tests/CI, then re-run Step 6.5 once more
- **Non-blocking findings** → document them in the PR body under `## Known limitations / follow-ups`, then proceed to Step 7

The rubber-duck result (pass/fail + key findings) MUST appear in the PR body or the Ship/Revise recommendation output.

## Step 7: Ship/Revise recommendation

Output recommendation:
- **Ship ✅** if: all acceptance criteria met, tests pass, no known risks
- **Revise ❌** if: missing criteria, test failures, open questions

## Common mistakes to avoid

- Never push directly to the current default branch
- Never auto-merge — founder review required
- Never start implementation from `GITHUB_ISSUE: TBD`
- Never ignore a dirty checkout or lingering linked worktrees when the issue
  expects clean branch creation or branch cleanup
- Always include `Closes {YOUR_REPO}#N` in PR body
- Always include the Co-authored-by trailer for Copilot commits
- Always use `Closes {YOUR_REPO}#N` (full cross-repo reference) — shorthand `#N` won't auto-close from other repos
- When implementing in a different repo, create the PR in that repo, not in {YOUR_REPO}
- For cross-repo work, cd to the target repo before creating the feature branch
