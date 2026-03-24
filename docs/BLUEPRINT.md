# Trading Wizard - Blueprint

How the system works, workflow by workflow, node by node.

---

## System Overview

Trading Wizard is a chat-based trading assistant built as a set of connected n8n workflows. You write messages in a chat window, and an AI agent figures out what you need and calls the right workflow.

```
You (chat) --> AI Agent --> scan_market    --> scans stocks, returns recommendations
                       --> log_trade      --> records your trade results
                       --> run_review     --> analyzes your performance over time
                       --> get_history    --> shows past trades
                       --> get_insights   --> loads lessons learned
                       --> web_search     --> real-time web search via SearXNG
```

The AI agent speaks your language (default: Polish), but all internal logic runs in English because AI models handle English instructions more reliably.

---

## Data Flow

```
FMP API (free)        Google News RSS (free)     Finnhub (free)
  |                        |                        |
  |-- Top gainers         |-- Headlines            |-- Company news
  |-- Top losers          |                        |
  |-- Most active         |                        |
  |-- Company news        |                        |
  |-- Earnings calendar   |                        |
  |                        |                        |
  v                        v                        v
[Market Scanner Workflow]
  |
  |-- Filters out junk (penny stocks, OTC, too expensive)
  |-- Merges news from multiple sources
  |-- Sends everything to AI (OpenRouter / Gemini Flash)
  |-- AI returns structured signals: BUY or SKIP
  |-- Each BUY signal includes: entry, stop-loss, take-profit,
  |   cost, potential profit/loss, risk level, explanation
  |
  v
[Chat] --> AI Agent presents results to you in your language
  |
  v
You trade manually in XTB
  |
  v
[Trade Logger Workflow] --> saves results to Postgres
  |
  v
[Weekly Review Workflow] --> AI analyzes your history, generates insights
  |                          that improve future scans
  v
[Postgres] -- signals, trades, insights tables
```

---

## Workflows

### 1. Chat (main interface)

**File:** `07-chat-main.json`

This is the only workflow you interact with directly. Everything else is called automatically.

**How it works:**

You type a message. The AI Agent reads it, decides what you want, and calls the appropriate tool. The AI remembers the last 20 messages in your conversation (within one session).

**Nodes:**

| Node                           | What it does                                                                                                                                                                                                                                                                                                                                                                            |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **When chat message received** | Listens for your messages in the n8n chat window. Every message you type enters here.                                                                                                                                                                                                                                                                                                   |
| **Detect Language**            | Code node that runs before the AI Agent. Detects whether the user's message is in Polish or English using word/character heuristics (Polish diacritics + common Polish function words). Prepends `[USER_LANGUAGE: ENGLISH]` or `[USER_LANGUAGE: POLISH]` to the message so the AI Agent knows which language to respond in - without guessing.                                          |
| **AI Agent**                   | The brain. Reads the language-tagged message, decides which tool to call, and formulates the response in the detected language. Uses the system prompt that defines its personality, rules, and risk disclosure requirements.                                                                                                                                                           |
| **OpenRouter Chat Model**      | The LLM connection. Sends messages to Google Gemini Flash via OpenRouter. Temperature 0.3 (low = predictable, concrete answers).                                                                                                                                                                                                                                                        |
| **Memory**                     | Remembers the last 20 messages so the AI knows what you said earlier in this conversation. Resets between sessions.                                                                                                                                                                                                                                                                     |
| **scan_market** (tool)         | Calls the Market Scanner workflow. Supports intraday and swing trade detection. Accepts budget, currency, market, and fractional_shares (boolean) parameters. Triggered when you mention a budget and want to scan.                                                                                                                                                                     |
| **log_trade** (tool)           | Calls the Trade Logger workflow. Triggered when you report a trade result.                                                                                                                                                                                                                                                                                                              |
| **run_review** (tool)          | Calls the Weekly Review workflow. Triggered when you ask for a performance review.                                                                                                                                                                                                                                                                                                      |
| **get_history** (tool)         | Calls the Get History sub-workflow. Triggered when you ask about past trades or stats.                                                                                                                                                                                                                                                                                                  |
| **get_insights** (tool)        | Calls the Get Insights sub-workflow. Called silently at the start of every conversation to check if a review is due.                                                                                                                                                                                                                                                                    |
| **web_search** (tool)          | SearXNG meta-search engine. The AI MUST use this as its primary source of truth for anything time-sensitive: current prices, latest news, earnings dates, market conditions. It searches BEFORE answering market questions and enriches scan results with real-time data. Self-hosted, free, no API limits. Requires SearXNG credential configured in n8n (URL: `http://searxng:8080`). |

