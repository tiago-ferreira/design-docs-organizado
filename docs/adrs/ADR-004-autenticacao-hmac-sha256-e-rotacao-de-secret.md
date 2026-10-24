# ADR-004: Assinatura HMAC-SHA256 com Secret por Endpoint e Rotação com Grace Period de 24 Horas

- **Status:** Aceito
- **Data:** 2026-10-24
- **Decisores:** Sofia (Engenheira de Segurança), Larissa (Tech Lead), Diego (Engenheiro Sênior - Plataforma), Bruno (Engenheiro Pleno - Pedidos)
- **Rastreabilidade:** `TRANSCRICAO.md` (`[09:19] Sofia`, `[09:20] Sofia`, `[09:20] Bruno`, `[09:21] Sofia`, `[09:21] Bruno`, `[09:22] Diego`, `[09:22] Sofia`, `[09:23] Sofia`, `[09:31] Marcos`, `[09:44] Diego`, `[09:44] Sofia`, `[09:45] Diego`, `[09:46] Sofia`); `CODIGO` (`src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`)

---

## Contexto

O Sistema de Webhooks enviará payloads contendo informações de negócio sobre pedidos (`order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`) para servidores externos controlados pelos clientes B2B (`[09:19] Sofia`, `[09:43] Diego`). Os receptores precisam de um mecanismo criptográfico padrão para:

1. Autenticar que a requisição HTTP partiu genuinamente da nossa plataforma OMS e não de um atacante (`[09:19] Sofia`);
2. Verificar a integridade do corpo da requisição contra adulteração em trânsito (`[09:19] Sofia`);
3. Mitigar ataques de *replay* e permitir a substituição segura de chaves caso uma credencial vaze em logs de aplicação do cliente — incidente que já ocorreu historicamente na base (`[09:22] Diego`, `[09:44] Diego`).

## Decisão

Adotar **assinatura criptográfica HMAC-SHA256 sobre o corpo JSON da requisição**, utilizando uma **secret exclusiva por endpoint de webhook**, transporte **HTTPS (TLS) obrigatório**, headers de rastreio de segurança (`X-Signature`, `X-Timestamp`, `X-Webhook-Id`) e **rotação de secret via API com janela de transição (*grace period*) de 24 horas** (`[09:20] Sofia`, `[09:21] Sofia`, `[09:22] Sofia`, `[09:23] Sofia`, `[09:44] Diego`, `[09:44] Sofia`).

As diretrizes normativas desta decisão são:

1. **Algoritmo HMAC-SHA256:** Cada envio calcula o digest HMAC-SHA256 do payload serializado usando a secret do endpoint e o envia no header `X-Signature` (`[09:20] Sofia`, `[09:22] Sofia`).
2. **Headers Complementares de Segurança e Identificação:** Além de `X-Signature` e `X-Event-Id`, o worker envia `X-Timestamp` (timestamp ISO/epoch do momento do disparo para detecção de *replay attack* pelo cliente) e `X-Webhook-Id` (UUID do cadastro do endpoint, permitindo que clientes com múltiplos endpoints identifiquem qual configuração originou o disparo) (`[09:44] Diego`, `[09:44] Sofia`, `[09:45] Diego`).
3. **Secret Exclusiva por Endpoint:** A secret é gerada pela plataforma no momento do cadastro (`POST /api/v1/webhooks`), armazenada por endpoint na tabela de configuração e devolvida na resposta de criação (`[09:21] Sofia`, `[09:21] Bruno`, `[09:31] Marcos`). Nunca será utilizada uma secret global da plataforma (`[09:21] Sofia`).
4. **Rotação de Secret com Grace Period de 24 Horas:** O cliente pode solicitar uma nova secret via endpoint dedicado (`POST /api/v1/webhooks/:id/rotate-secret`). Ao rotacionar, a secret anterior permanece válida em paralelo por **24 horas** (`previousSecret` + `previousSecretExpiresAt`), permitindo que o cliente atualize seus sistemas sem queda na validação; após 24 horas, a secret antiga expira definitivamente (`[09:21] Sofia`, `[09:22] Sofia`). Durante a janela de 24h, o header `X-Signature` pode transportar as assinaturas ativa e anterior para validação transparente.
5. **Obrigatoriedade de HTTPS:** URLs com esquema `http://` são rejeitadas na entrada via validação de schema Zod (`src/middlewares/validate.middleware.ts`) com código de erro `WEBHOOK_INVALID_URL` (`[09:23] Sofia`, `[09:28] Bruno`).

## Alternativas Consideradas

### Alternativa 1: Secret Global Compartilhada na Plataforma
- **Descrição:** Utilizar uma única chave secreta da plataforma OMS para assinar webhooks de todos os clientes e endpoints (`[09:21] Sofia`).
- **Motivo do Descarte / Trade-off:** Cria um ponto único de falha catastrófico (*blast radius* total): se um único cliente vazar a chave ou tiver seu ambiente comprometido, todos os demais clientes ficariam vulneráveis a falsificação de eventos (`[09:21] Sofia`).

### Alternativa 2: Rotação de Secret com Invalidação Imediata (Sem Grace Period)
- **Descrição:** Substituir a secret imediatamente no banco ao chamar o endpoint de rotação, invalidando a chave antiga no mesmo instante (`[09:21] Sofia`).
- **Motivo do Descarte / Trade-off:** Como não há atualização atômica entre a nossa API e o deploy das variáveis de ambiente no servidor do cliente B2B, invalidar a secret instantaneamente causaria falha de verificação de assinatura em todos os eventos disparados enquanto o cliente atualiza sua infraestrutura (`[09:21] Sofia`).

### Alternativa 3: Permitir Endpoints HTTP Sem TLS
- **Descrição:** Aceitar URLs `http://` para facilitar testes ou ambientes legados de clientes (`[09:23] Sofia`).
- **Motivo do Descarte / Trade-off:** Expõe payloads de pedidos em texto claro na internet pública. O uso de HTTPS é mandatório e validado diretamente no schema Zod (`[09:23] Sofia`).

## Consequências

### Positivas
- **Isolamento de Blast Radius por Endpoint:** O vazamento de uma secret afeta exclusivamente um único endpoint de um único cliente (`[09:21] Sofia`, `[09:22] Diego`).
- **Zero Downtime em Rotações de Credenciais:** O *grace period* de 24 horas permite resposta rápida a incidentes de vazamento em log sem quebrar a integração em produção durante a janela de migração do cliente (`[09:21] Sofia`).
- **Compatibilidade Universal:** HMAC-SHA256 é amplamente suportado pela biblioteca padrão de qualquer linguagem moderna usada pelos clientes B2B (`[09:20] Sofia`).

### Negativas e Trade-offs Explícitos
- **Complexidade Adicional no Modelo e na Assinatura durante o Grace Period:** O schema da tabela de endpoints precisa manter `secret`, `previous_secret` e `previous_secret_expires_at`, e o gerador de assinatura precisa incluir a assinatura secundária enquanto estiver dentro das 24h (`[09:21] Sofia`). **Trade-off aceito:** Custo de implementação baixo frente ao ganho operacional e de segurança; haverá revisão dedicada de 2 dias úteis pela engenheira de segurança Sofia antes do deploy (`[09:46] Sofia`).
