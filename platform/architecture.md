# Architecture

This page describes how Maxim Protocol is built and how a payment flows through the platform from your agent to the target service and back.

---

## Overview

Maxim Protocol is built in three layers: a **gateway** that handles protocol detection and policy enforcement, a **protocol layer** that implements x402 and MPP payment handshakes, and a **Solana settlement layer** that records every transaction on-chain.

---

## Technology stack

| Layer | Technology | Rationale |
|---|---|---|
| Gateway API | Go | Low latency, high concurrency, minimal memory overhead |
| x402 handler | Go + Rust (signing) | Ed25519 signing via Solana web3 primitives |
| MPP handler | Go | Session management and multi-rail support |
| Solana program | Rust + Anchor framework | On-chain wallet registry and transaction ledger |
| Settlement token | USDC (SPL Token) | Most widely adopted stablecoin on Solana |
| Session store | Redis | Sub-millisecond MPP session cache |
| State store | PostgreSQL | Agent registry, policy store, account state |
| Dashboard | Next.js + TypeScript | |

---

## Payment flow

Every `POST /v1/pay` request follows this path:

```
Agent sends POST /v1/pay
        │
        ▼
Maxim Protocol Gateway
  - API key authentication
  - TLS termination
  - Rate limiting (per account)
        │
        ▼
Policy Engine
  - Checks domain allowlist / blocklist
  - Checks per-call spend limit
  - Checks daily budget remaining
  - Checks rate limit window
  - Rejects here if any check fails (no funds spent)
        │
        ▼
Protocol Detector
  - Sends original request to target endpoint
  - Receives 402 response
  - Reads response to determine x402 or MPP
        │
        ├── x402 ───────────────────────────────────────────────────────┐
        │                                                               │
        │   x402 Handler                                                │
        │   - Parses PaymentRequired object                             │
        │   - Constructs PaymentPayload                                 │
        │   - Signs with agent keypair (Ed25519)                        │
        │   - Retransmits original request with X-PAYMENT header        │
        │   - Receives 200 response from resource server                │
        │                                                               │
        └── mpp ──────────────────────────────────────────────────────┐ │
                                                                      │ │
            MPP Handler                                               │ │
            - Checks session cache for existing session               │ │
            - If no session: negotiates new session via Tempo rail    │ │
            - Authorizes payment with session credential              │ │
            - Receives 200 response from resource server              │ │
                                                                      │ │
        ◄─────────────────────────────────────────────────────────────┘ │
        ◄───────────────────────────────────────────────────────────────┘
        │
        ▼
Solana Settlement Bridge
  - Constructs SPL Token transfer instruction
  - Signs with agent keypair
  - Submits to Solana mainnet
  - Waits for confirmation (under 1000ms)
  - Writes on-chain ledger record to maximprotocol.sol program
        │
        ▼
Response returned to agent
  - PaymentResult with status, data, txHash, amountPaid, protocol
```

---

## Solana program: maximprotocol.sol

The Maxim Protocol on-chain program handles:

**Wallet registry**
Each agent wallet is a program-derived address (PDA) derived from `[agent_id, program_id]`. The PDA owns a USDC associated token account (ATA) via the SPL Token program.

**Transaction ledger**
Every payment writes a ledger account containing:
- Agent wallet address
- Payee address
- USDC amount (u64, 6 decimal places)
- Protocol (x402 or MPP, stored as enum)
- Endpoint hash (SHA256 of the endpoint URL)
- Timestamp (Unix epoch, i64)
- Parent transaction hash (for multi-agent chains, or null)
- Policy check result

**Payment instruction**
The payment instruction:
1. Verifies the signer is the agent's keypair
2. Transfers USDC from the agent's ATA to the payee's ATA via SPL Token CPI
3. Writes the ledger account
4. Emits a program event for the dashboard indexer

---

## Spend policy enforcement

Policies are enforced in two places:

**Gateway layer (primary):** Before any request is sent to the target endpoint. Fast rejection with no network cost and no on-chain fee.

