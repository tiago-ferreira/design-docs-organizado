# RFC-001: Arquitetura do Sistema de Webhooks de Notificação de Pedidos (Outbound)

| Metadado | Valor |
| :--- | :--- |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Aceito / Em Revisão Final para Implementação |
| **Data** | 2026-10-24 |
| **Revisores** | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Engenheiro Pleno - Pedidos), Diego (Engenheiro Sênior - Plataforma), Sofia (Engenheira de Segurança) |
| **Prazo Estimado** | 3 Sprints (incluindo 2 dias úteis de revisão de segurança da Sofia antes do deploy) |

---

## 1. Resumo Executivo (TL;DR)

Este RFC propõe a arquitetura de um sistema de **Outbound Webhooks** para notificar clientes B2B em quase tempo real (`< 10 segundos`) sobre mudanças de status de pedidos no Order Management System (OMS) (`[09:00] Marcos`, `[09:02] Marcos`, `[09:03] Sofia`).

A solução adota o padrão **Transactional Outbox no MySQL existente** combinado com um **worker em processo separado operando em loop de polling a cada 2 segundos** (`[09:06] Diego`, `[09:09] Diego`, `[09:11] Diego`). O registro do evento na tabela `webhook_outbox` ocorre atomicamente dentro da mesma transação SQL de `OrderService.changeStatus`, persistindo o snapshot JSON do evento filtrado pelos status de interesse do cliente (`[09:34] Bruno`, `[09:40] Bruno`, `[09:52] Larissa`). Falhas de entrega são tratadas com **backoff exponencial em 5 tentativas** (`1m`, `5m`, `30m`, `2h`, `12h`, janela de ~15h) e movimentação posterior para uma tabela dedicada de **Dead Letter Queue (`webhook_dead_letter`)** com endpoint de replay manual restrito a `ADMIN` (`[09:17] Larissa`, `[09:18] Diego`, `[09:36] Sofia`). A autenticidade é assegurada por **assinatura HMAC-SHA256 com secret exclusiva por endpoint**, suporte a rotação com *grace period* de 24 horas e entrega **at-least-once** com deduplicação via header `X-Event-Id` (`[09:22] Sofia`, `[09:26] Larissa`).

---

## 2. Contexto e Problema

