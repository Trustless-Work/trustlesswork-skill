---
description: >
  Trustless Work Core API v2 — authentication, escrow lifecycle, transaction
  signing, trustline creation, error taxonomy, and agent wiring.
label: Core API
path: trustless-work-dev/skills/api/v2
scope: api-agent-orchestration
type: reference
version: "2.0.0"
---

# Trustless Work Core API v2 Reference

> The Core API skill gives an AI agent everything needed to author valid
> Stellar transactions, interact with the Trustless Work v2 REST API, and
> recover from domain errors — without hard-coding any contract-specific logic.

---

## 1 · Authentication (OAuth 2.0 + PKCE)

Agents MUST authenticate with the Trustless Work API using OAuth 2.0 with
PKCE (`code_challenge_method=S256`).  The authorization flow is described in
detail in the [Authenticating a Trustless Work Agent](../authenticating-an-agent.md)
skill.

The minimum required scope for core API operations is `api:read api:write`.

**Authorization URL**

```
https://auth.stellar.org/authorize
  ?client_id=<app-client-id>
  &redirect_uri=<registered-redirect-uri>
  &response_type=code
  &code_challenge=<S256-challenge>
  &code_challenge_method=S256
  &scope=api:read api:write
```

**Token endpoint**

```
POST https://auth.stellar.org/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=<auth-code>
&code_verifier=<original-verifier>
&redirect_uri=<registered-redirect-uri>
&client_id=<app-client-id>
```

The response yields an `access_token` (short-lived) and a `refresh_token`.
Refresh tokens are single-use and rotated on every refresh.

**Authorization header**

```
Authorization: Bearer <access_token>
```

### 1.1 · Scoped access tokens

Each Trustless Work API resource is protected by a scope.  The table below
lists the scopes an agent will most commonly need.

| Scope             | Access granted                              |
| ----------------- | ------------------------------------------- |
| `api:read`        | Read your own accounts and balances.        |
| `api:write`       | Create / update resources you own.          |
| `escrow:manage`   | Create escrows, trigger completions, refund.|
| `tx:submit`       | Submit Stellar transactions on your behalf. |
| `trustline:create`| Create trustlines in your name.             |
| `agent:bind`      | Bind an agent key to your identity.         |

An agent that requests only the scopes it needs at the moment reduces the
blast radius of a leaked token.

### 1.2 · Refreshing a token

When the access token expires (default TTL: **15 minutes**), call the token
endpoint again with `grant_type=refresh_token`.

```
POST https://auth.stellar.org/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=<refresh-token>
&client_id=<app-client-id>
```

A new `access_token` and a new (rotated) `refresh_token` are returned.
Discard the old refresh token immediately — it is invalidated.

---

## 2 · Escrow Lifecycle

An **escrow** is a time-locked Stellar payment that can be completed or
refunded.  The escrow service exposes a small, composable REST API.

### 2.1 · Create an escrow

`POST /escrow/escrows`

```json
{
  "asset": "XLM",
  "amount": "100.0000000",
  "sender": "GBXYZ...",
  "receiver": "GABC...",
  "receiver_asset": "XLM",
  "condition": {
    "type": "timelock",
    "condition_data": {
      "unlock_at": "2026-10-31T00:00:00Z"
    }
  }
}
```

The response returns the escrow object including its `id`, a `stellar_tx_hash`
(or `null` until submitted), and links to the completion / refund endpoints.

### 2.2 · Complete an escrow

`POST /escrow/escrows/{escrow_id}/complete`

Optional body (used by condition types that require a proof):

```json
{
  "condition_data": { ... }
}
```

The API returns the completed escrow object.  If the condition was a
time-lock, the server verifies the current time before proceeding.

### 2.3 · Refund an escrow

`POST /escrow/escrows/{escrow_id}/refund`

Returns the refunded escrow object.  A refund may only be issued while the
escrow is still in `open` state and the lock has not expired (unless the
sender has enabled early-refund).

### 2.4 · Poll escrow state

`GET /escrow/escrows/{escrow_id}`

Use this endpoint when an agent must wait for an asynchronous completion
(e.g., waiting for a Stellar transaction to be confirmed).  The response
includes the current `state` field.

---

## 3 · Transaction Signing

Agents often need to sign Stellar transactions for users who have not
directly exposed their secret keys.  The Trustless Work API provides a
**signing proxy** endpoint.

### 3.1 · Sign a transaction

