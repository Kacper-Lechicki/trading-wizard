# CLAUDE.md

Instructions for AI agents (Claude Code, Cursor, etc.) working on this repository.

---

## Core Principles

Every change you make must satisfy these four pillars — in this priority order:

1. **Security** — never expose secrets, never trust external input blindly, never weaken existing safeguards.
2. **Reliability** — the system must survive API failures, malformed data, and edge cases gracefully. Every external call can fail; plan for it.
3. **Minimalism** — solve the problem with the least code. Don't add features, abstractions, or "improvements" that weren't requested. Three similar lines are better than a premature helper function.
4. **Optimization** — respect FMP's 250 req/day limit. Minimize OpenRouter token usage. Batch where possible. Don't over-fetch.

If a change doesn't clearly serve one of these — don't make it.

---

## Project Overview

Trading Wizard is a personal intraday trading assistant built as n8n workflows. It scans US and European stock markets, generates AI-powered trade recommendations with exact entry/exit prices, tracks results, and learns from the user's trading history over time.

Self-hosted via Docker (n8n + Postgres). Only external cost: OpenRouter API tokens (~$0.15-0.50/month).

## Architecture

```
n8n (Docker) — 7 workflow files, 5 AI agent tools
├── 07-chat-main.json      Chat (main interface — AI Agent with 5 tools)
├── 05-market-scanner.json  Market Scanner (FMP + news + AI analysis)
├── 04-trade-logger.json    Trade Logger (P&L calculation + Postgres)
├── 06-weekly-review.json   Weekly Review (stats + AI review + insights)
├── 02-get-history.json     Get History (Postgres query)
├── 03-get-insights.json    Get Insights (Postgres query + review check)
├── 01-db-init.json         DB Init (schema creation — run once)
│
├── Postgres (Docker) — 3 tables: signals, trades, insights
│
└── External APIs:
    ├── FMP (screener, quotes, news, earnings) — 250 req/day free
    ├── Google News RSS — unlimited, no key
    ├── Yahoo Finance (fallback quotes) — unlimited, unofficial
    └── OpenRouter / Gemini Flash (AI analysis) — pay-per-use
```

## Key Design Decisions

1. **No cron jobs.** Self-hosted on user's PC — machine is not always on. Weekly review is triggered on-demand (AI Agent checks at conversation start).
2. **English internals, multilingual responses.** All system prompts, tool descriptions, AI analysis prompts in English. The AI Agent responds in user's language (default: Polish).
3. **Risk transparency.** Every BUY signal must include: estimated cost, potential profit/loss ($ and %), risk/reward ratio, stop-loss, risk warnings, and a plain-language risk summary. The user is learning — never hide downside.
4. **Multi-source data.** FMP as primary, Google RSS for news diversity. Never depend on a single source.
5. **Learning loop.** Insights from weekly reviews are injected into future scan prompts. The system personalizes over time.
6. **Resilient pipeline.** Every HTTP Request node continues on failure. Every merge node uses try/catch. The scanner always reaches AI Analysis with whatever data it could gather.

---

## Repository Structure

```
trading-wizard/
├── docker-compose.yml          # n8n + Postgres orchestration
├── .env.example                # Environment template (copy to .env)
├── .gitignore                  # Excludes .env, volumes, IDE files
├── README.md                   # User-facing setup and usage guide
├── CLAUDE.md                   # This file — AI agent instructions
├── workflows/
│   ├── 01-db-init.json         # Schema creation (run once)
│   ├── 02-get-history.json     # Trade history query
│   ├── 03-get-insights.json    # Insights + review-due check
│   ├── 04-trade-logger.json    # P&L calc + save trade
│   ├── 05-market-scanner.json  # Full scan pipeline (19 nodes)
│   ├── 06-weekly-review.json   # Performance review + insight generation
│   └── 07-chat-main.json       # Main AI Agent interface
└── docs/
    └── BLUEPRINT.md            # Full technical blueprint — every node described
```

---

## Conventions

### Secrets & Security

