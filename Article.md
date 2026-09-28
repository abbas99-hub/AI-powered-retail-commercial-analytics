# Can AI Replace a Commercial Analyst? I Built an Enterprise Power BI Dashboard With Claude to Find Out

### A technical breakdown of building a full retail commercial analytics platform — semantic model, 85+ DAX measures, and an 8-page dashboard — using Claude, MCP server integration, and structured prompt engineering. Including every mistake the AI caught, and the moments it needed a human to step in.

---

## The Question I Actually Wanted Answered

Every few months, a new tool promises to make the "analyst" job obsolete. I've heard it about Excel macros, about self-service BI, about AutoML. I didn't believe the hype any of those times, and I didn't believe it this time either.

So instead of arguing about it, I ran an experiment: build a complete, production-grade commercial analytics solution in Power BI — from a raw retail dataset to a validated, styled, 8-page executive dashboard — using an AI system (Claude) connected directly to my live Power BI model via an MCP server, and see exactly where it succeeded, where it struggled, and where it flatly could not proceed without me.

This article is that breakdown. Not a highlight reel — the actual mechanics, the actual mistakes caught, and the actual limitations.

---

## What "MCP Integration" Actually Means Here (Not the Buzzword Version)

MCP (Model Context Protocol) is what let Claude stop being a chatbot that *talks about* Power BI and start being a tool that *operates on* a live Power BI Desktop instance.

Concretely: a local MCP server exposed Power BI Desktop's Analysis Services engine — the actual semantic model running in memory — as a set of callable operations: create a table, add a relationship, write a DAX measure, run a query, refresh a partition. Claude could connect to my open `.pbix` file, inspect its real structure, and make real, committed changes to it in the same session.

This is the detail that matters most and gets glossed over in most "AI built my dashboard" posts: **Claude wasn't generating code for me to paste.** It was running DAX queries against my actual 1-million-row fact table and reading back actual numbers, in the same conversation where it was deciding what to build next.

That distinction is the difference between an AI that *sounds* right and an AI that *is* right.

---

## The Architecture, in One Pass

**The dataset:** a retail chain — 1,000,000 transactions, 100,000 customers, 500 stores, 2,000 salespeople, 210 products, 50 marketing campaigns.

**The build, phase by phase:**

1. **Data profiling** — before touching a single visual, live DAX queries against every table: row counts, null checks, orphaned foreign keys, duplicate detection, value ranges.
2. **ETL hardening** — auditing the actual Power Query M code already in place, not assuming it was correct.
3. **Data modeling** — fixing relationship structure, resolving a hidden relationship cycle, correcting the date table wiring.
4. **Date intelligence** — enriching the calendar table, marking it as an official Date Table.
5. **DAX foundation → advanced DAX** — 85+ measures across Sales, Customer, Product, Store, Salesperson, Campaign, Time Intelligence, and Advanced Analytics.
6. **Dashboard UX/UI** — a custom Power BI theme, color system, and layout spec.
7. **Report build** — 8 full analytical pages with a persistent navigation sidebar.
8. **Validation** — every single measure cross-checked against independently computed figures before being called "done."

That's the skeleton. The interesting part is what happened *inside* each of those steps — specifically, the moments where an AI operating without real data access, or a junior analyst working fast under a deadline, would have shipped something wrong.

---

## The Prompt Engineering Pattern That Actually Made This Work

Before the mistake-catching stories, it's worth showing the actual mechanics — because the prompting style here mattered as much as the MCP connection did. Three patterns did most of the work.

**Pattern 1: Phase-gate everything, and force a status report before moving on.**

The very first prompt didn't ask for a dashboard. It asked for a *process*:

> "For every stage, provide: Objective, Recommended architecture, Exact implementation steps, Exact DAX measures, Data validation checks, Performance considerations... Do not skip ahead unless the required information is available... After completing each phase, clearly state: PHASE STATUS — Completed / Next / Validation Required."

