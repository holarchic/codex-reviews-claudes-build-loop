# codex-reviews-claudes-build-loop

Spec-first Build Loop for Claude Code: Claude builds, Codex CLI independently reviews the plan and the finished phase.

> **Status:** v0.2.0, October 2026. In daily use, but the rubric and prompts still change. In 83 review-fix commits across three projects, reviews caught 248 code defects the writer's own tests missed, mostly edge cases and error paths ([census](eval/census/SUMMARY.md)). In use, the loop also catches spec drift and missing tests. A controlled replay against Claude alone is under way in `eval/`.

A first-round architecture verdict from the eval pilot, verbatim apart from a shortened path:

```text
VERDICT: YELLOW
SUMMARY: The implementation appears to meet TASK.md’s behavior, with two architecture issues: internal positional tuples and an implicit public export surface.

ISSUES:

1. [MEDIUM] deps.py:26 — Missing dependencies and Tarjan traversal frames are stored as positional tuples, so callers identify fields by position (also lines 35, 76, 84).
   Fix: Use named records, such as `NamedTuple` or dataclasses, for missing-dependency pairs and traversal frames.

2. [LOW] deps.py:1 — The module has no explicit `__all__`; `heapq`, `Iterable`, and `Mapping` are also accessible as public module attributes, despite SPEC.md saying no other names are part of the public surface.
   Fix: Declare `__all__` with only `resolve`, `MissingDependencyError`, and `CycleError`.
```

## Why

It is for multi-phase work where a wrong design decision costs more than review time. Claude writes the spec and plan before any code. Codex, a reviewer from another model family, then grades the plan and each finished phase against a fixed 40-rule rubric until both come back GREEN.

**When not to use it**

