# Functional Specification: Go Event Processing Service

Version: 1.3.1

---

## 1. Executive Summary _(informative)_

A stateless, high‑performance service that transforms financial events into double‑entry transfers using template‑driven business logic. The core is pure/deterministic: given an event and the active templates/rules, it returns transfers or a validation error. Side effects (TigerBeetle commit, outbox durability, and Event Store publish) are thin adapters around the core.

---

## 2. Conventions _(normative)_

### 2.1 Normative Language (BCP 14)

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, NOT RECOMMENDED, MAY, and OPTIONAL are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

### 2.2 Requirements Syntax (EARS)

Ubiquitous, Event‑driven, State‑driven, Optional, and Complex patterns are used.
_Example_: “WHEN an event is received, the system SHALL validate it against the JSON schema.”

---

## 3. Design Principles _(normative)_

- **Stateless core**: The system SHALL maintain pure calculation from inputs to transfers
- **Fail Fast**: The system SHALL provide clear, early errors
- **No Defaults**: The system SHALL NOT apply implicit financial fallbacks
- **Immutability**: Processed events and transfers SHALL be immutable
- **Atomicity**: All transfers for an event SHALL succeed or none SHALL succeed
- **Auditability**: The system SHALL maintain a complete, immutable event record after commit
- **Timestamps**: All timestamps are Unix time in milliseconds since the Unix epoch

—

## 3.1 Event Directives _(normative)_

- The Event supports first‑class `directives` (array of enum tags) that control processing:
  - `manual_rollup_override` — allow direct posting to rollup accounts.
  - `skip_rollup_processing` — suppress rollup generation.
  - `seed_mode` — indicates backfill/seed operations and REQUIRES `ledger_timestamp`.
- `ledger_timestamp` (ms) is distinct from `occurred_at` and is REQUIRED when `seed_mode` is present.
- Reserved `__*` keys in `extra_metadata` are FORBIDDEN and SHALL be rejected. Clients MUST use `directives`.

---

## 4. Service Overview _(informative)_

### 4.1 Core Purpose

- Transform financial events into double‑entry transfers
- Validate events against business rules and template requirements
- Generate rollup transfers for aggregated accounting
- Maintain an immutable audit trail
- Submit transfers to the Ledger System (TigerBeetle)

### 4.2 Architecture Overview

```text
                    Ledger Service
                 (account + ref data)
                           ↓
Client Applications → Event Service → Ledger System (TigerBeetle transfers)
                                    ↘ Event Store
```

#### Key points

- Core logic is stateless and deterministic.
- Transfers are submitted atomically to TigerBeetle (TB).
- Account/reference data are fetched from Ledger Service.
- Successfully processed events are published to Event Store asynchronously from durable COMMITTED outbox records.
- Clear separation of business logic from persistence and integration concerns.

### 4.3 Scope

#### In scope

- Template‑driven processing of events into transfers
- Event deduplication via dummy transfer
- Expression evaluation (CEL) with integer arithmetic only
- Integration with Ledger Service, TigerBeetle, and Event Store
- Caching strategy; JSON schema and business‑rule validation
- Rollup transfer generation
- Immutable event and audit storage (Event Store + object storage for audit)
- HTTP APIs for processing and administration
- Health checks and metrics

#### Out of scope

- Specific dev tooling/CI/CD; code/algorithms internal to TB
- Infra/network topology; authn/authz beyond what’s stated
- Implementation details for caching/connection pools
- Audit slug lifecycle management (create, submit, approve, reject) and audit store writes are handled by a separate service. This service only reads slugs/templates and decides whether to use them.

### 4.4 Core Capabilities

#### Business Capabilities

- Template-driven event processing - Transform financial events into double-entry transfers using configurable templates
- Multi-template composition - Process up to 256 templates atomically per event for complex financial operations
- Automatic rollup generation - Create aggregated accounting entries for reporting
- Business rule enforcement - Apply site-level and template-level validation rules
- Maker-checker controls - Template lifecycle management with approval workflows
- Deterministic validation trace - Produce explain-style traces of variable evaluations, validation outcomes, and leg resolution for troubleshooting

#### Technical Capabilities

- Exactly-once processing - Guaranteed deduplication via dummy transfer mechanism
- Atomic batch operations - All-or-nothing submission of all transfers for an event
- Integer-only arithmetic - Precise financial calculations without floating-point errors
- CEL expression evaluation - Safe, sandboxed business logic execution
- Immutable audit trail - Complete history of events and transfers
- Transfer ID availability - TigerBeetle remains the sole source of truth for transfer identifiers; the service returns IDs from TB and exposes lookup APIs that read them back from TB on demand.

#### Integration Capabilities

- Account enrichment - Fetch and cache account, ledger, entity, account code, currency, and country data from Ledger Service
- TigerBeetle integration - Submit transfers to the double-entry ledger system
- Resilient event storage - Durable WAL outbox plus asynchronous Event Store publication
- Template and rule management - Load and cache templates/rules from audit-controlled storage
- Real-time cache refresh - Hot-swap reference data and templates without downtime
- Transfer lookup API - Expose committed transfer identifiers via HTTP for downstream reconciliation services

---

## 5. Data Models _(normative)_

### 5.1 Event Request (Input)

Formal schema: `../schemas/api/events-post-request.schema.json`

#### Identity and timestamps

- `event_id` MUST be **UUIDv7**. Non‑UUIDv7 SHALL be rejected with HTTP 400 (RFC 9457 Problem Details).
- `occurred_at` is REQUIRED. It is a Unix timestamp in milliseconds since the Unix epoch.
- `ledger_timestamp` (Unix milliseconds) is OPTIONAL and only valid when accompanied by the `seed_mode` directive; when present it overrides `occurred_at` for transfer timestamps (§6.1.9).

#### Site

- `site_id` (int, required). Identifies the installation/environment; templates and ledgers are scoped to this value. Included in CEL context as `event.site_id`.

#### Accounts

- `accounts` is a map of **purpose → internal account_id**.
- Clients MUST send **internal account IDs** (no external IDs).
- **Account purpose uniqueness**: no two accounts may share the same purpose. Error: `"Duplicate account purpose '{purpose}' found"`.

#### Templates

- `template_ids`: array of template IDs. Range: **1..65535** (fits TB `code` u16). Max per event: **256**.

#### `$schema` (canonical schema URL)

- `$schema` (string, required) MUST point at the hosted JSON Schema for the payload, e.g. `https://ledger-rocket.github.io/schemas/domain/v1.0.0/event.schema.json`.
- **Purpose**: analytics & provenance. Tooling and documentation dereference this URL; the Event Service still relies on the strict loader + domain validation instead of executing JSON Schema on the hot path.

#### Metadata

- `extra_metadata`: JSON object; size ≤ **16 KB**; depth ≤ **6**. User-defined keys MUST NOT start with `__`; reserved prefixes are rejected. All first-class processing switches are expressed via the `directives[]` array instead of metadata.

##### Integer‑Only Numeric Policy (Event)

- `amount` MUST be an integer (minor units).
- Numeric values inside `extra_metadata` MAY be floats if they are not referenced by any active template.
- If a template expression references `event.extra_metadata.<path>`, the value at `<path>` MUST be an integer at runtime; otherwise the server SHALL reject the request and identify the offending path.

#### Event linkage (optional)

- `original_event_id`: UUIDv7 of the first event in a logical chain (e.g., loan origination). When provided it MUST be UUIDv7 and MUST differ from `event_id`.
- `previous_event_id`: UUIDv7 of the immediate predecessor event (e.g., disbursal that triggered the current repayment). When provided it MUST be UUIDv7 and MUST differ from `event_id`.
- These fields do not affect processing behaviour; they allow downstream analytics to reconstruct event sequences without querying additional systems.

#### Runtime validations (beyond JSON Schema)

1. Account purpose uniqueness
2. Template metadata validation: `extra_metadata` MUST satisfy the `metadata_schema` of **all** specified templates
3. Reserved metadata fields: `__*` prefixes are rejected

### 5.2 Template

Formal schema: `../schemas/domain/template.schema.json`

#### Business logic components

1. **Variables**: CEL expressions evaluated sequentially
2. **Validations**: CEL predicates that MUST pass
3. **Legs**: transfer definitions referencing variables/selectors (no expressions)
4. **Metadata Schema**: JSON Schema for `extra_metadata`

#### `$schema` (canonical schema URL)

- `$schema` (string, required) MUST point at the hosted template schema, e.g. `https://ledger-rocket.github.io/schemas/domain/v1.1.0/template.schema.json`, ensuring IDEs and tooling can dereference the precise schema variant used to author the payload.

#### Capabilities _(template-level toggles)_

- `allows_fx_transfers`: Template may generate transfers where the debit and credit ledgers have different currencies (same entity). Each individual transfer still posts within a single ledger.
- `allows_intercompany_transfers`: Template may generate transfers where the debit and credit ledgers belong to different entities (same currency). Each leg remains same-ledger.
- `skip_account_type_validation`: Engine skips runtime validation that enriched account types match the leg declarations. Use sparingly for manual adjustment templates.
- `skip_financial_statements_impact_validation`: Engine skips reconciling generated transfers against `financial_statements_impact`. Useful when adjustments intentionally leave the impact list empty.
- Capabilities are declared as an array; omit the capability when the default behaviour should apply.

#### Variable‑only legs with whitelisted selectors

- **No CEL expressions in legs.** Each leg field (`debit_account`, `credit_account`, `amount`, `condition`) MUST be a **symbol** resolving to:
  1. a previously defined **variable**, or
  2. a **whitelisted dotted selector** into the event/enriched context.

#### Allowed selectors

- `event.*` (e.g., `event.amount`, `event.occurred_at`, `event.event_id`)
- `accounts.<purpose>.*` over **enriched** accounts (read‑only), including:
  - `accounts.<purpose>.account_id`
  - `accounts.<purpose>.ledger.ledger_id`
  - `accounts.<purpose>.ledger.currency.currency_code`
  - `accounts.<purpose>.ledger.currency.scale`
  - `accounts.<purpose>.account_code.account_type`
- Templates MAY alias account IDs into variables (e.g., `cash_account_id = accounts.cash.account_id`) and reference those variables in `debit_account` / `credit_account` fields.

#### Not allowed in legs

- Operators, functions, conditionals

##### Integer‑Only Numeric Policy (Template)

- Variables, validations, and conditions MUST use integer arithmetic only.
- Decimal literals (e.g., `0.01`, `1.5`) and floating‑point functions are **forbidden**. Template compilation SHALL fail on detection.
- Templates MAY declare `number` types in `metadata_schema`, but any metadata field actually referenced by expressions SHOULD be declared as `integer` (RECOMMENDED). At runtime, referenced metadata values MUST be integers.

