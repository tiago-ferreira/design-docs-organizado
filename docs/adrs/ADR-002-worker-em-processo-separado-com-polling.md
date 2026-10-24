# ADR-002: Worker em Processo Separado com Loop de Polling a Cada 2 Segundos

- **Status:** Aceito
- **Data:** 2026-10-24
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior - Plataforma), Bruno (Engenheiro Pleno - Pedidos), Marcos (PM)
- **Rastreabilidade:** `TRANSCRICAO.md` (`[09:01] Bruno`, `[09:02] Marcos`, `[09:08] Diego`, `[09:09] Diego`, `[09:09] Bruno`, `[09:10] Marcos`, `[09:10] Larissa`, `[09:11] Diego`, `[09:11] Larissa`, `[09:11] Bruno`, `[09:28] Bruno`, `[09:29] Diego`, `[09:30] Bruno`); `CODIGO` (`src/server.ts`, `src/config/database.ts`, `package.json`)

---

## Contexto

Com a adoção do padrão Transactional Outbox na tabela `webhook_outbox` do MySQL ([ADR-001](./ADR-001-padrao-outbox-no-mysql.md)), o sistema precisa de um mecanismo consumidor assíncrono para ler os eventos com status pendente, realizar as chamadas HTTP `POST` para os endpoints configurados pelos clientes B2B e atualizar o estado de entrega (`[09:06] Diego`, `[09:08] Larissa`).

Os clientes B2B definiram que qualquer entrega ocorrida em **menos de 10 segundos** após a mudança de status do pedido atende plenamente à expectativa de "tempo real" (`[09:01] Bruno`, `[09:02] Marcos`). Além disso, a arquitetura precisa definir o modelo de isolamento de processo entre o servidor HTTP da API (`src/server.ts`) e o consumidor da outbox (`[09:11] Diego`).

## Decisão

Executar o consumidor da outbox como um **worker em processo Node.js separado** da instância da API HTTP, operando em **loop de polling a cada 2 segundos** sobre a tabela `webhook_outbox` (`[09:09] Diego`, `[09:10] Larissa`, `[09:11] Diego`).

Os detalhes arquiteturais da decisão são:

1. **Entry-point e Processo Dedicado:** Criar um novo ponto de entrada `src/worker.ts` (análogo ao `src/server.ts` existente) acionado via script `npm run worker` no `package.json`, delegando a lógica de processamento para `src/modules/webhooks/webhook.worker.ts` (`[09:11] Larissa`, `[09:28] Bruno`).
2. **Instância Própria de `PrismaClient`:** Como o worker roda em outro processo do sistema operacional, ele instanciará seu próprio `PrismaClient` (reutilizando a fábrica `createPrismaClient()` de `src/config/database.ts` e a mesma variável `DATABASE_URL` apontando para o mesmo banco MySQL) (`[09:11] Bruno`, `[09:11] Diego`, `[09:29] Diego`, `[09:30] Bruno`).
3. **Polling de 2 Segundos em Lotes Pequenos:** A cada 2 segundos, o worker consulta os eventos `PENDING` (ou elegíveis para retry) mais antigos ordenados por `created_at` (aproveitando o índice composto em `status` e `created_at`), processa-os em batch pequeno e atualiza o status para `DELIVERED` ou agenda retentativa (`[09:08] Diego`, `[09:09] Diego`).
4. **Topologia Single-Worker Inicial:** Na fase atual, haverá uma única instância ativa do worker, garantindo ordenação implícita de entrega por `created_at` e evitando disputas de concorrência entre múltiplos consumidores (`[09:12] Diego`, `[09:13] Larissa`).

## Alternativas Consideradas

### Alternativa 1: Executar o Worker Dentro do Mesmo Processo da API (`src/server.ts`)
- **Descrição:** Iniciar um `setInterval` ou loop assíncrono dentro do próprio processo Express inicializado por `src/server.ts` (`[09:11] Diego`).
- **Motivo do Descarte / Trade-off:** Acopla o ciclo de vida do processador de webhooks ao servidor web da API. Se a instância da API reiniciar devido a deploy, pico de memória ou falha em rota HTTP, o processamento da outbox em andamento é interrompido abruptamente (`[09:11] Diego`). Além disso, picos de retentativas HTTP ou geração de assinaturas HMAC competiriam diretamente pelo mesmo Event Loop das requisições REST de pedidos.

### Alternativa 2: Reatividade via Triggers Nativos do Banco de Dados MySQL
- **Descrição:** Utilizar triggers no MySQL após `INSERT` na `webhook_outbox` para acionar imediatamente o worker sem necessidade de polling periódico (`[09:09] Bruno`).
- **Motivo do Descarte / Trade-off:** Diferentemente do PostgreSQL (que possui `LISTEN / NOTIFY` nativo), o MySQL não dispõe de mecanismo nativo para notificar processos externos a partir de uma trigger — triggers no MySQL apenas executam instruções SQL internas (`[09:09] Diego`). Contornar essa limitação exigiria improvisos frágeis e anti-padrões (como escrever em arquivos locais via UDF ou disparar chamadas de rede a partir do banco), enquanto o polling de 2 segundos cumpre com folga o SLA de `< 10 segundos` (`[09:09] Diego`, `[09:10] Marcos`).

## Consequências

### Positivas
- **Atendimento Folgado ao SLA de Negócio:** O intervalo de polling de 2 segundos garante latência mínima de detecção de no máximo 2 segundos no pior caso, mantendo a entrega total muito abaixo do limite de 10 segundos acordado com os clientes B2B (`[09:02] Marcos`, `[09:09] Diego`, `[09:10] Larissa`).
- **Isolamento de Falhas e Deploys Independentes:** Reinicializações da API HTTP não interrompem o envio de webhooks em andamento no worker e vice-versa (`[09:11] Diego`).
- **Mesma Stack Tecnológica:** Compartilha o mesmo repositório TypeScript, validações, logger Pino e schema Prisma (`src/config/database.ts`), sem introduzir novas linguagens ou runtimes (`[09:11] Diego`, `[09:30] Bruno`).

### Negativas e Trade-offs Explícitos
- **Latência Base de até 2 Segundos e Queries Periódicas no MySQL:** Mesmo quando não há pedidos mudando de status, o worker executa uma query leve a cada 2 segundos, e eventos recém-inseridos podem aguardar até 2s antes do início do disparo (`[09:10] Larissa`). **Trade-off aceito:** A consulta utiliza índice dedicado (`status, created_at`), tornando o custo no MySQL desprezível, e a latência de 2s foi validada pelo PM como ideal (`[09:08] Diego`, `[09:10] Marcos`).
- **Consumo de Conexões Adicionais no MySQL:** Como `PrismaClient` é instanciado por processo (`[09:30] Bruno`), o processo `src/worker.ts` mantém seu próprio pool de conexões separado do processo `src/server.ts`. **Trade-off aceito:** O número de processos é reduzido (single-worker) e perfeitamente suportado pelo MySQL.
