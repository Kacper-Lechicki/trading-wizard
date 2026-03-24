---
description: Rules for editing n8n workflow JSON files - node structure, connections, code patterns
globs: ['workflows/*.json']
---

# Workflow Editing

## JSON Structure

```
{ name, nodes[], connections, settings, tags }
```

Node: `{ parameters, id, name, type, typeVersion, position, credentials?, onError? }`

## Node IDs

- Unique UUIDs within the workflow - don't change existing ones
- New nodes: follow the file's existing UUID pattern

## Connections

- Reference nodes by `name` (not ID)
- Format: `"NodeName": { "main": [[ { "node": "Target", "type": "main", "index": 0 } ]] }`
- IF nodes: outer array index 0 = true branch, index 1 = false branch

## Code Nodes (n8n-nodes-base.code)

- Node.js JavaScript runtime
- Input: `$input.all()`, `$input.first().json`
- Cross-node: `$('NodeName').first().json`
- Return: `[{ json: { ... } }]`
- ALWAYS handle empty/null upstream data
- ALWAYS try/catch around `JSON.parse`

## HTTP Request Nodes

- `"timeout": 10000` for APIs, `120000` for AI calls
- `"onError": "continueRegularOutput"` on every external call
- FMP: `apikey=YOUR_FMP_KEY` in URL (their API design, not our choice)
- OpenRouter: `Authorization: Bearer YOUR_OPENROUTER_KEY` in header

## Postgres Nodes

- `"operation": "executeQuery"` with raw SQL
- Credential: `{ "postgres": { "id": "", "name": "Trading Wizard DB" } }`
- Nodes that might return 0 rows feeding downstream: `"alwaysOutputData": true`

## Positions

- Left-to-right flow, ~240px horizontal spacing
- Parallel branches: ~140px vertical offset

## After Editing

1. JSON valid? (no trailing commas, proper quoting)
2. All node names in `connections` match actual `nodes[].name`?
3. New HTTP nodes have timeout + onError?
4. Code nodes handle empty upstream?
5. Update `docs/BLUEPRINT.md` for the corresponding workflow section
