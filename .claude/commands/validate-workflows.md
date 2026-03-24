Validate all workflow JSON files in the `workflows/` directory.

For each file, check:
1. **Valid JSON** — file parses without errors
2. **Node references** — every node name in `connections` exists in `nodes[]`
3. **No orphan nodes** — every node (except triggers) is referenced in at least one connection
4. **HTTP nodes** — all HTTP Request nodes have `timeout` and `onError` set
5. **Credential consistency** — all Postgres nodes reference `"Trading Wizard DB"`, all OpenAI nodes reference `"OpenRouter"`
6. **Code node safety** — code nodes that parse JSON use try/catch
7. **Risk fields** — if the file is the market scanner, verify that Validate & Enrich checks for all mandatory risk fields
8. **Placeholder keys** — API keys are placeholders (`YOUR_FMP_KEY`, `YOUR_OPENROUTER_KEY`), not real values

Report as a table: file | check | status (PASS/FAIL) | details.