#### Condition variable

- `condition` MUST reference a boolean **variable** computed in CALCULATE. If unconditional, reference `always_true`.
- The engine provides a built‑in, read‑only boolean variable `always_true = true` that MAY be referenced by `condition` for unconditional legs. This name is reserved and cannot be overridden by templates.

#### Template invariants (engine‑enforced)

The following rules are enforced by the engine for every event and MUST NOT be restated as template validations:

- **Amounts**: integer minor units and > 0; any generated leg with amount < 1 is an error (§6.1.6, §6.1.4).
- **Integer‑only arithmetic**: variables/validations/conditions MUST use integer math; floats are forbidden and fail compilation (§5.2 Integer‑Only Numeric Policy).
- **Leg construction**: no expressions in legs; each leg field resolves to a variable or allowed selector (§5.2 Variable‑only legs).
- **Leg conditions**: `condition` references a boolean variable computed in CALCULATE; use `always_true` for unconditional legs (§5.2 Condition variable).
- **Same‑ledger enforcement**: per‑transfer same‑ledger is mandatory; cross‑entity/currency requires explicit capabilities (§6.1.8).
- **Rollup restrictions**: direct posting to rollup accounts is forbidden unless `manual_rollup_override` is present (§6.1.8).
- **Metadata schema validation**: `extra_metadata` is validated against every template's `metadata_schema` (§6.1.3).
- **Single external custodian**: each event MUST involve at most one external custodian across all generated transfers. The "internal group" consists of entities that own ledgers for the event's accounts; any custodian not in this group is "external". Events with multiple external custodians are rejected (§13.1).

#### Convenience aliases (no abbreviations)

- `accounts.<purpose>.id` → `accounts.<purpose>.account_id`
- `accounts.<purpose>.ledger_id` → `accounts.<purpose>.ledger.ledger_id`
- `accounts.<purpose>.currency` → `accounts.<purpose>.ledger.currency.currency_code`
- `accounts.<purpose>.account_type` → `accounts.<purpose>.account_code.account_type`

#### Status management

- Template status is derived from the latest audit slug and exposed on template GET/List: state, revision, effective_date, transition_date, slug_id, content_hash, maker/checker, `is_executable`, and loader error (if any).
- Execution gate: templates SHALL execute only when `state == LIVE` **and** `effective_date ≤ now`. Draft/pending/error templates remain readable but SHALL NOT be used for event processing.
- See §5.6 for audit slug rules.

#### Account code masks

- Masks MAY combine alternatives with `|` (logical OR). Any matching segment satisfies the requirement.
- The bare mask `*` matches any valid account code.
- In any non-bare pattern, `*` matches exactly one decimal digit.
- Pattern length is significant and SHALL match the decimal width of the account code.
- Examples:
  - `2****` matches any 5-digit account code starting with `2`
  - `2***` matches any 4-digit account code starting with `2`
  - `2**1*` matches any 5-digit account code whose first digit is `2` and fourth digit is `1`
  - `20000` matches only account code `20000`

#### Immutability

- LIVE templates are immutable; changes require a **new template with a new ID**. Clients choose `template_id` explicitly.
- See also “Execution gate” (above): templates SHALL execute only when `state == LIVE` **and** `effective_date ≤ now`. Draft/pending/error templates remain readable but SHALL NOT be used for event processing.

#### Account type validation

- Legs MUST declare `debit_account_type` and `credit_account_type` (ASSET, LIABILITY, EQUITY, REVENUE, EXPENSE). Runtime accounts MUST match.

#### Financial statements impact declarations

- Templates SHALL declare the net financial impact per account type in the `financial_statements_impact` array.
- Each impact declaration MUST specify:
  - `account_type`: The account type being affected (ASSET, LIABILITY, EQUITY, REVENUE, EXPENSE)
  - `impact`: The net impact on that account type ("increase", "decrease", "no_impact", "increase_or_no_impact", "decrease_or_no_impact")
- `ledger_scope`: The ledger scope identifier this impact applies to (lower_snake_case string matching a key in `ledger_scope_descriptions`)
  - `description`: Human-readable explanation of the impact
- For templates whose `capabilities` include `allows_intercompany_transfers` or `allows_fx_transfers`:
  - Each ledger scope used in legs MUST have impact declarations for ALL account types touched by legs in that scope.
  - Even if the net impact is zero (debits and credits cancel out), it MUST be declared explicitly as `"impact": "no_impact"` with an explanation.
- The compiler SHALL validate that declared impacts match the actual account type movements per ledger scope.
- **Rationale**: Explicit impact declarations per ledger scope ensure that template authors understand the financial consequences of cross-ledger operations and facilitate audit, compliance, and financial reporting.

#### Ledger scope descriptions

- Templates with the `allows_intercompany_transfers` or `allows_fx_transfers` capabilities SHALL provide `ledger_scope_descriptions`.
- `ledger_scope_descriptions` SHALL be an object whose keys are **lower_snake_case** ledger scope identifiers (for example `funding_ledger`, `payment_currency_ledger`). There is no upper bound on the number of scopes.
- Every ledger scope referenced by legs or financial statement impacts SHALL have a corresponding non-empty description entry, and no description entry MAY exist without at least one leg or impact using that scope.
- The content of each description is template-defined. Common patterns include:
  - `funding_ledger`: source ledger (originating currency, sender entity, or operating ledger)
  - `payment_currency_ledger`: conversion ledger (FX conversion, nostro funding, spread recognition)
  - `payment_ledger`: settlement ledger (counterparty payment execution)
- These descriptions feed UI and generated documentation so operators understand the business meaning behind each scope.

#### Treatment type catalogue & rules

- `TreatmentType` is a flat, global enumeration whose values are immutable once released.
- Naming SHALL use concise `snake_case`, consistent family prefixes (for example, `revaluation_*`, `derivative_*`, `crypto_*`), and suffixes indicating P&L direction (`*_revenue` / `*_income` vs. `*_cost` / `*_expense`, `*_provision` vs. `*_release`).
- Each posting carries exactly one treatment type. `adjustment_manual` is reserved for controlled corrective entries with explicit approval; business templates MUST NOT emit it.
- Period-close sweeps SHALL use `period_close_allocation` when moving P&L into equity.
- Template authors SHOULD expose any supporting metadata the policy requires for a chosen treatment via `metadata_schema` (for example, instrument identifiers for valuation adjustments or tax attributes for tax treatments).
- The catalogue covers several families; the JSON Schema (`docs/schemas/common/$defs.schema.json`) lists the authoritative values. Representative examples:
  - **Core operations** – cash movement, loan principal flows (`loan_principal_disbursement`, `loan_principal_repayment`), customer funding (`customer_deposit_receipt`, `customer_withdrawal`), internal movements (`internal_transfer`, `opening_balance_adjustment`, `system_correction`), fee recognition, interest accruals, and impairment lifecycle tags.
  - **FX & valuation** – unrealised and realised P&L (`revaluation_fx`, `realized_fair_value`, `day_one_pnl`, hedge adjustments).
  - **Capital & taxes** – equity events and tax flows (`capital_injection`, `dividend_distribution`, `tax_expense`, `withholding_tax`).
  - **Payments & cards** – dispute outcomes and card-related economics (`chargeback`, `representment`, `fee_rebate`, plus fee/commission treatments with channel metadata).
  - **Brokerage, commissions, and venue fees** – use the generic commission treatments (`commission_revenue`, `commission_cost`) with venue metadata as needed.
  - **Securities / repo / securities lending** – coupon/dividend income, repo interest, stock borrow/lend fees, settlement penalties.
  - **Derivatives & collateral** – margining, collateral movements, and valuation adjustments (`derivative_initial_margin`, `collateral_pledge`, `cva_adjustment`).
  - **Investments & crypto** – investment principal flows (`investment_principal_purchase`, `investment_principal_sale`) plus crypto protocol events (`crypto_network_fee`, `crypto_staking_reward_income`, `crypto_chain_reorg_adjustment`).
- Template metadata such as `fee_kind`, `network`, `execution_role`, `venue`, `instrument_type`, or `product_code` SHOULD capture channel- or product-specific detail that would otherwise require bespoke treatment names.

#### Loading logic

- Load all templates at startup.
- Activation: A template is active iff `state == LIVE` and `effective_date ≤ now`. `DEPRECATED` templates are not active.
- Template store reloads occur at process startup and when the administration endpoint is invoked. There is no automatic scheduler; operators MUST trigger a reload after template state changes go live.
- **LIVE templates MUST include validating examples**. Load fails if examples don't validate. If Ledger Service is unavailable at startup, the service SHALL delay readiness until validation completes.

### 5.3 Rule

Formal schema: `../schemas/domain/rule.schema.json`

Rules have no intrinsic status; status comes from audit slugs. Rules evaluate CEL against events; when true, the action (REJECT) is applied.

### 5.4 Transfer (Output to TigerBeetle)

Formal schema: `../schemas/domain/tigerbeetle-transfer.schema.json`
Human-readable format: `../schemas/domain/expected-transfer.schema.json`
All timestamps MUST be UTC.

#### Field mapping

- `code` := `template_id` (u16, **1..65535**)
- `ledger` := debit account's `ledger_id` (u32)
- `user_data_32` := Contains transfer metadata flags and leg index
- `user_data_64` := `occurred_at` (Unix timestamp in milliseconds since epoch, unsigned)
- `user_data_128` := `event_id` (128‑bit)
- When TigerBeetle rejects a transfer with `TransferDebitAccountNotFound` or `TransferCreditAccountNotFound`, the adapter SHALL look up the missing account(s) via the Ledger Service for the given site, create business accounts with the History flag (dedup accounts without History on the configured dedup ledger), and retry the batch exactly once before surfacing the error.

#### Transfer flags

Transfers support atomic linking and other coordination patterns as required by the ledger system. See Technical Specification for implementation details.

### 5.5 Domain Models (from Ledger Service OpenAPI)

Ledger Service is the source of truth for accounts, ledgers, entities, currencies, and related objects.

#### Account immutability

- Account properties NEVER change after creation.
- Balances are managed by TB and are not cached.
- “Changing” an account requires creating a new account.

#### Generated models _(examples)_

- `EnrichedAccount`, `EnrichedAccountResponse`, `EnrichedLedger`, `EnrichedEntity`, `AccountCode`, `Country`, `Currency`

#### EnrichedAccount highlights

- Identity: `account_id`, `name`, `external_id`
- Properties: `is_rollup`, `is_presentable`, `is_internal_gl_only`
- Nested references: `ledger`, `entity`, `account_code`
- Extra metadata: `map[string]interface{}`
- Currency scale: `ledger.currency.scale` (e.g., USD=2, JPY=0, BTC=8)

