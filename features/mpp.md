# MPP

MPP (Machine Payments Protocol) is an open payment standard co-authored by Stripe and Tempo, submitted to the IETF as a Standards Track Internet-Draft. It is purpose-built for high-frequency agent payments and supports multiple payment rails including stablecoins, cards, and Lightning.

---

## Background

MPP was launched in March 2026 with support from over 50 services at launch, including OpenAI, Anthropic, Google Gemini, Dune, Browserbase, Modal, fal.ai, and AgentMail. Enterprise partnerships include Mastercard, Visa, Shopify, DoorDash, and Revolut.

Unlike x402, which wraps existing blockchain infrastructure, MPP was designed from scratch for agent payment semantics: native sessions, pluggable payment rails, and direct IETF standardization.

---

## How MPP works

MPP payments flow over standard HTTP. The protocol negotiates a payment method, establishes a session, and settles in the background while the response is returned to the agent.

```
1. Agent sends request
        POST https://api.service.com/inference

2. Server responds with payment request
        HTTP/1.1 402 Payment Required
        Content-Type: application/json

        {
          "protocol": "mpp/1.0",
          "amount": { "value": "0.012", "currency": "USD" },
          "paymentMethods": ["tempo-stablecoin", "stripe-card"],
          "sessionSupported": true,
          "paymentEndpoint": "https://pay.service.com/mpp"
        }

3. Maxim Protocol selects a payment method and authorizes payment
        POST https://pay.service.com/mpp
        Content-Type: application/json

        {
          "method": "tempo-stablecoin",
          "amount": { "value": "0.012", "currency": "USD" },
          "authorization": { ... }
        }

4. Service confirms and returns response + session token
        HTTP/1.1 200 OK
        X-MPP-Session: sess_01j9x4k2m3n5...

        { ...response body... }

5. Subsequent calls reuse the session (no renegotiation)
        POST https://api.service.com/inference
        X-MPP-Session: sess_01j9x4k2m3n5...
```

---

## Key properties

| Property | Value |
|---|---|
| Co-authors | Stripe, Tempo |
| Standardization | IETF Standards Track Internet-Draft |
| Payment rails | Tempo stablecoins, Stripe cards, Lightning, custom |
| Native session support | Yes |
| Facilitator required | No |
| Settlement latency | Under 100ms (via escrow + EIP-712 vouchers) |
| Adopted by | OpenAI, Anthropic, Google Gemini, 50+ services |

---

## Payment rails

MPP separates payment negotiation from settlement. The protocol defines how agents and services agree on payment; the rail defines how money moves.

| Rail | Settlement | Use case |
|---|---|---|
| Tempo stablecoin | Blockchain (stablecoin) | Crypto-native services, high frequency |
| Stripe card | Stripe payment network | Services already on Stripe; existing card infrastructure |
| Lightning | Bitcoin Lightning Network | Bitcoin-denominated payments |
| Custom | Provider-defined | Enterprise or proprietary rails |

Maxim Protocol uses Tempo stablecoin as the default MPP rail, with on-chain recording to Solana for every transaction. Stripe card payments are supported as a fallback when a service does not offer a stablecoin rail.

---

## Sessions

MPP has native session support from v1. Sessions allow a payment authorization to cover multiple requests without renegotiating the full payment handshake each time.

Maxim Protocol manages MPP sessions automatically:
- Sessions are established on the first request to each service
- Session tokens are stored per agent, per service
- Subsequent calls to the same service reuse the session token
- Sessions are refreshed automatically when they expire

Session reuse significantly reduces latency for agents making repeated calls to the same service.

---

## x402 vs. MPP

Both protocols serve the same purpose: enabling agents to pay for services over HTTP. The differences are architectural:

| | x402 | MPP |
|---|---|---|
| Designed by | Coinbase | Stripe + Tempo |
| Architecture | Lightweight wrapper on blockchain | Purpose-built protocol |
| Settlement | On-chain stablecoins | Multi-rail (stablecoins, cards, Lightning) |
| Facilitator | Required | Not required |
| Standardization | Open specification | IETF Standards Track |
| Session support | v2 (added later) | v1 (native) |
| Blockchain dependency | Required | Optional |

Neither protocol is clearly dominant. Services are adopting one or both. Maxim Protocol supports both, so your agent can pay for any service regardless of which protocol it uses.

---

## How Maxim Protocol handles MPP

When `POST /v1/pay` is called against an MPP-gated service:

1. Maxim Protocol sends the request and receives the 402 with MPP requirements
2. The spend policy is checked against the required amount and domain
3. If a valid session exists for this service, Maxim Protocol reuses it
4. Otherwise, Maxim Protocol negotiates a new session using the Tempo stablecoin rail
5. Payment is authorized and the original request is fulfilled
6. Maxim Protocol records the transaction to the Solana on-chain ledger regardless of which rail settled it

The caller receives the same response format regardless of protocol.

---

## Further reading

- [MPP specification](https://mpp.dev)
- [x402 protocol](x402.md)
- [Payment Router](payment-router.md) — how Maxim Protocol chooses between x402 and MPP
