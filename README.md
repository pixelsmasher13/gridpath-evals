# GridPath vs Claude Code — spreadsheet agent benchmark (August 2026, round 3)

Same tasks, same model (`claude-sonnet-5`), **same effort (`xhigh`) on both
sides**, two harnesses: **GridPath**, a purpose-built spreadsheet agent, and
**Claude Code 2.1.191** running headless with file access. Five runs per task
per harness. Every run is published — no cherry-picking — and build outputs
were additionally scored by blind judge panels (see `judges/`).

> **Round 3 supersedes round 2.** The original dataset ran GridPath at
> `medium` effort against Claude Code's `xhigh` default — not a fair fight in
> either direction. Those runs are preserved in
> `runs-2026-08-16-medium-effort/` for transparency; `runs/` is the
> effort-parity dataset.

## Results (effort parity, N=5 per cell)

| task | harness | clean runs | assertions | median duration | median output tokens |
|---|---|---|---|---|---|
| fixture-model-edit | GridPath | **5/5** | 40/40 | **7s** | 316 |
| fixture-model-edit | Claude Code | 3/5 | 38/40 | 96s | 8,700 |
| dh-income-statement | GridPath | 2/5 | 40/45 | **513s** | 55,084 |
| dh-income-statement | Claude Code | 2/5 | 37/45 | 864s | 78,265 |
| aapl-forecast | GridPath | 3/5 | 36/40 | 1,018s | 106,134 |
| aapl-forecast | Claude Code | 2/5 | 35/40 | 1,181s | 87,008 |

**Blind judge quality scores (builds; mean of 2 judges x 5 runs, 1-10):**

| task | GridPath | Claude Code |
|---|---|---|
| dh-income-statement | 5.3 | **6.9** |
| aapl-forecast | 5.5 | **8.3** |

The two-sentence summary: **on edits to existing workbooks GridPath wins
everything** (clean 10/10 across both edit tasks incl. the private one,
11-14x faster, 12-27x fewer tokens, and Claude Code silently dropped
original file parts in 2 of 5 fixture runs — `parts_preserved` in the grade
reports). **On builds from scratch, efficiency is at parity and automated
checks slightly favor GridPath, but blind professional judging clearly
favors Claude Code** — its models cited management guidance, used reported
line items, and organized into Assumptions/Model/Notes tabs. We publish
that result too; the gap is our engineering roadmap.

Cost at standard API rates (cache writes at the 1h rate): GridPath's 15
public runs = **$27.76** metered-equivalent (it actually ran on a flat
subscription); Claude Code's 15 runs = **$118.42** metered. Nearly all of
the difference is earned on edits.

## Corrections made during this benchmark, in the open

1. **Grader bug fixed in Claude Code's favor** — the AAPL revenue assertion
   matched `/revenue/i`; Apple's P&L says "net sales". All runs re-graded.
2. **Effort parity** — round 2 ran GridPath at `medium` vs Claude Code's
   `xhigh` default. Round 3 re-ran GridPath's entire lane at `xhigh`.
3. **"Quality tied" retracted** — it was true of assertion counts and false
   under blind professional judging; both metrics are now published.

## Layout

```
tasks/        task specs: prompt + layout-independent assertions
harness/      grade.mjs (grader), run-claude-code.mjs, run-gridpath.mjs, compare.mjs
fixtures/     make_rich_model.py generates rich-model.xlsx (committed)
runs/         ROUND 3 (effort parity): <task>/<timestamp>-{gridpath|cc}/
              with output.xlsx, output.grade.json (recalc-graded), meta.json,
              original.xlsx where the task has one
runs-2026-08-16-medium-effort/   round-2 GridPath runs (medium effort), preserved
judges/       blind-panel scores with full de-anonymization mapping
```

## Grading

Assertions are label-driven and layout-independent; value assertions are
graded after normalizing outputs through **LibreOffice headless
recalculation** (`--recalc`), so neither harness's formula-cache behavior
can tilt results. Structure assertions (formulas present, original zip
parts preserved) always read the raw output file.

Build quality is scored by **blind judge panels**: two independent LLM
judges per task, each reading all 10 candidates (both harnesses, shuffled,
anonymized, agent self-references redacted) against a finance-professional
rubric. Judge agreement was within one point on nearly every candidate.
`judges/*-scores.json` carries every score plus the candidate-to-run
mapping so you can verify the de-anonymization yourself.

Re-grade any output:

```bash
npm install exceljs jszip
node harness/grade.mjs tasks/aapl-forecast.json runs/aapl-forecast/<run>/output.xlsx --recalc
node harness/grade.mjs tasks/fixture-model-edit.json runs/fixture-model-edit/<run>/output.xlsx \
  --original runs/fixture-model-edit/<run>/original.xlsx --recalc
```

`--recalc` requires LibreOffice (`SOFFICE_PATH` overrides discovery).

## Reproducing the Claude Code lane

```bash
node harness/run-claude-code.mjs fixture-model-edit --model claude-sonnet-5 --yolo
```

The GridPath lane (`run-gridpath.mjs`) drives the GridPath app's
self-driving eval mode and requires a GridPath build; the published outputs
and grade reports let you verify our side without one.

## Honest-methodology notes

- N=5 per cell, one machine, one model, August 2026 — a snapshot, not a law.
- The dh prompt is deliberately ambiguous about a forecast; judges were
  instructed to score interpretation-neutrally (see judges/).
- Build outputs contain model-generated estimates of public-company
  financials. Benchmark artifacts, not financial guidance.
- A fourth internal task (same edit design on a proprietary bank model) is
  excluded because readers can't reproduce it; GridPath was 5/5 clean there
  as well.