#### Rollup indicators

- `is_rollup=true` on accounts; `has_rollup=true` on account codes

### 5.6 Audit Slug (Maker‑Checker)

Formal schema: `../schemas/domain/audit-slug.schema.json`

#### States DRAFT, PENDING_APPROVAL, LIVE, DEPRECATED, REJECTED, DELETED

#### Transitions

- DRAFT → PENDING_APPROVAL (submit)
- PENDING_APPROVAL → LIVE (approve)
- PENDING_APPROVAL → REJECTED (reject)
- REJECTED → DRAFT (revise)
- DRAFT → DELETED (delete)
- DELETED → DRAFT (undelete)
- LIVE → DEPRECATED (superseded)

#### Rules

- Separation of duties: maker ≠ checker
- LIVE requires checker approval
- Full audit trail with content hash integrity
- Delete and undelete are allowed only from DRAFT and DELETED respectively; LIVE templates can only be deprecated.
- Deprecation, deletion, and undeletion do not re‑validate template payloads or examples; validation happens during submit/approve and template load.
- **Service boundary**: This Event Processing Service reads and writes audit slugs to manage template lifecycle via the admin endpoints, including maker‑checker transitions and effective date rules.
- Effective dates for `LIVE` transitions MUST be at least T+48h from approval. Operators SHALL trigger a template reload (via restart or admin endpoint) once state transitions become effective. No quiesce barrier is required.
- `$schema` handling: the service SHALL emit the canonical audit-slug schema URL when writing slugs and SHALL reject missing or non-canonical `$schema` values on read.

### 5.7 Event and Audit Storage

#### Event Store

- Only successfully processed events are stored (after TB confirmation)
- Failed events are logged (and optionally sent to a DLQ) but not stored in the Event Store
- Key: `event_id` (UUIDv7, canonical string)
- Structured serialization with schema registry
- Envelope: **raw event only** + canonical hashes (see §6.1.11)
- TigerBeetle transfer IDs are returned to clients immediately and can be fetched later by querying TigerBeetle via the transfer lookup API. This design keeps TigerBeetle as the canonical data store for transfers; the event outbox never persists transfer IDs.

#### Audit Store Interface

The Event Processing Service persists and queries audit slugs by template ID(s) to determine template usability (LIVE and effective). Slug lifecycle operations and consistency guarantees are handled by this service via the template administration endpoints.

- Persistence with no data loss
- Queries: by `(model_name, model_id, state)`, latest by model, pending approvals, history ordered by transition time
- Writes: insert‑only; atomic transitions with validation
- Strong consistency; permanent retention

### 5.8 Special System Entities

#### Deduplication Ledger

- For each `site_id`, the service uses a deterministic “dedup ledger” reserved for dummy transfers.
- Two deterministic system accounts exist on this ledger (dedup debit/credit) and are used only for the dummy transfer.
- **Deduplication scope is global**: TigerBeetle enforces transfer ID uniqueness at the cluster level, and the dummy transfer uses `id = event_id`. Therefore `event_id` (UUIDv7) MUST be globally unique across all sites sharing the TigerBeetle cluster.

#### System Accounts for Deduplication

- The service provisions (or verifies) the dedup ledger and both accounts during bootstrap via the Ledger Service; failures prevent startup.

---

## 6. Core Functional Requirements _(normative)_

### 6.1 Event Processing Pipeline

#### 6.1.0 Event independence and linkage _(informative)_

This subsection restates existing requirements for quick reference.

- **Stateless core**: The system SHALL maintain pure calculation from inputs to transfers (see §3).
- `original_event_id` / `previous_event_id`: These fields do not affect processing behaviour; they allow downstream analytics to reconstruct event sequences without querying additional systems (see §5.1).

#### 6.1.1 Deduplication Strategy

- **Design**: Use a dummy transfer with `id = event_id` on the dedup ledger.
- **Front-of-house success cache**: The service SHALL short-circuit duplicates that were already committed by consulting an in-memory success cache **before** TigerBeetle is touched. Defaults: 50,000 recent successes retained for ~10 minutes via bucketed rotation; duplicates seen within this window return HTTP 409 (RFC 9457 Problem Details) immediately. After expiry, deduplication still succeeds via the dummy transfer.
- **Flow**:
  1. Validate structure and business rules
  2. Generate transfers via templates
  3. Create batch: first dummy, then business transfers (all linked)
  4. Submit linked batch atomically to TB
  5. If dummy fails with “exists,” the event is a duplicate
  6. Return appropriate response

- **Behaviors**:
  - Same `event_id` is rejected as duplicate (permanent)
  - Exactly‑once even on crashes

#### 6.1.2 Deduplication Implementation

- **Special ledger**: deterministic per site; holds two deterministic system accounts
- **Atomic linked batch**:
  - Deduplication transfer using `event_id` as the transfer ID
  - Business transfers from template processing
  - The dummy transfer participates in the same atomic chain as business transfers as defined by the TigerBeetle atomic‑linking rules; the entire batch MUST commit or abort as a unit

- **Guarantees**: zero duplicates, atomicity, single round‑trip, minimal overhead
- **Errors**:
  - Index 0 `"exists"` → duplicate event
  - `"exists"` at index > 0 is unexpected (UUIDv7 collision); critical error

#### 6.1.3 Input Validation

- Structure: required fields; UUID formats; positive integer `amount`; non‑empty `accounts`
- Metadata: validate `extra_metadata` against every template’s `metadata_schema` (types, requireds, `additionalProperties`)

#### 6.1.4 Template Loading and Processing

#### Multi‑template atomicity & isolation

- All templates MUST validate or none are processed.
- All transfers from all templates are submitted in one atomic batch.
- If any template fails in ENRICH/CALCULATE/VALIDATE/GENERATE, the entire event fails.
- TigerBeetle supports **8191** transfers per linked batch; the service reserves one slot for the dummy transfer, so an event MUST NOT produce more than **8190** business transfers.
- Each template has an isolated variable namespace; no cross‑template references.

#### Fetch templates

- Load by `template_ids` and `site_id`.
- State MUST be LIVE and `effective_date` ≤ now; otherwise 404.

#### Compile (on load)

- Validate `metadata_schema`; compile CEL; verify legs only reference variables or allowed selectors; cache compiled forms.

#### Phases

- **ENRICH**: resolve account IDs; fetch enriched account data (with caching); assemble enriched event context
- **CALCULATE**: evaluate variables in order; variables can reference prior variables and enriched context
- **VALIDATE**: evaluate validation expressions; fail fast
- **GENERATE**: evaluate leg conditions; generate transfers; ERROR if any amount < 1

Templates for the same event SHALL run in parallel; outputs are combined into one atomic batch.

#### 6.1.5 Data Access and Caching

#### Static reference data

- Load at startup: ledgers, entities, account codes, countries, currencies; periodic refresh (hot swap) daily or on demand to capture new entries

#### Account data

- Clients send **internal account IDs**; fetch enriched accounts for validations
- Non‑client/system reference data (<10k) cached fully
- Client account cache: separate, bounded with eviction policy; size configurable
- No TTL needed (existing accounts are immutable); cache refresh captures newly created accounts
- Manual cache invalidation API deferred to future version

#### Batch support

- Coalesce misses; batch fetch from Ledger Service

#### 6.1.6 Amount Handling & Integer Arithmetic

- All amounts are integer minor units and MUST be > 0
- Currency scale at `account.ledger.currency.scale`
- Basis points only; NEVER floating point
- Percent formula: `amount * bps / 10000` with banker’s rounding
- Floating point literals/functions are forbidden; template compile MUST fail on detection
- Banker's rounding function available for integer division with proper rounding

#### 6.1.7 Expression Evaluation

#### CEL context

The CEL evaluation context contains:

- `event`: The original event request with all its fields (event_id, template_ids, site_id, occurred_at, extra_metadata, and any other fields like amount if provided)
- `accounts`: Map of purpose to enriched account objects (e.g., accounts.from, accounts.to, accounts.fee - where each contains the full EnrichedAccount data including ledgers, entity, currency)
- `always_true`: Built-in boolean constant (true) for unconditional leg conditions

Example context for an event with amount field and three account purposes (EnrichedAccount objects shown as placeholders for brevity):

```json
{
  "event": {
    "event_id": "01234567-89ab-cdef-0123-456789abcdef",
    "template_ids": [132],
    "site_id": 1,
    "amount": 5000,
    "occurred_at": 1700000000000,
    "extra_metadata": { "reference": "INV-001" }
  },
  "accounts": {
    "from": {
      "account_id": "01920000-0000-7000-8000-000000000001",
      "name": "Customer Checking Account",
      "ledger": {
        "ledger_id": 1,
        "currency": { "currency_code": "USD", "scale": 2 }
      }
    },
    "to": {
      "account_id": "01920000-0000-7000-8000-000000000002",
      "name": "Merchant Settlement Account",
      "ledger": {
        "ledger_id": 1,
        "currency": { "currency_code": "USD", "scale": 2 }
      }
    },
    "fee": {
      "account_id": "01920000-0000-7000-8000-00000000000f",
      "name": "Fee Income Account",
      "ledger": {
        "ledger_id": 1,
        "currency": { "currency_code": "USD", "scale": 2 }
      }
    }
  },
  "always_true": true
}
```

#### Custom function

Banker's rounding function for integer division (see Technical Specification section 18.5)

#### Evaluation order

1. Variables (sequential)
2. Validations (parallel)
3. Leg condition check (boolean variable)
4. Leg symbol resolution (variables or allowed selectors) - only for legs where condition is true

#### Available functions

Standard integer math, string, type conversion, and collection helpers (float/double banned).

#### 6.1.8 Business Rule Validation

- **Site rules**: load by `site_id` and effective date; REJECT on true
- **Template validations**: each MUST be true; fail fast
- **Rollup account check**: forbid direct posting to rollup accounts unless the `manual_rollup_override` directive is present
  - Valid use cases for override: Initial balance loading, data migration, error corrections requiring manual adjustment
  - Override MUST be audited and justified in event metadata
- **Cross‑entity/currency**:
  - If entities differ → template MUST declare `allows_intercompany_transfers`
  - If currencies differ → template MUST declare `allows_fx_transfers`
  - Default (neither capability present): all accounts in the event MUST be from the same ledger
  - **Per‑transfer same‑ledger is mandatory**

#### 6.1.8.1 Cross‑ledger modeling

Cross‑ledger activity SHALL be modeled as paired same‑ledger legs using designated intercompany/nostro/vostro accounts on each ledger.

