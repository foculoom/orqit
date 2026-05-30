---
name: market-scan
description: Quick market and feasibility scan for product ideas before spec investment. Use this for R&D bets, platform plays, or ideas where competition and complexity are unclear.
tier: extra-premium
---

# Market Scan

## Model

- **Preferred:** `gpt-5.5`
- **Cost-tier fallback:** `claude-sonnet-4.6` + `--effort xhigh` + rubber-duck — see `/fallback-mode`
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

## Escalation — legal/regulatory scans

If a scan is primarily legal or regulatory (for example patent landscape, COPPA compliance, KOSA, tax, or other policy/compliance interpretation), the executing agent must escalate that specific scan to `claude-opus-4.8` before final recommendations.

Perform a quick external scan before investing in a full spec. Best for R&D bets, platform plays, or ideas where the competitive landscape and build complexity are unclear.

## When to use

Run this skill when an idea is NOT an obvious tiny win and needs validation:
- **R&D / platform bets** — Maps, Focus Score, Repo Quality, AI-powered features
- **Crowded categories** — many existing competitors (e.g., habit trackers, to-do apps)
- **Platform-dependent** — relies on specific APIs, data sources, or hardware capabilities
- **Unclear differentiation** — not yet clear why your version would win

## Step 1: Market landscape

Identify 3–5 existing competitors or comparable products using web search. For each:
- What they do well
- What they do poorly
- Pricing model (free, freemium, paid)

## Step 2: Platform constraints

Identify restrictions that affect feasibility:
- App Store rules (e.g., offline data size limits, age-gating, health data APIs)
- API availability and cost (e.g., map tile providers, LLM pricing)
- Device capabilities (e.g., on-device ML, storage, battery impact)

## Step 3: Differentiators

What would make your product uniquely better?
- Mission alignment (example: focus, calm tech, screen-time awareness — replace with your product's mission differentiators)
- Privacy-first / offline-first design
- Kid-friendly or family-safe angle
- Integration with your existing products

## Step 4: Build complexity estimate

Rate as one of: **Easy** / **Medium** / **Hard** / **R&D**
Include a 1–2 sentence justification referencing specific technical challenges.

## Step 5: Major unknowns

List 2–3 things that must be resolved before a spec makes sense.

## Step 6: Recommendation

Choose one and provide reasoning:
- **Proceed** — hand off to `@planner` for prioritization
- **Defer** — not now, with specific reason (timing, dependency, market)
- **Kill** — fundamentally unviable, with specific reason
- **More research** — list the specific questions that need answers

## Output format

```
MARKET SCAN: [Idea Name]

Competitors:
1. [Name] — [what they do, strengths, weaknesses]
2. ...

Platform constraints:
- ...

Differentiators:
- ...

Build complexity: [Easy/Medium/Hard/R&D] — [justification]

Major unknowns:
1. ...

Recommendation: [Proceed/Defer/Kill/More research] — [reasoning]

## Sources
- [Title](URL) — what platform constraint, App Store rule, or API claim this anchors
(or: - NONE if all claims are general knowledge not requiring a reputable-source citation)
```
