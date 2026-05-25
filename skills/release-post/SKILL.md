---
name: release-post
description: Generate a release notes HTML page and tweet draft after an App Store version goes READY_FOR_SALE. Detect version state via ASC API, generate HTML into the website repo, open a PR, and draft the tweet body.
tier: standard
---

# Release Post

## Model

- **Preferred:** `claude-sonnet-4.6`
- **Cost-tier fallback:** `/model auto`
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

Content generation and tool orchestration at standard tier. Upgrade to Opus only if copy quality is unacceptable on a second pass.

## When to use

Invoke **only** after an App Store version goes `READY_FOR_SALE`. Do **not** invoke on TestFlight build approval or internal distribution — those are soak steps, not release events. The automated path is the ASC detector workflow (`asc-release-detector.yml`). Invoke manually when the detector missed a release or a backfill is needed.

## Scripts

| Script | Repo | Purpose |
|---|---|---|
| `scripts/asc-submit/detect_release.py` | `{YOUR_PROJECT_REPO}` | Poll ASC for version state; output JSON |
| `scripts/release-post/generate_release_post.py` | `{YOUR_PROJECT_REPO}` | Generate release notes HTML from JSON |
| `scripts/release-post/build_tweet.py` | `{YOUR_PROJECT_REPO}` | Build tweet draft from release JSON |
| `scripts/release-post/build_instagram_caption.py` | `{YOUR_PROJECT_REPO}` | Build Instagram caption + hashtags from release JSON |
| `scripts/release-post/build_pr_body.py` | `{YOUR_PROJECT_REPO}` | Generate PR body with tweet + IG caption side-by-side |
| `scripts/release-post/post_tweet.py` | `{YOUR_WEBSITE_REPO}` | Post tweet via X API v2 |

## Manual flow (when automation is unavailable)

> **Prerequisites:** Script paths below reflect this repository's current layout. Adapt each path to match your own infrastructure, repo names, and checkout locations before running the manual flow.

1. **Detect:** `python3 scripts/asc-submit/detect_release.py --app-id <id> [--version-id <id>]`
   - Exit 0 → READY_FOR_SALE. JSON written to stdout. Pipe to a file.
   - Exit 2 → Not yet ready. Check again later.

2. **Generate:** `python3 scripts/release-post/generate_release_post.py --input release.json --product {YOUR_APP_NAME} --output /path/to/{YOUR_WEBSITE_REPO}/{YOUR_APP_NAME}/releases/{version}.html`

3. **Build tweet:** `python3 scripts/release-post/build_tweet.py --input release.json --product {YOUR_APP_NAME} > tweet.txt`

4. **Build Instagram caption:** `python3 scripts/release-post/build_instagram_caption.py --input release.json --product {YOUR_APP_NAME} --version <M.N.P> > ig_caption.txt`
   - Outputs a validated caption (≤2,200 chars, 3–7 hashtags from lane allow-list).
   - Exits non-zero on any deny-list violation (hashtag or privacy-claim).
   - Add `--hashtags "#tag1,#tag2,..."` to override the default 5-tag set.

5. **Build PR body:** `python3 scripts/release-post/build_pr_body.py --input release.json --product {YOUR_APP_NAME} --tweet tweet.txt --ig-caption ig_caption.txt --ig-image-url <jpeg_url> --run-url <actions_url>`
   - Produces a PR body with `### Tweet draft` + `### Instagram caption draft` fenced blocks
     and a `### Instagram image preview` link to the rendered JPEG.

6. **Validate:** In the website repo: `python3 scripts/validate_site.py && python3 scripts/build_site.py`

7. **PR:** Open PR on `{YOUR_WEBSITE_REPO}` with label `release-post`. Review tweet + IG draft in PR body.

8. **Post:** After founder merges the PR, the `release-tweet.yml` workflow fires automatically and calls `post_tweet.py`.

## Tweet format

```
{Product} {version} is live 🎉

{one-line summary of changes}

{App Store URL}
{release notes URL}
```

Keep total ≤280 characters. If whatsNew is null, omit the summary line.

## Credential requirements

| Secret | Repo | Purpose |
|---|---|---|
| `ASC_KEY_ID` | `{YOUR_PROJECT_REPO}` | ASC API auth |
| `ASC_ISSUER_ID` | `{YOUR_PROJECT_REPO}` | ASC API auth |
| `ASC_PRIVATE_KEY` | `{YOUR_PROJECT_REPO}` | ASC API auth (p8 content) |
| `X_CLIENT_ID` | `{YOUR_WEBSITE_REPO}` | X OAuth 2.0 |
| `X_CLIENT_SECRET` | `{YOUR_WEBSITE_REPO}` | X OAuth 2.0 |
| `X_REFRESH_TOKEN` | `{YOUR_WEBSITE_REPO}` | X OAuth 2.0 refresh |
| `WEBSITE_DEPLOY_TOKEN` | `{YOUR_PROJECT_REPO}` | Cross-repo PR creation |

## Handoff boundary

This skill produces a draft PR + tweet text for founder review. **Do not auto-merge or auto-tweet.** Founder merges the PR; the tweet fires via the post-merge workflow.
