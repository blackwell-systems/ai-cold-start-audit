# Background Agent UX Audit Workflow

A repeatable process for running an AI agent inside a sandboxed environment to audit CLI tool UX as a new user.

---

## Prerequisites

### 1. User-level permissions (one-time setup)

Background agents cannot prompt for tool approval. Without an explicit `allow` rule, every `Bash` call is denied and the agent silently fails.

Ensure `~/.claude/settings.json` uses the correct permissions format:

```json
{
  "permissions": {
    "allow": ["Bash", "Read", "Write", "mcp__*"]
  }
}
```

> **Common mistake:** `allow_bash: true` is not a valid Claude Code permission format and is silently ignored.

After editing `~/.claude/settings.json`, **restart your Claude Code session** before launching background agents. Settings are loaded at session start and do not hot-reload.

### 2. Project-level permissions (optional, scoped)

If you want to restrict background agent Bash access to specific commands only (e.g. just docker exec on a named container), add `.claude/settings.json` to the project repo:

```json
{
  "permissions": {
    "allow": ["Bash(docker exec <container-name>*)"]
  }
}
```

> **Note:** Project-level permissions require user approval before they are trusted — they cannot bootstrap themselves. Use user-level permissions for background agents that need to work without interaction.

### 3. Hooks

Broken hooks block all Bash calls for background agents. Before launching agents, verify any `PreToolUse` hooks on `Bash` actually exist on disk:

```bash
ls ~/claude/hooks/  # check referenced hook scripts exist
```

Remove or stub out any missing hooks in `~/.claude/settings.json`.

---

## Quick Path: Use the Skill

If you have the `/cold-start-audit` skill installed (see README), the entire workflow is:

```bash
# Start your sandbox container, then:
/cold-start-audit run <container-name> <tool-name>
```

The skill runs the filler agent and audit agent automatically and writes the report to `docs/cold-start-audit.md`.

The steps below describe the manual workflow for when you want more control or aren't using Claude Code.

---

## The Workflow

### Step 1 — Build and start the sandbox

Build a Docker image with the tool installed in a realistic user environment.
See `sandbox-setup.md` for container design options and Dockerfile patterns.

```bash
docker build -t <image-name> -f <path/to/Dockerfile> .
docker run -d --name <container-name> --rm <image-name> sleep 3600
```

Verify the container is running:

```bash
docker ps --filter name=<container-name>
```

### Step 2 — Generate the audit prompt

Open `prompts/filler-agent-prompt.md`, substitute the three variables (`{{CONTAINER_NAME}}`, `{{TOOL_NAME}}`, `{{OUTPUT_PATH}}`), and paste the prompt into Claude Code (or any AI agent with bash access).

The filler agent will:
1. Run `--help` on every subcommand to discover flags and usage
2. Inspect the container environment (installed packages, PATH)
3. Construct audit areas with exact commands to run
4. Output a fully filled audit prompt — no placeholders remaining

Alternatively, fill `prompt-template.md` manually if you prefer to control the audit areas yourself.

### Step 3 — Launch the audit agent

Start a **new Claude Code session** (or a background agent) and paste the filled prompt from Step 2. A fresh session is important — the agent must have zero context about your tool, simulating a new user.

In Claude Code, you can launch a background agent:

```
"Run this audit in the background as a Task agent" + paste the filled prompt
```

The agent will:
1. Run 30+ commands against the container via `docker exec`
2. Observe output, errors, exit codes, and behavior at each step
3. Compile findings in the structured severity-tiered format
4. Write the final report to the specified output path

### Step 4 — Review findings

Triage by severity:

| Severity | Meaning | Action |
|---|---|---|
| UX-critical | Broken behavior or misleading output | Fix before next release |
| UX-improvement | Confusing but functional | Prioritize for next sprint |
| UX-polish | Minor friction | Batch into a cleanup PR |

---

## Checklist Before Launching Agent

- [ ] `~/.claude/settings.json` has `"allow": ["Bash", "Read", "Write", "mcp__*"]`
- [ ] Session was restarted after last settings change
- [ ] All `PreToolUse` hook scripts exist on disk (or hooks are removed)
- [ ] Sandbox container is running (`docker ps --filter name=...`)
- [ ] Audit prompt has been filled with tool-specific commands

---

## Files in This Folder

| File | Purpose |
|---|---|
| `workflow.md` | This file — the repeatable process |
| `sandbox-setup.md` | Container design, build steps, bind-mount vs docker exec |
| `prompt-template.md` | Generalized audit prompt with variable table |
| `filler-agent-prompt.md` | Agent that discovers tool metadata and fills the template |
| `cold-start-audit-skill.md` | Portable Claude Code `/cold-start-audit` skill |
