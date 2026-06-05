# Security

Security is built into Maxim Protocol's architecture. This page describes how we protect agent wallets, keypairs, API credentials, and on-chain data.

---

## Non-custodial wallet model

Agent wallets are non-custodial program-derived addresses (PDAs) on Solana. The key properties:

- Every payment requires a valid Ed25519 signature from the agent's keypair
- Maxim Protocol cannot construct a valid signature without the keypair
- There is no administrative override that allows Maxim Protocol to move funds unilaterally
- Wallet balances are visible on-chain at all times and are independent of Maxim Protocol's systems

If Maxim Protocol's infrastructure were unavailable, the USDC in agent wallets would remain intact and accessible via any Solana wallet that holds the agent keypair.

---

## Keypair storage

Agent keypairs (Ed25519) are stored using envelope encryption:

1. The keypair is encrypted with a per-agent data encryption key (DEK)
2. The DEK is encrypted with a per-account key encryption key (KEK)
3. The KEK is stored in an external key management service (KMS) and never written to application storage

The private key material is decrypted in memory only at the moment of signing a Solana transaction. It is never written to disk unencrypted, never logged, and never transmitted outside the signing process.

**Bring your own key (BYOK):** Enterprise accounts can supply their own KMS key. Maxim Protocol's KMS integration supports AWS KMS, Google Cloud KMS, and HashiCorp Vault. Contact support to enable BYOK.

---

## API key model

**Account keys** (`rp_live_...`) authenticate all API operations and have full access to your account. Store them in a secrets manager. Do not commit them to code or include them in client-side applications.

```bash
export MAXIM_API_KEY=rp_live_...
```

**Per-agent keys** (`rp_agent_...`) are scoped to a single agent. A compromised per-agent key cannot access other agents, read account configuration, or modify policies.

Create a per-agent key:

```bash
curl -X POST https://api.maximprotocol.com/v1/keys \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "agentId": "my-agent", "name": "production" }'
```

Rotate without downtime:

```bash
# Create the new key
curl -X POST https://api.maximprotocol.com/v1/keys \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "agentId": "my-agent", "name": "production-v2" }'

# Update your application to use the new key, then revoke the old one
curl -X DELETE https://api.maximprotocol.com/v1/keys/rp_agent_01j9... \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

---

## Network security

- All traffic between clients and Maxim Protocol is encrypted with TLS 1.3
- HTTPS is enforced on all API endpoints. HTTP requests are rejected
- The Solana RPC connection uses authenticated endpoints with per-account rate limits
- Webhook deliveries use HMAC-SHA256 request signing

**Verifying webhook signatures:**

Webhook deliveries include an `X-MaximProtocol-Signature` header. Verify it by computing `sha256=HMAC-SHA256(rawBody, webhookSecret)` using your webhook secret and comparing it to the header value. Any HMAC-SHA256 implementation works.

```bash
# Bash example using openssl
EXPECTED="sha256=$(echo -n "$RAW_BODY" | openssl dgst -sha256 -hmac "$RAIL_WEBHOOK_SECRET" | awk '{print $2}')"
[ "$SIGNATURE_HEADER" = "$EXPECTED" ] && echo "Verified" || echo "Invalid"
```

---

## Data at rest

| Data | Storage | Encryption |
|---|---|---|
| Agent keypairs | KMS (envelope encryption) | AES-256 DEK + KMS-managed KEK |
| API keys | Hashed (bcrypt) | Not stored in plaintext |
| Spend policies | PostgreSQL | AES-256 transparent data encryption |
| MPP session tokens | Redis | AES-256, in-memory only |
| Audit logs | PostgreSQL | AES-256 transparent data encryption |
| On-chain ledger | Solana | Public (by design) |

On-chain transaction records are public. Endpoint URLs are stored as SHA256 hashes on-chain. The full URL is stored in Maxim Protocol's off-chain database (encrypted) and shown in the dashboard, but is not visible on Solana Explorer.

---

## Spend policy as a security layer

Spend policies are not just budget controls. They are a security boundary. A compromised agent can only spend up to its per-call limit, only on allowlisted domains, and only at its rate limit. A stolen API key that gains access to an agent cannot drain the wallet arbitrarily.

For high-security deployments, use narrow allowlists and low per-call limits. Policy violations are logged and can trigger alerts.

---

## Compliance

| Framework | Status |
|---|---|
| SOC 2 Type II | Available on Enterprise plan |
| GDPR | Data processing agreement available |
| ISO 27001 | In progress |

Request compliance documentation: compliance@maximprotocol.com

---

## Responsible disclosure

If you discover a security vulnerability in Maxim Protocol, report it to security@maximprotocol.com. We respond to all reports within 24 hours and aim to patch critical vulnerabilities within 72 hours.

Do not disclose vulnerabilities publicly until we have had the opportunity to address them.
