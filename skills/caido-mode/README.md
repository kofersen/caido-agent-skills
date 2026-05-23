# Caido Mode

Dependency-free Caido skill for Codex and Claude. It gives agents a small, auditable CLI for searching HTTP history, replaying and editing requests with preserved auth, saving byte-safe bodies, managing scopes/filters/environments/findings, and controlling Caido from the terminal.

The executable client is pinned as a submodule at:

```text
client/caido-client.mjs
```

No `npm install` is required.

## Why

Cookies, JWTs, CSRF tokens, and other auth headers are often too large and fragile to copy by hand. The normal workflow is:

1. Find an organic request in Caido history that already has valid auth.
2. Use `edit` to change only the method, path, headers, or body.
3. Send it through Caido so the request, response, timing, and replay context stay visible in the UI.

## Requirements

- Node.js 24 or newer
- A running Caido instance
- A Caido Personal Access Token for first-time setup

## Setup

From the installed skill directory:

```bash
cd ~/.codex/skills/caido-mode
# Claude Code users: cd ~/.claude/skills/caido-mode
node client/caido-client.mjs setup <pat> <caido-url> --no-save-pat
node client/caido-client.mjs auth-status
```

For a local Caido instance:

```bash
node client/caido-client.mjs setup <pat> http://localhost:8080 --no-save-pat
```

`--no-save-pat` keeps the PAT out of `~/.claude/config/secrets.json`; the client stores cached access and refresh tokens instead.

## What's Covered

| Category | Commands |
|----------|----------|
| HTTP History | `search`, `recent`, `get`, `get-response`, `download`, `export-curl` |
| Edit & Replay | `edit`, `replay`, `send-raw`, `edit-session` |
| Replay Tab Lookup | `get-session`, `replay-entries`, `session-entries` |
| Sessions | `create-session`, `rename-session`, `move-session`, `replay-sessions`, `delete-sessions` |
| Collections | `replay-collections`, `create-collection`, `rename-collection`, `delete-collection` |
| Fuzzing | `create-automate-session`, `fuzz` |
| Scopes | `scopes`, `create-scope`, `update-scope`, `delete-scope` |
| Filter Presets | `filters`, `create-filter`, `update-filter`, `delete-filter` |
| Environments | `envs`, `create-env`, `select-env`, `env-set`, `delete-env` |
| Findings | `findings`, `get-finding`, `create-finding`, `update-finding` |
| Tasks | `tasks`, `cancel-task` |
| Projects | `projects`, `select-project` |
| Hosted Files | `hosted-files`, `delete-hosted-file` |
| Intercept | `intercept-status`, `intercept-enable`, `intercept-disable` |
| Info | `viewer`, `plugins`, `health`, `setup`, `auth-status` |

## Usage

All commands output JSON unless the command naturally returns text, such as `export-curl`.

### Search & Browse

```bash
node client/caido-client.mjs search 'req.method.eq:"POST" AND resp.code.eq:200'
node client/caido-client.mjs search 'req.host.cont:"api"' --limit 50
node client/caido-client.mjs search 'req.path.cont:"/admin"' --ids-only
node client/caido-client.mjs recent --limit 10
node client/caido-client.mjs get <request-id> --compact
node client/caido-client.mjs get-response <request-id> --compact
```

### Byte-Safe Downloads

Use `download` when a response may be binary or when you need exact bytes on disk:

```bash
node client/caido-client.mjs download <request-id> --out response-body.bin
node client/caido-client.mjs download <request-id> --response --raw --out response.http
node client/caido-client.mjs download <request-id> --request --raw --out request.http
```

`get` and `get-response` are JSON/text-oriented. `download` is the byte-safe path.

### Edit & Replay

```bash
node client/caido-client.mjs edit <id> --path /api/user/999 --compact
node client/caido-client.mjs edit <id> --method POST --body '{"role":"admin"}' --compact
node client/caido-client.mjs edit <id> --set-header "X-Forwarded-For: 127.0.0.1"
node client/caido-client.mjs edit <id> --remove-header "X-CSRF-Token"
node client/caido-client.mjs edit <id> --replace "user123:::user456"
node client/caido-client.mjs edit <id> --path /api/user/1001 --session <session-id> --compact
```