**Key behaviors:**

- At the start of every conversation, the AI silently checks if a weekly review is due (via get_insights). If yes, it suggests running one.
- Every BUY signal must include full risk disclosure: potential profit, potential loss, stop-loss explanation, risk level.
- Supports swing trades (multi-day holds) alongside intraday trades, with appropriate holding period guidance.
- Small budgets are supported via fractional shares - the price ceiling filter is disabled when fractional_shares is true.
- **Language detection**: A dedicated Code node (`Detect Language`) analyzes the user's message before the AI Agent sees it. It checks for Polish diacritics (ą, ć, ę, etc.) and common Polish words to determine the language, then tags the message with `[USER_LANGUAGE: ENGLISH]` or `[USER_LANGUAGE: POLISH]`. The AI Agent is instructed to respond in the tagged language - no guessing required.
- Feedback tags (like "exited too early") are translated to the user's language when presented.
- **Mandatory web search workflows**: The AI calls web_search automatically in specific situations: (A) after scan_market returns - searches top 2-3 tickers for latest news, (B) when user asks about a stock - searches before answering, (C) when user asks about market conditions - searches before answering, (D) when user asks to research/dig deeper - searches immediately, (E) before stating any current fact - verifies via search. Training data is treated as outdated for all market information.

---

### 2. Market Scanner

**File:** `05-market-scanner.json`

The core engine. Scans the market, gathers data, and asks AI to analyze it.

**Pipeline (what happens in order):**

```
[Trigger] receives budget, currency, market preference
    |
    v
[FMP Gainers + Most Active + Losers] -- 3 parallel API calls
    |                                    fetch ~150 stocks total
    v
[Merge & Deduplicate] -- combine all 3 lists, remove duplicates
    |                     typically ~80-100 unique stocks
    v
[Filter Candidates] -- remove junk, keep top 10
    |
    v
[Build Symbol List] -- prepare ticker list for next steps
    |
    +--> [FMP Batch Quote] -- try to get detailed quotes (may fail on free tier)
    +--> [FMP Earnings Check] -- check if any stock reports earnings in 48h
    |
    v
[Merge Quote Data] -- combine screener data + quotes + earnings warnings
    |                  resilient: works even if quote/earnings nodes failed
    v
[Select Top 10 for News] -- pick top candidates for news lookup
    |
    +--> [FMP News] -- company-specific news from FMP
    +--> [Google News RSS] -- general stock news from Google
    +--> [Finnhub News] -- supplementary company news (first ticker)
    |
    v
[Merge News] -- combine news from all three sources, deduplicate, limit 5 per ticker
    |            resilient: works even if one source failed
    v
[Load Insights] -- load lessons from previous weekly reviews (from Postgres)
    |               empty on first run, that's fine
    v
[Build AI Prompt] -- assemble everything into one prompt:
    |                 - candidate stocks with prices and changes
    |                 - news per ticker
    |                 - earnings warnings
    |                 - lessons from past performance
    |                 - rules for signal quality, risk disclosure, position sizing
    v
[AI Analysis] -- send prompt to Gemini Flash via OpenRouter
    |             model returns structured JSON with BUY/SKIP signals
    |             timeout: 2 minutes, max 8192 tokens
    v
[Validate & Enrich] -- parse AI response, enforce rules:
    |                   - confidence < 60%? downgrade to SKIP
    |                   - risk/reward < 1.5? downgrade to SKIP
    |                   - missing risk fields? calculate them
    |                   - cap at 5 BUY signals max
    v
[Format Response] -- format signals as readable text for the AI Agent
    |                 includes: ticker, entry, stop-loss, take-profit,
    |                 cost, profit/loss, risk level, warnings, reasoning, lesson
    v
[Return Response] -- guarantees this is the last node n8n sees
                     reads from Format Response, returns its data to the AI Agent
```

