---
description: Core principles for every interaction with Trading Wizard repo
globs: ['**/*']
---

# Core Principles

Priority order - when in conflict, higher wins:

1. **Security** - no leaked secrets, no SQL injection, no weakened safeguards
2. **Reliability** - every external call can fail; handle it
3. **Minimalism** - smallest change that solves the problem
4. **Optimization** - respect FMP 250 req/day, minimize AI tokens

## Hard Rules

- NEVER hardcode API keys or passwords in tracked files
- NEVER remove risk disclosure fields from signals
- NEVER remove `educational_note` or `reasoning` - user is learning
- NEVER weaken validation (60% confidence min, 1.5 R/R min)
- NEVER add cron jobs - machine isn't always on
- NEVER commit `.env` - only `.env.example` with placeholders
- NEVER refactor working code that wasn't asked about
- NEVER remove try/catch or error handling to "clean up"

## Before Every Change

1. Read the relevant workflow JSON + its `docs/BLUEPRINT.md` section
2. Trace data flow - what's upstream and downstream of your change
3. Identify blast radius - what depends on this

## After Every Change

1. Validate modified JSON is valid
2. Mentally walk the pipeline - does data arrive in expected shape?
3. Check edge cases: 0 candidates, malformed AI JSON, empty DB, all APIs down
4. Update docs if behavior changed (see CLAUDE.md → Documentation Sync)
5. Review the diff for security risks or resilience regressions
