---
name: caido-mode
description: Dependency-free Caido CLI integration for Codex and Claude. Search HTTP history, replay/edit requests, compare responses, manage scopes/filters/environments, create findings, export evidence and curl commands, byte-safe downloads, and control intercept through a pinned caido-headless-client submodule.
tags: [worker]
---

# Caido Mode Skill

Drive a running Caido instance from the terminal: search history, replay and edit requests
with their auth intact, compare responses, record findings, and export evidence. Every
request goes through Caido, so it lands in the UI with full replay context for later
analysis.

The client is a single dependency-free Node ESM file, pinned as a submodule at `client/`.

- Full command and flag catalog: `references/commands.md`
- HTTPQL fields, operators and examples: `references/httpql.md`

## Why edit, not copy-paste

Session cookies, JWTs and CSRF tokens are routinely 1-2 KB and fragile. So the loop is never
"build a request from scratch":

1. Find an organic request in history that already carries valid auth.
2. `edit` only what the test changes — path, method, header, body.
3. Send it through Caido, where the response arrives with the context preserved.

`replay --raw` and `send-raw` exist for the cases where nothing suitable is in history.
Reach for `edit` first.

## Setup

Create a PAT in [Dashboard → Developer → Personal Access Tokens](https://docs.caido.io/dashboard/guides/create_pat.html), then, from the installed skill directory:

```bash
cd ~/.claude/skills/caido-mode        # Codex: ~/.codex/skills/caido-mode
node client/caido-client.mjs setup <pat> <caido-url> --no-save-pat
node client/caido-client.mjs auth-status
```

`--no-save-pat` keeps the PAT off disk; the cached access and refresh tokens go to
`~/.claude/config/secrets.json`, and the refresh token is rotated automatically, so an
unattended run survives token expiry.

Auth resolution: `CAIDO_ACCESS_TOKEN` → valid cached token → refresh token → `CAIDO_PAT` →
stored PAT → an error telling you to run `setup`.

Set a short path once per session:

```bash
export CAIDO_CLIENT="$PWD/client/caido-client.mjs"    # in this repo: skills/caido-mode/client/caido-client.mjs
alias caido='node "$CAIDO_CLIENT"'                    # optional, and much easier to read back
```

## The loop

```bash
# 1. Confirm the instance and the credentials
node "$CAIDO_CLIENT" health
node "$CAIDO_CLIENT" recent --limit 1

# 2. See what is known on the target, then find an authenticated request
node "$CAIDO_CLIENT" sitemap target.com --all --limit 100
node "$CAIDO_CLIENT" search 'req.path.cont:"/api/user"' --scope "Target Corp" --limit 10

# 3. Read only what you need
node "$CAIDO_CLIENT" get <request-id> --headers-only

# 4. Change one thing and send it
node "$CAIDO_CLIENT" edit <request-id> --path /api/user/999 --compact

# 5. Ask what actually differs, instead of reading both responses
node "$CAIDO_CLIENT" compare <original-id> <replayed-id>

# 6. Record it, then export what a report needs
node "$CAIDO_CLIENT" create-finding <request-id> --title "IDOR on /api/user/:id" --dedupe-key "target-idor-user-profile"
node "$CAIDO_CLIENT" evidence <request-id> --out reports/<slug>/evidence
```

## What the proxy did to your traffic

Caido's match-and-replace rules rewrite traffic the client never issued — browser `fetch`,
raw sockets, other tools — and their effects look exactly like the target's own behaviour.
This has produced confidently wrong conclusions more than once.

**Run `rules` at the start of an engagement.** It lists every match-and-replace rule, what it
matches, what it substitutes and which `sources` it applies to — a rule listing `REPLAY`
rewrites this client's own sends too. Zero rules is a useful answer: it removes the
explanation.

Rules can also be written: `create-rule`, `update-rule`, `toggle-rule`, `rename-rule`,
`move-rule`, `delete-rule` and the collection equivalents. Reach for it when a program
requires an attribution header on all traffic, and remember what one costs — a rule rewrites
every message matching its condition, including traffic this client never sent, for as long as
it is enabled. New rules start disabled.

**Check `alteration` on anything surprising.** `get`, `search` and the send commands report
`alteration: TAMPER` when a rule changed that message and `edited: true` when it was modified
by hand. Absent means neither. A response body that mentions a host you did not expect, on a
request carrying `alteration: TAMPER`, is the proxy talking, not the target.

**Setting a header the proxy also sets gives you a doubled value, not an override.**
`x-foo: a` plus a rule adding `x-foo: b` goes out as `x-foo: a, b`, and origins reject that in
ways that look like a ban. This one sentence is worth the whole section.

## Engagement discipline

**Probe read-only first.** Establish the boundary with a GET or an unchanged replay before
sending anything that writes. When a mutating step is the only way to prove impact, do the
smallest version of it, and say in the writeup what was deliberately not run — unexplained
restraint reads as a gap in the evidence, attributed restraint reads as discipline. Never
touch billing, subscriptions or another party's data to make a point.

**Treat backoff as a stop, not a result.** Sends report a `backoff` object for
`rate-limited`, `challenge`, `service-unavailable` or `retry-after`. A 429 written down as
"blocked" or "protected" is a wrong verdict that outlives the request. Pace with
`--delay <ms>` (or `CAIDO_MIN_INTERVAL_MS`) for the whole engagement; batch sends already
pace themselves and stop on the first signal.

**Re-request any single-path anomaly slowly before believing it.** A sweep run too fast
produces lone 403s and short truncated bodies on individual paths, which is the exact shape of
an access-control carve-out and is not one. Rate-induced artefacts disappear on a slow
re-request; real findings do not. Check for 429s and `error code: 1015` bodies elsewhere in
the same run — that is the tell.

**Stay in scope.** Define the engagement's scope in Caido once, then search with
`--scope <name>` so history from other work never enters the picture.

**One project per engagement** (`projects`, `select-project`), so traffic does not mix and
deleting an engagement's data later is one action. History, findings, sitemap, streams and
match-and-replace rules are all project data: the same command answers differently depending
on which project is selected, and `auth-status` says which one that is. An empty `rules` or a
short history usually means the wrong project, not a clean slate.

**Name every replay session** after what it tests — `rename-session <id> "idor-user-profile"`
— because the UI is where a human picks this up afterwards.

**Reuse dedupe keys.** `create-finding --dedupe-key <program>-<class>-<endpoint>` makes
findings the memory of what has already been proven, across sessions.

**Keep raw bodies out of context.** `--headers-only` and `--compact` while exploring,
`compare` instead of two full responses, `--json-compact` on list output, `download` when
bytes need to reach disk rather than the transcript.

**WebSocket traffic is invisible to `search`.** Streams are not requests. If the target talks
over WebSocket or SSE, `streams` lists them and `stream-messages <id>` shows the frames;
nothing else in this client will surface them. Both take `--filter` in StreamQL, not HTTPQL —
`ws.raw.cont:"token"` searches payloads, `stream.path.cont:"/socket"` finds the stream. Filter
values are lower case even though the output is upper case: a frame shown as
`"direction": "CLIENT"` matches `ws.direction.eq:"client"` and nothing at all as `"CLIENT"`.

**Find traffic you sent from outside the client** by bounding the search rather than guessing:
note the newest id first (`recent --limit 1`), then `search 'row.id.gt:<id>'` afterwards. When
even that is ambiguous, send a unique marker header and search for it — which doubles as the
check that the browser or script is actually going through Caido at all. The skill's claim
that everything goes through Caido holds for what this client sends, and is silently false for
anything not configured to use the proxy.

## Commands used in nearly every session

```bash
# History
node "$CAIDO_CLIENT" search '<httpql>' --scope <name> --limit 10 [--ids-only] [--desc]
node "$CAIDO_CLIENT" get <id> [--headers-only|--compact]
node "$CAIDO_CLIENT" get-response <id> --compact

# Send
node "$CAIDO_CLIENT" edit <id> --path /new --set-header "X-Test: 1" --session <id> --compact
node "$CAIDO_CLIENT" replay <id> --compact
node "$CAIDO_CLIENT" send-raw --host example.com --raw @request.txt

# Understand the result
node "$CAIDO_CLIENT" compare <id-a> <id-b>

# Record and export
node "$CAIDO_CLIENT" create-finding <id> --title "..." --dedupe-key "..."
node "$CAIDO_CLIENT" evidence <id> --out <dir>
node "$CAIDO_CLIENT" export-curl <id>
node "$CAIDO_CLIENT" download <id> --response --raw --out response.http
```

### compare

Two requests in, one answer out: same status, same length, which headers moved, and the
differing region of the body. Minified bodies are handled by windowing around the first
differing character rather than printing the head, and equality is decided on bytes, so two
binary bodies are never called identical because both decoded to `U+FFFD`.

```bash
node "$CAIDO_CLIENT" compare 1201 1202                      # responses
node "$CAIDO_CLIENT" compare 1201 1202 --request            # request side
node "$CAIDO_CLIENT" compare 1201 1202 --all-headers        # include date/cf-ray/etc
```

This is the authz primitive: send the same request as two identities, compare, and the answer
is a few lines instead of two full responses.

### Batch sends

```bash
node "$CAIDO_CLIENT" edit <id> --path '/api/user/{}' --values 1-100
node "$CAIDO_CLIENT" edit <id> --replace 'ORIG:::{}' --values @ids.txt --delay 500
```

One request per value through a single replay session, one row each — value, status, length,
request id. `{}` marks where the value goes. Paces at 250ms unless `--delay` says otherwise,
caps at 1000 values, and stops at the first backoff signal or send error, reporting `stopped`
and the rows completed. For wordlist-driven fuzzing use Caido's Automate instead.

### evidence

Writes `request.http`, `response.http`, `curl.sh` and `meta.json` into a directory and prints
the manifest — the shape a report needs. `--finding <id>` starts from a finding instead of a
request id. Files are `0600` because they carry cookies and authorization headers.
`request.http` and `response.http` are the stored bytes; `curl.sh` is a convenience and is not
byte-exact.

## HTTPQL, the parts that bite

**Quote strings, not integers.** `req.method.eq:"POST" AND resp.code.eq:200`

**There is no `NOT`.** Use `ne`, `ncont`, `nlike`, `nregex`. `req.path.ncont:"/admin"`, never
`NOT req.path.cont:"/admin"`.

```httpql
req.host.cont:"api" OR req.path.cont:"/api/"
resp.code.gte:400 AND resp.code.lt:500
"password" OR "secret" OR "api_key"
source:"replay"
```

Everything else — the field table, operators, dates, presets — is in `references/httpql.md`.

## Not wrapped

Caido exposes surface this client does not: workflows, certificate export/import, DNS
upstreams and rewrites, sending or editing WebSocket messages (reading them is wrapped),
hosted-file upload, plugin installation, upstream proxy chaining, backups, `createRequest`,
`deleteFindings`, project create/rename/delete, and driving the intercept queue. Do not reach for a command from that
list; `client/README.md` records what was checked and why each one is out.

## Errors

| Symptom | Cause and fix |
|---|---|
| Auth errors | `auth-status`; re-run `setup <pat> <url> --no-save-pat` |
| Connection refused | Caido is not running or the URL is wrong; `health` |
| `InstanceNotReadyError` | Caido is still starting; wait and retry |
| A send hangs for ~30s and returns no response | The target stalled the connection; the entry is recorded without a response |

Reads retry twice on transient transport failures. Mutations never retry — a resend would put
a second request on the target.

## Related

- `vulnerability-reports` turns the evidence bundle into a submission.