`POST /transactions/sign`

Request body:

```json
{
  "network_passphrase": "Test SDF Network ; September 2015",
  "source": "GABC...",
  "envelope_xdr": "AAAA..."
}
```

The response contains the signed envelope XDR:

```json
{
  "signed_envelope_xdr": "AAAA...SIGNATURE"
}
```

The agent must then submit the signed envelope via `POST
/transactions/submit`.

### 3.2 · Submit a transaction

`POST /transactions/submit`

```json
{
  "signed_envelope_xdr": "AAAA...SIGNATURE"
}
```

Response:

```json
{
  "hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "status": "PENDING"
}
```

Poll `GET /transactions/{hash}` to check the final status.

---

## 4 · Trustline Creation

Before an account can hold a non-native asset, it must have a trustline.
The API provides a convenient wrapper around the Stellar `ChangeTrust`
operation.

### 4.1 · Create a trustline

`POST /trustlines`

```json
{
  "account": "GBXYZ...",
  "asset": {
    "type": "credit_alphanum4",
    "code": "USDC",
    "issuer": "GA5ZSEJYB37JRC5AVCIA5MOPRQH2KSENAQKKTKCECDAUURCXKHHJHUBP"
  },
  "limit": "1000000.0000000"
}
```

The response includes the resulting `transaction_hash` and the new trustline
object.

---

## 5 · Error Taxonomy

All Trustless Work API errors follow the **Stellar envelope error model**: a
JSON object containing a `type` (machine-readable identifier) and a
`message` (human-readable description).  The `type` value is stable across
API versions; only `message` may change wording.

Error objects look like:

```json
{
  "type": "ESCROW_RECEIVER_TRUSTLINE_MISSING",
  "message": "The receiver account does not have a trustline for the escrow asset.",
  "detail": {
    "account": "GABC...",
    "asset_code": "USDC",
    "asset_issuer": "GA5Z..."
  }
}
```

### 5.1 · Error groups

Errors are grouped by the domain they belong to.  The group determines which
documentation URL is authoritative.  Agents SHOULD always render the full
grouped URL (see §5.2) because the ungrouped variant is deprecated and may
return a 404.

| Group    | Path prefix            | Example error codes                    |
| -------- | ---------------------- | -------------------------------------- |
| `escrow` | `/errors/escrow/`      | `ESCROW_RECEIVER_TRUSTLINE_MISSING`, `ESCROW_CONDITION_EXPIRED`, `ESCROW_STATE_INVALID` |
| `token`  | `/errors/token/`       | `TOKEN_TRUSTLINE_MISSING`, `TOKEN_LIMIT_EXCEEDED`, `TOKEN_ISSUER_UNKNOWN` |
| `auth`   | `/errors/auth/`        | `AUTHENTICATION_REQUIRED`, `INSUFFICIENT_SCOPE`, `TOKEN_EXPIRED` |
| `tx`     | `/errors/tx/`          | `TX_SUBMIT_FAILED`, `TX_EXPIRED`, `TX_PRECONDITION_FAILED` |

### 5.2 · Documentation URLs

Every error `type` has a companion documentation page.  The canonical URL
pattern is:

```
https://docs.trustlesswork.com/trustless-work/v2-en/api-rest/errors/<group>/<code-in-kebab-case>
```

where `<group>` is the error-group name from the table above and
`<code-in-kebab-case>` is the error type with underscores replaced by
hyphens and lowercased.

**Example — escrow error:**

```
Type: ESCROW_RECEIVER_TRUSTLINE_MISSING
URL:  https://docs.trustlesswork.com/trustless-work/v2-en/api-rest/errors/escrow/escrow-receiver-trustline-missing
```

**Example — token error:**

```
Type: TOKEN_TRUSTLINE_MISSING
URL:  https://docs.trustlesswork.com/trustless-work/v2-en/api-rest/errors/token/token-trustline-missing
```

Agents SHOULD NOT use the ungrouped pattern
`/errors/<code-in-kebab-case>` because those pages have been retired and
return HTTP 404.

### 5.3 · Handling errors as an agent

When the API returns a non-2xx response, the agent MUST:

1. Read the `type` field to determine the error group.
2. Render the grouped documentation URL (from §5.2) and show it to the user
   so they can look up the meaning.
