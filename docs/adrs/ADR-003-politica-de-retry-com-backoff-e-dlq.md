# ADR-003: Política de Retry com Backoff Exponencial (5 Tentativas) e Dead Letter Queue (DLQ) Dedicada

- **Status:** Aceito
- **Data:** 2026-10-24
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior - Plataforma), Bruno (Engenheiro Pleno - Pedidos), Marcos (PM), Sofia (Engenheira de Segurança)
- **Rastreabilidade:** `TRANSCRICAO.md` (`[09:14] Larissa`, `[09:15] Diego`, `[09:15] Bruno`, `[09:16] Bruno`, `[09:16] Diego`, `[09:16] Larissa`, `[09:17] Diego`, `[09:17] Marcos`, `[09:17] Larissa`, `[09:18] Diego`, `[09:18] Bruno`, `[09:19] Larissa`, `[09:35] Diego`, `[09:35] Larissa`, `[09:36] Sofia`, `[09:36] Larissa`, `[09:42] Sofia`, `[09:42] Diego`); `CODIGO` (`src/middlewares/auth.middleware.ts`, `src/shared/logger/index.ts`)

---

## Contexto

Os endpoints HTTP configurados pelos clientes B2B operam fora da infraestrutura do OMS e estão sujeitos a falhas transitórias de rede, lentidão de resposta ou janelas de manutenção planejada de até 2 horas (`[09:16] Diego`). Quando o worker de webhooks ([ADR-002](./ADR-002-worker-em-processo-separado-com-polling.md)) realiza o disparo HTTP `POST`, precisamos definir:

1. Em quanto tempo uma requisição sem resposta deve ser abortada por timeout (`[09:42] Sofia`);
2. Quantas vezes e com qual espaçamento temporal o sistema deve retentar entregas que falharam (`[09:14] Larissa`, `[09:15] Bruno`);
3. Como armazenar eventos que esgotaram todas as tentativas e como permitir seu reprocessamento seguro (`[09:17] Larissa`, `[09:18] Bruno`).

## Decisão

Adotar uma política de **até 5 tentativas de retry com backoff exponencial progressivo** (`1m`, `5m`, `30m`, `2h`, `12h`), **timeout HTTP de 10 segundos por tentativa**, persistência de falhas permanentes em uma **tabela dedicada `webhook_dead_letter` (DLQ)** e **replay manual restrito ao perfil `ADMIN`** (`[09:17] Larissa`, `[09:18] Diego`, `[09:36] Larissa`, `[09:42] Diego`).

Os parâmetros normativos desta decisão são:

1. **Timeout HTTP de 10 Segundos:** Qualquer chamada HTTP `POST` do worker cujo servidor de destino não responda dentro de 10 segundos é abortada, registrada como tentativa falha e agendada para retry (`[09:42] Sofia`, `[09:42] Diego`).
2. **Teto de 5 Tentativas com Backoff Progressivo (~15 horas de janela):** Após a falha inicial no envio imediato, o evento passa por até 5 retentativas com os intervalos:
   - Tentativa 1: **+1 minuto**
   - Tentativa 2: **+5 minutos**
   - Tentativa 3: **+30 minutos**
   - Tentativa 4: **+2 horas** (120 minutos)
   - Tentativa 5: **+12 horas** (720 minutos)
   - Janela total acumulada: **14 horas e 36 minutos** (~15 horas) entre a primeira falha e a última tentativa (`[09:17] Diego`, `[09:17] Marcos`, `[09:17] Larissa`).
3. **Tabela Separada `webhook_dead_letter`:** Esgotadas as 5 tentativas sem sucesso (respostas HTTP não-2xx ou timeout), o evento é considerado falha permanente e gravado na tabela dedicada `webhook_dead_letter` contendo o payload original, o motivo da falha/última resposta e o timestamp (`[09:15] Diego`, `[09:18] Diego`).
4. **Replay Manual Autenticado com Role `ADMIN` e Auditoria:** O reprocessamento de eventos na DLQ é feito manualmente via endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na `webhook_outbox` como pendente (`[09:18] Diego`, `[09:35] Diego`). Este endpoint exige obrigatoriamente a role `ADMIN` no JWT (reutilizando `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts`) e registra em log estruturado Pino (`src/shared/logger/index.ts`) o identificador do administrador que acionou o replay para fins de auditoria (`[09:35] Larissa`, `[09:36] Sofia`, `[09:36] Larissa`).

