# caido-mode

Agent skill for using Caido through the pinned `caido-headless-client` submodule.

## Client Command

From this skill directory:

```bash
node client/caido-client.mjs <command>
```

From the repository root:

```bash
node skills/caido-mode/client/caido-client.mjs <command>
```

## Setup

```bash
node client/caido-client.mjs setup <pat> <caido-url> --no-save-pat
node client/caido-client.mjs auth-status
```

## Common Commands

```bash
node client/caido-client.mjs search 'req.host.cont:"api"' --limit 20
node client/caido-client.mjs recent --limit 10
node client/caido-client.mjs get <request-id> --compact
node client/caido-client.mjs get-response <request-id> --compact
node client/caido-client.mjs download <request-id> --out body.bin
node client/caido-client.mjs replay <request-id> --compact
node client/caido-client.mjs edit <request-id> --path /api/test --compact
node client/caido-client.mjs plugins
node client/caido-client.mjs health
```

See `SKILL.md` for the full agent workflow.
