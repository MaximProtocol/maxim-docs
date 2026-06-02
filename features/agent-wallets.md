# Agent Wallets

Every agent in Maxim Protocol has a non-custodial Solana wallet. The wallet is the source of funds for all payments the agent makes. Maxim Protocol cannot move funds without a valid signed instruction from the agent's keypair.

---

## How wallets work

Agent wallets are **program-derived addresses (PDAs)** on Solana, derived from the agent's unique ID and the Maxim Protocol on-chain program. The PDA is deterministic — the same agent ID always produces the same wallet address.

USDC is held in a Solana Associated Token Account (ATA) owned by the PDA. When an agent makes a payment, Maxim Protocol constructs a Solana transaction signed by the agent's keypair, transfers USDC to the payee, and records the payment in the on-chain ledger.

---

## Creating a wallet

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

## Funding a wallet

### Via dashboard

Deposit USDC to the wallet address at [app.maximprotocol.com](https://app.maximprotocol.com). The dashboard also supports card top-ups that convert to USDC on-chain automatically.

### Via direct transfer

Send USDC (SPL Token) to the wallet address from any Solana wallet. Funds arrive within one block confirmation (under 1000ms).

### Via API

```bash
curl -X POST https://api.maximprotocol.com/v1/wallets/my-agent/fund \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "usdc": 100 }'
```

This initiates a transfer from the funding source configured in your account settings.

---

## Checking balance

```bash
curl https://api.maximprotocol.com/v1/wallets/my-agent \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

```json
{
  "agentId": "my-agent",
  "address": "7xKXtg2xpAGMq8KLwpSodE9LFKq8Z1EscezSShpump",
  "balance": { "usdc": 42.50 },
  "network": "solana-mainnet"
}
```

---

## Withdrawing funds

Withdraw USDC from an agent wallet to any Solana wallet address:

```bash
curl -X POST https://api.maximprotocol.com/v1/wallets/my-agent/withdraw \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "9zXKtg2xpAGMq8KLwpSodE9LFKq8Z1EscezSShpump",
    "usdc": 20
  }'
```

Withdrawals require authentication and are logged on-chain.

---

## Keypair management

Each agent wallet is controlled by an Ed25519 keypair. Maxim Protocol generates and stores the keypair in encrypted form in your account. The private key never leaves Maxim Protocol's key management service.

For Enterprise accounts with bring-your-own-key (BYOK) requirements:

```bash
curl -X POST https://api.maximprotocol.com/v1/wallets/import \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "my-agent",
    "keypair": "<base64-encoded 64-byte keypair>"
  }'
```

The keypair must be a Solana Ed25519 keypair (64-byte array, base64-encoded). Once imported, Maxim Protocol stores the keypair under your account's KMS encryption.

---

## Multiple wallets

Each agent has exactly one wallet. If you need multiple spending accounts within a single agent (for example, separating payments by task type), create separate agent IDs:

```bash
curl -X POST https://api.maximprotocol.com/v1/wallets \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "agentId": "my-agent-compute" }'

curl -X POST https://api.maximprotocol.com/v1/wallets \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "agentId": "my-agent-data" }'
```

Each agent ID corresponds to a distinct PDA and a distinct USDC balance.

---

## On-chain visibility

Because agent wallets are Solana PDAs, every transaction is publicly visible on Solana Explorer. Anyone with the wallet address can verify the full payment history independently of Maxim Protocol.

```bash
curl https://api.maximprotocol.com/v1/wallets/my-agent \
  -H "Authorization: Bearer $MAXIM_API_KEY"
# Returns address field: 7xKXtg2xpAGMq8KLwpSodE9LFKq8Z1EscezSShpump
# Open: https://explorer.solana.com/address/7xKXtg2xpAGMq8KLwpSodE9LFKq8Z1EscezSShpump
```

---

## Non-custodial guarantee

Maxim Protocol holds the encrypted agent keypair, but cannot unilaterally move funds. Every payment transaction requires:

1. A valid signed instruction from the agent's keypair
2. A spend policy check that passes
3. A valid x402 or MPP payment request from an authorized domain

There is no administrative override. Maxim Protocol support cannot transfer funds from an agent wallet without the keypair signature.

---

## Further reading

- [Spend policies](spend-policies.md)
- [Security and key management](../platform/security.md)
- [REST API reference](../platform/architecture.md)
