# Filler Agent Protocol

The filler agent discovers tool metadata and produces a filled audit prompt. The steps adapt per mode — but the output is always the same: a self-contained prompt with no placeholders.

## Step 1 — Discover the Tool

**Container mode:** run via `docker exec <container-name>`
**Local mode:** run with env vars set — `env KEY=VALUE <tool-name> --help`
**Worktree mode:** run from inside the temp dir — `cd <temp-dir> && <tool-name> --help`

Commands to run (adjust exec prefix per mode):
1. `<tool-name> --help` — top-level help and subcommand list
2. `<tool-name> --version` — confirm the version
3. For each subcommand: `<tool-name> <subcommand> --help` — flags and usage

## Step 2 — Discover the Environment

**Container mode:** adapt to the container's package manager:
- apt (Debian/Ubuntu): `dpkg --get-selections | grep -v deinstall | head -30`
- apk (Alpine): `apk list --installed | head -30`
- Homebrew: `brew list --formula`
- No package manager: `ls /usr/local/bin/ | head -30`

Also run: `which <tool-name>`, `echo $PATH`

**Local mode:** describe the env var isolation context — e.g., `COMMITMUX_DB=/tmp/audit-xxx/db.sqlite3`. List the env vars and their temp paths. No package discovery needed.

**Worktree mode:** describe the temp dir contents — `ls <temp-dir>`. Note the source path it was copied from.

## Step 3 — Construct Audit Areas

Build a numbered list of areas with exact commands (not placeholders). Use the exec prefix appropriate for the mode:

1. **Discovery** — `--help`, `--version`, `<subcommand> --help` for each
2. **Setup / onboarding** — first-run commands, initialization, status checks
3. **Core feature** — primary command with all flag combinations
4. **Data / tracking** — if the tool collects data: generate activity, wait, query
5. **Explanation / detail** — per-item drill-down with valid and invalid inputs
6. **Diagnostics** — doctor/health/debug subcommands with all flags
7. **Destructive / write operations** — remove/delete/reset (safe in sandbox — no --dry-run needed unless the tool provides it as a UX signal worth testing)
8. **Edge cases** — no args, unknown subcommand (`blorp`), invalid flags, invalid enum values
9. **Output review** — table alignment, colors, header/footer clarity, terminology consistency

## Step 4 — Write the Filled Prompt

Produce a self-contained audit prompt with all variables substituted. Write to `docs/cold-start-audit-prompt.md`. The filled prompt must follow this structure:

```
# Cold-Start UX Audit Prompt

**Metadata:**
- Audit Date: <YYYY-MM-DD>
- Tool: <tool-name>
- Tool Version: <version from --version>
- Sandbox mode: container | local | worktree
- Sandbox: <container-name | env vars | temp dir path>
- Exec prefix: `<exec-prefix>`

---

You are performing a UX audit of <tool-name> — a tool that <description from help>.
You are acting as a **new user** encountering this tool for the first time.

Sandbox: <sandbox context>

Run all commands using: `<exec-prefix> <command>`

## Audit Areas

<filled audit areas with exact commands>

Run ALL commands listed. Do not skip areas.
Note exact output, errors, exit codes, and behavior at each step.
Describe color usage (e.g. "package names appear in bold white, tier labels in green/yellow/red").

You are not trying to find every possible issue — you are discovering what friction a new user would naturally encounter by following the help text and trying obvious commands.

## Findings Format

For each issue found, use:

### [AREA] Finding Title
- **Severity**: UX-critical / UX-improvement / UX-polish
- **What happens**: What the user actually sees
- **Expected**: What better behavior looks like
- **Repro**: Exact command(s)

Severity guide:
- **UX-critical**: Broken, misleading, or completely missing behavior that blocks the user
- **UX-improvement**: Confusing or unhelpful behavior that a user would notice and dislike
- **UX-polish**: Minor friction, inconsistency, or missed opportunity for clarity

## Report

- Group findings by area
- Include a summary table at the top: total count by severity
- Write the complete report to docs/cold-start-audit.md using the Write tool

IMPORTANT: Run ALL commands via `<exec-prefix> <command>`.
Do not bypass the sandbox — do not run <tool-name> against production state.
```

**CRITICAL — Do NOT add regression verification sections:**

The prompt template above is complete. DO NOT append:
- Regression verification sections
- Checklists of known issues to validate
- "Verify the following fixed behaviors" sections
- Comparisons to previous audit rounds
- Any content that gives the audit agent prior knowledge of what issues to look for

Every audit must be pure discovery with zero context. If old issues regressed, they surface through organic rediscovery (agent encounters the same friction) not through explicit verification checklists.

## Step 5 — Validate the Filled Prompt

Before returning, check for unfilled placeholders:

```bash
grep -o '{{[^}]*}}' docs/cold-start-audit-prompt.md
```

If any placeholders remain, report them as errors and do NOT launch the audit agent. Common issues:
- `{{TOOL_NAME}}` — forgot to substitute tool name
- `{{EXEC_PREFIX}}` — forgot to substitute docker exec or env prefix
- `{{AUDIT_AREAS}}` — didn't construct the command list

**Fix:** Re-run filler agent or manually edit the prompt file.

## Agent Type Preference

Use custom agent types when launching agents. These provide:
- Tool-level enforcement (filler cannot execute in sandbox, audit cannot bypass isolation)
- Behavioral instructions in the type definition
- Better observability in claudewatch metrics (separate success rates per agent type)

**Filler agent:** Use `subagent_type: filler` when spawning the metadata discovery agent.
**Audit agent:** Use `subagent_type: audit` when spawning the sandbox execution agent.

If these custom types fail to load, automatically fall back to `subagent_type: general-purpose` with the full prompt content from `prompts/agents/filler.md` or `prompts/agents/audit.md`.

Custom agent types are **optional** — users can install them for enhanced observability:
```bash
mkdir -p ~/.claude/agents
ln -sf $(pwd)/prompts/agents/filler.md ~/.claude/agents/filler.md
ln -sf $(pwd)/prompts/agents/audit.md ~/.claude/agents/audit.md
```

## Launching the Filler Agent

Launch as a background Task agent:

- subagent_type: `filler` (falls back to `general-purpose` if not installed)
- run_in_background: true
- prompt: Instructions with `{{TOOL_NAME}}`, `{{SANDBOX_MODE}}`, `{{EXEC_PREFIX}}`, `{{SANDBOX_CONTEXT}}`, `{{OUTPUT_PATH}}` substituted
