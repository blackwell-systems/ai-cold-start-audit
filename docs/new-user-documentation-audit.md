# New User Documentation Audit - agentic-cold-start-audit

**Audit Date:** 2026-03-06
**Auditor:** Claude Agent (simulating new user perspective)
**Repository:** agentic-cold-start-audit v1.1.0

---

## Executive Summary

The documentation is comprehensive and well-structured but has **significant onboarding friction** for new users. The core concepts (cold-start audit, filler/audit agents, sandbox isolation, pure discovery) are well-explained individually, but the **relationships between components** and **end-to-end workflow** are unclear until reading 3-4 documents. Critical gaps include: missing "what happens when" flow diagrams, unclear skill installation instructions, and prerequisites scattered across multiple files. The documentation assumes familiarity with AI agents, Claude Code, and background agent permissions.

**Strengths:** Excellent depth, strong examples (brewprune), clear severity tiers, good anti-pattern documentation.

**Critical issue:** A new user cannot successfully run their first audit without reading 600+ lines across 5 documents and reverse-engineering prerequisites.

---

## Critical Gaps (Blocks Understanding)

### 1. No Installation/Setup Section in README

**Where encountered:** README.md - jumps directly to "Quickstart" (line 34)

**User impact:** New users don't know what to install before starting. The skill commands (`/cold-start-audit`) appear without explanation of how to get them.

**What's missing:**
- How to install the `/cold-start-audit` skill itself
- Where to put `prompts/cold-start-audit-skill.md`
- Prerequisite: Claude Code must be installed
- Prerequisite: Docker required for container mode
- User-level vs. custom agent types installation distinction

**Suggested fix:**
```markdown
## Installation

### Prerequisites
- [Claude Code](https://code.claude.com) (CLI or IDE extension)
- Docker (for container mode only)

### Install the Skill
1. Copy the skill to your Claude Code skills directory:
   ```bash
   mkdir -p ~/.claude/skills
   cp prompts/cold-start-audit-skill.md ~/.claude/skills/
   ```
2. Restart Claude Code (skills are loaded at session start)
3. Verify installation: type `/cold-start-audit` and see the skill appear

### Optional: Install Custom Agent Types
For enhanced observability and tool enforcement:
```bash
mkdir -p ~/.claude/agents
ln -sf $(pwd)/prompts/agents/filler.md ~/.claude/agents/filler.md
ln -sf $(pwd)/prompts/agents/audit.md ~/.claude/agents/audit.md
```
Then restart Claude Code.
```

---

### 2. The "Filler Agent" Concept Is Never Explained Before Use

**Where encountered:** README.md line 6, line 177; used throughout without definition

**User impact:** Users see "filler agent reads your project's help output" in the tagline but don't understand:
- Why two agents are needed (why not one?)
- What problem the filler agent solves
- When they interact with it vs. when it runs in background
- What "populates a structured audit prompt" means

**What's missing:**
- Definition: "The filler agent is a metadata discovery agent that..."
- Rationale: "We use two agents because the audit must run with zero context..."
- Analogy: "Think of it as auto-configuring a test harness before running tests"

**Suggested fix:** Add a "Core Concepts" section in README before "Why":
```markdown
## Core Concepts

**Cold-Start Audit:** Simulating a brand-new user who has never seen your tool, following only help text, and reporting every friction point encountered.

**Two-Agent Pattern:**
1. **Filler Agent** - Reads `--help` output from your tool, discovers subcommands/flags, and generates a customized audit prompt with exact commands to test. Runs once per audit round (or reuse previous).
2. **Audit Agent** - Executes the filled prompt in a fresh session with zero knowledge, runs 30+ commands in sandbox, reports findings. This is the "new user" simulation.

Why two agents? The audit agent MUST have zero context (simulating a new user). If it read help text itself, it would know what to expect. The filler agent handles the "setup" knowledge separately.

**Sandbox Isolation:** All commands run through an isolation layer (Docker container, env vars, or temp directory) so destructive operations are safe.

**Pure Discovery:** The audit agent has no memory of previous rounds. It discovers friction organically, not from a checklist.
```

---

