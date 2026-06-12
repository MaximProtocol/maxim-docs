# Error Reference

All API errors return a `4xx` or `5xx` HTTP status with a JSON body. The `error` field is a machine-readable code you can branch on in your application. The `detail` field is a human-readable description intended for logging.

```json
{
  "error": "policy_violation",
  "reason": "per_call_limit_exceeded",
  "detail": "Requested 2.50 USDC, limit is 1.00 USDC"
}
```

---

## HTTP status codes

| Status | When it occurs |
|---|---|
| `400 Bad Request` | Malformed request body or missing required field |
| `401 Unauthorized` | API key missing, malformed, or revoked |
| `402 Payment Required` | Payment was rejected before funds were spent (see policy errors below) |
| `403 Forbidden` | Per-agent key used for an account-level operation |
| `404 Not Found` | Agent ID, transaction hash, or resource does not exist |
| `409 Conflict` | Duplicate idempotency key with a different request body |
| `429 Too Many Requests` | Account-level API rate limit exceeded |
| `500 Internal Server Error` | Internal error; safe to retry with backoff |

---

## Payment errors

These errors are returned on `POST /v1/pay` when a payment cannot be completed.

### `policy_violation`

A spend policy blocked the payment before any funds were spent. The `reason` field identifies which policy was triggered.

```json
{
  "error": "policy_violation",
  "reason": "per_call_limit_exceeded",
  "detail": "Requested 2.50 USDC, limit is 1.00 USDC"
}
```

| `reason` | Policy that fired |
|---|---|
| `per_call_limit_exceeded` | The required payment exceeds `perCallLimit` |
| `daily_budget_exceeded` | The agent has hit its daily budget for the rolling 24h window |
| `domain_not_allowed` | The target domain is not in `allowedDomains` |
| `domain_blocked` | The target domain is in `blockedDomains` |
| `rate_limit_exceeded` | Too many payment calls within the `rateLimit` window |

Policy violations are logged in the dashboard under **Policy Events** and can trigger [webhook alerts](webhooks.md).

---

### `insufficient_funds`

The agent wallet does not have enough USDC to cover the payment. The `required` field shows how much was needed; `available` shows the current balance.

```json
{
  "error": "insufficient_funds",
  "detail": "Wallet balance 0.80 USDC, payment requires 1.00 USDC",
  "required": { "usdc": 1.00 },
  "available": { "usdc": 0.80 }
}
```

---

### `payment_protocol_error`

The target endpoint returned `402` but the response body did not match either the x402 or MPP schema. This usually means the service uses a payment mechanism Maxim Protocol does not support, or the 402 was returned for a non-payment reason.

```json
{
  "error": "payment_protocol_error",
  "detail": "Endpoint returned 402 with unrecognized payment schema"
}
```

---

### `payment_failed`

The payment was attempted but failed after all retries. This can happen if the x402 facilitator or MPP payment endpoint was unreachable. No funds were deducted unless on-chain confirmation was received.

```json
{
  "error": "payment_failed",
  "detail": "Payment failed after 3 attempts: facilitator timeout"
}
```

See [retry behavior](../features/payment-router.md) for the backoff schedule.

---

### `budget_exceeded`

Returned by `POST /v1/agents/invoke` when a sub-agent's cumulative spend hits the `budget` cap passed in the request.

```json
{
  "error": "budget_exceeded",
  "detail": "Sub-agent spend reached budget cap of 2.00 USDC"
}
```

---

## Authentication errors

### `unauthorized`

Returned when the `Authorization` header is missing, malformed, or the key has been revoked.

```json
{
  "error": "unauthorized",
  "detail": "Invalid or missing API key"
}
```

---

### `forbidden`

Returned when a per-agent key (`rp_agent_...`) is used to call an endpoint that requires an account-level key (`rp_live_...`), such as creating new agents or modifying account settings.

```json
{
  "error": "forbidden",
  "detail": "Per-agent key cannot perform account-level operations"
}
```

---

## Rate limit errors

### `rate_limited`

Returned when the account-level API rate limit is exceeded. The `retryAfter` field is the number of seconds to wait before retrying.

```json
{
  "error": "rate_limited",
  "detail": "API rate limit exceeded",
  "retryAfter": 12
}
```

---

## Idempotency errors

### `idempotency_conflict`

Returned when a request uses an `idempotencyKey` that was previously used with a different request body. Change the key or use the original request body.

```json
{
  "error": "idempotency_conflict",
  "detail": "Idempotency key already used with a different request body"
}
```

---

## Further reading

- [Payment Router](../features/payment-router.md) — retry behavior and protocol detection
- [Spend Policies](../features/spend-policies.md) — configuring policy limits
- [Webhooks](webhooks.md) — getting notified when policy violations occur
