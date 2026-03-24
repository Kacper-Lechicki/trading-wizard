---
description: Mandatory documentation sync after any behavioral change
globs: ["workflows/*.json", "docker-compose.yml", ".env.example", "CLAUDE.md", "docs/BLUEPRINT.md"]
---

# Documentation Sync

After every modification, check this table and update accordingly:

| What changed | What to update |
|---|---|
| Tool description in 07-chat-main.json | `docs/BLUEPRINT.md` → Chat section |
| Scanner pipeline (add/remove/modify nodes) | `docs/BLUEPRINT.md` → Market Scanner section |
| Trade logger logic | `docs/BLUEPRINT.md` → Trade Logger section |
| Weekly review logic | `docs/BLUEPRINT.md` → Weekly Review section |
| Database schema (column, table, index) | `docs/BLUEPRINT.md` → Database section + `01-db-init.json` |
| New API source | `docs/BLUEPRINT.md` → External APIs + `.env.example` + `CLAUDE.md` Architecture |
| New workflow file | `CLAUDE.md` → Repository Structure + Architecture |
| Risk disclosure fields | `CLAUDE.md` → Risk Disclosure table |
| docker-compose.yml | `README.md` setup instructions |

**Skipping doc updates means the next agent works with stale info and makes wrong decisions. This is mandatory, not optional.**
