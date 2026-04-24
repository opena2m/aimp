# AIMP 03 — AI Protocol Integration

**Status:** Draft · **Version:** 1.0.0-draft · **Informative & Normative**

AIMP is designed to ride on top of existing AI-tooling standards. This chapter specifies
how AIMP surfaces through three of them:

- **Model Context Protocol (MCP)** — the primary binding.
- **OpenAI-compatible function calling** — direct JSON-Schema tool export.
- **Agent-to-Agent (A2A)** topologies — how AIMP fits multi-agent systems.

If your agent can call tools through any of these, you can drive physical machines through
AIMP without touching an HTTP library directly.

## 1. MCP Binding (Primary)

### 1.1 Topology

```
┌───────────────┐   MCP (JSON-RPC 2.0)   ┌──────────────────────┐   AIMP/HTTPS   ┌──────────┐
│  MCP Client   │ ─────────────────────▶ │  AIMP MCP Server     │ ─────────────▶ │ Hardware │
│  (LLM host)   │                        │  (= AIMP Gateway)    │                │ Adapters │
└───────────────┘                        └──────────────────────┘                └──────────┘
```

An AIMP gateway SHOULD expose an MCP server interface. A compliant MCP client can then
discover and call AIMP tools with no AIMP-specific code. The gateway is responsible for
translating MCP `tools/call` invocations into the equivalent AIMP method calls from §01.

### 1.2 Tool Surface

The gateway MUST expose, at minimum, the following MCP tools:

| MCP tool name | AIMP method | Notes |
|---|---|---|
| `aimp.discover` | §01.6.1 `discover` | returns device list + domains |
| `aimp.quote` | §01.6.2 `quote` | returns `quote_id` + cost breakdown |
| `aimp.execute` | §01.6.3 `execute` | requires prior `quote_id` |
| `aimp.telemetry` | §01.6.4 `telemetry` | returns state + progress + media URLs |
| `aimp.abort` | §01.6.5 `abort` | never rate-limited |

The gateway MAY additionally expose per-domain convenience tools such as
`aimp.manufacturing.print.2d.quick_print` that bundle quote + execute for low-risk
routine operations with explicit budget caps. Such convenience tools MUST still follow
the quote-lock-execute lifecycle internally.

### 1.3 Example MCP `tools/list` Response

```json
{
  "tools": [
    {
      "name": "aimp.quote",
      "description": "Obtain a firm price quote for a physical job. Does NOT start any physical work.",
      "inputSchema": {
        "type": "object",
        "required": ["device_id", "domain", "payload"],
        "properties": {
          "device_id": {"type": "string"},
          "domain":    {"type": "string", "pattern": "^[a-z]+(\\.[a-z0-9_]+)+\\.v[0-9]+$"},
          "asset":     { "$ref": "https://aimp.dev/schemas/core/asset.json" },
          "payload":   { "description": "Validated against the schema for `domain`" },
          "logistics": { "$ref": "https://aimp.dev/schemas/core/logistics.json" },
          "budget_limit": {
            "type": "object",
            "properties": {
              "amount":   {"type": "number"},
              "currency": {"type": "string", "pattern": "^[A-Z]{3}$"}
            }
          }
        }
      }
    },
    {
      "name": "aimp.execute",
      "description": "Commit a prior quote and start physical execution. Irreversible without abort.",
      "inputSchema": { "$ref": "https://aimp.dev/schemas/core/execute-request.json" }
    }
    // ...
  ]
}
```

### 1.4 Resources and Prompts

MCP resources and prompts MAY also be used:

- **Resources:** an AIMP gateway SHOULD publish each device's current state and recent
  telemetry as MCP resources with URIs like `aimp://device/printer-01/state`.
- **Prompts:** gateways MAY publish starter prompts that guide LLMs through safe
  quote-then-execute flow for common domains.

## 2. Function-Calling Binding

For agents that use OpenAI / Anthropic / Gemini function-calling directly (without MCP),
the gateway publishes JSON-Schema tool definitions at:

```
GET /v1/tools.json
```

The result is an array of tool objects directly loadable into any compliant function-calling
runtime. It mirrors the MCP `tools/list` content but in the vendor-neutral JSON-Schema
form.

## 3. Agent-to-Agent (A2A) Topology

In a multi-agent system, AIMP plays exactly one role: **physical executor interface**.
All other coordination happens between agents.

### 3.1 Recommended Role Separation