- Templates that use these patterns MUST declare `allows_intercompany_transfers` and/or `allows_fx_transfers` as applicable.
- `ledger_scope_descriptions` MUST name the clearing/settlement scopes (for example, `nostro_ledger`, `vostro_ledger`, `intercompany_ledger`) so the accounting intent is explicit.

#### 6.1.9 Timestamp Handling & Overrides

- `event.occurred_at` is required.
- When the `seed_mode` directive is present, the client MUST provide `ledger_timestamp` (Unix ms); the engine SHALL use that value as `occurred_at` for generated transfers.
- All times are Unix timestamps in milliseconds since the Unix epoch.
- TB `recorded_at` is TB cluster time; we do not import timestamps into TB in normal flow.

#### 6.1.10 Rollup Transfer Generation

- Identify rollup need via `account.account_code.has_rollup == true`
- Skip rollup generation if:
  - The `skip_rollup_processing` directive is present (used for testing or when rollups are handled externally)
  - Both debit and credit accounts have the same account code (internal transfers within same category)
- Fetch memo account and rollup parent account (`is_rollup=true`) per ledger; both MUST exist
- Generate rollups per ledger after business legs; set:
  - `user_data_128` = `event_id`
  - `user_data_64` = `occurred_at`
  - `user_data_32` flags with `IS_ROLLUP` and `leg_index`

- Rollups are per ledger only (never across ledgers)

#### 6.1.11 Event Publication: event outbox, Event Store, Lifeboat

1. Event outbox / durable PREPARED: After validation and transfer rendering, the service SHALL persist a PREPARED record to the local fsync’d Write‑Ahead Log (event outbox) before submitting to TigerBeetle.
2. TB commit: Only after durable PREPARED succeeds SHALL the service submit the transfer batch to TigerBeetle.
3. Logical COMMITTED: On TigerBeetle success, the service SHALL transition the record to COMMITTED and return HTTP 200. The acknowledgement `status` for `POST /events` is always `COMMITTED`.
4. Publish: Event Store publication is fully asynchronous and driven only by the outbox publisher.
   - On success: mark the outbox record PUBLISHED and truncate it later per retention policy.
   - On failure: keep the record COMMITTED, apply retry backoff, and retry until success.
5. No panic: If Event Store publish fails, do not panic. Keep the durable outbox record and report degraded health/metrics.
6. Background publisher: The service runs background outbox-publisher workers that lease batches of COMMITTED records, publish them, then repeat until the backlog is empty.
7. Startup recovery: On restart, replay PREPARED records to the ledger first. After recovery completes, the outbox publisher drains COMMITTED records to the Event Store. Duplicate replay responses mark the record COMMITTED; deterministic ledger business errors mark the record FAILED; transient/system errors leave the record PREPARED for a subsequent retry.

> In this version the local outbox is the authoritative recovery mechanism; there is no separate lifeboat queue.

##### 6.1.11.1 Outbox recovery replay semantics _(informative)_

This is a restatement of Step 7 “Startup recovery” above.

- On restart, replay PREPARED records to the ledger before serving traffic.
- Duplicate responses mark the record COMMITTED; successful replays do the same.
- Deterministic business errors mark the record FAILED to prevent infinite retries; transient/system errors leave the record PREPARED for a subsequent retry.
- After recovery finishes, background outbox-publisher workers drain COMMITTED records to the Event Store.

The service SHALL implement a transactional outbox with a local, fsync‑backed Write‑Ahead Log (event outbox).

For each request, the service SHALL:

- Persist a PREPARED event outbox record and fsync it before submitting to TigerBeetle.
- On TigerBeetle success, transition the record to COMMITTED and return success.
- Publish the envelope to the Event Store only from background outbox-publisher workers; on success, mark PUBLISHED.
- On publish failure, retain COMMITTED and retry with backoff.

After TB success, the service SHALL return HTTP 200 with:

- `status="COMMITTED"`; publication is asynchronous and the envelope SHALL remain COMMITTED in the event outbox until the publisher marks it PUBLISHED.

Because publishing is asynchronous, clients SHOULD assume `status="COMMITTED"` and rely on the transfer lookup APIs to inspect posted transfers. The Event Store is not required to provide read‑after‑write consistency.

Transfer identifiers SHALL be retrievable via `GET /events/{event_id}/transfers`, which queries TigerBeetle for transfers whose `user_data_128` matches the event ID and returns them in submission order. If TigerBeetle contains no transfers for the event ID, the endpoint SHALL return HTTP 404 (RFC 9457 Problem Details). No duplicate copy of transfer identifiers is kept in the event outbox; durability comes from TigerBeetle itself.

**REQ-182**: WHEN TigerBeetle returns successfully and the event is durably recoverable via the outbox, the system SHALL return an acknowledgement with `status="COMMITTED"`.
_Source: Section 6.1.11 | Testable: Yes_

**REQ-183**: The system SHALL expose `GET /events/{event_id}/transfers` that retrieves TigerBeetle transfer IDs for the event in submission order.
_Source: Sections 6.1.11; 6.2.4 | Testable: Yes_

**REQ-184**: WHEN TigerBeetle has no transfers for an event ID, `GET /events/{event_id}/transfers` SHALL return HTTP 404 (RFC 9457 Problem Details).
_Source: Sections 6.1.11; 6.2.4 | Testable: Yes_

### 6.2 API Endpoints _(normative)_

#### 6.2.1 Process Event

`POST /events`
**Body**: EventRequest

#### Processing

1. Validate request structure and metadata
   - On failure: return HTTP 400 (RFC 9457 Problem Details) immediately
2. Enrich accounts; run CALCULATE and VALIDATE
   - On account not found: return HTTP 422 (RFC 9457 Problem Details)
   - On validation failure: return HTTP 422 (RFC 9457 Problem Details)
   - On CEL failure: return HTTP 422 (RFC 9457 Problem Details)
3. Generate legs; check same‑ledger and account types; amounts > 0
   - On ledger mismatch: reject the request (RFC 9457 Problem Details)
   - On rollup violation: reject the request (RFC 9457 Problem Details)
4. Prepare a single atomic TB batch (chain):
   - Index 0: dummy transfer `id = event_id` on the configured dedup ledger
   - Index 1..n: business transfers
   - Atomicity is enforced using TigerBeetle’s atomic‑linking rules; the entire batch commits or aborts as a unit

5. Persist a PREPARED outbox record and fsync it locally
   - On failure: return HTTP 503 (RFC 9457 Problem Details) and do not submit to TB
6. Submit the batch atomically to TB (timeout: 10 seconds)
   - On duplicate event: return HTTP 409 (RFC 9457 Problem Details)
   - On insufficient funds: reject the request (RFC 9457 Problem Details)
7. On TB success: transition the outbox record to COMMITTED and return HTTP 200 with `status="COMMITTED"`. Publication to the Event Store happens asynchronously via the outbox publisher.
8. On failure before TB commit: map TB indexed errors to HTTP status codes and Problem Details as defined in section 8.2; do not mark event outbox COMMITTED or publish/store

#### Responses

- Success: `../schemas/api/events-post-response.schema.json` (acknowledgement containing `status="COMMITTED"`, `event_id`, and `processed_at`). Event Store is not required to provide read‑after‑write consistency and clients SHOULD use the transfer lookup endpoints for posted transfer details.
- Error: `../schemas/api/problem-details.schema.json` (RFC 9457 Problem Details)

#### 6.2.2 Event Validation (Dry Run)

`POST /events:validate`
Runs full pipeline **without** TB submission or storage. Requires Ledger Service connectivity. Maximum batch size: 1 event per validation request.

#### Event Validation Responses

- Success: `../schemas/api/event-validations-post-response.schema.json`
- Error: Various validation and not found errors
- The `trace` field is omitted (`null`) for this endpoint.

#### 6.2.3 Event Validation Trace (Explain Mode)

`POST /events:validate:explain`

- Request body is identical to `POST /events:validate`.
- Executes the same deterministic validation pipeline and additionally collects an execution trace containing:
  - Variables: evaluated CEL expressions and their resolved values per template
  - Validations: pass/fail status with errors for each validation expression
  - Legs: condition outcomes, skip reasons, resolved debit/credit account IDs, and generated amounts
- Intended for debugging and observability tooling; response size grows with the number of templates and legs, so clients SHOULD scope usage to troubleshooting scenarios.

**Response**

- Success: `../schemas/api/event-validations-post-response.schema.json` with the `trace` object populated
- Error: same error taxonomy as `POST /events:validate`

**REQ-185**: The system SHALL provide `POST /events:validate:explain`, returning the standard validation response augmented with a deterministic execution trace.
_Source: Section 6.2.3 | Testable: Yes_

#### 6.2.4 Event Transfer Retrieval

`GET /events/{event_id}/transfers`

- Path parameter `event_id` MUST be a UUIDv7; malformed IDs SHALL result in HTTP 400 (RFC 9457 Problem Details).
- Query parameters:
  - `limit` (optional, default 200, max 8190). Number of transfers to return per page.
  - `timestamp_max` (optional). Upper bound (inclusive) for transfer timestamps in Unix milliseconds (same unit as `occurred_at` and the `timestamp` field returned on transfers).
  - `cursor` (optional). Base64url cursor returned by a previous page; contains the last transfer’s pagination position and the frozen `timestamp_max` fence for deterministic paging.
  - `enrich`/`site_id` behave as before (see §6.2.1).
- Results are ordered by TigerBeetle timestamp descending (newest first).
- The response echoes the effective `timestamp_max`. If the client omits it on the first page, the server SHALL derive it from the newest matched transfer and return it so callers can reuse the same fence on subsequent page requests.

**Response**

- Success: `../schemas/api/event-transfers-get-response.schema.json` (includes `transfers[]`, `count`, `has_more`, `next_cursor`, and `timestamp_max`).
- Empty result sets SHALL return HTTP 200 with `transfers=[]`, `count=0`, `has_more=false`, and `next_cursor=null`.

##### Account Transfer Lookup

`GET /accounts/{account_id}/transfers`

- Path parameter `account_id` MUST be a UUID; malformed IDs SHALL result in HTTP 400 (RFC 9457 Problem Details).
- Query parameters:
  - `side` (optional, enum: `debit` | `credit`). Filter by transfer side. Omit for both sides.
  - `template_id` (optional). Filter by template ID.
  - `limit` (optional, server default 200, max 8190). Number of transfers to return per page.
  - `timestamp_max` (optional). Upper bound (inclusive) for transfer timestamps in Unix milliseconds (same unit as `occurred_at` and the `timestamp` field returned on transfers).
  - `cursor` (optional). Base64url cursor returned by a previous page.
- Results are ordered by TigerBeetle timestamp descending (newest first) and use the same paging contract as event transfer retrieval, including `timestamp_max` echoing.

**Response**

