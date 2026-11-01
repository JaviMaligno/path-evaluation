# Grading the path vs grading the result

Code and data for the article **[More Than One Way to Prove It](https://javieraguilar.ai/en/blog/more-than-one-way-to-prove-it)**
([Spanish](https://javieraguilar.ai/es/blog/more-than-one-way-to-prove-it)).

The question: when an agent's task is exploratory (a proof, a debugging session), does grading the
agent's *path* against reference paths, as in
[CORE (Michelakis, Hadjiyiannis and Stamoulis, 2025)](https://arxiv.org/abs/2509.20998), catch more
bad work than grading the final state plus a list of forbidden and required actions? And what does
it cost in valid work rejected?

## Worlds

| World | Tasks | Truth |
|---|---|---|
| **A** — real bugs | 10 bugs reintroduced into [humanize](https://github.com/python-humanize/humanize) (`bugs/`, library copy in `repo/` under its MIT licence) | the library's suite + hidden tests, run against the agent's `src/` |
| **B** — Lean 4, no Mathlib | 26 theorems (`theorems/`), two reference proofs each with different strategies | compiles, statement intact, no `sorry`, `#print axioms` ⊆ {propext, Classical.choice, Quot.sound} |
| **C** — Lean 4 + Mathlib | 9 [miniF2F](https://github.com/yangky11/miniF2F-lean4) problems (`theorems-mathlib/`) | same as B |

Lean references were written by a model that is not among those evaluated; `validate_theorems.py`
checks that each theorem's two references compile and use different strategy skeletons.

## Models

Haiku 5.5, Sonnet 5.5, Opus 5.5 (Claude Code), GPT-6.1 Sol and GPT-6 Luna (Codex, effort `medium`),
Qwen 3.5 9B (OpenRouter, `qwen_loop.py`). Three runs per model and task. `models.json` maps the short
names used in the data (`haiku`, `sonnet`, `opus`, `codex` = Sol, `luna`, `qwen`).

## Graders

- **E0** final state: the issue test passes / `lean` exits 0.
- **E1** E0 and nothing forbidden: no test edited / no `sorry`, `admit`, `axiom`, `native_decide`, changed statement.
- **E2** E1 and order obligations: tests run before the first edit and after the last / compile after the last edit.
- **E3** E0 and CORE's path correctness ≥ τ = 0.8 (`pc.py`). In A the path is CORE's condensation of
  state-changing calls; in Lean it is a strategy skeleton (`world_b.strategy`), our translation of the idea.

The frozen rules are in `DESIGN.md` ("Reglas congeladas"), written before the full run.

## Reproduce the numbers

```bash
uv sync
uv run python analyze.py            # every grader vs truth, per world; writes nothing
uv run python report_truth.py rerun # world A truth limited to what each bug report implies
uv run python coarse_routes.py      # how many genuinely different Lean routes there are
uv run pytest -q tests/
```

`analyze.py` re-derives actions from the raw event streams in `runs/events/` and takes, for each
model, the episodes from the file listed in its `SOURCES`.

## Data

| File | What |
|---|---|
| `runs/full-a.jsonl` | world A, all models |
| `runs/full-b.jsonl` | world B; valid for Haiku, Sonnet, Opus, Qwen |
| `runs/full-b-elanfix.jsonl` | world B, Sol and Luna rerun after the sandbox fix (below) |
| `runs/trial-mathlib*.jsonl` | world C; Luna's valid runs are the `*-pathfix` files |
| `runs/events/` | raw event stream of every episode |
| `runs/events-pre-*` | event streams of the superseded Sol/Luna runs |
| `runs/report-cases.json` | blind judge's verdicts on which hidden cases follow from each bug report |
| `runs/report-truth.jsonl` | world A episodes re-judged on those cases |
| `runs/analysis.txt` | output of `analyze.py` |

Two checks have a second version, reported next to the frozen one: the statement check also accepts
ASCII spellings Lean reads as the same token (`->` for `→`, `<=`, `>=`, leading `import` lines), and
world A's truth is also given limited to the hidden cases a blind judge says follow from the report
(three votes, majority, with the issue test as a control).

## Running it yourself

Written for and run on Windows 11. Notes that matter if you rerun it:

- Codex runs in its Windows sandbox with its own `CODEX_HOME` (`agents.prepare_codex_home`). The
  sandbox user does not share your profile: set `ELAN_HOME` and put the toolchain's `lean.exe` first on
  `PATH` (`agents.env_for`), or `lean` either fails or, through the elan proxy, takes minutes per
  Mathlib compile. `run.py` refuses to start if `lean --version` does not run inside the sandbox.
- Every `CODEX_HOME` on a machine shares the sandbox's Windows users but keeps its own copy of their
  passwords; `agents.sync_sandbox_users` keeps the experiment's home and `~/.codex` in step.
- `mem_guard.py <world>` stops a run before the next episode when free memory stays low; runs are
  resumable (`run.py` skips finished episodes and retries errored ones).
- World C needs a Mathlib project; its `LEAN_PATH` is read from `~/lean/pe2math-LEAN_PATH.txt`.

## Licence

MIT for the code in this repository. `repo/` is a copy of humanize under its own licence (`repo/LICENCE`).
