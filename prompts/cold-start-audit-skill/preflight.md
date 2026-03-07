# Preflight Validation (`preflight <tool-name> --mode <mode> [args]`)

Validate all prerequisites before launching agents. Automatically run before `setup` and `run` commands. Fail early with actionable fix steps.

## All Modes — Common Checks

**1. Tool installation:**
```bash
<tool-name> --help
```
- **✓ Pass:** Command succeeds, help text displayed
- **✗ Fail:** Command not found or errors
  - **Fix:** Install the tool first. Check installation docs.

**2. Bash permissions:**
```bash
# Check ~/.claude/settings.json for:
{
  "permissions": {
    "allow": ["Bash", "Read", "Write"]
  }
}
```
- **✓ Pass:** "Bash" in allow list
- **✗ Fail:** Missing or using invalid format (`allow_bash: true`)
  - **Fix:**
    ```bash
    # Edit ~/.claude/settings.json and add:
    {
      "permissions": {
        "allow": ["Bash", "Read", "Write"]
      }
    }
    # Then restart Claude Code session (settings don't hot-reload)
    ```

**3. Session restart check:**
- **Warn:** "If you edited settings recently, restart Claude Code now. Settings are loaded at session start only."

**4. Hooks check:**
```bash
# Parse ~/.claude/settings.json for PreToolUse hooks on Bash
# For each hook script referenced, check:
ls <hook-script-path>
```
- **✓ Pass:** All hook scripts exist
- **✗ Fail:** Hook script missing
  - **Fix:** Remove hook reference from settings or create stub script

## Container Mode — Additional Checks

**5. Docker running:**
```bash
docker ps
```
- **✓ Pass:** Command succeeds, shows container list
- **✗ Fail:** Cannot connect to Docker daemon
  - **Fix:**
    ```bash
    # macOS/Linux:
    # Check if Docker Desktop is running, or start docker service
    sudo systemctl start docker  # Linux
    # macOS: Open Docker Desktop app
    ```

**6. Container exists and is running:**
```bash
docker ps --filter name=<container-name> --format '{{.Names}}'
```
- **✓ Pass:** Container name appears in output
- **✗ Fail:** No output (container not running)
  - **Check if container exists but is stopped:**
    ```bash
    docker ps -a --filter name=<container-name> --format '{{.Names}}'
    ```
  - **Fix (container exists):**
    ```bash
    docker start <container-name>
    ```
  - **Fix (container doesn't exist):**
    ```bash
    # Use the container build command:
    /cold-start-audit container build <tool-name>

    # Or manually:
    docker run -d --name <container-name> <image-name> sleep 3600
    ```

**7. Tool installed in container:**
```bash
docker exec <container-name> <tool-name> --help
```
- **✓ Pass:** Help text displayed
- **✗ Fail:** Command not found in container
  - **Fix:** Rebuild image with tool installed. Check Dockerfile.sandbox.

## Local Mode — Additional Checks

**8. Temp directory writable:**
```bash
# Extract temp path from env var args
TEMP_DIR=$(echo "<env args>" | grep -oP '/tmp/[^"]+' | head -1 | xargs dirname)
mkdir -p $TEMP_DIR && touch $TEMP_DIR/.test && rm $TEMP_DIR/.test
```
- **✓ Pass:** Directory created and writable
- **✗ Fail:** Permission denied
  - **Fix:**
    ```bash
    # Use a path you have write access to:
    mkdir -p /tmp/audit-$$
    # Then use: --env MYTOOL_DB=/tmp/audit-$$/db
    ```

**9. Tool respects env var:**
```bash
env <KEY=VALUE> <tool-name> --help
```
- **✓ Pass:** Command succeeds with env var set
- **✗ Fail:** Tool errors or ignores env var
  - **Fix:** Check tool docs for correct env var name. Tool may not support env var isolation.

## Worktree Mode — Additional Checks

**10. Source directory exists:**
```bash
ls <source-path>
```
- **✓ Pass:** Directory exists and readable
- **✗ Fail:** No such file or directory
  - **Fix:** Check the path. Use absolute path or correct relative path.

## Output Format

When all checks pass:
```
✓ Preflight checks passed (10/10)

All prerequisites met. Ready to run audit.

Next: /cold-start-audit run <tool-name> --mode <mode> [args]
```

When checks fail:
```
✗ Preflight checks failed (7/10 passed)

Failures:
  ✗ Docker not running
    Fix: Start Docker Desktop or run: sudo systemctl start docker

  ✗ Container brewprune-r8 not found
    Fix: /cold-start-audit container build brewprune
    Or manually: docker run -d --name brewprune-r8 brewprune-r8 sleep 3600

  ✗ Tool not installed in container
    Fix: Rebuild image. Check Dockerfile.sandbox includes tool installation.

Run these fixes, then retry: /cold-start-audit preflight <tool-name> --mode <mode> [args]
```

**Auto-integration:** `setup` and `run` commands automatically run preflight first. If preflight fails, stop and show fix steps. User must resolve before continuing.
