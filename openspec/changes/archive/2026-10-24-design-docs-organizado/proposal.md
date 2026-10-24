# Proposal

## Why

Três clientes B2B estratégicos da plataforma de Order Management System (Atlas Comercial, MaxDistribuição e Nova Cargo) necessitam receber notificações quase em tempo real (< 10s) sobre mudanças de status de pedidos para substituir o modelo atual de polling constante em `GET /orders`, sob risco de churn até o final do trimestre. Como as decisões arquiteturais e de produto foram tomadas em uma reunião técnica registrada apenas em `TRANSCRICAO.md`, é necessário produzir o pacote completo e rastreável de Design Docs (PRD, RFC, FDD, ADRs, Tracker e README de processo) para viabilizar a implementação segura e alinhada pelo time de engenharia.

## What Changes

- **Produção do PRD (`docs/PRD.md`)**: Especificação de produto cobrindo problema, público-alvo B2B, metas quantitativas (< 10s de latência, retenção dos 3 clientes B2B), requisitos funcionais (mínimo 8) e não funcionais, escopo e exclusões explícitas (notificações por e-mail, dashboard visual, rate limiting de saída, inbound webhooks), riscos com probabilidade/impacto/mitigação e critérios de aceite.
- **Produção do RFC (`docs/RFC.md`)**: Proposta arquitetural concisa para revisão da equipe (Larissa, Marcos, Bruno, Diego, Sofia), detalhando a abordagem Outbox + Worker em polling, alternativas descartadas com seus trade-offs (chamada HTTP síncrona na transação, Redis Streams/Cluster, triggers nativos no MySQL, retry com 3 tentativas ou indefinido, exactly-once) e questões em aberto (rate limiting de envio, política de arquivamento após 30 dias, endurecimento de RBAC no CRUD).
- **Produção do FDD (`docs/FDD.md`)**: Especificação técnica acionável de implementação contendo modelagem Prisma (`webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`), fluxos detalhados (outbox transacional com snapshot JSON, polling de 2s, retry exponencial 1m/5m/30m/2h/12h, DLQ e replay), contratos HTTP públicos (pelo menos 6 endpoints com payloads e status codes), matriz de erros `WEBHOOK_*`, observabilidade (Pino logs, métricas e tracing) e seção obrigatória de integração com ao menos 4 arquivos reais da codebase.
- **Produção de 6 ADRs (`docs/adrs/ADR-001-*.md` a `ADR-006-*.md`)**: Registro formal no padrão MADR das 6 decisões arquiteturais centrais tomadas na reunião (Outbox no MySQL, Worker separado em polling de 2s, Retry com backoff e DLQ em tabela separada, Segurança HMAC-SHA256 com rotação de secret 24h, Entrega at-least-once com `X-Event-Id`, e Reuso dos padrões existentes da codebase).
- **Produção do Tracker de Rastreabilidade (`docs/TRACKER.md`)**: Matriz cruzada mapeando no mínimo 80% dos itens documentados às suas fontes exatas em `TRANSCRICAO.md` (`[hh:mm] Nome`) e no código-fonte (`CODIGO` com caminhos reais), garantindo zero alucinação.
- **Atualização do README (`README.md`)**: Documentação completa do processo de engenharia assistida por IA, ferramentas utilizadas, workflow OpenSpec, prompts customizados e ciclos de iteração/refinamento.
- **Restrição Estrita de Não-Alteração de Código**: Nenhuma modificação será feita em `src/`, `prisma/`, `tests/`, `TRANSCRICAO.md` ou arquivos de configuração da aplicação.

## Capabilities

### New Capabilities
- `order-webhook-design-docs`: Governança, estrutura, rastreabilidade e requisitos técnicos/funcionais do pacote completo de Design Docs (PRD, RFC, FDD, ADRs, Tracker e README) para o Sistema de Webhooks de Notificação de Pedidos.

### Modified Capabilities
<!-- Nenhuma capability existente foi modificada pois o repositório não possuía especificações prévias em openspec/specs/. -->

## Impact

- **Documentação (`docs/` e `README.md`)**: Preenchimento completo de `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`, criação de `docs/adrs/ADR-001-*.md` até `ADR-006-*.md` e substituição de `README.md`.
- **Código-fonte e Banco de Dados (`src/`, `prisma/`, `tests/`)**: Impacto zero (leitura estrita como referência arquitetural; nenhuma linha de código ou schema será editada).
