# Authentication

## API Key Authentication

All LedgerRocket API requests are authenticated using an API key passed in the `x-api-key` HTTP header. Every request to every service (Ledger Service, Event Service, Balance Service) must include this header.

```bash
curl -s "https://ledger.dev.ledgerrocket.com/v2/ledger/api/v1/sites" \
  -H "x-api-key: your-api-key-here"
```

### Header Format

| Header | Value | Required |
|--------|-------|----------|
| `x-api-key` | Your API key string | Yes, on all authenticated endpoints |

### Public Endpoints (No Auth Required)

The following endpoints are accessible without an API key:

- `/livez` -- liveness probe
- `/readyz` -- readiness probe
- `/docs` -- interactive API documentation (Scalar)
- `/openapi.json` -- OpenAPI specification

## Obtaining an API Key

API keys are provisioned by the LedgerRocket team during onboarding. Contact your account representative or reach out to the platform team to request credentials.

Each key is scoped to an environment and may carry specific permissions. You will receive separate keys for each environment you need access to.

## Environment-Specific Keys

LedgerRocket provides separate API keys per environment. Never use a production key in development, and never use a development key in production.

| Environment | Base URL | Key Prefix |
|-------------|----------|------------|
| Development | `https://ledger.dev.ledgerrocket.com` | `lr-test-*` |
| Staging | `https://ledger.staging.ledgerrocket.com` | `lr-staging-*` |
| Production | `https://ledger.ledgerrocket.com` | `lr-prod-*` |

### Setting Up Shell Variables

Store your key in an environment variable to avoid embedding it in commands:

```bash
export LR_API_KEY="lr-test-20250701"

# Then use it in requests
curl -s "https://ledger.dev.ledgerrocket.com/v2/ledger/api/v1/sites" \
  -H "x-api-key: $LR_API_KEY"
```

## OAuth / JWT Authentication

For production deployments, LedgerRocket supports OAuth 2.0 with JWT tokens validated against an AWS Cognito user pool. When OAuth is enabled, the API key header is replaced by a standard `Authorization: Bearer <token>` header.

### OAuth Scopes

The Event Service defines three scope levels:

| Scope | Access |
|-------|--------|
| `events.read` | Read access to events, transfers, templates, and balances |
| `events.write` | Write access -- submit events, validate events |
| `events.admin` | Admin access -- template lifecycle management, debug endpoints |

Scopes are hierarchical: `events.admin` includes `events.write`, which includes `events.read`.

### Token Format

When using OAuth, include the JWT in the `Authorization` header:

```bash
curl -s "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/events" \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIs..."
```

Contact the platform team for OAuth configuration details specific to your deployment.

## Example: Full Request with Authentication

A complete authenticated request to submit an event:

```bash
curl -s -X POST "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/events" \
  -H "Content-Type: application/json" \
  -H "x-api-key: $LR_API_KEY" \
  -d '{
    "event_id": "01956a3b-c4d5-7e6f-8901-234567890abc",
    "site_id": 1,
    "occurred_at": 1711612800000,
    "template_ids": [532],
    "amount": 150000,
    "event_source": "quickstart",
    "accounts": [
      {"account_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "purpose": "sender"},
      {"account_id": "2c1a8f2e-3b47-4d28-9f4d-8b6f2d9a0c11", "purpose": "receiver"}
    ],
    "extra_metadata": {
      "reference": "INV-2026-001"
    }
  }'
```

### Error Responses

If authentication fails, the API returns an RFC 9457 Problem Details response:

```json
{
  "type": "https://ledger-rocket.dev/event-service/problems/auth/unauthorized",
  "title": "Unauthorized",
  "status": 401,
  "detail": "Missing or invalid API key",
  "instance": "/api/v1/events"
}
```

| Status | Meaning |
|--------|---------|
| `401 Unauthorized` | Missing API key, invalid key, or expired JWT |
| `403 Forbidden` | Valid credentials but insufficient scope for the requested operation |

## Security Best Practices

1. **Never commit API keys to source control.** Use environment variables, secrets managers (AWS Secrets Manager, HashiCorp Vault), or CI/CD secret injection.

2. **Rotate keys regularly.** Request new keys from the platform team on a schedule that fits your security policy.

3. **Use the narrowest scope possible.** If a service only reads balances, it should not hold a key with write access.

4. **Use separate keys per environment.** Development, staging, and production should each have their own credentials. Never reuse keys across environments.

5. **Audit key usage.** Monitor API access logs for unexpected usage patterns -- unusual request volumes, requests from unknown IPs, or access to endpoints outside normal workflows.

6. **Prefer OAuth for production.** API keys are convenient for development and testing. For production workloads, OAuth with short-lived JWTs provides stronger security guarantees including automatic token expiry and scope-based access control.

7. **Transmit keys only over HTTPS.** All LedgerRocket endpoints use TLS. Never send API keys over unencrypted connections.

## Next Steps

- [Architecture](04-architecture.md) -- how the services fit together
- [Quickstart](02-quickstart.md) -- end-to-end tutorial with curl examples
