# Marketplace stack vs. plain REST — running tally

Two implementations of the same chat app, measured as they are built rather than
estimated afterwards. Both talk to OpenRouter, and both use the *same* memory
tools published by `MxcliChatCore` over MCP, so the only variable is how the chat
itself is assembled.

| | Version A — `MxcliChatAgent` | Version B — `MxcliChatRest` |
|---|---|---|
| Status | **complete** — chat page renders, message sends, blocked only by egress | not started |
| Modules relied on | AgentCommons, ConversationalUI, MCPClient, GenAICommons, OpenAIConnector | OpenAIConnector only (REST activity + JSON mappings) |
| Own microflows | 6 | — |
| Own entities | 0 | — |
| Own pages | 1 (a data view and one snippet call) | — |
| Own Java actions | 1 (a workaround, not a feature) | — |
| Lines of MDL | 300 | — |

## Shared, and excluded from both totals

`MxcliChatCore` — 1 entity, 1 enumeration, 8 microflows, 2 Java actions, ~430
lines of MDL. Publishes `memory_search`, `memory_list`, `memory_add` and
`memory_forget` at `/memory/mcp`.

## Version A so far

Five microflows, no entities of its own, and that is the headline: the chat
domain model — conversations, messages, tool calls, streaming state — comes from
ConversationalUI and GenAICommons rather than being modelled here.

| Microflow | What it does |
|---|---|
| `SUB_Seed_Configuration` | OpenRouter as an OpenAI Connector configuration |
| `SUB_Seed_DeployedModel` | `openai/gpt-oss-20b:free`, wired to that configuration |
| `SUB_Seed_McpService` | registers the app's own `/memory/mcp` with the MCP client |
| `SUB_Seed_Agent` | Agent + in-use Version + the MCP tool link |
| `ASU_AgentSetup` | calls the four above, create-if-absent |
| `ACT_Chat_Open` | creates a ChatContext for the agent and opens the page |

**The agent is data, not a document.** Agent-editor documents (`create agent`)
cannot be authored headlessly — see FINDINGS 29–32 — so the agent is seeded as
`AgentCommons` rows in a startup microflow. That is a supported route (the module
documents agents as something Agent Admins create in the UI), but it is worth
being explicit that version A is *not* using the Studio Pro Agent Editor, which
is the part of the marketplace offer this workflow cannot reach.

Verified after boot:

```
AgentCommons.Agent            MxcliChat Assistant  / Conversational
AgentCommons.Version          v1, IsDraftVersion=false
AgentCommons.Tool             mxclichat-memory, IsEnabled=true
OpenAIConnector.OpenAIDeployedModel  openai/gpt-oss-20b:free
    -> OpenAIConnector.ChatCompletions_WithHistory_Execute
MCPClient.ConsumedMCPService  http://localhost:8080/memory/mcp
```

## Still to do

- **Version A:** the chat page — a ConversationalUI snippet over a `ChatContext`
  from `AgentCommons.ChatContext_Create_ForAgent`, plus the action microflow it
  runs on send. Watch for `ChatContext_Create_ForAgent.ActionMicroflow`: if that
  is microflow-typed in the model like `AddTool.ExecutingMicroflow` was, it needs
  the same Java bridge (FINDINGS 36).
- **Version A:** the OpenRouter API key, entered on
  `OpenAIConnector.Configuration_Overview`, before any of it can actually answer.
- **Version B:** everything.

## What is already clear

The marketplace stack's value is not the agent call — it is everything around it:
chat history, message and tool-call entities, streaming, the tool-approval UI and
the MCP client. Version A has written no entity of its own. Whether that trade is
worth the eight modules and their upgrade burden is exactly what version B is for.

Two costs are already on the board and belong in any honest comparison: the
CE0066 that installing the OpenAI Connector inflicts (FINDINGS 24–25), and the
fact that the Agent Editor — the marketing centrepiece — is unreachable without
Studio Pro (FINDINGS 29–32).

## Version A is done

The chat page is one container, one data view and one snippet call — 24 lines of
MDL. Everything visible is ConversationalUI's: message list, composer, tool-call
rendering, streaming, the "press Enter to submit" hint. Verified in a headless
browser: the page renders in the ledger theme, a message can be typed, and it
reaches the model call.

It does not answer yet, for two reasons that are both outside the model:

1. No OpenRouter API key — it belongs in the running app, not this repo.
2. The runtime cannot resolve `openrouter.ai` from this container. Its REST call
   ignores the environment's egress proxy (FINDINGS 44). `curl` from the shell
   works; the JVM does not.

So the honest status is: **wired end to end, unproven at the last hop.** The final
error moved from ConversationalUI's dispatcher, through the agent, down to
`OpenAIConnector.Request_POST` returning 503 — which is itself the evidence that
every link between the page and the HTTP call is connected.

### What version A cost

| | |
|---|---|
| Written by hand | 6 microflows, 1 page, 1 Java action, 300 lines of MDL |
| Not written | chat/message/tool-call entities, the tool loop, streaming, the chat UI, the MCP client |
| Paid for it | 8 marketplace modules + 3 transitive dependencies |
| Incidents on the way | CE0066 on install (24–25), two microflow-typed-parameter bridges (36, 42), one action microflow that looked right and was not (43) |

The one-line summary so far: **the modules removed the hard parts and added a
different kind of work** — finding out what the modules actually expect. None of
the four failures above were logic errors; every one was a wiring convention that
had to be discovered by reading module internals or a stack trace.

Version B is the control. If it takes 25 microflows and a week of JSON mapping,
the modules earned their keep. If it takes 12 and never surprises anyone, that is
worth knowing too.