- Success: `api.TransfersListResponse` (includes `transfers[]`, `count`, `has_more`, `next_cursor`, and `timestamp_max`).
- Empty result sets SHALL return HTTP 200 with `transfers=[]`, `count=0`, `has_more=false`, and `next_cursor=null`.

##### Generic Transfer Query

`GET /transfers`

- At least one of `template_id` or `ledger_id` MUST be supplied.
- Query parameters:
  - `template_id` (optional). Filter by template ID.
  - `ledger_id` (optional). Filter by ledger ID.
  - `limit` (optional, default 200, max 8190). Number of transfers to return per page.
  - `timestamp_max` (optional). Upper bound (inclusive) for transfer timestamps in Unix milliseconds (same unit as `occurred_at` and the `timestamp` field returned on transfers).
  - `cursor` (optional). Base64url cursor returned by a previous page.
- Results use the same descending-order paging contract as event/account transfer retrieval.

**Response**

- Success: `api.TransfersListResponse`
- Empty result sets SHALL return HTTP 200 with `transfers=[]`, `count=0`, `has_more=false`, and `next_cursor=null`.

##### Transfer Count Endpoints

- `GET /events/{event_id}/transfers:count`
- `GET /accounts/{account_id}/transfers:count`
- `GET /transfers:count`

- Count endpoints accept the same filter dimensions as their sibling list endpoints, except they do NOT accept `limit` or `cursor`.
- The server SHALL issue a single TigerBeetle query capped at one probe over the public page limit. The response includes:
  - `count`: exact count when the result set fits within the cap, otherwise the cap value (`8190`)
  - `is_capped`: `true` when more matches may exist beyond `count`
  - `timestamp_max`: the effective upper-bound fence for follow-up paging
- Empty result sets SHALL return HTTP 200 with `count=0` and `is_capped=false`.

#### 6.2.5 Event Lookup by Original Event ID

`GET /events?original_event_id={original_event_id}`

- Query parameter `original_event_id` MUST be UUIDv7; malformed IDs SHALL return HTTP 400 (RFC 9457 Problem Details).
- Returns all events that share the provided `original_event_id`, ordered by commit time (`linked_at`) ascending so callers can replay the lineage in ingestion order.
- **Implementation note**: The PostgreSQL event store populates the `event_original_index` table transactionally. Rocket Store/Kafka deployments MUST persist the same mapping before this endpoint can be enabled; until then the service SHALL respond with HTTP 501 (RFC 9457 Problem Details).

**Response**

- Success: JSON object containing `original_event_id`, `count`, and `events[]`.
- Error: `../schemas/api/problem-details.schema.json` with HTTP 501 when the configured Event Store cannot serve linkage queries.

#### 6.2.6 Template Validation

`POST /templates:validate`

#### Processing

1. **Enhanced Validation with Structured Errors**: Compile template CEL expressions and validate structure with detailed, user-friendly error messages
   - Categorizes errors by type: JSON_STRUCTURE, CEL_EXPRESSION, BUSINESS_LOGIC, REFERENCE, TYPE, RANGE
   - Provides field-specific error locations (e.g., `variables[0].expression`, `legs[2].amount`)
   - Includes actionable suggestions for fixing validation issues
   - Returns both simple string errors (`validation_errors`) and detailed structured errors (`validation_details`)
2. If `event_request` provided:
   - Generate transfers using the template
   - Convert to expected-transfer format (with account names, codes, types)
3. If `expected_transfers` provided:
   - Compare generated vs expected (ignoring transfer IDs)
   - Report any mismatches
4. If `expected_transfers` NOT provided but `event_request` IS provided:
   - Auto-generate expected transfers from the event
   - Return them in the response for the user to copy

#### Response

- Success: `../schemas/api/template-validations-post-response.schema.json`
- **Enhanced Error Reporting**:
  - `validation_errors`: Simple string array for basic tools
  - `validation_details`: Structured error objects with categories, field paths, suggestions, and technical details
- Includes `generated_expected_transfers` array in expected-transfer format
- Validation results and any mismatches

#### 6.2.7 Template Refresh Cache

`POST /sites/{site_id}/templates:refresh-cache`
Fetch from storage; validate CEL; check audit slug effective dates; replace in‑memory cache (site-scoped). Optional body may specify `template_ids` to reload a subset.

**Response**: `api.TemplateCacheRefreshResponse`

Refresh operations are read‑only: fetch current artifacts from audit‑controlled storage, validate/compile, and hot‑swap in memory. No writes or state transitions are performed by this endpoint.

#### 6.2.8 Pre‑Submission Validation Rules

#### Phase 1: Request (no deps)

- Structure validation; errors like “Missing required field …”
- Account purpose extraction/verification

#### Phase 2: Enrichment (parallel)

- Account enrichment; error “Account {id} not found”
- Metadata schema validation per template

#### Phase 3: Business logic (needs enrichment)

- Template validations; error “Validation {name} failed …”
- Site business rules; error “Rule {rule_id} rejected …”
- Rollup restriction; error naming offending rollup account; explicit override via metadata

#### Phase 4: Transfer generation

- Account type validation vs leg expectations
- Same‑ledger validation per transfer
- Amount > 0

#### Performance concurrent validations; fail‑fast; batch account fetches; pre‑compiled expressions

#### 6.2.9 Cache Refresh

`POST /sites/{site_id}/templates:refresh-cache`

- Atomic swap; detailed report.
- Template refresh supports **subset reloads** via optional body:

  ```json
  { "template_ids": [<uint16>...] }
  ```

  - If body is empty, reload ALL templates for the site.
  - If `template_ids` is provided, reload only that subset for the site.

- Response summary SHALL include `mode` (`full`/`subset`), `cache_total` after reload, and echoes of `site_id`/`template_ids` for subset requests.

### 6.3 Storage and Caching Strategy _(normative)_

- Static reference data: load at startup; periodic hot refresh
- Dynamic account data: cached with bounded eviction for client accounts; system accounts retained aggressively; batch fetching

### 6.4 External Service Integrations _(normative)_

#### 6.4.1 Ledger Service

Purpose: fetch static reference and enriched account info
Auth: API key
API Version: v1.0
Operations: startup loads; get enriched accounts (single/batch); query by filters

#### 6.4.2 Ledger System (TigerBeetle)

Purpose: submit atomic batches of transfers
Operations: submit linked batches; unique transfer ID enforcement; per‑index error results
Guarantees: atomicity for linked batches; persistent connections with auto‑reconnect

#### 6.4.3 Event Store

Purpose: store successful event envelopes
Key: `event_id` (UUIDv7)
Idempotency: at storage key

#### 6.4.4 Outbox publisher

Purpose: drain durable COMMITTED outbox records to the Event Store
Implementation: background worker pool with outbox reservations/leases

---

## 7. Service Initialization _(normative)_

### 7.1 Startup Validation

#### 7.1.1 Infrastructure Verification

#### Dedup & rollup infrastructure verification

1. **Dedup**: ensure the per‑site dedup infrastructure exists (auto‑provision if missing)
   - Create/verify the dedup account code
   - Create/verify the per‑site dedup entity
   - Create/verify the per‑site dedup ledger
   - Create/verify the per‑site dedup debit/credit accounts
   - On any failure (ledger service unavailable, permission denied, validation error): log a fatal bootstrap error and exit
2. **Rollup**: per ledger, verify memo account and rollup parent accounts
   - Rollup accounts are identified by account code configuration

- For each account code with `has_rollup=true`, the system looks for the corresponding account with `is_rollup=true` on each ledger
- The event service now enforces this at startup: it fetches account-code metadata from the ledger service and verifies that every configured ledger has a rollup parent for every code flagged `has_rollup=true`. Missing parents halt startup with an explicit error.
- The specific account codes that require rollup accounts are configurable per deployment
  - On missing rollup accounts: log "FATAL: Rollup account for code {code} not found on ledger {id}" and exit
  - On missing memo accounts: log "FATAL: Memo account not found on ledger {id}" and exit

3. Log counts and configuration
   - Log "Verified {n} dedup accounts, {m} rollup accounts across {p} ledgers"

Service MUST NOT start if any validation fails

Service MUST NOT start if any required memo or rollup parent account is missing for any ledger configured for rollups.

**Recovery procedure**: Fix account configuration in Ledger Service and restart

#### 7.1.2 Template/Rule Loading

- Load templates/rules; compile CEL
- Validate LIVE template examples (requires Ledger Service)
  - Template examples are included in the `examples` field of the template JSON (see `../schemas/domain/template.schema.json`)
  - Each example contains an `event_request` and `expected_transfers` for validation
- If any LIVE example fails validation, do not load that template
- If **zero** templates load, refuse to start service
- Readiness requires completion of validations
- This service does not persist or transition templates/rules/slugs; it only validates and caches what it reads.

#### 7.1.3 Cache Initialization

- Initialize account caches; load non‑client accounts (system/rollup/memo); initialize ledger cache; set up cache metrics

---

### 7.2 event outbox/Spool Initialization

- Initialize the event outbox directory and fsync settings; configure a crash‑safe rename protocol.
- Initialize the local durable spool and the background publisher worker.
- The service MUST refuse to start if the event outbox/spool cannot be created with durability guarantees.

---

## 8. Error Handling _(normative)_

### 8.1 Error Response Shape (RFC 9457)

All HTTP error responses (4xx and 5xx) MUST follow RFC 9457 Problem Details:

- `Content-Type: application/problem+json; charset=utf-8`
- `Cache-Control: no-store`
- The JSON body MUST contain ONLY these fields:
  - `type` (string, optional; service-scoped problem type URI under `https://ledger-rocket.dev/event-service/problems/...`)
  - `title` (string, optional; use the HTTP status text)
  - `status` (number, optional; the HTTP status code)
  - `detail` (string, optional; human-readable message)
  - `instance` (string, optional; request path)
  - `errors` (array, optional; structured validation errors for template validation failures)

No additional keys are permitted beyond the fields above (no `error_code`, etc).

The `detail` message SHOULD be explicit and actionable when possible (for example: include missing `account_id` values with `site_id`, or state the required capability such as `allows_intercompany_transfers` / `allows_fx_transfers` when cross-ledger validation fails).

Problem type URIs are documented in `docs/problem-types.md`.

#### Example

```json
{
  "type": "https://ledger-rocket.dev/event-service/problems/events/business-validation-failed",
  "title": "Unprocessable Entity",
  "status": 422,
  "detail": "Business validation failed: amount must be >= 100",
  "instance": "/events"
}
```

```json
{
  "type": "https://ledger-rocket.dev/event-service/problems/events/account-not-found",
  "title": "Unprocessable Entity",
  "status": 422,
  "detail": "account_id not found for site_id=7: 01920000-0000-7000-8000-000000000999",
  "instance": "/events"
}
```

