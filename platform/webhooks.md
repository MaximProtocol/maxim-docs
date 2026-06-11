# Webhooks

Maxim Protocol can send HTTP POST notifications to your server when payment events occur. Use webhooks to react to policy violations, budget thresholds, and spend anomalies without polling the API.

---

## Registering a webhook

Webhooks are attached to agents via the alerts API. Each alert targets one event type and one notification channel.

```bash
curl -X POST https://api.maximprotocol.com/v1/alerts \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "my-agent",
    "type": "policy_violation",
    "notify": { "type": "webhook", "url": "https://hooks.yourapp.com/maxim" }
  }'
```

You can register multiple alerts per agent, including multiple webhooks for different event types.

---

## Event types

| Alert type | Fires when |
|---|---|
| `policy_violation` | Any payment is rejected by a spend policy |
| `budget_pct` | Agent spend reaches a percentage of its daily budget |
| `spend_rate` | Agent spend exceeds a USDC threshold within a time window |

### `policy_violation`

Fires on every rejected payment. No threshold configuration required.

```bash
curl -X POST https://api.maximprotocol.com/v1/alerts \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "my-agent",
    "type": "policy_violation",
    "notify": { "type": "webhook", "url": "https://hooks.yourapp.com/maxim" }
  }'
```

### `budget_pct`

Fires when the agent has consumed a specified percentage of its daily budget. Setting `threshold: 80` gives you a heads-up before the budget is exhausted.

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

### `spend_rate`

Fires when the agent spends more than `threshold` USDC within the specified `window`.

```bash
curl -X POST https://api.maximprotocol.com/v1/alerts \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "my-agent",
    "type": "spend_rate",
    "threshold": 5,
    "window": "1h",
    "notify": { "type": "webhook", "url": "https://hooks.yourapp.com/maxim" }
  }'
```

---

## Webhook payloads

All webhook deliveries share the same envelope: an `event` field identifying the type, and an `agentId` and `timestamp` on every payload.

### `policy.violation`

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

Possible `reason` values: `per_call_limit_exceeded`, `daily_budget_exceeded`, `domain_not_allowed`, `rate_limit_exceeded`.

### `budget.warning`

```json
{
  "event": "budget.warning",
  "agentId": "my-agent",
  "threshold": 80,
  "usedPct": 81.2,
  "usedUsdc": 8.12,
  "dailyBudgetUsdc": 10.00,
  "timestamp": "2026-05-09T14:23:01Z"
}
```

### `spend_rate.exceeded`

```json
{
  "event": "spend_rate.exceeded",
  "agentId": "my-agent",
  "window": "1h",
  "spentUsdc": 6.41,
  "thresholdUsdc": 5.00,
  "timestamp": "2026-05-09T14:23:01Z"
}
```

---

## Signature verification

Every webhook delivery includes an `X-MaximProtocol-Signature` header. The value is `sha256=<HMAC-SHA256 of the raw request body>`, signed with your webhook secret.

Retrieve your webhook secret from **Settings → Webhooks** in the dashboard.

Always verify the signature before processing a webhook. Reject requests where the signature does not match.

**Bash:**

```bash
EXPECTED="sha256=$(echo -n "$RAW_BODY" | openssl dgst -sha256 -hmac "$WEBHOOK_SECRET" | awk '{print $2}')"
[ "$SIGNATURE_HEADER" = "$EXPECTED" ] && echo "Verified" || echo "Invalid"
```

**Node.js:**

```typescript
import { createHmac, timingSafeEqual } from "crypto";

function verifyWebhook(rawBody: string, signature: string, secret: string): boolean {
  const expected = "sha256=" + createHmac("sha256", secret).update(rawBody).digest("hex");
  return timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}
```

Use a constant-time comparison (`timingSafeEqual`) to prevent timing attacks.

---

## Retry behavior

If your endpoint does not return a `2xx` status within 10 seconds, Maxim Protocol retries the delivery with exponential backoff:

| Attempt | Delay after failure |
|---|---|
| 1st retry | 1 minute |
| 2nd retry | 5 minutes |
| 3rd retry | 30 minutes |
| 4th retry | 2 hours |
| Give up | Event is dropped |

After the 4th retry, the event is dropped and logged in the dashboard under **Settings → Webhooks → Delivery log**. Failed deliveries do not affect payment processing.

---

## Notification channels

In addition to webhooks, alerts support email and Slack:

```json
{ "type": "email", "address": "you@example.com" }
```

```json
{ "type": "slack", "url": "https://hooks.slack.com/services/..." }
```

Slack uses an incoming webhook URL. The payload is formatted as a Slack message block automatically.

---

## Further reading

- [Spend policies](../features/spend-policies.md)
- [Observability](../features/observability.md)
- [Security](security.md)