3. Attempt recovery only when the error type is in the **recoverable** set:

   | Recoverable types                                      | Suggested action                                      |
   | ------------------------------------------------------ | ----------------------------------------------------- |
   | `ESCROW_CONDITION_EXPIRED`                             | Notify the sender; suggest reopening the escrow.      |
   | `TOKEN_TRUSTLINE_MISSING`                              | Prompt the user to create the missing trustline.      |
   | `TX_EXPIRED`                                           | Rebuild the transaction with a fresh sequence + memo. |
   | `AUTHENTICATION_REQUIRED`                              | Re-authenticate via the OAuth flow.                   |

   All other types are **fatal** for the current operation and should be
   surfaced to the user with the documentation URL.

---

## 6 · Agent Wiring

This skill is intended to be composed with other skills in an agent
toolkit.  Below is a minimal wiring example for an agent that uses the
Core API to create an escrow.

### 6.1 · Tool definition (OpenAI function-calling schema)

```json
{
  "name": "trustless_work_create_escrow",
  "description": "Create a time-locked escrow on Stellar via the Trustless Work API.",
  "parameters": {
    "type": "object",
    "properties": {
      "asset": { "type": "string", "enum": ["XLM", "USDC", "EURC"] },
      "amount": { "type": "string", "pattern": "^\\\\d+(\\\\.\\\\d{1,7})?$" },
      "sender": { "type": "string", "description": "Stellar source account (G...)"},
      "receiver": { "type": "string", "description": "Stellar receiver account (G...)"},
      "receiver_asset": { "type": "string" },
      "unlock_at": { "type": "string", "format": "date-time", "description": "ISO 8601 UTC timestamp after which the escrow may be completed."}
    },
    "required": ["asset", "amount", "sender", "receiver", "unlock_at"]
  }
}
```

### 6.2 · Implementation sketch (Python)

```python
import httpx
from datetime import datetime, timezone

BASE = "https://api.trustlesswork.com/v2"

def create_escrow(access_token: str, payload: dict) -> dict:
    headers = {"Authorization": f"Bearer {access_token}"}
    resp = httpx.post(f"{BASE}/escrow/escrows", json=payload, headers=headers)
    resp.raise_for_status()
    return resp.json()

def handle_error(resp: httpx.Response) -> None:
    if resp.status_code < 400:
        return
    body = resp.json()
    err_type = body.get("type")
    msg = body.get("message", "Unknown error")
    group = _group_for_type(err_type)
    doc_url = (
        f"https://docs.trustlesswork.com/trustless-work/v2-en/api-rest/errors/"
        f"{group}/{err_type.lower().replace('_', '-')}"
    )
    raise RuntimeError(f"{err_type}: {msg} — see {doc_url}")

def _group_for_type(error_type: str) -> str:
    mapping = {
        "ESCROW_": "escrow",
        "TOKEN_": "token",
        "AUTH_":  "auth",
        "TX_":    "tx",
    }
    for prefix, group in mapping.items():
        if error_type.startswith(prefix):
            return group
    return "general"
```

### 6.3 · Error-example response body

```json
{
  "type": "ESCROW_RECEIVER_TRUSTLINE_MISSING",
  "message": "Receiver lacks a trustline for the escrow asset.",
  "detail": {
    "receiver": "GABC123...",
    "asset_code": "USDC",
    "asset_issuer": "GA5Z..."
  },
  "_links": {
    "documentation": {
      "href": "https://docs.trustlesswork.com/trustless-work/v2-en/api-rest/errors/escrow/escrow-receiver-trustline-missing"
    }
  }
}
```

Note the `_links.documentation.href` field — it contains the **grouped**
URL.  Agents MUST use the `_links.documentation.href` value when available
rather than constructing the URL manually.

---

## 7 · Rate Limits & Best Practices

* The API enforces a per-account rate limit of **60 requests / minute**.
* Escrow polling (`GET /escrow/escrows/{id}`) should use exponential
  back-off with a maximum interval of 30 seconds.
* Always include an `Idempotency-Key` header on `POST` requests that modify
  state (escrow creation, completion, refund, trustline creation).

---

## 8 · Related Skills

| Skill | Purpose |
| ----- | ------- |
| [Authenticating an Agent](../authenticating-an-agent.md) | OAuth 2.0 + PKCE flow |
| [Escrow Agent Patterns](../patterns/escrow-agent.md) | State-machine wiring for escrow workflows |
| [Transaction Builder](../sdk/transaction-builder.md) | Low-level Stellar transaction construction |
