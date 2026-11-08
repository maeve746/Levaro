# LEVARO Engineering Rules

This file governs all work in this repository. More specific `AGENTS.md` files may add constraints for a subdirectory, but may not weaken these rules.

## Product Direction

LEVARO is a production-oriented, AI-native sales CRM. The intended stack is Next.js, FastAPI, Neon PostgreSQL, Auth.js, SQLAlchemy, Alembic, Redis, Celery, and OpenAI.

Work in one explicitly named phase at a time. Do not begin a later phase until the current phase has acceptance criteria, tests appropriate to its risk, and an updated `BUILD_PROGRESS.md`.

## Non-Negotiable Product Rules

- Do not add fake authentication, mock production APIs, seeded leads, demo conversations, hardcoded AI responses, or simulated synchronization.
- Product data must originate from an authenticated real integration or PostgreSQL. Tests may use isolated fixtures and test doubles that cannot be enabled in production.
- Never display an integration as connected until the provider has verified the credentials and the result has been persisted.
- Encrypt OAuth access and refresh tokens at rest. Never log tokens, authorization codes, passwords, session secrets, API keys, message bodies, or unnecessary personal data.
- Enforce workspace isolation in every database query and mutation. Client-supplied workspace identifiers are never sufficient authorization.
- AI-generated or user-composed outbound messages require an explicit, recorded human approval before sending. Drafting and sending must be separate operations.
- Use Alembic migrations for every schema change. Do not mutate production schemas with startup hooks or ad hoc SQL.
- Use real error handling, observable job states, idempotency, timeouts, and retry policies. Do not report success before the underlying operation succeeds.
- Report unavailable credentials, provider limitations, and blocked integrations honestly.
- Do not deploy, provision infrastructure, enable paid services, or incur usage charges without explicit authorization.
- Preserve existing working code and unrelated user changes. Inspect before editing.

## Architecture Boundaries

- `web` (Next.js) owns rendering, browser interactions, and Auth.js session initiation. It must not hold provider secrets or access the database directly from the browser.
- `api` (FastAPI) owns business rules, workspace authorization, integration verification, AI orchestration, approvals, and persistence.
- PostgreSQL is the source of truth for durable application state.
- Redis is transient infrastructure for queues, locks, rate limits, and short-lived cache data; it is not a system of record.
- Celery workers execute asynchronous operations. Jobs must be idempotent and scoped to a workspace.
- External providers and OpenAI are accessed only through server-side adapters with explicit timeouts and sanitized error handling.

See `ARCHITECTURE.md` for the target system design. It is a design baseline, not evidence that a component has been implemented.

## Security and Privacy

- Default to deny. Authenticate the user, resolve membership server-side, then authorize the action and resource.
- Scope reads and writes by `workspace_id`, including background jobs and webhook processing. Prefer repository/service interfaces that require workspace context.
- Use secure, HTTP-only, same-site cookies in production and validate CSRF protections for state-changing browser requests.
- Validate OAuth `state`; use PKCE where the provider supports it; use exact redirect URIs; verify webhook signatures before parsing or enqueueing events.
- Store only encrypted provider tokens. Keep encryption keys outside the database and support key rotation by storing a key version with ciphertext.
- Use least-privilege OAuth scopes and database roles. Never expose server-only environment variables through `NEXT_PUBLIC_*`.
- Treat prompt input and provider content as untrusted. Tools/actions invoked by AI must pass the same authorization and approval checks as direct API calls.
- Record security-relevant events, approvals, sends, integration changes, and privileged operations in an append-only audit trail without secret material.

## Data and Migration Practices

- Use UUIDs for externally visible identifiers unless a documented decision says otherwise.
- Store timestamps as timezone-aware UTC values.
- Add foreign keys, uniqueness constraints, and indexes deliberately; application validation does not replace database integrity.
- Migrations must be reversible when practical and safe for rolling deployment. Document irreversible or locking operations.
- Do not create production seed scripts containing customer or fabricated CRM data. Minimal lookup/reference data must be deterministic and documented.

## API and Background Jobs

- Define typed request and response schemas. Reject unknown or invalid input at the boundary.
- Return stable error codes and safe messages; keep provider/debug details in protected structured logs.
- Use idempotency keys for sends, imports, synchronization, webhook ingestion, and retryable mutations.
- Persist asynchronous job status (`pending`, `running`, `succeeded`, `failed`, or `cancelled`) rather than simulating completion in the UI.
- Bound retries with exponential backoff and jitter. Do not retry permanent authentication or validation failures.
- Prevent duplicate sends with a database-enforced approval/send state transition and provider idempotency where available.

## AI and Messaging

- Persist model name, prompt/template version, relevant input references, output, usage metadata, and failure state when policy permits.
- Do not claim that AI output is factual or sent. Clearly label drafts and surface generation failures.
- A draft may be regenerated without approval; any content change after approval invalidates that approval.
- Sending requires a fresh authorization check and an immutable approval record tied to the exact message revision.
- AI must never directly call a send adapter. Only the approved-send service may enqueue delivery.

## Quality Gates

Every implemented phase must include, as applicable:

- unit tests for business rules and authorization;
- integration tests against isolated PostgreSQL and Redis services;
- migration upgrade/downgrade checks;
- API contract and validation tests;
- workspace-isolation and cross-tenant denial tests;
- approval and duplicate-send prevention tests;
- linting, formatting, type checking, and dependency vulnerability review;
- documented manual verification for real provider integrations.

Tests may mock external network boundaries, but mocks must be confined to test code and may not drive a production or demo mode.

## Change Workflow

Before changing code:

1. Read `ARCHITECTURE.md` and `BUILD_PROGRESS.md`.
2. Inspect the relevant code, migrations, tests, and current working tree.
3. State the phase and acceptance criteria.
4. Identify required credentials or external access without committing secrets.

Before reporting completion:

1. Run the relevant quality gates.
2. Verify migrations and failure paths, not only the happy path.
3. Update `BUILD_PROGRESS.md` with evidence, remaining risks, and blockers.
4. Distinguish implemented, tested, manually verified, and blocked work.

## Documentation Truthfulness

- Do not mark a feature complete because a UI exists.
- Do not mark an integration connected based only on stored credentials.
- Do not call an endpoint production-ready without authentication, authorization, validation, persistence, error handling, and tests.
- Use exact dates in progress updates and identify the verification environment.

