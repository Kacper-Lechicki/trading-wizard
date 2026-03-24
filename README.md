# Trading Wizard

Personal AI-powered intraday trading assistant for US and European stocks. Built as n8n workflows, self-hosted via Docker. Scans the market, gives you exact trading instructions, and learns from your results.

---

## What It Does

You type your budget into a chat, the system scans the entire market and returns specific instructions:

> _"Buy 5 shares of AAPL at $185.20-$185.60. Stop-loss: $184.10. Target: $187.40. Cost: $927.00. Potential profit: +$9.00 (+1.08%). Potential loss: -$6.50 (-0.70%). Risk/Reward: 1.54. This is a moderate-risk trade — the stock shows strong volume and positive momentum, but earnings are in 5 days."_

Every signal includes an educational explanation — you learn WHY while you trade.

Over time, the system learns your patterns and adapts recommendations to your trading style.

---

## How It Works

**Morning** — before market open:

1. Open n8n chat (`http://localhost:5678`)
2. Type: "I have $1000 for today, scan US market"
3. The system autonomously: screens for movers, fetches price data, calculates technical indicators (RSI, MACD, Bollinger Bands), gathers news from 3 sources, checks earnings calendar, runs AI analysis
4. You get ranked signals with exact entry/exit/stop-loss, cost, risk, and plain-language explanations

**Evening** — report results:

- Type: "AAPL bought at 185.40, sold at 187.20, 5 shares, hit take-profit"
- System logs the trade, calculates P&L, stores feedback

**Weekly** — automatic review prompt:

- System detects when a review is due and suggests it
- Analyzes your last 20-30 trades, generates personalized insights
- Insights are injected into all future scans

---

## Risk Transparency

Every signal includes full risk disclosure:

| Field             | Description                                         |
| ----------------- | --------------------------------------------------- |
| Entry range       | Realistic buy price range                           |
| Stop-loss         | Maximum loss price — where to exit if wrong         |
| Take-profit       | Target sell price                                   |
| Estimated cost    | Total position cost in your currency                |
| Potential profit  | Dollar amount and percentage if target hit          |
| Potential loss    | Dollar amount and percentage if stop-loss hit       |
| Risk/Reward ratio | How much you stand to gain vs lose (minimum 1.5)    |
| Confidence        | AI's confidence level (0-100%)                      |
| Reasoning         | Why this trade, in plain language                   |
| Risk warnings     | Earnings proximity, low volume, conflicting signals |

**The system will never hide downside risk.** If a trade has unfavorable risk, it will be marked as SKIP with an explanation.

---

## Tech Stack

| Component       | Technology                | Cost         |
| --------------- | ------------------------- | ------------ |
| Workflow engine | n8n (self-hosted)         | Free         |
| Database        | Postgres (Docker)         | Free         |
| Market data     | FMP (250 req/day)         | Free         |
| News (source 1) | FMP stock news            | Free         |
| News (source 2) | Finnhub (optional)        | Free         |
| News (source 3) | Google News RSS           | Free         |
| Fallback data   | Yahoo Finance             | Free         |
| AI analysis     | OpenRouter (Gemini Flash) | ~$0.005/scan |

**Estimated monthly cost: $0.15-0.50** (22 trading days).

---

## Quick Start

### Prerequisites

- Docker + Docker Compose
- OpenRouter API key ([openrouter.ai/keys](https://openrouter.ai/keys))
- FMP API key ([financialmodelingprep.com](https://financialmodelingprep.com))
- (Optional) Finnhub API key ([finnhub.io](https://finnhub.io))

### Setup (5 minutes)

```bash
# 1. Clone
git clone <repo-url>
cd trading-wizard-n8n

# 2. Configure environment
cp .env.example .env
# Edit .env — fill in your API keys and set strong passwords

# 3. Start
docker compose up -d

# 4. Open n8n
# Go to http://localhost:5678
# First time: create a local account (email + password, purely local)
# Then: set up credentials and import workflows (see docs/BLUEPRINT.md)
```

### Environment Variables

| Variable                    | Required | Description                             |
| --------------------------- | -------- | --------------------------------------- |
| `POSTGRES_USER`             | Yes      | Database username                       |
| `POSTGRES_PASSWORD`         | Yes      | Database password (choose a strong one) |
| `POSTGRES_DB`               | Yes      | Database name                           |
| `N8N_DEFAULT_USER_PASSWORD` | Yes      | n8n login password                      |
| `OPENROUTER_API_KEY`        | Yes      | OpenRouter API key for AI analysis      |
| `FMP_API_KEY`               | Yes      | FMP key for market data                 |
| `FINNHUB_API_KEY`           | No       | Finnhub key for additional news         |
| `TIMEZONE`                  | No       | Default: `Europe/Warsaw`                |

---

## Workflows

| Workflow           | Purpose                                            | Trigger           |
| ------------------ | -------------------------------------------------- | ----------------- |
| **Chat**           | Main interface — routes to all other workflows     | Chat message      |
| **Market Scanner** | Screens market, calculates indicators, AI analysis | scan_market tool  |
| **Trade Logger**   | Records trade results, calculates P&L              | log_trade tool    |
| **Weekly Review**  | Analyzes performance, generates insights           | run_review tool   |
| **Get History**    | Queries trade history                              | get_history tool  |
| **Get Insights**   | Returns insights + checks if review is due         | get_insights tool |

Full node-by-node documentation: [docs/BLUEPRINT.md](docs/BLUEPRINT.md).

---

## Disclaimers

- **Delayed data.** FMP free tier has ~15 min delay. The system sets price levels to target, not minute-precise timing. You see real-time prices in your broker.
- **Intraday is hard.** Realistic win rate after 2-3 months: 55-65%. Profitable only with strict stop-loss discipline.
- **Self-hosted.** n8n runs only when your computer is on. No cron dependency — the AI agent checks if tasks are due at every conversation start.
- **Paper trade first.** Minimum 4 weeks of paper trading, 30+ closed positions, and win rate above 52% before considering real money.
- **Not financial advice.** This is a tool to assist your decisions. You are responsible for every trade you make.

---

## Documentation

- [Blueprint](docs/BLUEPRINT.md) — full technical reference, every node described
- [CLAUDE.md](CLAUDE.md) — AI agent instructions for working on this repo

---

## License

MIT