**Nodes in detail:**

| Node                       | What it does                                                                                                                                                                                                                                                                                                                                                                                    |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Trigger**                | Receives parameters from the Chat AI Agent: budget (number), currency (USD/EUR/PLN), market (US/EU/BOTH), fractional_shares (boolean, default true).                                                                                                                                                                                                                                            |
| **FMP Gainers**            | Fetches today's top gaining stocks from FMP (`/stable/biggest-gainers`). Returns ~50 stocks with symbol, price, change%. Continues on failure.                                                                                                                                                                                                                                                  |
| **FMP Most Active**        | Fetches today's most traded stocks (`/stable/most-actives`). High volume = high liquidity = easier to trade. Continues on failure.                                                                                                                                                                                                                                                              |
| **FMP Losers**             | Fetches today's biggest losers (`/stable/biggest-losers`). Oversold stocks sometimes bounce - potential opportunities. Continues on failure.                                                                                                                                                                                                                                                    |
| **Merge & Deduplicate**    | Combines all three lists into one and removes duplicate tickers. A stock that appears in both gainers and most-active is kept once.                                                                                                                                                                                                                                                             |
| **Filter Candidates**      | Removes stocks that are unsuitable: penny stocks (< $1), OTC-listed stocks. Price ceiling (> 30% of budget) is disabled when fractional shares are enabled. Sorts by absolute price change and keeps top 10.                                                                                                                                                                                    |
| **Build Symbol List**      | Prepares a comma-separated list of ticker symbols for the next API calls.                                                                                                                                                                                                                                                                                                                       |
| **FMP Batch Quote**        | Tries to fetch detailed quotes (volume, day high/low, market cap) for all candidates in one call. May fail on free tier - that's OK, pipeline continues with screener data.                                                                                                                                                                                                                     |
| **FMP Earnings Check**     | Checks if any candidate reports earnings in the next 48 hours. Earnings are binary events (stock can jump 10% either way) - AI should warn about this. May fail on free tier.                                                                                                                                                                                                                   |
| **Merge Quote Data**       | Combines screener data with quote data and earnings flags. If quote or earnings nodes failed, it uses screener data only. Resilient - never crashes the pipeline.                                                                                                                                                                                                                               |
| **Select Top 10 for News** | Picks the top 10 candidates for news lookup. Limits API usage.                                                                                                                                                                                                                                                                                                                                  |
| **FMP News**               | Fetches company-specific news for selected tickers (`/stable/news/stock`). Returns headlines, summaries, sources. Continues on failure.                                                                                                                                                                                                                                                         |
| **Google News RSS**        | Fetches general stock news from Google News RSS feed. Free, no API key, no limit. Catches news that financial APIs might miss. Continues on failure.                                                                                                                                                                                                                                            |
| **Finnhub News**           | Fetches company news for the first candidate ticker from Finnhub (3-day lookback). Supplements FMP and Google news. Free, 60 req/min. Continues on failure.                                                                                                                                                                                                                                     |
| **Merge News**             | Combines news from all three sources (FMP, Google, Finnhub), groups by ticker, removes duplicates, limits to 5 per ticker. Resilient - uses try/catch on each source.                                                                                                                                                                                                                           |
| **Load Insights**          | Queries Postgres for the latest weekly review insights. These are lessons learned from your past trades (e.g., "signals with high volume have 78% win rate"). Empty on first run. Set to always output data so the pipeline doesn't stop.                                                                                                                                                       |
| **Build AI Prompt**        | The most important node. Assembles the complete prompt for AI analysis: all candidate data, news, earnings warnings, past insights, and detailed rules for signal quality and risk disclosure. Classifies trade type (intraday/swing) based on price patterns. Handles small budget scenarios with fractional share guidance. Instructs AI to write reasoning in user's language.               |
| **AI Analysis**            | Sends the assembled prompt to Gemini Flash via OpenRouter. Requests JSON output. 8192 token limit, 2-minute timeout. Cost: ~$0.01 per scan.                                                                                                                                                                                                                                                     |
| **Validate & Enrich**      | Parses the AI JSON response. Enforces hard rules that the AI might miss: minimum 60% confidence, minimum 1.5 risk/reward ratio. Calculates missing fields (shares, cost, profit/loss). Defaults trade_type to "intraday" and suggested_holding to "same_day" if missing. Supports fractional shares (DECIMAL precision). Caps BUY signals at 5. Downgrades violations to SKIP with explanation. |
| **Format Response**        | Converts validated signals into readable text that the Chat AI Agent will present to you. Includes all risk disclosure fields and trade type label (INTRADAY/SWING).                                                                                                                                                                                                                            |
| **Return Response**        | Final node in the chain. Reads from Format Response and guarantees it is always the `lastNodeExecuted` in n8n. This is required because parallel branches (FMP calls, news calls) can complete after Format Response, and n8n uses chronological order to determine output. Without this node, the toolWorkflow would return the wrong node's data.                                             |

