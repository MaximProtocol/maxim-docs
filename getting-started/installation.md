# Installation

Maxim Protocol is accessed entirely through a REST HTTP API. No package installation required.

---

## Base URL

```
https://api.maximprotocol.com/v1
```

All endpoints require HTTPS. HTTP requests are rejected.

---

## Authentication

Every request must include your API key in the `Authorization` header:

```bash
Authorization: Bearer rp_live_...
```

Obtain your API key from the [Maxim Protocol dashboard](https://app.maximprotocol.com) under **Settings → API Keys**.

---

## Quick check

Verify your key works:

```bash
curl https://api.maximprotocol.com/v1/wallets \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```

---

## CI/CD environments

Set the API key as an environment variable and pass it with each request:

```bash
export MAXIM_API_KEY=rp_live_...
```

---

## Per-agent keys

For production deployments, use per-agent keys (`rp_agent_...`) scoped to a single agent. A compromised per-agent key cannot access other agents or modify account configuration.

Create a per-agent key:

```bash
curl -X POST https://api.maximprotocol.com/v1/keys \
  -H "Authorization: Bearer $MAXIM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "agentId": "my-agent", "name": "production" }'
```

Rotate without downtime by creating the new key first, updating your application, then revoking the old one:

```bash
curl -X DELETE https://api.maximprotocol.com/v1/keys/rp_agent_01j9... \
  -H "Authorization: Bearer $MAXIM_API_KEY"
```