## Alternativas Consideradas

### Alternativa 1: Limite Curto de 3 Tentativas de Retry
- **Descrição:** Realizar apenas 3 tentativas rápidas (cobrindo cerca de 30 minutos) antes de descartar ou mover para DLQ (`[09:16] Bruno`).
- **Motivo do Descarte / Trade-off:** Muito agressivo para o perfil B2B da plataforma. A empresa já teve clientes com janelas de manutenção planejada de 2 horas pela manhã; com apenas 3 tentativas em 30 minutos, todos os eventos dessa janela cairiam prematuramente na DLQ exigindo intervenção manual (`[09:16] Diego`).

### Alternativa 2: Retry Indefinido com Backoff
- **Descrição:** Continuar retentando indefinidamente sem teto máximo de tentativas até que o cliente volte a responder (`[09:15] Diego`).
- **Motivo do Descarte / Trade-off:** Se um cliente desativar seu servidor ou mudar de endereço sem atualizar o cadastro, os eventos ficariam pendurados eternamente na fila de processamento, poluindo a `webhook_outbox` e consumindo ciclos do worker indefinidamente (`[09:15] Diego`).

### Alternativa 3: Marcar `status = 'FAILED'` na Própria Tabela `webhook_outbox` (Sem Tabela DLQ Separada)
- **Descrição:** Manter os eventos esgotados dentro da mesma tabela `webhook_outbox` apenas alterando a coluna de status para `FAILED` (`[09:17] Larissa`).
- **Motivo do Descarte / Trade-off:** Misturar registros operacionais de trânsito rápido com falhas permanentes degrada a limpeza e previsibilidade da `webhook_outbox` principal. Uma tabela `webhook_dead_letter` dedicada isola as evidências de falha (payload, motivo do erro e timestamps) para investigação de suporte e replay administrativo (`[09:18] Diego`, `[09:18] Bruno`).

## Consequências

### Positivas
- **Resiliência a Manutenções Planejadas:** A janela de ~15 horas absorve tranquilamente indisponibilidades de 2 a 12 horas dos sistemas ERP/WMS dos clientes B2B sem perda de eventos nem acionamento manual da equipe (`[09:16] Diego`, `[09:17] Marcos`).
- **Isolamento Operacional da Outbox:** Eventos problemáticos saem do fluxo quente da `webhook_outbox` após a 5ª falha e vão para `webhook_dead_letter`, mantendo as queries de polling do worker rápidas (`[09:18] Diego`).
- **Governança e Segurança no Reprocessamento:** O uso de `requireRole('ADMIN')` com log de auditoria impede que operadores comuns manipulem filas de entrega indevidamente (`[09:36] Sofia`, `[09:36] Larissa`).

### Negativas e Trade-offs Explícitos
- **Atraso na Detecção de Falha Definitiva (~15h):** Um endpoint configurado com URL quebrada só chegará à DLQ após ~15 horas. **Trade-off aceito:** O cliente pode consultar falhas parciais a qualquer momento via histórico de entregas (`GET /api/v1/webhooks/:id/deliveries`, `[09:34] Marcos`), enquanto alertas proativos por e-mail foram adiados para uma fase futura (`[09:37] Larissa`).
- **Ocupação do Worker por até 10s em Timeouts:** Um endpoint que aceita a conexão TCP mas não responde pode segurar a requisição por até 10 segundos (`[09:42] Diego`). **Trade-off aceito:** O limite rígido de 10s impede bloqueios indefinidos e o envio do lote pode ser paralelizado com `Promise.allSettled`.
