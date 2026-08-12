# Marketplace stack vs. plain REST — running tally

Two implementations of the same chat app, measured as they are built rather than
estimated afterwards. Both talk to OpenRouter, and both use the *same* memory
tools published by `MxcliChatCore` over MCP, so the only variable is how the chat
itself is assembled.

| | Version A — `MxcliChatAgent` | Version B — `MxcliChatRest` |
|---|---|---|
| Status | backend seeded and verified; **chat UI outstanding** | not started |
| Modules relied on | AgentCommons, ConversationalUI, MCPClient, GenAICommons, OpenAIConnector | OpenAIConnector only (REST activity + JSON mappings) |
| Own microflows so far | 5 | — |
| Own entities so far | 0 | — |
| Lines of MDL so far | 190 | — |

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
