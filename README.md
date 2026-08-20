# GridPath spreadsheet-agent benchmark

Same tasks, same model (`claude-sonnet-5`), matched effort settings, two
harnesses: **GridPath**, a purpose-built spreadsheet agent, and **Claude
Code 2.1.191** running headless with file access. Every run is published —
outputs, grades, timings, tokens — no cherry-picking.

The suite focuses on the work people actually do with spreadsheets:
**changing a workbook that already exists.**

## 1. Quarterly roll-forward (`pltr-rollforward`)

The headline test. Start with a real quarterly Palantir model — 3 sheets,
**663 live formulas**, an assumptions tab driving the forecast columns,
segment splits, growth rows, and a check row reconciling two independent
revenue builds. Its actuals stop at Q1 2026. The ask is one sentence:
*"update this model with Palantir's Q2 2026 actual results."*

| | GridPath | Claude Code |
|---|---|---|
| assertions passed | 9 / 10 | **10 / 10** |
| time | **259 s** | 1,014 s |
| API cost | **$2.34** | $8.76 |
| turns / batches | 1 batch | 77 turns |

Both tied revenue to the reported **$1,935,464 thousand**, tied net income
and diluted EPS, flipped the estimate column to actual, kept the check row
at zero, preserved all 663 formulas and all 13 internal file parts.
GridPath's single miss: one stale line of explanatory text on the
assumptions tab still describing Q2'26 as a forecast driver.

The fixture was **built by Claude Code** (`pltr-q1-model`, also published
here) so neither harness is editing a workbook shaped to suit it.

> Fixture caveat: Palantir discloses Government/Commercial and
> US/Rest-of-World, but not the four-way cross. The fixture's four-way
> segment split is therefore partly derived — US figures from rounded
> press-release highlights, International back-solved as residuals — and
> carries more apparent precision than the disclosures support. Realistic
> for an analyst model; worth knowing when reading it. Total revenue, net
> income and EPS all tie exactly to SEC XBRL, and those are what the
> assertions grade.

## 2. Edit fidelity (`fixture-model-edit`, `gs-model-edit`)

A one-row memo added to a feature-dense workbook, with the instruction
*"change nothing else."* What's measured is everything you didn't ask it
to touch. N=5 per harness.

| task | harness | clean runs | all file parts intact | median time | median output tokens |
|---|---|---|---|---|---|
| fixture-model-edit | GridPath | **5/5** | **5/5** | **7 s** | 316 |
| fixture-model-edit | Claude Code | 3/5 | 3/5 | 96 s | 8,700 |
| gs-model-edit (private file) | GridPath | **5/5** | 5/5 | **16 s** | 714 |
| gs-model-edit (private file) | Claude Code | 5/5 | 5/5 | 177 s | 14,903 |

Claude Code silently dropped parts of the workbook in **2 of 5** runs on
the feature-dense fixture — the edit lands, the file opens, and content
the library doesn't model is gone. Structural, not a bug: it edits by
loading the file through a Python library and writing the whole thing back
out.

## 3. Model builds from scratch (`dh-*`, `aapl-forecast`)

Also published, with blind judge panels — see `judges/`. Summary: at
matched effort the two harnesses are close on automated checks and on
cost, while blind professional judging favours Claude Code's builds
(dh 6.9 vs 5.3; aapl 8.3 vs 5.5 on a 1–10 rubric). Full scores, per-run
mapping and judge prompts are in `judges/*-scores.json`.

## Corrections made during this benchmark, in the open

1. **Grader bug fixed in Claude Code's favour** — the AAPL revenue
   assertion matched `/revenue/i`; Apple's P&L says "net sales".
2. **Second grader bug, also in Claude Code's favour** — the AAPL EPS
   assertions matched `/eps/i`, but filings say "Diluted earnings per
   share". After re-grading every run, Claude Code's AAPL clean rate went
   from 2/5 to **5/5** (40/40 assertions) and GridPath's from 3/5 to 4/5
   (38/40). The tables above and in `judges/` reflect the corrected data.
3. **Effort parity** — an earlier round ran GridPath at `medium` against
   Claude Code's `xhigh` default. Everything here is matched-effort;
   the older runs are preserved in `runs-2026-08-16-medium-effort/`.

## Layout

```
tasks/        task specs: prompt + layout-independent assertions
harness/      grade.mjs (grader), run-claude-code.mjs, run-gridpath.mjs, compare.mjs
fixtures/     rich-model.xlsx (make_rich_model.py) + pltr-q1-2026-model.xlsx
runs/         <task>/<timestamp>-{gridpath|cc}/ — output.xlsx, output.grade.json,
              meta.json, original.xlsx where the task edits an existing file
judges/       blind-panel build scores with full de-anonymization mapping
```

## Grading

Assertions are label-driven and layout-independent. Value assertions are
graded after normalizing the output through **LibreOffice headless
recalculation** (`--recalc`), so neither harness's formula-cache behaviour
can tilt results; structure assertions (formulas present, original zip
parts preserved, stale labels retired) read the raw file.

```bash
npm install exceljs jszip
node harness/grade.mjs tasks/pltr-rollforward.json runs/pltr-rollforward/<run>/output.xlsx \
  --original fixtures/pltr-q1-2026-model.xlsx --recalc
```

`--recalc` requires LibreOffice (`SOFFICE_PATH` overrides discovery).

## Reproducing the Claude Code lane

```bash
node harness/run-claude-code.mjs pltr-rollforward --model claude-sonnet-5 --yolo
```

The GridPath lane (`run-gridpath.mjs`) drives the app's self-driving eval
mode and needs a GridPath build; the published outputs and grade reports
let you verify our side without one.

## Honest-methodology notes

- One machine, one model, August 2026. N=5 on the edit tasks; the
  roll-forward head-to-head is a single run per harness so far.
- Agents are stochastic — medians and rates beat anecdotes, but this is a
  snapshot, not a law.
- Claude Code is the general-purpose baseline, not a survey of
  spreadsheet-specific tools.
- `gs-model-edit` runs on a proprietary bank model and is excluded from
  this repo; its results are reported above but not reproducible here.
- Build outputs contain model-generated estimates of public-company
  financials. Benchmark artifacts, not financial guidance.
