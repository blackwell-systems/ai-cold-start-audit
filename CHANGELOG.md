# Changelog

## Unreleased

### Added
- Audit prompt template with variable table, severity tiers, and structured findings format
- Filler agent prompt that auto-discovers tool metadata from `--help` output and environment
- Workflow guide with prerequisites, permissions setup, launch steps, and triage process
- Sandbox setup guide covering container design, Dockerfile patterns, and docker exec vs bind mount access patterns
- Self-contained Claude Code `/cold-start-audit` skill with inline setup, permissions, Dockerfile patterns, and audit template
- Quickstart section in README with 3-command example
- Example output from real brewprune audit (21 findings)
- `init` subcommand for scaffolding `.claude/settings.json` permissions
- Multi-package-manager support in filler agent (apt, apk, Homebrew, none)
- **Timestamp metadata in audit reports** (2026-02-28): Audit report template now includes metadata header with Audit Date, Tool Version, Container name, and Environment. Makes staleness visible and enables regression tracking across audits. Helps prioritize findings based on when they were discovered. ([commit 67958a5](https://github.com/anthropics/agentic-cold-start-audit/commit/67958a5))
- **Container lifecycle documentation** (2026-02-28): Container Setup section now clarifies the three-tier distinction between Dockerfile (stable infrastructure template), image (rebuilt per round from source changes), and container (ephemeral running instance). Separates first-time setup from subsequent rounds. Prevents unnecessary Dockerfile recreation when only source code changed. Recommends round-specific naming (mytool-r1, mytool-r2) for clarity. ([commit b7a5a41](https://github.com/anthropics/agentic-cold-start-audit/commit/b7a5a41))

### Changed
- Skill is now fully self-contained - works from any project directory without cloning the repo
- Filler agent generalized from Homebrew-specific to language/platform-agnostic

### Pattern Evolution Timeline

The timestamp metadata improvement emerged from real-world usage:

**brewprune Round 4 (2026-02-28):**
- Scout phase produced IMPL doc with 31 P1/P2 findings
- P0 manual fixes committed between scout and wave execution
- Wave 1 launched days later - 7 of 17 findings already implemented (41%)
- **Gap identified:** Can't tell how stale audit findings are
- **Solution:** Add metadata header with audit date, tool version, container, environment

**Result:** Makes staleness visible and enables regression tracking across audit rounds

See [scout-and-wave/docs/LESSONS-ROUND4.md](https://github.com/anthropics/scout-and-wave/blob/main/docs/LESSONS-ROUND4.md) for complete case study.
