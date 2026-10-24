# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

| Metadado | Valor |
| :--- | :--- |
| **Produto / Plataforma** | Order Management System (OMS) |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos (Outbound) |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Engenharia & Segurança** | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Prazo Alvo** | Final de Novembro / 3 Sprints (incluindo revisão de segurança) |
| **Documentos Técnicos** | [RFC-001](./RFC.md) \| [FDD](./FDD.md) \| [ADRs](./adrs/) \| [Tracker](./TRACKER.md) |

---

## 1. Resumo e Contexto da Feature

O Order Management System (OMS) opera em produção gerenciando o ciclo de vida completo de pedidos B2B (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`), incluindo controle transacional de estoque e histórico de auditoria (`prisma/schema.prisma`, `src/modules/orders/order.service.ts`). No entanto, a plataforma não dispõe de nenhum mecanismo de notificação ativa para sistemas externos quando o status de um pedido muda (`[09:00] Marcos`).

Esta feature introduz um **Sistema de Webhooks Outbound de Notificação de Pedidos**, permitindo que clientes B2B cadastrem endpoints HTTPS via API, selecionem quais mudanças de status desejam escutar, recebam notificações autenticadas via HMAC-SHA256 em menos de 10 segundos e consultem o histórico de entregas (`[09:02] Marcos`, `[09:03] Sofia`, `[09:20] Sofia`, `[09:31]–[09:34] Marcos`).

---

## 2. Problema e Motivação

Três clientes B2B estratégicos — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — formalizaram a necessidade urgente de receber notificações automáticas em tempo real sobre mudanças de status de seus pedidos (`[09:00] Marcos`).

Atualmente:
- Os sistemas integrados desses clientes precisam consultar repetidamente o endpoint `GET /api/v1/orders` (*polling* externo) para descobrir se houve atualização (`[09:00] Marcos`).
- Esse modelo torna a integração **lenta, ineficiente e cara** para os clientes e gera carga desnecessária de leitura na API do OMS (`[09:00] Marcos`).
- **Risco crítico de Churn:** A **Atlas Comercial** sinalizou explicitamente que, caso a funcionalidade não seja entregue até o final do trimestre (fim de novembro), poderá migrar sua operação para um concorrente (`[09:00] Marcos`, `[09:45] Marcos`).

---

## 3. Público-Alvo e Cenários de Uso

### 3.1. Público-Alvo
1. **Clientes B2B Integradores (Atlas Comercial, MaxDistribuição, Nova Cargo e futuros clientes B2B):** Equipes de engenharia dos clientes e usuários representantes autenticados via JWT na nossa API que integram seus ERPs/WMS/sistemas logísticos ao OMS (`[09:00] Marcos`, `[09:32] Marcos`).
2. **Administradores da Plataforma OMS (`Role: ADMIN`):** Engenheiros de suporte/plataforma responsáveis por monitorar falhas permanentes na Dead Letter Queue (DLQ) e acionar o reprocessamento manual (`replay`) quando o ambiente de um cliente se recupera (`[09:18] Diego`, `[09:36] Sofia`).

### 3.2. Cenários de Uso Principais
- **Cenário 1 — Acompanhamento Logístico Seletivo:** Um cliente B2B cadastra seu endpoint HTTPS filtrando apenas os eventos `SHIPPED` e `DELIVERED` para acionar seu sistema de recebimento assim que a mercadoria sai para entrega, sem receber ruído de status intermediários (`[09:33] Marcos`).
- **Cenário 2 — Recuperação após Manutenção Planejada do Cliente:** O ERP do cliente entra em manutenção planejada por 2 horas pela manhã; o OMS retenta automaticamente com backoff exponencial (até ~15h de janela) e entrega todas as notificações pendentes assim que o sistema do cliente volta ao ar, sem perda de dados (`[09:16] Diego`, `[09:17] Diego`).
- **Cenário 3 — Rotação de Credencial sem Downtime:** O time do cliente percebe que a `secret` do webhook vazou em um log interno de aplicação; ele aciona o endpoint de rotação de secret pela API, atualiza seu servidor dentro da janela de 24 horas (*grace period*) em que ambas as secrets permanecem válidas e conclui a migração sem perder nenhum evento (`[09:21] Sofia`, `[09:22] Diego`).
- **Cenário 4 — Auditoria e Autoatendimento de Diagnóstico:** O desenvolvedor do cliente consulta `GET /api/v1/webhooks/:id/deliveries` para inspecionar os últimos 100 disparos, verificando payload enviado, status HTTP retornado pelo seu servidor e tempo de resposta (`[09:34] Marcos`).

---

## 4. Objetivos e Métricas de Sucesso

| ID | Objetivo de Produto / Negócio | Métrica Quantitativa | Meta de Sucesso | Fonte |
| :--- | :--- | :--- | :--- | :--- |
| **OBJ-01** | Entregar notificações de mudança de status em "tempo real" para eliminar o polling manual dos clientes B2B | Latência fim-a-fim entre o commit da mudança de status do pedido e o disparo HTTP pelo worker (com ciclo de polling de 2s) | **< 10 segundos** (latência base de detecção de **<= 2 segundos** em operação normal) | `[09:02] Marcos`, `[09:09] Diego`, `[09:10] Larissa` |
| **OBJ-02** | Garantir a retenção comercial dos clientes B2B solicitantes dentro do prazo trimestral | Taxa de retenção dos 3 clientes B2B solicitantes (Atlas Comercial, MaxDistribuição e Nova Cargo) e entrega em produção | **100% de retenção (3 de 3 clientes)** com entrega concluída em **<= 3 sprints** (até fim de novembro) | `[09:00] Marcos`, `[09:45] Marcos`, `[09:46] Larissa` |
| **OBJ-03** | Assegurar confiabilidade na entrega frente a indisponibilidades transitórias dos clientes | Cobertura de janela de retentativa automática antes de envio para DLQ e perda zero de eventos commitados | **100% dos eventos commitados registrados na outbox** com cobertura automática de indisponibilidade de até **~15 horas (5 tentativas)** | `[09:06] Diego`, `[09:17] Diego`, `[09:40] Bruno` |

---

## 5. Escopo (Incluso e Fora de Escopo)

### 5.1. Escopo Incluso (Fase Atual)
- Disparo exclusivo de **Outbound Webhooks** (do OMS para o cliente) a cada transição de status em `OrderService.changeStatus` (`[09:02] Marcos`, `[09:03] Sofia`, `[09:40] Bruno`).
- API REST autenticada via JWT para **CRUD completo de endpoints de webhook** (`POST`, `GET`, `PATCH`, `DELETE`) vinculados a um `customerId`, com filtro configurável de status (`OrderStatus`) por endpoint (`[09:31]–[09:33] Marcos/Larissa/Bruno`).
- Geração automática de `secret` exclusiva por endpoint e endpoint para **rotação de secret com *grace period* de 24 horas** (`[09:21] Sofia`, `[09:31] Marcos`).
- Assinatura **HMAC-SHA256** (`X-Signature`), identificador de idempotência (`X-Event-Id`), timestamp de envio (`X-Timestamp`) e identificador do webhook (`X-Webhook-Id`) (`[09:20] Sofia`, `[09:25] Diego`, `[09:44] Diego`, `[09:44] Sofia`).
- Consulta do **histórico das últimas 100 entregas** por endpoint (`GET /api/v1/webhooks/:id/deliveries`) (`[09:34] Marcos`).
- Política de **5 tentativas com backoff exponencial** (`1m`, `5m`, `30m`, `2h`, `12h`), persistência em **Dead Letter Queue (`webhook_dead_letter`)** e endpoint administrativo (`Role: ADMIN`) para **replay manual** com log de auditoria (`[09:17]–[09:18] Diego`, `[09:35]–[09:36] Diego/Sofia/Larissa`).

### 5.2. Fora de Escopo (Descartados ou Adiados na Reunião)
1. **Notificações por E-mail em Caso de Falhas Consecutivas (`[09:37] Marcos`, `[09:37] Larissa`):** O envio de alertas por e-mail ao cliente após 3 falhas seguidas foi explicitamente descartado para esta fase e adiado para uma fase futura após medição de impacto.
2. **Dashboard Visual / Painel Frontend para Gestão de Webhooks (`[09:39] Marcos`, `[09:40] Larissa`):** Não haverá interface gráfica nesta entrega; toda a operação será feita via endpoints de API REST documentados no portal do desenvolvedor.
3. **Webhooks Inbound (Recebimento de Eventos dos Clientes) (`[09:02] Sofia`, `[09:02] Marcos`):** Os clientes apenas recebem notificações da plataforma; fluxos de entrada via webhook não fazem parte do produto.
4. **Rate Limiting de Envio (Saída) por Cliente (`[09:38] Diego`, `[09:39] Larissa`):** O controle de vazão de chamadas disparadas para um mesmo cliente em picos de atualização não será implementado agora; o comportamento será apenas observado em produção.
5. **Rotina Automatizada de Arquivamento/Expurgo de Eventos Entregues Após 30 Dias (`[09:08] Diego`):** O arquivamento periódico de registros antigos na `webhook_outbox` foi deixado fora do escopo desta feature.

---

## 6. Requisitos Funcionais

| ID | Requisito Funcional | Descrição Detalhada | Origem |
| :--- | :--- | :--- | :--- |
| **RF-01** | **Cadastro de Endpoint de Webhook (`POST`)** | O sistema deve permitir cadastrar um endpoint de webhook informando `customerId` (no body), `url` (obrigatoriamente `https://`) e a lista de status (`OrderStatus`) de interesse, gerando e retornando uma `secret` exclusiva na criação. | `[09:21] Sofia`, `[09:23] Sofia`, `[09:31] Marcos`, `[09:32] Larissa` |
| **RF-02** | **Listagem, Edição e Remoção de Webhooks (`GET`, `PATCH`, `DELETE`)** | O sistema deve permitir listar os webhooks de um cliente (`GET`), editar URL, lista de status assinados ou estado ativo (`PATCH`) e remover um endpoint cadastrado (`DELETE`), autenticado via JWT padrão. | `[09:33] Bruno`, `[09:36] Marcos`, `[09:37] Sofia` |
| **RF-03** | **Filtro de Eventos por Status na Inserção da Outbox** | Ao mudar o status de um pedido em `OrderService.changeStatus`, o sistema deve verificar quais webhooks ativos do cliente assinam o novo status (`toStatus`) e inserir registros na `webhook_outbox` apenas para os endpoints inscritos (se nenhum assinar, nada é inserido). | `[09:33] Marcos`, `[09:34] Bruno`, `[09:34] Diego` |
| **RF-04** | **Registro Atômico de Snapshot do Evento na Mudança de Status** | A inserção do evento na `webhook_outbox` deve ocorrer dentro da mesma transação SQL da mudança de status do pedido, salvando o snapshot JSON já renderizado no momento da transição (`event_id`, `event_type: "order.status_changed"`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`, sem `items`). | `[09:06] Diego`, `[09:40] Bruno`, `[09:43] Diego`, `[09:52] Larissa` |
| **RF-05** | **Disparo Assíncrono com Headers de Segurança e Identificação** | O worker deve disparar requisições HTTP `POST` para a URL do endpoint contendo o snapshot JSON e os headers `Content-Type: application/json`, `X-Event-Id` (UUID), `X-Signature` (HMAC-SHA256 do payload), `X-Timestamp` e `X-Webhook-Id`. | `[09:20] Sofia`, `[09:25] Diego`, `[09:44] Diego`, `[09:44] Sofia` |
| **RF-06** | **Rotação de Secret com Grace Period de 24 Horas** | O cliente deve poder solicitar uma nova `secret` para seu endpoint via API; após a rotação, a `secret` anterior deve permanecer válida em paralelo por **24 horas** para permitir migração sem perda de eventos, expirando após esse período. | `[09:21] Sofia`, `[09:22] Sofia` |
| **RF-07** | **Política de Retry com Backoff Exponencial e Envio para DLQ** | Quando um disparo falhar (resposta não-2xx ou timeout de 10s), o sistema deve retentar até **5 vezes** com intervalos de `1m`, `5m`, `30m`, `2h` e `12h`; persistindo a falha após a 5ª tentativa, o evento deve ser movido para a tabela `webhook_dead_letter` com payload, motivo da falha e timestamp. | `[09:15] Diego`, `[09:17] Diego`, `[09:18] Diego`, `[09:42] Diego` |
| **RF-08** | **Consulta do Histórico das Últimas 100 Entregas (`GET /webhooks/:id/deliveries`)** | O cliente deve poder consultar as últimas 100 tentativas de entrega de um endpoint de webhook, visualizando sucesso/falha, payload enviado, resposta recebida (`statusCode` / `responseBody`) e tempo de resposta (`durationMs`). | `[09:34] Marcos` |
| **RF-09** | **Replay Manual de DLQ Restrito a `ADMIN` com Auditoria** | O sistema deve disponibilizar o endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` restrito exclusivamente a usuários com role `ADMIN` no JWT (`requireRole('ADMIN')`), recolocando o evento na `webhook_outbox` como pendente e registrando em log de auditoria quem executou a ação. | `[09:18] Diego`, `[09:35] Diego`, `[09:36] Sofia`, `[09:36] Larissa` |

---

## 7. Requisitos Não Funcionais

| ID | Categoria | Requisito Não Funcional | Origem |
| :--- | :--- | :--- | :--- |
| **RNF-01** | **Latência / Performance** | O intervalo de polling do worker deve ser de **2 segundos**, garantindo latência de entrega fim-a-fim **abaixo de 10 segundos** em condições normais de operação. | `[09:02] Marcos`, `[09:09] Diego`, `[09:10] Larissa` |
| **RNF-02** | **Isolamento de Processo** | O worker deve rodar como um processo Node.js separado (`src/worker.ts` / `npm run worker`) com instância própria de `PrismaClient`, evitando que reinicializações da API ou lentidões de webhooks afetem um ao outro. | `[09:11] Diego`, `[09:11] Larissa`, `[09:30] Bruno` |
| **RNF-03** | **Segurança de Transporte e Tamanho** | Apenas URLs com protocolo **HTTPS (TLS)** são aceitas (validadas via Zod); o tamanho máximo do payload é de **64KB** (retornando erro caso ultrapasse). | `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa` |
| **RNF-04** | **Timeout de Rede Externa** | Toda requisição HTTP disparada pelo worker deve ter timeout máximo de **10 segundos**, sendo tratada como falha elegível para retry caso exceda esse limite. | `[09:42] Sofia`, `[09:42] Diego` |
| **RNF-05** | **Semântica de Entrega e Ordenação** | Garantia de entrega **at-least-once** com deduplicação no cliente via `X-Event-Id` (UUID) e ordenação implícita por `created_at` enquanto operar em modo *single-worker*. | `[09:12] Diego`, `[09:13] Larissa`, `[09:24] Diego`, `[09:25] Diego` |
| **RNF-06** | **Padronização Arquitetural** | Reuso integral de `AppError` com códigos prefixados por `WEBHOOK_`, logger estruturado Pino (`src/shared/logger/index.ts`), `errorMiddleware` e IDs em formato UUID. | `[09:28] Bruno`, `[09:29] Larissa`, `[09:30] Larissa`, `[09:51] Larissa` |

---

## 8. Decisões e Trade-offs Principais

1. **Outbox no MySQL vs. Chamada Síncrona ou Redis Streams ([ADR-001](./adrs/ADR-001-padrao-outbox-no-mysql.md)):** Escolhemos gravar na tabela `webhook_outbox` dentro da mesma transação SQL de `OrderService.changeStatus` para garantir atomicidade perfeita sem precisar provisionar e manter um Redis Cluster para uma equipe enxuta (`[09:06] Diego`, `[09:07] Diego`).
2. **Polling de 2s em Processo Separado vs. Triggers MySQL ou In-Process ([ADR-002](./adrs/ADR-002-worker-em-processo-separado-com-polling.md)):** Como o MySQL não possui `LISTEN/NOTIFY` nativo para processos externos, adotamos polling indexado a cada 2s em um processo separado (`src/worker.ts`), atendendo ao SLA de `< 10s` com isolamento operacional (`[09:09] Diego`, `[09:11] Diego`).
3. **5 Tentativas (~15h) + Tabela DLQ Separada vs. 3 Tentativas ou Retry Infinito ([ADR-003](./adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md)):** 3 tentativas (~30m) falhariam em manutenções planejadas de 2h dos clientes, e retry infinito acumularia lixo eterno; 5 tentativas (`1m/5m/30m/2h/12h`) com DLQ dedicada equilibram resiliência e higiene da outbox (`[09:15]–[09:18] Diego`).
4. **Secret por Endpoint com Grace Period de 24h vs. Secret Global ([ADR-004](./adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md)):** Isola o impacto de vazamentos a um único endpoint e permite rotação sem queda na integração do cliente (`[09:21] Sofia`).
5. **At-Least-Once com `X-Event-Id` vs. Exactly-Once ([ADR-005](./adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)):** Segue o padrão da indústria (Stripe/GitHub), transferindo a deduplicação por UUID ao cliente (documentada no portal) em troca de simplicidade e alta disponibilidade (`[09:25] Diego`, `[09:26] Marcos`).

---

## 9. Dependências

- **Módulo de Pedidos (`src/modules/orders/order.service.ts`):** Ponto de extensão transacional no método `changeStatus` (`[09:40] Bruno`).
- **Infraestrutura de Autenticação e Erros (`src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/shared/errors/app-error.ts`):** Reaproveitamento de `authenticate`, `requireRole('ADMIN')` e hierarquia `AppError` (`[09:28] Bruno`, `[09:36] Larissa`).
- **Revisão de Segurança Obrigatória:** Alocação de **pelo menos 2 dias úteis da engenheira de segurança Sofia** na 3ª sprint para auditoria da implementação de HMAC-SHA256 e geração/rotação de secrets antes do deploy em produção (`[09:46] Sofia`, `[09:47] Larissa`).
- **Documentação no Portal do Desenvolvedor:** Atualização do guia de integração para clientes B2B pelo PM Marcos destacando a deduplicação via `X-Event-Id` e o consumo dos endpoints de API (`[09:26] Marcos`, `[09:40] Marcos`).

---

## 10. Riscos e Mitigação

| ID | Risco Identificado | Probabilidade | Impacto | Plano de Mitigação | Fonte |
| :--- | :--- | :---: | :---: | :--- | :--- |
| **RISK-01** | **Indisponibilidade prolongada ou manutenção planejada nos servidores dos clientes B2B** causando perda de eventos ou bloqueio de processamento | **Alta** | **Alto** | Processamento assíncrono em worker separado com timeout de 10s por chamada, política de 5 retries com backoff exponencial cobrindo até ~15 horas (`1m`, `5m`, `30m`, `2h`, `12h`) e persistência em `webhook_dead_letter` com replay manual via endpoint `ADMIN`. | `[09:04] Bruno`, `[09:16] Diego`, `[09:17] Diego`, `[09:18] Diego`, `[09:42] Diego` |
| **RISK-02** | **Vazamento de `secret` de webhook em logs de aplicação do cliente B2B** (incidente já ocorrido no passado) ou falsificação de requisições | **Média** | **Alto** | Uso de `secret` única por endpoint (nunca global), assinatura HMAC-SHA256 com `X-Timestamp`, obrigatoriedade de HTTPS, redação no logger Pino e endpoint de rotação de secret com *grace period* de 24 horas para troca sem indisponibilidade. | `[09:20] Sofia`, `[09:21] Sofia`, `[09:22] Diego`, `[09:23] Sofia`, `[09:44] Diego` |
| **RISK-03** | **Processamento duplicado de eventos no ERP do cliente** devido à semântica *at-least-once* em casos de timeout de rede | **Média** | **Médio** | Inclusão do UUID único e imutável do evento no header `X-Event-Id` e no corpo JSON (`event_id`), combinada com documentação destacada no portal do desenvolvedor orientando a deduplicação pelo cliente. | `[09:24] Diego`, `[09:25] Diego`, `[09:26] Marcos` |
| **RISK-04** | **Sobrecarga de chamadas em clientes com alto volume de transições simultâneas** (ex.: 50 mudanças de status em 1 minuto) | **Baixa** | **Médio** | Monitorar métricas de envio por cliente em produção na fase inicial e implementar *rate limiting* de saída caso o comportamento se confirme problemático. | `[09:38] Diego`, `[09:39] Larissa` |

---

## 11. Critérios de Aceitação de Produto

- [ ] Clientes autenticados conseguem cadastrar (`POST`), listar (`GET`), editar (`PATCH`) e excluir (`DELETE`) webhooks informando `customerId`, `url` HTTPS e lista de status de pedido (`[09:31]–[09:33] Marcos/Larissa/Bruno`).
- [ ] Tentativas de cadastrar URL com `http://` são recusadas com erro de validação `WEBHOOK_INVALID_URL` (`[09:23] Sofia`, `[09:28] Bruno`).
- [ ] Ao alterar o status de um pedido em `PATCH /api/v1/orders/:id/status`, um evento é gravado atomicamente na `webhook_outbox` apenas para os webhooks ativos daquele cliente que assinem o novo status (`[09:34] Bruno`, `[09:40] Bruno`).
- [ ] O worker separado dispara a notificação em até 2 segundos (cumprindo a meta `< 10s`) com payload enxuto (sem `items`) e headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id` (`[09:02] Marcos`, `[09:09] Diego`, `[09:43]–[09:44] Diego/Sofia`).
- [ ] A rotação de secret devolve uma nova chave e mantém a anterior funcional por 24 horas (`[09:21] Sofia`).
- [ ] O cliente consegue visualizar as últimas 100 entregas (sucesso/falha, payload, response, tempo de resposta) em `GET /api/v1/webhooks/:id/deliveries` (`[09:34] Marcos`).
- [ ] Eventos que falham nas 5 tentativas (`1m`, `5m`, `30m`, `2h`, `12h`) vão para a DLQ e só podem sofrer replay por usuários com role `ADMIN`, gerando registro de auditoria (`[09:17]–[09:18] Diego`, `[09:36] Sofia`).

---

## 12. Estratégia de Testes e Validação

1. **Testes Unitários (Vitest):**
   - Geração e verificação de assinatura HMAC-SHA256, incluindo validação durante e após a janela de *grace period* de 24 horas (`[09:20]–[09:21] Sofia`).
   - Cálculo dos intervalos de backoff exponencial (`1m`, `5m`, `30m`, `2h`, `12h`) e validação do limite de payload de `64KB` (`[09:17] Diego`, `[09:24] Diego`).
   - Schemas Zod (`webhook.schemas.ts`) rejeitando URLs `http://` e validando `OrderStatus` (`[09:23] Sofia`).
2. **Testes de Integração e Ponta a Ponta (Vitest + Supertest + MySQL):**
   - **Atomicidade Outbox:** Verificar que `OrderService.changeStatus` grava na `webhook_outbox` na mesma transação e que uma falha forçada na outbox reverte a mudança de status do pedido (`[09:40] Bruno`, `[09:46] Larissa`).
   - **Filtragem na Inserção:** Confirmar que mudanças para status não assinados pelo webhook do cliente não geram linhas na `webhook_outbox` (`[09:34] Bruno`).
   - **Ciclo Completo do Worker e DLQ:** Simular endpoint receptor retornando `200 OK`, `500 Internal Server Error` e timeout (`> 10s`), validando o registro em `webhook_deliveries`, a transição para `webhook_dead_letter` na 5ª falha e o replay via `POST /api/v1/admin/webhooks/dead-letter/:id/replay` bloqueando role `OPERATOR` (`403 Forbidden`) e aceitando `ADMIN` (`200 OK`) (`[09:36] Sofia`, `[09:46] Larissa`).
3. **Revisão de Segurança Pré-Deploy:**
   - Auditoria dedicada de 2 dias úteis conduzida por Sofia sobre o código de assinatura HMAC, geração de entropia das secrets e controle de acesso antes da liberação em produção (`[09:46] Sofia`, `[09:48] Sofia`).
