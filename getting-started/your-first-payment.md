# Your First Payment

This guide walks through the full lifecycle of making an agent payment with Maxim Protocol: creating a wallet, setting spend policies, paying for an API service, and reading the on-chain record.

---

## What we are building

An agent that:
- Holds its own non-custodial Solana wallet
- Operates under a daily budget and a per-call spend limit
- Pays for a real data API call using x402
- Logs every payment to Solana with a verifiable transaction hash

---

## 1. Create the wallet

```bash
curl -X POST https://api.maximprotocol.com/v1/wallets \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "agentId": "research-agent" }'
```

```json
{
  "agentId": "research-agent",
  "address": "7xKXtg2xpAGMq8KLwpSodE9LFKq8Z1EscezSShpump",
  "balance": { "usdc": 0 },
  "network": "solana-mainnet"
}
```

The wallet is a program-derived address (PDA) on Solana. Only the agent's keypair can authorize payments from it. Maxim Protocol cannot move funds without a valid signed instruction.

Fund the wallet by transferring USDC to the address above, or via the dashboard at [app.maximprotocol.com](https://app.maximprotocol.com).

---

## 2. Set spend policies

Policies are checked at the gateway layer before any payment is submitted. A call that would breach the daily budget is rejected before funds leave the wallet.

```bash
curl -X PUT https://api.maximprotocol.com/v1/policies/research-agent \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "dailyBudget": { "usdc": 25 },
    "perCallLimit": { "usdc": 1.00 },
    "allowedDomains": ["api.dune.com", "api.browserbase.com", "fal.ai"],
    "rateLimit": { "calls": 200, "window": "1h" }
  }'
```

Verify the policy is active:

```bash
curl https://api.maximprotocol.com/v1/policies/research-agent \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

```json
{
  "agentId": "research-agent",
  "dailyBudget": { "usdc": 25 },
  "perCallLimit": { "usdc": 1.00 },
  "allowedDomains": ["api.dune.com", "api.browserbase.com", "fal.ai"],
  "rateLimit": { "calls": 200, "window": "1h" },
  "protocolPreference": "auto"
}
```

---

## 3. Make a payment

Call the REST API to make a payment:

```bash
curl -X POST https://api.maximprotocol.com/v1/pay \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "research-agent",
    "endpoint": "https://api.dune.com/v1/query/1234/results",
    "method": "GET"
  }'
```

```json
{
  "status": 200,
  "protocol": "x402",
  "amountPaid": { "usdc": 0.002 },
  "txHash": "5yJ8kXtg2xp...mQ9pL",
  "data": { }
}
```

The gateway handles everything: reading the 402 response, constructing the payment payload, signing with the agent's wallet, and retrying the original request with payment attached. The API response comes back in `data` as if the payment never happened.

---

## 4. Handle policy rejections

If a call would exceed the spend policy, Maxim Protocol rejects it before any funds are spent. The API returns a `402` with a structured error body:

```bash
curl -X POST https://api.maximprotocol.com/v1/pay \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "research-agent",
    "endpoint": "https://api.expensive-service.com/query",
    "method": "POST"
  }'
```

```json
{
  "error": "policy_violation",
  "reason": "per_call_limit_exceeded",
  "detail": "Requested 2.50 USDC, limit is 1.00 USDC"
}
```

Policy violations are also logged to the dashboard and can trigger webhook alerts.

---

## 5. View the on-chain record

List recent transactions for the agent:

```bash
curl "https://api.maximprotocol.com/v1/transactions?agentId=research-agent&limit=10" \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

```json
{
  "transactions": [
    {
      "txHash": "5yJ8kXtg2xp...mQ9pL",
      "protocol": "x402",
      "amountPaid": { "usdc": 0.002 },
      "endpoint": "api.dune.com/v1/query/1234/results",
      "timestamp": "2026-05-09T14:23:01Z"
    }
  ]
}
```

Inspect a single transaction:

```bash
curl https://api.maximprotocol.com/v1/transactions/5yJ8kXtg2xp...mQ9pL \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

```json
{
  "txHash": "5yJ8kXtg2xp...mQ9pL",
  "agentId": "research-agent",
  "protocol": "x402",
  "amountPaid": { "usdc": 0.002 },
  "endpoint": "api.dune.com/v1/query/1234/results",
  "policyCheck": "passed",
  "parentTxHash": null,
  "solanaExplorer": "https://explorer.solana.com/tx/5yJ8kXtg2xp...mQ9pL",
  "timestamp": "2026-05-09T14:23:01Z"
}
```

The `solanaExplorer` link goes to the public explorer entry for the transaction. The ledger is independent of Maxim Protocol. Even if Maxim Protocol went offline, your payment history would remain on-chain.

---

## 6. View the dashboard

The dashboard at [app.maximprotocol.com](https://app.maximprotocol.com) shows real-time spend, protocol split, policy events, and a live transaction feed with links to Solana Explorer for every entry.

---

## Next steps

- [How the payment router selects protocols](../features/payment-router.md)
- [Full spend policy reference](../features/spend-policies.md)
- [Building multi-agent payment chains](../features/multi-agent-payments.md)
- [REST API reference](../platform/architecture.md)
