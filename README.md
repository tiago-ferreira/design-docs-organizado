# Da Reunião ao Documento: Design Docs Gerados por IA — Sistema de Webhooks (OMS)

> Documentação do processo de engenharia de software assistida por IA para especificação completa do **Sistema de Webhooks de Notificação de Pedidos**. O enunciado original do desafio está preservado como referência em [`design-docs-organizado.md`](./design-docs-organizado.md).

---

## 1. Sobre o Desafio

Neste desafio técnico, o objetivo foi transformar a gravação literal de uma reunião de arquitetura e produto de ~55 minutos ([`TRANSCRICAO.md`](./TRANSCRICAO.md)) e o código-fonte real de um **Order Management System (OMS)** em Node.js + TypeScript + Express + Prisma + MySQL em um pacote completo, coeso e acionável de **Design Docs** (`PRD`, `RFC`, `FDD`, `6 ADRs` e `Tracker de Rastreabilidade`).

A reunião entre Larissa (Tech Lead), Marcos (PM), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior) e Sofia (Engenheira de Segurança) definiu como atender três clientes B2B estratégicos (**Atlas Comercial**, **MaxDistribuição** e **Nova Cargo**) que exigiam notificações de mudança de status de pedidos em menos de 10 segundos. O desafio exigiu separar rigorosamente o que foi **decidido**, o que foi **descartado** (como chamadas HTTP síncronas na transação, Redis Streams e 3 tentativas de retry) e o que foi **adiado/deixado em aberto** (como notificações por e-mail, dashboard visual, rate limiting de saída e arquivamento após 30 dias), garantindo 100% de rastreabilidade ao código e à transcrição, sem alterar nenhuma linha de código da aplicação (`src/`, `prisma/`, `tests/`).

---

## 2. Ferramentas de IA Utilizadas

| Ferramenta / Agente | Papel Desempenhado no Processo |
| :--- | :--- |
| **Google Antigravity (Gemini) + Agente de Engenharia** | Leitura cruzada da transcrição (`TRANSCRICAO.md`) e inspeção estática profunda do repositório (`prisma/schema.prisma`, `src/modules/orders/order.service.ts`, middlewares, erros e logger), estruturação dos documentos em alturas distintas e auditoria de consistência. |
| **OpenSpec (`spec-driven` workflow: `/openspec-propose` & `/openspec-apply-change`)** | Governança do ciclo de planejamento antes da escrita: geração estruturada de `proposal.md`, `specs/order-webhook-design-docs/spec.md`, `design.md` e `tasks.md` para validar todos os critérios de aceite antes de redigir os artefatos finais em `docs/`. |

---

## 3. Workflow Adotado

Para evitar sobreposição de conteúdo entre os documentos (respeitando a altura de cada artefato: *Decisão Pontual → Arquitetura → Implementação → Produto → Rastreabilidade*), adotamos um workflow estruturado em **6 etapas sequenciais**:

1. **Exploração e Mapeamento Cruzado (`TRANSCRICAO.md` × `src/` e `prisma/`):**
   - Leitura integral dos 324 parágrafos da transcrição (`[09:00]` a `[09:53]`, incluindo a conversa pós-reunião entre Diego, Bruno e Larissa sobre UUIDs e snapshot do payload renderizado na inserção).
   - Inspeção dos arquivos reais do OMS (`src/modules/orders/order.service.ts`, `prisma/schema.prisma`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `src/routes/index.ts`, `src/app.ts`, `src/server.ts`).
2. **Planejamento Guiado por Especificação (`OpenSpec`):**
   - Criação da mudança `design-docs-organizado` com especificação formal dos requisitos documentais e checklist verificável de 13 tarefas.
3. **Produção dos 6 ADRs Primeiro (`docs/adrs/ADR-001` a `ADR-006`):**
   - Formalização no padrão MADR das 6 decisões arquiteturais centrais tomadas na call, documentando contexto, decisão, alternativas descartadas e consequências com trade-offs explícitos.
4. **Consolidação Arquitetural no RFC (`docs/RFC.md`):**
   - Redação concisa (nível arquitetural) apresentando o resumo executivo, a visão da solução, as 6 alternativas descartadas com seus trade-offs, as 5 questões em aberto/adiadas e os links para os 6 ADRs.
5. **Detalhamento de Implementação no FDD (`docs/FDD.md`) e Visão de Produto no PRD (`docs/PRD.md`):**
   - No **FDD**: definição dos 4 modelos Prisma (`WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery`, `WebhookDeadLetter`), diagramas de sequência/fluxo, 7 contratos de endpoints HTTP com payloads de exemplo, matriz de erros `WEBHOOK_*`, observabilidade (Pino, métricas, tracing) e tabela de integração com 9 arquivos reais do projeto.
   - No **PRD**: consolidação dos objetivos de negócio, métricas quantitativas (`< 10s` de latência, retenção dos 3 clientes B2B em 3 sprints), 9 requisitos funcionais, 6 não funcionais, itens fora de escopo e matriz de riscos.