**Resilience design:**

The scanner is built to survive failures at every level:

- **FMP completely unavailable (API limit reached):** Pipeline continues in "news only" mode. Select Top 10 uses fallback tickers (SPY, QQQ, AAPL, MSFT, TSLA) for news lookup. Build AI Prompt tells the AI to base analysis on news data and its own knowledge. AI can still produce recommendations.
- **Individual API failures:** FMP free tier doesn't support all endpoints - Batch Quote and Earnings Check may return 403. Every external API node has "continue on failure" enabled, and every merge node uses try/catch.
- **Pipeline always reaches AI Analysis** with whatever data it could gather - from full FMP data + 3 news sources down to news-only mode.

---

### 3. Trade Logger

**File:** `04-trade-logger.json`

Records the results of your trades. Called when you tell the AI something like "I bought AAPL at 185, sold at 187, 5 shares."

**Pipeline:**

```
[Trigger] receives trade details from AI Agent
    |
    v
[Calculate P&L] -- computes profit/loss in dollars and percentage
    |               auto-determines outcome: WIN/LOSS/BREAKEVEN
    v
[Find Matching Signal] -- searches Postgres for the AI signal
    |                      that recommended this ticker today
    |                      links your trade to the original recommendation
    v
[Save Trade] -- inserts trade record into Postgres
    |            includes: prices, P&L, outcome, feedback tags, notes
    v
[Format Confirmation] -- returns confirmation text to AI Agent
                         "Trade logged! AAPL: +$9.00 (+1.08%), WIN"
```

**Nodes in detail:**