`edit`, `replay`, and `send-raw` support connection overrides:

```bash
node client/caido-client.mjs replay <id> --connect-host 10.0.0.5 --connect-port 8443 --sni example.com
```

### Raw Replay

```bash
node client/caido-client.mjs send-raw --host example.com --raw "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n"
node client/caido-client.mjs send-raw --host example.com --raw @request.txt --name "GET /"
cat request.txt | node client/caido-client.mjs send-raw --host example.com --raw -
```

`--raw` accepts an inline string, `@file`, or `-` for stdin.

### Replay Sessions

```bash
node client/caido-client.mjs get-session <session-id-or-name> --compact
node client/caido-client.mjs replay-entries <session-id-or-name> --limit 20
node client/caido-client.mjs edit-session <session-id-or-name> --body '{"test":true}' --compact
node client/caido-client.mjs create-session <request-id>
node client/caido-client.mjs rename-session <session-id> "idor-user-profile"
node client/caido-client.mjs replay-sessions --limit 20
```

### Collections & Fuzzing

```bash
node client/caido-client.mjs replay-collections --limit 20
node client/caido-client.mjs create-collection "IDOR Tests"
node client/caido-client.mjs move-session <session-id> <collection-id>
node client/caido-client.mjs create-automate-session <request-id>
node client/caido-client.mjs fuzz <session-id>
```

Configure payloads and markers in Caido UI before starting `fuzz`.

### Scopes, Filters & Environments

```bash
node client/caido-client.mjs scopes
node client/caido-client.mjs create-scope "Target" --allow "*.example.com"
node client/caido-client.mjs filters
node client/caido-client.mjs create-filter "API Errors" --query 'req.path.cont:"/api/" AND resp.code.gte:400'
node client/caido-client.mjs envs
node client/caido-client.mjs create-env "IDOR-Test"
node client/caido-client.mjs env-set <env-id> victim_id "user_456"
```

### Findings

```bash
node client/caido-client.mjs findings --limit 20
node client/caido-client.mjs create-finding <request-id> \
  --title "IDOR in user profile endpoint" \
  --description "Can access another profile by changing an ID"
node client/caido-client.mjs update-finding <finding-id> --title "Updated title"
```

### Info, Intercept & Health

```bash
node client/caido-client.mjs viewer
node client/caido-client.mjs plugins
node client/caido-client.mjs intercept-status
node client/caido-client.mjs intercept-enable
node client/caido-client.mjs intercept-disable
node client/caido-client.mjs health
```

## Output Control

These flags work with `get`, `get-response`, `replay`, `edit`, and `send-raw`:

| Flag | Description |
|------|-------------|
| `--max-body <n>` | Max response body lines. Use `0` for unlimited. |
| `--max-body-chars <n>` | Max response body characters. Use `0` for unlimited. |
| `--no-request` | Skip request raw output. |
| `--headers-only` | Show only HTTP headers. |
| `--compact` | Shorthand for context-friendly output. |

## HTTPQL Quick Reference

String values must be quoted. Integer values are not quoted.

```httpql
req.method.eq:"POST" AND resp.code.eq:200
req.host.cont:"api"
req.path.regex:"/users/[0-9]+"
resp.code.gte:400
resp.len.gt:100000
"password" OR "secret"
source:"replay"
preset:"My Filter"
```

HTTPQL does not have a `NOT` operator. Use negated operators such as `ne`, `ncont`, `nlike`, and `nregex`.

## Agent Integration

Agents should read `SKILL.md` for the full workflow, command catalog, and operational guidance. This README is the human-facing quick reference.

## Architecture

- Single Node ESM client.
- No npm dependencies, install scripts, or package manager execution.
- Direct GraphQL, GraphQL WebSocket, and REST calls using Node built-ins.
- Exact client revision pinned by git submodule.
- `download` writes raw bytes directly instead of converting through UTF-8 text.

## Security Notes

- Prefer `setup ... --no-save-pat`.
- Do not commit `~/.claude/config/secrets.json` or env files containing tokens.
- Review client submodule diffs before updating the pin.
