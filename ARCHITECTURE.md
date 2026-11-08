# LEVARO Architecture

Status: target architecture for a greenfield repository. No component described here is implemented or connected as of 2026-11-08.

## Goals

LEVARO is a multi-tenant, AI-native sales CRM that centralizes durable customer data, verified provider integrations, AI-assisted drafting, and human-approved outbound communication.

The architecture prioritizes:

- strict workspace isolation;
- verifiable integration state;
- human control over outbound actions;
- auditable, idempotent asynchronous work;
- replaceable provider adapters;
- production migrations, tests, and operational visibility.

## System Context

```text
Browser
  |
  | HTTPS / secure session
  v
Next.js web application
  |
  | authenticated server-to-server API calls
  v
FastAPI application ------------------> OpenAI API
  |          |                          (generation only)
  |          +------------------------> CRM / email providers
  |                                     (verified adapters)
  v
Neon PostgreSQL <---- Celery workers <---- Redis queue
  ^                         |
  |                         +-----------> provider APIs
  |
  +---- durable state, approvals, audit events, job results
```

PostgreSQL is authoritative. Redis and Celery support asynchronous execution but do not define product truth. The UI renders persisted states returned by the API and never invents success or connectivity.

## Deployable Components

### Next.js web

- Renders authenticated application routes and public authentication screens.
- Uses Auth.js for login/session lifecycle.
- Sends the API a verifiable server-side identity/session assertion using a mechanism chosen during the authentication phase.
- Contains no database credentials, OAuth client secrets, token-encryption keys, or OpenAI keys.
- Treats drafts, approvals, sends, sync runs, and integration state as server-owned resources.

### FastAPI API

- Resolves the authenticated user and workspace membership for every protected operation.
- Owns domain services for contacts/leads, integrations, synchronization, conversations, AI drafts, approvals, and delivery.
- Persists state through SQLAlchemy and versioned Alembic migrations.
- Exposes typed, versioned endpoints (initially `/api/v1`).
- Issues transactional outbox records for external side effects that must follow database commits.

### Celery workers

- Execute sync, webhook follow-up, AI generation, and approved delivery jobs.
- Receive identifiers, not secret-rich payloads; reload authorized state from PostgreSQL.
- Use idempotency keys, bounded retries, timeouts, and structured failure states.
- Never bypass workspace authorization or approval invariants.

### PostgreSQL (Neon)

- Stores all durable application, integration, approval, audit, and job state.
- Uses foreign keys and constraints to enforce ownership and valid transitions.
- Uses separate database credentials for migrations and runtime where supported.
- Requires TLS in hosted environments.

### Redis

- Serves as Celery broker/result transport as configured, distributed lock store, rate-limit store, and short-lived cache.
- Must not contain the only copy of approvals, messages, sync cursors, or integration status.
- Uses TLS and authentication in hosted environments.

### External providers

- Are isolated behind typed adapters.
- OAuth connections become `connected` only after token exchange plus a provider identity/health request succeeds.
- Webhooks require signature verification, replay protection, and idempotent event storage.
- Provider failures are mapped to stable internal categories while retaining sanitized diagnostics.

## Trust Boundaries and Request Flow

1. Auth.js authenticates a user through a configured real identity provider or production credential flow selected in the authentication phase.
2. FastAPI validates the server-provided authentication artifact; it does not trust browser-supplied user or workspace IDs.
3. The API loads active membership and derives the effective workspace context.
4. Domain services require that context and add `workspace_id` predicates to every tenant-owned query.
5. Mutations commit domain state and any outbox event atomically.
6. Workers claim outbox/jobs, reload current state, and re-check invariants before external effects.

The exact Auth.js-to-FastAPI assertion protocol remains an architecture decision for the authentication phase. It must avoid sharing a database session in an unsafe or undocumented way and must support revocation and key rotation.

## Initial Domain Model

This is a conceptual model; table names and fields must be finalized through migrations during their owning phase.

| Aggregate | Purpose | Required invariants |
| --- | --- | --- |
| User | Human identity | Unique normalized identity; no provider secrets |
| Workspace | Tenant boundary | All business data belongs to exactly one workspace |
| Membership | User-to-workspace role | Unique active membership per user/workspace |
| Integration | Provider connection metadata | Status reflects verified provider state only |
| OAuth credential | Encrypted provider tokens | Ciphertext, key version, expiry, scopes; never plaintext |
| Contact/lead | CRM person/account data | Workspace-scoped; provider IDs unique per connection |
| Conversation/message | Inbound/outbound communication | Workspace-scoped; immutable provider IDs and direction |
| Draft revision | AI- or human-authored candidate | Immutable content revision; never itself evidence of send |
| Approval | Human authorization | Approver, timestamp, exact revision, policy context |
| Delivery | External send attempt/result | One idempotency key; approval required before enqueue |
| Sync run/cursor | Provider synchronization state | Durable checkpoint and explicit terminal status |
| Webhook event | Deduplicated inbound event | Provider event ID/signature verification result |
| Job/outbox event | Durable async intent | Transactional creation and observable processing state |
| Audit event | Security/product audit trail | Append-only, actor/action/resource metadata, no secrets |

