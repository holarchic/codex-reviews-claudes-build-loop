# Contributing

## Setup

Clone the repo, then point Claude Code at your checkout:

```
claude plugin marketplace add ./
claude plugin install codex-reviews-claudes-build-loop@codex-reviews-claudes-build-loop
```

The eval needs only Python 3 and its standard library. Run its tests from the repo root:

```
python3 -m unittest discover -s eval/tests -t .    # expect: OK
```

## Conventions

- The skills stay self-contained. They can't depend on files outside `skills/`, except the rubric that codex-review reads from `../build-loop/references/`.
- A rule change goes in `skills/build-loop/references/code-architecture-rules.md` only. Both review passes read that file.
- A new reviewer (Gemini, Grok) changes `skills/codex-review/SKILL.md` § Invocation only. The prompts and verdict format stay model-neutral.
- If a change alters what runs, the same PR bumps `version` in `.claude-plugin/plugin.json`, because installed users are pinned to that version.

## Pull requests

- One concern per PR.
- Say what you ran it on and paste the verdicts it produced.
- Update the README in the same PR as the behaviour it describes.

## Bug reports

Include:
- the Claude Code and `codex --version` output;
- the command you ran;
- the verdict or error;
- whether the diff was large (the reviewer's context limit is the usual cause of runs dying mid-way).
