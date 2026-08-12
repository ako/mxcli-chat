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

## Agent stack

The app is built on Mendix's agent-editor stack, so chats, tools and MCP servers
are first-class model documents (`create agent`, `create model`,
`create knowledge base`, `create consumed mcp service`) rather than hand-rolled
REST plumbing. Installed: GenAI Commons, Mendix Cloud GenAI Connector, Agent
Commons, Agent Editor, MCP Client, Conversational UI, Encryption, Community
Commons, plus the Markdown viewer and Events widgets.

`AgentEditorCommons.ASU_AgentEditor` is wired as the after-startup microflow —
it materialises the model's agent documents into runtime rows at boot.

`Encryption.EncryptionKey` is set in the `Default` configuration to a **development
key that is committed to this repo**. Override it per environment before deploying.

## LLM backend: OpenRouter

The app talks to OpenRouter through the Mendix **OpenAI Connector** (9.1.0), which
works against any OpenAI-compatible endpoint. Configure it in the running app on
`OpenAIConnector.Configuration_Overview`:

| Field | Value |
|---|---|
| Endpoint | `https://openrouter.ai/api/v1` |
| API type | OpenAI |
| Is native OpenAI | **false** |
| API key | your OpenRouter key — **entered in the app, never committed** |

Then add a deployed model whose name is an OpenRouter model id. Free models carry
a `:free` suffix; MCP tool use needs one that supports tool calling — list them with

```bash
curl -s https://openrouter.ai/api/v1/models | \
  jq -r '.data[] | select(.id|endswith(":free"))
         | select(.supported_parameters|index("tools")) | .id'
```

Installing the connector needs `mdlsource/openai-connector-security-fix.mdl` run
once afterwards — see finding 25 in `FINDINGS.md` for why, and what it costs.

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
