# caido-agent-skills

Codex and Claude skills for controlling Caido through a pinned dependency-free client.

This repository does not include npm dependencies. The Caido client is pinned as a git submodule:

```text
skills/caido-mode/client
```

## Included Skills

- `caido-mode` - Search HTTP history, retrieve responses, byte-safe downloads, replay/edit requests, manage scopes/filters/environments/findings, control intercept, inspect plugins, and run Automate tasks through Caido.

## Clone

```bash
git clone --recurse-submodules https://github.com/kofersen/caido-agent-skills.git
```

If you already cloned without submodules:

```bash
git submodule update --init --recursive
```

SSH works too if you prefer authenticated GitHub access:

```bash
git clone --recurse-submodules git@github.com:kofersen/caido-agent-skills.git
```

## Install For Codex

```bash
mkdir -p ~/.codex/skills
cp -a skills/caido-mode ~/.codex/skills/caido-mode
```

## Install For Claude Code

```bash
mkdir -p ~/.claude/skills
cp -a skills/caido-mode ~/.claude/skills/caido-mode
```

## Setup

From an installed skill directory:

```bash
cd ~/.codex/skills/caido-mode
node client/caido-client.mjs setup <pat> <caido-url> --no-save-pat
node client/caido-client.mjs auth-status
```

For a local Caido instance:

```bash
node client/caido-client.mjs setup <pat> http://localhost:8080 --no-save-pat
```

## Updating The Client Pin

```bash
git submodule update --remote skills/caido-mode/client
git add .gitmodules skills/caido-mode/client
git commit -m "Update caido-headless-client"
```

Review the client diff before updating the pin. Keeping the client as a submodule makes the exact automation code auditable by commit hash.

## Security Notes

- Prefer `setup ... --no-save-pat`.
- Do not commit `~/.claude/config/secrets.json` or shell env files containing PATs.
- The skill repo links a pinned client commit instead of installing from npm.
