# Payment Router

The Maxim Protocol payment router sits between your agent and every paid API endpoint. It detects which payment protocol a service requires, handles the full negotiation, and returns the API response. Your agent calls one method. The protocol is invisible.

---

## How detection works

When `POST /v1/pay` is called, the router sends the original request to the target endpoint. If the response is HTTP 402, the router inspects the response body to determine which protocol to use:

| Indicator | Protocol selected |
|---|---|
| `X-PAYMENT-REQUIREMENTS` header with `scheme` field | x402 |
| Response body contains `"protocol": "mpp/1.0"` | MPP |
| Both indicators present | x402 (default preference) |
| Neither indicator present | `PaymentProtocolError` returned |

Detection happens in under 5ms. It adds no meaningful latency to the payment flow.

---

## Protocol preference

By default, Maxim Protocol prefers x402 when a service supports both protocols. x402 is simpler, requires no session negotiation on the first call, and settles directly on-chain without an intermediary.

Override the preference per agent via the policies API:

```bash
curl -X PUT https://api.maximprotocol.com/v1/policies/my-agent \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "protocolPreference": "mpp" }'
```

Valid values: `"auto"` (default), `"x402"`, `"mpp"`.

---

## The `POST /v1/pay` endpoint

```
POST /v1/pay
Authorization: Bearer $MAXIM_API_KEY
Content-Type: application/json

{
  "agentId": "my-agent",
  "endpoint": "https://api.service.com/resource",
  "method": "GET",
  "headers": { "Custom-Header": "value" },
  "body": { },
  "idempotencyKey": "unique-key"
}
```

`headers` and `body` are optional. `idempotencyKey` is optional and prevents duplicate payments on retry.

**Response:**

```json
{
  "status": 200,
  "protocol": "x402",
  "amountPaid": { "usdc": 0.002 },
  "txHash": "5yJ8kX...mQ9pL",
  "solanaExplorer": "https://explorer.solana.com/tx/5yJ8kX...mQ9pL",
  "data": { },
  "sessionId": "sess_01j9..."
}
```

`sessionId` is only present for MPP responses where a session was established or reused.

---

## Idempotency

To prevent duplicate payments if your code retries a failed request, pass an `idempotencyKey`:

```bash
curl -X POST https://api.maximprotocol.com/v1/pay \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "my-agent",
    "endpoint": "https://api.service.com/resource",
    "method": "POST",
    "body": { "query": "..." },
    "idempotencyKey": "run-abc123-step-3"
  }'
```

If a payment with the same key was completed within the last 24 hours, Maxim Protocol returns the cached result without charging again.

---

## Retry behavior

If the payment succeeds but the API returns a non-2xx status, the router does not retry automatically. The payment was made; the error belongs to the API.

If the payment itself fails (e.g. network error before the request reaches the facilitator), the router retries with exponential backoff:

| Attempt | Delay |
|---|---|
| 1st retry | 500ms |
| 2nd retry | 1s |
| 3rd retry | 2s |
| Give up | `PaymentError` returned |

A failed payment does not deduct from the agent's wallet unless on-chain confirmation was received.

---

## Session management

For MPP services, the router maintains a session cache per agent per service domain. Sessions reduce per-request overhead by allowing payment authorization to be reused across multiple calls.

View active sessions:

```bash
curl "https://api.maximprotocol.com/v1/sessions?agentId=my-agent" \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

Clear sessions for a service (forces renegotiation on next call):

```bash
curl -X DELETE "https://api.maximprotocol.com/v1/sessions?agentId=my-agent&domain=api.service.com" \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

---

## Error types

| Error | Cause |
|---|---|
| `PolicyViolationError` | Payment would breach a spend policy |
| `InsufficientFundsError` | Agent wallet balance too low |
| `PaymentProtocolError` | Endpoint returned 402 but with an unrecognized protocol |
| `PaymentError` | Payment failed after all retries |
| `DomainNotAllowedError` | Endpoint domain is not on the agent's allowlist |

---

## Further reading

- [x402 protocol deep-dive](x402.md)
- [MPP protocol deep-dive](mpp.md)
- [Spend policies](spend-policies.md)
- [REST API reference](../platform/architecture.md)
