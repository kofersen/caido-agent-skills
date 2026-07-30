# Caido Mode

Dependency-free Caido skill for Codex and Claude. It gives agents a small, auditable CLI for searching HTTP history, replaying and editing requests with preserved auth, saving byte-safe bodies, managing scopes/filters/environments/findings, and controlling Caido from the terminal.

The executable client is pinned as a submodule at:

```text
client/caido-client.mjs
```

No `npm install` is required.

## Upstream parity

This skill mirrors Caido's official [`caido/skills`](https://github.com/caido/skills) `caido-mode` (v3.0.1) — same command names, same flags, same output field shapes, and the same `~/.claude/config/secrets.json` path and format, so the two are drop-in interchangeable. Nothing upstream ships is missing here.

Checked 2026-07-30: upstream `caido-mode` is still v3.0.1, untouched since 2026-06-08 and pinned to `@caido/sdk-client` ^0.4.0, and every command it ships is present here. The embedded GraphQL is field-checked against the canonical documents in [`caido/sdk-js`](https://github.com/caido/sdk-js) — `@caido/sdk-client` 0.5.0, generated from `@caido/schema-proxy` 0.57.0 — with no drift, since every document change between 0.4.0 and 0.5.0 was additive. Details in [`client/README.md`](client/README.md#compatibility).

### What differs

| | This skill | Upstream `caido-mode` v3.0.1 |
|---|---|---|
| Implementation | one dependency-free `.mjs` on Node's built-in `fetch` and `WebSocket` | `@caido/sdk-client` + `graphql-tag` + `tsx`, installed from npm |
| Auditability | exact client revision pinned by submodule commit hash | dependency range (`^0.4.0`) resolved at install time |
| Binary bodies | `download` writes raw bytes — `--out`, `--request`/`--response`, `--raw`, `--body-only`, `--force` | no such command; bodies only come back through JSON text |
| Comparing two results | `compare` reports status, length, header deltas and the differing body region, decided on bytes | not available; read both responses and judge |
| Report evidence | `evidence` writes `request.http`, `response.http`, `curl.sh`, `meta.json` at `0600` | assemble by hand from `download` and `export-curl` |
| Repeating one request | `edit --values` sends one per value in a single session, a row each, stopping on backoff | one invocation per value |
| Rate discipline | `--delay` paces sends per host across processes; 429, challenge, 503 and `Retry-After` are reported as `backoff` | no pacing, no backoff signal |
| Scope | `search --scope` filters history by a Caido scope | not exposed |
| Expired access token | refresh token is stored, rotated on the first auth failure, and the call retried | refresh token is never stored; the run exits telling you to re-run `setup <pat>` |
| PAT on disk | `setup --no-save-pat` keeps it out of `secrets.json` | `setup` always writes the PAT |
| Auth env vars | `CAIDO_ACCESS_TOKEN`, `CAIDO_INSTANCE_URL`, `CAIDO_URL`, `CAIDO_PAT` | `CAIDO_URL`, `CAIDO_PAT` |
| Schema versions | GraphQL forked in-client at the 0.57.0 threshold (`*_V056` / `*_V057`) | delegated to the SDK's own transport forks |
| Older Node | optional `ws` fallback below Node 22.4 | handled by the SDK's transport |

The last two rows are a trade rather than a win: tracking Caido's schema by hand is what the missing dependencies cost, which is why the verification note above carries a date.

## Why

Cookies, JWTs, CSRF tokens, and other auth headers are often too large and fragile to copy by hand. The normal workflow is:

1. Find an organic request in Caido history that already has valid auth.
2. Use `edit` to change only the method, path, headers, or body.
3. Send it through Caido so the request, response, timing, and replay context stay visible in the UI.

## Requirements

- Node.js 24 or newer (Node 22.4+ also works; older Node requires the optional `ws` fallback below)
- A running Caido instance
- A Caido Personal Access Token for first-time setup

### WebSocket fallback for older Node

The client uses Node's built-in global `WebSocket`, which is enabled by default in Node 22.4+. On older runtimes you'll see `Global WebSocket is not available...`. To fix without upgrading Node, install the optional `ws` package inside the client directory — it's auto-detected at runtime and adds no required dependency to the skill:

```bash
cd skills/caido-mode/client && npm install ws
```

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
| HTTP History | `search` (with `--scope`), `recent`, `get`, `get-response`, `download`, `export-curl` |
| Analysis & Evidence | `compare`, `evidence` |
| Edit & Replay | `edit` (with `--values` for batch sends), `replay`, `send-raw`, `edit-session` |
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

### Not covered

`@caido/sdk-client` 0.5.0 exposes surface this client does not wrap, recorded here so it does not have to be rediscovered:

- Workflows — list, get, create, update, delete, toggle, test and run (reached the SDK 2026-07-21)
- Certificates — export the CA as PKCS#12, import, regenerate (2026-07-23)
- DNS upstream resolvers and DNS rewrites
- WebSocket replay sessions and entries (`ReplaySessionWs` / `ReplayEntryWs`), new in Caido 0.57
- Hosted-file upload and rename; plugin installation
- `createRequest`, `deleteFindings`, and project create/rename/delete
- Instance settings, which carry AI provider API keys — deliberately out of scope

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

Agents read `SKILL.md` for the loop and the engagement rules, and go to `references/` for
depth: `references/commands.md` is the full command and flag catalog, `references/httpql.md`
the query language. This README is the human-facing quick reference.

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