### 3. "Container Mode" Quickstart Example Has a Logical Gap

**Where encountered:** README.md lines 58-62

**User impact:** Shows two commands but doesn't explain:
- What `container build` does (first time users see this command)
- Why `mytool-r1` appears - where did "r1" come from?
- What happens between these two commands

**Current text:**
```bash
/cold-start-audit container build mytool
/cold-start-audit run mytool --mode container mytool-r1
```

**Suggested fix:**
```bash
# Step 1: Build a Docker container with your tool installed
# This auto-names the container "mytool-r1" (round 1)
/cold-start-audit container build mytool

# Step 2: Run the full audit using that container
# The agent will execute 30+ commands inside mytool-r1 and write findings
/cold-start-audit run mytool --mode container mytool-r1
```

---

### 4. Dockerfile.sandbox Convention Introduced Too Late

**Where encountered:** README.md line 44-54, but convention not explained until sandbox-setup.md

**User impact:** User sees "Container mode expects `Dockerfile.sandbox` by convention" but doesn't know:
- Why this specific filename?
- What if I already have a Dockerfile?
- Can I use my existing Dockerfile?
- What's in Dockerfile.sandbox that's different?

**What's missing:** The contract requirements (non-root user, sudo, standard path, multi-stage) aren't mentioned in README at all. User might try using existing production Dockerfile and hit confusing errors.

**Suggested fix:** Add brief explanation in README:
```markdown
**Create a Dockerfile** (container mode only):

Container mode requires an **audit-ready Dockerfile** (not your production Dockerfile). This special Dockerfile:
- Installs the tool at `/usr/local/bin/<tool-name>`
- Creates a non-root user with sudo access
- Simulates a realistic user environment

You have three options:
[... existing options ...]

See [sandbox-setup.md](sandbox-setup.md#dockerfile-pattern) for the full contract.
```

---

### 5. No "First Run Success" Validation Step

**Where encountered:** Throughout documentation - no guidance on "how do I know it worked?"

**User impact:** After running first audit, user has files in `docs/` but doesn't know:
- Did it complete successfully?
- Should I see agent progress output?
- What does a successful run look like vs. a failure?
- How long should it take?

**Suggested fix:** Add "Verify First Run" section after quickstart:
```markdown
### Verify Your First Audit

After running `/cold-start-audit run`:
1. **Check agent output** - You'll see the filler agent discover subcommands, then the audit agent launch in background
2. **Monitor progress** - Look for "Running command X/30..." style output
3. **Expected duration** - 5-10 minutes for typical CLI tool with 5-10 subcommands
4. **Check results** - `ls docs/` should show:
   - `cold-start-audit-prompt.md` - The filled audit prompt
   - `cold-start-audit.md` - The findings report

If you see files but they're empty, check permissions (see Troubleshooting below).
```

---

### 6. Permissions Section Is Confusing

**Where encountered:** README.md lines 290-323

**User impact:**
- "Background agents cannot prompt for tool approval" - user doesn't know what a background agent is
- User-level vs project-level distinction unclear
- "Not hot-reloaded" mentioned twice but no explanation of what happens if you forget to restart
- No troubleshooting: "How do I know if permissions are wrong?"

**What's missing:**
- Definition of background agents
- Symptom-based guidance: "If agents launch but produce no output, check permissions"
- Visual example of where `~/.claude/settings.json` is

**Suggested fix:** Restructure as troubleshooting-first:
```markdown
## Permissions

**Symptom:** Agents launch but produce no output, or audit completes with empty files.
**Cause:** Background agents need explicit permission to use Bash, Read, and Write tools.

### Quick Fix (Recommended)
Add to `~/.claude/settings.json` (in your home directory):
```json
{
  "permissions": {
    "allow": ["Bash", "Read", "Write"]
  }
}
```

Then **restart Claude Code** (settings load at session start).

To verify: Try `/cold-start-audit preflight <tool>` - it will check permissions.

### Advanced: Project-Level Scoped Permissions
[... move existing advanced content here ...]
```

---

### 7. Mode Selection Algorithm Buried

**Where encountered:** SPEC.md lines 66-86, mentioned but not explained in README

