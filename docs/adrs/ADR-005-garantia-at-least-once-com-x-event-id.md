# ADR-005: Semântica de Entrega At-Least-Once com `X-Event-Id` e Ordenação Implícita Single-Worker

- **Status:** Aceito
- **Data:** 2026-10-24
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior - Plataforma), Bruno (Engenheiro Pleno - Pedidos), Marcos (PM), Sofia (Engenheira de Segurança)
- **Rastreabilidade:** `TRANSCRICAO.md` (`[09:12] Larissa`, `[09:12] Diego`, `[09:13] Bruno`, `[09:13] Diego`, `[09:13] Larissa`, `[09:14] Marcos`, `[09:24] Diego`, `[09:25] Bruno`, `[09:25] Diego`, `[09:25] Sofia`, `[09:26] Marcos`, `[09:26] Larissa`, `[09:43] Diego`, `[09:44] Diego`, `[09:51] Larissa`); `CODIGO` (`package.json`, `prisma/schema.prisma`)

---

## Contexto

Em um sistema distribuído de notificações via HTTP sobre a internet pública, falhas parciais são inevitáveis: o servidor do cliente B2B pode receber e processar o webhook com sucesso, mas a conexão TCP cair antes que o worker do OMS receba o status HTTP `200 OK`, levando a uma retentativa do mesmo evento ([ADR-003](./ADR-003-politica-de-retry-com-backoff-e-dlq.md)) (`[09:24] Diego`).

Além disso, um mesmo pedido pode sofrer transições sucessivas rápidas (ex.: `PAID` → `PROCESSING` → `SHIPPED`), levantando a questão sobre qual nível de garantia de ordenação e de unicidade de entrega a plataforma oferecerá aos integradores (`[09:12] Larissa`, `[09:25] Bruno`).

## Decisão

Adotar a garantia de entrega **At-Least-Once (pelo menos uma vez)** combinada com um identificador único de evento **`X-Event-Id` (UUID)** gerado na entrada da outbox para deduplicação idempotente no receptor, e estabelecer **ordenação implícita por pedido (`created_at` na outbox) sob execução single-worker** (`[09:12] Diego`, `[09:13] Larissa`, `[09:24] Diego`, `[09:25] Diego`, `[09:26] Larissa`).

Os termos desta decisão são:

1. **Semântica At-Least-Once:** O OMS garante que todo evento assinado será entregue pelo menos uma vez (salvo indisponibilidade superior à janela de 5 tentativas / ~15h, quando vai para DLQ), podendo ocorrer entregas duplicadas em cenários de timeout ou reprocessamento (`[09:24] Diego`).
2. **Identificador Único `X-Event-Id` (UUID):** Quando o evento é inserido na tabela `webhook_outbox`, um UUID v4 único é gerado (reutilizando a biblioteca `uuid` já listada em `package.json` e o padrão `@default(uuid())` de `prisma/schema.prisma`) (`[09:25] Diego`, `[09:51] Larissa`). Esse mesmo UUID é enviado tanto no header HTTP `X-Event-Id` quanto no campo `event_id` do corpo JSON em todas as tentativas de entrega daquele evento (`[09:25] Diego`, `[09:43] Diego`, `[09:44] Diego`).
3. **Deduplicação Idempotente pelo Cliente:** Cabe ao sistema receptor do cliente B2B armazenar/verificar o `X-Event-Id` recebido para descartar entregas repetidas de forma idempotente; esse contrato será documentado em destaque no portal do desenvolvedor (`[09:25] Diego`, `[09:25] Sofia`, `[09:26] Marcos`).
4. **Ordenação Implícita sob Single-Worker (Limitação Conhecida Documentada):** Enquanto houver um único worker lendo a `webhook_outbox` por ordem de `created_at`, eventos de um mesmo `order_id` serão disparados sequencialmente na ordem em que ocorreram (`[09:12] Diego`). Não há garantia de ordenação global se um evento individual entrar em fila de retry ou caso o sistema escale para múltiplos workers em paralelo no futuro (`[09:12] Diego`, `[09:13] Larissa`, `[09:14] Marcos`).

## Alternativas Consideradas

### Alternativa 1: Garantia de Entrega Exactly-Once (Exatamente Uma Vez)
- **Descrição:** Implementar protocolo de coordenação distribuída (two-phase commit ou handshakes de confirmação transacional) entre o OMS e os servidores dos clientes B2B para impedir qualquer entrega duplicada (`[09:25] Diego`).
- **Motivo do Descarte / Trade-off:** Exigiria coordenação complexa de estado dos dois lados da integração HTTP e impõe barreira técnica desproporcional aos clientes B2B. O padrão de mercado consolidado (adotado por Stripe, GitHub, etc.) é *at-least-once* com `event_id` para idempotência no consumidor, resolvendo 99% dos casos com simplicidade (`[09:25] Diego`).

### Alternativa 2: Escalonamento Multi-Worker Imediato com Particionamento por `order_id` ou Lock Pessimista
- **Descrição:** Construir desde o primeiro dia particionamento de fila por `order_id` ou *pessimistic locking* distribuído para permitir múltiplos workers paralelos mantendo ordenação estrita (`[09:13] Bruno`, `[09:13] Diego`).
- **Motivo do Descarte / Trade-off:** Os clientes B2B nunca solicitaram garantia de ordenação global estrita (`[09:14] Marcos`), e o volume atual é perfeitamente atendido por um único worker em polling de 2s (`[09:12] Diego`, `[09:13] Diego`). Resolver concorrência multi-worker agora seria *premature optimization*; fica registrado como evolução futura caso haja necessidade de escala horizontal (`[09:13] Diego`, `[09:13] Larissa`).

## Consequências

### Positivas
- **Simplicidade e Confiabilidade no Emissor:** O worker só precisa confirmar se recebeu status HTTP `2xx` dentro de 10s; em caso de dúvida ou timeout, agenda retry com segurança sabendo que o `X-Event-Id` permanece imutável (`[09:25] Diego`).
- **Alinhamento com Padrões de Mercado (Stripe/GitHub):** Desenvolvedores dos clientes B2B já estão habituados a implementar idempotência baseada em header `X-Event-Id` e `timestamp` (`[09:25] Diego`).
- **Ordenação Natural no Cenário Nominal:** Com single-worker lendo por `created_at`, transições rápidas (`PAID` → `PROCESSING` → `SHIPPED`) são processadas em ordem cronológica sem necessidade de coordenadores complexos (`[09:12] Diego`).

### Negativas e Trade-offs Explícitos
- **Responsabilidade de Idempotência Transferida ao Cliente:** Se o cliente B2B não deduplicar por `X-Event-Id`, poderá processar duas vezes a mesma mudança de status em caso de retentativa pós-timeout (`[09:25] Sofia`). **Trade-off aceito:** Mitigado por documentação explícita no portal do desenvolvedor pelo PM Marcos e inclusão de `X-Event-Id` e `X-Timestamp` em todos os disparos (`[09:26] Marcos`, `[09:44] Diego`).
- **Possibilidade de Inversão de Ordem em Caso de Falha Transitória:** Se a transição `PAID` falhar na primeira tentativa (entrando em backoff de 1 minuto) e a transição subsequente `PROCESSING` acontecer segundos depois quando o endpoint do cliente já se recuperou, `PROCESSING` pode ser entregue antes da retentativa de `PAID`. **Trade-off aceito:** O payload carrega `timestamp`, `from_status` e `to_status` explícitos (`[09:43] Diego`), permitindo ao cliente reconciliar o estado corretamente.
