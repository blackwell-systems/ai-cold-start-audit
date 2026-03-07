---
name: cold-start-audit
description: Simulate a new user in a sandboxed environment to discover CLI tool UX friction. Use when auditing a tool's onboarding experience or running isolation-mode audits.
argument-hint: [mode|preflight|setup|run|report|init|container] <tool-name> [--mode container|local|worktree] [args]
disable-model-invocation: true
allowed-tools: Bash, Read, Write
---

Cold-Start UX Audit: AI agents simulate new users in sandboxed environments.

## Arguments

- `mode <tool-name>`: Analyze the tool's help output and recommend the appropriate sandbox isolation mode. See [Mode Advisory](#mode-advisory-mode-tool-name).
- `preflight <tool-name> --mode <mode> [mode-args]`: Validate all prerequisites before running an audit. Auto-run before `setup` and `run`. See [preflight.md](preflight.md).
- `container build <tool-name> [--dockerfile PATH]`: Auto-increment round number, check for reusable containers/images, build new image if needed, start container. See [container.md](container.md).
- `container list <tool-name>`: Show all images and containers for a tool with their status. See [container.md](container.md).
- `container cleanup <tool-name> [--keep-latest N]`: Remove old containers and images, keeping the latest N rounds. See [container.md](container.md).
- `setup <tool-name> --mode container <container-name>`: Run the filler agent to discover tool metadata from the running container and produce a filled audit prompt at `docs/cold-start-audit-prompt.md`.
- `setup <tool-name> --mode local --env KEY=VALUE [--env KEY=VALUE ...]`: Run the filler agent in local mode. Creates temp dir, sets env vars, discovers tool metadata.
- `setup <tool-name> --mode worktree --dir <source-path>`: Run the filler agent in worktree mode. Copies `<source-path>` to a temp dir, discovers tool metadata.
- `run <tool-name> --mode <mode> [mode-args]`: Full audit. Checks for existing prompt (reuse or regenerate), launches audit agent, writes findings to `docs/cold-start-audit.md`. Same mode args as `setup`.
- `report`: Read and summarize findings from `docs/cold-start-audit.md`, grouped by severity.
- `init <container-name>`: Scaffold `.claude/settings.json` with scoped permissions for the container (container mode only).


## Sandbox Modes

Three modes satisfy the core invariant — **the audit agent must not modify production state** — using different isolation mechanisms:

| Mode | Argument form | When to use |
|---|---|---|
| `container` | `--mode container <container-name>` | Tool has destructive ops (remove, delete, system writes) |
| `local` | `--mode local --env KEY=VALUE [--env KEY=VALUE ...]` | Tool writes only to self-managed state (own DB or config) |
| `worktree` | `--mode worktree --dir <source-path>` | Tool reads/writes files in the current directory |

**Container mode** (default): audit agent prefixes every command with `docker exec <container-name>`. Requires Docker and a running sandbox container.

**Local mode**: audit agent sets env vars that redirect the tool's state to a temp directory. No Docker needed. Example: `--mode local --env COMMITMUX_DB=/tmp/audit-$$/db.sqlite3`.

**Worktree mode**: audit agent runs inside a fresh copy of a directory. Example: `--mode worktree --dir /path/to/project`.

## Prompt Reuse Strategy

When running subsequent audits on the same tool:

