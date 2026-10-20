# Architecture

A Claude Code plugin holding two skills that share one rubric file.

## Plugin manifest

**Files:** `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`

The repo is both the plugin and its own single-plugin marketplace (`source: "./"`), so `/plugin marketplace add` on the repo URL installs it. Claude Code discovers the skills under `skills/`.

## build-loop skill

**Files:** `skills/build-loop/SKILL.md`, `skills/build-loop/templates/`, `skills/build-loop/references/`

`SKILL.md` holds the Build Loop, the plan-file lifecycle and resume, and the always-in-context coding rules. `templates/` holds the starting SPEC and ARCHITECTURE (created in a target project when missing) and the per-phase plan file. `references/code-architecture-rules.md` is the full rubric, read on demand by the builder.

## codex-review skill

**Files:** `skills/codex-review/SKILL.md`

The independent reviewer. It runs Codex CLI read-only and stops if Codex is missing or logged out — there is no fallback reviewer. It reads the rubric from `../build-loop/references/code-architecture-rules.md`, relative to its own directory. That sibling path is the one coupling between the skills: both must be installed side by side, which the plugin layout guarantees.
