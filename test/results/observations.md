# A2A Lab Observations

This file documents what I observed while running the local A2A multi-agent system:

- Registry Stub (`http://127.0.0.1:7861`)
- Travel Assistant Agent (`http://127.0.0.1:10001`)
- Flight Booking Agent (`http://127.0.0.1:10002`)

## What A2A messages were exchanged between agents

During the discovery/booking workflow, the Travel Assistant invoked the Flight Booking Agent via an A2A JSON-RPC 2.0 call.

From the Travel Assistant logs (A2A request):

```json
{
  "jsonrpc": "2.0",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        {
          "kind": "text",
          "text": "I need to book flight ID 1 for 2 seats. Please reserve the seats, confirm the reservation, and process the payment."
        }
      ],
      "messageId": "95dbb65fe42041cdb7f079f0d4811d25"
    }
  }
}
```

At a high level, this message was sent from:

- Travel Assistant (client/orchestrator) → Flight Booking Agent (specialist)

The response returned to Travel Assistant contained the final agent output in `result.artifacts[*].parts[*].text` (the A2A SDK models the interaction as a task with artifacts/history).

## How the Travel Assistant discovered the Flight Booking Agent

The Travel Assistant used the registry’s semantic discovery endpoint by sending a query describing the needed capability.

From the Registry Stub logs:

- Query: `flight booking reservation payment confirmation`
- Endpoint: `POST /api/agents/discover/semantic?query=...&max_results=5`
- Registry behavior: returns a canned match for the Flight Booking Agent

From the Travel Assistant logs, the discovery tool call looked like:

- `discover_remote_agents(query='flight booking reservation payment confirmation', max_results=5)`
- It reported `Found 1 agents` and then cached the agent as ID `/flight-booking-agent`.

## JSON-RPC request/response format observed

Observed request structure (JSON-RPC 2.0 envelope):

- `jsonrpc`: `"2.0"`
- `method`: `"message/send"`
- `params.message.role`: `"user"`
- `params.message.parts`: `[{ "kind": "text", "text": "..." }]`
- `params.message.messageId`: per-message unique ID

Observed response structure (conceptual):

- Response returns a JSON-RPC envelope with `jsonrpc` and `id` (matching the request) and a `result`
- `result.artifacts`: contains the final consolidated output (text)
- `result.history`: contains message chunks produced over time
- `result.status.state`: indicates completion (e.g., `"completed"`)

## What was in the agent card and how it was used

The Flight Booking Agent exposes its Agent Card at:

- `http://127.0.0.1:10002/.well-known/agent-card.json`

The Travel Assistant fetched this card before/while initializing the A2A client to know:

- **Where to send requests** (`url`)
- **How to communicate** (`preferredTransport` = `JSONRPC`, `protocolVersion`)
- **What it can do** (`skills`: `check_availability`, `reserve_flight`, `confirm_booking`, `process_payment`, `manage_reservation`)
- **Whether streaming is supported** (`capabilities.streaming`)

## Benefits and limitations (my observations)

### Benefits

- **Loose coupling**: the Travel Assistant doesn’t hardcode the booking agent; it discovers it at runtime via a registry query.
- **Self-describing services**: agent cards make integration easier by advertising transport/protocol/skills in one place.
- **Delegation pattern**: the orchestrator agent can focus on intent + planning, while specialists handle domain-specific actions.

### Limitations

- **Registry quality is critical**: discovery depends heavily on the registry’s semantic matching and the accuracy of agent descriptions.
- **Operational complexity**: running multiple services (and keeping ports/health correct) increases setup overhead.
- **Protocol/version/streaming nuances**: transport details and SDK behavior (e.g., streaming conformance warnings) can cause interoperability issues if agents/frameworks differ.

