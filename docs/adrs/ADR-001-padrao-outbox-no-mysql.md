# ADR-001: Padrão Transactional Outbox no MySQL para Eventos de Mudança de Status de Pedidos

- **Status:** Aceito
- **Data:** 2026-10-24
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior - Plataforma), Bruno (Engenheiro Pleno - Pedidos), Marcos (PM), Sofia (Engenheira de Segurança)
- **Rastreabilidade:** `TRANSCRICAO.md` (`[09:03] Larissa`, `[09:04] Bruno`, `[09:06] Diego`, `[09:07] Larissa`, `[09:07] Diego`, `[09:08] Diego`, `[09:08] Larissa`, `[09:34] Bruno`, `[09:40] Bruno`, `[09:41] Diego`, `[09:51] Larissa`, `[09:52] Larissa`, `[09:52] Diego`); `CODIGO` (`src/modules/orders/order.service.ts`, `prisma/schema.prisma`)

---

## Contexto

O Order Management System (OMS) precisa notificar clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) sempre que o status de um pedido sofrer alteração (`[09:00] Marcos`). Atualmente, a transição de estado ocorre no método `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que abre uma transação interativa do Prisma (`this.prisma.$transaction`) para:

1. Validar a transição de estado (`canTransition`) em `src/modules/orders/order.status.ts`;
2. Debitar ou estornar `stockQuantity` na tabela `products` (`debitStock` / `replenishStock`);
3. Atualizar o campo `status` na tabela `orders`;
4. Inserir o registro de auditoria na tabela `order_status_history`.

O problema central consiste em garantir atomicidade e consistência entre a mudança de status do pedido no banco relacional MySQL e o registro do evento de notificação para envio HTTP externo (`[09:03] Larissa`, `[09:06] Diego`). Se o status do pedido for alterado no banco, o evento de webhook jamais pode ser perdido; inversamente, se a transação de mudança de status sofrer rollback, nenhum evento inválido pode ser emitido para os clientes (`[09:06] Diego`, `[09:40] Bruno`).

## Decisão

Adotar o padrão **Transactional Outbox no MySQL existente** (`[09:06] Diego`, `[09:08] Larissa`), persistindo os eventos de webhook na tabela `webhook_outbox` dentro da mesma transação SQL (`prisma.$transaction`) executada pelo método `OrderService.changeStatus` (`[09:40] Bruno`, `[09:41] Diego`).

As regras estruturais desta decisão são:

1. **Atomicidade Transacional via `TxClient`:** O método `OrderService.changeStatus` invocará uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` passando o cliente transacional `tx` (`Prisma.TransactionClient`) antes do commit (`[09:41] Bruno`, `[09:41] Diego`). Caso a inserção na `webhook_outbox` falhe, toda a mudança de status sofre rollback (`[09:40] Bruno`).
2. **Filtragem na Inserção:** Antes de inserir na `webhook_outbox`, a função verifica se o `customerId` do pedido possui endpoints de webhook ativos inscritos para o `toStatus` correspondente. Caso nenhum webhook do cliente escute aquele status, nenhuma linha é inserida na outbox (`[09:34] Bruno`, `[09:34] Diego`).
3. **Snapshot do Payload Renderizado na Inserção:** A linha inserida em `webhook_outbox` armazena o payload JSON já renderizado no momento exato da transição (incluindo `event_id` UUID, `event_type: "order.status_changed"`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`, sem a lista de `items`), garantindo que mutações posteriores no pedido não corrompam a fotografia histórica do evento (`[09:43] Diego`, `[09:51] Bruno`, `[09:52] Larissa`, `[09:52] Diego`).
4. **Identificadores e Índices:** O registro na `webhook_outbox` utiliza `UUID` (`@db.Char(36)`) como chave primária seguindo o padrão de `prisma/schema.prisma` (`[09:51] Larissa`) e mantém índices compostos/simples em `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`) e `created_at` para leitura eficiente em pequenos lotes pelo worker (`[09:08] Diego`).

## Alternativas Consideradas

### Alternativa 1: Disparo HTTP Síncrono dentro de `OrderService.changeStatus`
- **Descrição:** Realizar a chamada HTTP `POST` para a URL do cliente diretamente dentro do fluxo de `changeStatus` na API (`[09:03] Larissa`).
- **Motivo do Descarte / Trade-off:** A transação de mudança de status já é pesada (atualiza `orders`, insere em `order_status_history` e decrementa/incrementa `stock_quantity` em `products`). Incluir uma chamada de rede externa no meio da transação bloquearia conexões/locks do banco e faria com que qualquer endpoint de cliente lento travasse mudanças de status de outros pedidos (`[09:04] Bruno`). Além disso, se o endpoint do cliente estivesse fora do ar, o sistema enfrentaria o dilema inaceitável entre dar rollback na mudança de status do pedido ou perder a notificação (`[09:04] Bruno`, `[09:06] Diego`).

### Alternativa 2: Broker Externo Dedicado (Redis Streams / Redis Cluster)
- **Descrição:** Publicar os eventos de mudança de status em uma infraestrutura de mensageria em memória/stream como Redis Streams (`[09:07] Larissa`).
- **Motivo do Descarte / Trade-off:** A equipe é enxuta e a aplicação hoje opera exclusivamente com Node.js e MySQL (`docker-compose.yml`, `prisma/schema.prisma`). Provisionar, operar e manter alta disponibilidade de um Redis Cluster adicionaria complexidade operacional e custo de infraestrutura desnecessários (*overengineering*) para um requisito que o MySQL existente atende com garantia transacional nativa (`[09:07] Diego`). Ademais, publicar no Redis fora do MySQL introduziria o problema de *dual-write* (banco commita, mas publicação no Redis falha).

### Alternativa 3: Armazenar Apenas `order_id` na Outbox e Renderizar o Payload no Envio
- **Descrição:** Gravar somente a referência (`order_id`) na `webhook_outbox` e montar o JSON do evento apenas quando o worker for disparar a chamada HTTP (`[09:51] Bruno`).
- **Motivo do Descarte / Trade-off:** Se o pedido sofresse novas alterações de status ou dados segundos após a transição original, o worker leria o estado mais recente do pedido no momento do envio, enviando um payload inconsistente com o evento histórico que gerou a entrada na outbox (`[09:52] Larissa`, `[09:52] Diego`).

## Consequências

### Positivas
- **Consistência Transacional Absoluta (Zero Perda / Zero Fantasma):** Como `orders`, `order_status_history`, `products` e `webhook_outbox` compartilham a mesma transação ACID no MySQL, o evento só existe se a transição de status commitar, e desaparece se houver rollback (`[09:06] Diego`, `[09:40] Bruno`).
- **Desacoplamento de Latência na API:** O endpoint `PATCH /api/v1/orders/:id/status` responde imediatamente após o commit local no MySQL, sem depender da disponibilidade ou latência de rede dos servidores dos clientes B2B (`[09:04] Bruno`).
- **Simplicidade Operacional:** Reutiliza 100% da infraestrutura de banco de dados MySQL e Prisma ORM já presentes no projeto (`[09:07] Diego`).
- **Economia de Armazenamento e Imutabilidade:** Filtrar eventos não assinados antes da inserção evita linhas inúteis (`[09:34] Bruno`), e o snapshot JSON preserva o estado exato do momento da transição (`[09:52] Larissa`).

### Negativas e Trade-offs Explícitos
- **Crescimento da Tabela `webhook_outbox` no MySQL:** Como todo evento assinado gera um registro com snapshot JSON no banco relacional, o volume de linhas crescerá continuamente. **Trade-off aceito:** Mitigado por índices em `status` e `created_at`, filtragem prévia na inserção, limite de 64KB por payload (`[09:24] Diego`) e uma política futura (fora do escopo desta fase) de arquivamento/expurgo de registros entregues após 30 dias (`[09:08] Diego`).
- **Acoplamento Transacional na Escrita de `changeStatus`:** Um erro de banco ao inserir na `webhook_outbox` aborta a mudança de status do pedido. **Trade-off aceito:** Comportamento intencional exigido pela equipe para garantir que nenhuma transição de pedido ocorra silenciosamente sem sua respectiva notificação (`[09:40] Bruno`, `[09:41] Diego`).
