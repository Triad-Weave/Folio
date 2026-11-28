# Agent guide for Folio

## Purpose and current status

Folio is an AI-powered invoice processing pipeline. It extracts structured data
from English PDF, PNG and JPEG invoices, validates it with deterministic rules,
uses a stronger model for one repair attempt, and routes uncertain results to
human review. Automated evaluations are intended to gate changes to AI behavior.

This repository currently contains design documentation and directory scaffolding
(`.gitkeep` files), not a working application. There are no package manifests,
lockfiles, application sources, tests, deployment manifests or CI workflows yet.
Treat the stack and component responsibilities below as intended architecture,
not implemented behavior. Recheck the checkout as implementation progresses.

## Read first

1. [README.md](README.md): product summary.
2. [Architecture](docs/architecture/README.md): consolidated design, requirement
   traceability, reliability invariants, evaluation policy and open decisions.
3. [Tickets](docs/tickets/tickets.md): FOL/PLAT work items, dependencies and
   acceptance criteria. Read the relevant ticket before implementing a feature.
4. [AI-assisted development log](docs/ai-assisted-development-log.md): currently
   a heading-only log; record substantive AI-assisted work as the project evolves.
5. `docs/adr/`: reserved for decision records; check for new decisions before
   changing architecture.

The architecture distinguishes Required, Diagram-backed, Proposed, Not specified
and Implemented evidence. Preserve those distinctions in code reviews and docs.
The source SRS and platform PDFs are referenced there but are not tracked here;
do not assume their machine-local paths are available.

Use the consolidated architecture's alignment decisions when interpreting older
ticket text. For example, the enumerated requirements contain 17 Must and 5
Should items despite an older summary saying 18 Must. Some tickets refer to
future `docs/design/` files; the current architecture lives in
`docs/architecture/README.md`. Flag unresolved conflicts rather than silently
inventing a requirement or claiming a proposal has been approved.

## Repository map and intended stack

| Path | Intended responsibility |
| --- | --- |
| `apps/web/` | React + Vite + TypeScript UI; TanStack Query server state; invoice upload, status and review |
| `apps/api/` | NestJS REST boundary for intake, queries, review and audit orchestration |
| `apps/worker/` | Asynchronous invoice processing and durable queue handling |
| `apps/eval/` | TypeScript evaluation runner and quality-gate reporting |
| `packages/contracts/` | Shared types and boundary contracts |
| `packages/schemas/` | Zod extraction schemas |
| `packages/validation/` | Deterministic invoice business rules |
| `packages/ai/` | Gateway client and versioned prompts |
| `packages/database/` | Prisma schema, migrations and database access |
| `datasets/generated/` | Generated invoices with known ground truth |
| `datasets/public/` | Public samples with checked labels |
| `deployment/helm/` | Folio configuration consuming the shared Helm library |
| `.github/workflows/` | Future CI integration |
| `docs/` | Architecture, tickets, design, ADRs, evaluation evidence and AI log |

Tailwind CSS/shadcn UI and React-PDF are diagram-backed frontend choices.
Package manager, runtime versions, build orchestration and outbox relay placement
are undecided. Do not assume npm, pnpm, Yarn, Nx or Turborepo is configured.
Avoid imports from another application's internals; reusable logic belongs in
shared packages. Browser bundles must not include Prisma or service credentials.

## Processing rules to preserve

- Public states are `RECEIVED`, `EXTRACTING`, `NEEDS_REVIEW`, `APPROVED` and
  `FAILED`. Keep business review distinct from operational failure and the DLQ.
- Intake stores the original, then commits invoice and outbox records in one
  database transaction. The proposed `POST /invoices` returns `202 Accepted`
  after the commit. Extraction runs asynchronously, outside the HTTP request.
- Identical-file uploads return the existing invoice without new processing.
  Enforce deduplication durably, including concurrent submissions; an in-memory
  check is insufficient. Hash algorithm and deduplication scope need decisions.
