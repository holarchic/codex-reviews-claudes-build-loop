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

Operator-only tooling that measures whether the loop improves a model's code; the skills never load it. Stdlib Python, run from the repo root as `python3 -m eval validate | probe | freeze | run | judge | report [--experiment NAME]`. Raw runs, transcripts and reports live in `eval/data/` (the pilot at its root, other experiments in `eval/data/<name>/`; gitignored, Dropbox-ignored).

- **Tasks** (`tasks.py`, `eval/tasks/<id>/`): writer-visible `TASK.md` and `FOLLOWUP.md`; private hidden suites, references, wrong and degraded variants, traps. `validate` proves each suite passes its reference and fails every wrong variant, then writes `expected.json` — the frozen test IDs every score divides by. `dev-*` tasks exercise the harness and are never scored.
- **Sessions** (`agents.py`, `audit.py`, `procs.py`): every model call goes through `agents.run(Agent(cli, model), …)`, dispatching to `run_claude` or `run_codex`, which own the isolation flags, tool policy per role (writer, reviewer, judge), timeouts and the terminal class (completed, timeout, infra). Either CLI can be the writer; prompts travel on stdin. Claude runs with project-only settings and a fail-closed Bash sandbox that denies reads of private roots; Codex runs from a throwaway home under an outer Seatbelt profile that denies private reads and writes and lets only a writer write its run dir. Run dirs and Codex homes are random names under a runs root no session can list (`audit.anonymous_workdir`). `audit.py` also records tool calls that name a private path, as a diagnostic. `procs.py` runs each child in its own process group and kills the group on timeout. `probe.py` proves the isolation with positive controls.
- **Runs** (`loop.py`, `prompts.py`): an `Arm` names its writer and reviewer agents and its ablation switches (writer rules, plan review, review passes). `execute_run` takes one task × arm from an empty repo to a scored build, committing after each writer step so infrastructure retries restart from HEAD; `execute_followup` applies FOLLOWUP.md to a copy of a finished build. `prompts.py` builds reviewer prompts from the shipped `codex-review` templates and rubric, read as bytes at call time. When those templates ask for a `Basis:`, `parse_verdict` reads a HIGH/MEDIUM issue without one as LOW and the writer is told each issue's effective severity.
- **Scoring** (`score.py`, `runner.py`): `runner.py` runs a hidden suite inside the candidate's process with a per-test alarm and streams results, so a killed run still scores finished tests; `score.py` adds static metrics.
- **Judging** (`judge.py`): blind pairwise comparison of two builds' Python files by one Claude and one Codex frontier model, in both orders, plus calibration against degraded references.
- **Experiments** (`experiments.py`, `manifest.py`, `pilot.py`, `records.py`, `report.py`): an `Experiment` names its arms, judged pairs, seed, data dir and pre-registration; `pilot` plus tier, reverse-direction and ablation experiments are registered. `freeze --experiment X` hashes X's pre-registration, the harness, the scored tasks and the shipped skill files into X's own manifest (`eval/MANIFEST.json` for the pilot, `eval/manifests/<name>.json` otherwise); `run`, `judge` and `report` refuse to start if any changed, so an experiment's report is regenerated from its freeze commit once the harness moves on. `pilot.py` runs an experiment's seeded schedule three jobs at a time, never two of one task at once, and resumes by skipping records `records.py` validates; `report.py` regenerates the report and applies the pilot's pre-registered decision rule (other experiments are descriptive until their pre-registration defines one).