- **NEVER** hardcode API keys, passwords, or tokens in any file.
- All secrets live in `.env` (gitignored). `docker-compose.yml` reads via `${VAR}`.
- Workflow JSON files use placeholder strings (`YOUR_FMP_KEY`, `YOUR_OPENROUTER_KEY`) — users replace after import into n8n, or configure n8n credentials.
- Before committing, always verify: no real keys in tracked files. Run `grep -r "sk-or-v1-[^y]" .` or similar to catch leaked keys.
- Credential nodes in workflows use `"id": ""` — n8n fills these on import.

### Language

- All code, comments, node names, tool descriptions, AI prompts: **English**.
- User-facing text (reasoning, educational notes, summaries): generated in user's language by the AI.

### Node Naming

- Descriptive English names. Examples: `FMP Gainers`, `Calculate P&L`, `Build AI Prompt`, `Validate & Enrich`.

### Error Handling

- Every HTTP Request node: `timeout: 10000` (10s minimum), `onError: "continueRegularOutput"`.
- AI Analysis node: `timeout: 120000` (2 min) — LLM calls are slow.
- Every AI JSON response must be parsed in a try/catch. Invalid JSON → user-friendly error message, never raw stack trace.
- Merge nodes must use try/catch on each data source independently — one source failing must not crash the pipeline.

### Risk Disclosure

Every BUY signal must include ALL of these fields — enforced at three levels (prompt, validation code, chat presentation):

| Field | Description |
|---|---|
| `estimated_cost` | `suggested_shares * midpoint(entry_low, entry_high)` |
| `upside_pct` | % gain if take-profit hit |
| `downside_pct` | % loss if stop-loss hit (negative number) |
| `risk_reward` | `abs(upside_pct / downside_pct)` — must be >= 1.5 |
| `potential_profit` | $ gain if take-profit hit |
| `potential_loss` | $ loss if stop-loss hit (negative number) |
| `stop_loss` | Price level |
| `risk_level` | `low` / `moderate` / `high` |
| `risk_warnings` | Array of warnings (earnings proximity, low volume, etc.) |
| `reasoning` | Why this signal — in user's language |
| `educational_note` | Short lesson — in user's language |

**Never remove `educational_note` or `reasoning` from signals.** The user is learning.

---

## Database Schema (Postgres)

3 tables — full schema in `docs/BLUEPRINT.md` → Database section:

- **`signals`** — AI-generated trading signals (BUY/SKIP) with price data, indicators, news, reasoning.
- **`trades`** — User's actual results with P&L, outcome, feedback tags. Linked to signals via `signal_id`.
- **`insights`** — Weekly review conclusions injected into future scans.

Schema is created by `01-db-init.json`. Includes `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` for safe upgrades.

---

## Workflow Details

Full node-by-node descriptions: `docs/BLUEPRINT.md`.

Quick reference — what calls what:

```
User message → 07-chat-main.json (AI Agent decides tool)
  ├── scan_market  → 05-market-scanner.json
  ├── log_trade    → 04-trade-logger.json
  ├── run_review   → 06-weekly-review.json
  ├── get_history  → 02-get-history.json
  └── get_insights → 03-get-insights.json
```

---

## Modification Rules

### Before You Change Anything

1. **Read the relevant workflow JSON** and its BLUEPRINT.md section first. Understand the full pipeline before touching a node.
2. **Read this entire CLAUDE.md.** These are your constraints.
3. **Identify blast radius.** Which other workflows or nodes depend on what you're changing? Trace the data flow.

### Security Checklist (every change)

