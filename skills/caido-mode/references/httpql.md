# HTTPQL reference

Caido's query language, used by `search` and by filter presets. Read this when a query needs
a field or operator the common ones in `SKILL.md` do not cover.

**String values are quoted, integers are not.**

**There is no `NOT` operator.** Use the negated variant: `ne`, `ncont`, `nlike`, `nregex`.
`NOT req.path.cont:"/admin"` is a syntax error; `req.path.ncont:"/admin"` is the query.

## Namespaces and fields

| Namespace | Field | Type | Description |
|-----------|-------|------|-------------|
| `req` | `ext` | string | File extension, including the `.` |
| `req` | `host` | string | Hostname |
| `req` | `method` | string | HTTP method, uppercase |
| `req` | `path` | string | URL path |
| `req` | `query` | string | Query string |
| `req` | `raw` | string | Full raw request |
| `req` | `port` | int | Port |
| `req` | `len` | int | Request body length |
| `req` | `created_at` | date | Creation timestamp |
| `req` | `tls` | bool | Is HTTPS |
| `resp` | `raw` | string | Full raw response |
| `resp` | `code` | int | Status code |
| `resp` | `len` | int | Response body length |
| `resp` | `roundtrip` | int | Roundtrip time in ms |
| `row` | `id` | int | Request id |
| `source` | — | special | `"intercept"`, `"replay"`, `"automate"`, `"workflow"` |
| `preset` | — | special | A saved filter preset, by name or alias |

## Operators

- **String:** `eq`, `ne`, `cont`, `ncont`, `like`, `nlike`, `regex`, `nregex`
- **Integer:** `eq`, `ne`, `gt`, `gte`, `lt`, `lte`
- **Boolean:** `eq`, `ne`
- **Logical:** `AND`, `OR`, parentheses for grouping

## Examples

```httpql
# POST requests that succeeded
req.method.eq:"POST" AND resp.code.eq:200

# API traffic
req.host.cont:"api" OR req.path.cont:"/api/"

# A bare string searches both request and response
"password" OR "secret" OR "api_key"

# Client errors only
resp.code.gte:400 AND resp.code.lt:500

# Large responses, a cheap lead on data exposure
resp.len.gt:100000

# Slow endpoints
resp.roundtrip.gt:5000

# Auth surfaces
req.path.regex:"/(login|auth|signin|oauth)/"

# Only traffic this client generated
source:"replay" OR source:"automate"

# Since a date
req.created_at.gt:"2026-01-01T00:00:00Z"

# Exclusions, since NOT does not exist
req.path.ncont:"/static"
req.method.ne:"OPTIONS"
req.path.ncont:"/health" AND req.path.ncont:"/metrics"

# A saved preset
preset:"API 4xx"
```

Scope filtering is not part of HTTPQL: pass `search --scope <id-or-name>` instead of
enumerating hosts in the query.
