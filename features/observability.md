# Observability

Maxim Protocol records every payment on Solana and surfaces a real-time view of your agent fleet's spend, protocol usage, policy events, and transaction history in the dashboard. The blockchain is the primary source of truth. The dashboard is a lens on it.

---

## Dashboard

The dashboard is available at [app.maximprotocol.com](https://app.maximprotocol.com).

### Live transaction feed

The default view shows every payment as it settles, in real time. Each entry includes:

- Agent ID
- Protocol used (x402 or MPP)
- Amount paid (USDC)
- Endpoint called
- Solana transaction hash (links to Solana Explorer)
- Time

### Spend breakdown

The **Spend** tab shows USDC spend broken down by:

- Agent
- Protocol (x402 vs. MPP)
- Time window (1h, 24h, 7d, 30d, custom)
- Domain

Use this to identify which agents are spending the most and where that spend is going.

### Protocol split

The **Protocol** tab shows x402 vs. MPP volume over time. Useful for understanding which protocols your agent's services use and whether the protocol preference setting is having the desired effect.

### Policy events

The **Policies** tab shows all rejected payments with:

- Rejection reason (daily budget exceeded, per-call limit, domain not allowed, rate limit)
- Endpoint that triggered the rejection
- Amount that was requested vs. the limit
- Agent ID and timestamp

### On-chain ledger

The **Ledger** tab shows every transaction with a direct link to its Solana Explorer entry. The data in this view is derived from on-chain state. It cannot be modified by Maxim Protocol.

---

## Querying transactions

```bash
# List recent transactions for an agent
curl "https://api.maximprotocol.com/v1/transactions?agentId=my-agent&limit=50" \
  -H "Authorization: Bearer $MAXIM_API_KEY"

# Filter by protocol
curl "https://api.maximprotocol.com/v1/transactions?agentId=my-agent&protocol=x402" \
  -H "Authorization: Bearer $MAXIM_API_KEY"

# Filter by time window
curl "https://api.maximprotocol.com/v1/transactions?agentId=my-agent&since=24h" \
  -H "Authorization: Bearer $MAXIM_API_KEY"

# Filter by domain
curl "https://api.maximprotocol.com/v1/transactions?agentId=my-agent&domain=api.dune.com" \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

```json
{
  "transactions": [
    {
      "txHash": "5yJ8kX...mQ9pL",
      "protocol": "x402",
      "amountPaid": { "usdc": 0.002 },
      "endpoint": "api.dune.com/v1/query/...",
      "timestamp": "2026-05-09T14:23:01Z"
    },
    {
      "txHash": "3xH7jY...nP8qM",
      "protocol": "mpp",
      "amountPaid": { "usdc": 0.012 },
      "endpoint": "api.browserbase.com/v1/sessions",
      "timestamp": "2026-05-09T14:20:01Z"
    }
  ]
}
```

Inspect a single transaction:

```bash
curl https://api.maximprotocol.com/v1/transactions/5yJ8kX...mQ9pL \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

```json
{
  "txHash": "5yJ8kX...mQ9pL",
  "agentId": "my-agent",
  "protocol": "x402",
  "amountPaid": { "usdc": 0.002 },
  "endpoint": "api.dune.com/v1/query/1234/results",
  "policyCheck": "passed",
  "parentTxHash": null,
  "solanaExplorer": "https://explorer.solana.com/tx/5yJ8kX...mQ9pL",
  "timestamp": "2026-05-09T14:23:01Z"
}
```

---

## Spend summary

Use the metrics endpoint to retrieve aggregated spend data:

```bash
curl "https://api.maximprotocol.com/v1/agents/my-agent/metrics?metrics=spend,tx_count,protocol_split&window=7d" \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

```json
{
  "agentId": "my-agent",
  "window": "7d",
  "metrics": {
    "spend": { "total": 8.42, "byDomain": {
      "api.dune.com": 3.20,
      "api.browserbase.com": 2.14,
      "fal.ai": 1.92,
      "api.coingecko.com": 1.16
    }},
    "tx_count": 312,
    "protocol_split": { "x402": 0.729, "mpp": 0.271 }
  }
}
```

---

## Alerts

Set threshold-based alerts to be notified before problems become incidents.

### Budget threshold alert

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

Fires when the agent has consumed 80% of its daily budget.

### Spend rate alert

```bash
curl -X POST https://api.maximprotocol.com/v1/alerts \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "my-agent",
    "type": "spend_rate",
    "threshold": 5,
    "window": "1h",
    "notify": { "type": "email", "address": "you@example.com" }
  }'
```

Fires when the agent spends more than 5 USDC in any 1-hour window.

### Policy violation alert

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

Fires on every rejected payment.

**Notification channels:**

- Webhook (POST JSON payload to your URL)
- Email
- Slack (via webhook URL)

---

## Metrics API

All dashboard data is available via the REST API for integration into external monitoring infrastructure.

```
GET /v1/agents/{agentId}/metrics
  ?metrics=spend,tx_count,policy_violations,protocol_split
  &window=24h
  &resolution=1h
Authorization: Bearer $MAXIM_API_KEY
```

Response:

```json
{
  "agentId": "my-agent",
  "window": "24h",
  "resolution": "1h",
  "metrics": {
    "spend": [
      { "timestamp": "2026-05-09T00:00:00Z", "value": 1.24 },
      { "timestamp": "2026-05-09T01:00:00Z", "value": 0.98 }
    ],
    "protocol_split": {
      "x402": 0.72,
      "mpp": 0.28
    },
    "policy_violations": [
      { "timestamp": "2026-05-09T03:00:00Z", "count": 3, "reason": "per_call_limit_exceeded" }
    ]
  }
}
```

---

## Solana as the source of truth

The Maxim Protocol dashboard is a view on the on-chain ledger. Every transaction hash links to a public Solana Explorer entry that contains:

- Sender address (agent wallet)
- Receiver address (service wallet)
- USDC amount
- Block timestamp
- Program instruction data (including protocol and endpoint hash)

This data cannot be modified by Maxim Protocol. If you ever need to verify a transaction independently of the dashboard, use the Solana transaction hash directly. The `solanaExplorer` field in every transaction response provides the direct link.

---

## Further reading

- [Spend policies](spend-policies.md)
- [Multi-agent payment chains](multi-agent-payments.md)
- [REST API reference](../platform/architecture.md)
