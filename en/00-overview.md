# AIMP 00 — Overview & Architecture

**Status:** Draft · **Version:** 1.0.0-draft · **Audience:** all readers

## 1. Abstract

The **AI-to-Machine Protocol (AIMP)** is a standardized protocol set that allows AI systems
to produce tangible, real-world products through heterogeneous physical machines —
printers, 3D printers, CNC mills, robotic arms, laboratory instruments, cooking appliances,
drones, and beyond — under uniform semantics for capability discovery, cost quoting,
execution, multimodal audit, and logistics fulfillment.

AIMP is deliberately **not** a new transport protocol. It is a semantic layer designed to
ride on top of existing AI-tool protocols (MCP, A2A, OpenAI-compatible function calling)
and existing transport protocols (HTTP/REST, MQTT, WebSocket). Its only contribution is a
disciplined contract between *AI intent* and *physical execution*.

## 2. Problem Statement

Modern large language models can generate images, 3D models, G-code, CAD drawings, circuit
layouts, chemical recipes, and action plans. Turning any of this into physical reality today
requires either:

- Bespoke glue code per device / per vendor, with no shared semantics for cost, safety,
  or observability; or
- Single-vendor walled gardens (cloud print-on-demand, proprietary factories) with no
  interoperability.

The two missing primitives are:

1. A **uniform lifecycle contract** — so an AI agent can drive a 3D printer the same way
   it drives a pipetting robot or a laser cutter.
2. A **safety envelope** — so autonomous agents cannot bankrupt, damage, or injure by
   mistake. Physical actions are irreversible; prompt injection and model hallucination
   become consequential.

AIMP addresses both.

## 3. Design Principles

### 3.1 Micro-kernel + Domain Plugins

A tiny, strictly domain-agnostic **Core Protocol** (§01) handles the universal concerns —
capability discovery, quoting, execution handshake, telemetry, abort — of any physical job.
Everything domain-specific (nozzle temperatures, joint angles, pipette volumes, cutting
feed rates) lives in **Domain Schemas** (§02) organized under hierarchical namespaces such
as `manufacturing.additive.fdm.v1` or `kinematics.robotic_arm.v2`.

This is the same split that made POSIX, OpenGL, and the web stack durable: a stable kernel
plus versioned extensions.

### 3.2 Zero-Trust by Default

Every physical action passes through a **Quote → Lock → Execute → Audit → Settle** pipeline.
An AI may never jump straight to execution. The gateway tracks:

- monetary cost
- consumables (grams of filament, milliliters of reagent, kWh, paper, ink, …)
- wall-clock time
- risk class (irreversible? involves humans? involves hazardous material?)

Budgets are enforced at the gateway, not on the honor system of the agent. Human-in-the-loop
checkpoints are first-class protocol states, not afterthoughts.

### 3.3 Asynchronous and Observable

Physical time is orders of magnitude slower than compute time. AIMP is asynchronous
end-to-end: the agent POSTs intent and receives a `job_id`; it then polls `/telemetry`
or subscribes to webhooks for state transitions. Every job exposes:

- a normalized `progress` number in `[0.0, 1.0]`
- a normalized `state` from the core state machine
- optional multimodal telemetry: camera frames, sensor readings, logs

### 3.4 Embed, Don't Reinvent

AIMP does **not** try to compete with MCP, A2A, OpenAI tools, or LangGraph. Instead:

- To an agent calling via **MCP**, an AIMP gateway is just another MCP server offering
  tools like `aimp.quote`, `aimp.execute`, `aimp.telemetry`.
- In an **A2A** (agent-to-agent) swarm, AIMP is the *only* interface the dedicated
  "Executor Agent" is allowed to touch. Design, audit, finance, and dispatch agents talk
  among themselves in A2A; they bottleneck through AIMP to touch the world.
- AIMP requests can be carried over any transport capable of bidirectional JSON exchange.

### 3.5 Closed-Loop Sensing

The protocol is not write-only. Gateways **MUST** be able to stream back multimodal
telemetry: camera snapshots, sensor series, process logs. This enables:

- AI-side vision verification ("does the part match the CAD model?")
- Early abort on defects (the classic FDM "spaghetti failure")
- Adaptive parameter tuning in subsequent jobs

## 4. Architecture

AIMP defines three logical components and four layers in the broader stack.

### 4.1 Components

