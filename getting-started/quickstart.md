# Quickstart

Make your first on-chain agent payment in under five minutes.

---

## Prerequisites

- A Maxim Protocol account at [app.maximprotocol.com](https://app.maximprotocol.com)

---

## Step 1: Get your API key

After signing in, go to **Settings → API Keys** in the dashboard and create an API key. Set it as an environment variable:

```bash
export MAXIM_API_KEY=rp_live_...
```

---

## Step 2: Create an agent wallet

Every agent that makes payments needs a wallet. Wallets are non-custodial Solana program-derived addresses owned by the agent's keypair.

```bash
curl -X POST https://api.maximprotocol.com/v1/wallets \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "agentId": "my-agent" }'
```

```json
{
  "agentId": "my-agent",
  "address": "7xKXtg2xpAGMq8KLwpSodE9LFKq8Z1EscezSShpump",
  "balance": { "usdc": 0 },
  "network": "solana-mainnet"
}
```

---

## Step 3: Fund the wallet

Top up via the dashboard at [app.maximprotocol.com](https://app.maximprotocol.com), or transfer USDC directly to the wallet address shown above.

To check balance at any time:

```bash
curl https://api.maximprotocol.com/v1/wallets/my-agent \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

```json
{
  "agentId": "my-agent",
  "balance": { "usdc": 50 }
}
```

---

## Step 4: Set spend policies

Policies are enforced at the gateway before any payment is submitted. Set them once.

```bash
curl -X PUT https://api.maximprotocol.com/v1/policies/my-agent \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "dailyBudget": { "usdc": 10 },
    "perCallLimit": { "usdc": 0.50 },
    "allowedDomains": ["api.dune.com", "api.browserbase.com"]
  }'
```

---

## Step 5: Make your first payment

```bash
curl -X POST https://api.maximprotocol.com/v1/pay \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "my-agent",
    "endpoint": "https://api.dune.com/v1/query/1234/results",
    "method": "GET"
  }'
```

```json
{
  "status": 200,
  "protocol": "x402",
  "amountPaid": { "usdc": 0.002 },
  "txHash": "5yJ8kX...mQ9pL",
  "data": { }
}
```

Maxim Protocol detects the payment protocol automatically and handles the full handshake. You do not need to know whether the service uses x402 or MPP.

---

## Step 6: View the on-chain record

Every payment is recorded on Solana. View your transaction history:

```bash
curl "https://api.maximprotocol.com/v1/transactions?agentId=my-agent" \
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
      "timestamp": "2026-05-18T14:23:01Z"
    }
  ]
}
```

Click the transaction hash in the dashboard to open the Solana Explorer entry directly.

---

## Next steps

- [How the payment router works](../features/payment-router.md)
- [Agent wallet reference](../features/agent-wallets.md)
- [Full spend policy options](../features/spend-policies.md)
- [REST API reference](../platform/architecture.md)
