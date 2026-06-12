# TypeScript Examples

The Maxim Protocol API is a standard REST API. These examples use the native `fetch` API available in Node.js 18+ and all modern runtimes.

---

## Setup

```typescript
const BASE_URL = "https://api.maximprotocol.com/v1";
const API_KEY = process.env.MAXIM_API_KEY!;

async function maxim(path: string, options: RequestInit = {}) {
  const res = await fetch(`${BASE_URL}${path}`, {
    ...options,
    headers: {
      Authorization: `Bearer ${API_KEY}`,
      "Content-Type": "application/json",
      ...options.headers,
    },
  });

  const body = await res.json();
  if (!res.ok) throw Object.assign(new Error(body.detail ?? "API error"), { body });
  return body;
}
```

---

## Create an agent wallet

```typescript
const wallet = await maxim("/wallets", {
  method: "POST",
  body: JSON.stringify({ agentId: "my-agent" }),
});

console.log(wallet.address); // 7xKXtg2xpAGMq8KLwpSodE9LFKq8Z1EscezSShpump
console.log(wallet.network); // solana-mainnet
```

---

## Set spend policies

```typescript
await maxim("/policies/my-agent", {
  method: "PUT",
  body: JSON.stringify({
    dailyBudget: { usdc: 25 },
    perCallLimit: { usdc: 1.00 },
    allowedDomains: ["api.dune.com", "api.browserbase.com", "fal.ai"],
    rateLimit: { calls: 200, window: "1h" },
  }),
});
```

---

## Make a payment

```typescript
const result = await maxim("/pay", {
  method: "POST",
  body: JSON.stringify({
    agentId: "my-agent",
    endpoint: "https://api.dune.com/v1/query/1234/results",
    method: "GET",
    idempotencyKey: `run-${crypto.randomUUID()}`,
  }),
});

console.log(result.protocol);           // "x402"
console.log(result.amountPaid.usdc);    // 0.002
console.log(result.txHash);             // "5yJ8kX...mQ9pL"
console.log(result.data);               // the API response body
```

---

## Handle policy rejections

```typescript
try {
  const result = await maxim("/pay", {
    method: "POST",
    body: JSON.stringify({
      agentId: "my-agent",
      endpoint: "https://api.expensive-service.com/query",
      method: "POST",
      body: { query: "..." },
    }),
  });
} catch (err: any) {
  if (err.body?.error === "policy_violation") {
    console.warn("Payment blocked:", err.body.reason, err.body.detail);
    // reason: "per_call_limit_exceeded"
  } else if (err.body?.error === "insufficient_funds") {
    console.error("Wallet needs funding:", err.body.detail);
  } else {
    throw err;
  }
}
```

---

## Check wallet balance

```typescript
const wallet = await maxim("/wallets/my-agent");
console.log(`Balance: ${wallet.balance.usdc} USDC`);
```

---

## List recent transactions

```typescript
const { transactions } = await maxim(
  "/transactions?agentId=my-agent&limit=20"
);

for (const tx of transactions) {
  console.log(`${tx.timestamp}  ${tx.protocol}  ${tx.amountPaid.usdc} USDC  ${tx.endpoint}`);
}
```

---

## Multi-agent: invoke a sub-agent with a budget cap

```typescript
const invocation = await maxim("/agents/invoke", {
  method: "POST",
  body: JSON.stringify({
    agentEndpoint: "https://research-agent.yourplatform.com/run",
    task: {
      query: "Summarize Solana TVL trends for Q1 2026",
      outputFormat: "markdown",
    },
    budget: { usdc: 2.00 },
  }),
  headers: { "X-Agent-Id": "orchestrator-agent" },
});

console.log(invocation.output);
console.log(`Total cost: ${invocation.totalCost.usdc} USDC`);
console.log("Payment chain:", invocation.paymentChain);
```

---

## Fetch spend metrics

```typescript
const metrics = await maxim(
  "/agents/my-agent/metrics?metrics=spend,tx_count,protocol_split&window=7d"
);

console.log(`7-day spend: ${metrics.metrics.spend.total} USDC`);
console.log(`Transactions: ${metrics.metrics.tx_count}`);
console.log(`x402 share: ${(metrics.metrics.protocol_split.x402 * 100).toFixed(1)}%`);
```

---

## Verify a webhook signature

```typescript
import { createHmac, timingSafeEqual } from "crypto";

function verifyWebhook(rawBody: string, signatureHeader: string, secret: string): boolean {
  const expected = "sha256=" + createHmac("sha256", secret).update(rawBody).digest("hex");
  try {
    return timingSafeEqual(Buffer.from(signatureHeader), Buffer.from(expected));
  } catch {
    return false;
  }
}

// In your webhook handler (Express example):
app.post("/webhooks/maxim", express.raw({ type: "application/json" }), (req, res) => {
  const valid = verifyWebhook(
    req.body.toString(),
    req.headers["x-maximprotocol-signature"] as string,
    process.env.WEBHOOK_SECRET!
  );

  if (!valid) return res.status(401).end();

  const event = JSON.parse(req.body.toString());

  if (event.event === "policy.violation") {
    console.warn(`Policy violation for ${event.agentId}: ${event.reason}`);
  }

  res.status(200).end();
});
```

---

## Further reading

- [Quickstart](../getting-started/quickstart.md)
- [Payment Router](../features/payment-router.md)
- [Error Reference](../platform/errors.md)
- [Webhooks](../platform/webhooks.md)
