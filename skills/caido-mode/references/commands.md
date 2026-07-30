# Caido client command reference

Every command and flag. `SKILL.md` carries the loop and the commands used in almost every
session; read this when you need a command that is not there, or the full flag list for one
that is.

All commands print JSON unless noted. `$CAIDO_CLIENT` is `client/caido-client.mjs` inside the
installed skill directory.

## Contents

- [Global options](#global-options)
- [HTTP history](#http-history)
- [Comparing](#comparing)
- [Evidence](#evidence)
- [Replay and editing](#replay-and-editing)
- [Batch sends](#batch-sends)
- [Replay sessions and collections](#replay-sessions-and-collections)
- [Automate](#automate)
- [Scopes](#scopes)
- [Filter presets](#filter-presets)
- [Environments](#environments)
- [Findings](#findings)
- [Tasks, projects, hosted files](#tasks-projects-hosted-files)
- [Intercept](#intercept)
- [Info and health](#info-and-health)
- [Output control](#output-control)

## Global options

| Flag | Description |
|---|---|
| `--json-compact` | One-line JSON instead of indented. Roughly a quarter fewer bytes on list output. Also `CAIDO_COMPACT_JSON=1` |
| `--delay <ms>` | Minimum gap between sends to one host, best-effort across processes. Also `CAIDO_MIN_INTERVAL_MS`. `--delay 0` switches off the batch default |

Auth environment: `CAIDO_ACCESS_TOKEN`, `CAIDO_PAT`, `CAIDO_URL`, `CAIDO_INSTANCE_URL`.

## HTTP history

### search

```bash
node "$CAIDO_CLIENT" search 'req.method.eq:"POST" AND resp.code.eq:200'
node "$CAIDO_CLIENT" search 'req.host.cont:"api"' --limit 50
node "$CAIDO_CLIENT" search 'req.path.cont:"/admin"' --ids-only
node "$CAIDO_CLIENT" search '' --scope "Target Corp" --limit 20
node "$CAIDO_CLIENT" search 'resp.raw.cont:"password"' --after <cursor>
```

| Flag | Description |
|---|---|
| `--limit <n>` | Page size, default 20 |
| `--after <cursor>` | Continue from a previous page's cursor |
| `--ids-only` | Just the request ids |
| `--desc`, `--latest` | Newest first |
| `--scope <id-or-name>` | Only requests inside that Caido scope |

### recent, get, get-response

```bash
node "$CAIDO_CLIENT" recent --limit 10
node "$CAIDO_CLIENT" get <request-id> --headers-only
node "$CAIDO_CLIENT" get-response <request-id> --compact
```

### download

Byte-safe; use it whenever the body may be binary.

```bash
node "$CAIDO_CLIENT" download <request-id> --out body.bin
node "$CAIDO_CLIENT" download <request-id> --response --raw --out response.http
node "$CAIDO_CLIENT" download <request-id> --request --raw --out request.http
```

| Flag | Description |
|---|---|
| `--out <file>` | Required |
| `--request` / `--response` | Which side, default response |
| `--raw` | Full HTTP message including headers |
| `--body-only` | Body only, the default |
| `--force` | Overwrite an existing file. Symlinks are refused either way |

### export-curl

```bash
node "$CAIDO_CLIENT" export-curl <request-id>
```

Prints a curl command. It is a reconstruction, not byte-exact: bodies are trimmed and
re-quoted. For evidence use `download --raw` or `evidence`.

## Comparing

```bash
node "$CAIDO_CLIENT" compare <id-a> <id-b>
node "$CAIDO_CLIENT" compare <id-a> <id-b> --request
node "$CAIDO_CLIENT" compare <id-a> <id-b> --all-headers --max-diff-lines 40
```

Answers whether two responses differ and where, without pulling both into context. Reports
status, length delta, header differences and the differing body region.

| Flag | Description |
|---|---|
| `--response` | Compare responses, the default |
| `--request` | Compare the request side instead |
| `--all-headers` | Include per-response headers (`date`, `cf-ray`, `x-request-id`, `age`, `report-to`, `nel`, `alt-svc`, `server-timing`, `x-amz-*`) that are otherwise listed as `ignored` |
| `--max-diff-lines <n>` | Differing lines per side, default 20 |
| `--max-line-chars <n>` | Width per emitted line, default 300. Minified bodies are one long line, so the output windows around the first differing character and reports `firstDiffOffset` |
| `--max-bytes <n>` | Body bytes compared per side, default 262144 |

Equality is decided on bytes, then described in text. `binaryOnlyDifference: true` means the
bodies decode to the same text but the bytes differ.

## Evidence

```bash
node "$CAIDO_CLIENT" evidence <request-id> --out reports/<slug>/evidence
node "$CAIDO_CLIENT" evidence --finding <finding-id> --out findings/<slug>
```

Writes `request.http`, `response.http`, `curl.sh` and `meta.json`, and prints the manifest.
`request.http` and `response.http` are the bytes Caido stored; `curl.sh` is a convenience.

| Flag | Description |
|---|---|
| `--out <dir>` | Required; created if missing |
| `--finding <id>` | Take the request from a finding instead of a request id |
| `--force` | Overwrite existing files. The whole set is checked first, so a conflict never leaves half a bundle |

Files are written `0600`: they carry session cookies and authorization headers.

## Replay and editing

### edit — the default way to send

Modifies an existing request while preserving cookies and auth headers.

```bash
node "$CAIDO_CLIENT" edit <id> --path /api/user/999
node "$CAIDO_CLIENT" edit <id> --method POST --body '{"role":"admin"}'
node "$CAIDO_CLIENT" edit <id> --set-header "X-Forwarded-For: 127.0.0.1"
node "$CAIDO_CLIENT" edit <id> --remove-header "X-CSRF-Token"
node "$CAIDO_CLIENT" edit <id> --replace "user123:::user456"
node "$CAIDO_CLIENT" edit <id> --path /api/user/1001 --session <session-id> --compact
```

| Flag | Description |
|---|---|
| `--method <METHOD>` | Change the method |
| `--path <path>` | Change the path |
| `--set-header <Name: Value>` | Add or replace a header, repeatable |
| `--remove-header <Name>` | Remove a header, repeatable |
| `--body <content>` | Set the body; Content-Length follows |
| `--replace <from>:::<to>` | Find and replace anywhere in the request, repeatable |
| `--session <id>` | Reuse a replay tab instead of creating one |
| `--collection <id>` | Put a new session in a collection |
| `--values <spec>` | Batch mode, see below |

### replay, send-raw, edit-session

```bash
node "$CAIDO_CLIENT" replay <request-id>
node "$CAIDO_CLIENT" replay <id> --raw "GET /modified HTTP/1.1\r\nHost: example.com\r\n\r\n"
node "$CAIDO_CLIENT" send-raw --host example.com --port 443 --tls --raw @request.txt --name "G /"
cat request.txt | node "$CAIDO_CLIENT" send-raw --host example.com --raw -
node "$CAIDO_CLIENT" edit-session <session-id-or-name> --body '{"test":true}' --compact
```

`--raw` accepts a string with `\r\n` escapes, `@file`, or `-` for stdin.

### Connection overrides

Work with `edit`, `replay`, `send-raw` and `edit-session`.

| Flag | Description |
|---|---|
| `--sni <host>` | Override TLS SNI |
| `--connect-host <host>` | Connect elsewhere while keeping the request intact |
| `--connect-port <port>` | Connect to a different port |
| `--connect-tls` / `--connect-no-tls` | Force TLS or plaintext |

### Backoff reporting

Every send reports a `backoff` object when the response is one of `rate-limited` (429),
`challenge` (a `cf-mitigated` header), `service-unavailable` (503) or `retry-after`. Treat it
as a stop signal, not a result: a 429 recorded as "blocked" is a verdict that outlives the
request.

## Batch sends

```bash
node "$CAIDO_CLIENT" edit <id> --path '/api/user/{}' --values 1-100
node "$CAIDO_CLIENT" edit <id> --replace 'ORIG:::{}' --values @ids.txt --delay 500
node "$CAIDO_CLIENT" edit <id> --path '/{}' --values admin,internal,debug --session <id>
```

One request per value through a single replay session, one summary row each instead of full
bodies. `{}` in `--method`, `--path`, `--body`, `--set-header` or `--replace` is where the
value goes; without one the command refuses to run.

`--values` takes `1-100` (ascending, capped at 1000), a comma list, or `@file` with one value
per line. Sends pace at 250ms unless `--delay` says otherwise, and the run stops at the first
backoff signal or send error, reporting `stopped` and the rows completed so far.

For payload-driven fuzzing with wordlists, use Caido's Automate instead.

## Replay sessions and collections

```bash
node "$CAIDO_CLIENT" create-session <request-id> [--collection <id>]
node "$CAIDO_CLIENT" rename-session <session-id> "idor-user-profile"
node "$CAIDO_CLIENT" replay-sessions --limit 50
node "$CAIDO_CLIENT" move-session <session-id> <collection-id>
node "$CAIDO_CLIENT" delete-sessions <id-1>,<id-2>
node "$CAIDO_CLIENT" get-session <id-or-name> --compact
node "$CAIDO_CLIENT" replay-entries <id-or-name> --limit 20 [--raw]

node "$CAIDO_CLIENT" replay-collections --limit 50
node "$CAIDO_CLIENT" create-collection "IDOR Testing"
node "$CAIDO_CLIENT" rename-collection <collection-id> "Auth Bypass Tests"
node "$CAIDO_CLIENT" delete-collection <collection-id>
```

`session-entries` is an alias for `replay-entries`. Sessions resolve by id or by name.

## Automate

```bash
node "$CAIDO_CLIENT" create-automate-session <request-id>
node "$CAIDO_CLIENT" fuzz <session-id>
```

Configure payloads and markers in the Caido UI before starting.

## Scopes

```bash
node "$CAIDO_CLIENT" scopes
node "$CAIDO_CLIENT" create-scope "Target Corp" --allow "*.target.com,*.target.io" --deny "*.cdn.target.com"
node "$CAIDO_CLIENT" update-scope <scope-id> --allow "*.target.com,*.api.target.com"
node "$CAIDO_CLIENT" delete-scope <scope-id>
```

Glob patterns: `*.example.com` matches any subdomain.

## Filter presets

```bash
node "$CAIDO_CLIENT" filters
node "$CAIDO_CLIENT" create-filter "API Errors" --query 'req.path.cont:"/api/" AND resp.code.gte:400'
node "$CAIDO_CLIENT" create-filter "Auth Endpoints" --query 'req.path.regex:"/(login|auth|oauth)/"' --alias auth
node "$CAIDO_CLIENT" update-filter <filter-id> --query 'resp.code.gte:500'
node "$CAIDO_CLIENT" delete-filter <filter-id>
```

Use a saved preset from a query with `preset:"API Errors"`.

## Environments

```bash
node "$CAIDO_CLIENT" envs
node "$CAIDO_CLIENT" create-env "IDOR-Test"
node "$CAIDO_CLIENT" env-set <env-id> victim_user_id "user_456"
node "$CAIDO_CLIENT" select-env <env-id>     # no id deselects
node "$CAIDO_CLIENT" delete-env <env-id>
```

## Findings

```bash
node "$CAIDO_CLIENT" findings --limit 50
node "$CAIDO_CLIENT" get-finding <finding-id>
node "$CAIDO_CLIENT" create-finding <request-id> \
  --title "IDOR in user profile endpoint" \
  --description "Reads another tenant's profile by changing the id" \
  --dedupe-key "target-idor-user-profile"
node "$CAIDO_CLIENT" update-finding <finding-id> --title "Updated title" [--hidden|--visible]
```

`--dedupe-key` is what stops the same finding being filed twice across sessions. Derive it
from program, class and endpoint rather than inventing one per run.

## Tasks, projects, hosted files

```bash
node "$CAIDO_CLIENT" tasks
node "$CAIDO_CLIENT" cancel-task <task-id>
node "$CAIDO_CLIENT" projects
node "$CAIDO_CLIENT" select-project <project-id>
node "$CAIDO_CLIENT" hosted-files
node "$CAIDO_CLIENT" delete-hosted-file <file-id>
```

## Sitemap

```bash
node "$CAIDO_CLIENT" sitemap                                  # root domains
node "$CAIDO_CLIENT" sitemap --scope "Target Corp" --limit 50
node "$CAIDO_CLIENT" sitemap target.com                       # direct children
node "$CAIDO_CLIENT" sitemap target.com --all --limit 500      # whole subtree
```

Caido's deduplicated view of what has been seen on a host — the coverage question without
paging through history. Entries carry their path, kind (`DOMAIN`, `DIRECTORY`, `REQUEST`,
`REQUEST_QUERY`, `REQUEST_BODY`), method and the request id behind them, so anything
interesting goes straight to `get` or `compare`.

| Flag | Description |
|---|---|
| `--scope <id-or-name>` | Only roots inside that scope |
| `--all` | Whole subtree instead of direct children |
| `--limit <n>` | Entries returned, default 200; `truncated` says when there were more |

Paths come from each entry's own request. A directory's path ends in `/`, which is how
`/admin` and `/admin/` stay distinct — they are two different entries.

## WebSocket and SSE

```bash
node "$CAIDO_CLIENT" streams --limit 20 [--scope "Target Corp"]
node "$CAIDO_CLIENT" stream-messages <stream-id> --limit 50
node "$CAIDO_CLIENT" stream-messages <stream-id> --raw --max-body-chars 500
```

Streams are not requests, so `search` and `recent` do not see them at all: an application
that talks over WebSocket is invisible until you look here. `streams` lists them with host,
path, protocol (`WS` or `SSE`) and direction; `stream-messages` lists a stream's frames with
direction (`CLIENT` or `SERVER`), format, length and `alteration`.

Payloads only appear with `--raw`, truncated by the usual output flags. Binary frames report
their size instead of being decoded.

Reading is wrapped; sending or editing WebSocket messages is not — see the skill README.

## Match-and-replace rules

```bash
node "$CAIDO_CLIENT" rules
```

Read-only. Lists every rule collection with each rule's name, whether it is enabled, which
part of the message it rewrites, its HTTPQL condition (empty means it applies to everything),
and for header and body rules the matcher and the replacement. Other sections report which
part they rewrite without the substitution detail.

Rules apply to traffic this client never issued, so their effects otherwise read as the
target's behaviour. Requests and responses a rule changed carry `alteration: TAMPER`; a
message edited by hand carries `edited: true`. Neither field appears when there is nothing to
report.

## Intercept

```bash
node "$CAIDO_CLIENT" intercept-status
node "$CAIDO_CLIENT" intercept-enable
node "$CAIDO_CLIENT" intercept-disable
```

## Info and health

```bash
node "$CAIDO_CLIENT" health        # version and ready state
node "$CAIDO_CLIENT" viewer        # current user
node "$CAIDO_CLIENT" plugins
node "$CAIDO_CLIENT" auth-status
node "$CAIDO_CLIENT" setup <pat> <url> --no-save-pat
```

## Output control

Work with `get`, `get-response`, `replay`, `edit`, `send-raw`, `edit-session`,
`replay-entries` and `get-session`.

| Flag | Description |
|---|---|
| `--max-body <n>` | Max body lines, default 200, `0` unlimited |
| `--max-body-chars <n>` | Max body chars, default 5000, `0` unlimited |
| `--no-request` | Skip the request raw |
| `--headers-only` | Headers, no body |
| `--compact` | `--no-request --max-body 50 --max-body-chars 5000` |
