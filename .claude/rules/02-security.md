---
description: Security rules — secrets, SQL injection, credential handling
globs: ["**/*.json", "docker-compose.yml", ".env.example"]
---

# Security

## Secrets

- All secrets in `.env` (gitignored). `docker-compose.yml` uses `${VAR}`.
- `.env.example` — only placeholders: `CHANGE_ME_...`, `your-...-here`, `sk-or-v1-your-key-here`
- Workflow JSONs use `YOUR_FMP_KEY` and `YOUR_OPENROUTER_KEY` — intentional placeholders, don't replace with env var syntax (n8n HTTP nodes don't support it)
- Credential nodes: `"id": ""` is correct — n8n maps by name on import. Never fill real IDs.

## Before Committing

Check for:
- `sk-or-v1-` followed by anything other than `your-key-here`
- Long alphanumeric strings that look like real API keys
- Passwords that aren't `CHANGE_ME_...`
- Bearer tokens that aren't `YOUR_OPENROUTER_KEY`
- `.env` accidentally staged

## SQL Injection

n8n builds SQL via expressions like `'{{ $json.ticker }}'`. Mitigations:
- Ticker symbols are filtered upstream (alpha + dot only)
- Trade data is computed from validated numeric inputs
- Text fields use `.replace(/'/g, "''")`

When adding SQL queries: validate input before it reaches the query. Use numeric types for numbers. Sanitize strings for single-quote injection.