**User impact:** README says "Run `/cold-start-audit mode mytool` if unsure" but doesn't explain:
- What questions the algorithm asks
- How it makes the decision
- What the output looks like
- When you should override the recommendation

**Suggested fix:** Add Mode Selection section to README before Quickstart:
```markdown
## Choosing a Sandbox Mode

Not sure which mode to use? Run:
```bash
/cold-start-audit mode mytool
```

This analyzes your tool's `--help` output and asks:
1. **Q1:** Does the tool have destructive operations? (remove, delete, uninstall, etc.)
   → If yes: **container mode** (strongest isolation)

2. **Q2:** Does the tool write only to self-managed state? (redirectable via env var)
   → If yes: **local mode** (no Docker needed)

3. **Q3:** Does the tool operate on the current directory? (file transformers, scaffolders)
   → If yes: **worktree mode** (copy directory, test in copy)

**Example output:**
```
Analyzing mytool --help...

Found destructive commands: prune, remove, clean
Recommendation: container mode

Rationale: Tool can delete files/packages. Container isolation prevents
accidental damage to host system.

Next: /cold-start-audit container build mytool
```
```

---

## Moderate Gaps (Causes Friction)

### 8. Relationship Between Files Unclear

**Where encountered:** README.md "Files" section (lines 356-365)

**User impact:** Table lists 6 files with purposes, but doesn't explain:
- Reading order for new users
- Which are reference vs. tutorial
- Which are required vs. optional
- Where custom agent types fit in

**Suggested fix:**
```markdown
## Documentation Guide

**Start here (in order):**
1. **README.md** (this file) - Overview, quickstart, concepts
2. **workflow.md** - Step-by-step first audit walkthrough
3. **sandbox-setup.md** - Dockerfile creation and mode details

**Reference:**
- **SPEC.md** - Formal specification (invariants, algorithms, guarantees)
- **CLAUDE.md** - Instructions for AI agents working on this repo
- **CHANGELOG.md** - Pattern evolution and lessons learned

**Prompts (used by skill, not for manual reading):**
- `prompts/prompt-template.md` - Template the filler agent fills
- `prompts/filler-agent-prompt.md` - Instructions for filler agent
- `prompts/agents/` - Custom agent type definitions (optional)
```

---

### 9. "Pure Discovery" Concept Introduced Multiple Times

**Where encountered:** README lines 23-27, SPEC lines 117-119, CLAUDE.md Core Principles

**User impact:** This critical concept is explained 3 times in different ways, suggesting it's important, but the first explanation (README) is vague. User doesn't understand WHY this matters until CLAUDE.md.

**What's missing:** User impact explanation - "What does this mean for me?"

**Suggested fix:** Consolidate in README with clear user impact:
```markdown
### Pure Discovery vs. Regression Testing

**Important:** Each audit discovers friction organically with zero knowledge of previous rounds. The agent doesn't verify a checklist of known issues.

**Why this matters:**
- You never maintain test cases manually
- If old issues regress, the agent rediscovers them naturally (same friction a new user hits)
- The agent isn't biased by knowing what "should" work
- Coverage gaps are acceptable - we want salient friction, not exhaustive validation

**What this means for you:**
- Don't expect the agent to confirm "Issue #23 is fixed"
- Do expect fresh perspectives every round
- Fixed issues won't appear in new reports (unless they regressed)
```

---

### 10. Container Lifecycle Terminology Inconsistent

**Where encountered:** Multiple places use "image", "container", "round", "Dockerfile" interchangeably

**User impact:** User confused about:
- When to rebuild Dockerfile vs. image vs. container
- What "round" means (is it a container? an image? both?)
- Why `brewprune-r7` appears in examples - what's the naming pattern?

**Suggested fix:** Add terminology glossary to README:
```markdown
## Terminology

**Dockerfile.sandbox** - Template for building container (stable infrastructure)
**Image** (e.g., `mytool-r1`) - Compiled snapshot from Dockerfile (rebuilt when source changes)
**Container** (e.g., `mytool-r1`) - Running instance of image (ephemeral, for one audit)
**Round** - An audit iteration (round 1 = first audit, round 2 = after fixing issues)

**Naming convention:** `<tool>-r<N>` where N increments each round (mytool-r1, mytool-r2, etc.)

**When to rebuild:**
- Dockerfile: Only when runtime dependencies change (rare)
- Image: When tool source code changes
- Container: Automatically created per round by `container build` command
```

