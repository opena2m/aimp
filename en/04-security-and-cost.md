# AIMP 04 — Security & Cost Management

**Status:** Draft · **Version:** 1.0.0-draft · **Normative**

Physical actions are irreversible. Prompt injection, model hallucination, and simple
mistakes become expensive, destructive, or dangerous the moment they touch matter. This
chapter specifies the minimum security and cost controls an AIMP gateway MUST enforce.

## 1. Threat Model

AIMP gateways defend against:

| # | Threat | Mitigation(s) |
|---|---|---|
| T1 | Runaway agent (hallucinated loop, infinite print runs) | Budget ceiling, rate limits, quote-lock lifecycle. |
| T2 | Prompt injection through assets or telemetry | Asset hashing, sandboxed audit vision, scrubbed tool schemas. |
| T3 | Compromised adapter sending false "success" | Gateway-verified telemetry, required sensor readings, signed webhooks. |
| T4 | Stolen or leaked client credentials | Short-lived tokens, per-device scope, anomaly detection. |
| T5 | Physical hazard (fire, injury, chemical release) | Risk tiers, human-in-the-loop gates, abort primacy. |
| T6 | Privacy breach via telemetry media | Short-lived pre-signed URLs, per-tenant data partitioning. |

## 2. Authentication

- Clients MUST authenticate on every call. Unauthenticated requests MUST return HTTP 401
  with no information disclosure in the body.
- The RECOMMENDED scheme is OAuth 2.1 with:
  - short-lived access tokens (≤1 hour),
  - refresh tokens bound to device fingerprint,
  - per-scope tokens (e.g., `aimp:quote`, `aimp:execute:manufacturing.print.2d.v1`).
- Mutual TLS (mTLS) is RECOMMENDED between gateway and adapters.
- Webhooks MUST be HMAC-signed (§01.7).

## 3. Authorization Model

### 3.1 Principals

Three principal types exist and MUST be distinguishable in the audit log:

- **agent** — an autonomous AI principal (`agent://org/agent-name`).
- **human** — a natural-person principal identified by stable user ID.
- **system** — the gateway itself or a trusted automated policy.

An **approval_token** (§03.3.3) binds the three: an `agent` requests, a `human`
authorizes, the `system` enforces.

### 3.2 Policy Evaluation

On every `execute` call, the gateway MUST evaluate in order:

1. **Token validity** — signature, expiry, scope.
2. **Domain allowlist** — is this token permitted to call this `domain`?
3. **Device allowlist** — is this token permitted to drive this `device_id`?
4. **Budget check** — is the quote amount within the remaining budget?
5. **Risk tier policy** — does this domain require human approval?
6. **Rate limit** — are we within the per-token ceiling?

Any failure MUST abort the call with a specific error code (§06) and MUST be logged.

## 4. Risk Tiers

Every Domain Schema declares a risk tier. Gateways MAY override upward (more cautious) but
MUST NOT override downward.

| Tier | Examples | Default policy |
|---|---|---|
| **routine** | 2D printing, label making, simple thermal cycling | Agent-only approval; budget ceiling enforced. |
| **restricted** | 3D printing with flammable filament, CNC, food preparation served to humans | Human confirmation REQUIRED above a monetary threshold (default 50 USD equivalent) OR on any asset not previously audited. |
| **hazardous** | High-power lasers, chemical reactors, biological handling, anything involving motion near humans | Human confirmation REQUIRED on every `execute`. Aborts MUST be one-touch from a physical stop button in addition to `/abort`. |

Gateways SHOULD surface the risk tier in `discover` so the client can reason about policy
before spending cycles on planning.

## 5. The Cost Lifecycle

Every physical action flows through five accounting states. Gateways MUST maintain a
per-principal **ledger** recording each.

```
estimate ──▶ quote ──▶ reserve ──▶ commit ──▶ settle
                                     │
                                     └──▶ partial_refund (on abort)
```

### 5.1 Quote

The gateway produces a price with a stated `valid_until`. During its validity window the
gateway MUST honor the quote unless the underlying asset is revoked or fails a content
check.

### 5.2 Reserve

On `execute`, the gateway reserves:

- the monetary amount (hold on a payment instrument, or decrement of an internal budget);
- the consumables (deduct virtually from stock);
- the machine time slot (reject overlapping reservations).

