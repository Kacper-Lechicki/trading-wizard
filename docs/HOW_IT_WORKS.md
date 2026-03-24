# How Trading Wizard Works

A plain-language explanation of what happens behind the scenes when you use Trading Wizard.

---

## 1. The Big Picture

You type a message, the system figures out what you need, does it, and responds.

At the center of everything is the **AI Agent** - think of it as the brain. It reads your message, decides what you need, and calls the right tool to get it done. There are six tools it can use:

```
                          +------------------+
                          |   scan_market    |  Scan stocks, find opportunities
                          +------------------+
                                  ^
                                  |
+--------+    +-----------+   +--+--+   +------------------+
|  You   | -> | AI Agent  | --+     +-- |    log_trade     |  Record your results
| (chat) |    | (brain)   |   +--+--+   +------------------+
+--------+    +-----------+      |
                                 +----> +------------------+
                                 |      |   run_review     |  Analyze your performance
                                 |      +------------------+
                                 |
                                 +----> +------------------+
                                 |      |   get_history    |  Show past trades
                                 |      +------------------+
                                 |
                                 +----> +------------------+
                                 |      |   get_insights   |  Load lessons learned
                                 |      +------------------+
                                 |
                                 +----> +------------------+
                                        |   web_search     |  Search the internet
                                        +------------------+
```

There is also a one-time setup workflow (DB Init) that creates the database tables when you first install the system. You run it once and never think about it again.

The system supports both **US markets** (NYSE, NASDAQ) and **European markets** (London, Frankfurt, Euronext). You choose which markets to scan when you ask for recommendations.

The key insight: **the AI Agent is the brain.** It reads your message, decides what you need, and calls the right tool. You never need to remember commands or navigate menus - just talk to it naturally.

---

## 2. What Happens When You Ask for a Scan

This is the longest and most important section, because the market scanner is where all the magic happens. Here is what happens step by step when you type something like "I have $1000, scan US market":

**Step 1 - You ask.**
You type your budget and market preference into the chat. The AI Agent recognizes that you want a market scan and calls the Market Scanner tool with your budget, currency, and market choice.

**Step 2 - Fetch today's movers.**
The scanner asks a financial data provider (FMP - Financial Modeling Prep) for three lists of stocks that are moving today:

- **Top gainers** - stocks that went up the most today
- **Top losers** - stocks that dropped the most (oversold stocks sometimes bounce back)
- **Most active** - stocks with the highest trading volume (lots of buyers and sellers = easier to trade)

These three lists are combined and duplicates are removed. This gives roughly 80-100 unique stocks to consider.

**Step 3 - Filter out the noise.**
Not every stock is suitable for trading. The scanner removes:

- Penny stocks (priced under $1) - too volatile and risky
- OTC stocks (not listed on major exchanges) - too illiquid
- Stocks too expensive for your budget (unless you use fractional shares - buying a portion of one share)

After filtering, the top 10 candidates remain, sorted by how much they moved today.

**Step 4 - Get detailed price data.**
For those 10 candidates, the scanner fetches detailed quotes: current price, day's high and low, trading volume, and market cap. From this data, it calculates technical indicators:

- **RSI** (Relative Strength Index) - a number from 0 to 100 that tells you if a stock is "overbought" (above 70, might drop soon) or "oversold" (below 30, might bounce up)
- **MACD** - shows whether a stock's momentum is increasing or decreasing, and in which direction
- **Bollinger Bands** - shows if the current price is unusually high or low compared to its recent range

These indicators help the AI make better decisions. If the detailed data is not available (a limitation of the free data plan), the scanner continues with the basic data from step 2 - it never stops.

**Step 5 - Gather news from three sources.**
The scanner checks three independent news sources for each candidate:

- Company-specific news from FMP (the financial data provider)
- General stock news from Google News (free, no limits)
- Supplementary company news from Finnhub (another financial data provider)

Using three sources means if one is down or misses a story, the others catch it. News is grouped by stock and limited to the 5 most relevant headlines per stock.

**Step 6 - Check the earnings calendar.**
The scanner checks if any candidate is about to report earnings (quarterly financial results) in the next 48 hours. Earnings announcements are unpredictable - the stock can jump 10% in either direction regardless of what the charts say. If a stock has upcoming earnings, the AI will flag it as an extra risk.

**Step 7 - Load your personal insights.**
The scanner checks the database for lessons from your previous weekly reviews. These are things like "signals with high trading volume have a 78% win rate for this user" or "this user tends to exit trades too early." On your first use, this is empty - and that is perfectly fine.

**Step 8 - Ask the AI.**
Everything gathered so far - stock data, indicators, news, earnings warnings, and your personal insights - is packaged into a structured prompt and sent to an AI model (Google Gemini Flash). The AI analyzes each stock and returns a verdict:

- **BUY** - worth trading, with specific entry price, stop-loss, take-profit, and reasoning
- **SKIP** - not a good trade today, with an explanation of why

**Step 9 - Validate the results.**
The AI is smart but not perfect. A validation layer checks every signal against hard rules:

- Is the confidence level at least 60%? If not, the signal is downgraded to SKIP.
- Is the risk/reward ratio at least 1.5? (Meaning the potential gain is at least 1.5 times the potential loss.) If not, downgraded to SKIP.
- Are all required fields present? If anything is missing, the system calculates it automatically.
- No more than 5 BUY signals per scan - quality over quantity.

Signals that fail these checks are automatically changed to SKIP with a clear explanation of why.

**Step 10 - Present the results.**
The AI Agent presents the final results in your language with full risk disclosure for every BUY signal:

- Entry price range (where to buy)
- Stop-loss (where to sell if it goes against you - your safety net)
- Take-profit (where to sell if it goes your way)
- Estimated cost (how much the position costs)
- Potential profit in dollars and percentage
- Potential loss in dollars and percentage
- Risk/reward ratio
- Risk warnings (upcoming earnings, low volume, conflicting signals)
- A plain-language explanation of why this stock was selected
- An educational note - a short lesson related to this signal

Here is the full pipeline as a diagram:

```
You: "I have $1000, scan US market"
    |
    v
AI Agent recognizes scan intent
    |
    v
+-------------------------------------------+
|           MARKET SCANNER                   |
|                                            |
|  1. Fetch top movers from FMP             |
|     (gainers + losers + most active)       |
|              |                             |
|  2. Filter out junk, keep top 10          |
|              |                             |
|  3. Get detailed prices + indicators       |
|     (RSI, MACD, Bollinger Bands)           |
|              |                             |
|  4. Gather news (3 sources)               |
|     (FMP + Google News + Finnhub)          |
|              |                             |
|  5. Check earnings calendar               |
|              |                             |
|  6. Load your personal insights           |
|              |                             |
|  7. Send everything to AI for analysis    |
|              |                             |
|  8. Validate: confidence >= 60%?          |
|              risk/reward >= 1.5?           |
|              all fields present?           |
|              max 5 BUY signals?            |
|              |                             |
|  9. Format results with full risk info    |
+-------------------------------------------+
    |
    v
AI Agent presents results in your language
```

---

## 3. Logging Your Trades

After you act on a recommendation and complete a trade, you tell the chat what happened. For example:

> "AAPL bought at 185.40, sold at 187.20, 5 shares, hit take-profit"

Here is what happens next:

1. **The system calculates your profit or loss** - both in dollars and as a percentage. In this example: +$9.00 (+0.97%).
2. **It determines the outcome** - WIN, LOSS, or BREAKEVEN - based on the numbers.
3. **It links your trade to the original AI signal** that recommended it. This connection is important: during weekly reviews, the system can compare what the AI predicted versus what actually happened.
4. **Everything is saved to the database** - prices, quantity, profit/loss, outcome, and any feedback you want to add (like "I exited too early" or "the stop-loss saved me").

You can also add personal notes and feedback tags to help the system learn your patterns. For example, tagging a trade with "exited too early" tells the weekly review that you have a tendency to sell before hitting your target.

---

## 4. Weekly Review and Learning

This is where Trading Wizard gets smarter over time.

**When does it happen?**
The system does not run reviews on a fixed schedule (your computer might not be on at the right time). Instead, every time you start a new conversation, the AI silently checks: "Has it been more than 7 days since the last review? Are there enough trades to analyze?" If both are true, it suggests running a review.

**What does the review do?**

1. **Loads your recent trades** - the last 20-30 trades from the past month, along with the original AI signals that recommended them.
2. **Calculates statistics** - win rate, average profit on winners, average loss on losers, total profit/loss, performance broken down by confidence level, and which feedback tags come up most often.
3. **Sends everything to the AI for analysis** - the AI looks for patterns in your trading. What distinguishes your winning trades from your losing ones?
4. **Generates personalized insights** - concrete, actionable observations. For example:
   - "Signals with high trading volume have a 78% win rate for you"
   - "You tend to exit winning trades too early - average remaining upside was +1.2%"
   - "Your best results come from stocks with confidence above 75%"
   - "TSLA: 2 wins out of 6 trades - consider being more cautious with this stock"
5. **Saves the insights to the database** - and here is the key part: these insights are loaded into every future market scan. The AI reads your past lessons before making new recommendations.

This creates a learning loop:

```
+------------------+       +------------------+       +------------------+
|   Your Trades    | ----> |  Weekly Review   | ----> | Insights Saved   |
| (logged results) |       | (AI finds        |       | (lessons learned)|
+------------------+       |  patterns)       |       +------------------+
        ^                  +------------------+               |
        |                                                     |
        |                                                     v
+------------------+       +------------------+       +------------------+
| Better Decisions | <---- | Better Scans     | <---- | Insights Loaded  |
| (you improve)    |       | (AI adapts to    |       | (into next scan) |
+------------------+       |  your patterns)  |       +------------------+
                           +------------------+
```

**The system gets smarter the more you use it.** Early scans are generic. After a few weeks of trading and reviews, the AI knows your strengths, weaknesses, and preferences - and adjusts its recommendations accordingly.

---

## 5. What If Something Goes Wrong?

Trading Wizard is built with the assumption that things will go wrong - and it handles them gracefully.

**If an API call fails:**
Every external data request (stock prices, news, earnings data) has a timeout and a fallback. If one call fails, the system continues with whatever data it has. It never crashes because one data source is temporarily unavailable.

**If one news source is down:**
The scanner uses three independent news sources. If one or even two are down, the remaining source still provides context for the AI analysis.

**If detailed price data is not available:**
The free data plan does not support all endpoints. If detailed quotes or earnings data cannot be fetched (you might see a 403 error in the logs), the scanner falls back to the basic price data from the initial stock lists. The analysis is slightly less detailed, but it still works.

**If all APIs fail:**
In the unlikely event that every data source is unreachable, you get a clear error message explaining what happened - not a cryptic crash or a blank screen.

**If the AI returns a bad response:**
Sometimes the AI model returns malformed data (incomplete or badly structured). The system wraps every AI response in error handling. If the response cannot be parsed, you get a friendly message asking you to try again, rather than seeing raw error data.

**Your data is safe:**
All your trades, signals, and insights are stored locally in your Docker database. The data survives restarts, updates, and even temporarily losing internet access. Nothing is stored on external servers.

**The golden rule:** the scanner always reaches the AI analysis step with whatever data it could gather. A scan with partial data is better than no scan at all.
