---
name: caido-mode
description: Dependency-free Caido CLI integration for Codex and Claude. Search HTTP history, replay/edit requests, manage scopes/filters/environments, create findings, export curl commands, byte-safe downloads, and control intercept through a pinned caido-headless-client submodule.
tags: [worker]
---

# Caido Mode Skill

## Overview

Full-coverage Caido skill backed by a pinned dependency-free client submodule. The client uses Node's built-in `fetch` and `WebSocket`. Covers:

- **HTTP History** - Search, retrieve, replay, edit requests with HTTPQL
- **Replay & Sessions** - Sessions, collections, entries, fuzzing
- **Scopes** - Create and manage testing scopes (allowlist/denylist patterns)
- **Filter Presets** - Save and reuse HTTPQL filter presets
- **Environments** - Store test variables (victim IDs, tokens, etc.)
- **Findings** - Create, list, update security findings
- **Tasks** - Monitor and cancel background tasks
- **Projects** - Switch between testing projects
- **Hosted Files** - Manage files served by Caido
- **Intercept** - Enable/disable request interception programmatically
- **Plugins** - List installed plugins
- **Export** - Convert requests to curl commands for PoCs
- **Health** - Check Caido instance status

All traffic goes through Caido, so it appears in the UI for further analysis.

### Why This Model?

**Cookies and auth tokens can be huge** - session cookies, JWTs, CSRF tokens can easily be 1-2KB. Rather than manually copy-pasting:

1. **Find an organic request** in Caido's HTTP history that already has valid auth
2. **Use `edit` to modify just what you need** (path, method, body) while keeping all auth headers intact
3. **Send it** - response comes back with full context preserved

## Authentication Setup

### Setup (One-Time)

