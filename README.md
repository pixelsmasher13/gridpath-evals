# GridPath spreadsheet-agent benchmark

Same tasks, same model (`claude-sonnet-5`), matched effort settings, two
harnesses: **GridPath**, a purpose-built spreadsheet agent, and **Claude
Code 2.1.191** running headless with file access. Every run is published —
outputs, grades, timings, tokens — no cherry-picking.

The suite focuses on the work people actually do with spreadsheets:
**changing a workbook that already exists.**

## MCP lane on a large real model (round 6, 2026-10-01)

Round 5 below used a small fixture. This round repeats the comparison,
**Claude Code alone** vs **Claude Code + the gridpath MCP server**, on a real
analyst model: 1.4 MB, 12 sheets, about 55,000 formulas (4,400 `INDIRECT`
calls, add-in functions, 3,400 named ranges, comments, VML drawings, custom
XML). Sonnet 5.5, Claude Code 2.1.261, same prompt in both lanes.

Five runs per cell, medians, gridpath 0.1.8:

| task | Claude Code alone | Claude Code + gridpath MCP |
|---|---|---|
| `p159-model-edit` — append a labelled row to a log sheet | 5/5 clean · 23 s · 2,510 out-tokens · $0.54 | 5/5 clean · **15 s** · **906** · **$0.36** |
| `p159-insert-row` — insert a row at the top of a table, everything below shifts | 5/5 clean · 52 s · 6,060 · $0.86 | 5/5 clean · **17 s** · **989** · **$0.35** |

One run per cell (a pilot, not a median):

| task | Claude Code alone | Claude Code + gridpath MCP |
|---|---|---|
| `p159-scenario` — change 12 input cells across four quarters, report the recalculated annual figure | correct · 161 s · 17,104 out-tokens · $1.62 | correct · **29 s** · **2,236** · **$0.49** |
| `p159-add-metric-row` — insert a row mid-block and fill 26 formulas; references below must follow | correct · 86 s · 9,805 · $1.36 | correct · **52 s** · **3,715** · **$0.66** |

**What this shows.** On a large workbook the MCP lane is 1.5–3× faster on
simple edits and about 5× faster on the scenario task, with 2.5–7.6× fewer
output tokens and 1.5–3.3× lower cost. The gap grows with the file and with
the task: without a calculation engine the plain lane has to inspect the XML,
write a script, and, when asked for a recalculated number, write its own
formula evaluator (that is where the 17,000 tokens in the scenario run went).

**What it doesn't show, and where we lose.**

- **Correctness is equal.** Every run in both lanes passed every assertion,
  and both lanes reported the right recalculated figure in the scenario task.
- **Fidelity is equal or slightly in the plain lane's favour.** Package parts
  byte-identical to the original: `p159-model-edit` 75/76 with the MCP,
  74–75/76 without; `p159-insert-row` 73/76 with the MCP, 72–74/76 without;
  `p159-add-metric-row` 73/76 with the MCP, 75/76 without; `p159-scenario`
  71/76 with the MCP (it rewrites every sheet whose stored results changed),
  75/76 without.
- **A bug the scenario task found in gridpath.** When an edit changes the
  value of a downstream formula cell, the save rewrites that cell to refresh
  its stored result and drops its dynamic-array flags (`cm`, `t="array"`).
  In the scenario run that touched 67 cells the task never asked for
  (`stray_changes` in `verify.json`); the plain lane touched only the twelve
  inputs. Values are unaffected, but it is a change nobody requested. Open
  as of 0.1.8.
- **The first sweep was slower, and it is published too.** On gridpath 0.1.7
  the MCP lane measured 28 s and 31 s on the two five-run tasks. Profiling
  showed about 16 s of that was the server, not the model (a quadratic pass
  over the named ranges and a reload after every save). 0.1.8 fixes both.
  Runs stamped `2026-10-01T14-0x` and `14-1x` are 0.1.7; `14-4x` onward are
  0.1.8; those stamped `2026-09-29` are one run per lane on an earlier
  development build.
- **Cost is mostly context, not output.** Output tokens are roughly a tenth
  of an MCP-lane run's cost; most of it is Claude Code re-reading its own
  context each turn, which the tool descriptions add to.

**Method notes for this round.** The model is a licensed file, so the
workbooks are not in this repo: `runs/p159-*` hold `meta.json` and the grade
report only, and the round cannot be reproduced from here. LibreOffice was
not installed on the benchmark machine, so value assertions read the saved
file without a recalculation pass, and the plain lane had no external
recalculation available either. The stricter cell-by-cell checks for the two
pilot tasks (`harness/verify-p159-complex.mjs`, results in `verify.json`)
load the original and the output into gridpath's own engine and compare; the
same evaluator is applied to both lanes' files.

## 0. MCP lane: your agent + GridPath's engine (round 5, 2026-09-30)

GridPath now also ships as an MCP server (`npx gridpath mcp`). This round
compares **Claude Code alone** with **Claude Code + the gridpath MCP server**
on three edit tasks on the public fixture, five runs per cell, Sonnet 5.5,
same prompt. The MCP lane's prompt adds one sentence telling the agent the
gridpath tools are connected for the edit.

| task | Claude Code alone | Claude Code + gridpath MCP |
|---|---|---|
| `fixture-add-year` — add an FY2029E column, formula pattern carried | 5/5 clean · 22 s · 2,559 out-tokens · $0.40 | 5/5 clean · **12 s** · **1,113** · $0.32 |
| `fixture-quarterly-totals` — margin column + FY total row, formulas | 5/5 clean · 26 s · 2,583 · $0.41 | 5/5 clean · **12 s** · **1,158** · $0.34 |
| `fixture-insert-row` — insert Opex row; CF/DV/merge/freeze must follow | 5/5 clean · 31 s · 3,382 · $0.47 | 5/5 clean · **21 s** · **2,223** · $0.51 |

Medians. Wall-clock is the whole `claude -p` run; cost is Claude Code's own
metering. Package parts byte-identical to the original after the edit: MCP
lane 14/16 on every run (the edited sheet plus `styles.xml` for new cell
formats), Claude Code 15/16 on 14 runs and 13/16 on one.

**What this shows, and what it doesn't.** With the MCP, edits are 1.5–2.2×
faster and use 1.5–2.3× fewer output tokens (a tool call instead of a Python
script); cost is about equal on files this small because the 31 tool
descriptions add input tokens. Correctness and fidelity are **equal**: on
Sonnet 5.5, plain Claude Code chose zip-level XML surgery on its own every
time and preserved every part. The reason to use the MCP is not that the
other path breaks files; it is the review step before anything is written,
that it preserves parts by construction rather than by the model's judgment
on the day, and that it needs nothing installed but Node (the plain lane used
Python, openpyxl and a LibreOffice recalc pass). The earlier rounds below,
where a coding agent dropped parts in 2 of 5 runs, were measured on the
August 2026 model generation.

Reproduce: `node harness/run-claude-code.mjs fixture-insert-row --mcp --yolo --model claude-sonnet-5-5`
(this round ran `gridpath@0.1.4`; the harness now pins 0.1.8). Runs are in `runs/<task>/2026-09-30*-cc-mcp/`
and `…-cc/`.

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
  The `p159-*` tasks run on a licensed analyst model: task specs, timings
  and grade reports are published, the workbooks are not.
- Build outputs contain model-generated estimates of public-company
  financials. Benchmark artifacts, not financial guidance.
