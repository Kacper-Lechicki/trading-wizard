Scan all tracked files in this repository for security issues.

Check for:

1. Hardcoded API keys, passwords, or tokens (anything that looks like a real secret, not a placeholder like `YOUR_FMP_KEY` or `CHANGE_ME_...`)
2. `.env` file accidentally staged for commit
3. SQL injection vectors in workflow JSON files (string interpolation in SQL queries without sanitization)
4. Missing timeout or error handling on HTTP Request nodes
5. Credential nodes with non-empty `id` fields (would leak n8n-specific credential IDs)
6. Any file that should be in `.gitignore` but isn't

Report findings as a checklist: PASS/FAIL for each category, with file:line references for any failures.
