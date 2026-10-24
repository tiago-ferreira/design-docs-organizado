# Design

## Context

O Order Management System (OMS) atual (`order-management-api`) é uma aplicação Node.js (>=20) + TypeScript com Express 4.21, Prisma 5.22 (MySQL), Zod 3.23, Pino 9.5 e autenticação JWT com roles `ADMIN` e `OPERATOR` (ver `proposal.md - Why` para a motivação de negócio). O método `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) já executa uma transação interativa (`prisma.$transaction`) que valida a máquina de estados (`src/modules/orders/order.status.ts`), debita ou repõe estoque em `products` e grava auditoria em `order_status_history`. Não existe nenhum mecanismo de publicação de eventos ou webhooks.

Esta mudança produz exclusivamente o pacote de Design Docs (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-001..006.md`, `docs/TRACKER.md` e `README.md`), respeitando a restrição absoluta de não modificar o código-fonte (`src/`, `prisma/`, `tests/`, `TRANSCRICAO.md`).

## Goals / Non-Goals

**Goals:**
- Estruturar cada documento na sua altura correta (PRD: Produto; RFC: Arquitetura/Proposta; ADRs: Decisões pontuais; FDD: Implementação acionável; Tracker: Matriz de rastreabilidade transversal), eliminando redundância entre eles.
- Garantir 100% de fidelidade aos fatos de `TRANSCRICAO.md` e à estrutura real da codebase (`prisma/schema.prisma`, `src/modules/orders/order.service.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/shared/logger/index.ts`, `src/routes/index.ts`, `src/app.ts`, `src/server.ts`).
- Assegurar cumprimento integral de todos os checklists de aceite do desafio (incluindo proporções de rastreabilidade no `docs/TRACKER.md`: >= 80% de cobertura dos itens, >= 70% `TRANSCRICAO` e >= 5 linhas `CODIGO`).

**Non-Goals:**
- Implementar código TypeScript, alterar o schema Prisma ou criar migrations/testes reais no repositório.
- Inventar requisitos ou tecnologias não fundamentados na reunião ou no código (ex.: filas externas, dashboards visuais, notificações por e-mail).

## Decisions

### 1. Sequência de Produção ("Bottom-Up" Arquitetural para "Top-Down" de Produto)
- **Escolha**: Produzir os documentos na ordem: (1) ADRs (`ADR-001` a `ADR-006`) → (2) `docs/RFC.md` → (3) `docs/FDD.md` → (4) `docs/PRD.md` → (5) `docs/TRACKER.md` → (6) `README.md`.
- **Racional**: Formalizar primeiro as 6 decisões fechadas nos ADRs estabelece os blocos fundamentais. O RFC consolida a visão arquitetural referenciando os ADRs e registrando as alternativas descartadas e questões em aberto. O FDD traduz a arquitetura em contratos HTTP, schemas Prisma, fluxos transacionais e ganchos de código. O PRD consolida a visão de produto/negócio sem vazar detalhes de implementação. O Tracker audita todos os documentos prontos contra `TRANSCRICAO.md` e `CODIGO`, e o README documenta o processo completo.
- **Alternativa considerada**: Gerar o PRD primeiro e descer até o FDD. Descartada porque aumenta o risco de colocar detalhes técnicos de implementação no PRD ou no RFC, violando a fronteira de altura entre os documentos.

### 2. Escopo e Recorte dos 6 ADRs (`docs/adrs/`)
- **Escolha**: Criar exatamente 6 ADRs no formato MADR cobrindo todas as 6 decisões principais exigidas:
  1. `ADR-001-padrao-outbox-no-mysql.md`: Uso de tabela `webhook_outbox` na mesma transação SQL (`prisma.$transaction`) de `OrderService.changeStatus` com snapshot do payload renderizado e filtragem na inserção, descartando chamada HTTP síncrona e Redis Streams/Cluster (`[09:04] Bruno`, `[09:06] Diego`, `[09:34] Bruno`, `[09:52] Larissa`).
  2. `ADR-002-worker-em-processo-separado-com-polling.md`: Entry-point dedicado `src/worker.ts` (`npm run worker`) com instância própria de `PrismaClient` lendo eventos pendentes em loop de polling a cada 2 segundos, descartando execução in-process na API e triggers no MySQL (`[09:09] Diego`, `[09:11] Diego`, `[09:11] Larissa`, `[09:30] Bruno`).
  3. `ADR-003-politica-de-retry-com-backoff-e-dlq.md`: Até 5 tentativas com progressão de backoff exponencial (`1m`, `5m`, `30m`, `2h`, `12h`, totalizando ~15h), timeout HTTP de 10s por tentativa, e movimentação para tabela dedicada `webhook_dead_letter` com replay manual via `POST /api/v1/admin/webhooks/dead-letter/:id/replay` exigindo role `ADMIN`, descartando 3 tentativas, retry indefinido e flag simples na outbox (`[09:15]–[09:18] Diego`, `[09:36] Sofia`, `[09:42] Diego`).
  4. `ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md`: Assinatura HMAC-SHA256 sobre o corpo JSON enviada em `X-Signature` com `X-Timestamp` e `X-Webhook-Id`, secret exclusiva por endpoint, obrigatoriedade de HTTPS (TLS) e rotação de secret via API com grace period de 24 horas onde a secret anterior permanece válida (`[09:20]–[09:23] Sofia`).
  5. `ADR-005-garantia-at-least-once-com-x-event-id.md`: Semântica de entrega *at-least-once* com UUID único por evento no header `X-Event-Id` para deduplicação idempotente pelo cliente e ordenação implícita por `created_at` enquanto single-worker, descartando entrega *exactly-once* distribuída e ordenação global multi-worker (`[09:12]–[09:13] Diego`, `[09:24]–[09:26] Diego`).
  6. `ADR-006-reuso-dos-padroes-existentes-do-projeto.md`: Alinhamento estrito à arquitetura modular em `src/modules/webhooks/`, reutilizando `AppError` (`src/shared/errors/app-error.ts`) com códigos prefixados por `WEBHOOK_`, `errorMiddleware` (`src/middlewares/error.middleware.ts`), validação `validate` com Zod (`src/middlewares/validate.middleware.ts`), `authenticate` e `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`), `logger` Pino com redaction (`src/shared/logger/index.ts`), paginação `paginated` (`src/shared/http/response.ts`) e IDs UUID `@db.Char(36)` no Prisma (`prisma/schema.prisma`) (`[09:27]–[09:30] Bruno/Larissa`, `[09:51] Larissa`).

