# ai-ux-audit

AI-driven UX auditing for CLI tools using containerized sandboxes.

A setup agent reads your project's help output and README, then populates a structured prompt template. An audit agent executes every subcommand as a simulated new user inside a Docker container, producing a severity-tiered findings report with reproduction steps.

## Why

You can't UX-test your own tool — you know too much. AI agents have no prior context, making them ideal surrogate new users. Run them in a disposable container with full access and they'll find the friction you can't see.

## How

1. Start a sandbox container with your tool installed and bind-mounted
2. A **setup agent** inspects `--help`, the README, and available subcommands to fill the audit template
3. An **audit agent** executes the filled template inside the container as a new user
4. Findings are written as a structured report grouped by area with severity tiers

## Prompts

- [`prompts/audit-template.md`](prompts/audit-template.md) — Prompt template for the audit agent. Variables are filled by the setup agent.
- [`prompts/setup-agent.md`](prompts/setup-agent.md) — Prompt for the setup agent that inspects the project and fills the template.

## Severity Tiers

| Tier | Meaning |
|------|---------|
| **UX-critical** | Blocks or confuses a new user — they'd give up |
| **UX-improvement** | Friction that slows users down but doesn't block them |
| **UX-polish** | Minor rough edges — cosmetic or nice-to-have |

## Container Setup

Scope Claude Code permissions at the project level so agents can exec into the sandbox:

```json
// .claude/settings.json
{
  "permissions": {
    "allow": ["Bash(docker exec my-sandbox*)"]
  }
}
```

## License

MIT
