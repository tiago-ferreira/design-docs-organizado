# Spec Delta

## Purpose

Defines the normative structure, architectural coverage, and strict source-to-document traceability requirements for the Order Notification Webhooks design documentation suite (PRD, RFC, FDD, ADRs, Tracker, and process README).

## ADDED Requirements

### Requirement: PRD Completeness and Scope Boundaries
The system documentation SHALL provide `docs/PRD.md` in Markdown containing all 12 mandatory product sections, at least 8 functional requirements, at least 1 objective with a quantitative metric and target, at least 2 out-of-scope items explicitly discarded or deferred in the technical meeting, and at least 2 risks detailing probability, impact, and mitigation.

#### Scenario: PRD validates product scope and metrics
- **WHEN** `docs/PRD.md` is reviewed against the acceptance criteria
- **THEN** it includes all 12 required sections, >= 8 functional requirements (such as endpoint registration, event filtering, secret rotation, delivery history, and admin DLQ replay), quantitative latency target (< 10s), >= 2 out-of-scope items (e.g., email failure notifications, visual dashboard), and >= 2 risks with probability, impact, and mitigation

### Requirement: RFC Architectural Proposal and Trade-offs
The system documentation SHALL provide `docs/RFC.md` as a concise architectural review document (2-4 pages) containing metadata with the 5 meeting participants as reviewers, executive summary, context, technical proposal, at least 2 discarded alternatives with explicit trade-offs, at least 2 open/deferred questions, impact/risks, and links to at least 2 ADRs.

#### Scenario: RFC captures alternatives, open questions, and ADR links
- **WHEN** `docs/RFC.md` is inspected
- **THEN** it lists Larissa, Marcos, Bruno, Diego, and Sofia in metadata, documents at least 2 discarded alternatives with trade-offs (e.g., synchronous HTTP calls inside order transaction, Redis Streams/Cluster, MySQL triggers), documents at least 2 open questions (e.g., outbound rate limiting per customer, 30-day outbox archiving), and links to at least 2 ADRs in `docs/adrs/`

### Requirement: FDD Actionable Specification and Codebase Integration
The system documentation SHALL provide `docs/FDD.md` containing detailed flows (transactional outbox insertion with snapshot payload, 2s polling worker, exponential backoff retry, and DLQ), at least 4 HTTP endpoint contracts with request/response payloads and status codes, an error matrix using the `WEBHOOK_` prefix, resilience and observability (metrics, logs, tracing), and an explicit "Integração com o sistema existente" section referencing at least 4 real file paths from the repository.

#### Scenario: FDD specifies HTTP contracts, WEBHOOK_ errors, and real file hooks
- **WHEN** `docs/FDD.md` is evaluated for implementation readiness
- **THEN** it defines >= 4 HTTP endpoints with example payloads and status codes, lists error codes prefixed with `WEBHOOK_`, specifies outbound headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`), and details integration with at least 4 existing files (such as `src/modules/orders/order.service.ts`, `prisma/schema.prisma`, `src/shared/errors/app-error.ts`, `src/middlewares/auth.middleware.ts`, `src/routes/index.ts`, and `src/app.ts`)

### Requirement: ADR Architectural Coverage and MADR Format
The system documentation SHALL provide between 5 and 8 ADR files in `docs/adrs/` named `ADR-NNN-titulo-em-kebab-case.md`, each containing Status, Contexto, Decisão, Alternativas Consideradas, and Consequências (positive, negative, and explicit trade-offs), covering at least 5 of the 6 core decisions from the meeting and referencing existing codebase files/patterns in at least 1 ADR.

#### Scenario: ADR suite covers all 6 core decisions and references codebase
- **WHEN** the directory `docs/adrs/` is inspected
- **THEN** it contains 6 ADR files (`ADR-001` through `ADR-006`) covering MySQL Outbox, 2s Polling Worker in separate process, Exponential Backoff Retry (1m/5m/30m/2h/12h) with separate DLQ table, HMAC-SHA256 per-endpoint secret with 24h rotation grace period, At-Least-Once delivery with `X-Event-Id`, and Reuse of project patterns (explicitly referencing `src/` and `prisma/` files)

### Requirement: Cross-Document Traceability Tracker
The system documentation SHALL provide `docs/TRACKER.md` with a Markdown table containing columns `ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização`, mapping at least 80% of identifiable items across PRD, RFC, FDD, and ADRs, with at least 70% of rows sourced from `TRANSCRICAO` (`[hh:mm] Nome`) and at least 5 rows sourced from `CODIGO` (real repository file paths).

#### Scenario: Tracker validates anti-hallucination and source ratios
- **WHEN** `docs/TRACKER.md` is audited against the documentation suite and repository
- **THEN** >= 80% of documented requirements, decisions, and constraints are mapped, >= 70% of entries cite `TRANSCRICAO` with valid `[hh:mm] Speaker` timestamps, >= 5 entries cite `CODIGO` with existing file paths, and zero entries contradict `TRANSCRICAO.md` or reference non-existent code files

### Requirement: Process Documentation in Root README and Code Immutability
The system documentation SHALL replace `README.md` with the AI-assisted production narrative (covering challenge overview, AI tools used, workflow, >= 2 custom prompts in code blocks, >= 2 concrete iterations/adjustments, and navigation guide) while preserving `src/`, `prisma/`, `tests/`, and `TRANSCRICAO.md` completely unmodified.

#### Scenario: Root README is updated and application code remains untouched
- **WHEN** `README.md` and `git status` are checked after applying the change
- **THEN** `README.md` contains all 6 mandatory process sections including >= 2 custom prompts and >= 2 iterations, and no files under `src/`, `prisma/`, `tests/`, or `TRANSCRICAO.md` have been modified
