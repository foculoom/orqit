<img src="assets/orqit-icon.png" width="128" alt="orqit icon" />

# orqit

Copilot CLI workflow orchestration plugin for individual developers.

## Install

```
copilot plugin install foculoom/orqit
```

## What's included

- **Agents:** conductor, planner, builder, reviewer

## Skills

| Skill | Description |
| --- | --- |
| `5-whys` | Structured 5-whys postmortem template for severity-gated mistakes |
| `automation-registry` | Living registry of automatable vs. truly manual platform operations |
| `dev-session` | Unified session lifecycle for starting, resuming, monitoring, and ending work |
| `fallback-mode` | Cost-tier step-down mode with extra verification discipline |
| `kids-safety` | Per-feature COPPA/KOSA checklist for any kids product |
| `market-scan` | Quick market and feasibility scan for product ideas before spec investment |
| `model-audit` | Weekly refresh trigger for the Model Routing Matrix |
| `new-feature` | Intake a new feature idea through the PLANNER decision process |
| `qa-capture-ios` | Exact simulator-only QA capture procedure for iOS apps on macOS Simulator |
| `qa-validate` | QA validation workflow for any product |
| `release-post` | Generate a release notes HTML page and tweet draft after an App Store version goes READY_FOR_SALE |
| `reviewer-qa-gate` | Default REVIEWER on-device QA pass after BUILDER validation or TestFlight upload |
| `risk-review` | Structured pre-spec risk assessment for sensitive product ideas |
| `ship-issue` | Implement a single scoped GitHub Issue and prepare a PR |
| `status` | Unified pipeline audit and founder digest |
| `usage` | Check current GitHub Copilot premium request burn |

## License

MIT — Copyright (c) 2026 Foculoom LLC

## Configuration

After install, the copied skills and agents use uppercase template markers such as `{YOUR_REPO}`, `{YOUR_WEBSITE_REPO}`, `{YOUR_BRAND_ASSET_PATH}`, `{YOUR_ORG}`, and `{YOUR_DTWIN_DB_PATH}`.

Replace the placeholders that apply to your setup:

| Placeholder | Replace with |
|-------------|-------------|
| `{YOUR_REPO}` | Your tracking or primary project repo (for example, `owner/repo`) |
| `{YOUR_WEBSITE_REPO}` | Your website repo |
| `{YOUR_BRAND_ASSET_PATH}` | Your canonical brand-asset path or repo path |
| `{YOUR_ORG}` / `{YOUR_ORG_NAME}` | Your GitHub org or studio name |
| `{YOUR_PRODUCT}` / `{YOUR_APP_NAME}` | Your product or app name |
| `{YOUR_DTWIN_DB_PATH}` | Your canonical `.dtwin` database path, or remove the reference |
| `{YOUR_PINNED_COMMIT}` / `{YOUR_CONTRACT_HASH}` | Your recorded validator commit and contract hash where required |

These are intentional template markers for customization. The plugin works with the placeholders left in place for reference, but replacing them makes the docs and agent prompts match your own infrastructure.
