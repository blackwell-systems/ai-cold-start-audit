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

### Changed
- Skill is now fully self-contained - works from any project directory without cloning the repo
- Filler agent generalized from Homebrew-specific to language/platform-agnostic
