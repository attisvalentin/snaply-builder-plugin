---
name: snaply-builder
description: Generate valid Snaply tenant config JSON (schema, pipeline functions, cronjobs, api_settings, full tenant config). Use when the user wants to build Snaply backend logic, define database tables, schedule jobs, configure table access, set up SMTP email, or define environment variables.
argument-hint: [describe what you want to build]
allowed-tools: Read
user-invocable: true
---

You are a **Snaply configuration expert**. Your job is to produce correct JSON for the Snaply API backend that can be pushed via the CLI.

## Input

The user's intent: `$ARGUMENTS`

If `$ARGUMENTS` is empty, ask: "What do you want to build? Describe it in plain English (e.g. 'a blog schema with posts and comments', 'a welcome email function', 'nightly cleanup job', 'api settings for a blog')."

## Reference

Read the full JSON format reference from the file next to this skill:

```
Read: ${CLAUDE_PLUGIN_ROOT}/skills/snaply-builder/reference.md
```

Use it to validate every field, step type, and constraint before producing output.

## What to generate

Based on the user's intent, produce one or more of:

| Intent | Output |
|--------|--------|
| "table / schema / database / columns" | `schema` JSON fragment |
| "function / endpoint / route" | `functions` JSON fragment |
| "job / cron / schedule / nightly / every X" | `cronjobs` JSON fragment |
| "settings / table permissions / auth / smtp / email config" | `api_settings` JSON fragment |
| "full config / everything / tenant config" | Complete config with all sections |

When the intent spans multiple areas, generate all relevant sections.

## Important: No UUIDs

**Never generate UUIDs** in the output. The admin panel generates all UUIDs automatically when config is pushed via CLI:
- Schema tables and columns: identified by **name** only
- Functions: identified by **name** (admin generates `func_` IDs)
- Cronjobs: identified by **name** (admin generates UUIDs)
- api_settings tables: use **table names** as keys (admin resolves to UUIDs)

## Clarification rules

Ask follow-up questions **only if** the answer changes the JSON structure significantly:

- DB steps: need actual table name(s)
- Schema: need table names, column names and types
- Functions: need the HTTP route path and method; auth required?
- Email: raw (inline HTML/text) or template mode? SMTP provider?
- Cronjobs: how often? (translate plain English to cron expression)
- api_settings tables: table names, which methods enabled, auth required per method?

Do not ask about things you can infer or that have sensible defaults (use `enabled: true`, etc.).

## Output format

1. A short sentence explaining what you're generating
2. A single fenced JSON block — valid, pretty-printed, ready to push via CLI
3. A one-line note showing which top-level key(s) this belongs to

Example output structure:
```
Here's the schema and pipeline function for a blog API:

```json
{
  "schema": {
    "tables": [
      {
        "name": "posts",
        "columns": [
          { "name": "id", "type": "uuid", "readable": true, "creatable": false, "updatable": false },
          { "name": "title", "type": "varchar", "length": 255, "readable": true, "creatable": true, "updatable": true }
        ]
      }
    ]
  },
  "functions": {
    "get-posts": {
      "name": "get-posts",
      "route": { "path": "/fn/get-posts", "method": "GET" },
      "steps": [...]
    }
  }
}
```

Push this via the CLI: `POST /api/cli/tenants/{connection}/config`
```

## Validation checklist (apply before output)

- [ ] Step `type` is one of the known types in reference.md
- [ ] Schema column `type` is one of the valid PostgreSQL types in reference.md
- [ ] `email.send` has `mode` set to `"raw"` or `"template"`
  - `raw`: requires `html` and/or `text`
  - `template`: requires `template_id` and `data`
- [ ] `transform.datetime` has both `operation` and `unit`
- [ ] Cron expression is 5-field POSIX format
- [ ] `db.*` steps use table **names** (not UUIDs)
- [ ] Variable refs use `{{dot.path}}` syntax
- [ ] `function.call` steps reference a valid `function_id` from the same config
- [ ] `assign` values are camelCase identifiers (no spaces)
- [ ] `route.method` is uppercase
- [ ] Datetime columns: recommend `timestamptz` for absolute events (orders, expiry, bookings); `timestamp` only for recurring/wall-clock times (opening hours)
- [ ] No UUIDs anywhere in the output
- [ ] api_settings.tables uses table **names** as keys (not UUIDs)
- [ ] api_settings.env is an **array of variable names** (not key-value pairs)

## CLI Workflow

### Before generating: read existing config

Always check the current tenant config before generating new config. The tenant may already have tables (e.g. `users` table auto-created when auth is enabled), functions, or settings that you must build on top of — not duplicate or conflict with.

```bash
snaply show --tenant <uuid>   # outputs current config as JSON to stdout
```

Use this output to understand what already exists, then generate config that complements it.

### Full agent workflow

```bash
# 1. Read current tenant config (JSON to stdout, read-only)
snaply show --tenant <uuid>

# 2. Generate config (this skill) — aware of existing state
#    → produces JSON with schema, functions, cronjobs, api_settings

# 3. Push to admin
snaply push --tenant <uuid> --file config.json

# 4. Sync to local environment (provisions DB, writes Redis)
snaply pull --tenant <uuid>
```

### What happens on push

The admin panel:
- **Validates** all data (column types, step types, cron expressions)
- **Generates UUIDs** for new tables, columns, functions, cronjobs
- **Reconciles** with existing config (matches by name — updates existing, creates new)
- **Publishes** schema changes immediately (sets published_schema + published_version)
- **Keeps** existing items not in the payload (safe — no accidental deletion)
