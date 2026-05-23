# caido-mode

Agent skill for using Caido through the pinned `caido-headless-client` submodule.

## Client Path

From this skill directory:

```bash
export CAIDO_CLIENT="$PWD/vendor/caido-headless-client/caido-client.mjs"
```

From the repository root:

```bash
export CAIDO_CLIENT="$PWD/skills/caido-mode/vendor/caido-headless-client/caido-client.mjs"
```

## Setup

```bash
node "$CAIDO_CLIENT" setup <pat> <caido-url> --no-save-pat
node "$CAIDO_CLIENT" auth-status
```

## Common Commands

```bash
node "$CAIDO_CLIENT" search 'req.host.cont:"api"' --limit 20
node "$CAIDO_CLIENT" recent --limit 10
node "$CAIDO_CLIENT" get <request-id> --compact
node "$CAIDO_CLIENT" get-response <request-id> --compact
node "$CAIDO_CLIENT" download <request-id> --out body.bin
node "$CAIDO_CLIENT" replay <request-id> --compact
node "$CAIDO_CLIENT" edit <request-id> --path /api/test --compact
node "$CAIDO_CLIENT" plugins
node "$CAIDO_CLIENT" health
```

See `SKILL.md` for the full agent workflow.
