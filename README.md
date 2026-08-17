# GridPath vs Claude Code — spreadsheet agent benchmark (August 2026)

Same tasks, same model (`claude-sonnet-5`), two harnesses: **GridPath**, a
purpose-built spreadsheet agent, and **Claude Code 2.1.191** running headless
with file access. Five runs per task per harness, collected 2026-08-16/17 on
one machine. Every run in this repo is published — no cherry-picking; the
medians below are over all five runs of each cell.

## Results

| task | harness | clean runs | assertions | median duration | median output tokens |
|---|---|---|---|---|---|
| fixture-model-edit | GridPath | **5/5** | 40/40 | **6.2s** | 260 |
| fixture-model-edit | Claude Code | 3/5 | 38/40 | 96s | 8,700 |
| dh-income-statement | GridPath | 0/5 | 33/45 | **178s** | 16,158 |
| dh-income-statement | Claude Code | 2/5 | 37/45 | 864s | 78,265 |
| aapl-forecast | GridPath | 1/5 | 32/40 | **253s** | 25,300 |
| aapl-forecast | Claude Code | 2/5 | 35/40 | 1,181s | 87,008 |

Totals: assertions GridPath 105/125 vs Claude Code 110/125; clean runs 6/15
vs 7/15. **Quality is effectively tied. The differences are speed (5–15×),
output tokens (3–36×), and fidelity: Claude Code silently dropped original
workbook parts (charts, conditional formatting, validation, defined names)
in 2 of 5 edit runs — `parts_preserved` in the grade reports; GridPath
dropped none in any run.** Claude Code's 15 runs cost $118.42 metered.
GridPath ran on a flat subscription; pricing its exact token usage at the
same standard API rates (cache writes at the 1-hour rate) gives a metered
equivalent of **$9.21** — 13× cheaper for the same work.

## Layout

```
tasks/        task specs: prompt + layout-independent assertions
harness/      grade.mjs (grader), run-claude-code.mjs, run-gridpath.mjs, compare.mjs
fixtures/     make_rich_model.py generates rich-model.xlsx (committed), the
              feature-dense workbook behind fixture-model-edit
runs/<task>/<timestamp>-{gridpath|cc}/
              output.xlsx        the workbook the agent produced
              output.grade.json  per-assertion pass/fail (recalc-graded)
              meta.json          model, timing, tokens (and cost for CC)
              original.xlsx      starting file, where the task has one
```

## Grading

Assertions are label-driven and layout-independent (find the row whose label
matches a regex, check its cells), so different layouts for the same task
grade fairly. Value assertions are graded after normalizing the output
through **LibreOffice headless recalculation** (`--recalc`): a throwaway
profile seeded with "always recalculate on load" evaluates every formula and
writes fresh cached values, so grading doesn't depend on either harness's
cache behavior. Structure assertions (formulas present, original zip parts
preserved) always read the raw output file.

Re-grade any output yourself:

```bash
npm install exceljs jszip
node harness/grade.mjs tasks/aapl-forecast.json runs/aapl-forecast/<run>/output.xlsx --recalc
# the fixture edit task also checks round-trip fidelity against the original:
node harness/grade.mjs tasks/fixture-model-edit.json runs/fixture-model-edit/<run>/output.xlsx \
  --original runs/fixture-model-edit/<run>/original.xlsx --recalc
```

`--recalc` requires LibreOffice (`SOFFICE_PATH` overrides discovery).

## Reproducing the Claude Code lane

```bash
node harness/run-claude-code.mjs fixture-model-edit --model claude-sonnet-5 --yolo
```

runs the task headless in an isolated workdir, then grades. `--yolo` skips
permission prompts (required for unattended runs — only use with prompts you
trust). The GridPath lane (`run-gridpath.mjs`) drives the GridPath app's
self-driving eval mode and requires a GridPath build; the published outputs
and grade reports let you verify our side of the table without one.

## Honest-methodology notes

- **One assertion was corrected during analysis, in Claude Code's favor.**
  The AAPL revenue assertion originally matched labels against `/revenue/i`;
  Apple's own P&L calls the line "net sales", and all five Claude Code runs
  had the correct figure under that label. The assertion now accepts both,
  and every run was re-graded before publishing.
- **Build-task accuracy is hard for both harnesses.** The recurring misses
  are omitted anchor rows (FY-actuals columns, net income, CAGR summaries),
  not wrong arithmetic. Neither harness "wins" builds; the efficiency gap is
  the story.
- Both lanes used the same model at each vendor-default effort setting;
  agents are stochastic, so treat any single run as a sample, not a verdict.
  Timing medians exclude nothing — failed assertions still count their run.
- Build outputs contain model-generated estimates of public-company
  financials. They are benchmark artifacts, not financial guidance.
