# Autonomous B2B Lead-Generation & Outreach Pipeline
**`n8n` · `Kubernetes` · `PostgreSQL` · `self-hosted meta-search` · `scraping microservice`**
 
> A fully automated prospecting system covering the entire top of the sales funnel — it discovers businesses across several markets, extracts and cleans their contacts, scores and deduplicates them, runs a multi-stage tracked email sequence, hands the hottest leads to a human, and reports the whole funnel weekly — running unattended with zero manual steps.
 
---
 
## The problem
The company connects suppliers in one region with importers/retailers in another, and sales needed a steady flow of qualified businesses to contact. Prospecting was manual: someone searching, copying contact details off websites, guessing which leads were worth pursuing, and emailing them one by one — slow, inconsistent, and impossible to scale across multiple countries and product categories. The system replaces that entirely: a scheduled pipeline that continuously finds candidate businesses, extracts real contact methods, filters out non-companies, scores each lead, sequences outreach, surfaces the hottest leads to a human for the final conversation, and gives management a weekly view of pipeline health.
 
## What I built
Eight coordinated workflows spanning the full top-of-funnel: ingestion (search → scrape → clean → dedup → score), stats logging, a multi-stage outreach sequence, event-driven engagement tracking, weekly deduplication hygiene, sales handover, weekly reporting, and a monthly data-quality audit. Each stage writes to a shared PostgreSQL store and hands off to the next; a human only enters at the very end, for the sales conversation.
 
## Architecture
```
INGESTION (scheduled)
  read query pool (LRU rotation + cooldown, cap per run)
    → idle? → "no eligible queries" alert   [silence can't look like success]
    → self-hosted meta-search (Google/Bing/DuckDuckGo)
    → LAYER-1 junk filter (domain/directory/URL-path/title patterns)
    → scraping microservice: extract emails/phones/socials
    → LAYER-2 junk filter (fetched page <title>) + normalise + clean
    → dedup INSERT into leads_master (guarded: NOT EXISTS on email/whatsapp/site)
    → [DB trigger scores the lead on insert]
 
OUTREACH (every 6h)          ENGAGEMENT (event-driven webhook)
  leads due (score ≥ 50,       provider webhook: opened/click/unsub/complaint
   stage < 4, cadence gates)     → update lead state + log
    → build stage 1–4 email
    → send via gateway → log
 
HYGIENE (weekly)   HANDOFF (weekly)        REPORTING (weekly)  AUDIT (monthly)
  merge dupes,       hot leads → sales        ~7 aggregate       data-quality
  keep highest        sheet + summary          queries → HTML     checks → report
  score                                        report
```
 
Core tables: `leads_master` (contact fields, score 0–100 + reason, status, outreach stage, engagement flags), `lead_outreach_log` (one row per send/event), and `lead_scrape_runs` (per-run telemetry). **Lead scoring is a weighted rule-based model implemented as PL/pgSQL triggers that fire automatically on insert** — no workflow node needed, and every score is explainable.
 
## Engineering decisions worth discussing
- **Two-layer junk filtering — fix data quality at intake, not downstream.** Results are filtered once on the search snippet/URL, then again on the actual page title after scraping, because a page's rendered title often differs from what search returned. Validated by a real incident: during a search-engine hiccup, every query for one country came back as dictionary pages — both layers rejected them and nothing entered the DB. Junk in the leads table inflates counts and corrupts every metric built on top of it.
- **Idempotent, deduplicating inserts + a separate merge job.** The insert is guarded by a NOT EXISTS check on email/WhatsApp/website, so re-scraping the same business across different queries is a no-op, not a duplicate. A weekly merge job consolidates any dupes that slip through, keeping the highest-scored record. This is the difference between a demo scraper and something that runs unattended for months without rotting.
- **`executeOnce` on every aggregation node — learned by fan-out.** In n8n a node runs once per input item by default. An early report listed leads six times because an aggregation node received multiple rows and fanned out. The fix is tiny; the lesson — you have to understand your tool's *execution model*, not just its UI — is the point.
- **LRU query rotation with cooldown + idle alerting.** Queries carry a last-run timestamp; each run picks the least-recently-used eligible ones. An earlier 7-day cooldown once starved the pipeline into five straight days of zero output before it was caught — so the cooldown was shortened and an explicit "no eligible queries" alert added, so silence can never again masquerade as success.
- **SQL correctness under NULL semantics.** A migration left `last_contact_date` NULL on some leads; because `NULL <= <date>` evaluates to NULL (not true), those leads were silently excluded from the "due for outreach" query and stuck at one stage. The practice it drove: cross-check migrations across *all* related columns, not just the obvious one — silent exclusions are the hardest bugs to notice because nothing errors.
## Impact
I won't invent numbers. Honest state at the last snapshot: a small dataset (tens of active leads across three of five target markets), with open/click tracking only just live — so there isn't yet a meaningful reply-rate or conversion figure. What's worth pulling (all already in the tables): throughput (leads/run and /week), extraction hit rate, junk-rejection rate (a strong data-quality stat), full-funnel conversion (scraped → qualified → contacted → opened → replied), dedup effectiveness, and a clearly-labelled time-saved estimate (hours/week of manual prospecting replaced). For the portfolio, one populated weekly-report screenshot plus a throughput + funnel + time-saved trio reads far better than any invented percentage.
 
## Tech stack
`n8n (self-hosted, Kubernetes)` · `PostgreSQL (PL/pgSQL triggers + analytical SQL)` · `self-hosted meta-search` · `internal contact-extraction microservice (Cheerio/fetch)` · `transactional email provider + engagement webhooks` · `Google Sheets (OAuth2)` · `SQL` · `JavaScript`
 
## Retrospective — what I'd do differently
- **Replace regex junk-filtering and name-cleaning with an LLM classifier.** "Is this a real business that imports/sells goods, and what's its clean name?" is exactly the fuzzy judgement rules keep *almost* getting right. This is the honest bridge to the AI Automation Engineer role — it turns a rule-based pipeline into a genuinely AI-driven one, and it's the single highest-leverage change.
- **Parameterise the SQL.** Queries were built by interpolating scraped values (attacker-influenced free text) into Code nodes with hand-rolled escaping. It works, but parameterised queries / a thin data layer would remove a whole class of injection risk.
- **Version-control the workflows in Git** instead of hand-incrementing JSON filenames. Export-on-commit makes the iteration history an asset, not a folder of near-duplicates.
- **Instrument the filter layers.** The two-stage junk filtering works, but I'd add explicit counters on how many candidates each layer rejects vs. accepts — turning data-quality into a live metric instead of something inferred after the fact.
- **Real observability + a job queue** in place of cron-plus-email-alerts: structured logs, run dashboards, retries with backoff as first-class concerns.
- **Handle bot-protected sites properly** (headless browser / scraping framework) — the pipeline currently detects and skips challenge pages rather than getting past them, which is safe but leaves reachable leads on the table.

**What I learned:** n8n's per-item execution model and where it bites; PostgreSQL NULL comparison semantics the hard way; that data quality must be enforced at intake; that idempotency and guarded inserts are what let automation run unattended — and that most of these lessons arrived as incidents, not textbook reading, which is itself the story worth telling.
