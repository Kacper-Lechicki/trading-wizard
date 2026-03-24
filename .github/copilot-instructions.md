# GitHub Copilot Instructions — Trading Wizard

## Project Context

This is an n8n workflow-based intraday trading assistant. The codebase is workflow JSON files + Docker config + documentation. No traditional source code, no npm, no build step.

## Key Constraints

1. **No hardcoded secrets.** API keys use placeholders (`YOUR_FMP_KEY`, `YOUR_OPENROUTER_KEY`). Passwords use `${VAR}` in docker-compose.yml. Real values in `.env` (gitignored).
2. **Every HTTP call can fail.** Always include `timeout` and `onError: "continueRegularOutput"` on HTTP Request nodes.
3. **AI responses are unreliable.** Always parse with try/catch. Always validate required fields. Always provide user-friendly fallback.
4. **Risk fields are mandatory.** Every BUY signal: `estimated_cost`, `upside_pct`, `downside_pct`, `risk_reward`, `potential_profit`, `potential_loss`, `stop_loss`, `risk_level`, `risk_warnings`, `reasoning`, `educational_note`.
5. **FMP has 250 req/day limit.** Don't add unnecessary API calls. Batch where possible.
6. **English internals, Polish responses.** Code and prompts in English. AI generates user-facing text in user's language.

## When Suggesting Code in Workflow JSONs

- n8n Code nodes use Node.js JavaScript
- Input access: `$input.all()`, `$input.first().json`
- Cross-node access: `$('NodeName').first().json`
- Return: `[{ json: { ... } }]`
- Always handle empty/null upstream data
- Always use try/catch around JSON.parse
- Sanitize strings before SQL interpolation (`.replace(/'/g, "''")`)

## Documentation

- `CLAUDE.md` — full agent instructions with checklists
- `docs/BLUEPRINT.md` — node-by-node workflow descriptions
- Update docs when changing workflow behavior

## Don't

- Don't add cron jobs (self-hosted, machine not always on)
- Don't add npm dependencies
- Don't refactor working code that wasn't asked about
- Don't remove error handling or try/catch blocks
- Don't weaken validation (60% confidence min, 1.5 R/R min)
