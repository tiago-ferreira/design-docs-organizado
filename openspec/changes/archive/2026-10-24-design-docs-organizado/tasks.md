# Tasks

## 1. Architectural Decision Records (ADRs)

- [x] 1.1 Create `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` in MADR format (Status, Contexto, Decisão, Alternativas Consideradas, Consequências) covering transactional insertion in `webhook_outbox` with rendered snapshot payload and insertion-time status filtering, and verify all 5 MADR sections and explicit trade-offs against synchronous HTTP and Redis Streams are present.
- [x] 1.2 Create `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` in MADR format covering dedicated worker process (`src/worker.ts`) with its own `PrismaClient` polling every 2 seconds, and verify all 5 MADR sections and trade-offs against in-process execution and MySQL triggers are present.
- [x] 1.3 Create `docs/adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md` in MADR format covering 5 retry attempts (`1m`, `5m`, `30m`, `2h`, `12h`), 10s HTTP timeout, separate `webhook_dead_letter` table, and manual replay via `ADMIN` endpoint, and verify all 5 MADR sections and trade-offs against 3 retries, infinite retries, and in-table status flag are present.
- [x] 1.4 Create `docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md` in MADR format covering HMAC-SHA256 payload signing (`X-Signature`, `X-Timestamp`, `X-Webhook-Id`), per-endpoint secret, mandatory HTTPS, and 24-hour grace period secret rotation, and verify all 5 MADR sections and trade-offs against global secret and instant invalidation are present.
- [x] 1.5 Create `docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md` in MADR format covering at-least-once delivery semantics, client-side deduplication via `X-Event-Id` UUID, and implicit per-order ordering under single-worker execution, and verify all 5 MADR sections and trade-offs against distributed exactly-once and multi-worker partitioning are present.
- [x] 1.6 Create `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md` in MADR format referencing real codebase files (`src/modules/orders/order.service.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `prisma/schema.prisma`) and `WEBHOOK_*` error codes, and verify all referenced file paths exist in the repository.

## 2. Request for Comments (RFC)

- [x] 2.1 Write `docs/RFC.md` (2-4 pages) with metadata (author, status, date, and reviewers Larissa, Marcos, Bruno, Diego, Sofia), executive summary (TL;DR), context/problem, high-level technical proposal, discarded alternatives with trade-offs (>= 2), open/deferred questions (>= 2), impact/risks, and links to `ADR-001` through `ADR-006`, and verify all mandatory RFC acceptance criteria are met without duplicating FDD implementation details.

## 3. Feature Design Document (FDD)

- [x] 3.1 Write the foundational, data model, and flow sections of `docs/FDD.md` covering technical context, technical objectives, scope/exclusions, Prisma data models (`webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`), and detailed end-to-end flows (transactional outbox snapshot creation, 2s worker polling loop, 5-step exponential backoff retry, DLQ persistence, and manual ADMIN replay), and verify alignment with `TRANSCRICAO.md` and `prisma/schema.prisma`.
- [x] 3.2 Write the HTTP contracts, outbound webhook payload/headers, error matrix, resilience, observability, and codebase integration sections of `docs/FDD.md` covering >= 6 HTTP endpoints (including request/response JSON examples and status codes), outbound headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`), `WEBHOOK_*` error matrix, 10s timeout / 64KB limit, Pino logs + metrics + tracing, and "Integração com o sistema existente" referencing >= 6 real repository file paths, and verify every cited file path exists on disk.

## 4. Product Requirements Document (PRD)

- [x] 4.1 Write `docs/PRD.md` containing all 12 mandatory product sections (summary/context, problem/motivation, target audience/use cases, objectives/quantitative success metrics including < 10s latency and retention of Atlas Comercial, MaxDistribuição, and Nova Cargo, in-scope and >= 2 out-of-scope items, >= 8 functional requirements, non-functional requirements, key decisions/trade-offs, dependencies, >= 2 risks with probability/impact/mitigation, acceptance criteria, and testing/validation strategy), and verify all PRD acceptance criteria are satisfied.

## 5. Traceability Tracker & Process README

- [x] 5.1 Populate `docs/TRACKER.md` using the mandatory Markdown table format (`ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização`), mapping >= 80% of identifiable items across `PRD.md`, `RFC.md`, `FDD.md`, and `ADR-001..006.md`, and verify that >= 70% of rows use `Fonte = TRANSCRICAO` with valid `[hh:mm] Nome` timestamps and >= 5 rows use `Fonte = CODIGO` with existing repository file paths.
- [x] 5.2 Replace `README.md` with the complete production process documentation (Sobre o desafio, Ferramentas de IA utilizadas, Workflow adotado, >= 2 Prompts customizados in code blocks, >= 2 Iterações e ajustes concretos, and Como navegar a entrega), and verify all 6 required sections are complete.
- [x] 5.3 Run a final verification check across the repository (`git status`, file existence checks, and acceptance criteria audit) to confirm all deliverables exist in Markdown, no nonexistent code paths are cited, and `src/`, `prisma/`, `tests/`, and `TRANSCRICAO.md` remain completely unmodified.