### 3. Desenho de Contratos e Integração no FDD (`docs/FDD.md`)
- **Escolha**: Especificar no FDD:
  - **Modelagem de Dados Prisma**: Tabelas `webhook_endpoints` (com suporte a `secret`, `previousSecret` e `previousSecretExpiresAt` para o grace period de 24h, além de `events` JSON/array de `OrderStatus`), `webhook_outbox` (com snapshot do payload JSON, `status`, `attempts`, `nextAttemptAt`), `webhook_deliveries` (histórico de tentativas com status code, response body, durationMs) e `webhook_dead_letter` (eventos esgotados após 5 falhas com motivo e suporte a replay).
  - **Contratos HTTP Públicos (7 endpoints sob `/api/v1`)**:
    1. `POST /api/v1/webhooks` (cadastro de endpoint com `customerId`, `url` HTTPS, `events`; retorna `secret` gerada)
    2. `GET /api/v1/webhooks` (listagem paginada por `customerId`)
    3. `PATCH /api/v1/webhooks/:id` (atualização de `url`, `events`, `active`)
    4. `DELETE /api/v1/webhooks/:id` (remoção de webhook)
    5. `POST /api/v1/webhooks/:id/rotate-secret` (rotação de secret com grace period de 24h)
    6. `GET /api/v1/webhooks/:id/deliveries` (histórico das últimas entregas paginado/limitado a 100 com payload, status, response e latência)
    7. `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (reprocessamento manual exigindo `requireRole('ADMIN')` e log de auditoria)
  - **Matriz de Erros `WEBHOOK_*`**: `WEBHOOK_NOT_FOUND` (404), `WEBHOOK_INVALID_URL` (400/422 - ex. HTTP não-TLS), `WEBHOOK_PAYLOAD_TOO_LARGE` (422 - > 64KB), `WEBHOOK_SECRET_REQUIRED` (400), `WEBHOOK_CUSTOMER_NOT_FOUND` (404), `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404), `WEBHOOK_REPLAY_CONFLICT` (409).
  - **Ganchos de Integração com o Código Existente**: Mapeamento detalhado de `src/modules/orders/order.service.ts` (chamada a `publishWebhookEvent(tx, refreshedOrder, from, to)` dentro de `changeStatus`), `prisma/schema.prisma`, `src/shared/errors/app-error.ts` & `src/shared/errors/http-errors.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/shared/logger/index.ts`, `src/routes/index.ts` e `src/app.ts`.

### 4. Estratégia de Rastreabilidade no `docs/TRACKER.md`
- **Escolha**: Construir um sistema de IDs padronizados por documento (`PRD-*`, `RFC-*`, `FDD-*`, `ADR-001` a `ADR-006`) cobrindo mais de 50 itens granulares, garantindo que > 75% apontem para `TRANSCRICAO` com `[hh:mm] Nome` exato e pelo menos 10 itens apontem para `CODIGO` com caminhos reais verificados no repositório.

## Risks / Trade-offs

- **[Risco] Sobreposição de conteúdo entre RFC e FDD** → **Mitigação**: Manter o RFC estritamente entre 2 e 4 páginas focado em arquitetura, alternativas descartadas e questões em aberto, delegando schemas Prisma, payloads JSON completos e assinaturas de métodos exclusivamente ao FDD.
- **[Risco] Alucinação de timestamps ou nomes de arquivos no Tracker** → **Mitigação**: Conferir cada linha do `docs/TRACKER.md` contra as linhas reais de `TRANSCRICAO.md` (`[09:00]` a `[09:53]`) e a árvore real de `src/` e `prisma/`.
- **[Risco] Violação acidental de imutabilidade do código** → **Mitigação**: Restringir escritas na fase de apply exclusivamente a `docs/**` e `README.md`, validando ao final com `git status` / `git diff` que `src/`, `prisma/`, `tests/` e `TRANSCRICAO.md` permanecem intactos.
