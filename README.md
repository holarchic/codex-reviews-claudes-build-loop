# codex-reviews-claudes-build-loop

Spec-first Build Loop for Claude Code: Claude builds, Codex CLI independently reviews the plan and the finished phase.

> **Status:** v0.2.0, October 2026. In daily use, but the rubric and prompts still change. There is no measured evidence yet that the loop beats Claude working alone. The eval under `eval/` is testing that.

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

```
/plugin marketplace add holarchic/codex-reviews-claudes-build-loop
/plugin install codex-reviews-claudes-build-loop@codex-reviews-claudes-build-loop
```

Check that `codex --version` runs. If a review fails with a model error, update the CLI first.

**Manual:** copy `skills/build-loop/` and `skills/codex-review/` into `~/.claude/skills/`. Keep them side by side, because codex-review reads the rubric from `../build-loop/references/`.

Plugin skills are namespaced, so the full command is `/codex-reviews-claudes-build-loop:build-loop plan`. The bare `/build-loop` works unless another command claims it. Claude also picks the skill up from plain requests like "plan this" or "build it".

Only Claude and Codex are supported for now. The reviewer is called in one place, `skills/codex-review/SKILL.md` § Invocation, and the prompts, rubric and verdict format don't depend on the model. PRs for Gemini or Grok are welcome.

## Development

`eval/` is operator tooling that measures whether the loop improves a model's code. The skills never load it. Its structure is in [ARCHITECTURE.md](ARCHITECTURE.md).

```
python3 -m unittest discover -s eval/tests -t .    # expect: OK
```

## Credits

The four discipline rules are adapted from Andrej Karpathy's observations on LLM coding failure modes. The review exclusions follow Cloudflare's [AI code review](https://blog.cloudflare.com/ai-code-review/) write-up. Many architecture primitives come from John Ousterhout's *A Philosophy of Software Design* and Alexis King's "Parse, don't validate". Decisions use Olaf Zimmermann's Y-statement format.

## Attribution

For those who care about such things, the majority of the code in this repo was produced by Claude Opus 5.5 (Anthropic), with david guiding Claude, providing insights, direction and critical review throughout the process.

## License

MIT
