# Snaply JSON Reference

This file is the authoritative reference for generating valid Snaply tenant config JSON.

**Important:** Never include UUIDs in generated output. The admin panel generates all UUIDs when config is pushed via CLI.

---

## Config Shape (what the CLI pushes)

```json
{
  "schema": { "tables": [], "relations": [] },
  "api_settings": { ... },
  "functions": { "<name>": { ... } },
  "cronjobs": { "<name>": { ... } }
}
```

All sections are optional — push any combination. When generating partial configs, produce only the relevant top-level key(s).

---

## Schema

Defines database tables, columns, and relationships.

```json
{
  "schema": {
    "tables": [
      {
        "name": "posts",
        "exposed": true,
        "columns": [
          { "name": "id", "type": "uuid", "readable": true, "creatable": false, "updatable": false },
          { "name": "title", "type": "varchar", "length": 255, "nullable": false, "readable": true, "creatable": true, "updatable": true },
          { "name": "body", "type": "text", "nullable": true, "readable": true, "creatable": true, "updatable": true },
          { "name": "status", "type": "enum", "enum_values": ["draft", "published", "archived"], "default_value": "draft", "readable": true, "creatable": true, "updatable": true },
          { "name": "view_count", "type": "int", "default_value": "0", "readable": true, "creatable": false, "updatable": false },
          { "name": "price", "type": "decimal", "precision": 10, "scale": 2, "readable": true, "creatable": true, "updatable": true },
          { "name": "created_at", "type": "timestamptz", "default_value": "NOW()", "readable": true, "creatable": false, "updatable": false }
        ]
      }
    ],
    "relations": [
      { "from": { "table": "comments", "column": "post_id" }, "to": { "table": "posts", "column": "id" }, "type": "one-to-many" }
    ]
  }
}
```

### Valid Column Types

| Type | PostgreSQL | Notes |
|------|-----------|-------|
| `primary` | `UUID PRIMARY KEY DEFAULT gen_random_uuid()` | Auto-generated PK |
| `uuid` | `UUID` | |
| `int` | `INTEGER` | |
| `bigint` | `BIGINT` | |
| `smallint` | `SMALLINT` | |
| `bigserial` | `BIGSERIAL` | Auto-incrementing |
| `decimal` | `DECIMAL(precision,scale)` | Use `precision` and `scale` |
| `numeric` | `NUMERIC(precision,scale)` | Use `precision` and `scale` |
| `real` | `REAL` | 4-byte float |
| `double` | `DOUBLE PRECISION` | 8-byte float |
| `varchar` | `VARCHAR(length)` | Use `length` (default 255) |
| `char` | `CHAR(length)` | Fixed-width |
| `text` | `TEXT` | Unlimited length |
| `boolean` | `BOOLEAN` | |
| `timestamp` | `TIMESTAMP` | Wall-clock, no timezone |
| `timestamptz` | `TIMESTAMPTZ` | **Recommended** for events |
| `date` | `DATE` | |
| `time` | `TIME` | |
| `interval` | `INTERVAL` | |
| `json` | `JSON` | |
| `jsonb` | `JSONB` | Indexed, preferred |
| `bytea` | `BYTEA` | Binary |
| `enum` | `CREATE TYPE ... AS ENUM` | Requires `enum_values` |

### Column Properties

| Property | Type | Required | Notes |
|----------|------|----------|-------|
| `name` | string | Yes | Max 63 chars |
| `type` | string | Yes | One of the valid types above |
| `nullable` | boolean | No | Default: false |
| `default_value` | string | No | Literal or SQL expression (`NOW()`, `GEN_RANDOM_UUID()`, `CURRENT_TIMESTAMP`) |
| `unique` | boolean | No | Default: false |
| `enum_values` | string[] | If type=enum | 1-100 values, max 63 chars each |
| `length` | integer | No | For varchar/char (1-10000) |
| `precision` | integer | No | For decimal/numeric (1-38) |
| `scale` | integer | No | For decimal/numeric (0-38) |
| `readable` | boolean | No | Column visible in API queries |
| `creatable` | boolean | No | Column settable on create |
| `updatable` | boolean | No | Column settable on update |

### Relation Types

| Type | Meaning |
|------|---------|
| `one-to-one` | Single record relationship |
| `one-to-many` | Parent has many children |
| `many-to-many` | Join table relationship |

---

## api_settings