- Tier 1 runs first. Schema or business-rule failure triggers one Tier 2 attempt.
  If that fails validation, route to review. A valid but low-confidence result
  goes directly to review; it does not trigger Tier 2.
- Validate untrusted data at runtime. Model confidence cannot override schema or
  business-rule failures. Missing confidence must not silently pass approval.
- Keep semantic retries, gateway retries and queue delivery retries separate.
  Park operational failures after five failed deliveries total, with backoff and
  a visible reason. Invalid extraction and review routing are not queue failures.
- Persist results and audit evidence before acknowledging queue messages.
  Redelivery must not produce a second committed outcome or reset Tier 2 progress.
  Follow architecture section 6 for outbox, claim recovery and stale-write guards.
  Do not hold database transactions open during LLM calls or promise exactly one
  external model call/charge.
- Human review is oldest first, shows original and extracted fields with reasons,
  and records who changed what and when. Dataset promotion is an explicit action
  separate from correction and approval.

## Platform and data boundaries

Folio consumes a shared platform: PostgreSQL, S3-compatible storage, a queue/DLQ,
an LLM gateway, observability and Kubernetes delivery. MinIO, RabbitMQ, LiteLLM,
KEDA and Argo CD are diagram-backed selections pending platform agreement.

All model calls, including evaluations, go through the shared gateway. Provider
credentials belong only in the gateway. Do not bypass budgets or allow-lists.
Platform changes belong in the platform repository; the documented workflow
requires the other two mentees' approval. Do not duplicate platform services in
Folio or treat this guide as authorization to contact reviewers.

Use generated or public sample invoices only. Real-company/confidential invoices,
handwriting, non-English invoices and accounting integrations are outside scope.
Never commit secrets or log document bytes, keys or full extracted payloads.
The browser talks to the API, not directly to the database, queue or LLM provider.
Queue messages carry document references rather than document bytes.

## Evaluation and verification

The planned labelled dataset has at least 60 invoices, including at least 40
generated invoices; the remainder are manually labelled public samples. Labelled
invoices must never be prompt examples. Preserve provenance, dataset versions and
split membership, and keep ground truth out of extraction inputs.

For prompt, schema or model-configuration changes, the required gate compares a
20-invoice subset against main. Fail if field accuracy drops **more than 2
percentage points** or mean cost per invoice rises **more than 20%**. Equality at
either boundary does not fail. Use pinned, comparable datasets/configurations;
never exclude failed invoices from scoring denominators. See architecture
section 10 for metrics and unresolved comparison details.

No install, build, lint, test, development-server or evaluation command is
currently available. Do not invent commands or report unrun checks as passing.
When scaffolding is requested, document the selected toolchain and actual commands
in the README and update this guide. Once tests exist, run checks appropriate to
the change, including validation/routing tests and integration coverage for
transactions, duplicate delivery and crash recovery when those paths change.

The hard reliability acceptance test is zero lost or duplicate invoices in a
50-invoice worker-kill run. Accuracy, cost, latency and straight-through approval
targets are documented goals, not measured achievements in this checkout.

## Working conventions

- Inspect Git status and relevant files first; preserve unrelated user changes
  and local IDE metadata. Keep work focused on the requested scope.
- Use ticket IDs and acceptance criteria where applicable. Prioritize Must work
  before optional exports, replay UI or cosmetic dashboards.
- Settle only decisions needed for the task. Record new architectural choices,
  assumptions and tradeoffs in the architecture or an ADR; do not present
  unspecified API contracts, monetary tolerances or confidence policies as facts.
- Keep architecture, requirements and implementation claims consistent. Update
  this guide when the repository gains real tooling or changes its structure.
- In handoffs, state what changed, what was verified, and any remaining decisions
  or limitations. Documentation-only changes need link/content checks, not claims
  of application tests that do not yet exist.
