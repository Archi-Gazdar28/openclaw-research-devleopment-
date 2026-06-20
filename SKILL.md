---
name: rnd-company-intelligence
description: Generate a full R&D intelligence report for a given Company + Product, covering company/product details, turnover (financials), patents, trend analysis, competitor analysis, related research papers, and a synthesized SWOT. After the report, offer to export it as a PDF or drill into any section, then offer a technology-stack step covering both the product's detected current stack and a recommended stack for building something similar. Trigger this whenever the user gives a company name and a product name and asks for research, due diligence, market analysis, competitive analysis, patent search, SWOT, financial/turnover lookup, trend report, or a "company + product report" — even if they only name the company and product without using the word "report".
license: Apache-2.0
metadata:
  author: openclaw-user
  version: "1.4.0"
  openclaw:
    requires:
      env:
        - SERPAPI_KEY
        - CRUNCHBASE_API_KEY
      bins:
        - python3
      python:
        - requests>=2.31.0
        - matplotlib>=3.7.0
      primaryEnv: SERPAPI_KEY
---

# R&D Company & Product Intelligence

This skill turns the agent into an R&D / market-intelligence analyst. Given just a **company name** and a **product name**, it pulls structured data from a handful of paid APIs, synthesizes it into a seven-section report, and then offers two follow-on steps: exporting the report and recommending or detecting a technology stack.

## When to use this skill

Trigger this skill whenever the user:

- Gives a company name + product name and asks for research, a report, "intel", due diligence, or an overview
- Asks for any single piece that this skill produces on its own — turnover/revenue, patents, trend analysis, competitor analysis, related research papers, or a SWOT — for a named company/product
- Asks to export that kind of report as a PDF
- Asks "what tech stack does [company] use for [product]" or "what stack should I use to build something like [product]"

If the user only gives a company name with no product, or only a product with no company, ask for the missing piece before calling any API — every downstream command takes both.

## Inputs

### Required

| Input | Notes |
|---|---|
| Company name | Used to resolve a Crunchbase organization and to scope patent/competitor/financial lookups |
| Product name | Used to scope patents, trend analysis, and research-paper search; without it, results skew to the whole company instead of the specific product |

If either is missing or ambiguous (e.g., "Apple" could be the company or several unrelated entities), ask a single clarifying question rather than guessing.

### Optional — ask for these, or use them if the user already mentioned them

None of these block the report — the script will resolve the same things itself via search — but each one removes a disambiguation step and sharpens a specific section. If the user's request already contains any of them, pass them straight through instead of re-deriving them. Don't interrogate the user for all of them up front; one good clarifying question covering the 2–3 most relevant to the request is enough, and it's fine to proceed with sensible defaults if the user just wants the report now.

| Input | Sharpens | Why it helps |
|---|---|---|
| Company website / domain | Tech-stack detection, Crunchbase match | Skips name-collision guesswork and is required anyway for `tech-stack-detect` |
| Stock ticker | Turnover | Goes straight to Financial Modeling Prep instead of guessing public vs. private |
| Public or private | Turnover routing | Tells you immediately whether to expect an income statement or a funding history |
| Industry / sector | Competitor analysis, SWOT | Narrows the "similar companies" search so it doesn't return unrelated firms that happen to share a category tag |
| Product category (SaaS, hardware, mobile app, pharma, IoT, etc.) | Recommended tech stack, patent classification | The recommended stack and the relevant patent classes differ enormously by category |
| HQ country / region | Crunchbase disambiguation, trend geo | Resolves same-name companies in different countries and sets a sensible default `--geo` for trends |
| Known competitors (a few names) | Competitor analysis | Seeds the merge/dedupe step so the section starts from something concrete instead of a cold search |
| Specific technology / keyword terms | Patents, research papers | The product name alone is sometimes too broad ("Cybertruck") or too narrow ("ChatGPT" vs. the underlying model); a keyword like "structural battery pack" or "transformer architecture" narrows precisely |
| Time window (e.g., last 3/5/10 years) | Patents, trends, research papers | Overrides the script's default lookback when the user cares about a specific period |
| Target customer segment (B2B / B2C / enterprise / consumer) | Recommended tech stack, SWOT | Changes the scale and compliance assumptions baked into the recommendation |