### 8.2 Status Codes (HTTP contract)

The service uses HTTP status codes (not machine-readable error codes) as the stable contract surface:

- `400 Bad Request` — invalid request body, path parameter, or query parameter
- `404 Not Found` — requested resource does not exist
- `409 Conflict` — duplicate event submission (idempotency key already committed)
- `422 Unprocessable Entity` — request is syntactically valid but fails validation/business rules
- `501 Not Implemented` — endpoint/feature is not implemented
- `503 Service Unavailable` — dependency unavailable / service not ready
- `500 Internal Server Error` — unexpected failures

### 8.3 Failure Modes

- Template/rule loading: skip invalid; continue with valid (prevents one faulty template from impacting entire system); alert on high failure rate
- Account enrichment: retry with backoff; fail if critical account missing; no stale fallback
- CEL failure: return detailed error; include line/column when available; fail fast
- Ledger Service failure: service unavailable error; no partial submission
- Event Store publish failure: retain the durable COMMITTED outbox record, retry in background, and surface degradation via backlog metrics/health

---

## 9. Performance Requirements _(normative)_

### 9.1 Throughput

Scale horizontally; process high event volumes; handle bursts. The system MUST support the transaction volumes of global financial institutions. See Technical Specification for specific performance targets and scaling strategies.

### 9.2 Latency

Low response times for most requests with predictable tails. Detailed latency budgets for Ledger Service and TB calls are in the Technical Specification.

---

## 10. Configuration _(informative)_

### 10.1 Required Configuration

| Category    | Item                           | Description                                                 |
| ----------- | ------------------------------ | ----------------------------------------------------------- |
| Storage     | Templates Storage              | Location of template definitions                            |
|             | Rules Storage                  | Location of business rules                                  |
| Event Store | Connection                     | Event store configuration                                   |
|             | Event Envelope Schema Registry | JSON Schema for the event envelope; compatibility: backward |
| Lifeboat    | Queue Config                   | Per-site queue configuration                                |
| Services    | Ledger Service                 | Account data service configuration                          |
|             | Ledger System                  | Double-entry ledger configuration                           |
| Operations  | Service Config                 | Service operational parameters                              |

### 10.2 One‑Time Setup Requirements

- Grant the event-service credentials permission to provision required system resources in the Ledger Service (dedup ledger/accounts, rollup infra)
- Configure memo and rollup parent accounts per ledger
- Configure Event Store and Event Envelope Schema Registry
- Provision durable Event Store infrastructure and size the outbox retention/backpressure settings appropriately

### 10.3 Event Directives

First-class processing toggles are expressed through the `directives[]` array on the event:

- `skip_rollup_processing` (bool) — suppresses rollup generation when external systems handle it.
- `manual_rollup_override` (bool) — allows direct posting to rollup accounts for tightly controlled adjustments; events MUST include audit justification in metadata.
- `seed_mode` (bool) — marks backfill/seed flows and REQUIRES `ledger_timestamp` to supply the authoritative timestamp for generated transfers.

---

## 11. Monitoring and Observability _(informative)_

### 11.1 Metrics

#### Event processing

- Events processed by template and site
- Event processing duration
- Event deduplication rejections
- Transfers created by template and site

#### Performance

- Transfer batch sizes
- Ledger service roundtrip times
- Account cache hit ratios
- CEL evaluation duration by template

#### Dependencies

- Ledger service request counts and status
- Ledger system connection errors
- Event store publish failures
- Lifeboat writes and backlog
- Lifeboat replay counts

#### Alert Thresholds

- Error rate > 1% → ALERT
- P99 latency > 500ms → ALERT
- Cache hit ratio < 80% → ALERT
- Lifeboat backlog > 1000 → ALERT
- Ledger Service errors > 0.1% → ALERT

### 11.2 Logging

No PII is stored or processed. Structured JSON logs include `event_id`, `template_id`, `site_id`, `duration_ms`, `error`, and selected `extra_metadata` fields as configured.

### 11.3 Health Checks

`GET /health` → `../schemas/api/health-get-response.schema.json`

#### Ready when

- Dedup + rollup infrastructure verified
- LIVE templates loaded and validated
- TigerBeetle reachable
- Ledger Service reachable
- event outbox/spool initialized

#### Healthy when

- All Ready conditions are met, and
- Event Store is reachable, and
- Spool backlog is below a configurable threshold

If Event Store is unreachable or spool backlog exceeds the threshold, report degraded/unhealthy.

---

## 12. Security Considerations _(normative)_

### 12.1 Expression Evaluation

CEL is sandboxed; no filesystem/network access; resource limits apply (CPU/time, program size).

### 12.2 Data Validation

Validate UUIDs (v7 for `event_id`); protect against integer overflow; sanitize external data; disallow floats in expressions.

### 12.3 Authentication

- Internal only behind service mesh; mTLS/JWT as per platform
- API key for Ledger Service
- Maker‑checker identity enforced at audit transitions

Maker‑checker identity enforcement occurs in the audit lifecycle service (out of scope).

---

## 13. Critical Requirements Summary _(normative)_

### 13.1 Non‑Negotiable

- Double‑entry balance maintained; each transfer is same‑ledger
- No data loss/corruption
- All amounts are integer minor units; bps with integer division; no float
- CEL only; compiled/validated; no FS/network access from CEL
- Fail‑fast; no partial TB submission; explicit validation errors
- Successfully processed events stored immutably in Event Store; `event_id` is UUIDv7
- LIVE templates MUST have validating examples; invalid/non‑compiling templates rejected at load
- Single external custodian per event: events MUST involve at most one external custodian across all transfers

### 13.2 Performance Critical Paths

- Template/rule loading: in-memory caches; preloaded at startup; manual refresh via admin endpoint
- Account enrichment: batch fetch; pooled connections; timeouts
- Transfer generation: pre‑compiled CEL; minimal allocations

### 13.3 Data Integrity Guarantees

- Deduplication via dummy transfer; TB ID constraints provide idempotency; no duplicates across crashes/restarts
- Transfers: atomicity via linking; same‑ledger enforced; rollup restrictions enforced
- Event store: publish after TB commit; the durable outbox plus background publisher ensures eventual Event Store completeness without reversals
- Hashes recorded (variables/accounts) for forensic verification

### 13.4 Accounting invariants (quick reference) _(informative)_

This section is a restatement of existing requirements in §13.1 and §13.3.

- Double‑entry balance maintained; each transfer is same‑ledger
- All amounts are integer minor units; bps with integer division; no float
- Deduplication via dummy transfer; TB ID constraints provide idempotency; no duplicates across crashes/restarts
- Transfers: atomicity via linking; same‑ledger enforced; rollup restrictions enforced

---

## 14. Time & Identity Semantics _(informative)_

- **`event_id`**: UUIDv7; sortable by creation time; global uniqueness
- **`occurred_at`**: when the real‑world event happened (client‑supplied)
- **`recorded_at`**: TB cluster time when transfers are committed
- **Override**: When the `seed_mode` directive supplies `ledger_timestamp`, that timestamp replaces `occurred_at` for generated transfers (not TB time); there is no max backdate

---

## Appendix A: CEL Expression Examples _(informative)_

### Cel Validations

#### Variables (integer only)

```cel
"amount * 50 / 10000";
"max(100, amount * 50 / 10000)";
"amount > 1000000 ? amount * 100 / 10000 : amount * 50 / 10000";
"base_fee + (amount * fee_basis_points / 10000)";
```

#### Validations

```cel
"amount >= 100 && amount <= 1000000";
"has(accounts.from) && has(accounts.to)";
"has(event.extra_metadata.customer_name)";
"accounts.from.ledger.ledger_id == accounts.to.ledger.ledger_id";
```

#### Condition variables (used by legs)

```cel
"amount > 10000";                         // should_charge_fee
"amount > 1000000";                       // is_high_value_transfer
"!has(event.extra_metadata.fee_waived)";  // apply_fee
```

#### Leg references (no expressions in legs)

```cel
// Allowed: variables or allowed selectors
amount: "transfer_amount"         // variable
debit_account: "accounts.from.account_id"
credit_account: "accounts.to.account_id"
condition: "should_charge_fee"    // boolean variable
```

---

## Appendix B: Event Store Format _(informative)_

Events are stored with the following logical structure:

- Event ID (UUIDv7)
- Occurred at timestamp (Unix milliseconds)
- Site ID
- Template IDs (array)
- Account mappings (purpose to ID)
- Extra metadata (canonical JSON)
- Cryptographic hashes for audit

The exact serialization format is defined in the technical specification.

---

## Appendix C: Limits & Constants _(normative)_

- Template IDs: **1..65535**
- Max templates per event: **256**
- Max legs per template: **64**
- Max legs per event: **8190** (TigerBeetle linked batch capacity **8191**, minus one dummy transfer)

- Max accounts per event: **32**
- Transfer batch size: bounded by the TigerBeetle atomic batch capacity (**8191** transfers); one slot is reserved for the dummy transfer
- Request size: **≤ 64 KB**; response size: **≤ 256 KB**

- Supported UUID for `event_id`: **UUIDv7 only** (accept UUIDv4 nowhere for `event_id`)
- Currency scale: non‑negative integer; no fixed upper bound enforced by this service.
- Overflow guard: The service SHALL prevent integer overflow in all arithmetic (e.g., amount \* bps) given the integer width of the engine/TigerBeetle; on overflow risk the system SHALL fail the request with a clear error.
- Operational note (informative): Choose scales that balance precision with operability; very large scales may constrain maximum representable amounts under fixed‑width integers.
- No other changes required for scale.

---

## Appendix D: Glossary _(informative)_

### Banker's Rounding

Also known as "round half to even" or "convergent rounding". When rounding a number that is exactly halfway between two possible values, round to the nearest even number. This method reduces rounding bias in repeated calculations. Example: 2.5 rounds to 2, 3.5 rounds to 4.

### Durable Outbox

A local durable spool of event envelopes used to guarantee replay and eventual publication. Records transition through PREPARED, COMMITTED, and PUBLISHED states.

### EARS (Easy Approach to Requirements Syntax)

A structured syntax for writing requirements with five patterns:

- **Ubiquitous**: The system SHALL [requirement]
- **Event-driven**: WHEN [trigger] the system SHALL [requirement]
- **State-driven**: WHILE [in state] the system SHALL [requirement]
- **Optional**: WHERE [feature is included] the system SHALL [requirement]
- **Complex**: Combinations of the above patterns

### JCS (JSON Canonicalization Scheme)

RFC 8785. A standard way to serialize JSON objects into a canonical form for consistent hashing and digital signatures. Ensures identical JSON data always produces the same byte sequence.