Role design and permissions are intentionally deferred to the authentication/authorization phase. Until defined and tested, privileged actions must remain unavailable.

## Integration State Machine

```text
disconnected
    |
    | OAuth initiated (not connected)
    v
authorizing
    |
    | token exchange + provider identity check succeed
    v
connected ---- health/auth failure ----> action_required
    |                                      |
    | user revokes / provider confirms     | successful re-verification
    v                                      v
disconnected <------------------------- connected
```

An OAuth callback alone does not prove connectivity. Persist `connected` only after encrypted credential storage and a successful provider verification request. A failed or expired credential must not be presented as healthy.

## Draft, Approval, and Send State

```text
draft revision -> pending approval -> approved -> queued -> sending -> sent
       ^                  |              |          |          |
       |                  v              |          v          v
       +-------------- rejected          |        failed   provider result
                                          |
                              any content change
                                          v
                                    new unapproved revision
```

- Approval binds to the exact immutable draft revision and actor.
- The API authorizes the approver at approval time and reauthorizes the sender/job at enqueue and execution time.
- A database transaction and unique idempotency key prevent duplicate delivery.
- Only the delivery service can call a send adapter; AI generation has no send capability.
- Provider acceptance and provider delivery are distinct states when the provider exposes both.

## AI Boundary

OpenAI is a server-side generation dependency, not a source of authorization or durable truth.

- Prompt templates are versioned.
- Model identifiers are configured, not hardcoded as product claims.
- Inputs are minimized and filtered according to data-handling policy.
- Generated text is persisted as an explicitly labeled draft revision.
- Tool use, if introduced, is allowlisted and executes through ordinary domain authorization.
- Rate limits, timeouts, usage metadata, and provider request identifiers are recorded without exposing secrets.
- Failed generation remains a failed job; the UI must not substitute canned content.

## Token Encryption

OAuth tokens are encrypted at the application layer using an authenticated encryption scheme. The envelope must include a key version and nonce alongside ciphertext. Keys live in a secret manager or protected environment configuration, separate from PostgreSQL. Rotation decrypts with the old version and re-encrypts with the active version through a controlled job.

The concrete cryptographic library and encoding will be selected and threat-reviewed in the integrations foundation phase. Home-grown cryptography is prohibited.

## Reliability and Observability

- Structured JSON logs include request/job IDs, workspace-safe identifiers, provider, operation, latency, and sanitized outcome.
- Metrics cover API latency/errors, queue age, retries, sync lag, provider failures, AI usage, approvals, and delivery outcomes.
- Distributed traces propagate correlation IDs across web, API, workers, and provider adapters when enabled.
- Health endpoints distinguish liveness from dependency readiness and do not leak configuration.
- Reconciliation jobs compare ambiguous external results before retrying side effects.
- Backups, recovery objectives, retention, and deletion procedures must be established before production use.

## Repository Shape (Target)

```text
apps/
  web/                 Next.js application
  api/                 FastAPI application and Alembic environment
packages/
  contracts/           Generated/shared API contracts where justified
infra/                 Local development and authorized deployment definitions
docs/                  ADRs, runbooks, threat model, provider notes
tests/                  Cross-service integration and end-to-end tests
```

This layout is proposed, not created by the documentation phase. Each directory should be introduced only when its owning implementation phase begins.

## Planned Phase Order

1. Repository and engineering baseline.
2. Local infrastructure, database foundation, migrations, and CI quality gates.
3. Real authentication, workspace membership, and authorization.
4. CRM core data model and APIs.
5. Integration framework, encrypted OAuth storage, and verified connection lifecycle.
6. One explicitly selected real provider integration and synchronization.
7. Conversation ingestion and durable messaging model.
8. AI draft generation with evaluation and usage controls.
9. Human approval and approved-send delivery pipeline.
10. Production hardening, observability, recovery, privacy, and authorized deployment.

Only one phase is active at a time. Provider selection, paid usage, infrastructure provisioning, and deployment require explicit authorization.

## Open Architecture Decisions

- Monorepo tooling and package manager.
- Auth.js provider(s) and the Auth.js-to-FastAPI trust protocol.
- Workspace roles and permission matrix.
- First CRM/email provider and required OAuth scopes.
- Token encryption library and production key-management service.
- Celery broker/result-backend topology and Redis provider.
- OpenAI model, retention policy, prompt-evaluation criteria, and usage budgets.
- Hosting topology, regions, backup objectives, and compliance requirements.

Until these decisions are made, no document or UI may imply the related integration or production control is active.

