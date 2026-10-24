# ADR-006: Reuso Integral dos Padrões Arquiteturais e Utilitários Existentes da Codebase

- **Status:** Aceito
- **Data:** 2026-10-24
- **Decisores:** Larissa (Tech Lead), Bruno (Engenheiro Pleno - Pedidos), Diego (Engenheiro Sênior - Plataforma), Sofia (Engenheira de Segurança)
- **Rastreabilidade:** `TRANSCRICAO.md` (`[09:27] Bruno`, `[09:28] Diego`, `[09:28] Bruno`, `[09:29] Larissa`, `[09:29] Bruno`, `[09:30] Bruno`, `[09:30] Larissa`, `[09:36] Larissa`, `[09:41] Bruno`, `[09:51] Larissa`); `CODIGO` (`src/modules/orders/order.service.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `src/shared/http/response.ts`, `src/routes/index.ts`, `src/app.ts`, `prisma/schema.prisma`)

---

## Contexto

A codebase do Order Management System (`order-management-api`) já possui convenções maduras e consistentes para estruturação de domínios, tratamento centralizado de erros, validação de entrada, autenticação/autorização, logging estruturado e modelagem relacional (`[09:27] Bruno`, `[09:28] Bruno`, `[09:29] Bruno`).

Ao projetar o novo Sistema de Webhooks, a equipe precisava decidir se introduziria novas bibliotecas/frameworks para roteamento, validação, logging ou tratamento de exceções, ou se manteria aderência estrita às abstrações já consolidadas no projeto (`[09:27] Bruno`, `[09:30] Larissa`).

## Decisão

Adotar o **reuso máximo das abstrações, middlewares e convenções existentes na codebase**, estruturando a funcionalidade como um módulo padrão em `src/modules/webhooks/` sem adicionar dependências externas desnecessárias (`[09:27] Bruno`, `[09:30] Larissa`).

Os elementos concretos do código-fonte que serão reutilizados e estendidos são:

1. **Arquitetura Modular por Domínio (`src/modules/webhooks/`):** Seguir exatamente a divisão de responsabilidades observada em `src/modules/orders/` e `src/modules/customers/`, organizando o módulo em `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts` e `webhook.worker.ts` (acionado pela entry-point `src/worker.ts`) (`[09:27] Bruno`, `[09:28] Bruno`).
2. **Hierarquia de Erros (`src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`):** Estender `AppError` (e subclasses como `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `ValidationError`) utilizando exclusivamente códigos de erro com prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_PAYLOAD_TOO_LARGE`, `WEBHOOK_DEAD_LETTER_NOT_FOUND`), exatamente como `InsufficientStockError` (`INSUFFICIENT_STOCK`) e `InvalidStatusTransitionError` (`INVALID_STATUS_TRANSITION`) fazem em `src/shared/errors/http-errors.ts` (`[09:28] Bruno`, `[09:29] Larissa`).
3. **Middleware Global de Erros (`src/middlewares/error.middleware.ts`):** Reaproveitar integralmente `errorMiddleware`, que já intercepta instâncias de `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError` (`P2002`, `P2025`), formatando o envelope `{ error: { code, message, details } }` sem necessidade de alterações (`[09:29] Bruno`).
4. **Validação com Zod (`src/middlewares/validate.middleware.ts`):** Validar `body`, `query` e `params` nas rotas de webhooks usando o middleware `validate({ body, query, params })` e schemas Zod (incluindo validação de URL `https://` obrigatória) (`[09:23] Sofia`, `[09:30] Larissa`).
5. **Autenticação JWT e Controle de Acesso (`src/middlewares/auth.middleware.ts`):** Proteger todas as rotas do CRUD de webhooks com o middleware `authenticate` e restringir o endpoint administrativo de replay da DLQ (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`) com `requireRole('ADMIN')` (`[09:32] Larissa`, `[09:36] Sofia`, `[09:36] Larissa`).
6. **Logging Estruturado com Pino (`src/shared/logger/index.ts`):** Utilizar a instância `logger` (Pino) existente tanto na API quanto no worker (e para auditoria do replay de DLQ), adicionando caminhos sensíveis como `*.secret` e `*.previousSecret` à lista `redactPaths` (`[09:29] Bruno`, `[09:36] Sofia`).
7. **Paginação Padronizada (`src/shared/http/response.ts`) e Registro de Rotas (`src/routes/index.ts`, `src/app.ts`):** Utilizar o helper `paginated(data, page, pageSize, total)` para listagens de webhooks e histórico de entregas (`deliveries`), registrando o roteador em `buildApiRouter` (`src/routes/index.ts`) e injetando as dependências em `buildControllers` (`src/app.ts`).
8. **Convenções de Modelagem Prisma (`prisma/schema.prisma`) e Hook Transacional (`src/modules/orders/order.service.ts`):** Utilizar chaves primárias UUID `@id @default(uuid()) @db.Char(36)` em todas as novas tabelas (`[09:51] Larissa`) e integrar a criação de eventos na outbox dentro da transação `this.prisma.$transaction` em `OrderService.changeStatus` via função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` (`[09:40] Bruno`, `[09:41] Bruno`, `[09:41] Diego`).

## Alternativas Consideradas

### Alternativa 1: Introduzir Novos Frameworks de Fila/Validação/Logger ou Tratamento de Erros Paralelo
- **Descrição:** Adicionar bibliotecas externas específicas para o módulo de webhooks (como Winston para logs do worker, BullMQ/agenda ou um formato de erro próprio diferente de `AppError`) (`[09:29] Bruno`).
- **Motivo do Descarte / Trade-off:** Fragmentaria a observabilidade e os contratos JSON da API REST, aumentaria o tamanho do bundle e criaria carga cognitiva desnecessária para a equipe de manutenção, já que `AppError`, `errorMiddleware`, `validate` (Zod) e `logger` (Pino) já atendem 100% dos casos de uso (`[09:29] Bruno`, `[09:30] Larissa`).

### Alternativa 2: Utilizar IDs Numéricos Auto-Incrementais (`Int @default(autoincrement())`) na Outbox e DLQ
- **Descrição:** Modelar as tabelas de webhook com chaves primárias inteiras auto-incrementais em vez de UUID (`[09:51] Diego`).
- **Motivo do Descarte / Trade-off:** Divergiria do padrão de todas as entidades de domínio em `prisma/schema.prisma` (`User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`), que utilizam `@id @default(uuid()) @db.Char(36)`, além de que o próprio `event_id` exposto no header `X-Event-Id` precisa ser um UUID (`[09:25] Diego`, `[09:51] Larissa`).

## Consequências

### Positivas
- **Consistência de Contratos e DX (Developer Experience):** Os consumidores da API recebem exatamente o mesmo formato de paginação (`src/shared/http/response.ts`) e envelope de erro (`src/middlewares/error.middleware.ts`) em `/api/v1/webhooks` que já recebem em `/api/v1/orders` (`[09:28] Bruno`, `[09:29] Bruno`).
- **Velocidade de Implementação (Estimativa de 3 Sprints):** Reaproveitar `authenticate`, `requireRole`, `validate`, `AppError` e `logger` elimina trabalho de boilerplate e reduz a superfície de bugs (`[09:30] Larissa`, `[09:46] Larissa`).
- **Baixo Acoplamento no `OrderService`:** O uso de uma função pura `publishWebhookEvent(tx, ...)` recebendo `Prisma.TransactionClient` evita injetar toda a árvore de serviços/repositórios de webhook no construtor de `OrderService` (`[09:41] Bruno`, `[09:41] Diego`).

### Negativas e Trade-offs Explícitos
- **Atenção à Redação de Segredos no Logger Compartilhado:** Como o `logger` Pino (`src/shared/logger/index.ts`) hoje censura apenas `password`, `passwordHash`, `token` e `accessToken`, reutilizá-lo exige cuidado para não logar o campo `secret` dos webhooks em texto claro (`[09:22] Diego`). **Trade-off aceito:** Basta incluir `*.secret` e `*.previousSecret` no array `redactPaths` de `src/shared/logger/index.ts`.
