# Setup Guide

From zero to your first trading scan. Takes about 15 minutes.

---

## 1. What You'll Need

Before starting, install these two things:

- **Docker Desktop** - runs the entire system in containers (think of containers as isolated programs that come pre-packaged with everything they need - no manual dependency installation).
  - [Mac](https://docs.docker.com/desktop/install/mac-install/)
  - [Windows](https://docs.docker.com/desktop/install/windows-install/)
  - [Linux](https://docs.docker.com/desktop/install/linux/)

- **Git** - to download the project. If you're reading this, you probably already have it. If not: [git-scm.com/downloads](https://git-scm.com/downloads)

---

## 2. Get Your API Keys

You need two keys (one optional). All free tiers, no credit card required for FMP and Finnhub.

> **Note:** The system also uses SearXNG for real-time web search, but it's self-hosted - no API key needed. You'll configure its credential in n8n during [step 5](#configure-the-searxng-credential).

### OpenRouter (required)

OpenRouter is an AI gateway - it routes your requests to language models like Gemini Flash. This is the "brain" that analyzes market data and generates trading signals.

1. Go to [openrouter.ai/keys](https://openrouter.ai/keys)
2. Create an account
3. Generate an API key
4. Copy the key (starts with `sk-or-v1-...`)

Cost: free credits to start, then pay-per-use. A typical market scan costs ~$0.005. Expect ~$0.15–0.50/month with regular use.

### FMP - Financial Modeling Prep (required)

FMP provides market data: stock prices, gainers/losers, earnings calendars, and company news.

1. Go to [financialmodelingprep.com](https://financialmodelingprep.com)
2. Sign up for a free account
3. Go to your Dashboard to find your API key
4. Copy the key

Free tier: 250 API requests per day. No credit card needed.

### Finnhub (optional)

Finnhub provides supplementary company news. You can skip this - FMP news and Google RSS still cover your needs.

1. Go to [finnhub.io](https://finnhub.io)
2. Sign up for a free account
3. Go to your Dashboard to find your API key
4. Copy the key

Free tier: 60 requests per minute. No credit card needed.

---

## 3. Download & Configure

### Clone the repository

```bash
git clone <repository-url>
cd trading-wizard
```

### Create your environment file

```bash
cp .env.example .env
```

**What is `.env`?** It's a file that holds your secrets (passwords, API keys) locally on your machine. It's listed in `.gitignore`, so it never gets committed to Git or shared anywhere. You edit it once, and Docker reads it automatically.

### Fill in your `.env`

Open `.env` in any text editor and update these values:

| Variable                    | What it is                           | What to do                                                                                                                                        |
| --------------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `POSTGRES_USER`             | Database username                    | Leave as `trading_wizard` (default is fine)                                                                                                       |
| `POSTGRES_PASSWORD`         | Database password                    | **Change this.** Pick something strong. Only used locally.                                                                                        |
| `POSTGRES_DB`               | Database name                        | Leave as `trading_wizard` (default is fine)                                                                                                       |
| `POSTGRES_PORT`             | Host-side port for external DB tools | Leave as `5432`. If you remove this line entirely, Docker defaults to `5433` instead. Only change if another Postgres is already using that port. |
| `N8N_PORT`                  | n8n web UI port                      | Leave as `5678`. Only change if something else uses that port.                                                                                    |
| `N8N_DEFAULT_USER_EMAIL`    | n8n login email                      | Leave as `admin@local.host` or set your own. Local only.                                                                                          |
| `N8N_DEFAULT_USER_PASSWORD` | n8n login password                   | **Change this.** Pick something strong.                                                                                                           |
| `TIMEZONE`                  | Your local timezone                  | Change to your timezone (e.g., `America/New_York`, `Europe/London`). Affects scheduling displays.                                                 |
| `OPENROUTER_API_KEY`        | Your OpenRouter key                  | Paste the key you got in step 2                                                                                                                   |
| `FMP_API_KEY`               | Your FMP key                         | Paste the key you got in step 2                                                                                                                   |
| `FINNHUB_API_KEY`           | Your Finnhub key                     | Paste the key you got in step 2, or leave as-is if skipping Finnhub                                                                               |

---

## 4. Start the System

```bash
docker compose up -d
```

This starts three containers:

- **n8n** - the workflow engine (your trading assistant's brain)
- **postgres** - the database (stores signals, trades, and insights)
- **searxng** - meta-search engine (gives the AI real-time web search capability)

### Verify everything is running

```bash
docker compose ps
```

You should see both containers with status `running` or `healthy`.

### If something looks wrong

Check the logs:

```bash
docker compose logs n8n
docker compose logs postgres
```

---

## 5. Set Up n8n

This is the main setup section. Take it step by step.

### First login

1. Open [http://localhost:5678](http://localhost:5678) in your browser
2. Create your account using the email and password from your `.env`
3. This account is local only - nothing is shared externally

### Configure the Postgres credential

The workflows need to talk to your trading database. You set this up once.

1. Go to **Settings** (gear icon) → **Credentials** → **Add Credential**
2. Search for **Postgres** and select it
3. Fill in these fields:

| Field        | Value                                      |
| ------------ | ------------------------------------------ |
| **Name**     | `Trading Wizard DB`                        |
| **Host**     | `postgres`                                 |
| **Port**     | `5432`                                     |
| **Database** | Same as `POSTGRES_DB` in your `.env`       |
| **User**     | Same as `POSTGRES_USER` in your `.env`     |
| **Password** | Same as `POSTGRES_PASSWORD` in your `.env` |

> **The credential name MUST be exactly `Trading Wizard DB`.** All 7 workflows reference this exact name. Any variation - extra space, different capitalization, different wording - causes silent failures where database operations just don't work.

> **Why `postgres` as the host and not `localhost`?** Inside Docker, containers talk to each other by service name. The database container is named `postgres` in docker-compose.yml, so that's the hostname n8n uses. The internal Docker port is always `5432`, regardless of what you set for `POSTGRES_PORT` in `.env` (that setting only affects access from outside Docker).

> **Note:** n8n's own internal database connection is handled automatically by Docker. This credential is specifically for your trading data (signals, trades, insights).

### Configure the SearXNG credential

The AI agent uses SearXNG for real-time web search (verifying prices, checking news, researching companies).

1. Go to **Settings** → **Credentials** → **Add Credential**
2. Search for **SearXNG** and select it
3. Fill in:

| Field    | Value                 |
| -------- | --------------------- |
| **Name** | `SearXNG`             |
| **URL**  | `http://searxng:8080` |

> Same Docker networking logic as Postgres - `searxng` is the container name, `8080` is its internal port.

### Import workflows

1. In n8n, click the **"..."** menu (or **Import from File** option)
2. Import all 7 files from the `workflows/` folder, in this order:
   - `01-db-init.json`
   - `02-get-history.json`
   - `03-get-insights.json`
   - `04-trade-logger.json`
   - `05-market-scanner.json`
   - `06-weekly-review.json`
   - `07-chat-main.json`
3. After each import, verify the workflow loaded correctly - you should see nodes connected with lines between them

### Replace API key placeholders

The workflow files contain placeholder strings instead of real keys. You need to replace them with your actual keys.

| Placeholder           | Replace with            | Where to find it                                                                                    |
| --------------------- | ----------------------- | --------------------------------------------------------------------------------------------------- |
| `YOUR_FMP_KEY`        | Your FMP API key        | In the **URL** (query parameter `apikey=`) of FMP HTTP Request nodes                                |
| `YOUR_OPENROUTER_KEY` | Your OpenRouter API key | In the **Authorization header** of OpenRouter HTTP Request nodes                                    |
| `YOUR_FINNHUB_KEY`    | Your Finnhub API key    | In the **URL** (query parameter `token=`) of Finnhub HTTP Request nodes. Skip if not using Finnhub. |

**How to find and replace them:**

1. Open a workflow (e.g., `05-market-scanner`)
2. Click on any HTTP Request node
3. Look in the URL field or Headers section for the placeholder text
4. Replace the placeholder with your actual key
5. Save the workflow
6. Repeat for all HTTP Request nodes across all workflows

**Alternative:** You can use n8n's built-in credential system to create "Header Auth" credentials and attach them to the relevant nodes instead of editing URLs directly.

### Publish and activate workflows

> **This is the most common "it doesn't work" mistake. Read carefully.**

There are two separate concepts in n8n:

- **Publishing** - makes a workflow available to be called by other workflows (as a sub-workflow / tool)
- **Activating** - starts a workflow's trigger so it listens for events (like chat messages)

**Sub-workflows (02 through 06) must be published:**

The main chat workflow (07) calls workflows 02–06 as tools. If they aren't published, you'll get "workflow not found" errors or tools will silently fail and return nothing.

How to publish each one:

1. Open the workflow
2. Click the **"..."** menu → **Publish**, or toggle the workflow status

Do this for: `02-get-history`, `03-get-insights`, `04-trade-logger`, `05-market-scanner`, `06-weekly-review`.

**Main chat workflow (07) must be activated:**

This workflow has a chat trigger that listens for your messages. It needs to be active.

1. Open `07-chat-main`
2. Toggle the **Active** switch (top-right corner) to ON

**01-db-init does NOT need publishing or activation.** You run it manually once (next step).

---

## 6. Initialize the Database

1. Open the `01-db-init` workflow in n8n
2. Click **Execute Workflow** (the play button)
3. Wait for it to complete

What it does: creates 3 database tables (`signals`, `trades`, `insights`) with all required columns and indexes. This is a one-time operation - running it again is safe (it uses `IF NOT EXISTS`).

**How to verify:** both nodes should show green checkmarks after execution. If you see a red error on the Postgres node, your database credential is misconfigured - go back to the "Configure the Postgres credential" step.

---

## 7. Your First Scan

1. Open the `07-chat-main` workflow in n8n
2. Click the **chat icon** (bottom-right corner) to open the chat panel
3. Type: **"I have $500, scan US market"**
4. Wait 30–60 seconds for the system to process

**What happens behind the scenes:** the AI agent calls the market scanner, which fetches gainers/losers from FMP, enriches them with quotes and news, runs AI analysis on each candidate, validates risk/reward ratios, and returns ranked trading signals with entry prices, stop-losses, and risk assessments.

**Best results during US market hours:** 9:30 AM – 4:00 PM Eastern Time. Outside these hours, market data may be stale or the screener may return no results.

---

## 8. Troubleshooting

| Problem                                          | Solution                                                                                                                                                       |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Port 5678 already in use                         | Change `N8N_PORT` in `.env`, then restart: `docker compose down && docker compose up -d`                                                                       |
| Docker not running                               | Start Docker Desktop first, then run `docker compose up -d` again                                                                                              |
| n8n can't connect to Postgres                    | Verify the credential name is exactly `Trading Wizard DB`. Host must be `postgres` (not `localhost`). Check that user/password/database match your `.env`.     |
| "Invalid API key" errors                         | Double-check key values in HTTP Request nodes. Watch for extra spaces or leftover quotes around the key.                                                       |
| Scanner returns no results                       | Market may be closed. Try during US trading hours (9:30 AM – 4:00 PM Eastern).                                                                                 |
| Workflows not triggering / chat doesn't respond  | Make sure `07-chat-main` is toggled **Active** (top-right switch).                                                                                             |
| "Workflow not found" or tools not working        | Sub-workflows (02–06) must be **published** in n8n. See step 5.                                                                                                |
| Credential mapping lost after import             | Re-open the Postgres node in each workflow, click the credential dropdown, and re-select `Trading Wizard DB`.                                                  |
| Web search not working / AI agent never searches | Make sure SearXNG credential is configured: name `SearXNG`, URL `http://searxng:8080`. Check that the `searxng` container is running with `docker compose ps`. |
| `docker compose` command not found               | You may need to use `docker-compose` (with hyphen) on older Docker versions, or update Docker Desktop.                                                         |