**Check for existing prompt first:**
1. Look for `docs/cold-start-audit-prompt.md`
2. If found, read the metadata header to see the previous container name and date
3. Ask user: "Found existing prompt from [date] using container [name]. Options:
   - **Reuse** - Update only container name and date (fast, use when tool commands haven't changed)
   - **Regenerate** - Run filler agent to rediscover everything (use when tool structure changed)"

**Implementation:**
- For reuse: Use Edit tool to update container name throughout and metadata date
- For regenerate: Run the full filler agent as before

This prevents wasteful parallel execution of filler agent when audit agent doesn't need it.

## Permissions

Background agents cannot prompt for tool approval. Without an explicit `allow` rule, every Bash call is denied silently.

### Project-level (recommended, scoped)

Create `.claude/settings.json` in the project repo. Use a wildcard that covers all round numbers so it doesn't need updating each round:

```json
{
  "permissions": {
    "allow": ["Bash(docker exec <tool>-r*)"]
  }
}
```

Example for brewprune: `"Bash(docker exec brewprune-r*)"` covers r1 through r99 without ever needing to update the settings file.

### User-level (broader, needed for background agents)

Ensure `~/.claude/settings.json` includes:

```json
{
  "permissions": {
    "allow": ["Bash", "Read", "Write"]
  }
}
```

Common mistake: `allow_bash: true` is not valid and is silently ignored.

After editing settings, restart the Claude Code session. Settings do not hot-reload.

## Mode Advisory (`mode <tool-name>`)

When the argument is `mode <tool-name>`, analyze the tool's help output and emit an isolation mode recommendation. Run the tool directly on the host (no sandbox yet):

### Step 1 — Discover commands and flags

1. `<tool-name> --help` — collect top-level subcommand list
2. For each subcommand: `<tool-name> <subcommand> --help` — collect flags

### Step 2 — Answer three diagnostic questions

**Q1: Does the tool have destructive operations?**
Look for subcommands or flags that: remove/delete/uninstall/reset/purge system state, write to package managers, modify system config, or send external requests (email, webhooks). If yes → lean toward **container**.

**Q2: Does the tool write only to self-managed state?**
Look for: `--db`, `--config`, `--data-dir`, or env vars that redirect all persistent state to a single path. If the tool's entire blast radius is one file or directory controlled by an env var → lean toward **local**.

**Q3: Does the tool operate on the current directory or file tree?**
Look for: commands that read/write files in `./`, git operations, file processors, or path arguments with no state isolation flag. If the tool's output is files on disk → lean toward **worktree**.

### Step 3 — Emit recommendation

```
Mode recommendation: container | local | worktree

Rationale: [2-3 sentences explaining which diagnostic questions triggered the recommendation]

Confidence: high | medium | low

If medium or low — explain what's ambiguous and how to resolve it.

Suggested commands:
  /cold-start-audit container build <tool-name>  # (if container mode)
  /cold-start-audit run <tool-name> --mode <mode> [mode-specific args]
```

If the tool clearly fits multiple modes (e.g., has both destructive ops AND a self-contained DB), recommend **container** — it's the most conservative isolation.

## Launching the Audit Agent

The audit agent MUST run in a fresh context with zero knowledge of the project. Launch as a background Task agent:

- subagent_type: `audit` (falls back to `general-purpose` if not installed)
- run_in_background: true
- prompt: the filled audit prompt content (read from `docs/cold-start-audit-prompt.md` or pass directly as text)

**Sequencing (when regenerating the prompt):**
1. Run preflight first — auto-validates prerequisites
2. Launch filler agent and **wait for completion**
3. Validate prompt — check for unfilled placeholders
4. **Then** launch audit agent with that prompt content

**Anti-pattern to avoid:**
- Launching filler agent in background + immediately launching audit agent with manually-edited old prompt = wasteful parallel execution
- Either wait for filler to complete, OR skip filler entirely and reuse/adapt the existing prompt

See [filler-agent.md](filler-agent.md) for the full filler agent protocol.

## Severity Tiers

| Tier | Meaning | Action |
|------|---------|--------|
| UX-critical | Broken, misleading, or blocks the user | Fix before next release |
| UX-improvement | Confusing but functional | Prioritize for next sprint |
| UX-polish | Minor friction or inconsistency | Batch into a cleanup PR |

## Supporting Files

- [preflight.md](preflight.md) — Preflight validation checks per mode with fix steps
- [container.md](container.md) — Container lifecycle: build, list, cleanup, Dockerfile pattern
- [filler-agent.md](filler-agent.md) — Filler agent protocol: discovery, environment, audit area construction, prompt writing
