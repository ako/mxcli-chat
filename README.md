# MxcliChat

A Mendix app, provisioned and developed with [mxcli](https://github.com/ako/mxcli).

## What this app is

A chat interface with an LLM backend. It should support custom MCP servers.
The app stores memories from the chat conversations, the chats, and the messages.

Single user — no authentication.

## What it keeps track of

- **Chats** — a conversation.
- **Messages** — the turns within a chat.
- **Memories** — facts distilled from conversations and carried into later ones.
- **MCP servers** — the custom tool servers the assistant may call.

## Who logs in

Nobody. Single user, no auth.

## Build facts

| | |
|---|---|
| Mendix version | 11.13.0 |
| Theme | `ledger` (warm paper, hairline rules, serif headings) |
| mxcli | built from source, `ako/mxcli` `main` @ `d762d2e` |
| Project file | `MxcliChat.mpr` at the repo root |
| Database | local PostgreSQL 16, database `mxclichat` |
| Marketplace content | all modules and widgets on their latest 11.13.0-compatible versions, except NanoflowCommons 6.0.0 (its installed version was unpublished, so `marketplace update` cannot baseline it — see `FINDINGS.md`) |

## Working on it

The `mxcli` binary is git-ignored (~86 MB). `.claude/bootstrap-mxcli.sh` rebuilds it
from `ako/mxcli` on session start, then caches MxBuild, starts PostgreSQL and
provisions the database.

```bash
./mxcli run --local -p MxcliChat.mpr     # boot the warm dev loop on :8080
./mxcli check <script.mdl>               # validate MDL
./mxcli exec  <script.mdl>               # apply MDL to the model
./mxcli lint                             # check the model for issues
```

See `AGENTS.md` for the full command reference and `FINDINGS.md` for what has
already bitten us.
