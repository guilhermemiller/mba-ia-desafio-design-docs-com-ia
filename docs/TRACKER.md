# Tracker de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | `docs/PRD.md` | Requisito de Negócio | Clientes B2B (Atlas, Max, Nova Cargo) pediram webhooks para substituir polling. | `TRANSCRICAO` | [09:00] Marcos |
| PRD-CTX-02 | `docs/PRD.md` | Prazo / Risco | Cliente Atlas ameaçou migrar para concorrente; prazo final do trimestre (nov). | `TRANSCRICAO` | [09:00] Marcos |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Latência considerada "tempo real" pelos clientes é abaixo de 10s. | `TRANSCRICAO` | [09:02] Marcos |
| PRD-SCOPE-01 | `docs/PRD.md` | Restrição | Apenas webhooks outbound (sistema para clientes). | `TRANSCRICAO` | [09:02] Sofia |
| ADR-001-01 | `docs/adrs/ADR-001-outbox-pattern-mysql.md` | Decisão Arquitetural | Padrão Outbox no MySQL: evento inserido na mesma transação de banco. | `TRANSCRICAO` | [09:06] Diego |
| ADR-001-02 | `docs/adrs/ADR-001-outbox-pattern-mysql.md` | Alternativa | Rejeição de broker de mensagens (Redis Streams) por ser overengineering. | `TRANSCRICAO` | [09:07] Diego |
| ADR-002-01 | `docs/adrs/ADR-002-worker-polling-strategy.md` | Decisão Arquitetural | Worker vai ler tabela de outbox fazendo polling a cada 2 segundos. | `TRANSCRICAO` | [09:09] Diego |
| ADR-002-02 | `docs/adrs/ADR-002-worker-polling-strategy.md` | Restrição | Worker deve rodar como processo separado para não cair se a API reiniciar. | `TRANSCRICAO` | [09:11] Diego |
| ADR-002-03 | `docs/adrs/ADR-002-worker-polling-strategy.md` | Limitante Conhecida | Garantia de ordering apenas por `order_id` assumindo worker único atual. | `TRANSCRICAO` | [09:12] Diego |
| ADR-003-01 | `docs/adrs/ADR-003-retry-policy-dlq.md` | Decisão Arquitetural | Retry Progressivo (Backoff Exponencial) com 5 tentativas limite. | `TRANSCRICAO` | [09:15] Diego |
| ADR-003-02 | `docs/adrs/ADR-003-retry-policy-dlq.md` | Parâmetro | Progressão definida: 1m, 5m, 30m, 2h, 12h. | `TRANSCRICAO` | [09:17] Diego |
| ADR-003-03 | `docs/adrs/ADR-003-retry-policy-dlq.md` | Decisão Arquitetural | Isolar falhas totais numa DLQ separada e prover Endpoint de Replay Manual. | `TRANSCRICAO` | [09:18] Diego |
| ADR-004-01 | `docs/adrs/ADR-004-hmac-sha256-webhook-security.md` | Segurança | Assinatura com HMAC-SHA256 para comprovar veracidade e não-adulteração. | `TRANSCRICAO` | [09:20] Sofia |
| ADR-004-02 | `docs/adrs/ADR-004-hmac-sha256-webhook-security.md` | Segurança | Secret única e individual gerada por endpoint do cliente (nada de secret global). | `TRANSCRICAO` | [09:21] Sofia |
| ADR-004-03 | `docs/adrs/ADR-004-hmac-sha256-webhook-security.md` | Segurança | Suporte para rotação de Segredo provendo Grace Period duplo de 24h. | `TRANSCRICAO` | [09:21] Sofia |
| PRD-FR-02 | `docs/PRD.md` | Segurança | TLS/HTTPS mandatório no cadastro da URL. Rejeitar http. | `TRANSCRICAO` | [09:23] Sofia |
| ADR-005-01 | `docs/adrs/ADR-005-at-least-once-delivery-x-event-id.md` | Decisão Arquitetural | Garantia do tipo At-Least-Once usando ID Único gerado para os eventos. | `TRANSCRICAO` | [09:24] Diego |
| ADR-005-02 | `docs/adrs/ADR-005-at-least-once-delivery-x-event-id.md` | Padrão Integrativo | Header `X-Event-Id` p/ deduplicação é entregue como UUID nativo no payload. | `TRANSCRICAO` | [09:25] Diego |
| ADR-006-01 | `docs/adrs/ADR-006-reuse-existing-patterns.md` | Padrões de Código | Padrão arquitetural MVC/Service local de subpastas `src/modules/...` conservado. | `TRANSCRICAO` | [09:27] Bruno |
| ADR-006-02 | `docs/adrs/ADR-006-reuse-existing-patterns.md` | Padrões de Código | Reuso do modelo da classe estendida `AppError` criando prefixos `WEBHOOK_*`. | `TRANSCRICAO` | [09:28] Bruno |
| ADR-006-03 | `docs/adrs/ADR-006-reuse-existing-patterns.md` | Padrões de Código | Reaproveitamento do `error middleware` central, do Logger `Pino` e do DB. | `TRANSCRICAO` | [09:29] Bruno |
| FDD-INT-01 | `docs/FDD.md` | Integração (Código) | Integração central ocorrerá no método `changeStatus` englobando tudo num `$transaction`. | `CODIGO` | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração (Código) | Chamada do worker exigirá DB `PrismaClient` segregado e rotina em `worker.ts`. | `CODIGO` | `src/server.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração (Código) | A classe pai para tratativas centralizadas é a `AppError`. | `CODIGO` | `src/shared/errors/app-error.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração (Código) | Inserção do módulo `webhook` necessitará instanciação manual local no setup. | `CODIGO` | `src/app.ts` |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Endpoint Webhooks CRUD + Histórico. | `TRANSCRICAO` | [09:34] Marcos |
| PRD-NFR-03 | `docs/PRD.md` | Segurança | JWT Role "ADMIN" será requerido pro fluxo do DLQ Replay de Endpoint. | `TRANSCRICAO` | [09:36] Sofia |
| PRD-OUT-01 | `docs/PRD.md` | Fora de Escopo | E-mails como notificação de queda no envio desconsiderados (Fase Futura). | `TRANSCRICAO` | [09:37] Larissa |
| PRD-OUT-02 | `docs/PRD.md` | Fora de Escopo | Controle de Rate Limiting por excesso de Webhooks foi posposto. | `TRANSCRICAO` | [09:39] Larissa |
| PRD-OUT-03 | `docs/PRD.md` | Fora de Escopo | Painel/UI do Cliente negado, mantendo a prioridade 100% via integrações API. | `TRANSCRICAO` | [09:40] Larissa |
| FDD-PAY-01 | `docs/FDD.md` | Detalhamento Técnico | Tempo Limite (Timeout) das chamadas HTTP do Worker marcado em 10 segundos. | `TRANSCRICAO` | [09:42] Diego |
| FDD-PAY-02 | `docs/FDD.md` | Detalhamento Técnico | Payload de Saída não envia items da order, só base fields (evita payload > 64KB). | `TRANSCRICAO` | [09:43] Diego |
| FDD-PAY-03 | `docs/FDD.md` | Detalhamento Técnico | Headers Obrigatórios de Saída: Signature, Event-Id, Webhook-Id, Timestamp. | `TRANSCRICAO` | [09:44] Sofia |
| PRD-EST-01 | `docs/PRD.md` | Estimativa | Aprovação final do ciclo base em 3 Sprints e c/ review prévia de 2 dias p/ Sec. | `TRANSCRICAO` | [09:46] Larissa |
| ADR-007-01 | `docs/adrs/ADR-007-snapshot-payload-at-insertion.md` | Decisão Arquitetural | Salvar Snapshot exato no `outbox` da payload pra prevenir assimetria por edição atrasada. | `TRANSCRICAO` | [09:51] Larissa |