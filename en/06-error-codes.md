# AIMP 06 — Error Codes

**Status:** Draft · **Version:** 1.0.0-draft · **Normative**

AIMP defines a closed set of standard error codes. Gateways and adapters MUST use these
codes when surfacing failures through responses, telemetry, and webhooks. Vendor-specific
codes are permitted only under the `VENDOR_` prefix and MUST also carry a mapping to the
closest standard code.

## 1. Error Envelope

All error responses MUST follow this shape:

```json
{
  "error": {
    "code": "ERR_BUDGET_EXCEEDED",
    "message": "Quoted amount 28.40 USD exceeds remaining daily budget 14.60 USD.",
    "category": "policy",
    "retryable": false,
    "retry_after_seconds": null,
    "details": {
      "budget_ceiling": 200.00,
      "budget_consumed": 185.40,
      "quote_amount": 28.40,
      "currency": "USD"
    },
    "remediation": "Reduce job scope, wait for budget reset, or request ceiling increase.",
    "trace_id": "0f7a..."
  }
}
```

Fields:

- **`code`** — from the enumerated list below. MUST be one of the reserved values.
- **`message`** — human-readable, single sentence, agent-safe (no secrets).
- **`category`** — one of: `validation`, `auth`, `policy`, `resource`, `hardware`,
  `asset`, `network`, `safety`, `vendor`, `internal`.
- **`retryable`** — boolean; indicates whether the same request could succeed later.
- **`retry_after_seconds`** — OPTIONAL; present when a retry window is known.
- **`details`** — free-form, domain-appropriate diagnostics.
- **`remediation`** — actionable next step for the caller.
- **`trace_id`** — correlates with logs.

## 2. Standard Error Codes

### 2.1 Validation (`validation`)

| Code | HTTP | Meaning |
|---|---|---|
| `ERR_INVALID_ENVELOPE` | 400 | Common envelope is malformed or missing required fields. |
| `ERR_INVALID_DOMAIN` | 400 | `domain` namespace not recognized by the gateway. |
| `ERR_INVALID_PAYLOAD` | 400 | `payload` does not validate against the Domain Schema. |
| `ERR_VERSION_UNSUPPORTED` | 400 | `aimp_version` is incompatible with the gateway. |
| `ERR_DUPLICATE_IDEMPOTENCY_CONFLICT` | 409 | Same `idempotency_key` used with a different body. |

### 2.2 Authentication & Authorization (`auth`)

| Code | HTTP | Meaning |
|---|---|---|
| `ERR_UNAUTHENTICATED` | 401 | Missing or invalid credentials. |
| `ERR_TOKEN_EXPIRED` | 401 | Access token is past its `exp`. |
| `ERR_FORBIDDEN_DOMAIN` | 403 | Token not scoped for the requested `domain`. |
| `ERR_FORBIDDEN_DEVICE` | 403 | Token not scoped for the requested `device_id`. |
| `ERR_APPROVAL_REQUIRED` | 403 | Policy requires a human `approval_token` for this action. |
| `ERR_APPROVAL_INVALID` | 403 | `approval_token` signature invalid, expired, or scope mismatch. |

### 2.3 Policy (`policy`)

| Code | HTTP | Meaning |
|---|---|---|
| `ERR_BUDGET_EXCEEDED` | 402 | Executing the quote would exceed the budget ceiling. |
| `ERR_RATE_LIMITED` | 429 | Too many requests for this principal. `retry_after_seconds` REQUIRED. |
| `ERR_RISK_TIER_BLOCKED` | 403 | Domain risk tier is disabled for this environment. |
| `ERR_QUOTE_EXPIRED` | 410 | `quote_id` is past its `valid_until`. |
| `ERR_QUOTE_UNKNOWN` | 404 | `quote_id` does not exist. |

### 2.4 Resource (`resource`)

| Code | HTTP | Meaning |
|---|---|---|
| `ERR_DEVICE_NOT_FOUND` | 404 | `device_id` does not exist. |
| `ERR_DEVICE_BUSY` | 409 | Device is servicing another job; try later. |
| `ERR_DEVICE_OFFLINE` | 503 | Adapter unreachable. |
| `ERR_INSUFFICIENT_CONSUMABLE` | 409 | Requested material/consumable is not available in sufficient quantity. |
| `ERR_UNSUPPORTED_CAPABILITY` | 422 | Request combines parameters the device does not support (e.g., volume too large). |

### 2.5 Hardware (`hardware`)

