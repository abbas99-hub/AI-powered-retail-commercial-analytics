# Implementation Phases

This project was built in 20 gated phases, each requiring an explicit validation checkpoint before
the next began. That structure — not just the DAX or the model — is a core part of what made the
result trustworthy rather than merely plausible-looking. Full phase-by-phase detail lives in the
project's original working session; this document summarizes what each phase actually produced.

| Phase | What it did | Key output |
|---|---|---|
| 1. Data Profiling | Live DAX queries against every table — row counts, nulls, orphans, ranges | `01-data-profiling.md` |
| 2. Power Query / ETL | Audited existing M code; added error resilience, currency typing, path parameterization | Hardened, portable Power Query layer |
| 3. Data Modeling | Fixed the date relationship, removed Auto Date/Time artifacts, resolved the store-manager relationship cycle | `04-architecture.md` |
| 4. Date Intelligence | Enriched `dim_dates` with 7 display/sort columns; marked as official Date Table | Working calendar hierarchy |
| 5. DAX Foundation | Core Sales measures (7) | `_Measures` table established |
| 6. Advanced DAX | Customer (9), Product (7), Store (7), Salesperson (7), Campaign (6) measures | 43 measures, each independently validated |
| 7. Time Intelligence | MTD/QTD/YTD, period-over-period growth, rolling windows (13 measures) | Full time-intelligence layer |
| 8. Advanced Analytics | Concentration, Pareto, volatility, best/worst period measures (10) | 76 measures total, final count |
| 9. Dashboard UX/UI | Custom Power BI theme (color system, typography, spacing rules) | Theme JSON |
| 9 (cont.) | Executive Overview page build | First working report page |
| — | UI/UX redesign pass: enterprise polish, vibrant color system, per-visual card styling | Visual quality bar raised |
| — | All 7 remaining analytical pages built | Sales, Product, Store, Customer, Salesperson, Campaign, Data Quality pages |
| — | Slicer, chart-label, and color bug fixes | Diagnosed and resolved with live-data validation |
| — | Persistent sidebar navigation across all 8 pages | Native Power BI Page Navigator, not custom buttons |
| — | Overlap bug diagnosis | Traced to incomplete file replacement, not a layout defect |

## The validation discipline that ran through every phase

No phase was marked complete on the strength of a formula "running without erroring." Every
measure was checked one of three ways before being trusted:

1. **Cross-reconciliation** — a new measure's output checked against an independently computed
   total from an earlier phase (e.g., `Sales YTD` at any date always equals `Running Total Sales`
   at that same date — if they ever diverge, something broke).
2. **Known-answer testing** — e.g., confirming `weekday_name` mapping against the real calendar
   (Jan 1, 2024 actually was a Monday) before trusting the label.
3. **Edge-case testing** — confirming `DIVIDE()` returns blank rather than erroring on a
   zero-denominator filter context, confirming `USERELATIONSHIP()` reproduces an independently
   known result exactly.

## What "gated" actually meant in practice

Each phase ended with an explicit status block — what was completed, what came next, and what
needed explicit confirmation before proceeding. Nothing moved forward on an assumption when a
one-line confirmation could remove the ambiguity instead. See `ARTICLE.md` for real examples of
the prompts that drove this pattern.