---

### 11. Preflight Validation Output Examples Missing

**Where encountered:** README lines 185-217 shows example output, but commands section (lines 132-135) doesn't mention it runs automatically

**User impact:** User might skip to Commands section, not realize preflight runs automatically, and be confused by validation output appearing unexpectedly.

**Suggested fix:** Add to Commands section:
```markdown
/cold-start-audit preflight <tool-name> --mode <mode> [mode-args]
  Validate all prerequisites before running an audit. Checks tool installation,
  permissions, Docker status, and provides actionable fix steps if any fail.

  Note: Runs automatically before setup and run commands - you rarely need
  to call this manually. Use it to debug permission/Docker issues.
```

---

### 12. "Subsequent Rounds" Section Too Early

**Where encountered:** README lines 76-90 - appears before user understands first round

**User impact:** Overwhelming. User hasn't completed first audit yet and is already seeing "after fixing issues, running the next round..."

**Suggested fix:** Move "Subsequent Rounds" section after "Preflight Validation Example", or create a "Advanced Usage" section at end of README.

---

### 13. Scout-and-Wave Integration Mentioned Too Late

**Where encountered:** README lines 367-377 (near end)

**User impact:** User completes audit, has findings, doesn't know what to do next. The integration point is mentioned as "composing" but user wants "what do I do with these findings?"

**Suggested fix:** Add "What's Next?" section after "Example Output":
```markdown
## What to Do With Findings

After getting your audit report:

**Option 1: Manual fixes**
1. Read the report: `/cold-start-audit report`
2. Triage by severity (UX-critical first)
3. Fix issues in your codebase
4. Run next audit round to verify

**Option 2: Automated parallel fixes (scout-and-wave)**
Hand the findings to [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave) for automated parallel implementation:
```bash
/saw scout "Fix UX findings from cold-start audit"
/saw wave
```
The scout filters already-fixed items, assigns remaining findings to parallel agents, and executes in waves.

See [agentic-workflows](https://github.com/blackwell-systems/agentic-workflows) for the full Audit→Fix→Rediscover cycle.
```

---

### 14. No Troubleshooting Section

**Where encountered:** Nowhere - troubleshooting scattered across workflow.md, sandbox-setup.md

**User impact:** When things go wrong, user doesn't know where to look. Common issues:
- "Agent produced empty report"
- "Container not found"
- "Permission denied"
- "Help text shows but audit doesn't run"

**Suggested fix:** Add Troubleshooting section to README:
```markdown
## Troubleshooting

### Agent launches but produces no output or empty files
**Cause:** Permissions not configured or session not restarted.
**Fix:**
1. Check `~/.claude/settings.json` has `"allow": ["Bash", "Read", "Write"]`
2. Restart Claude Code session
3. Run `/cold-start-audit preflight <tool>` to verify

### Container not found
**Cause:** Container name mismatch or container not running.
**Fix:** Run `/cold-start-audit container list <tool>` to see available containers, or build a new one with `/cold-start-audit container build <tool>`

### Docker not running
**Cause:** Docker daemon not started.
**Fix:** Start Docker Desktop (macOS) or `sudo systemctl start docker` (Linux)

### "Dockerfile.sandbox not found"
**Cause:** Container mode requires special audit-ready Dockerfile.
**Fix:** Generate one: `/dockerfile-sandbox-gen <tool>` or see sandbox-setup.md for manual template

### Audit runs but finds nothing
**Cause:** Tool may not have obvious friction, or isolation mode incorrect.
**Fix:**
1. Check that commands actually run in sandbox: look for "docker exec" or env var in agent output
2. Try `/cold-start-audit mode <tool>` to verify mode recommendation
3. Review filled prompt at `docs/cold-start-audit-prompt.md` - are commands realistic?
```