**On-chain layer (high-value transactions):** For transactions above a configurable threshold, the policy is also encoded into the Solana instruction and verified by the on-chain program. This ensures that even if the gateway were bypassed, the on-chain program would reject the transfer.

The default high-value threshold is 10 USDC per transaction. Enterprise accounts can configure this.

---

## MPP session store

MPP sessions are cached in Redis with a TTL matching the session's expiry. The key is `{agentId}:{serviceDomain}`. On cache miss, the MPP handler negotiates a new session with the service, stores it, and proceeds.

Sessions are not stored on-chain. Only the final payment settlement is on-chain.

---

## Dashboard indexer

A lightweight indexer subscribes to Solana program events via `geyser` and writes denormalized records to the dashboard read database. The indexer runs with a one-second lag target. If the indexer falls behind, the dashboard shows a staleness indicator and instructs users to check Solana Explorer directly.

---

## Data residency

By default, Maxim Protocol stores off-chain state (agent registry, policy store, MPP sessions) in the United States. On-chain data (wallet balances, transaction ledger) is global and replicated across all Solana validators.

Enterprise plans can request off-chain data residency in:

- European Union (Frankfurt)
- United Kingdom (London)
- Asia Pacific (Singapore)

Contact support to configure data residency for your account.

---

## Key management

Agent wallet keypairs are stored encrypted in Maxim Protocol's key management service (KMS). The KMS uses envelope encryption: each keypair is encrypted with a per-account key, which is itself encrypted with a root KMS key.

The private key is decrypted in memory only at the point of signing a Solana transaction. It is never written to disk unencrypted and never transmitted outside the signing process.

Enterprise accounts can bring their own KMS key (BYOK). See [Security](security.md) for details.

---

## REST API reference

All endpoints are under `https://api.maximprotocol.com/v1`. Every request requires `Authorization: Bearer <key>`.

### Wallets

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/wallets` | Create an agent wallet |
| `GET` | `/v1/wallets/{agentId}` | Get wallet balance and address |
| `POST` | `/v1/wallets/{agentId}/fund` | Fund a wallet from your account's configured funding source |
| `POST` | `/v1/wallets/{agentId}/withdraw` | Withdraw USDC to an external Solana address |
| `POST` | `/v1/wallets/import` | Import an existing Ed25519 keypair |

### Payments

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/pay` | Make a payment — protocol auto-detected |
| `POST` | `/v1/agents/invoke` | Invoke a sub-agent with an optional budget cap |

### Policies

| Method | Path | Description |
|---|---|---|
| `PUT` | `/v1/policies/{agentId}` | Set or replace spend policies |
| `GET` | `/v1/policies/{agentId}` | Get current spend policies and usage counters |

### Transactions

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/transactions` | List transactions; filter by `agentId`, `protocol`, `domain`, `since`, `status` |
| `GET` | `/v1/transactions/{txHash}` | Get a single transaction |
| `GET` | `/v1/transactions/{txHash}/chain` | List all transactions in a multi-agent chain |
| `GET` | `/v1/transactions/{txHash}/tree` | Get the full payment tree with cost rollup |

### Sessions

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/sessions` | List active MPP sessions for an agent |
| `DELETE` | `/v1/sessions` | Clear sessions; filter by `agentId` and `domain` |

### Metrics

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/agents/{agentId}/metrics` | Aggregated spend, tx count, and protocol split; supports `window` and `resolution` |

### Keys and alerts

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/keys` | Create a per-agent API key |
| `DELETE` | `/v1/keys/{keyId}` | Revoke an API key |
| `POST` | `/v1/alerts` | Create a budget, spend-rate, or policy-violation alert |

---

## Error responses

All API errors return JSON with a machine-readable `error` field:

```json
{
  "error": "policy_violation",
  "reason": "per_call_limit_exceeded",
  "detail": "Requested 2.50 USDC, limit is 1.00 USDC"
}
```

| HTTP status | Meaning |
|---|---|
| `401` | API key missing or invalid |
| `402` | Payment rejected by policy (see `error` body) |
| `403` | Per-agent key used for an account-level operation |
| `429` | Account-level rate limit exceeded |
| `500` | Internal error |