```json
{
  "api_settings": {
    "timezone": "Europe/Budapest",
    "tables": {
      "posts": {
        "query":  { "enabled": true, "auth": true, "row_policy": "user_id = $auth.id" },
        "create": { "enabled": true, "auth": true },
        "update": { "enabled": true, "auth": true, "row_policy": "user_id = $auth.id" },
        "delete": { "enabled": true, "auth": true, "row_policy": "user_id = $auth.id" }
      }
    },
    "auth_endpoints": { "register": true, "login": true, "refresh": true },
    "crud_endpoints":  { "query": true, "create": true, "update": true, "delete": true },
    "env": ["STRIPE_API_KEY", "WEBHOOK_SECRET"],
    "email": {
      "smtp": {
        "enabled": true,
        "provider": "mailpit",
        "host": "localhost",
        "port": 1025,
        "username": "",
        "password": "",
        "encryption": "none",
        "from_email": "no-reply@example.com",
        "from_name": "My App"
      },
      "templates": {
        "welcome": {
          "subject": "Welcome, {{.Name}}!",
          "html": "<h1>Hello {{.Name}}</h1><p>Thanks for joining.</p>",
          "text": "Hello {{.Name}}, thanks for joining."
        }
      }
    }
  }
}
```

**`tables`**: Use table **names** as keys (not UUIDs). The admin resolves names to UUIDs.

**`timezone`**: IANA timezone string (e.g. `"America/New_York"`, `"Europe/Budapest"`). Controls:
- `{{now}}` value in pipeline steps
- Cron schedule expression interpretation (e.g. `0 22 * * *` fires at 10pm in this timezone)
- How `timestamp without time zone` DB column values are interpreted in pipeline conditions

Does **not** affect `timestamptz` columns — those are always stored as UTC and need no special handling.
Omit or set `"UTC"` for default UTC behaviour.

**`env`**: Array of environment variable **names** (not key-value pairs). Values are set separately. Names must match `/^[a-zA-Z_][a-zA-Z0-9_]*$/`.

**`email.smtp.encryption`**: `none`, `tls`, or `ssl`.

**Row policy variables:** `$auth.id` (user UUID from JWT `sub`), `$auth.email`

---

## Function

```json
{
  "functions": {
    "get-users": {
      "name": "get-users",
      "route": {
        "path": "/fn/get-users",
        "method": "GET",
        "auth_required": true,
        "url_params": { "page": "1", "status": "" }
      },
      "steps": [ ... ]
    }
  }
}
```

- Use the function **name** as the key (admin generates `func_` IDs)
- `method`: `GET | POST | PUT | PATCH | DELETE | PRIVATE`
- `PRIVATE` functions are only callable via `function.call`, not via HTTP
- `url_params`: optional, GET only. Key = param name, value = default (empty = required). Omit for unrestricted params.

---

## Cronjob

```json
{
  "cronjobs": {
    "nightly-cleanup": {
      "name": "nightly-cleanup",
      "schedule": "0 22 * * *",
      "enabled": true,
      "steps": [ ... ]
    }
  }
}
```

Use the cronjob **name** as the key (admin generates UUIDs).

**Cron examples:**

| Expression | Meaning |
|-----------|---------|
| `0 22 * * *` | Every night at 10 PM |
| `*/15 * * * *` | Every 15 minutes |
| `0 0 1 * *` | First of every month at midnight |
| `0 9 * * 1` | Every Monday at 9 AM |

Cron context: only `env.*` and `now` available. No `auth.*`, `request.*`, `query.*`.

---

## Step Types & Fields

### db.select
```json
{
  "type": "db.select",
  "table": "users",
  "columns": ["id", "email", "name"],
  "where": { "id": "{{auth.user_id}}", "status": { "$eq": "active" } },
  "assign": "user"
}
```
- `columns`: optional, omit for all columns
- `where`: optional filter (same operators as CRUD API)
- `assign`: stores result array (or single object if one row matched by id)

### db.insert
```json
{
  "type": "db.insert",
  "table": "orders",
  "data": { "user_id": "{{auth.user_id}}", "total": "{{request.total}}" },
  "assign": "order"
}
```

### db.update
```json
{
  "type": "db.update",
  "table": "users",
  "where": { "id": "{{auth.user_id}}" },
  "data": { "last_login": "{{now}}" },
  "assign": "updated"
}
```

### db.delete
```json
{
  "type": "db.delete",
  "table": "sessions",
  "where": { "expires_at": { "$lt": "{{threshold}}" } },
  "assign": "result"
}
```
Assigns `{ "deleted": N }`.

### if
```json
{
  "type": "if",
  "condition": "{{user.id}} != null",
  "then": [ { "type": "..." } ],
  "else": [ { "type": "..." } ]
}
```

### foreach
```json
{
  "type": "foreach",
  "items": "{{orders}}",
  "as": "order",
  "steps": [ { "type": "..." } ]
}
```
Max 1000 iterations.

### set_variable
```json
{
  "type": "set_variable",
  "assign": "myVar",
  "value": "hello"
}
```

