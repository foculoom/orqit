---
name: ship-issue
description: Implement a single scoped GitHub Issue and prepare a PR. Use this when an issue has acceptance criteria defined and is ready for implementation. Follows the BUILDER workflow.
tier: standard
---

# Ship Issue (BUILDER)

## Model

- **Preferred:** `claude-sonnet-4.6`
- **Premium-exhausted fallback:** `/model auto` → `claude-sonnet-4.5`; `gpt-5.3-codex` for pure-code edits — see `/fallback-mode`
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

Track-C-style references: see your project's issue tracker for worktree collision precedents.

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
gh issue view <N> --repo {your-github-org}/{your-main-repo} --json title,body,labels
```

Verify:
- Acceptance criteria exist and are testable
- Scope is clear (if not, stop and ask for clarification)
- Target repo is identified
- The handoff points at a real `{your-github-org}/{your-tracking-repo}#N`; if it still says
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

If the tracking issue lives in your main repo but implementation belongs in a different repo, check out the target repo before creating the feature branch.

**Add your studio's target repos here** (e.g., your iOS app repo, brand assets repo, website repo, LLC infrastructure repo). Use the full `owner/repo` reference for each.

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

1. Check `{your-brand-assets-repo}/assets/` for the required assets
2. Verify each asset exists in the canonical source
3. If an asset is referenced in brand-kit.md but missing from `{your-brand-assets-repo}`:
   - **STOP implementation**
   - Report: "Asset X is defined in brand-kit.md but not present in `{your-brand-assets-repo}`"
   - Route to brand-asset-pipeline for onboarding
4. Only source brand assets from `{your-brand-assets-repo}` — never from external URLs

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

Closes {your-github-org}/{your-tracking-repo}#N

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

Closes {your-github-org}/{your-tracking-repo}#N"
```

For cross-repo PRs (in a companion repo), use the full reference:
`Closes {your-github-org}/{your-tracking-repo}#N`

## Step 5.25: App Reach gate

**Applies to every iOS app submission (new app and every subsequent version).**

This gate is required by the App Reach policy (`docs/policy/app-reach.md`). Verify the policy document exists in your repo and replace the reference with your own platform-reach requirements.

Before uploading to TestFlight, verify all three checks. Each check must either confirm the policy default is met **or** cite an exception in the spec's `Platform & Technical Notes` section:

- [ ] **Deployment target** — `project.yml` deployment target is the lowest-safe target for the current SDK (today: iOS 17.0), **or** the spec's `Platform & Technical Notes` contains a one-line cited exception (e.g., `exception: FoundationModels requires 26.0`).
- [ ] **Device family** — `TARGETED_DEVICE_FAMILY` includes both `1` and `2` (iPhone + iPad) and App Store Connect has "Make this app available on Mac" enabled, **or** the spec's `Platform & Technical Notes` contains a cited exception.
- [ ] **Hard dependency fallback** — if the app uses a hardware/OS feature not available on all devices at the declared minimum OS (Apple Intelligence, LiDAR, Action Button, etc.), the spec links to a fallback/empty-state spec; **or** the spec records `N/A — <FeatureName> available on all devices at iOS <version>`.

Failing any check **blocks the upload**. Passing requires either the default state or a cited exception — no uncited exceptions are permitted.

See `docs/policy/app-reach.md` for the full policy, citation format, and worked examples.

## Step 5.5: iOS Release Gate

**Applies to every iOS release (1.0 and every subsequent version).**

Establish your studio's TestFlight soak policy before promoting any iOS build to App Store Review. Adapt this gate to match your platform release policy.

### §4.1 Mandatory gate

For every iOS release:

1. Build is uploaded to App Store Connect (via `xcodebuild -exportArchive … destination: upload` or `xcrun altool`).
2. Build is enabled for **internal TestFlight** distribution (founder + team).
3. Build soaks for at least the minimum window (§4.2 table below).
4. Smoke-test checklist (§4.3) is run against the TestFlight build on a **real device**.
5. **`reviewer-qa-gate` skill PASS** (mandatory; see `.github/skills/reviewer-qa-gate/SKILL.md`). Catches HUD/visual/accessibility regressions that smoke-test misses.
6. Only after smoke-test passes AND `reviewer-qa-gate` returns PASS may the build be promoted to App Store Review via your app's submit script.

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
- Always include `Closes {your-github-org}/{your-tracking-repo}#N` in PR body
- Always include the Co-authored-by trailer for Copilot commits
- Always use `Closes {your-github-org}/{your-tracking-repo}#N` (full cross-repo reference) — shorthand `#N` won't auto-close from other repos
- When implementing in a different repo, create the PR in that repo, not in the tracking repo
- For cross-repo work, cd to the target repo before creating the feature branch
