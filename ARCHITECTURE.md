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

## Eval harness

**Files:** `eval/__main__.py`, `eval/harness/`, `eval/tasks/`, `eval/PREREG.md`, `eval/MANIFEST.json`, `eval/tests/`

Operator-only tooling that measures whether the loop improves a model's code; the skills never load it. Stdlib Python, run from the repo root as `python3 -m eval validate | probe | freeze | run | judge | report`. Raw runs, transcripts and the report live in `eval/data/` (gitignored, Dropbox-ignored).

- **Tasks** (`tasks.py`, `eval/tasks/<id>/`): writer-visible `TASK.md` and `FOLLOWUP.md`; private hidden suites, references, wrong and degraded variants, traps. `validate` proves each suite passes its reference and fails every wrong variant, then writes `expected.json` — the frozen test IDs every score divides by. `dev-*` tasks exercise the harness and are never scored.
- **Sessions** (`agents.py`, `audit.py`, `procs.py`): every model call goes through `run_claude` or `run_codex`, which own the isolation flags, tool policy per role (writer, reviewer, judge), timeouts and the terminal class (completed, timeout, infra). Claude runs with project-only settings and a fail-closed Bash sandbox that denies reads of private roots; Codex runs from a throwaway `CODEX_HOME`. `audit.py` flags tool calls touching private roots (the only control for Codex). `procs.py` runs each child in its own process group and kills the group on timeout. `probe.py` proves the isolation with positive controls.
- **Runs** (`loop.py`, `prompts.py`): `execute_run` takes one task × arm from an empty repo to a scored build, committing after each writer step so infrastructure retries restart from HEAD; `execute_followup` applies FOLLOWUP.md to a copy of a finished build. `prompts.py` builds reviewer prompts from the shipped `codex-review` templates and rubric, read as bytes at call time.
- **Scoring** (`score.py`, `runner.py`): `runner.py` runs a hidden suite inside the candidate's process with a per-test alarm and streams results, so a killed run still scores finished tests; `score.py` adds static metrics.
- **Judging** (`judge.py`): blind pairwise comparison of two builds' Python files by one Claude and one Codex frontier model, in both orders, plus calibration against degraded references.
- **Pilot** (`manifest.py`, `pilot.py`, `report.py`): `freeze` hashes the pre-registration, harness code, pilot tasks and shipped skill files; `run`, `judge` and `report` refuse to start if any changed. `pilot.py` runs the seeded schedule three jobs at a time and resumes by skipping complete records; `report.py` regenerates the report and applies the pre-registered decision rule.