---

## Minor Gaps (Polish Issues)

### 15. SPEC.md Too Formal for New Users

**Where encountered:** SPEC.md throughout - academic style, no examples

**User impact:** New users who read SPEC.md first (because it's linked early) get overwhelmed. Terms like "invariant", "precondition", "correctness guarantee" are intimidating.

**Suggested fix:** Add preamble to SPEC.md:
```markdown
# Cold-Start Audit Specification

**Note:** This is a formal specification for implementers and contributors. If you're new to the project, start with [README.md](README.md) and [workflow.md](workflow.md) for practical guidance.

This document defines the mathematical guarantees and formal constraints that make the protocol safe and correct.
```

---

### 16. Severity Tiers Explained Twice

**Where encountered:** README lines 348-355 (table) and lines 56-59 (prompt template)

**User impact:** Minor redundancy. Table is clearer but appears after examples.

**Suggested fix:** Reference the table from template: "See [Severity Tiers](#severity-tiers) below for definitions."

---

### 17. Example Output Uses brewprune Without Introduction

**Where encountered:** README lines 324-346 - first mention of brewprune

**User impact:** User doesn't know what brewprune is or why it's the example. Is it part of this tool? A test case?

**Suggested fix:** Add context:
```markdown
## Example Output

A real cold-start audit of [brewprune](https://github.com/blackwell-systems/brewprune) (a Homebrew package manager tool) found 18 issues across 6 rounds of iterative improvement:
```

---

### 18. Version Badge Shows 1.1.0 But No Changelog Link

**Where encountered:** README line 4 - version badge, but CHANGELOG.md not linked until file list

**User impact:** User sees version, wonders "what's new in 1.1.0?" but no easy way to find out.

**Suggested fix:** Link the badge to CHANGELOG.md:
```markdown
[![Version](https://img.shields.io/badge/version-1.1.0-blue)](CHANGELOG.md)
```

---

### 19. "When to Use It" Section Could Be Checklist

**Where encountered:** README lines 18-33 - prose format

**User impact:** Hard to quickly assess "is this for me?"

**Suggested fix:** Make it a checklist:
```markdown
## Is This For You?

Use cold-start auditing if you check 2+ boxes:
- [ ] Tool ships to users who don't know you (public/open-source)
- [ ] Tool has multiple subcommands, flags, or modes
- [ ] Help text is primary documentation (not web docs)
- [ ] You iterate on UX frequently
- [ ] You want to catch regressions before users report them

Don't use if:
- [ ] Tool is internal with small team that knows the author
- [ ] Tool requires non-reproducible external setup (live APIs, third-party accounts)
```

---

### 20. Colima Note Feels Random

**Where encountered:** sandbox-setup.md lines 206-213

**User impact:** Appears out of nowhere at end of container section. User doesn't know what Colima is or why it matters.

**Suggested fix:** Move to a "Platform Notes" section or add context:
```markdown
### Platform Notes

**macOS with Colima:** If you installed Docker via [Colima](https://github.com/abiosoft/colima) (common on macOS), note that `docker compose` may not be available. Use `docker build` and `docker run` instead. All other commands (`docker exec`, `docker ps`, `docker rm`, `docker rmi`) work normally.
```

---

## Strengths (What Works Well)

1. **Excellent use of examples** - brewprune audit findings are concrete and instructive
2. **Clear severity tiers** - UX-critical/improvement/polish distinction is well-defined
3. **Comprehensive SPEC.md** - formal guarantees give confidence to advanced users
4. **Strong "Why" section** - motivation is compelling ("you can't UX-test your own tool")
5. **Anti-patterns documented** - CLAUDE.md explicitly calls out what NOT to do
6. **Automation commands** - `container build`, `preflight`, `cleanup` save manual work
7. **Multiple isolation modes** - container/local/worktree covers wide variety of tools
8. **Graceful fallback** - custom agent types optional, skill works without them
9. **Pattern evolution** - CHANGELOG.md shows real-world lessons learned
10. **Self-contained skill** - works from any project, no repo cloning needed

---

## Recommended Priority Fixes

### High Priority (Blocks First-Time Success)

1. **Add Installation section to README** - Critical. Users can't start without this.
2. **Add Core Concepts section before "Why"** - Explain filler/audit agents upfront.
3. **Add Troubleshooting section** - Empty output is confusing and common.
4. **Clarify Container Build quickstart** - Explain what commands do between steps.
5. **Add "Verify First Run" section** - Users need success criteria.

### Medium Priority (Reduces Friction)

6. **Restructure Permissions section** - Symptom-first troubleshooting approach.
7. **Add Mode Selection explanation to README** - Show sample output from `mode` command.
8. **Add Terminology glossary** - Dockerfile vs image vs container vs round.
9. **Add "What's Next?" section** - Guide users from findings to action.
10. **Improve file relationship documentation** - Reading order guide.

### Low Priority (Polish)

11. Move "Subsequent Rounds" later in README
12. Add preamble to SPEC.md for new users
13. Add context to brewprune example
14. Link version badge to CHANGELOG
15. Convert "When to Use" to checklist

---

## New User Questions Still Unanswered

After reading all documentation:

- **How do I install the skill?** (Not in README)
- **What is a "background agent" and when do I interact with it?** (Mentioned but not defined)
- **Can I run an audit without Docker?** (Yes, local/worktree modes, but not obvious from quickstart)
- **How long does an audit take?** (No guidance)
- **What if my tool doesn't have a --help flag?** (Not addressed)
- **Can I customize what the audit tests?** (prompt-template.md exists but README doesn't mention manual filling)
- **What's the relationship between this and claude-code?** (Assumes familiarity)
- **Do I need to know Docker?** (Unclear - Dockerfile.sandbox is "auto-generated" but examples show manual commands)
- **What happens if I don't restart Claude Code after settings change?** (Silent failures, not explained)
- **Can I audit a tool that's not installed globally?** (e.g., ./bin/mytool - not addressed)
- **What's `$$` in the local mode example?** (`--env MYTOOL_DB=/tmp/audit-$$/db` - shell expansion not explained)
- **How do rounds relate to git commits?** (Are rounds tied to commits? Branches? Tags?)

---

## Overall Assessment

**Time-to-first-run estimate:** 45-60 minutes for a motivated new user (reading docs, troubleshooting permissions, building container, running first audit)

**Comparison to similar tools:**
- **Better than:** Most CLI tool testing frameworks (comprehensive, well-motivated)
- **Worse than:** Tools with interactive setup wizards (`npm init`, `rails new`)
- **Similar to:** Docker documentation (thorough but assumes prerequisite knowledge)

**Top 3 improvements needed:**

1. **Add Installation + Prerequisites section** - Currently the biggest blocker. Users can't get started.

2. **Add visual workflow diagram** - Show the end-to-end flow: skill → filler agent → filled prompt → audit agent → findings report. Include where Docker, permissions, and Claude Code fit in.

3. **Add "Quick Start (5 minutes)" section** - Absolute minimum to see it work:
   ```markdown
   ## Quick Start (5 Minutes)

   Want to see it in action before learning details?

   1. Install skill: `cp prompts/cold-start-audit-skill.md ~/.claude/skills/`
   2. Set permissions: Add `"allow": ["Bash", "Read", "Write"]` to `~/.claude/settings.json`
   3. Restart Claude Code
   4. Run: `/cold-start-audit mode yourTool` (analyzes and recommends mode)
   5. Follow the recommendation

   First audit takes 5-10 minutes. Results in `docs/cold-start-audit.md`.

   Now read below to understand what just happened and how to iterate...
   ```

---

## Final Notes

This is a **sophisticated system with excellent depth**, but it suffers from **expert blind spot** - the authors know the system so well they've forgotten what a new user doesn't know. The documentation is comprehensive for someone who already understands the concepts, but the **learning curve is steep** for cold starts.

The irony: a tool for discovering cold-start friction itself has cold-start friction in its documentation.

**Recommendation:** Invest 2-3 hours in adding the High Priority fixes above. They will reduce time-to-first-run from 45-60 minutes to 15-20 minutes and dramatically improve new user success rate.
