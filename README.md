# Maxim Protocol Documentation

**The payment rail for autonomous AI agents.**

Maxim Protocol is the unified payment layer for AI agents. It routes payments across the x402 and MPP protocols, settles every transaction on Solana, and gives every agent a non-custodial wallet. One API call handles the rest.

---

## What is Maxim Protocol?

AI agents are spending money. They rent compute, call APIs, browse the web, and pay other agents for services. The infrastructure to support this is fragmented: two major protocols are competing for the standard, settlement happens across multiple chains, and developers are rebuilding payment logic from scratch for every project.

Maxim Protocol solves this at the infrastructure level. It provides:

- **A universal payment router** that detects whether a service requires x402 or MPP and handles the full payment handshake automatically
- **Non-custodial agent wallets** on Solana, provisioned in one line of code
- **Spend policies** enforced at the gateway before any funds leave the wallet
- **On-chain settlement** for every payment, regardless of protocol, with a public Solana transaction hash
- **Multi-agent payment chains** where every sub-agent payment is linked to the parent transaction on-chain

Maxim Protocol does not build agents, choose AI models, or lock you into proprietary formats. It is payment infrastructure. Fast, transparent, and out of the way.

---

## Documentation

[Quickstart](getting-started/quickstart.md)

[Installation](getting-started/installation.md)

[Your First Payment](getting-started/your-first-payment.md)

[x402 Protocol](features/x402.md)

[MPP Protocol](features/mpp.md)

[Payment Router](features/payment-router.md)

[Agent Wallets](features/agent-wallets.md)

[Spend Policies](features/spend-policies.md)

[Multi-Agent Payments](features/multi-agent-payments.md)

[Observability](features/observability.md)

[Architecture](platform/architecture.md)

[Security](platform/security.md)
