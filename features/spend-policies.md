# Spend Policies

Spend policies define the rules under which an agent is permitted to make payments. They are enforced at the Maxim Protocol gateway before any transaction is submitted to the blockchain. A call that would breach a policy is rejected before funds leave the wallet.

---

## Setting policies

```bash
curl -X PUT https://api.maximprotocol.com/v1/policies/my-agent \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "dailyBudget": { "usdc": 100 },
    "perCallLimit": { "usdc": 1.00 },
    "allowedDomains": ["api.dune.com", "api.browserbase.com", "fal.ai"],
    "rateLimit": { "calls": 500, "window": "1h" }
  }'
```

---

## Policy fields

### `dailyBudget`

**Type:** `{ usdc: number }` — **Optional**

Maximum USDC the agent can spend in a rolling 24-hour window. The window resets 24 hours after the first payment of the current period.

```json
{ "usdc": 100 }
```

When 80% of the daily budget is consumed, Maxim Protocol emits a `budget.warning` webhook event. At 100%, all subsequent payments are rejected with `DailyBudgetExceededError` until the window resets.

Set `dailyBudget` to `null` to remove the limit.

---

### `perCallLimit`

**Type:** `{ usdc: number }` — **Optional**

Maximum USDC the agent can spend on a single `POST /v1/pay` call. Payments that would exceed this limit are rejected before the x402 or MPP handshake begins.

```json
{ "usdc": 1.00 }
```

This is a hard cap per individual transaction, independent of the daily budget.

---

### `allowedDomains`

**Type:** `string[]` — **Optional**

Whitelist of domains the agent is permitted to pay. Calls to domains not on this list are rejected with `DomainNotAllowedError` before any payment is attempted.

```json
["api.dune.com", "api.browserbase.com", "fal.ai"]
```

Subdomain matching: `"api.dune.com"` allows only that exact subdomain. `"dune.com"` allows `dune.com` and all subdomains.

Omit this field to allow payments to any domain. Use this with caution — unrestricted domains mean any endpoint the agent resolves to can be paid.

---

### `blockedDomains`

**Type:** `string[]` — **Optional**

Explicit blocklist that takes precedence over `allowedDomains`. A domain on the blocklist is always rejected, even if it is also on the allowlist.

```json
["api.legacy-service.com"]
```

---

### `rateLimit`

**Type:** `{ calls: number, window: string }` — **Optional**

Maximum number of payment calls within a time window. This limits how frequently the agent can make payments, regardless of the amount.

```json
{ "calls": 500, "window": "1h" }
```

Valid window values: `"1m"`, `"5m"`, `"15m"`, `"1h"`, `"24h"`.

---

### `protocolPreference`

**Type:** `"auto" | "x402" | "mpp"` — **Default:** `"auto"`

Controls which protocol the payment router prefers when a service supports both x402 and MPP.

```json
"auto"
```

- `"auto"` — x402 preferred, falls back to MPP
- `"x402"` — always use x402 if available
- `"mpp"` — always use MPP if available

---

## Viewing active policies

```bash
curl https://api.maximprotocol.com/v1/policies/my-agent \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

```json
{
  "agentId": "my-agent",
  "dailyBudget": { "usdc": 100, "usedToday": 12.40 },
  "perCallLimit": { "usdc": 1.00 },
  "allowedDomains": ["api.dune.com", "api.browserbase.com", "fal.ai"],
  "blockedDomains": [],
  "rateLimit": { "calls": 500, "window": "1h", "usedThisHour": 47 },
  "protocolPreference": "auto"
}
```

---

## Policy violation events

When a policy rejects a payment, the event is:
- Logged in the dashboard under the **Policy Events** tab
- Available via `GET /v1/transactions?agentId=...&status=rejected`
- Sent as a webhook if you have configured a `policy.violation` webhook

Webhook payload:

```json
{
  "event": "policy.violation",
  "agentId": "my-agent",
  "reason": "per_call_limit_exceeded",
  "detail": "Requested 2.50 USDC, limit is 1.00 USDC",
  "endpoint": "https://api.expensive-service.com/query",
  "timestamp": "2026-05-09T14:23:01Z"
}
```

---

## Budget alerts

Set a webhook to be notified when the daily budget reaches a threshold:

```bash
curl -X POST https://api.maximprotocol.com/v1/alerts \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "my-agent",
    "type": "budget_pct",
    "threshold": 80,
    "notify": { "type": "webhook", "url": "https://hooks.yourapp.com/maxim" }
  }'
```

This fires when the agent has consumed 80% of its daily budget.

---

## Example: locked-down research agent

A tightly scoped agent that can only query specific data APIs and cannot spend more than $5 per day:

```bash
curl -X PUT https://api.maximprotocol.com/v1/policies/my-research-agent \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "dailyBudget": { "usdc": 5 },
    "perCallLimit": { "usdc": 0.25 },
    "allowedDomains": ["api.dune.com", "api.coingecko.com", "api.thegraph.com"],
    "rateLimit": { "calls": 100, "window": "1h" },
    "protocolPreference": "x402"
  }'
```

---

## Further reading

- [Agent wallets](agent-wallets.md)
- [Observability](observability.md)
- [REST API reference](../platform/architecture.md)
