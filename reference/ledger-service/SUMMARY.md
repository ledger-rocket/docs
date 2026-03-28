# Ledger Service v3.73.6 (Python)

Reference data authority for sites, entities, ledgers, accounts, and account codes.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Health check |
| GET | `/livez` | Liveness probe |
| GET | `/readyz` | Readiness probe |
| GET | `/api/v1/countries` | List countries |
| POST | `/api/v1/countries` | Create countries |
| GET | `/api/v1/countries/{country_code}` | Get country by code |
| GET | `/api/v1/currencies` | List currencies |
| POST | `/api/v1/currencies` | Create currencies |
| GET | `/api/v1/currencies/{currency_code}` | Get currency by code |
| GET | `/api/v1/metadata/categories` | Get product categories |
| GET | `/api/v1/metadata/event-types` | Get event types |
| GET | `/api/v1/metadata/products` | Get products |
| GET | `/api/v1/sites` | List sites |
| POST | `/api/v1/sites` | Create site |
| GET | `/api/v1/sites/{site_id}` | Get site |
| PATCH | `/api/v1/sites/{site_id}` | Update site |
| DELETE | `/api/v1/sites/{site_id}` | Delete site |
| GET | `/api/v1/sites/{site_id}/account-codes` | List account codes |
| POST | `/api/v1/sites/{site_id}/account-codes` | Create account code |
| GET | `/api/v1/sites/{site_id}/account-codes/{code}` | Get account code |
| GET | `/api/v1/sites/{site_id}/accounts` | List accounts |
| POST | `/api/v1/sites/{site_id}/accounts` | Create account |
| GET | `/api/v1/sites/{site_id}/accounts/custodians` | Distinct custodians |
| POST | `/api/v1/sites/{site_id}/accounts/filter` | Filter accounts by external IDs |
| GET | `/api/v1/sites/{site_id}/accounts/{account_id}` | Get account |
| GET | `/api/v1/sites/{site_id}/enriched-entities` | List enriched entities |
| GET | `/api/v1/sites/{site_id}/enriched-entities/{entity_id}` | Get enriched entity |
| GET | `/api/v1/sites/{site_id}/entities` | List entities |
| POST | `/api/v1/sites/{site_id}/entities` | Create entity |
| POST | `/api/v1/sites/{site_id}/entities/filter` | Filter entities |
| GET | `/api/v1/sites/{site_id}/entities/{entity_id}` | Get entity |
| GET | `/api/v1/sites/{site_id}/entities/{entity_id}/accounts` | Entity's accounts |
| GET | `/api/v1/sites/{site_id}/ledgers` | List ledgers |
| POST | `/api/v1/sites/{site_id}/ledgers` | Create ledger |
| GET | `/api/v1/sites/{site_id}/ledgers/owners` | Distinct ledger owners |
| GET | `/api/v1/sites/{site_id}/ledgers/{ledger_id}` | Get ledger |
| GET | `/api/v1/sites/{site_id}/ledgers/{ledger_id}/accounts` | Ledger accounts |
| GET | `/api/v1/sites/{site_id}/ledgers/{ledger_id}/entities` | Ledger entities |
| GET | `/api/v1/sites/{site_id}/stats` | Site DB stats |
| GET | `/api/v1/stats/sites/{site_id}` | Database stats |

41 endpoints total. All list endpoints support advanced filtering via query parameters.

## Key Data Models

### Site

Top-level organizational unit. All entities, ledgers, and accounts are scoped to a site.

### Entity

Represents a beneficial owner or custodian. Entities own ledgers and are associated with accounts.

### Ledger

Scoped to an entity and currency pair. Contains accounts and tracks balances within that scope.

### Account

Dimensional model with the following classification axes:

- **EntityScope** -- ownership scope of the account
- **CounterpartyType** -- relationship to counterparty
- **FunctionalDomain** -- business function
- **ValuationRole** -- valuation purpose

Accounts support linked relationships (nostro/vostro).

### AccountCode

Categorizes accounts into standard types: `ASSET`, `LIABILITY`, `EQUITY`, `REVENUE`, `EXPENSE`, `SUPPLEMENTAL`.

### Currency

ISO currency code with scale (decimal precision).

### Country

ISO country code with metadata.