That single formatting requirement — forcing an explicit `PHASE STATUS` block at the end of every response — is what made a 20-phase project auditable instead of a wall of unverifiable output. Every phase ended with something like:

> **PHASE STATUS**
> **Completed:** Full data profiling on live model — row counts, null checks, orphan FK checks... Zero data-quality defects found.
> **Next:** PHASE 2 — Power Query / ETL...
> **Validation Required:** Confirm you're OK with me proceeding against the actual live model (lowercase names) rather than renaming everything to match your brief's spec.

And the follow-up prompt was never "continue" alone — it was always confirmation-plus-continue:

> "Validation confirmed 1. Make as Date Table. 2. Proceed with calendar-year. Proceed with Next: Phase 5 - DAX Foundation..."

That habit — echoing back exactly what was being confirmed before saying "go" — is what prevented silent scope drift across a conversation this long.

**Pattern 2: Report the bug, not the diagnosis.**

When something broke, the most useful prompts described symptoms precisely and let the AI trace the cause, rather than guessing at a fix themselves:

> "as per How_To_Install_pbir.md, after copying definition folder, I opened .PBIP file in power bi and it is showing below error: Required property 'reportVersionAtImport' was not included in the /themeCollection/baseTheme property of report.json."

That's a copy-pasted error string, nothing more — and it was enough to trace to the exact missing schema property in one pass. Compare that to the later UI bug report, which deliberately included a screenshot and *specific, numbered complaints* rather than a vague "it looks wrong":

> "1. in slicers, values are not getting displayed, headers also not visible. 2. in Sales by segment pie chart, values are not clearly visible, make the values in 100k/1M, if percentage then, only 1 decimal values. 3. also color formatting make it more creative/attractive."

Numbered, specific, one issue per line — each one became its own isolated fix with its own validation step, instead of one vague "make it better" that would have been impossible to verify.

**Pattern 3: Push back with evidence, not just a complaint.**

The most important prompt in the whole project might be this one, sent after being told the report layer was a limitation that couldn't be worked around:

> "I want you to build complete visuals in Power BI. identify the solution to the problem and act accordingly."

That's not "try harder." It's a refusal to accept a stated limitation at face value, paired with a direct instruction to *find the actual workaround* — which is what led to discovering Power BI's PBIP project format as a way to author the report layer as files after all. The lesson generalizes: when an AI states a hard limitation, the right response is often to ask it to search for a workaround before accepting the limitation as final — sometimes there genuinely isn't one, but it's worth spending one prompt to find out.

---

## Where AI Caught What a Rushed Human Might Have Missed

This is the part I actually want recruiters and skeptics to read carefully, because "AI wrote some DAX" is not the interesting claim. **AI catching real data-integrity problems before they reached a dashboard** is.

### 1. The "Top Product" Trap

Two different products in the catalog were both named "Track Pants" — different brand, different SKU, same display name. In fact, only 186 of 210 products had unique names.

**What a human under deadline often does:** builds a "Top 10 Products" table using `product_name`, ships it, and never notices that two unrelated SKUs are silently displaying under one label — until a stakeholder in a meeting asks "wait, which Track Pants is this?"

**What happened here:** the AI caught the name collision *before* building the ranking measures, by checking `DISTINCTCOUNT(product_name)` against the row count first. Every table that displays a product name now shows brand alongside it. Same finding, worse magnitude, on the stores table: only 50 distinct store *names* across 500 physical *locations* — meaning "store name" alone was closer to a brand/chain label than a unique identifier. Store name is now always paired with location and store ID.

### 2. The Relationship Cycle Nobody Would Have Noticed Until It Broke a Report

