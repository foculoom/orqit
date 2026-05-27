---
name: risk-review
description: Structured pre-spec risk assessment for ideas touching health, privacy, children, finance, or legal claims. Use this before writing specs for sensitive product ideas.
tier: premium
---

# Risk Review

## Model

- **Preferred:** `claude-opus-4.7`
- **Cost-tier fallback:** `claude-sonnet-4.6` + `--effort xhigh` + rubber-duck — see `/fallback-mode`
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

Perform a structured risk assessment before any spec is written for a sensitive idea.

## When to use

Run this skill when an idea touches ANY of these categories:
- **Health / wellness / medical** — symptom checkers, diagnosis aids, doctor report analysis, menopause guidance
- **Children / family** — content for minors, parental data, child-directed ads restrictions (COPPA)
- **Privacy-sensitive personal data** — offline-only claims, local-only storage, biometric or behavioral data
- **Finance / money / tax** — business management, invoicing, tax computation, financial advice
- **Legal / patent / trademark** — patent-pending claims, trademark registration, regulatory compliance
- **AI-generated content** — claims about AI accuracy, AI-generated medical/legal/financial advice

## Step 1: Identify risk categories

For each risk category that applies, flag it:

```
RISK CATEGORIES:
- [ ] Health / Medical
- [ ] Children / Family (COPPA)
- [ ] Privacy / Data
- [ ] Finance / Tax
- [ ] Legal / IP
- [ ] AI Accuracy Claims
```

## Step 2: Red flags

List specific risks. Examples:
- "Document Analyzer interprets medical test results — this borders on medical advice"
- "Focus Score stores behavioral data — CCPA/GDPR personal data"
- "Small Business does tax computation — incorrect calculations = liability"
- "itsaboutme targets a vulnerable population — menopause health advice"

## Step 3: Claims to avoid

List marketing or product claims that would create legal exposure:
- "Never claim diagnostic accuracy for health tools"
- "Never claim tax advice — always disclaim 'not tax/legal advice'"
- "Never claim patent-pending without actual patent application"
- "Never say 'private' or 'secure' without specific technical backing"

## Step 4: Required founder approvals

Flag decisions that must be made before spec work begins:
- Regulatory scope (which jurisdictions?)
- Disclaimer strategy (in-app, store listing, ToS?)
- Data handling commitment (local-only? encrypted? no telemetry?)
- Age gate requirements (under 13 restrictions?)

## Step 5: Route to next artifact

Based on risk level, recommend the safe next step:

| Risk level | Next artifact |
|-----------|---------------|
| Low (utility, no sensitive data) | `@planner` → spec |
| Medium (privacy, children, or financial) | `@planner` with risk brief → disclaimers → spec |
| High (medical, legal, or patent claims) | `@planner` with risk brief → **Founder must approve scope** → spec |
| Blocker (regulated industry, unclear legality) | `@planner` — recommend **Defer** until legal review |

## Output format

```
RISK REVIEW: [Idea Name]

Risk categories: [list]
Risk level: Low / Medium / High / Blocker

Red flags:
1. ...

Claims to avoid:
1. ...

Required founder approvals:
1. ...

Recommended next: [agent or action]

## Sources
- [Statute/guideline title](URL) — underlying statute or guideline for each regulatory claim
  (Required for regulatory claims: e.g., 16 CFR § 312 for COPPA,
   App Store Guideline 5.1.1, FTC guidance URL, KOSA text URL)
(or: - NONE if no regulatory/external claims)
```