- **Small changes.** If you can describe the diff in one sentence, the review rounds cost more than they find.
- **No Codex account.** There is no fallback reviewer.
- **[GitHub Spec Kit](https://github.com/github/spec-kit)** does spec-first work across many coding agents, but has no reviewer from another model.
- **[superpowers](https://github.com/obra/superpowers)** is a broader skill library with plan and review steps on one model, and no second CLI to install.
- **Claude Code's plan mode** needs no setup at all.

## What's inside

| Path | What |
|---|---|
| `skills/build-loop/` | The Build Loop, resuming from the plan file, and the coding rules that stay in context. |
| `skills/build-loop/references/code-architecture-rules.md` | The rubric: Karpathy's 4 rules plus 40 architecture primitives. The builder writes to it and the reviewer grades against it. |
| `skills/build-loop/templates/` | Starter SPEC and ARCHITECTURE, plus the per-phase plan file. |
| `skills/codex-review/` | The reviewer: plan review, and a two-pass code review. The correctness pass reads the diff; the architecture pass reads every touched module in full. |

## The loop

`/build-loop plan` runs steps 1–2 and stops. `/build-loop build` (or `implement`) runs the whole loop:

0. Capture the phase-start commit.
1. Read SPEC.md.
2. Write the plan to `plans/<phase>.md`.
3. Update SPEC first: a Y-statement decision, acceptance criteria and failure modes.
4. Re-plan against the updated spec.
5. **Plan review** by Codex, repeated until GREEN.
6. Build in small verified commits. Update ARCHITECTURE and move lasting decisions from the plan into SPEC.
7. **Phase review** of the whole diff in two passes, repeated until both are GREEN. There are no skip conditions.
8. Coherence pass, mark the plan landed, confirm a clean tree, report.

Editing in response to a finding doesn't resolve it. The loop moves on only when a re-review returns GREEN or you rule on a disagreement, with a cap of 5 rounds.

On an existing codebase, a missing SPEC.md or ARCHITECTURE.md is first reverse-engineered from the code. On a new project, the plan proposes the contract and step 3 creates SPEC from it.

## Files the loop keeps in your repo

| File | Holds |
|---|---|
| `SPEC.md` | The contract: objective, non-goals, interface, decisions, acceptance criteria, failure modes. |
| `ARCHITECTURE.md` | What was built, in present tense, by subsystem. No history. |
| `plans/<phase>.md` | One phase's working record: the plan, the phase-start SHA, a chunk checklist, review verdicts, dead ends. A fresh session resumes from its first unchecked chunk. |

## Requirements

- [Claude Code](https://claude.com/claude-code).
- [Codex CLI](https://github.com/openai/codex), installed and logged in (`npm i -g @openai/codex`, then `codex login`). Without it the review steps stop and say so.

Cost per build, measured in the eval pilot on five small greenfield tasks (Claude Opus 5.5 writing, Codex `gpt-6.1-sol` reviewing):

| | Claude alone | Full loop |
|---|---|---|
| Wall time | 0.7 min | 20 min |
| Claude output tokens | 9K | 117K |
| Codex input tokens | none | 1.05M |
| Review rounds (plan / phase) | none | 4.5 / 3.3 of 5 |

Larger codebases cost more, because the reviewer reads every touched module.

## Install

Paste this into Claude Code:

```text
Install the codex-reviews-claudes-build-loop plugin for me by following the Install steps in
https://github.com/holarchic/codex-reviews-claudes-build-loop#install
```

The agent runs these from a terminal:

```
codex --version            # Codex CLI is on PATH; if not, stop and ask the user to install it and run `codex login`
claude plugin marketplace add holarchic/codex-reviews-claudes-build-loop
claude plugin install codex-reviews-claudes-build-loop@codex-reviews-claudes-build-loop
claude plugin list         # expect codex-reviews-claudes-build-loop in the list
```

Restart Claude Code so the skills load. Inside a session, `/plugin marketplace add` and `/plugin install` do the same.

**Manual:** copy `skills/build-loop/` and `skills/codex-review/` into `~/.claude/skills/`. Keep them side by side, because codex-review reads the rubric from `../build-loop/references/`.

## Usage

| Command | Does |
|---|---|
| `/build-loop plan` | Steps 1–2: read SPEC, write the plan, stop. |
| `/build-loop build` or `implement` | The whole loop. |
| `/codex-review plan <path>` | Review one plan file. |
| `/codex-review` | Review uncommitted work, or the last commit if the tree is clean. |
| `/codex-review --base <ref>`, `--commit <sha>`, `--uncommitted` | Review a fixed scope. |

Plugin skills are namespaced, so the full form is `/codex-reviews-claudes-build-loop:build-loop plan`. The bare `/build-loop` works unless another command claims it. Claude also picks the skill up from plain requests like "plan this" or "build it".

## Configuration

| File | Controls |
|---|---|
| `~/.codex/config.toml` | The Codex model. Reasoning effort is pinned to `high` in every review call. |
| `skills/build-loop/references/code-architecture-rules.md` | The rubric both passes grade against. |
| `skills/codex-review/SKILL.md` § Invocation | The one place the reviewer is called. |

If a review fails with a model error, update the Codex CLI first. Only Claude and Codex are supported for now. The prompts, rubric and verdict format don't depend on the model, so another reviewer means changing § Invocation only.

## Development

`eval/` is operator tooling that measures whether the loop improves a model's code. The skills never load it. Its structure is in [ARCHITECTURE.md](ARCHITECTURE.md).

```
python3 -m unittest discover -s eval/tests -t .    # expect: OK
```

## Contributing

Issues and PRs are welcome, especially adapters for other reviewers (Gemini, Grok). See [CONTRIBUTING.md](CONTRIBUTING.md).

## Credits

The four discipline rules are adapted from Andrej Karpathy's observations on LLM coding failure modes. The review exclusions follow Cloudflare's [AI code review](https://blog.cloudflare.com/ai-code-review/) write-up. Many architecture primitives come from John Ousterhout's *A Philosophy of Software Design* and Alexis King's "Parse, don't validate". Decisions use Olaf Zimmermann's Y-statement format.

## Attribution

For those who care about such things, the majority of the code in this repo was produced by Claude Opus 5.5 (Anthropic), with david guiding Claude, providing insights, direction and critical review throughout the process.

## License

MIT