The brief called for a direct relationship from `Stores` to `Salespersons` (to identify each store's manager). Building that as an **active** relationship would have created a cycle: the fact table already reaches the Salespersons table two ways — directly (who made the sale) and indirectly through Stores (who manages the store where the sale happened).

**What a human might do:** build the relationship, get a vague Power BI error about ambiguous filter paths, and either give up on the feature or — worse — force it active and get silently wrong numbers on a report six months later when someone builds a "sales by manager" visual.

**What happened here:** the relationship was deliberately built **inactive**, paired with a `USERELATIONSHIP()` DAX pattern for the one specific "sales by store manager" measure that needed it. Then — this is the part that matters — that measure was cross-validated: the top 5 "managed store" sales figures were checked against the top 5 store totals from an entirely separate part of the model, and they matched exactly, with every top manager correctly holding the job title "Manager." That's not "the formula ran without erroring." That's **proof the formula is right.**

### 3. The Metric That Almost Became a Lie

The brief asked for Campaign ROI. The dataset had a `campaign_budget` column.

**What a lot of dashboards do:** `ROI = Sales / Budget`, label it "ROI," ship it, and let a VP make a budget decision based on a number that was never actually a return-on-investment calculation.

**What happened here:** the AI refused to label it that way. True ROI requires knowing actual attributable marketing spend and having a non-campaign baseline to measure incremental lift against — neither existed in this dataset (every single transaction was already tagged to *some* campaign, meaning there was no "what would have happened anyway" baseline to compare against). The measure was built and named **"Sales to Budget Ratio,"** with an explicit warning baked into its own description field, so the next person who opens the model doesn't accidentally relabel it either.

This is the single clearest example in the whole project of AI doing *commercial* reasoning, not just technical execution — and it's worth noting the guardrail was set by the prompt, not invented by the AI on its own: the original brief explicitly said *"Do NOT calculate true ROI unless actual campaign spend is available. Do not label Sales/Budget as true ROI unless the business confirms."* The AI didn't independently decide to be cautious about ROI — it was told to be, in writing, up front. That's a prompt engineering lesson in itself: naming the specific mistake you don't want, in advance, is far more reliable than trusting an AI to intuit where the line is.

### 4. Auto Date/Time Was Quietly Breaking the Calendar

Power BI's own "Auto Date/Time" feature had generated three hidden duplicate date tables and silently connected the fact table's transaction date to one of *those* — not to the real, purpose-built date dimension sitting right there in the model.

**What a human might do:** build a date slicer, notice it "sort of" works, and never realize it's filtering against the wrong table entirely until a year-over-year comparison quietly returns nonsense.

**What happened here:** caught in the first data-profiling pass, fixed by disabling the feature and rebuilding the correct relationship — then verified by checking that a year-to-date total matched an independently computed running-total figure, to the cent.

### 5. A Genuine Business Insight, Not a Fabricated One

When asked to analyze revenue concentration (the classic 80/20 Pareto question every retail business wants answered), the AI found something worth knowing *before* presenting a chart: this business's revenue is **unusually evenly distributed** — no single product, store, or customer dominates. The top 10 products account for only ~4.9% of revenue against a proportional ~4.8% baseline. That's a real finding, cross-checked against three independent measures, and it changes how the concentration chart should even be interpreted by a stakeholder — a nearly flat Pareto curve, not the usual hockey stick.

The point isn't that the number is impressive. It's that the AI reported what the data actually showed instead of assuming the textbook 80/20 pattern would appear, and said so explicitly.

---

## The Moment the Report Layer Broke — and How the AI Actually Debugged It

Here's a failure story, because it's more instructive than another success story.

Power BI's report-canvas storage format changed — Microsoft moved from a single `report.json` file (legacy) to a split-folder "PBIR" format, and the switch happened live during this project's timeline. The first report package built didn't load. Power BI returned a specific schema error.

The AI didn't guess. It searched for the exact error, identified that Power BI Desktop had changed its default save format entirely, rebuilt the file generator against the new schema, and — critically — **validated every fix programmatically** before handing it back: parsing every JSON file, checking every visual's coordinates against the canvas bounds, and cross-referencing all 61 distinct field references in the report against the live model's actual measure and column list, catching zero mismatches before the file was ever opened in Desktop.

The prompt that triggered the actual fix was three lines, reporting the symptom and nothing else:

> "great its working. next step: 1. UI/UX looks very poor, junior portfolio project. I need enterprise level dashboard for stakeholders... 2. add other remaining pages"

Notice what that prompt does *not* contain: no diagnosis, no suggested fix, no Power BI schema knowledge. Just an honest, specific reaction ("looks like a junior portfolio project") plus a clear bar to clear ("enterprise level... for stakeholders"). That's the whole trick to prompting a technical AI collaborator on a subjective problem: state the gap in outcome, not your guess at the mechanism, and let it find the mechanism.

That last step is worth sitting with. A hallucinated field name in a hand-typed DAX formula is the single most common way an AI-assisted analytics project quietly ships something broken. Here, it was checked against the live model, not assumed.

---

## What the AI Could Not Do — and Why That Matters More Than What It Could

This is the section that actually answers the question in the title, so I'm not going to soften it.

**1. It cannot see its own output.**
Claude generated 76 dashboard visuals across 8 pages and validated every data reference programmatically — but it cannot open Power BI Desktop and look at the screen. When a color scheme looked flat, when slicers rendered with no visible values, when visuals overlapped after a file was reinstalled incorrectly — a human had to notice, screenshot it, and describe what was actually wrong. AI-driven analytics without a human in the loop watching the actual rendered output is not a finished product; it's a draft.

**2. It cannot create net-new tables in a live semantic model through this integration.**
Every dimension table, every measure table, had to already exist or be manually created in Power BI Desktop (a 10-second "Enter Data" action) before the AI could act on it. A real architectural boundary, not a workaround — worth knowing before promising a fully hands-off build.

**3. It will not invent a business definition.**
When asked to build "New Customers," it built it — but explicitly labeled it as a first-purchase-date proxy, because no account-signup date existed in the data, and said so rather than quietly presenting a proxy as a fact. When "High Value" and "Churn Risk" customer segments turned out to already exist as real labels in the data, it checked before building a synthetic threshold that would have duplicated something already there. That checking-before-assuming behavior is a design choice, not a limitation — but it means **someone still has to be available to answer the question when AI can't resolve it on its own**, like confirming a fiscal year start date, or deciding whether campaign budget represents real attributable spend.

**4. It cannot fix a file-system problem it can't see.**
When old and new report files ended up overlapping on disk because of an incomplete manual reinstall, the AI diagnosed the *cause* correctly from a screenshot (a classic leftover-file problem from PBIR's per-visual folder structure) — but it could not reach into the user's file system and delete the stale folder itself. It could only explain, precisely, what to check and what to remove.

**5. Design taste is still a human judgment call.**
The AI built a full color theme, spacing system, and layout — genuinely competent, defensible design decisions. But "does this look right for *our* stakeholders" is a taste question, and every round of this project ended with a human looking at the result and asking for something to be more attractive, more spaced out, more on-brand. That's not a failure of the AI. That's the actual, permanent shape of the collaboration.

---

## So — Can AI Replace a Commercial Analyst?

No. And having watched exactly where it stopped and handed control back, I'm more confident in that "no" than I was before I ran this experiment.

But the honest second half of that answer matters more than the first half:

**We are not moving toward AI replacing commercial analysts. We are moving toward AI-integrated commercial analytics** — where the analyst's actual job shifts from hand-writing every DAX formula and manually profiling every table, to directing the work, validating the output, making the judgment calls AI explicitly can't make, and catching the last 10% that only a human eye can see.

The analysts who lose ground here aren't losing it to AI. They're losing it to other analysts who figured out how to make AI do the profiling, the validation, and the first-draft modeling — freeing themselves up to spend their time on the commercial judgment that was always the actual job.

**AI won't replace analysts. It will replace the ones who don't integrate it.**

---