If several of these are already implied by the conversation (e.g., the user is clearly talking about a public company, or already named two competitors), use them without asking again.

## API configuration

This skill is built on paid APIs rather than ad-hoc web search, so results are structured and repeatable. None of these need the user to give a URL or workspace — the helper script has the right endpoints baked in.

| Capability | Service | Env var | Required? |
|---|---|---|---|
| Company profile, funding history, similar companies | Crunchbase API | `CRUNCHBASE_API_KEY` | yes |
| Web/news search, Google Patents, Google Trends, Google Scholar | SerpApi | `SERPAPI_KEY` | yes |


## How to use this skill

The skill ships with a single Python helper at `scripts/rnd_report.py` that wraps every API call. Run it from the agent's shell — don't hand-roll new HTTP clients for these services.

```bash
python3 {baseDir}/scripts/rnd_report.py <command> [args...]
```

Get the full command list at any time with:

```bash
python3 {baseDir}/scripts/rnd_report.py --help
```

### Core workflow

1. **Confirm inputs** — company name and product name, plus any of the optional enrichment inputs above that are already implied or worth one quick clarifying question. Pass whatever you have straight into the matching flags below; don't make the script re-derive something the user already told you.
2. **Run the seven data-gathering commands** (below) — these are deterministic API calls, so let the script do them; don't try to answer from memory.
3. **Synthesize the SWOT yourself.** Unlike the other six sections, SWOT isn't a single API call — it's your own analysis of the six data sections you just gathered (a strong patent portfolio is a Strength, a thin one next to a dominant competitor is a Weakness, a rising trend line is an Opportunity, and so on). Don't skip straight to generic SWOT boilerplate; ground every bullet in something the report actually surfaced.
4. **Render the report** using the template in "Output format" below — including the charts described there for the Turnover, Trend Analysis, and Competitor Analysis sections, which are the report's market-analysis core.
5. **Offer the follow-up menu** (PDF export / drill into a section / stop here).
6. **If the user wants the tech-stack step**, run both the detection and recommendation sub-steps described below.

### Command reference

**Company & product detail**
- `company-details --company "..." --product "..." [--domain example.com] [--hq-country US]` — Crunchbase org profile (description, founding date, HQ, employee count, leadership) plus a SerpApi knowledge-panel pass for the specific product. `--domain` and `--hq-country` resolve name collisions instead of guessing from search results.

**Turnover (financials)**
- `turnover --company "..." [--ticker SYM] [--public | --private]` — if the company is public (or `--ticker` is given), pulls revenue/income statement from Financial Modeling Prep; otherwise falls back to Crunchbase total funding raised and latest valuation, clearly labeled as funding data, not revenue, since private companies rarely disclose turnover. Pass `--public`/`--private` if the user already told you, so the script doesn't waste a call probing FMP for a private company.

**Patents**
- `patents --company "..." --product "..." [--keywords "term1,term2"] [--since YYYY] [--limit N]` — SerpApi's Google Patents engine, scoped to assignee = company and keyword = product; returns title, filing date, status, and a short abstract per patent. `--keywords` narrows or redirects the search when the product name alone is too broad or too narrow for the underlying technology; `--since` limits to recent filings.

**Trend analysis**
- `trends --product "..." [--geo US] [--since YYYY]` — SerpApi's Google Trends engine; returns interest-over-time and interest-by-region for the product/category. Defaults to a 5-year window and worldwide geo if neither is given.

**Competitor analysis**
- `competitors --company "..." --product "..." [--industry "..."] [--known "Name A,Name B"] [--limit N]` — Crunchbase "similar companies" plus a SerpApi web search for "{product} alternatives" / "{company} competitors"; dedupes and merges both lists. `--industry` keeps Crunchbase's similarity search from drifting into unrelated companies; `--known` seeds the merge with names the user already gave you so they're confirmed and enriched rather than re-discovered.

**Research papers**
- `research-papers --product "..." [--keywords "term1,term2"] [--since YYYY] [--limit N]` — SerpApi's Google Scholar engine, scoped to the product/technology keywords; returns title, authors, year, citation count, and link.