### LRU (Least Recently Used)

A cache eviction policy that removes the least recently accessed items first when the cache reaches its capacity limit.

### Minor Units

The smallest denomination of a currency, expressed as an integer. For example, cents for USD (scale=2), yen for JPY (scale=0), or satoshis for Bitcoin (scale=8). All amounts in the system are represented in minor units to avoid floating-point arithmetic errors.

### TigerBeetle (TB)

The double-entry accounting database system used as the ledger system.

### UUIDv7

A time-ordered UUID format that embeds a Unix timestamp, making UUIDs sortable by creation time while maintaining uniqueness.

---

## Appendix E: Testable Requirements _(normative)_

This appendix extracts and formalizes all testable requirements from this specification using EARS (Easy Approach to Requirements Syntax) patterns. Each requirement is numbered for traceability and categorized for test planning.

### E.1 Identity and Validation Requirements

**REQ-001**: WHEN an event is received, the system SHALL validate that `event_id` is in UUIDv7 format
_Source: Section 5.1 | Testable: Yes_

**REQ-002**: WHEN `event_id` is not UUIDv7, the system SHALL reject the request with HTTP 400 (RFC 9457 Problem Details).
_Source: Section 5.1 | Testable: Yes_

**REQ-004**: The system SHALL validate that `site_id` is present and is a positive integer
_Source: Section 5.1 | Testable: Yes_

**REQ-005**: WHEN duplicate account purposes are found in the `accounts` map, the system SHALL reject with error "Duplicate account purpose '{purpose}' found"
_Source: Section 5.1 | Testable: Yes_

**REQ-006**: WHEN processing an event, the system SHALL validate `extra_metadata` against the `metadata_schema` of ALL specified templates
_Source: Section 5.1 | Testable: Yes_

**REQ-007**: WHEN `extra_metadata` exceeds 16 KB, the system SHALL reject the request
_Source: Section 5.1 | Testable: Yes_

**REQ-008**: WHEN `extra_metadata` depth exceeds 6 levels, the system SHALL reject the request
_Source: Section 5.1 | Testable: Yes_

### E.2 Template Processing Requirements

**REQ-009**: WHEN loading templates, the system SHALL only activate templates where `state` is LIVE AND `effective_date` ≤ now
_Source: Section 5.2 | Testable: Yes_

**REQ-010**: WHEN a template ID is not found or not effective, the system SHALL reject the request with RFC 9457 Problem Details.
_Source: Section 6.1.4 | Testable: Yes_

**REQ-011**: The system SHALL enforce a maximum of 256 templates per event
_Source: Section 5.1 | Testable: Yes_

**REQ-012**: The system SHALL enforce template IDs in the range 1..65535
_Source: Section 5.1 | Testable: Yes_

**REQ-013**: WHEN any template validation fails, the system SHALL reject the entire event (no partial processing)
_Source: Section 6.1.4 | Testable: Yes_

**REQ-014**: The system SHALL enforce a maximum of 64 legs per template
_Source: Appendix C | Testable: Yes_

**REQ-015**: WHEN loading templates, the system SHALL NOT activate templates where `state` is DEPRECATED or DELETED
_Source: Section 5.2 | Testable: Yes_

**REQ-016**: The system SHALL expose an administrative template reload endpoint so operators can refresh caches on demand
_Source: Section 6.2.7 | Testable: Yes_

**REQ-017**: WHEN loading LIVE templates without validating examples, the system SHALL fail to load that template
_Source: Section 5.2 — Loading logic (LIVE templates MUST include validating examples) | Testable: Yes_

**REQ-155**: The system SHALL require `occurred_at` in every `EventRequest` and SHALL reject requests where it is absent with RFC 9457 Problem Details.
_Source: Sections 5.1; 6.1.9 | Testable: Yes_

**REQ-156**: The system SHALL persist and fsync a PREPARED event outbox record before submitting to TigerBeetle for each incoming request. The record SHALL transition to COMMITTED only after TigerBeetle confirms success.
_Source: Section 6.1.11 | Testable: Yes_

**REQ-069**: The system SHALL execute templates for the same event in parallel
_Source: Section 6.1.4 — Template loading and processing (parallelization) | Testable: Yes_

### E.3 Transfer Generation Requirements

**REQ-018**: The system SHALL ensure each individual transfer has debit and credit accounts from the same ledger
_Source: Section 5.2 — Cross‑ledger transfer flags (Per‑transfer same‑ledger is mandatory) | Testable: Yes_

**REQ-019**: WHEN generating transfers, the system SHALL enforce that all amounts are greater than 0
_Source: Section 6.1.6 | Testable: Yes_

**REQ-020**: The system SHALL reject any template containing floating-point literals or functions
_Source: Section 6.1.6 | Testable: Yes_

**REQ-021**: WHEN accounts from different entities are used AND the template omits the `allows_intercompany_transfers` capability, the system SHALL reject the event
_Source: Section 6.1.8 | Testable: Yes_

**REQ-022**: WHEN accounts with different currencies are used AND the template omits the `allows_fx_transfers` capability, the system SHALL reject the event
_Source: Section 6.1.8 | Testable: Yes_

**REQ-023**: The system SHALL ensure the total business transfers per event fit within the TigerBeetle atomic batch capacity when combined with one dummy transfer (TigerBeetle linked batch capacity = 8191; business transfers per event ≤ 8190)
_Source: Section 6.1.4 — Multi‑template atomicity & isolation; Section 6.1.2 — Deduplication implementation | Testable: Yes_

**REQ-024**: WHEN leg account types don't match declared types, the system SHALL reject with error
_Source: Section 5.2 — Account type validation | Testable: Yes_

**REQ-173**: The system SHALL NOT attempt to set TigerBeetle `recorded_at`; TB cluster time SHALL supply `recorded_at`
_Source: Section 6.1.9 | Testable: Yes_

**REQ-174**: WHEN an event involves accounts with multiple distinct external custodians, the system SHALL reject the event with RFC 9457 Problem Details.
_Source: Section 13.1 — Single external custodian per event | Testable: Yes_

### E.4 Deduplication Requirements

**REQ-025**: WHEN processing an event, the system SHALL create a dummy transfer with `id = event_id` as the first transfer in the batch
_Source: Section 6.1.2 | Testable: Yes_

**REQ-026**: WHEN a dummy transfer with the same `event_id` already exists, the system SHALL reject the request with HTTP 409 (RFC 9457 Problem Details).
_Source: Section 6.1.2 | Testable: Yes_

**REQ-027**: The system SHALL submit all transfers (dummy + business) as an atomic linked batch
_Source: Section 6.1.2 | Testable: Yes_

**REQ-028**: WHEN any transfer in the batch fails, the system SHALL reject the entire batch (atomicity)
_Source: Section 6.1.2 | Testable: Yes_

### E.5 Rollup Processing Requirements

**REQ-029**: WHEN an account has `account_code.has_rollup == true`, the system SHALL generate rollup transfers
_Source: Section 6.1.10 | Testable: Yes_

**REQ-030**: WHEN the `skip_rollup_processing` directive is present, the system SHALL NOT generate rollup transfers
_Source: Section 6.1.10 | Testable: Yes_

**REQ-031**: WHEN both debit and credit accounts have the same account code, the system SHALL NOT generate rollup transfers
_Source: Section 6.1.10 | Testable: Yes_

**REQ-032**: WHEN posting directly to a rollup account WITHOUT the `manual_rollup_override` directive, the system SHALL reject the request with RFC 9457 Problem Details.
_Source: Section 6.1.8 | Testable: Yes_

**REQ-033**: WHEN `manual_rollup_override` is used, the system SHALL require audit justification in event metadata
_Source: Section 6.1.8 | Testable: Yes_

### E.6 Account Enrichment Requirements

**REQ-034**: WHEN an account ID is not found during enrichment, the system SHALL reject the request with RFC 9457 Problem Details.
_Source: Section 6.2.1 | Testable: Yes_

**REQ-035**: The system SHALL only accept internal account IDs (not external IDs)
_Source: Section 5.1 | Testable: Yes_

**REQ-036**: WHEN Ledger Service is unavailable, the system SHALL return `LEDGER_STORE_ERROR`
_Source: Section 8.2 | Testable: Yes_

### E.7 Error Handling Requirements

**REQ-037**: WHEN validation fails, the system SHALL return RFC 9457 Problem Details per `problem-details.schema.json`.
_Source: Section 8.1 | Testable: Yes_

**REQ-038**: WHEN CEL evaluation fails, the system SHALL return `CEL_EVALUATION_ERROR` with details
_Source: Section 8.2 | Testable: Yes_

**REQ-039**: WHEN accounts would go negative or hit limits, the system SHALL reject the request with RFC 9457 Problem Details.
_Source: Section 8.2 | Testable: Yes_

**REQ-040**: The batch for TigerBeetle SHALL be formed as a single atomic chain including the dummy transfer at index 0 and all business transfers at subsequent indices, using TigerBeetle’s atomic‑linking rules
_Source: Sections 6.1.1–6.1.2; 6.2.1 | Testable: Yes_

**REQ-081**: WHEN Event Store publish fails, the system SHALL retain the event in the local durable outbox in COMMITTED state and retry in background until publish succeeds
_Source: Section 6.1.11 | Testable: Yes_

**REQ-082**: WHEN Event Store publish repeatedly fails, the system SHALL log the failure, retain the durable COMMITTED outbox record, continue background retries, and SHALL NOT panic
_Source: Section 6.1.11 | Testable: Yes_

### E.8 System Initialization Requirements

**REQ-043**: WHEN deduplication ledger is not found at startup, the system SHALL NOT start
_Source: Section 7.1.1 | Testable: Yes_

**REQ-044**: WHEN deduplication accounts are not found at startup, the system SHALL NOT start
_Source: Section 7.1.1 | Testable: Yes_

**REQ-045**: WHEN rollup accounts are missing for configured account codes, the system SHALL NOT start
_Source: Section 7.1.1 | Testable: Yes_

**REQ-046**: WHEN zero templates load successfully, the system SHALL refuse to start
_Source: Section 7.1.2 | Testable: Yes_

**REQ-047**: WHEN Ledger Service is unavailable at startup, the system SHALL delay readiness until validation completes
_Source: Section 5.2 | Testable: Yes_

**REQ-090**: WHILE startup validations are incomplete, the system SHALL report Not Ready
_Source: Sections 7.1.2; 11.3 | Testable: Yes_

**REQ-171**: WHILE Event Store is unavailable and TigerBeetle is reachable and dedup/rollup infrastructure is verified and LIVE templates are loaded, the system SHALL report Ready
_Source: Section 11.3 (Not required for readiness) | Testable: Yes_