```
┌──────────────────────┐       ┌──────────────────────┐       ┌──────────────────────┐
│   AI Client / Agent  │──────▶│   AIMP Gateway       │──────▶│  Hardware Adapter    │
│  (LLM + tools)       │◀──────│   (Core protocol)    │◀──────│  (ROS/G-code/Modbus) │
│                      │       │                      │       │                      │
│  - intent            │       │  - routing           │       │  - translates AIMP   │
│  - budget policy     │       │  - auth & policy     │       │    JSON → machine    │
│  - vision audit      │       │  - quote/lock/exec   │       │  - sensor readback   │
│                      │       │  - telemetry relay   │       │                      │
└──────────────────────┘       └──────────────────────┘       └──────────────────────┘
         ▲                               │                               │
         │                               ▼                               ▼
         │                        ┌──────────────┐                ┌────────────────┐
         │                        │   Ledger &   │                │  Physical      │
         └────── webhooks ────────│   Audit Log  │                │  Machine(s)    │
                                  └──────────────┘                └────────────────┘
```

**AI Client / Agent.** The brain. Holds the user's intent, the budget, and the vision
models used for audit. Calls AIMP tools via MCP or function-calling bindings.

**AIMP Gateway.** The spinal cord. A stateful service that authenticates callers, enforces
policy, issues quotes, locks resources, dispatches executions, relays telemetry, and writes
the audit log. The gateway is the single place where money is actually spent.

**Hardware Adapter.** The nerves. Runs at or near the device. Translates validated AIMP
payloads into device-native languages (G-code, PostScript, ROS actions, OPC-UA, Modbus, or
vendor REST APIs). Reports back sensor telemetry.

### 4.2 Stack Layering

| Layer | Name | Concerns | Typical Protocol |
|---|---|---|---|
| L7 | Intent | natural-language goal | human / LLM prompt |
| L6 | Collaboration | multi-agent planning, review | A2A, AutoGen, Swarm |
| L5 | Tool connection | function call packaging | **MCP**, OpenAI tools |
| L4 | Physical orchestration | quote / lock / execute / audit | **AIMP Core (§01)** |
| L3 | Domain action | nozzle temp, joint angle, volume | **AIMP Domain Schemas (§02)** |
| L2 | Device driver | pulses and bytes | ROS, G-code, Modbus, OPC-UA |
| L1 | Physics | atoms and photons | — |

AIMP occupies strictly L4 + L3. It is an **abstraction layer**, not a competitor to any
other row.

## 5. Relationship to Adjacent Standards

- **MCP (Model Context Protocol).** AIMP gateways SHOULD expose themselves as MCP servers
  so any MCP-capable agent can discover and use them without hand-coding. See §03.
- **OpenAI / Anthropic / Gemini function calling.** The AIMP core methods map 1:1 onto
  function tools. A JSON-Schema export of the core is published under `schemas/core/`.
- **A2A / Agent-to-Agent protocols.** AIMP is the "physical executor" role in any A2A
  topology. See §03.
- **Industry control stacks (ROS, OPC-UA, MTConnect).** AIMP sits *above* these. Hardware
  adapters are the bridge.
- **Print-on-demand APIs (Printful, Printify, Alibaba cloud manufacturing, etc.).** These
  are *one instance* of an AIMP gateway, specialized to the 2D-print domain.

## 6. Versioning & Conformance Levels

AIMP versions follow semantic versioning on the document set. An implementation declares
conformance at one of three levels:

- **L1 Core.** Implements all five core methods in §01. No audit media, no budget
  enforcement. Minimum bar.
- **L2 Standard.** L1 plus quote-lock-execute-settle cost pipeline and basic telemetry
  (progress + state).
- **L3 Full.** L2 plus multimodal audit (camera/sensor streams), human-in-the-loop
  checkpoints, and conformance to at least one published Domain Schema.

Gateways advertise their level through `GET /capabilities` (§01.4).

## 7. Out of Scope

AIMP does **not** specify:

- Which transport to use (HTTP/REST is RECOMMENDED; MQTT/WebSocket is permitted).
- Identity or key management (bring your own OAuth2 / mTLS / signed tokens).
- Pricing for any specific consumable (gateways self-declare).
- The internal planning algorithm of any agent (that's the agent's business).

## 8. Next Reading

- §01 Core Protocol — the five verbs, the state machine, the request envelope.
- §02 Domain Extensions — how to write and register a Domain Schema.
- §03 AI Protocol Integration — MCP and A2A bindings, worked example.