**Tech stack — detected (current)**
- `tech-stack-detect --domain example.com` — BuiltWith API lookup against the product's live domain; returns frontend/backend frameworks, hosting, analytics, and CMS detected on the site. Requires `BUILTWITH_API_KEY`; if unset, tell the user this part needs to be skipped or answered via web search instead. If the user already gave a domain in step 1, reuse it here instead of asking again.

**Orchestration**
- `full-report --company "..." --product "..." [--domain ...] [--ticker ...] [--industry ...] [--keywords ...] [--known ...] --output report.json` — runs company-details, turnover, patents, trends, competitors, and research-papers in sequence and writes one combined JSON file, threading through any optional flags the user supplied. Use this instead of calling each command separately when the user wants the whole report — it's one process instead of six, and the script auto-throttles SerpApi calls if needed.

### Tech-stack step (after the main report)

This is a separate, optional step the user opts into after seeing the report — don't run it automatically as part of the main seven sections.

The user already told us this step should cover **both** angles:

1. **Detected current stack** — run `tech-stack-detect` against the product's actual website. If you don't have the domain yet, ask, or infer it from the Crunchbase profile pulled in step 1. If `BUILTWITH_API_KEY` isn't configured, say so plainly and offer to do a best-effort web search instead.
2. **Recommended stack to build something similar** — this is reasoning, not an API call. Base it on the product category and target customer segment (use what the user gave you in step 1, or ask now if you skipped it earlier) and on what you learned in steps 1–6 about scale, integrations, and competitors. Present it as layers (frontend / backend / data / infra) with a one-line rationale per choice, not just a name-drop list.

## Output format

### Main report — always use this structure

```markdown
# R&D Intelligence Report: {Product} by {Company}

## 1. Company & Product Overview
## 2. Turnover / Financials
## 3. Patents
## 4. Trend Analysis
## 5. Competitor Analysis
## 6. Related Research
## 7. SWOT Analysis
```

- For sections 2–6, render results as a compact markdown table (e.g., patents: title, filing date, status, link). Don't paste raw JSON. Keep these as plain markdown tables — no colored cells, no bolded header rows beyond markdown's default, nothing styled; "Document styling" below covers the same plain-table rule for the eventual PDF/docx export.
- For the three market-analysis sections built on numeric/proportional data — Turnover (2), Trend Analysis (4), and Competitor Analysis (5) — also render one or more charts (bar and/or pie, generated via `matplotlib` per "Charts & visualizations" below) right after that section's table. The table stays as the detailed reference; the chart is for an at-a-glance market read.
- For section 7 (SWOT), use a 2x2 markdown table: Strengths / Weaknesses on top row, Opportunities / Threats on bottom row, 2–4 bullets each.
- Cite where a claim came from inline only when it materially matters (e.g., "per the latest Crunchbase funding round") — don't footnote every sentence.
- Paraphrase abstracts and paper summaries in your own words; never reproduce more than a short, attributed snippet from a patent abstract or paper abstract.

#### Charts & visualizations

A wall of numbers doesn't communicate trend direction or relative scale the way a chart does, so each market-analysis section gets at least one chart in addition to its table. Charts are generated as image files with `matplotlib` rather than any chat-client-specific widget tool, so this works the same way regardless of which runtime or interface is consuming the skill, and it carries over cleanly into PDF/docx exports.

**Pie vs. bar — decision rule.** These two chart types answer different questions, so don't pick one by default and force every dataset into it:
- **Bar chart** for comparing magnitudes across categories (revenue by year, one metric across several competitors) or anything where the values don't sum to a meaningful whole. This is the default when in doubt.
- **Pie chart** only for genuine share-of-a-whole data with **at most 6 segments** — e.g., funding raised per round, search interest split by region, or competitor market share by a metric that sums to a total. If a proportional dataset has more than 6 categories, group the smallest into an "Other" slice rather than rendering a 10-slice pie no one can read; if it can't be reduced sensibly, fall back to a bar chart instead.
- **Never** use a pie chart for time series (interest-over-time, revenue-by-year) — those stay line or bar charts.