### transform.sum
```json
{ "type": "transform.sum", "input": "orders.total", "assign": "totalSpent" }
```

### transform.count
```json
{ "type": "transform.count", "input": "orders", "assign": "orderCount" }
```

### transform.map
```json
{ "type": "transform.map", "input": "orders", "field": "id", "assign": "orderIds" }
```
Or multi-field:
```json
{ "type": "transform.map", "input": "orders", "fields": { "id": "id", "amount": "total" }, "assign": "summary" }
```

### transform.filter
```json
{ "type": "transform.filter", "input": "orders", "condition": "{{item.status}} == \"active\"", "assign": "activeOrders" }
```

### transform.math
```json
{ "type": "transform.math", "expression": "{{price}} * {{quantity}} * 1.2", "assign": "totalWithTax" }
```

### transform.datetime
```json
{
  "type": "transform.datetime",
  "operation": "subtract",
  "value": 30,
  "unit": "days",
  "input": "{{now}}",
  "assign": "threshold"
}
```
- `operation`: `add | subtract`
- `unit`: `seconds | minutes | hours | days`
- `input`: defaults to `{{now}}` if omitted

### response
```json
{ "type": "response", "status_code": 200, "data": { "user": "{{user}}", "total": "{{totalSpent}}" } }
```
No-op in cronjob context.

### function.call
```json
{
  "type": "function.call",
  "function_id": "calculate_discount",
  "args": { "user_id": "{{user.id}}", "amount": "{{totalSpent}}" },
  "assign": "discount"
}
```
Called function receives args as `{{args.user_id}}`, `{{args.amount}}`. Max recursion depth: 10.

### http.request
```json
{
  "type": "http.request",
  "url": "https://api.stripe.com/v1/charges",
  "method": "POST",
  "headers": { "Authorization": "Bearer {{env.stripe_api_key}}" },
  "body": { "amount": "{{totalCents}}", "currency": "usd" },
  "assign": "charge"
}
```

### email.send (raw mode)
```json
{
  "type": "email.send",
  "mode": "raw",
  "to": ["{{user.email}}"],
  "subject": "Your order #{{order.id}} is confirmed",
  "html": "<h1>Hi {{user.name}}</h1><p>Order total: {{order.total}}</p>",
  "text": "Hi {{user.name}}, order total: {{order.total}}",
  "reply_to": "support@example.com",
  "assign": "emailResult"
}
```

### email.send (template mode)
```json
{
  "type": "email.send",
  "mode": "template",
  "to": ["{{user.email}}"],
  "template_id": "welcome",
  "data": { "Name": "{{user.name}}" },
  "assign": "emailResult"
}
```
- Template uses Go `html/template` syntax: `{{.Name}}`, `{{.Amount}}`, etc.
- `subject` overrides template subject if set
- Templates defined in `api_settings.email.templates`

**email.send assign result:**
```json
{ "status": "sent", "mode": "raw", "provider": "mailpit", "from": "Snaply <no-reply@...>", "to": [...], "accepted": 1 }
```

---

## Pipeline Context

| Key | Available in functions | Available in cronjobs |
|-----|----------------------|----------------------|
| `now` | Yes (in tenant timezone) | Yes (in tenant timezone) |
| `auth.user_id` | Yes (auth_required: true) | No |
| `auth.email` | Yes (auth_required: true) | No |
| `request.*` | Yes POST/PUT/PATCH | No |
| `query.*` | Yes | No |
| `env.*` | Yes | Yes |

---

## Timestamp Column Types

Use the right PostgreSQL column type for datetime data:

| Column type | Stores | Use for |
|-------------|--------|---------|
| `timestamptz` | UTC (converts on read) | Any absolute moment — orders, payments, expiry, bookings |
| `timestamp` | Bare wall-clock, no tz | Recurring times — opening hours, shift start times |
| `date` | Calendar date only | Delivery date, birthday |
| `time` | Time of day only | Daily open/close hours |

**Default recommendation: use `timestamptz` for all event timestamps.**

- `timestamptz` works correctly for single-timezone and multi-timezone apps alike
- `timestamp` + `api_settings.timezone` works only when all users are in the same timezone
- If you use `timestamp` columns, always set `api_settings.timezone` — otherwise pipeline comparisons like `{{item.expires_at}} < {{now}}` will be off by the UTC offset

---

## Variable Syntax Rules

| Pattern | Result |
|---------|--------|
| `"{{user.id}}"` | Resolved value, type-preserved (UUID stays UUID, int stays int) |
| `"Hello {{user.name}}"` | String concatenation |
| `"{{orders.0.total}}"` | Array index 0 |
| `"{{orders.total}}"` | Collect `total` field from all items in array |

