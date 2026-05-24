# orqit

Copilot CLI workflow orchestration plugin for solo founders.

## Install

```
copilot plugin install foculoom/orqit
```

## What's included

- **Agents:** conductor, planner, builder, reviewer
- **Skills:** 5-whys, automation-registry, dev-session, fallback-mode, model-audit, new-feature, qa-validate, reviewer-qa-gate, risk-review, ship-issue, status, usage

## License

MIT — Copyright (c) 2026 Foculoom LLC

## Configuration

After install, the agent files in `agents/` contain `{studio-name}`, `{your-github-org}`, `{your-tracking-repo}`, `{your-brand-assets-repo}`, `{your-mcp-twin-tool}`, `{your-ratified-strategy-issue}`, and `{your-dtwin-path}` placeholders.

Replace these with your studio's values:

| Placeholder | Replace with |
|-------------|-------------|
| `{studio-name}` | Your studio or company name |
| `{your-github-org}` | Your GitHub org or username |
| `{your-tracking-repo}` | Your issue-tracking repo (e.g., `owner/repo`) |
| `{your-brand-assets-repo}` | Your brand assets repo |
| `{your-mcp-twin-tool}` | Your AI twin MCP tool name, or remove the reference |
| `{your-ratified-strategy-issue}` | Your strategy ratification issue number |
| `{your-dtwin-path}` | Your `.dtwin/` path, or remove the reference |

These are intentional template markers for full-power customization — the skills work out of the box without any setup. Agent files have optional placeholders that unlock additional capabilities (digital-twin sync, org-specific tracking) when filled in, but are not required for day-to-day use.
