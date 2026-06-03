# Multi-Agent Payments

When an orchestrating agent delegates work to sub-agents, those sub-agents incur their own payment costs. Maxim Protocol tracks the full payment chain on-chain: every sub-agent payment is linked to the parent transaction, and the entire cost tree is auditable from a single root transaction hash.

---

## How it works

Each agent has its own wallet and its own spend policies. When Agent A invokes Agent B and Agent B makes payments:

1. Agent B's payments are settled on-chain from Agent B's wallet
2. Each of Agent B's transactions includes a `parentTxHash` field linking it to the invocation from Agent A
3. The dashboard and REST API can resolve the full transaction tree from any node

This means you can trace the total cost of any task, across any number of agent hops, from a single root transaction hash.

---

## Invoking a sub-agent with a budget cap

```bash
curl -X POST https://api.maximprotocol.com/v1/agents/invoke \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Agent-Id: orchestrator-agent" \
  -d '{
    "agentEndpoint": "https://research-agent.yourplatform.com/run",
    "task": {
      "query": "Summarize Solana TVL trends for Q1 2026",
      "outputFormat": "markdown"
    },
    "budget": { "usdc": 2.00 }
  }'
```

```json
{
  "output": "...",
  "totalCost": { "usdc": 0.84 },
  "paymentChain": ["5yJ8kX...mQ9pL", "3aB2cD...nR7qK"]
}
```

The `budget` field enforces a hard cap on the sub-agent's spend for this invocation. If the sub-agent's payments would exceed the budget, further payment calls are rejected and the sub-agent receives a `BudgetExceededError`.

---

## On-chain transaction chain

Every transaction in a multi-agent chain is linked on-chain via the `parentTxHash` field stored in the Maxim Protocol program's ledger account. The chain is queryable from either end.

```bash
curl https://api.maximprotocol.com/v1/transactions/5yJ8kX...mQ9pL/chain \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

```json
{
  "root": "5yJ8kX...mQ9pL",
  "chain": [
    {
      "txHash": "5yJ8kX...mQ9pL",
      "agentId": "orchestrator-agent",
      "amountPaid": { "usdc": 0 },
      "depth": 0
    },
    {
      "txHash": "3aB2cD...nR7qK",
      "agentId": "research-agent",
      "protocol": "x402",
      "amountPaid": { "usdc": 0.002 },
      "endpoint": "api.dune.com/v1/query/1234/results",
      "depth": 1
    },
    {
      "txHash": "7cE4fG...pS2hJ",
      "agentId": "research-agent",
      "protocol": "mpp",
      "amountPaid": { "usdc": 0.012 },
      "endpoint": "api.browserbase.com/v1/sessions",
      "depth": 1
    }
  ]
}
```

---

## Sequential pipeline

Chain agents where each step uses the output of the previous one. Issue each `POST /v1/agents/invoke` call sequentially, passing the `output` from the previous response into the next request's `task` body:

```bash
# Step 1: fetch raw data
curl -X POST https://api.maximprotocol.com/v1/agents/invoke \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Agent-Id: orchestrator-agent" \
  -d '{
    "agentEndpoint": "https://data-agent.yourplatform.com/run",
    "task": { "source": "dune", "queryId": "1234" },
    "budget": { "usdc": 0.50 }
  }'

# Step 2: analyze — include the output from step 1 in the task body
curl -X POST https://api.maximprotocol.com/v1/agents/invoke \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Agent-Id: orchestrator-agent" \
  -d '{
    "agentEndpoint": "https://analyst-agent.yourplatform.com/run",
    "task": { "data": "<output from step 1>" },
    "budget": { "usdc": 1.00 }
  }'

# Step 3: write report — include the analysis from step 2
curl -X POST https://api.maximprotocol.com/v1/agents/invoke \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Agent-Id: orchestrator-agent" \
  -d '{
    "agentEndpoint": "https://writer-agent.yourplatform.com/run",
    "task": { "analysis": "<output from step 2>", "format": "markdown" },
    "budget": { "usdc": 0.50 }
  }'
```

The total cost across the three steps is visible in the dashboard, aggregated under the orchestrator's run ID.

---

## Concurrent fan-out

Run multiple sub-agents in parallel by issuing the `POST /v1/agents/invoke` requests concurrently from your orchestrator:

```bash
# Issue all three requests concurrently (shown here sequentially for clarity)
curl -X POST https://api.maximprotocol.com/v1/agents/invoke \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Agent-Id: orchestrator-agent" \
  -d '{
    "agentEndpoint": "https://research-agent.yourplatform.com/run",
    "task": { "topic": "Solana ecosystem Q1 2026" },
    "budget": { "usdc": 1.00 }
  }'

curl -X POST https://api.maximprotocol.com/v1/agents/invoke \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Agent-Id: orchestrator-agent" \
  -d '{
    "agentEndpoint": "https://research-agent.yourplatform.com/run",
    "task": { "topic": "Ethereum ecosystem Q1 2026" },
    "budget": { "usdc": 1.00 }
  }'

curl -X POST https://api.maximprotocol.com/v1/agents/invoke \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Agent-Id: orchestrator-agent" \
  -d '{
    "agentEndpoint": "https://research-agent.yourplatform.com/run",
    "task": { "topic": "Bitcoin ecosystem Q1 2026" },
    "budget": { "usdc": 1.00 }
  }'
```

Total budget exposure: 3.00 USDC. Actual cost per sub-agent is visible on-chain.

---

## Agent-to-agent invoicing

For marketplace scenarios where one agent charges another for its services, Maxim Protocol supports direct agent-to-agent payment using x402 or MPP. The payee agent exposes an HTTP endpoint that returns a 402 response. The paying agent calls `POST /v1/pay` against that endpoint:

```bash
curl -X POST https://api.maximprotocol.com/v1/pay \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "paying-agent",
    "endpoint": "https://specialist-agent.marketplace.com/run",
    "method": "POST",
    "body": { "task": "classify this document" }
  }'
```

The specialist agent's endpoint returns a 402 with x402 requirements. Maxim Protocol handles the payment handshake and returns the classification result. See [x402: accepting payments](x402.md) for the server-side response format.

---

## Viewing payment trees in the dashboard

The dashboard at [app.maximprotocol.com](https://app.maximprotocol.com) shows multi-agent payment chains in a tree view. Select any transaction to expand its children and see the full cost breakdown at each level.

Via the REST API:

```bash
curl https://api.maximprotocol.com/v1/transactions/<root-tx-hash>/tree \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

```json
{
  "root": {
    "agentId": "orchestrator-agent",
    "amountPaid": { "usdc": 0 },
    "children": [
      {
        "agentId": "research-agent",
        "amountPaid": { "usdc": 0.002 },
        "protocol": "x402",
        "endpoint": "api.dune.com/..."
      },
      {
        "agentId": "research-agent",
        "amountPaid": { "usdc": 0.012 },
        "protocol": "mpp",
        "endpoint": "api.browserbase.com/..."
      }
    ]
  },
  "totalCost": { "usdc": 0.019 },
  "txCount": 4
}
```

---

## Further reading

- [Agent wallets](agent-wallets.md)
- [Spend policies](spend-policies.md)
- [Observability](observability.md)
- [REST API reference](../platform/architecture.md)