**REQ-071**: WHILE dedup/rollup infrastructure is not verified OR LIVE templates are not validated OR TigerBeetle is unreachable the system SHALL report Not Ready
_Source: Section 11.3 — Health Checks (Ready when / Critical dependencies) | Testable: Yes_

**REQ-167**: On restart, the service SHALL replay PREPARED records first; if it finds PREPARED records for which TigerBeetle already committed (detected via duplicate dummy on replay), it SHALL mark them COMMITTED. After recovery, the outbox publisher SHALL resume publication of COMMITTED records.
_Source: Section 6.1.11 | Testable: Yes_

### E.9 Performance Requirements

**REQ-048**: The system SHALL support horizontal scaling to handle high event volumes
_Source: Section 9.1 | Testable: Yes_

**REQ-049**: The system SHALL timeout TigerBeetle submissions after 10 seconds
_Source: Section 6.2.1 | Testable: Yes_

**REQ-050**: The system SHALL consider Event Store publish failed after 5 seconds timeout
_Source: Section 6.1.11 | Testable: Yes_

**REQ-172**: The system SHALL maintain a persistent TigerBeetle client connection and SHALL auto-reconnect on disconnects
_Source: Section 6.4.2 | Testable: Yes_

### E.10 Data Integrity Requirements

**REQ-051**: The system SHALL store only successfully processed events in the Event Store
_Source: Section 5.7 | Testable: Yes_

**REQ-052**: The system SHALL maintain immutability of all processed events and transfers
_Source: Section 3 | Testable: Yes_

**REQ-053**: All transfers for an event SHALL succeed atomically or none SHALL succeed
_Source: Section 3 | Testable: Yes_

**REQ-054**: The system SHALL use only integer arithmetic for all financial calculations
_Source: Section 6.1.6 | Testable: Yes_

**REQ-055**: The system SHALL apply banker's rounding for integer division
_Source: Section 6.1.6 | Testable: Yes_

**REQ-056**: All timestamps in the system SHALL be Unix time in milliseconds since the Unix epoch
_Source: Section 3 | Testable: Yes_

**REQ-114**: The system SHALL use Unix time in milliseconds for all timestamps persisted or emitted
_Source: Sections 3; 6.1.9 | Testable: Yes_

**REQ-157**: The system SHALL publish COMMITTED event outbox records to the Event Store only from background outbox-publisher workers and SHALL continuously drain the committed backlog until it is empty.
_Source: Section 6.1.11 | Testable: Yes_

**REQ-163**: The event outbox record state transitions SHALL be PREPARED → COMMITTED → PUBLISHED. PREPARED SHALL be durably fsynced before TigerBeetle submission; COMMITTED and PUBLISHED SHALL be persisted for recovery and eventual publication.
_Source: Section 6.1.11 | Testable: Yes_

**REQ-164**: WHEN TigerBeetle commits and the event outbox is COMMITTED, the system SHALL guarantee eventual publication to the Event Store by continuous retry until success
_Source: Section 6.1.11 | Testable: Yes_

**REQ-159**: Deduplication scope SHALL be global: a given `event_id` (UUIDv7) SHALL uniquely identify an event across all sites
_Source: Section 5.8 | Testable: Yes_

**REQ-072**: The system SHALL NOT depend on a lifeboat message queue for correctness; durable recovery and eventual publication SHALL rely on the local outbox and background outbox publisher
_Source: Section 6.1.11 | Testable: Yes_

**REQ-073**: The core SHALL be deterministic given inputs; side effects SHALL be confined to adapters
_Source: Sections 3 (Design Principles) and 4.2 (Architecture Overview) | Testable: Yes_

### E.11 Business Rule Requirements

**REQ-057**: WHEN a site rule evaluates to true, the system SHALL reject the event with RFC 9457 Problem Details.
_Source: Section 6.1.8 | Testable: Yes_

**REQ-058**: WHEN a template validation predicate evaluates to false, the system SHALL reject with RFC 9457 Problem Details.
_Source: Section 6.1.8 | Testable: Yes_

**REQ-059**: WHEN the `seed_mode` directive is present, the system SHALL require `ledger_timestamp` and use it as `occurred_at`
_Source: Section 6.1.9 | Testable: Yes_

**REQ-060**: The system SHALL evaluate variables sequentially in the order defined
_Source: Section 6.1.7 | Testable: Yes_

### E.12 API Requirements

**REQ-061**: WHEN processing `POST /events`, the system SHALL follow the documented request pipeline in Section 6.2.1.
_Source: Section 6.2.1 | Testable: Yes_

**REQ-062**: WHEN processing `POST /events:validate`, the system SHALL run full pipeline WITHOUT TB submission
_Source: Section 6.2.2 | Testable: Yes_

**REQ-063**: The system SHALL limit event validation requests to 1 event per request
_Source: Section 6.2.2 | Testable: Yes_

**REQ-165**: The API success code for `POST /events` SHALL be 200 OK; 201 and 202 SHALL NOT be used
_Source: Section 6.2.1 | Testable: Yes_

**REQ-166**: The response body for `POST /events` SHALL include `status="COMMITTED"` when the request is acknowledged.
_Source: Sections 6.1.11; 6.2.1 | Testable: Yes_

**REQ-158**: WHEN TigerBeetle commits but Event Store publish is not yet successful, the system SHALL return HTTP 200 with `status="COMMITTED"`.
_Source: Section 6.1.11; 6.2.1 | Testable: Yes_

**REQ-174**: WHEN calling Ledger Service, the system SHALL authenticate with an API key
_Source: Section 6.4.1 | Testable: Yes_

### E.13 CEL Expression Requirements

**REQ-065**: The system SHALL provide a built-in `always_true` boolean variable for unconditional legs
_Source: Section 5.2 — Condition variable (built‑in `always_true`) | Testable: Yes_

**REQ-066**: Leg fields SHALL only reference variables or whitelisted selectors (no expressions)
_Source: Section 5.2 — Variable‑only legs with whitelisted selectors (Allowed selectors) | Testable: Yes_

**REQ-067**: The system SHALL enforce CEL sandboxing with no filesystem or network access
_Source: Section 12.1 | Testable: Yes_

**REQ-068**: The system SHALL apply resource limits (CPU/time, program size) to CEL evaluation
_Source: Section 12.1 | Testable: Yes_

**REQ-074**: The system SHALL protect against integer overflow during evaluation and SHALL sanitize external data before use
_Source: Section 12.2 — Data Validation | Testable: Yes_

**REQ-160**: Template effective dates SHALL be at least 48 hours after approval to avoid in-flight activation ambiguity. The system SHALL provide an administrative reload endpoint, and operators SHALL invoke it after template state transitions become effective.
_Source: Section 5.2 — Loading logic | Testable: Yes_

**REQ-161**: The system SHALL mark `/health` as unhealthy when Event Store is unreachable or when spool backlog exceeds a configurable threshold
_Source: Section 11.3 | Testable: Yes_

**REQ-162**: The system SHALL reject any user‑defined `extra_metadata` key whose name begins with `__`
_Source: Section 5.1 | Testable: Yes_

### E.13.2 Template Validation Requirements

**REQ-175**: WHEN processing template validation with an event*request, the system SHALL generate transfers in expected-transfer format including account names, codes, and types
\_Source: Section 6.2.6 | Testable: Yes*

**REQ-176**: WHEN expected*transfers are NOT provided AND event_request IS provided, the system SHALL auto-generate expected_transfers and return them in the response
\_Source: Section 6.2.6 | Testable: Yes*

**REQ-177**: WHEN comparing expected vs generated transfers, the system SHALL ignore transfer IDs and compare only business-relevant fields
_Source: Section 6.2.6 | Testable: Yes_

**REQ-178**: The system SHALL return generated transfers in the expected-transfer schema format with human-readable account information
_Source: Section 6.2.6 | Testable: Yes_
**REQ-179**: WHEN template validation fails, the system SHALL provide structured error details including field paths, error categories, and actionable suggestions
_Source: Section 6.2.6 | Testable: Yes_
**REQ-180**: The system SHALL categorize validation errors as JSON*STRUCTURE, CEL_EXPRESSION, BUSINESS_LOGIC, REFERENCE, TYPE, or RANGE
\_Source: Section 6.2.6 | Testable: Yes*
**REQ-181**: The system SHALL provide both simple string errors (validation*errors) and detailed structured errors (validation_details) for compatibility
\_Source: Section 6.2.6 | Testable: Yes*

### E.13.3 Template Validation Test Requirements

**TEST-001**: GIVEN a template validation request with event*request but no expected_transfers, WHEN processed, THEN the system SHALL auto-generate expected_transfers and set auto_generated=true
\_Validates: REQ-176 | Type: Unit Test*

**TEST-002**: GIVEN a template validation request with both event*request and expected_transfers, WHEN processed, THEN the system SHALL compare them and report mismatches in comparison_errors
\_Validates: REQ-177 | Type: Unit Test*

**TEST-003**: GIVEN generated transfers from template validation, WHEN formatted, THEN each transfer SHALL include account names, codes, types, and leg*number
\_Validates: REQ-175, REQ-178 | Type: Unit Test*

**TEST-004**: GIVEN a template with invalid CEL expressions, WHEN validated, THEN the system SHALL return status=INVALID with detailed validation*errors
\_Validates: REQ-066 | Type: Unit Test*

**TEST-005**: GIVEN auto-generated expected*transfers, WHEN used in subsequent template validation, THEN they SHALL match exactly with regenerated transfers
\_Validates: REQ-176 | Type: Integration Test*
**TEST-006**: GIVEN a template with invalid CEL expressions, WHEN validated, THEN the system SHALL return structured error details with category=CEL*EXPRESSION and field_path
\_Validates: REQ-179, REQ-180 | Type: Unit Test*
**TEST-007**: GIVEN a template with duplicate leg numbers, WHEN validated, THEN the system SHALL return structured error details with category=BUSINESS*LOGIC and actionable suggestions
\_Validates: REQ-179, REQ-180 | Type: Unit Test*
**TEST-008**: GIVEN template validation errors, WHEN processed, THEN the response SHALL contain both validation*errors array and validation_details array
\_Validates: REQ-181 | Type: Unit Test*

### E.14 Requirements Summary

Total Requirements: 95

- Identity & Validation: 8
- Template Processing: 12
- Transfer Generation: 8
- Deduplication: 4
- Rollup Processing: 5
- Account Enrichment: 3
- Error Handling: 6
- System Initialization: 9
- Performance: 4
- Data Integrity: 11
- Business Rules: 4
- API: 7
- CEL Expression: 8

All requirements are testable and traceable to source sections in this specification.

---

End of Functional Specification v1.3.1