- [ ] No hardcoded secrets in any tracked file
- [ ] No new `console.log` / debug output that could leak data
- [ ] SQL queries use parameterized values or validated inputs (watch for SQL injection via n8n expressions like `'{{ $json.ticker }}'` — ensure upstream code sanitizes)
- [ ] New HTTP endpoints have timeout and error handling
- [ ] No sensitive data in URLs (API keys should be in headers, not query params — note: FMP uses query params by design, that's their API)

### Reliability Checklist (every change)

- [ ] External API calls have `timeout` set and `onError: "continueRegularOutput"`
- [ ] AI response parsing wrapped in try/catch with user-friendly fallback
- [ ] Merge/combine nodes handle missing upstream data gracefully
- [ ] No assumption that optional fields exist — use `|| default` or `?? fallback`
- [ ] New logic works when database is empty (fresh install)

### Optimization Checklist (every change)

- [ ] Not adding unnecessary API calls (respect FMP 250/day limit)
- [ ] Not increasing AI prompt token count without justification
- [ ] Not fetching data that's already available upstream in the pipeline
- [ ] Batch API calls where possible (e.g., `FMP Batch Quote` for multiple tickers)

### Documentation Sync (mandatory)

After every modification, check if you need to update:

| What you changed | Update required |
|---|---|
| Tool description in chat workflow | `docs/BLUEPRINT.md` → Chat section |
| Scanner pipeline (add/remove/modify nodes) | `docs/BLUEPRINT.md` → Market Scanner section |
| Trade logger logic | `docs/BLUEPRINT.md` → Trade Logger section |
| Weekly review logic | `docs/BLUEPRINT.md` → Weekly Review section |
| Database schema (new column, new table) | `docs/BLUEPRINT.md` → Database section AND `01-db-init.json` |
| New API source | `docs/BLUEPRINT.md` → External APIs, `.env.example`, this file's Architecture section |
| New workflow file | This file's Repository Structure and Architecture sections |
| Risk disclosure fields | This file's Risk Disclosure table |
| docker-compose.yml | `README.md` setup instructions |

**If you skip a documentation update, the next agent will work with stale information and make wrong decisions. This is a hard requirement.**

### Verification (mandatory)

Before declaring any change complete:

1. **Validate JSON.** Every workflow JSON must be valid. Parse it (`JSON.parse`) or use a linter.
2. **Trace the data flow.** Walk through the pipeline mentally: does data arrive at each node in the expected shape? What if an upstream node failed?
3. **Check edge cases:**
   - What happens with 0 candidates after filtering?
   - What happens when AI returns malformed JSON?
   - What happens on fresh install (empty DB)?
   - What happens when all API calls fail?
4. **Verify no regressions.** If you changed a node, confirm that downstream nodes still receive the data shape they expect.
5. **Diff review.** Before committing, review every changed line. Ask: "Does this introduce a security risk? Does this break the pipeline if something upstream fails?"

---

## What NOT to Do

- **Don't add cron jobs.** The machine isn't always on. On-demand only.
- **Don't add new npm dependencies.** This is pure n8n + Postgres. No custom code outside workflow JSON.
- **Don't refactor working code** that you weren't asked to change. Functional code is correct code.
- **Don't remove error handling** to make code "cleaner." Every try/catch is there for a reason.
- **Don't add Finnhub integration** without explicit request — it was designed as optional and is not currently wired in.
- **Don't change the AI model** (Gemini Flash via OpenRouter) without explicit request — it's chosen for cost/speed balance.
- **Don't weaken validation rules** (60% confidence minimum, 1.5 R/R minimum) — these protect the user's money.
- **Don't remove risk fields** from signals or chat presentation — the user must see the full picture.
- **Don't commit `.env`** — ever. Only `.env.example` with placeholder values.

---

## Placeholder Keys in Workflow Files

The following placeholder strings appear in workflow JSONs and are **intentional** — they are NOT real keys:

- `YOUR_FMP_KEY` — in FMP API URLs (query parameter `apikey=`)
- `YOUR_OPENROUTER_KEY` — in OpenRouter Authorization headers

Users replace these after importing workflows into n8n. Do not replace them with `${ENV_VAR}` syntax — n8n HTTP Request nodes don't support that. Users either:
1. Replace the placeholder strings manually in n8n UI, or
2. Use n8n's built-in credential system (preferred).

---

## Quick Reference: File → Purpose

| File | Nodes | Key logic |
|---|---|---|
| `01-db-init.json` | 2 | CREATE TABLE + indexes + safe ALTER for upgrades |
| `02-get-history.json` | 3 | JOIN trades+signals, compute stats, format |
| `03-get-insights.json` | 3 | Load latest insights, check if review overdue |
| `04-trade-logger.json` | 5 | Calculate P&L, find matching signal, save, confirm |
| `05-market-scanner.json` | 19 | Full pipeline: screen → filter → enrich → news → AI → validate → format |
| `06-weekly-review.json` | 9 | Count check → load trades → stats → AI review → save insights → format |
| `07-chat-main.json` | 9 | Chat trigger → AI Agent (5 tools) + LLM + Memory |