| Node                     | What it does                                                                                                                                                                                                                                |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Trigger**              | Receives: ticker, buy_price, sell_price, quantity, hit_tp, hit_sl, exited_early, feedback_tags, notes.                                                                                                                                      |
| **Calculate P&L**        | Computes `pnl_amount = (sell - buy) * quantity` and `pnl_pct = ((sell - buy) / buy) * 100`. Auto-classifies outcome if not provided.                                                                                                        |
| **Find Matching Signal** | Queries Postgres: "find the most recent BUY signal for this ticker from today." Links the trade to its originating AI signal for later analysis. Returns null if no signal found (e.g., you traded something the scanner didn't recommend). |
| **Save Trade**           | Inserts a complete trade record into the `trades` table.                                                                                                                                                                                    |
| **Format Confirmation**  | Builds a human-readable confirmation with P&L summary.                                                                                                                                                                                      |

**Feedback tags you can use:**

| Tag                       | Meaning                            |
| ------------------------- | ---------------------------------- |
| `perfect_execution`       | Went exactly as planned            |
| `exited_too_early`        | Sold too soon, price kept rising   |
| `exited_too_late`         | Held too long, lost gains          |
| `wrong_entry_price`       | Entered at a worse price           |
| `news_changed_situation`  | Unexpected news changed everything |
| `should_have_skipped`     | Should not have entered            |
| `stop_loss_saved_me`      | Stop-loss prevented bigger loss    |
| `should_have_held_longer` | Exited but price kept going up     |

---

### 4. Weekly Review

**File:** `06-weekly-review.json`

Analyzes your trading history and generates insights that improve future scans. Not on a cron - triggered on-demand by the AI Agent when it detects a review is due, or when you ask for one.

**Pipeline:**

```
[Trigger] called by AI Agent
    |
    v
[Check Trade Count] -- how many trades in last 30 days?
    |
    v
[Enough Trades?] -- need at least 5 for meaningful analysis
    |         |
    | YES     | NO --> "Not enough trades yet, keep trading"
    v
[Load Trades + Signals] -- JOIN trades with their AI signals
    |                       last 30 days, up to 30 trades
    v
[Calculate Stats] -- compute:
    |                 - win rate, avg gain, avg loss, total P&L
    |                 - win rate per confidence range (60-69%, 70-79%, 80%+)
    |                 - feedback tag frequency
    |                 - per-ticker performance
    v
[AI Review] -- send stats + trades to Gemini Flash
    |           "What distinguishes wins from losses for this trader?"
    |           model returns: insights (English), adjustments, summary (user's language)
    v
[Validate Review] -- parse JSON, extract insights
    |
    v
[Save Insights] -- store in Postgres `insights` table
    |               these insights will be loaded by the scanner
    |               in every future scan
    v
[Format Response] -- return review summary to AI Agent
```

**Nodes in detail:**

| Node                      | What it does                                                                                                                                                                                         |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Trigger**               | Receives the call from AI Agent. No parameters needed.                                                                                                                                               |
| **Check Trade Count**     | Counts trades in last 30 days.                                                                                                                                                                       |
| **Enough Trades?**        | If < 5 trades, stops and returns a friendly message. You need enough data for statistics to mean anything.                                                                                           |
| **Load Trades + Signals** | JOINs the `trades` and `signals` tables so the AI can see both what it recommended and what actually happened.                                                                                       |
| **Calculate Stats**       | Pure math: win rate, average win/loss percentages, total P&L, breakdown by confidence range, breakdown by ticker, feedback tag frequency.                                                            |
| **AI Review**             | Sends all stats and trade data to Gemini Flash with instructions to find patterns. Insights are written in English (they'll be injected into future prompts). Summary is written in user's language. |
| **Validate Review**       | Parses JSON response, extracts insights array and adjustments.                                                                                                                                       |
| **Save Insights**         | Stores everything in Postgres. The Market Scanner's "Load Insights" node will read this on every future scan.                                                                                        |
| **Format Response**       | Returns a readable summary: win rate, key insights, recommendations.                                                                                                                                 |

**Example insights the system might generate after a few weeks:**

- "Signals with volume ratio above 1.5x have 78% win rate vs 51% for lower volume"
- "TSLA: 2 wins out of 6 trades - consider being more cautious"
- "When you tag 'exited_too_early', the average remaining upside was +1.2%"
- "Your best results come from stocks with confidence > 75%"

---

### 5. Get History (helper)

**File:** `02-get-history.json`

Simple query workflow. Returns your trade history when you ask "show me my trades" or "what's my win rate?"

| Node                | What it does                                                                                                    |
| ------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Trigger**         | Receives the call.                                                                                              |
| **Load Trades**     | Queries last 30 days of trades JOINed with signals.                                                             |
| **Format Response** | Computes summary stats (total trades, wins, losses, win rate, total P&L) and formats each trade as a one-liner. |

---

### 6. Get Insights (helper)

**File:** `03-get-insights.json`

Dual-purpose workflow: returns latest insights AND checks if a weekly review is due.

| Node                     | What it does                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| **Trigger**              | Receives the call.                                                                                                       |
| **Load Latest Insights** | Queries the most recent insights record, plus checks: has it been >7 days since last review? How many trades since then? |
| **Check Review Status**  | Computes `needs_review = (overdue AND enough_trades)`. Returns insights + the flag.                                      |

The AI Agent calls this silently at the start of every conversation. If `needs_review` is true, it suggests: "It's been a while since your last review. Want me to analyze your recent performance?"

---

## Database

Three tables in Postgres:

### signals

Every stock the AI analyzed, whether BUY or SKIP. One row per ticker per scan session.

Key fields: ticker, type (BUY/SKIP), confidence, entry range, stop-loss, take-profit, suggested shares (DECIMAL(10,4) for fractional share support), estimated cost, potential profit/loss, risk/reward, risk level, risk warnings, reasoning, educational note, catalysts, price data snapshot, news data snapshot, trade_type (intraday/swing), suggested_holding (same_day/2-3 days/3-5 days).

### trades

Your actual trading results. Linked to signals via `signal_id`.

Key fields: ticker, buy price, sell price, quantity (DECIMAL(10,4) for fractional shares), P&L (dollars and %), outcome (WIN/LOSS/BREAKEVEN), feedback tags, notes.

### insights

Weekly review conclusions. Injected into future scan prompts.

Key fields: week start date, trades analyzed, win rate, insights (JSON array of English statements), adjustments (suggested parameter changes), summary (in user's language).

---

## External APIs

| Service                           | What it provides                                                | Limit            | Cost        |
| --------------------------------- | --------------------------------------------------------------- | ---------------- | ----------- |
| **FMP** (Financial Modeling Prep) | Stock screener (gainers/losers/active), news, earnings calendar | 250 requests/day | Free        |
| **Google News RSS**               | General stock news headlines                                    | Unlimited        | Free        |
| **Finnhub**                       | Company news (supplementary, first candidate ticker)            | 60 requests/min  | Free        |
| **OpenRouter** (Gemini Flash)     | AI analysis of stocks, weekly review                            | Pay per use      | ~$0.01/scan |

**Monthly cost estimate: $0.15-0.50** (22 trading days).

---

## Risk Disclosure

Risk transparency is enforced at three levels:

**Level 1 - AI Prompt.** The scanner prompt explicitly requires every BUY signal to include: estimated cost, potential profit ($ and %), potential loss ($ and %), risk/reward ratio, risk level (low/moderate/high), and risk warnings.

**Level 2 - Validation code.** The Validate & Enrich node checks that all risk fields exist. If the AI omitted any, they are calculated mathematically. Signals with confidence < 60% or risk/reward < 1.5 are automatically downgraded to SKIP.

**Level 3 - Chat presentation.** The AI Agent's system prompt mandates that every BUY signal presented to the user includes a plain-language risk summary: "You could lose up to $X if stop-loss triggers, or gain up to $Y if target hits."

---

## Learning Loop

The system gets smarter over time:

1. You scan the market → AI generates signals
2. You trade based on signals → report results via chat
3. Results are stored in Postgres with feedback tags
4. Weekly review analyzes your history → generates insights
5. Insights are injected into the next scan's AI prompt
6. AI adjusts its analysis based on what worked for YOU specifically

This loop is what makes the system personalized. After 20-30 trades, the AI knows which tickers work for you, which confidence ranges to trust, and what patterns to watch for.

---

## Self-Hosted Design

The system runs on your computer via Docker. Key implications:

- **No cron jobs.** Your PC isn't always on, so scheduled tasks would miss. Instead, the AI Agent checks if a review is due at the start of every conversation.
- **Data stays local.** All trade data lives in your Postgres instance. Nothing is sent anywhere except stock tickers to FMP and analysis prompts to OpenRouter.
- **Survives restarts.** Postgres data is in a Docker volume. You can `docker compose down` and `up` without losing anything.
- **Survives `docker system prune`.** As long as you don't prune volumes, your data is safe.

---

_Blueprint version 3.1 - March 2026_