6. **Auditoria de Rastreabilidade (`docs/TRACKER.md`) e Documentação de Processo (`README.md`):**
   - Mapeamento de 65 itens granulares (78,5% `TRANSCRICAO` com `[hh:mm] Nome` e 14 linhas `CODIGO` com caminhos reais) e verificação final de imutabilidade do código-fonte via `git status`.

---

## 4. Prompts Customizados

Abaixo estão dois dos principais prompts dirigidos utilizados para orientar a IA na extração sem alucinações e na separação rigorosa de responsabilidades entre RFC e FDD:

### Prompt 1 — Extração Estruturada e Filtragem Anti-Alucinação da Transcrição

```markdown
Atue como Staff Software Engineer e Product Architect. Analise integralmente o arquivo `TRANSCRICAO.md` (do minuto [09:00] até o pós-call em [09:53]) em conjunto com o código real em `src/` e `prisma/schema.prisma`.

Classifique cada ponto discutido em 4 categorias estritas, citando obrigatoriamente `[hh:mm] Nome` ou o caminho real do arquivo:
1. DECISÕES FECHADAS E REQUISITOS APROVADOS (incluindo detalhes do pós-call como IDs UUID e snapshot do payload na inserção).
2. ALTERNATIVAS TÉCNICAS DESCARTADAS + o trade-off exato que motivou o descarte (para o RFC e ADRs).
3. ITENS FORA DE ESCOPO / ADIADOS PARA PRÓXIMA FASE (para a seção "Fora de escopo" do PRD).
4. QUESTÕES EM ABERTO ("observar e decidir depois", para a seção "Questões em aberto" do RFC).

Regra absoluta: Não invente nenhum requisito, header, endpoint ou tecnologia que não esteja em `TRANSCRICAO.md` ou no código-fonte.
```

### Prompt 2 — Geração do FDD Acionável com Ganchos Reais de Código e Contratos HTTP