Reservations MUST expire if `EXECUTING` is not reached within a reasonable deadline
(RECOMMENDED: 5 minutes). On expiry the held amount MUST be released.

### 5.3 Commit

When execution begins, the reservation becomes a commitment. Consumables are now
physically engaged.

### 5.4 Settle

On `COMPLETED`, the gateway records final cost, which may differ from the quote due to
metered reality (slightly more filament, slightly longer time). Gateways MUST NOT exceed
the quoted `amount` by more than a declared tolerance (RECOMMENDED: 10%). Overages beyond
tolerance are the gateway's to absorb.

### 5.5 Partial Refund on Abort

On `ABORTED`, the gateway calculates `final_cost` from the fraction of execution that
completed plus any fixed setup costs. Returned under `final_cost` in the abort response
(§01.6.5).

## 6. Budget Enforcement

Every token carries a **budget context**:

```json
{
  "budget": {
    "currency": "USD",
    "period": "daily",            // "per_job" | "daily" | "monthly" | "total"
    "ceiling": 200.00,
    "consumed": 37.40,
    "resets_at": "2026-04-25T00:00:00Z"
  }
}
```

Gateways MUST:

- Reject `execute` calls that would push `consumed` past `ceiling`.
- Expose remaining budget via `GET /v1/budget` for self-inspection by agents.
- Emit a webhook event `budget_warning` at 80% consumption and `budget_exhausted` at 100%.

## 7. Human-in-the-Loop (HITL)

HITL is not a feature; in AIMP it is a **state**.

When policy requires a human, the gateway transitions to `AUDITING` with
`human_action_required` populated:

```json
{
  "human_action_required": {
    "reason": "policy.risk_tier.hazardous",
    "approval_url": "https://approval.example.com/review/01J8...",
    "expires_at": "2026-04-24T06:30:00Z",
    "fallback_action": "ABORT"       // "ABORT" | "HOLD"
  }
}
```

The human reviewer fetches the asset, the quote, the audit media (if any), and signs an
approval token. The agent submits the token via `POST /v1/jobs/{id}/resume`. If the
window expires, the gateway executes `fallback_action`.

Gateways SHOULD integrate with common approval transports: email with one-time links,
Slack/Lark/DingTalk interactive messages, internal review portals. Approval signatures
MUST be verifiable by the gateway; simple "clicked a button" is not sufficient.

## 8. Asset Integrity

- Every digital asset referenced in a `quote` or `execute` MUST carry `hash_sha256` or an
  equivalent content hash.
- Adapters MUST verify the hash after download. Mismatch is a hard error
  (`ERR_ASSET_HASH_MISMATCH`).
- Assets fetched from client-provided URLs MUST be fetched via the gateway when possible
  to prevent adapter-side SSRF.

## 9. Abort Primacy

Aborts are a safety primitive:

- MUST be accepted from any authenticated principal with `aimp:abort` scope.
- MUST NOT be rate-limited.
- MUST NOT require a valid `approval_token`.
- MUST be acknowledged by the adapter before the gateway returns success.
- SHOULD be mirrored to a physical stop mechanism (button / cord) on hazardous devices.

An abort that cannot be confirmed by the adapter within a bounded time (RECOMMENDED: 3
seconds) MUST escalate: the gateway marks the job `FAILED` and surfaces a pager-grade
alert.

## 10. Logging & Audit

Gateways MUST maintain an append-only audit log with, per event:

- monotonic sequence number,
- UTC timestamp,
- principal (agent / human / system),
- action (method, state transition, policy decision),
- relevant IDs (`job_id`, `quote_id`, `device_id`),
- an integrity chain field (e.g., running hash of prior entries).

Audit logs MUST be retained for the regulatory minimum of the operating jurisdiction
(RECOMMENDED: at least 1 year). They MUST be queryable via `GET /v1/audit` by
appropriately authorized principals.

## 11. Privacy

- Telemetry media (camera frames) frequently contain private content (a person walking
  past the workshop). Gateways MUST:
  - store media in tenant-isolated buckets,
  - redact or blur on request,
  - honor deletion requests within the jurisdiction's deadline.
- Recipient addresses in the `logistics` block are PII. Gateways MUST encrypt them at rest
  and MUST NOT expose them through `GET /v1/audit` to non-authorized principals.

## 12. Next Reading

- §05 Audit & Telemetry — the multimodal feedback loop in detail.
- §06 Error Codes — how enforcement failures are reported.