**Generation steps:**
1. After each relevant command returns data, write a short standalone `python3` script that builds the chart with `matplotlib` (`Agg` backend, no display needed) and saves it as a PNG — e.g. `charts/turnover_bar.png`, `charts/trend_line.png`, `charts/competitors_pie.png`, next to wherever `report.json` is being written.
2. Embed each PNG directly under that section's table using standard markdown image syntax: `![Revenue by fiscal year](charts/turnover_bar.png)`, followed by a one-line italic caption restating what it shows (e.g. `*Figure: revenue by fiscal year, USD millions*`).
3. **Print/PDF quality, every chart:** `figsize=(6,4)`, `dpi=150` minimum, white `facecolor` (never transparent — looks broken on a printed/PDF page), a title, labeled axes (or percentage labels on pie slices), a legend whenever more than one series/category is shown, and `bbox_inches="tight"` on save so labels aren't clipped. Reuse the same small color palette across every chart in one report so the document reads as one cohesive piece rather than a grab-bag of mismatched default-color plots. Call `plt.close()` after each save.
4. When the rendered markdown is handed to the `pdf`/`docx` skill for export, size each embedded image to roughly 70–90% of the page's text width so it doesn't overflow margins or get auto-shrunk illegibly; defer to that skill's own image-handling guidance over anything here if the two conflict.
5. **Text fallback:** if the runtime turns out to only render plain text/terminal output (no image support), markdown image syntax will just show as a broken link or raw text. The first time this happens in a session, say so once and switch to a one-line trend summary instead for the rest of the report (e.g. "Trend: search interest roughly doubled from 2021 to 2024, peaking in late 2023") rather than continuing to emit images nothing will display.

| Section | Primary chart | Secondary chart (when data supports it) |
|---|---|---|
| 2. Turnover / Financials | Bar chart (public: revenue by year from FMP) or bar chart (private: funding raised by round from Crunchbase) | Pie chart: share of total funding by round type — private companies with 2+ distinct rounds only; skip for public companies or a single round |
| 4. Trend Analysis | Line chart of interest-over-time (always — this is a time series, never pie) | Pie chart: share of search interest by region, only if `trends` returned 2+ regions; group anything past the top 5 into "Other"; if regions can't be reduced to ≤6 sensibly, use a horizontal bar chart instead |
| 5. Competitor Analysis | Bar chart comparing company + competitors on one metric — funding raised or employee count, whichever `competitors` populated for the most entries | Pie chart: each entity's share of the same metric's combined total, only if that metric meaningfully sums to a whole (funding raised, employee count); skip for metrics like founding year that don't sum to anything meaningful |

Keep each chart focused — don't cram turnover, trends, and competitor data into a single combined chart, and don't add both a bar and a pie for the same section unless the table above explicitly calls for both. If a section's underlying command returned too little to chart meaningfully (a single data point, or a 404/empty result from that data source), skip the chart for that section and say so in one line rather than rendering an empty or misleading one.

#### Document styling (PDF / docx export) — keep it plain

This is a research report, not a branded deck, so the exported file should look plain: black text, white background, gray/black structural lines only. The `pdf`/`docx` skill's own examples default to a blue accent (e.g. a `2E75B6`-style border or theme color) for things like table borders and rule lines — when following that skill's instructions, override those defaults rather than accepting them as-is:

- **Table borders:** thin black or gray (`#000000` / `#808080`), never a colored or blue border.
- **Table header row:** bold black text on plain white or very light gray (`#F2F2F2`) shading — no colored fills, no brand-color `ShadingType.SOLID` backgrounds.
- **Headings:** use the document's normal Heading1/Heading2/etc. styles but keep the text black — don't recolor headings to a theme accent.
- **Rules/dividers/callouts:** if a divider is needed, a plain thin gray line; no colored horizontal rules or colored callout/pull-quote boxes.
- **Charts** keep the small color palette from "Charts & visualizations" above — that palette exists to distinguish data series within a chart, not to brand the document — but everything outside the chart images (tables, headings, borders) stays black/white/gray.

This override applies on top of whatever the `pdf` or `docx` skill's `SKILL.md` otherwise specifies for tables and styling — read that skill's section first, then apply the plain palette here instead of its color defaults.

### After the report — present this menu

Always offer these as explicit options once the seven sections are rendered, rather than assuming what the user wants next:

1. **Export as PDF** — hand the rendered markdown to the existing `pdf` (or `docx`) skill to produce the file; this skill does not generate PDFs itself, so read that skill's `SKILL.md` before generating the document. Since charts are saved as PNG files rather than chat-inline widgets, the same markdown-embedded images carry over into the PDF/docx export without extra work.
2. **Drill into a section** — re-run just that section's command with a larger `--limit` or narrower scope and expand it inline; if that section had a chart, redraw it with the expanded data too.
3. **Tech stack** — run the tech-stack step described above (detected + recommended).
4. **Stop here** — nothing further needed.

### Tech-stack output

```markdown
## Technology Stack

### Detected (current)
| Layer | Technology | Source |
|---|---|---|

### Recommended (to build something similar)
| Layer | Suggested technology | Why |
|---|---|---|
```

## Error handling

| Status | Meaning | What to do |
|---|---|---|
| 401 / 403 | Bad or expired key on Crunchbase or SerpApi | Tell the user exactly which env var to regenerate — error messages from the script identify the failing service |
| 404 | Company not found on Crunchbase, or no domain match on BuiltWith | Re-confirm the company/product name with the user; offer close matches if the API returned any |
| 422 / 400 | Malformed query (e.g., empty product string sent to Google Trends) | Re-check that both company and product were actually passed before retrying |
| 429 | Rate limited (SerpApi) | Wait for the reset window the script reports, then retry; the script auto-throttles bulk calls |
| 5xx | Upstream API outage | Tell the user which service is down and continue with whatever sections did succeed rather than failing the whole report |


## Examples

**User:** "Do an R&D report on Notion Labs for their Notion product."

Workflow:
1. Confirm company = "Notion Labs", product = "Notion" (already given). No optional inputs were given, so proceed without asking — they're enrichments, not requirements.
2. `full-report --company "Notion Labs" --product "Notion" --output report.json`
3. Render all seven sections from `report.json`, writing the SWOT yourself from the patterns in the other six, and adding the turnover/trend/competitor charts per "Charts & visualizations"
4. Present the follow-up menu (PDF / drill in / tech stack / stop)

**User:** "Run an R&D report on Stripe (stripe.com) for their Billing product — they're privately held, B2B SaaS, and I already know Chargebee and Recurly compete with them."

Workflow:
1. Company = "Stripe", product = "Billing" — plus domain `stripe.com`, private, B2B SaaS category, and two known competitors, all already given
2. `full-report --company "Stripe" --product "Billing" --domain stripe.com --industry "B2B SaaS" --known "Chargebee,Recurly" --output report.json`, passing `--private` so turnover goes straight to the Crunchbase funding path instead of probing FMP
3. Render the report; the competitor section will already include Chargebee and Recurly enriched with data rather than rediscovering them, its bar chart plots funding raised across all three companies, and — since all three have funding figures that sum to a real total — a secondary pie chart showing each company's share of that combined funding is also appropriate here; the eventual recommended tech stack can lean on "B2B SaaS" instead of asking the category question again

**User:** "What patents does Tesla hold around the Cybertruck, and how's the search trend looking?"

Workflow:
1. Company = "Tesla", product = "Cybertruck"
2. `patents --company "Tesla" --product "Cybertruck"` and `trends --product "Cybertruck"`
3. Answer with just those two sections — no need to run the full seven-section report when the user asked for two specific pieces; render the trend line chart alongside the table since that's one of the standard market-analysis visuals

**User:** "Now show me the tech stack — both what they're actually using and what I'd use to build a competitor."

Workflow (continuing a prior report on Company X / Product Y):
1. `tech-stack-detect --domain <product's domain, asked or inferred>`
2. Reason through a recommended stack using the product category and scale signals already gathered
3. Render both halves of the "Technology Stack" table from "Output format"

## Resources

- `scripts/rnd_report.py` — the CLI helper wrapping Crunchbase, SerpApi, FMP, and BuiltWith. Read this if you need to add a new data source or fix a payload shape.
- `references/api-cheatsheet.md` — endpoint paths, auth headers, and payload shapes for all four services, plus their individual rate limits.
- `references/report-template.md` — the full markdown template (including the tech-stack appendix) with placeholder text, for copy/paste consistency across reports.
