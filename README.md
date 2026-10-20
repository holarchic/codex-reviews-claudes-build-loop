# codex-reviews-claudes-build-loop

A spec-first Build Loop for Claude Code. The agent updates the spec before writing code, and an external reviewer, Codex, grades the plan and the finished phase.

## What's inside

| Path | What |
|---|---|
| `skills/build-loop/` | The Build Loop, the plan-file resume, and the always-in-context coding rules. |
| `skills/build-loop/references/code-architecture-rules.md` | The full rubric: Karpathy's 4 rules plus 40 architecture primitives. The builder writes to it and the reviewer grades against it. |
| `skills/build-loop/templates/` | Starter SPEC and ARCHITECTURE, plus the per-phase plan file. |
| `skills/codex-review/` | The external reviewer: plan review, and a two-pass code review (correctness over the diff, architecture over the full touched modules). |

## The loop

`/build-loop plan` runs steps 1–2 and stops. `/build-loop build` (or `/build-loop implement`) runs the whole loop:

0. Capture the phase-start commit.
1. Read SPEC.md.
2. Write the plan to `plans/<phase>.md`.
3. Update SPEC first: a Y-statement decision, acceptance criteria and failure modes.
4. Re-plan against the updated spec.
5. **Plan review** by the external reviewer, repeated until GREEN.
6. Build in small verified commits, then update ARCHITECTURE and promote durable decisions and dead ends from the plan file into SPEC, so the review grades current docs.
7. **Phase review** of the whole diff in two passes, repeated until both are GREEN. There are no skip conditions.
8. Coherence pass, mark the plan landed, confirm a clean tree, report.

On an existing codebase, a missing SPEC.md or ARCHITECTURE.md is first reverse-engineered from the code. On a new project, the plan proposes the contract and step 3 creates SPEC from it.

Fixing what a review flagged doesn't count as resolving it. The loop moves on only after a re-review returns GREEN or the user rules on a disagreement, with a cap of 5 rounds.

## Docs

| File | Holds |
|---|---|
| `SPEC.md` | Design intent, the contract: objective, non-goals, interface, decisions, acceptance criteria, failure modes. |
| `ARCHITECTURE.md` | What was built, in present tense, by subsystem. No history. |
| `plans/<phase>.md` | One phase's working record: the plan, the phase-start SHA, a chunk checklist, review verdicts, dead ends. A fresh session resumes from its first unchecked chunk. |

## Install

**Claude Code plugin**

```
/plugin marketplace add holarchic/codex-reviews-claudes-build-loop
/plugin install codex-reviews-claudes-build-loop@codex-reviews-claudes-build-loop
```

**Manual:** copy `skills/build-loop/` and `skills/codex-review/` into `~/.claude/skills/`. Keep them side by side, because codex-review reads the rubric from `../build-loop/references/`.

Plugin skills are namespaced, so the full form is `/codex-reviews-claudes-build-loop:build-loop plan`; the bare `/build-loop` works unless another command already claims it. Claude also picks the skill up from plain requests like "plan this" or "build it".

## Requirements

- [Claude Code](https://claude.com/claude-code)
- [Codex CLI](https://github.com/openai/codex), installed and logged in (`npm i -g @openai/codex`, then `codex login`). Required: the loop rests on two independent reviewers from different model families, so there is no fallback. Without Codex, the review steps stop and say so.

Just Claude and Codex for now. Open a PR for Gemini/Grok etc. The reviewer is called in one place, `skills/codex-review/SKILL.md` § Invocation; the prompts, rubric and verdict format don't depend on the model.

## Credits

The four discipline rules are adapted from Andrej Karpathy's observations on LLM coding failure modes. The review exclusions follow Cloudflare's [AI code review](https://blog.cloudflare.com/ai-code-review/) write-up. Many architecture primitives come from John Ousterhout's *A Philosophy of Software Design* and Alexis King's "Parse, don't validate". Decisions use Olaf Zimmermann's Y-statement format.

## License

MIT
