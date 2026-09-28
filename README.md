# AI-Native Retail Commercial Analytics Platform

**An enterprise-grade Power BI solution built end-to-end with Claude, connected live to Power BI Desktop via MCP (Model Context Protocol) — from raw data profiling to a validated 8-page executive dashboard.**

[![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=flat&logo=powerbi&logoColor=black)](.)
[![DAX](https://img.shields.io/badge/DAX-76%20Measures-blue)](docs/03-dax-measure-library.md)

📄 **Read the full technical write-up:** [`Article.md`](Article.md) — *"Can AI Replace a Commercial Analyst?"*

---

## Introduction

This repository documents a complete commercial analytics build for a retail business —
1,000,000 transactions across 100,000 customers, 500 stores, 2,000 salespeople, 210 products, and
50 marketing campaigns — built through a live connection between an AI system (Claude) and a real
Power BI Desktop session via an MCP server.

This is not a "prompt generated some DAX" project. Every measure, every relationship decision, and
every visual was built by querying the actual live model, validated against real computed results,
and gated behind an explicit review checkpoint before moving forward. The goal wasn't to see if AI
could *produce* a dashboard — it was to see if AI could produce one **an analyst could actually
trust**, and to document exactly where it couldn't do that alone.

## What makes this project different

Most "AI built my dashboard" projects show the output. This one shows the **audit trail**:

- **5 real data/modeling issues caught before they reached a stakeholder** — non-unique product
  and store names, a relationship cycle, a broken date relationship hiding inside Power BI's own
  Auto Date/Time feature, and a metric that was almost mislabeled as ROI. Full evidence in
  [`docs/02-data-quality-findings.md`](docs/02-data-quality-findings.md).
- **Every one of 76 DAX measures cross-validated** against independently computed figures — not
  just "the formula ran without erroring." See [`docs/03-dax-measure-library.md`](docs/03-dax-measure-library.md).
- **A documented list of what AI genuinely could not do** — see the "Limitations" section below
  and the closing section of [`ARTICLE.md`](ARTICLE.md).

## Repository structure

```
.
├── README.md                          <- you are here
├── ARTICLE.md                         <- full technical write-up (Medium-published)
├── LICENSE
├── docs/
│   ├── 01-data-profiling.md           <- baseline data audit, before any modeling
│   ├── 02-data-quality-findings.md    <- the 5 issues caught, with evidence
│   ├── 03-dax-measure-library.md      <- all 76 measures, organized and documented
│   ├── 04-architecture.md             <- star schema, relationships, why each decision was made
│   └── 05-implementation-phases.md    <- the 20-phase gated build process
├── data/
│   ├── README.md                      <- schema reference + sample-data notes
│   └── sample/                        <- illustrative sample CSVs (real dataset not included)
└── power-bi/
    ├── README.md                      <- how to rebuild and install this project
    ├── theme/                         <- custom Power BI theme (JSON)
    └── report-definition/             <- the report canvas, as portable PBIR JSON
```

## The dashboard

8 analytical pages behind a persistent navigation sidebar:

| Page | Focus |
|---|---|
| Executive Overview | Company-wide KPIs, sales trend, top products/stores |
| Sales Performance | Daily/weekly/category/store-type sales patterns |
| Product & Category Analytics | Catalog performance, ranking, contribution |
| Store Performance | Network performance, ranking, contribution |
| Customer Analytics | Segment behavior, repeat-purchase rate, top spenders |
| Salesperson Analytics | Staff performance, role breakdown, managed-store sales |
| Campaign Performance | Budget efficiency (explicitly not labeled as ROI — see findings) |
| Data Quality & Validation | 10 live referential-integrity tripwires |

## Key technical decisions (the interesting part)

| Decision | Why |
|---|---|
| `Sales to Budget Ratio`, not `ROI` | No confirmed marketing spend, no non-campaign baseline to measure lift against |
| Store-manager relationship built **inactive** + `USERELATIONSHIP()` | An active relationship would create a filter-path cycle |
| `product_name`/`store_name` always paired with a disambiguating field | Neither is a unique identifier in the real data (186/210 and 50/500 distinct respectively) |
| Manual fix to Auto Date/Time | It was silently relating the fact table to a hidden duplicate date table |
| Fiscal calendar not built | No fiscal-year-start month was confirmed — fabricating one risks silently wrong numbers |

Full reasoning for each: [`docs/02-data-quality-findings.md`](docs/02-data-quality-findings.md) and [`docs/04-architecture.md`](docs/04-architecture.md).

## Limitations — what AI could not do alone

- **Could not see its own rendered output.** Every visual needed a human to actually open Power BI
  and confirm it looked right — bugs like overlapping visuals from an incomplete file replacement
  were only caught by a human screenshot.
- **Could not create net-new tables in the live model** through the MCP connection — required a
  manual "Enter Data" step in Desktop first.
- **Would not invent a business definition** — flagged assumptions (like using first-purchase-date
  as a "new customer" proxy) explicitly rather than presenting them as fact, and asked rather than
  guessed when a definition was genuinely ambiguous (fiscal year, campaign spend attribution).
- **Could not fix filesystem-level problems it couldn't observe** — diagnosed a leftover-file bug
  correctly from a screenshot, but couldn't delete the stale folder itself.
- **Final design taste remained a human call** — every UI iteration ended with a person judging
  whether it actually looked right for the intended audience.

Full discussion: [`ARTICLE.md`](ARTICLE.md#what-the-ai-could-not-do--and-why-that-matters-more-than-what-it-could).

## Getting started

See [`power-bi/README.md`](power-bi/README.md) for full setup instructions, and
[`data/README.md`](data/README.md) for the sample dataset and schema reference.