| Agent | Responsibility | AIMP access? |
|---|---|---|
| **Design Agent** | Turns natural-language intent into CAD, images, G-code, recipes. | No. |
| **Review Agent** | Checks designs against physics, safety, regulations. | No. |
| **Finance Agent** | Holds the ledger. Signs `approval_token`s. | No. |
| **Executor Agent** | Talks to the AIMP gateway. The **only** AIMP caller. | **Yes.** |
| **Audit Agent** | Receives telemetry; runs vision checks; advises abort/resume. | Read-only. |

Centralizing AIMP access in the Executor Agent has three benefits:

1. **Auditability.** All physical calls funnel through one identifiable principal.
2. **Blast radius.** Prompt injection in the Design Agent cannot reach the machine.
3. **Consistency.** Rate limits and budgets apply to a known caller.

### 3.2 Representative Message Flow

```
User  ──"print a cyberpunk poster and ship to me"──▶ Design Agent
Design Agent  ──generates PNG + metadata──▶ Review Agent
Review Agent  ──approved──▶ Executor Agent
Executor Agent  ──aimp.quote──▶ AIMP Gateway            → quote: 18.20 USD
Executor Agent  ──quote──▶ Finance Agent
Finance Agent   ──approval_token──▶ Executor Agent
Executor Agent  ──aimp.execute(approval_token)──▶ AIMP Gateway
AIMP Gateway    ──state transitions──▶ Audit Agent (telemetry subscription)
Audit Agent     ──vision check at 50%──▶ Executor Agent (continue / abort)
AIMP Gateway    ──COMPLETED + tracking──▶ Executor Agent
Executor Agent  ──natural-language reply──▶ User
```

### 3.3 Cross-Protocol Contracts

Inter-agent messages are out of scope for AIMP. However, the `approval_token` passed from
Finance to Executor and on to the gateway is normative:

```json
{
  "approval_token": {
    "iss": "agent://org/finance-agent",
    "sub": "quote_01J8...",
    "iat": 1714000000,
    "exp": 1714003600,
    "max_amount": {"amount": 25.00, "currency": "USD"},
    "signature": "ed25519:..."
  }
}
```

Gateways MUST verify the signature against a configured trust anchor and reject tokens
whose `sub` does not match the quote being executed.

## 4. Worked Example: Poster from Prompt to Doorstep

A complete end-to-end flow using MCP + AIMP, with comments:

1. User: *"Make me an A3 cyberpunk poster, ship to my home. Budget $25."*
2. LLM generates a PNG, uploads it to object storage.
3. LLM calls `aimp.discover({"device_filter":{"domains":["manufacturing.print.2d.v1"]}})`.
   Gateway returns one cloud-print device.
4. LLM calls `aimp.quote` with the PNG URL and `payload = {paper_size: "A3", dpi: 300, color_mode: "CMYK"}`.
   Gateway returns `quote_id` and `estimated_cost.amount = 18.20`.
5. LLM checks `18.20 <= 25.00`, decides OK, calls `aimp.execute(quote_id)`.
6. Gateway transitions `LOCKED → EXECUTING`. LLM polls `aimp.telemetry` every 60 s.
7. At `FULFILLING`, gateway returns shipping `tracking_number`.
8. Gateway transitions `COMPLETED`. LLM replies to user with cost, tracking, and ETA.

The LLM never saw HTTP, never saw JSON-RPC, never saw MQTT. It saw five tools.

## 5. Security of the Binding

- MCP transports MUST be authenticated end-to-end (stdio + signed, or HTTPS + token).
- Tool descriptions in `tools/list` MUST NOT leak secrets; gateways return scrubbed
  schemas per caller.
- Tool results MUST NOT include full-resolution asset payloads. Use pre-signed URLs
  (§01.6.4).
- Agents SHOULD treat the output of `aimp.telemetry` media as **untrusted**: a compromised
  adapter could try prompt injection via rendered text in a camera frame. Vision models
  used for audit SHOULD be sandboxed with no tool access.

## 6. Non-Goals of This Binding

- AIMP does not define how agents negotiate who is the Executor in a multi-agent system.
  Use your A2A framework's role assignment.
- AIMP does not supply an agent framework. Use LangGraph, AutoGen, CrewAI, DSPy, or
  anything else; AIMP is the tool set they call.

## 7. Next Reading

- §04 Security & Cost — `approval_token` formats and budget enforcement.
- §05 Audit & Telemetry — processing audit frames safely.
