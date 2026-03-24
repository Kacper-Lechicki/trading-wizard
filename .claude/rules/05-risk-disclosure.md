---
description: Risk disclosure requirements - mandatory fields on every BUY signal
globs: ['workflows/05-market-scanner.json', 'workflows/07-chat-main.json']
---

# Risk Disclosure

Every BUY signal MUST include all of these fields - enforced at three levels:

## Level 1 - AI Prompt (Build AI Prompt node)

The prompt explicitly requires all risk fields in the JSON schema.

## Level 2 - Validation (Validate & Enrich node)

Hard rules:

- `confidence < 60%` → auto-downgrade to SKIP
- `risk_reward < 1.5` → auto-downgrade to SKIP
- Missing fields → calculate mathematically from entry/TP/SL
- Max 5 BUY signals per session

## Level 3 - Chat Presentation (AI Agent system prompt)

Agent must present: cost, profit $+%, loss $+%, R/R, stop-loss, risk warnings, plain-language summary.

## Mandatory Fields

| Field              | Formula / Source                                           |
| ------------------ | ---------------------------------------------------------- |
| `estimated_cost`   | `suggested_shares * midpoint(entry_low, entry_high)`       |
| `upside_pct`       | `((take_profit - entry_mid) / entry_mid) * 100`            |
| `downside_pct`     | `((stop_loss - entry_mid) / entry_mid) * 100` (negative)   |
| `risk_reward`      | `abs(upside_pct / downside_pct)`                           |
| `potential_profit` | `(take_profit - entry_mid) * suggested_shares`             |
| `potential_loss`   | `(stop_loss - entry_mid) * suggested_shares` (negative)    |
| `stop_loss`        | Price level below nearest support                          |
| `risk_level`       | `low` / `moderate` / `high`                                |
| `risk_warnings`    | Array: earnings proximity, low volume, conflicting signals |
| `reasoning`        | In user's language                                         |
| `educational_note` | In user's language                                         |

Never remove any of these. The user is learning and must see the full picture.