---

## Filter Operators (db steps where clause)

| Operator | SQL |
|----------|-----|
| `{ "col": "val" }` | `col = val` |
| `{ "col": { "$eq": "val" } }` | `col = val` |
| `{ "col": { "$ne": "val" } }` | `col != val` |
| `{ "col": { "$gt": 100 } }` | `col > 100` |
| `{ "col": { "$gte": 100 } }` | `col >= 100` |
| `{ "col": { "$lt": 50 } }` | `col < 50` |
| `{ "col": { "$lte": 50 } }` | `col <= 50` |
| `{ "col": { "$in": ["a","b"] } }` | `col IN ('a','b')` |
| `{ "col": { "$like": "%foo%" } }` | `col LIKE '%foo%'` |
| `{ "col": { "$null": true } }` | `col IS NULL` |
| `{ "col": { "$null": false } }` | `col IS NOT NULL` |

---

## Condition Expressions (if / transform.filter)

| Pattern | Meaning |
|---------|---------|
| `{{path}}` | Truthy (not nil, not 0, not "", not false) |
| `{{path}} == "value"` | Equality |
| `{{path}} != "value"` | Inequality |
| `{{path}} > 0` | Numeric comparison |
| `{{path}} == null` | Nil check |
| `{{a}} == {{b}}` | Compare two refs |

---

## Common Examples

### Blog schema with functions
```json
{
  "schema": {
    "tables": [
      {
        "name": "posts",
        "columns": [
          { "name": "id", "type": "uuid", "readable": true, "creatable": false, "updatable": false },
          { "name": "title", "type": "varchar", "length": 255, "readable": true, "creatable": true, "updatable": true },
          { "name": "body", "type": "text", "nullable": true, "readable": true, "creatable": true, "updatable": true },
          { "name": "author_id", "type": "uuid", "readable": true, "creatable": true, "updatable": false },
          { "name": "created_at", "type": "timestamptz", "default_value": "NOW()", "readable": true, "creatable": false, "updatable": false }
        ]
      }
    ]
  },
  "functions": {
    "get-posts": {
      "name": "get-posts",
      "route": { "path": "/fn/get-posts", "method": "GET" },
      "steps": [
        { "type": "db.select", "table": "posts", "assign": "posts" },
        { "type": "response", "data": { "posts": "{{posts}}" } }
      ]
    }
  },
  "api_settings": {
    "tables": {
      "posts": {
        "query": { "enabled": true, "auth": false },
        "create": { "enabled": true, "auth": true },
        "update": { "enabled": true, "auth": true, "row_policy": "author_id = $auth.id" },
        "delete": { "enabled": true, "auth": true, "row_policy": "author_id = $auth.id" }
      }
    }
  }
}
```

### Welcome email on register
```json
{
  "functions": {
    "welcome-email": {
      "name": "welcome-email",
      "route": { "path": "/fn/welcome", "method": "POST", "auth_required": false },
      "steps": [
        { "type": "db.select", "table": "users", "where": { "email": "{{request.email}}" }, "assign": "user" },
        {
          "type": "email.send",
          "mode": "template",
          "to": ["{{user.email}}"],
          "template_id": "welcome",
          "data": { "Name": "{{user.name}}" }
        },
        { "type": "response", "data": { "status": "ok" } }
      ]
    }
  }
}
```

### Nightly cleanup
```json
{
  "cronjobs": {
    "nightly-session-cleanup": {
      "name": "nightly-session-cleanup",
      "schedule": "0 2 * * *",
      "enabled": true,
      "steps": [
        { "type": "transform.datetime", "operation": "subtract", "value": 30, "unit": "days", "assign": "threshold" },
        { "type": "db.delete", "table": "sessions", "where": { "expires_at": { "$lt": "{{threshold}}" } }, "assign": "result" }
      ]
    }
  }
}
```

---

## CLI Usage

### Read current tenant config

```bash
# Output current config as JSON (read-only, no side effects)
snaply show --tenant <uuid>

# Filter with jq
snaply show --tenant <uuid> | jq '.schema'
```

### Push config to admin

```bash
# From file
snaply push --tenant <uuid> --file config.json

# From stdin (pipe from agent or tool)
echo '{"schema":{...}}' | snaply push --tenant <uuid>
```

### Full agent workflow

```bash
# 1. Authenticate (one-time)
snaply login

# 2. Read current config (check existing tables, settings, etc.)
snaply show --tenant <uuid>

# 3. Generate config JSON (this skill)
# 4. Push to admin
snaply push --tenant <uuid> --file config.json

# 5. Sync to local environment (provisions DB, writes Redis)
snaply pull --tenant <uuid>
```