1. Open [Dashboard -> Developer -> Personal Access Tokens](https://docs.caido.io/dashboard/guides/create_pat.html)
2. Create a new token
3. Resolve the client path relative to this skill directory, then run setup:

```bash
# From this repository root
export CAIDO_CLIENT="$PWD/skills/caido-mode/vendor/caido-headless-client/caido-client.mjs"

# From an installed skill directory such as ~/.codex/skills/caido-mode
export CAIDO_CLIENT="$PWD/vendor/caido-headless-client/caido-client.mjs"

node "$CAIDO_CLIENT" setup <your-pat> --no-save-pat

# Non-default Caido instance
node "$CAIDO_CLIENT" setup <pat> http://192.168.1.100:8080 --no-save-pat

# Or set env var instead
export CAIDO_PAT=caido_xxxxx
```

The `setup` command starts Caido's device-code flow, auto-approves it with the PAT, then saves the cached access token and refresh token to `~/.claude/config/secrets.json`. For compatibility it stores the PAT too; add `--no-save-pat` to avoid persisting the PAT.

### Check Status

```bash
node "$CAIDO_CLIENT" auth-status
```

### How Auth Works

The CLI uses Caido's device code flow directly. It starts the local auth flow through Caido GraphQL, uses the PAT against Caido Cloud to approve the device code, then receives an access token + refresh token through a local GraphQL WebSocket subscription.

Auth resolution: `CAIDO_ACCESS_TOKEN` env var -> valid cached access token -> refresh token -> `CAIDO_PAT` env var -> `secrets.json` PAT -> error with setup instructions

## CLI Tool

Use the `caido-headless-client` submodule located under `vendor/caido-headless-client`. In this repository, run `node skills/caido-mode/vendor/caido-headless-client/caido-client.mjs ...`; from an installed Codex or Claude skill directory, run `node "$CAIDO_CLIENT" ...`. All commands output JSON.

If `$CAIDO_CLIENT` is not already set, resolve it before running examples:

```bash
export CAIDO_CLIENT="$PWD/vendor/caido-headless-client/caido-client.mjs"
```

---

## HTTP History & Testing Commands

### search - Search HTTP history with HTTPQL

```bash
node "$CAIDO_CLIENT" search 'req.method.eq:"POST" AND resp.code.eq:200'
node "$CAIDO_CLIENT" search 'req.host.cont:"api"' --limit 50
node "$CAIDO_CLIENT" search 'req.host.cont:"api"' --desc --limit 10
node "$CAIDO_CLIENT" search 'req.path.cont:"/admin"' --ids-only
node "$CAIDO_CLIENT" search 'resp.raw.cont:"password"' --after <cursor>
```

### recent - Get recent requests

```bash
node "$CAIDO_CLIENT" recent
node "$CAIDO_CLIENT" recent --limit 50
```

### get / get-response - Retrieve full details

```bash
node "$CAIDO_CLIENT" get <request-id>
node "$CAIDO_CLIENT" get <request-id> --headers-only
node "$CAIDO_CLIENT" get-response <request-id>
node "$CAIDO_CLIENT" get-response <request-id> --compact
node "$CAIDO_CLIENT" download <request-id> --out response-body.bin
node "$CAIDO_CLIENT" download <request-id> --response --raw --out response.http
node "$CAIDO_CLIENT" download <request-id> --request --raw --out request.http
```

`download` is byte-safe and should be used when the body may be binary. By default it saves the response body only. Add `--raw` to save the full raw HTTP message including headers, `--request` to save the request instead of the response, and `--force` to overwrite an existing file.

### edit - Edit and replay (KEY FEATURE)

Modifies an existing request while preserving all cookies/auth headers:

```bash
# Change path (IDOR testing)
node "$CAIDO_CLIENT" edit <id> --path /api/user/999

# Change method and add body
node "$CAIDO_CLIENT" edit <id> --method POST --body '{"admin":true}'

# Add/remove headers
node "$CAIDO_CLIENT" edit <id> --set-header "X-Forwarded-For: 127.0.0.1"
node "$CAIDO_CLIENT" edit <id> --remove-header "X-CSRF-Token"

# Find/replace text anywhere in request
node "$CAIDO_CLIENT" edit <id> --replace "user123:::user456"

# Combine multiple edits
node "$CAIDO_CLIENT" edit <id> --method PUT --path /api/admin --body '{"role":"admin"}' --compact

# Reuse an existing replay tab/session for repeated probes
node "$CAIDO_CLIENT" edit <id> --path /api/user/1001 --session <session-id> --compact
```

| Option | Description |
|--------|-------------|
| `--method <METHOD>` | Change HTTP method |
| `--path <path>` | Change request path |
| `--set-header <Name: Value>` | Add or replace a header (repeatable) |
| `--remove-header <Name>` | Remove a header (repeatable) |
| `--body <content>` | Set request body (auto-updates Content-Length) |
| `--replace <from>:::<to>` | Find/replace text anywhere in request (repeatable) |
| `--session <id>` | Reuse an existing replay session instead of creating a new tab |
| `--collection <id>` | Put a newly created replay session in a collection |
| `--sni <host>` | Override TLS SNI |
| `--connect-host <host>` | Connect to a different host while preserving the HTTP request |
| `--connect-port <port>` | Connect to a different port |
| `--connect-tls` / `--connect-no-tls` | Force TLS/plaintext for the connection |

### replay / send-raw - Send requests

```bash
# Replay as-is
node "$CAIDO_CLIENT" replay <request-id>

# Replay with custom raw
node "$CAIDO_CLIENT" replay <id> --raw "GET /modified HTTP/1.1\r\nHost: example.com\r\n\r\n"

# Send completely custom request
node "$CAIDO_CLIENT" send-raw --host example.com --port 443 --tls --raw "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n"
node "$CAIDO_CLIENT" send-raw --host example.com --raw @request.txt --name "G /"
cat request.txt | node "$CAIDO_CLIENT" send-raw --host example.com --raw -

# Connect elsewhere while preserving the request Host/SNI you need
node "$CAIDO_CLIENT" replay <id> --connect-host 10.0.0.5 --connect-port 8443 --sni example.com
```

`--raw` accepts a string with `\r\n` escapes, `@file` to read from disk, or `-` to read from stdin.

### export-curl - Convert to curl for PoCs

```bash
node "$CAIDO_CLIENT" export-curl <request-id>
```

Outputs a ready-to-use curl command with all headers and body.

---

## Replay Tab Lookup

Use these when a Caido replay tab is already open and you want to work from its active entry directly.

```bash
node "$CAIDO_CLIENT" get-session <session-id-or-name> --compact
node "$CAIDO_CLIENT" replay-entries <session-id-or-name> --limit 20
node "$CAIDO_CLIENT" replay-entries <session-id-or-name> --raw --compact
node "$CAIDO_CLIENT" edit-session <session-id-or-name> --body '{"test":true}' --compact
```

`session-entries` is accepted as an alias for `replay-entries`.

---

## Replay Sessions & Collections

### Sessions

```bash
# Create replay session from an existing request
node "$CAIDO_CLIENT" create-session <request-id>
node "$CAIDO_CLIENT" create-session <request-id> --collection <collection-id>

# ALWAYS rename sessions for easy identification in Caido UI
node "$CAIDO_CLIENT" rename-session <session-id> "idor-user-profile"

# List all replay sessions
node "$CAIDO_CLIENT" replay-sessions
node "$CAIDO_CLIENT" replay-sessions --limit 50

# Move sessions between collections
node "$CAIDO_CLIENT" move-session <session-id> <collection-id>

# Delete replay sessions
node "$CAIDO_CLIENT" delete-sessions <session-id-1>,<session-id-2>
```

### Collections

Organize replay sessions into collections:

```bash
# List replay collections
node "$CAIDO_CLIENT" replay-collections
node "$CAIDO_CLIENT" replay-collections --limit 50

# Create a collection
node "$CAIDO_CLIENT" create-collection "IDOR Testing"

# Rename a collection
node "$CAIDO_CLIENT" rename-collection <collection-id> "Auth Bypass Tests"

# Delete a collection
node "$CAIDO_CLIENT" delete-collection <collection-id>
```

### Fuzzing

```bash
# Create automate session for fuzzing
node "$CAIDO_CLIENT" create-automate-session <request-id>

# Start fuzzing (configure payloads and markers in Caido UI first)
node "$CAIDO_CLIENT" fuzz <session-id>
```

---

## Scope Management

Define what's in scope for your testing. Uses glob patterns.

```bash
# List all scopes
node "$CAIDO_CLIENT" scopes

# Create scope with allowlist and denylist
node "$CAIDO_CLIENT" create-scope "Target Corp" --allow "*.target.com,*.target.io" --deny "*.cdn.target.com"

# Update scope
node "$CAIDO_CLIENT" update-scope <scope-id> --allow "*.target.com,*.api.target.com"

# Delete scope
node "$CAIDO_CLIENT" delete-scope <scope-id>
```

**Glob patterns:** `*.example.com` matches any subdomain of example.com.

---

## Filter Presets

Save frequently used HTTPQL queries as named presets.

```bash
# List saved filters
node "$CAIDO_CLIENT" filters

# Create filter preset
node "$CAIDO_CLIENT" create-filter "API Errors" --query 'req.path.cont:"/api/" AND resp.code.gte:400'
node "$CAIDO_CLIENT" create-filter "Auth Endpoints" --query 'req.path.regex:"/(login|auth|oauth)/"' --alias "auth"

# Update filter
node "$CAIDO_CLIENT" update-filter <filter-id> --query 'req.path.cont:"/api/" AND resp.code.gte:500'

# Delete filter
node "$CAIDO_CLIENT" delete-filter <filter-id>
```

---

## Environment Variables

Store testing variables that persist across sessions. Great for IDOR testing with multiple user IDs.

```bash
# List environments
node "$CAIDO_CLIENT" envs

# Create environment
node "$CAIDO_CLIENT" create-env "IDOR-Test"

# Set variables
node "$CAIDO_CLIENT" env-set <env-id> victim_user_id "user_456"
node "$CAIDO_CLIENT" env-set <env-id> attacker_token "eyJhbG..."

# Select active environment
node "$CAIDO_CLIENT" select-env <env-id>

# Deselect environment
node "$CAIDO_CLIENT" select-env

# Delete environment
node "$CAIDO_CLIENT" delete-env <env-id>
```

---

## Findings

Create, list, and update security findings. Shows up in Caido's Findings tab.

```bash
# List all findings
node "$CAIDO_CLIENT" findings
node "$CAIDO_CLIENT" findings --limit 50

# Get a specific finding
node "$CAIDO_CLIENT" get-finding <finding-id>

# Create finding linked to a request
node "$CAIDO_CLIENT" create-finding <request-id> \
  --title "IDOR in user profile endpoint" \
  --description "Can access other users' profiles by changing ID parameter" \
  --reporter "rez0"

# With deduplication key (prevents duplicates)
node "$CAIDO_CLIENT" create-finding <request-id> \
  --title "Auth bypass on /admin" \
  --dedupe-key "admin-auth-bypass"

# Update finding
node "$CAIDO_CLIENT" update-finding <finding-id> \
  --title "Updated title" \
  --description "Updated description"
```

---

## Tasks

Monitor and cancel background tasks (imports, exports, etc.).

```bash
# List all tasks
node "$CAIDO_CLIENT" tasks

# Cancel a running task
node "$CAIDO_CLIENT" cancel-task <task-id>
```

---

## Project Management

```bash
# List all projects
node "$CAIDO_CLIENT" projects

# Switch active project
node "$CAIDO_CLIENT" select-project <project-id>
```

---

## Hosted Files

```bash
# List hosted files
node "$CAIDO_CLIENT" hosted-files

# Delete hosted file
node "$CAIDO_CLIENT" delete-hosted-file <file-id>
```

---

## Intercept Control

```bash
# Check intercept status
node "$CAIDO_CLIENT" intercept-status

# Enable/disable interception
node "$CAIDO_CLIENT" intercept-enable
node "$CAIDO_CLIENT" intercept-disable
```

---

## Info, Health & Plugins

```bash
# Current user info
node "$CAIDO_CLIENT" viewer

# List installed plugins
node "$CAIDO_CLIENT" plugins

# Check Caido instance health (version, ready state)
node "$CAIDO_CLIENT" health
```

---

## Output Control

Works with `get`, `get-response`, `replay`, `edit`, `send-raw`:

| Flag | Description |
|------|-------------|
| `--max-body <n>` | Max response body lines (default: 200, 0=unlimited) |
| `--max-body-chars <n>` | Max body chars (default: 5000, 0=unlimited) |
| `--no-request` | Skip request raw in output |
| `--headers-only` | Only HTTP headers, no body |
| `--compact` | Shorthand: `--no-request --max-body 50 --max-body-chars 5000` |

---

## HTTPQL Reference

Caido's query language for searching HTTP history.

**CRITICAL**: String values MUST be quoted. Integer values are NOT quoted.

**CRITICAL**: HTTPQL has NO `NOT` operator. Never write `NOT expr`. Use the negated operator variant instead:
- `ncont` (not contains), `nlike` (not like), `nregex` (not regex), `ne` (not equals)
- Wrong: `NOT req.path.cont:"/admin"`
- Right: `req.path.ncont:"/admin"`

### Namespaces and Fields

| Namespace | Field | Type | Description |
|-----------|-------|------|-------------|
| `req` | `ext` | string | File extension (includes `.`) |
| `req` | `host` | string | Hostname |
| `req` | `method` | string | HTTP method (uppercase) |
| `req` | `path` | string | URL path |
| `req` | `query` | string | Query string |
| `req` | `raw` | string | Full raw request |
| `req` | `port` | int | Port number |
| `req` | `len` | int | Request body length |
| `req` | `created_at` | date | Creation timestamp |
| `req` | `tls` | bool | Is HTTPS |
| `resp` | `raw` | string | Full raw response |
| `resp` | `code` | int | Status code |
| `resp` | `len` | int | Response body length |
| `resp` | `roundtrip` | int | Roundtrip time (ms) |
| `row` | `id` | int | Request ID |
| `source` | - | special | `"intercept"`, `"replay"`, `"automate"`, `"workflow"` |
| `preset` | - | special | Filter preset reference |

### Operators

**String:** `eq`, `ne`, `cont`, `ncont`, `like`, `nlike`, `regex`, `nregex`
**Integer:** `eq`, `ne`, `gt`, `gte`, `lt`, `lte`
**Boolean:** `eq`, `ne`
**Logical:** `AND`, `OR`, parentheses for grouping

### Example Queries

```httpql
# POST requests with 200 responses
req.method.eq:"POST" AND resp.code.eq:200

# API requests
req.host.cont:"api" OR req.path.cont:"/api/"

# Standalone string searches both req and resp
"password" OR "secret" OR "api_key"

# Error responses
resp.code.gte:400 AND resp.code.lt:500

# Large responses (potential data exposure)
resp.len.gt:100000

# Slow endpoints
resp.roundtrip.gt:5000

# Auth endpoints by regex
req.path.regex:"/(login|auth|signin|oauth)/"

# Replay/automate traffic only
source:"replay" OR source:"automate"

# Date filtering
req.created_at.gt:"2024-01-01T00:00:00Z"

# Exclude paths (use ncont, NOT doesn't exist)
req.path.ncont:"/static"

# Not equal
req.method.ne:"OPTIONS"

# Combine negations
req.path.ncont:"/health" AND req.path.ncont:"/metrics"
```

---

## Dependency-Free Architecture

This CLI is a single Node ESM entrypoint with no npm dependencies:

```
vendor/caido-headless-client/caido-client.mjs  # Auth flow, GraphQL/REST transport, command dispatch, output formatting
vendor/caido-headless-client/package.json      # Script metadata only; no dependencies
vendor/caido-headless-client/README.md
SKILL.md
```

### API Coverage

All commands use direct GraphQL/REST operations against Caido:

| API Area | Commands |
|-----------|----------|
| Request GraphQL | search, recent, get, get-response, download, export-curl |
| Replay GraphQL + task subscription | replay, send-raw, edit, sessions, collections |
| Findings GraphQL | findings, get-finding, create-finding, update-finding |
| Management GraphQL | scopes, filters, environments, projects, hosted files, tasks |
| Intercept/plugin/automate GraphQL | intercept, plugins, create-automate-session, fuzz |
| REST `/health` | health |

---

## Workflow Examples

### 1. IDOR Testing (Primary Pattern)

```bash
# Find authenticated request
node "$CAIDO_CLIENT" search 'req.path.cont:"/api/user"' --limit 10

# Create scope
node "$CAIDO_CLIENT" create-scope "IDOR-Test" --allow "*.target.com"

# Create environment for test data
node "$CAIDO_CLIENT" create-env "IDOR-Test"
node "$CAIDO_CLIENT" env-set <env-id> victim_id "user_999"

# Test IDOR by changing user ID
node "$CAIDO_CLIENT" edit <request-id> --path /api/user/999

# Mark as finding if it works
node "$CAIDO_CLIENT" create-finding <request-id> --title "IDOR on /api/user/:id"

# Export curl for PoC
node "$CAIDO_CLIENT" export-curl <request-id>
```

### 2. Privilege Escalation Testing

```bash
node "$CAIDO_CLIENT" search 'req.path.cont:"/admin"' --limit 10
node "$CAIDO_CLIENT" edit <id> --path /api/admin/users --method GET
node "$CAIDO_CLIENT" edit <id> --method POST --body '{"role":"admin"}'
```

### 3. Header Bypass Testing

```bash
node "$CAIDO_CLIENT" edit <id> --set-header "X-Forwarded-For: 127.0.0.1"
node "$CAIDO_CLIENT" edit <id> --set-header "X-Original-URL: /admin"
node "$CAIDO_CLIENT" edit <id> --remove-header "X-CSRF-Token"
```

### 4. Fuzzing with Automate

```bash
node "$CAIDO_CLIENT" create-automate-session <request-id>
# Configure payload markers and wordlists in Caido UI
node "$CAIDO_CLIENT" fuzz <session-id>
```

### 5. Filter + Analyze Pattern

```bash
# Save useful filters
node "$CAIDO_CLIENT" create-filter "API 4xx" --query 'req.path.cont:"/api/" AND resp.code.gte:400 AND resp.code.lt:500'
node "$CAIDO_CLIENT" create-filter "Large Responses" --query 'resp.len.gt:100000'
node "$CAIDO_CLIENT" create-filter "Sensitive Data" --query '"password" OR "secret" OR "api_key" OR "token"'

# Quick search using preset alias
node "$CAIDO_CLIENT" search 'preset:"API 4xx"' --limit 20
```

---

## Instructions for Claude

1. **PREFER `edit` OVER `replay --raw`** - preserves cookies/auth automatically
2. **Workflow**: Search -> find request with valid auth -> use that ID for all tests via `edit`
3. **Don't dump raw requests into context** - use `--compact` or `--headers-only` when exploring
4. **Always check auth first**: `health` to verify connection, then `recent --limit 1`
5. **ALWAYS NAME REPLAY TABS**: `rename-session <id> "idor-user-profile"`
6. **Create findings** for anything interesting - they show up in Caido's Findings tab
7. **Use `export-curl`** when building PoCs for reports
8. **Create filter presets** for recurring searches to save typing
9. **Use environments** to store test data (victim IDs, tokens, etc.)
10. **Output is JSON** - parse response fields as needed
11. **NEVER use `NOT` in HTTPQL** - it doesn't exist. Use negated operators: `ne`, `ncont`, `nlike`, `nregex`

## Performance & Context Optimization

- `search`/`recent` omit `raw` field (~200 bytes per request, safe for 100+)
- `get` fetches `raw` (~5-20KB per request, fetch only what you need)
- Use `--limit` aggressively (start with 5-10)
- Use `--compact` flag for quick exploration
- Filter server-side with HTTPQL, not client-side

## Error Handling

- **Auth errors**: Run `node "$CAIDO_CLIENT" auth-status` to check, re-setup with `node "$CAIDO_CLIENT" setup <pat>`
- **Connection refused**: Caido not running -> `node "$CAIDO_CLIENT" health`
- **InstanceNotReadyError**: Caido is starting up, wait and retry

## Related Skills

- `caido-plugin-dev` - For building Caido plugins (backend + frontend)
- `spider` - Crawling with Katana (uses Caido as proxy)
- `website-fuzzing` - Remote ffuf fuzzing on hunt6
- `JsAnalyzer` - JS analysis for traffic-discovered files
