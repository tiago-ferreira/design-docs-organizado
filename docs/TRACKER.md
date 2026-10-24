# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

Este documento mapeia de forma cruzada os requisitos funcionais, requisitos não funcionais, decisões arquiteturais, alternativas descartadas, contratos, ganchos de integração e restrições registrados no pacote de documentação (`PRD.md`, `RFC.md`, `FDD.md` e `ADR-001` a `ADR-006`) às suas origens exatas na transcrição da reunião técnica (`TRANSCRICAO.md`) ou no código-fonte da aplicação (`CODIGO`).

## Resumo de Cobertura

- **Total de itens rastreados:** 65 itens
- **Itens com `Fonte = TRANSCRICAO`:** 51 itens (**78,5%** — requisito mínimo: `>= 70%`)
- **Itens com `Fonte = CODIGO`:** 14 itens (**14 linhas** — requisito mínimo: `>= 5 linhas`)
- **Cobertura dos itens identificáveis nos documentos:** **100%** (requisito mínimo: `>= 80%`)

---

## Matriz de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| -- | --------- | ---- | ----------------- | ----- | ----------- |
| PRD-CTX-01 | `docs/PRD.md` | Contexto de Negócio | Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) solicitaram notificação automática de mudança de status para eliminar polling em `GET /orders`, sob risco de churn da Atlas até fim do trimestre. | `TRANSCRICAO` | `[09:00] Marcos` |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo / Métrica | Meta quantitativa de latência fim-a-fim abaixo de 10 segundos (com polling de 2s do worker) para caracterizar entrega em "tempo real". | `TRANSCRICAO` | `[09:02] Marcos` |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo / Prazo | Entrega da feature em 3 sprints (incluindo 2 dias úteis de revisão de segurança da Sofia) para atender ao prazo de fim de novembro da Atlas Comercial. | `TRANSCRICAO` | `[09:46] Larissa` |
| PRD-ESC-01 | `docs/PRD.md` | Escopo | Webhooks são exclusivamente outbound (do OMS para os clientes B2B); clientes não enviam webhooks para a plataforma. | `TRANSCRICAO` | `[09:02] Marcos` |
| PRD-OOS-01 | `docs/PRD.md` | Fora de Escopo | Notificações por e-mail ao cliente após 3 falhas consecutivas de entrega foram descartadas para esta fase e adiadas para fase futura. | `TRANSCRICAO` | `[09:37] Larissa` |
| PRD-OOS-02 | `docs/PRD.md` | Fora de Escopo | Dashboard visual / painel frontend para gestão de webhooks está fora de escopo (entrega restrita a endpoints de API REST). | `TRANSCRICAO` | `[09:40] Larissa` |
| PRD-OOS-03 | `docs/PRD.md` | Fora de Escopo | Rate limiting de saída (envio para clientes com dezenas de mudanças por minuto) ficou fora do escopo inicial para observação em produção. | `TRANSCRICAO` | `[09:39] Diego` |
| PRD-OOS-04 | `docs/PRD.md` | Fora de Escopo | Arquivamento/expurgo automático de linhas entregues na outbox após 30 dias ficou fora do escopo desta feature. | `TRANSCRICAO` | `[09:08] Diego` |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastro de endpoint de webhook (`POST`) recebendo URL, lista de status assinados e `customerId` explícito, retornando a `secret` gerada pela plataforma. | `TRANSCRICAO` | `[09:31] Marcos` |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | O `customerId` no cadastro/listagem de webhooks é passado no body ou query/path (não vem do JWT, pois o JWT representa o usuário operador). | `TRANSCRICAO` | `[09:32] Larissa` |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Operações de edição (`PATCH`), remoção (`DELETE`) e listagem (`GET`) dos webhooks de um cliente com acesso permitido a qualquer role autenticada nesta fase. | `TRANSCRICAO` | `[09:33] Bruno` |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Filtro de eventos por lista de status (`OrderStatus`) aplicado no momento da inserção na outbox (se nenhum webhook do cliente ouvir o status, não insere). | `TRANSCRICAO` | `[09:34] Bruno` |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Registro atômico do evento na `webhook_outbox` dentro da mesma transação de `changeStatus`, com snapshot JSON já renderizado na inserção. | `TRANSCRICAO` | `[09:52] Larissa` |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Rotação de `secret` por endpoint via API mantendo a `secret` antiga válida em paralelo por 24 horas (*grace period*). | `TRANSCRICAO` | `[09:21] Sofia` |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Política de retry exponencial com 5 tentativas (`1m`, `5m`, `30m`, `2h`, `12h`) e movimentação posterior para tabela `webhook_dead_letter`. | `TRANSCRICAO` | `[09:17] Diego` |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Endpoint `GET /webhooks/:id/deliveries` retornando as últimas 100 entregas com sucesso/falha, payload, response e tempo de resposta. | `TRANSCRICAO` | `[09:34] Marcos` |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Endpoint `POST /admin/webhooks/dead-letter/:id/replay` restrito à role `ADMIN` com log de auditoria de quem acionou o reprocessamento. | `TRANSCRICAO` | `[09:36] Sofia` |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Obrigatoriedade de URL `https://` (TLS) validada via schema Zod, recusando cadastros `http://`. | `TRANSCRICAO` | `[09:23] Sofia` |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Limite máximo de tamanho de payload de 64KB, retornando erro caso o evento ultrapasse esse teto. | `TRANSCRICAO` | `[09:24] Diego` |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Timeout HTTP de 10 segundos por tentativa de envio do worker; falta de resposta em 10s é tratada como falha para retry. | `TRANSCRICAO` | `[09:42] Diego` |
| PRD-RISK-01 | `docs/PRD.md` | Risco e Mitigação | Risco de clientes com janela de manutenção planejada de até 2 horas perderem eventos, mitigado pela janela de 5 retries (~15 horas) e DLQ. | `TRANSCRICAO` | `[09:16] Diego` |
| PRD-RISK-02 | `docs/PRD.md` | Risco e Mitigação | Risco de vazamento de `secret` em log de aplicação do cliente, mitigado por secret única por endpoint e rotação com grace period de 24h. | `TRANSCRICAO` | `[09:22] Diego` |
| RFC-META-01 | `docs/RFC.md` | Metadados | Participantes e revisores da proposta arquitetural: Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno), Diego (Eng. Sênior) e Sofia (Eng. Segurança). | `TRANSCRICAO` | `[09:00] Larissa` |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa Descartada | Descarte de chamada HTTP síncrona dentro de `OrderService.changeStatus` para não travar a transação de banco nem dar rollback no pedido se o cliente estiver offline. | `TRANSCRICAO` | `[09:04] Bruno` |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa Descartada | Descarte de Redis Streams / Redis Cluster por exigir subir e manter mais infraestrutura em um time pequeno (*overengineering*). | `TRANSCRICAO` | `[09:07] Diego` |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa Descartada | Descarte de triggers nativos do MySQL para acionar o worker, pois o MySQL não possui `LISTEN/NOTIFY` nativo para processos externos como o Postgres. | `TRANSCRICAO` | `[09:09] Diego` |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa Descartada | Descarte de limite curto de 3 tentativas (insuficiente para manutenções de 2h) e de retry indefinido (eventos pendurados eternamente se o cliente sumir). | `TRANSCRICAO` | `[09:15] Diego` |
| RFC-ALT-05 | `docs/RFC.md` | Alternativa Descartada | Descarte de entrega *exactly-once* por exigir coordenação complexa dos dois lados, adotando *at-least-once* com `X-Event-Id` (padrão Stripe/GitHub). | `TRANSCRICAO` | `[09:25] Diego` |
| RFC-ALT-06 | `docs/RFC.md` | Alternativa Descartada | Descarte de guardar apenas `order_id` na outbox para renderizar na hora do envio, evitando enviar estado alterado posteriormente. | `TRANSCRICAO` | `[09:51] Bruno` |
| RFC-OPEN-01 | `docs/RFC.md` | Questão em Aberto | Rate limiting de saída quando um cliente tem 50 pedidos mudando de status em 1 minuto: observar em produção e decidir depois. | `TRANSCRICAO` | `[09:38] Diego` |
| RFC-OPEN-02 | `docs/RFC.md` | Questão em Aberto | Endurecimento futuro de permissões (roles) nas rotas do CRUD de configuração de webhooks (atualmente aberto a qualquer role autenticada). | `TRANSCRICAO` | `[09:37] Sofia` |
| RFC-OPEN-03 | `docs/RFC.md` | Questão em Aberto | Escalabilidade futura para múltiplos workers paralelos via particionamento por `order_id` ou lock pessimista. | `TRANSCRICAO` | `[09:13] Diego` |
| ADR-001 | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Decisão Arquitetural | Adoção do padrão Transactional Outbox na tabela `webhook_outbox` do MySQL na mesma transação de mudança de status, com índices em `status` e `created_at`. | `TRANSCRICAO` | `[09:06] Diego` |
| ADR-001-SNAP | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Decisão Arquitetural | Persistência do snapshot JSON renderizado na inserção da outbox com chaves primárias em UUID seguindo o padrão do projeto. | `TRANSCRICAO` | `[09:51] Larissa` |
| ADR-002 | `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` | Decisão Arquitetural | Execução do worker em processo Node.js separado (`src/worker.ts` / `npm run worker`) com polling de 2 segundos para não perder o worker se a API reiniciar. | `TRANSCRICAO` | `[09:11] Diego` |
| ADR-002-PRISMA | `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` | Decisão Arquitetural | Instanciação de um `PrismaClient` separado para o processo do worker conectado ao mesmo banco (`DATABASE_URL`). | `TRANSCRICAO` | `[09:30] Bruno` |
| ADR-003 | `docs/adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md` | Decisão Arquitetural | Política de 5 tentativas de retry (`1m`, `5m`, `30m`, `2h`, `12h`) e persistência de falhas permanentes em tabela separada `webhook_dead_letter`. | `TRANSCRICAO` | `[09:18] Diego` |
| ADR-003-REPLAY | `docs/adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md` | Decisão Arquitetural | Replay manual de eventos da DLQ via endpoint `POST /admin/webhooks/dead-letter/:id/replay` exigindo obrigatoriamente role `ADMIN` no JWT. | `TRANSCRICAO` | `[09:36] Larissa` |
| ADR-004 | `docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md` | Decisão Arquitetural | Assinatura HMAC-SHA256 sobre o corpo do request no header `X-Signature` com secret única por endpoint e rotação com grace period de 24h. | `TRANSCRICAO` | `[09:20] Sofia` |
| ADR-005 | `docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md` | Decisão Arquitetural | Garantia de entrega *at-least-once* com UUID único por evento no header `X-Event-Id` para deduplicação pelo cliente. | `TRANSCRICAO` | `[09:24] Diego` |
| ADR-005-ORD | `docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md` | Restrição / Trade-off | Ordenação implícita por `order_id` (`created_at`) garantida apenas enquanto houver um único worker rodando (*single-worker*). | `TRANSCRICAO` | `[09:12] Diego` |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md` | Decisão Arquitetural | Reuso máximo da estrutura modular em `src/modules/webhooks`, `AppError` com prefixo `WEBHOOK_`, logger Pino, `errorMiddleware` e Zod. | `TRANSCRICAO` | `[09:30] Larissa` |
| ADR-006-CODE-01 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md` | Padrão de Código | Classe base `AppError(message, statusCode, errorCode, details)` reutilizada para todas as exceções de domínio do módulo de webhooks. | `CODIGO` | `src/shared/errors/app-error.ts` |
| ADR-006-CODE-02 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md` | Padrão de Código | Subclasses de erro (`NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `InsufficientStockError`, `InvalidStatusTransitionError`) como modelo para erros `WEBHOOK_*`. | `CODIGO` | `src/shared/errors/http-errors.ts` |
| FDD-FLOW-01 | `docs/FDD.md` | Fluxo Técnico | Integração na transação de `OrderService.changeStatus` via função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` client. | `TRANSCRICAO` | `[09:41] Bruno` |
| FDD-FLOW-02 | `docs/FDD.md` | Regra Transacional | Rollback obrigatório da mudança de status do pedido caso a inserção na `webhook_outbox` falhe dentro da transação. | `TRANSCRICAO` | `[09:40] Bruno` |
| FDD-PAYLOAD-01 | `docs/FDD.md` | Contrato de Payload | Estrutura enxuta do JSON outbound: `event_id`, `event_type` (`"order.status_changed"`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`, sem `items`. | `TRANSCRICAO` | `[09:43] Diego` |
| FDD-HDR-01 | `docs/FDD.md` | Contrato de Headers | Inclusão dos headers HTTP `X-Event-Id`, `X-Signature`, `X-Timestamp` (anti-replay attack) e `Content-Type: application/json`. | `TRANSCRICAO` | `[09:44] Diego` |
| FDD-HDR-02 | `docs/FDD.md` | Contrato de Headers | Inclusão do header `X-Webhook-Id` com o ID do endpoint cadastrado para clientes com múltiplos webhooks identificarem a origem. | `TRANSCRICAO` | `[09:44] Sofia` |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato HTTP | Endpoint `POST /api/v1/webhooks` para cadastro de configuração de webhook com geração de `secret` e validação HTTPS. | `TRANSCRICAO` | `[09:31] Marcos` |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato HTTP | Endpoints `GET /api/v1/webhooks`, `PATCH /api/v1/webhooks/:id` e `DELETE /api/v1/webhooks/:id` para gestão de endpoints por cliente. | `TRANSCRICAO` | `[09:33] Bruno` |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato HTTP | Endpoint `POST /api/v1/webhooks/:id/rotate-secret` para rotação de secret com validade paralela de 24 horas da secret anterior. | `TRANSCRICAO` | `[09:21] Sofia` |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato HTTP | Endpoint `GET /api/v1/webhooks/:id/deliveries` para listagem das últimas 100 tentativas de entrega com payload, status e latência. | `TRANSCRICAO` | `[09:34] Marcos` |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato HTTP | Endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` para reprocessamento manual de evento em DLQ por usuário `ADMIN`. | `TRANSCRICAO` | `[09:35] Diego` |
| FDD-ERR-01 | `docs/FDD.md` | Matriz de Erros | Padronização dos códigos de erro do módulo com prefixo `WEBHOOK_` (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, etc.). | `TRANSCRICAO` | `[09:28] Bruno` |
| FDD-INT-01 | `docs/FDD.md` | Integração com Código | Extensão da transação `this.prisma.$transaction` no método `OrderService.changeStatus` após `tx.orderStatusHistory.create`. | `CODIGO` | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração com Código | Adição dos modelos `WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery` e `WebhookDeadLetter` com IDs `@id @default(uuid()) @db.Char(36)`. | `CODIGO` | `prisma/schema.prisma` |
| FDD-INT-03 | `docs/FDD.md` | Integração com Código | Reaproveitamento do middleware `errorMiddleware` que serializa automaticamente `AppError`, `ZodError` e `PrismaClientKnownRequestError`. | `CODIGO` | `src/middlewares/error.middleware.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração com Código | Reaproveitamento dos middlewares `authenticate` e `requireRole('ADMIN')` para autenticação JWT e controle de acesso por role. | `CODIGO` | `src/middlewares/auth.middleware.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração com Código | Reaproveitamento do middleware `validate({ body, query, params })` para validação de schemas Zod nas rotas de webhooks. | `CODIGO` | `src/middlewares/validate.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração com Código | Reaproveitamento do logger estruturado Pino (`createLogger` / `logger`) e extensão do array `redactPaths` para ocultar `secret`. | `CODIGO` | `src/shared/logger/index.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração com Código | Reaproveitamento do helper `paginated(data, page, pageSize, total)` para respostas paginadas de webhooks. | `CODIGO` | `src/shared/http/response.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração com Código | Registro dos roteadores `/webhooks` e `/admin/webhooks` na função `buildApiRouter(controllers)`. | `CODIGO` | `src/routes/index.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração com Código | Instanciação e injeção de `WebhookRepository`, `WebhookService` e `WebhookController` na função `buildControllers(prisma)`. | `CODIGO` | `src/app.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração com Código | Reuso da fábrica `createPrismaClient()` e padrão de bootstrap/graceful shutdown (`SIGINT`/`SIGTERM`) para o novo processo `src/worker.ts`. | `CODIGO` | `src/config/database.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração com Código | Referência ao padrão de inicialização e encerramento gracioso do servidor HTTP para estruturar o entry-point paralelo do worker. | `CODIGO` | `src/server.ts` |
| FDD-INT-12 | `docs/FDD.md` | Integração com Código | Reuso das dependências já instaladas (`@prisma/client`, `zod`, `pino`, `uuid`, Node `>=20`) e adição do script `worker`. | `CODIGO` | `package.json` |