Atualmente, o OMS gerencia o ciclo de vida de pedidos (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`) com controle transacional de estoque e histórico em `order_status_history` (`src/modules/orders/order.service.ts`), mas **não possui nenhum mecanismo de notificação ativa externa** (`[09:00] Marcos`).

Três clientes B2B estratégicos — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — formalizaram a necessidade de receber notificações automáticas sempre que o status de seus pedidos mudar (`[09:00] Marcos`). Hoje, esses clientes realizam *polling* constante no endpoint `GET /api/v1/orders`, tornando a integração lenta e custosa. A Atlas Comercial sinalizou risco concreto de *churn* (migração para concorrente) caso a funcionalidade não seja entregue até o final do trimestre (`[09:00] Marcos`, `[09:45] Marcos`).

Para os clientes, qualquer entrega realizada em **menos de 10 segundos** após a mudança de status é considerada "tempo real" (`[09:02] Marcos`). O escopo é estritamente **outbound** (do nosso OMS para os endpoints HTTPS dos clientes) (`[09:02] Marcos`, `[09:03] Sofia`).

---

## 3. Proposta Técnica (Visão Geral da Arquitetura)

A solução foi desenhada para maximizar a confiabilidade transacional e reutilizar a stack Node.js + TypeScript + Express + Prisma + MySQL + Pino + Zod já em produção (`[09:07] Diego`, `[09:30] Larissa`). Os detalhes de implementação, schemas e payloads estão especificados no [FDD](./FDD.md).

```mermaid
flowchart LR
    API["API OMS (Express)\nOrderService.changeStatus"] -->|Transação SQL Atômica| DB[("MySQL\norders + order_status_history\n+ webhook_outbox")]
    Worker["Worker Separado (src/worker.ts)\nPolling a cada 2s"] -->|Lê pendentes em batch| DB
    Worker -->|POST HTTPS (timeout 10s)\nHMAC-SHA256 + X-Event-Id| Client["Endpoint HTTPS do Cliente B2B"]
    Worker -->|Falha após 5 retries (~15h)| DLQ[("MySQL\nwebhook_dead_letter")]
    Admin["Admin (Role ADMIN)\nPOST /admin/.../replay"] -->|Recoloca como PENDING| DLQ
```

### 3.1. Componentes Principais

1. **Publicação Transacional na Outbox (`OrderService.changeStatus`):**
   - Dentro da transação `prisma.$transaction` que atualiza `orders`, `products` e `order_status_history`, o serviço invoca a função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` (`[09:40] Bruno`, `[09:41] Bruno`).
   - Se o cliente possuir endpoints ativos inscritos para o novo status (`toStatus`), um snapshot JSON enxuto do evento (com `event_id` UUID e sem os itens do pedido) é gravado na tabela `webhook_outbox` (`[09:34] Bruno`, `[09:43] Diego`, `[09:52] Larissa`). Se nenhum endpoint escutar aquele status, nada é inserido (`[09:34] Bruno`).
2. **Worker Desacoplado em Polling (`src/worker.ts`):**
   - Executado como processo separado (`npm run worker`) com instância dedicada de `PrismaClient` conectada ao mesmo banco MySQL (`[09:11] Diego`, `[09:11] Larissa`, `[09:30] Bruno`).
   - A cada **2 segundos**, busca em lote pequeno os registros pendentes/elegíveis mais antigos da `webhook_outbox` (usando índices em `status` e `created_at`) e realiza chamadas `POST` HTTPS com timeout de **10 segundos** e limite de payload de **64KB** (`[09:08] Diego`, `[09:09] Diego`, `[09:24] Diego`, `[09:42] Diego`).
3. **Resiliência, Retries e Dead Letter Queue (DLQ):**
   - Em caso de erro HTTP ou timeout de 10s, o worker agenda até **5 tentativas** com backoff exponencial: `1m`, `5m`, `30m`, `2h` e `12h` (~15h no total) (`[09:17] Diego`, `[09:17] Larissa`).
   - Esgotadas as 5 tentativas, o evento é movido para a tabela separada `webhook_dead_letter`, podendo ser reprocessado manualmente por usuários com role `ADMIN` via endpoint de replay auditado em log (`[09:18] Diego`, `[09:36] Sofia`).
4. **Segurança e Semântica de Entrega:**
   - Endpoints devem usar obrigatoriamente `https://` (`[09:23] Sofia`).
   - Cada endpoint possui uma `secret` exclusiva gerada pela plataforma, suportando rotação via API com *grace period* de **24 horas** (`[09:21] Sofia`).
   - Os disparos incluem `X-Signature` (HMAC-SHA256 sobre o corpo), `X-Event-Id` (UUID único para deduplicação *at-least-once* pelo cliente), `X-Timestamp` (prevenção de *replay attack*) e `X-Webhook-Id` (`[09:20] Sofia`, `[09:25] Diego`, `[09:44] Diego`, `[09:44] Sofia`).
5. **Gestão e Visibilidade via API REST (`src/modules/webhooks/`):**
   - Endpoints autenticados via JWT para CRUD de webhooks por `customerId`, rotação de secret, consulta das últimas 100 entregas (`GET /api/v1/webhooks/:id/deliveries`) e replay administrativo da DLQ (`[09:31] Marcos`, `[09:32] Larissa`, `[09:33] Bruno`, `[09:34] Marcos`, `[09:35] Diego`).

---

## 4. Alternativas Consideradas e Descartadas

Durante a reunião de arquitetura, as seguintes alternativas foram avaliadas e descartadas:

| Alternativa Descartada | Contexto da Discussão | Trade-off que Motivou o Descarte |
| :--- | :--- | :--- |
| **1. Chamada HTTP síncrona dentro da transação de `OrderService.changeStatus`** | Avaliada como opção inicial para disparar o webhook assim que o status mudasse (`[09:03] Larissa`). | A transação de mudança de status já atualiza `orders`, `order_status_history` e `products`. Uma chamada HTTP externa no meio da transação faria com que qualquer cliente lento bloqueasse locks no banco e travasse mudanças de status de outros pedidos, além de criar o dilema insolúvel de dar rollback no pedido caso o servidor do cliente estivesse offline (`[09:04] Bruno`, `[09:06] Diego`). |
| **2. Infraestrutura externa de mensageria (Redis Streams / Redis Cluster)** | Cogitada como fila dedicada em vez de usar o banco relacional (`[09:07] Larissa`). | Exigiria provisionar, monitorar e manter nova infraestrutura (Redis Cluster) em um time pequeno (*overengineering*), além de perder a garantia transacional ACID nativa obtida ao gravar na `webhook_outbox` dentro do mesmo commit SQL do MySQL (`[09:06] Diego`, `[09:07] Diego`). |
| **3. Reatividade por Triggers no MySQL em vez de Polling** | Questionada para evitar polling periódico de 2s (`[09:09] Bruno`). | Ao contrário do `LISTEN/NOTIFY` do PostgreSQL, triggers no MySQL executam apenas SQL interno e não notificam processos externos nativamente. Improvisar escrita em arquivo ou chamada externa via banco seria frágil, enquanto o polling de 2s atende com folga o requisito de `< 10s` (`[09:09] Diego`, `[09:10] Larissa`). |
| **4. Política de Retry com 3 tentativas ou Retry Indefinido** | Bruno sugeriu 3 tentativas (~30 min); Diego comentou sobre retry infinito (`[09:15] Diego`, `[09:16] Bruno`). | 3 tentativas em 30 minutos descartariam eventos durante manutenções planejadas de 2 horas que clientes da base já realizaram (`[09:16] Diego`). Já o retry indefinido manteria eventos órfãos pendurados eternamente caso um cliente desativasse seu endpoint (`[09:15] Diego`). |
| **5. Semântica de entrega Exactly-Once** | Discutida frente à responsabilidade do cliente em deduplicar eventos (`[09:25] Sofia`, `[09:25] Diego`). | Garantir *exactly-once* exigiria coordenação distribuída complexa entre os dois lados da chamada HTTP. O modelo *at-least-once* com `X-Event-Id` segue o padrão de mercado (Stripe, GitHub) e resolve 99% dos cenários sem complexidade desnecessária (`[09:25] Diego`). |
| **6. Renderização tardia do payload no momento do envio pelo Worker** | Avaliada frente a salvar o snapshot JSON pronto na inserção da outbox (`[09:51] Bruno`). | Se apenas o `order_id` fosse salvo e o pedido mudasse novamente antes do envio, o payload refletiria um estado posterior diferente daquele do momento da transição (`[09:52] Larissa`, `[09:52] Diego`). |

---

## 5. Questões em Aberto e Pontos Adiados

Os seguintes pontos foram identificados na reunião e deliberadamente deixados em aberto para observação em produção ou fases futuras:

1. **Rate Limiting de Saída por Cliente (`[09:38] Diego`, `[09:39] Larissa`):** Caso um cliente tenha 50 pedidos mudando de status em 1 minuto, o worker disparará 50 requisições. Foi decidido **observar o comportamento em produção** e implementar controle de vazão (*rate limiting* de envio) apenas se isso se tornar um problema real para os receptores.
2. **Política Automatizada de Arquivamento/Expurgo da Outbox após 30 Dias (`[09:08] Diego`):** Linhas com status `DELIVERED` na `webhook_outbox` deverão ser arquivadas/expurgadas após ~30 dias, mas a automação desse job ficou explícita como fora do escopo desta entrega inicial.
3. **Notificações Proativas por E-mail em Caso de Falhas Repetidas (`[09:37] Marcos`, `[09:37] Larissa`):** O envio de alerta por e-mail ao cliente após 3 falhas consecutivas foi adiado para uma **próxima fase**, após medição do impacto da versão inicial.
4. **Endurecimento de Permissões (RBAC) no CRUD de Configuração (`[09:36] Marcos`, `[09:37] Sofia`):** Na versão inicial, qualquer usuário autenticado via JWT (`OPERATOR` ou `ADMIN`) poderá operar o CRUD de configuração de webhooks (exceto o replay de DLQ, que já nasce restrito a `ADMIN`). O endurecimento de permissões por role/escopo no CRUD será reavaliado futuramente.
5. **Particionamento por `order_id` / Lock Pessimista para Múltiplos Workers (`[09:12] Diego`, `[09:13] Diego`):** Enquanto o sistema operar com *single-worker*, a ordenação por pedido é implícita via `created_at`. Caso seja necessário escalar horizontalmente para múltiplos workers em paralelo no futuro, será definido particionamento por `order_id` ou lock pessimista.

---

## 6. Impacto e Riscos

| Risco / Impacto | Probabilidade | Impacto | Mitigação Definida |
| :--- | :---: | :---: | :--- |
| **Aumento de escopo da transação em `OrderService.changeStatus`** | Baixa | Alto | A inserção na `webhook_outbox` ocorre na mesma transação local MySQL com filtragem prévia (só insere se houver webhook inscrito no status) e validação de tamanho máximo de 64KB (`[09:24] Diego`, `[09:34] Bruno`, `[09:40] Bruno`). |
| **Lentidão ou indisponibilidade dos endpoints dos clientes B2B** | Alta | Médio | Isolamento em worker separado (`src/worker.ts`), timeout rígido de 10s por chamada HTTP e política de 5 retries ao longo de ~15h seguida de DLQ (`[09:11] Diego`, `[09:17] Diego`, `[09:42] Diego`). |
| **Vazamento de Secret de Webhook em logs do cliente** | Média | Alto | Secrets exclusivas por endpoint (isolando o *blast radius*), redação no logger Pino e endpoint de rotação de secret com *grace period* de 24h (`[09:21] Sofia`, `[09:22] Diego`). |
| **Crescimento de armazenamento nas tabelas de Outbox e Deliveries** | Média | Baixo | Índices compostos em `(status, created_at)`, payload enxuto (sem array de `items`) e plano futuro de retenção de 30 dias (`[09:08] Diego`, `[09:43] Diego`). |

---

## 7. Decisões Relacionadas (ADRs)

Todas as decisões arquiteturais estruturantes desta proposta estão formalizadas nos seguintes registros:

- **[ADR-001: Padrão Transactional Outbox no MySQL](./adrs/ADR-001-padrao-outbox-no-mysql.md)** — Atomicidade com `OrderService.changeStatus`, filtragem na inserção e snapshot JSON.
- **[ADR-002: Worker em Processo Separado com Polling de 2s](./adrs/ADR-002-worker-em-processo-separado-com-polling.md)** — Isolamento em `src/worker.ts`, instância própria de `PrismaClient` e latência `< 10s`.
- **[ADR-003: Política de Retry com Backoff Exponencial e DLQ](./adrs/ADR-003-politica-de-retry-com-backoff-e-dlq.md)** — 5 tentativas (`1m/5m/30m/2h/12h`), timeout de 10s, tabela `webhook_dead_letter` e replay restrito a `ADMIN`.
- **[ADR-004: Autenticação HMAC-SHA256 e Rotação de Secret](./adrs/ADR-004-autenticacao-hmac-sha256-e-rotacao-de-secret.md)** — Secret por endpoint, HTTPS obrigatório, headers de segurança e *grace period* de 24h.
- **[ADR-005: Garantia At-Least-Once com `X-Event-Id`](./adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)** — Idempotência pelo cliente e ordenação implícita sob execução *single-worker*.
- **[ADR-006: Reuso dos Padrões Existentes do Projeto](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)** — Convenções de `src/modules/webhooks/`, `AppError` (`WEBHOOK_*`), Pino, Zod e Prisma UUID.