```markdown
Redija o arquivo `docs/FDD.md` operando estritamente na altura de IMPLEMENTAÇÃO (sem repetir o texto opinativo do RFC). O documento deve ser acionável para um desenvolvedor iniciar a codificação imediatamente e deve conter:

1. Modelagem Prisma compatível com `prisma/schema.prisma` (usando `@id @default(uuid()) @db.Char(36)`) para `webhook_endpoints` (incluindo suporte ao grace period de 24h na rotação de secret), `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`.
2. Fluxos detalhados com diagramas Mermaid:
   - Inserção transacional via função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), filtrando antes de inserir e validando o teto de 64KB.
   - Worker separado (`src/worker.ts`) em polling de 2s, timeout HTTP de 10s, 5 tentativas de retry (1m, 5m, 30m, 2h, 12h) e movimentação para `webhook_dead_letter`.
   - Replay manual via `POST /api/v1/admin/webhooks/dead-letter/:id/replay` exigindo `requireRole('ADMIN')` e log de auditoria no Pino.
3. Contratos HTTP públicos com payloads JSON de request/response e status codes para o CRUD de webhooks, rotação de secret, histórico das últimas 100 entregas (`deliveries`) e replay de DLQ, além do contrato outbound com os headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`.
4. Matriz de erros estendendo `AppError` (`src/shared/errors/app-error.ts`) com códigos prefixados por `WEBHOOK_`.
5. Seção "Integração com o sistema existente" detalhando a conexão com pelo menos 6 caminhos reais de arquivos do repositório.
```

---

## 5. Iterações e Ajustes

Durante a produção e revisão crítica dos documentos, realizamos **4 iterações principais** onde identificamos e corrigimos limitações importantes:

1. **Iteração 1 — Captura das Decisões do "Pós-Call" (`[09:50]` a `[09:53]`):**
   - *Problema identificado:* Em uma leitura superficial inicial, seria fácil encerrar a extração no resumo da Larissa às `[09:48]`, perdendo duas decisões cruciais tomadas depois que Marcos e Sofia saíram da call: o uso de **UUID** na outbox (`[09:51] Larissa`) e a persistência do **snapshot do payload já renderizado na hora da inserção** em vez de renderizar no momento do envio (`[09:51] Bruno`, `[09:52] Larissa`).
   - *Ajuste aplicado:* Incluímos explicitamente a regra de snapshot JSON na inserção e a justificativa de evitar inconsistência de estado tardio tanto no `ADR-001` quanto no `RFC.md`, `FDD.md` e `TRACKER.md`.
2. **Iteração 2 — Correção da Origem do `customerId` nas Rotas da API (`[09:31]` vs. `[09:32]`):**
   - *Problema identificado:* Às `[09:31]`, o PM Marcos sugeriu inicialmente que o `customer_id` viria implícito do JWT. Um gerador desatento transformaria isso em requisito. Porém, às `[09:32]`, Bruno e Larissa corrigiram o ponto explicando que o JWT atual autentica o usuário operador (`users` em `src/middlewares/auth.middleware.ts`), decidindo que o `customerId` deve ser passado explicitamente no body ou query/path.
   - *Ajuste aplicado:* Ajustamos todos os contratos de `POST /api/v1/webhooks` e `GET /api/v1/webhooks` no `FDD.md` e o `RF-01` no `PRD.md` para receberem `customerId` explicitamente, registrando a rastreabilidade em `[09:32] Larissa`.
3. **Iteração 3 — Modelagem do Grace Period de 24 Horas na Rotação de Secret e Redação no Logger Pino:**
   - *Problema identificado:* A primeira versão mental da tabela `webhook_endpoints` continha apenas uma coluna `secret`. Contudo, uma coluna única impediria cumprir o requisito de segurança da Sofia (`[09:21]`) de manter a secret antiga válida em paralelo por 24 horas após a rotação. Além disso, o `src/shared/logger/index.ts` existente não incluía `*.secret` no `redactPaths`.
   - *Ajuste aplicado:* Adicionamos `previousSecret` e `previousSecretExpiresAt` na modelagem Prisma do `FDD.md` e especificamos a inclusão de `'*.secret'` e `'*.previousSecret'` no `redactPaths` do Pino (`ADR-006` e `FDD.md`).
4. **Iteração 4 — Balanceamento de Fronteira entre RFC e FDD e Auditoria do Tracker:**
   - *Problema identificado:* Garantir que o `RFC.md` permanecesse conciso (2 a 4 páginas, focado em decisão, alternativas descartadas e questões em aberto) sem duplicar os schemas Prisma e payloads JSON extensos do `FDD.md`, e validar que o `TRACKER.md` superasse com folga os limites de `>= 80%` de cobertura, `>= 70%` `TRANSCRICAO` e `>= 5` linhas `CODIGO`.
   - *Ajuste aplicado:* Enxugamos a seção 3 do `RFC.md` para nível arquitetural com links diretos para o `FDD.md` e para os 6 ADRs, e estruturamos 65 linhas auditadas no `docs/TRACKER.md` (51 `TRANSCRICAO` [78,5%] e 14 `CODIGO`).

---

## 6. Como Navegar a Entrega

Abaixo está o mapa completo dos artefatos entregues e a **ordem sugerida de leitura** para compreender do problema de negócio até a especificação de implementação:

| Ordem Sugerida | Documento | Altura / Papel | Descrição do Conteúdo |
| :---: | :--- | :--- | :--- |
| **1º** | **[`docs/PRD.md`](./docs/PRD.md)** | Produto / Negócio (*Por que e o quê?*) | Contexto dos 3 clientes B2B, metas quantitativas (`< 10s`), escopo incluso e 5 itens fora de escopo, 9 requisitos funcionais, 6 não funcionais, riscos e estratégia de testes. |
| **2º** | **[`docs/RFC.md`](./docs/RFC.md)** | Arquitetura (*Como propomos resolver e o que está em aberto?*) | Proposta arquitetural concisa para revisão dos 5 participantes, 6 alternativas descartadas com trade-offs, 5 questões em aberto e links para os ADRs. |
| **3º** | **[`docs/adrs/`](./docs/adrs/)** | Decisões Pontuais (*Por que decidimos exatamente assim?*) | **6 ADRs no formato MADR:**<br>• [`ADR-001-padrao-outbox-no-mysql.md`](./docs/adrs/ADR-001-padrao-outbox-no-mysql.md)<br>• [`ADR-002-worker-em-processo-separado-com-polling.md`](./docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md)<br>• [`ADR-003-politica-de-retry-com-backoff-e-dlq.md`](./docs/adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md)<br>• [`ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md`](./docs/adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md)<br>• [`ADR-005-garantia-at-least-once-com-x-event-id.md`](./docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)<br>• [`ADR-006-reuso-dos-padroes-existentes-do-projeto.md`](./docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| **4º** | **[`docs/FDD.md`](./docs/FDD.md)** | Implementação (*Como construir em detalhe?*) | Schemas Prisma, diagramas de fluxo (Outbox, Worker 2s, Retry 5x, DLQ, Replay ADMIN), 7 contratos HTTP com exemplos JSON, matriz de erros `WEBHOOK_*`, observabilidade e integração com 9 arquivos reais da codebase. |
| **5º** | **[`docs/TRACKER.md`](./docs/TRACKER.md)** | Rastreabilidade Transversal (*De onde veio cada coisa?*) | Tabela cruzada com 65 itens ligando cada requisito, decisão, contrato e integração aos timestamps `[hh:mm] Nome` de `TRANSCRICAO.md` (78,5%) e aos arquivos reais de `CODIGO` (14 itens). |
