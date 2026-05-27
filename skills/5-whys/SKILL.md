---
name: 5-whys
description: Structured 5-whys postmortem template for severity-gated mistakes, with canonical output marker and required destinations.
tier: basic
---

# 5-Whys Postmortem

## Model

- **Preferred:** `claude-haiku-4.5`
- **Cost-tier fallback:** `claude-sonnet-4.6` for severity gates (i)-(iii) or disputed root cause
- **Source of truth:** Model Routing Matrix in `.github/skills/dev-session/SKILL.md`

`.github/copilot-instructions.md` §5-whys mistake protocol is authoritative for trigger logic and severity gates.

## Template

Use this structure:

1. **Problem statement**
2. **Why 1**
3. **Why 2**
4. **Why 3**
5. **Why 4**
6. **Why 5**
7. **Root cause**
8. **Action items** (specific, testable, owner/trigger where possible)

## Output block (required)

```
FIVE_WHYS_POSTMORTEM: <root cause>
Problem statement: <...>
Why 1: <...>
Why 2: <...>
Why 3: <...>
Why 4: <...>
Why 5: <...>
Root cause: <...>
Action items:
- <...>
```

## Output destinations (both required)

- GitHub issue comment — must begin with `FIVE_WHYS_POSTMORTEM:`
- `store_memory` fact/content — must begin with `FIVE_WHYS_POSTMORTEM:`