| Code | HTTP | Meaning |
|---|---|---|
| `ERR_HARDWARE_FAULT` | 500 | Adapter reports device-side fault. |
| `ERR_HARDWARE_STALL` | 500 | Motion stall / mechanical jam. |
| `ERR_HARDWARE_OVERTEMP` | 500 | Thermal limit exceeded. |
| `ERR_HARDWARE_EMERGENCY_STOP` | 500 | A physical emergency stop was pressed. |
| `ERR_CALIBRATION_REQUIRED` | 503 | Device needs calibration / tool change / bed leveling. |

### 2.6 Asset (`asset`)

| Code | HTTP | Meaning |
|---|---|---|
| `ERR_ASSET_UNREACHABLE` | 502 | Could not fetch from `asset.url`. |
| `ERR_ASSET_HASH_MISMATCH` | 422 | Downloaded bytes did not match `hash_sha256`. |
| `ERR_ASSET_FORMAT_REJECTED` | 415 | Format not in the Domain Schema's acceptable list. |
| `ERR_ASSET_TOO_LARGE` | 413 | Asset exceeds gateway or device limits. |
| `ERR_ASSET_CONTENT_POLICY` | 451 | Asset violated content or safety policy. |

### 2.7 Network (`network`)

| Code | HTTP | Meaning |
|---|---|---|
| `ERR_UPSTREAM_TIMEOUT` | 504 | Upstream dependency (vendor API, storage) did not respond in time. |
| `ERR_WEBHOOK_DELIVERY_FAILED` | — | Emitted as telemetry only; the client did not ACK a webhook. |

### 2.8 Safety (`safety`)

| Code | HTTP | Meaning |
|---|---|---|
| `ERR_UNSAFE_PARAMETER` | 422 | Parameters would exceed machine-declared safe envelope. |
| `ERR_COLLISION_RISK` | 422 | Motion plan intersects forbidden zones or other active jobs. |
| `ERR_INTERLOCK_OPEN` | 503 | Safety interlock (door, light curtain) is not closed. |
| `ERR_VISION_AUDIT_FAILED` | — | Vision audit verdict was `failure`. Accompanies `ABORTED` state. |

### 2.9 Vendor (`vendor`)

| Code | HTTP | Meaning |
|---|---|---|
| `VENDOR_<name>_<code>` | varies | Vendor-specific error. MUST be accompanied by `standard_equivalent` in `details`. |

### 2.10 Internal (`internal`)

| Code | HTTP | Meaning |
|---|---|---|
| `ERR_INTERNAL` | 500 | Gateway bug; retry with backoff is permitted. |
| `ERR_SERVICE_DEGRADED` | 503 | Gateway operating in reduced mode; partial features available. |

## 3. Error Reporting Through Telemetry

A job that ends in `FAILED` MUST expose its error via telemetry:

```json
{
  "job_id": "01J8K9...",
  "state": "FAILED",
  "progress": 0.32,
  "updated_at": "2026-04-24T05:59:00Z",
  "error": {
    "code": "ERR_HARDWARE_OVERTEMP",
    "category": "hardware",
    "retryable": false,
    "details": {"channel": "extruder_temp", "value": 285.0, "limit": 260.0}
  }
}
```

A job that ends in `ABORTED` due to a safety condition MUST carry the applicable safety
code, e.g., `ERR_VISION_AUDIT_FAILED` or `ERR_INTERLOCK_OPEN`.

## 4. Error Recovery Guidance for Agents

A conforming AI client SHOULD implement the following recovery patterns:

| Category | Suggested behaviour |
|---|---|
| `validation` | Do not retry; fix the request. |
| `auth` | Refresh tokens; if still failing, escalate to operator. |
| `policy` | If `ERR_BUDGET_EXCEEDED`, do not retry; seek approval or reduce scope. If `ERR_RATE_LIMITED`, honor `retry_after_seconds`. |
| `resource` | For `ERR_DEVICE_BUSY`, schedule retry with jitter; for `ERR_INSUFFICIENT_CONSUMABLE`, notify operator. |
| `hardware` | Surface to human; agents SHOULD NOT automatically re-execute hardware-faulted jobs more than once per fault. |
| `asset` | Regenerate or fetch the asset; recompute hash; retry. |
| `network` | Exponential backoff with jitter up to a reasonable ceiling. |
| `safety` | NEVER auto-retry. Require explicit human intervention. |
| `internal` | Exponential backoff. |

## 5. Extension by Domain

Domain Schemas MAY define additional domain-specific codes under a namespaced prefix,
e.g., `ERR_FDM_NOZZLE_CLOG` registered under `manufacturing.additive.fdm.v1`. These MUST
also map to one of the standard categories above for generic handling.

## 6. Localization

Error `message` text SHOULD be localized based on the `Accept-Language` request header.
The `code` field MUST remain in its canonical ASCII form regardless of locale.
