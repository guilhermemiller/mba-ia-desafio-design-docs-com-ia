# FDD — Feature Design Document: Webhooks de Pedidos

## 1. Contexto e Motivação Técnica

Os principais clientes B2B do Order Management System exigem ser notificados em tempo real quando os seus pedidos sofrem mudança de status (ex: `PENDING` → `PAID` → `SHIPPED`). A arquitetura atual, focada em APIs REST síncronas, os obriga a fazer polling periódico no endpoint de listagem, o que onera ambos os sistemas. A implementação deste sistema de webhooks (*outbound*) preencherá a lacuna de integração *event-driven* da plataforma, despachando as notificações automaticamente para os endpoints dos clientes, de forma segura, rastreável e resiliente a falhas transitórias, garantindo at-least-once delivery.

## 2. Objetivos Técnicos

1. Capturar, de forma atômica, todas as transições de status de pedidos (`OrderService.changeStatus`).
2. Entregar eventos (formato JSON) via HTTP(S) para as URLs cadastradas com latência p95 < 10 segundos.
3. Prover um ciclo de vida robusto (Retry, Backoff Exponencial, Dead Letter Queue) para endpoints indisponíveis.
4. Assegurar autenticidade e integridade da mensagem via assinatura HMAC-SHA256, com segredo por endpoint.
5. Permitir a gestão autônoma das URLs, segredos e histórico de entregas via API CRUD.

## 3. Escopo e Exclusões

**In Scope:**
- CRUD de configurações de webhook, protegido por JWT.
- Tabela `webhook_outbox` inserida atomicamente junto com a mudança do status do pedido.
- Worker autônomo (processo Node.js segregado) que faz polling no banco de dados.
- Lógica de Retry progressivo e DLQ para falhas permanentes (5 tentativas).
- Endpoint Admin para Replay de mensagens mortas.
- Consulta de Histórico de Envios (`GET /webhooks/:id/deliveries`).

**Exclusões (Out of Scope):**
- Throttling e Rate Limiting explícito no envio para o cliente (adiado para v2).
- Envio de notificações de falhas críticas (e.g., webhook travado) por e-mail para o cliente.
- Interface visual (Dashboard UI) – O escopo é 100% backend/API.
- Ordenação garantida e estrita de processamento em ambiente de multi-workers particionados.
- Exactly-Once Delivery (forneceremos At-Least-Once com `X-Event-Id` para deduplicação do lado do cliente).

## 4. Integração com o Sistema Existente

O módulo respeitará os padrões adotados no restante da aplicação, interligando-se da seguinte forma:

1. **`src/modules/orders/order.service.ts`:**
   No método `changeStatus` (aprox. linha 158), adicionaremos a chamada à nova função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`. Esta função filtrará as configurações ativas do `customer_id` e persistirá as mensagens na `webhook_outbox` utilizando o mesmo `Prisma.TransactionClient` (`tx`) em vigor, assegurando o comportamento ACID.

2. **`src/shared/errors/app-error.ts` e `http-errors.ts`:**
   O padrão `AppError` será herdado por classes do domínio de webhook (ex. `WebhookNotFoundError`, `WebhookSignatureError`), com códigos no padrão `WEBHOOK_*`.

3. **`src/middlewares/error.middleware.ts`:**
   Como estenderemos o `AppError`, o *error middleware* central não exigirá alterações e tratará adequadamente a formatação das respostas (HTTP 4xx/5xx).

4. **`src/middlewares/auth.middleware.ts`:**
   Endpoints expostos de CRUD usarão `authenticate` para verificação de JWT. O endpoint de Replay da DLQ consumirá o validador `requireRole('ADMIN')`.

5. **`src/app.ts` e `src/worker.ts`:**
   A injeção de dependências do módulo de webhook será orquestrada no `buildControllers` do `app.ts`. O worker será criado no novo entry-point `src/worker.ts`, que criará sua própria instância autônoma do `PrismaClient` para não colidir com o event loop do servidor express.

## 5. Fluxos Detalhados

### 5.1 Geração e Registro do Evento (Outbox)
1. Cliente envia `PATCH /orders/:id/status` para trocar status de A para B.
2. `OrderService.changeStatus` abre a transação `$transaction`.
3. Executa as validações, altera status, debita/credita stock e insere histórico.
4. Chama `publishWebhookEvent`. Esta função levanta os webhooks do `customer_id` que escutam a transição (status B).
5. Se existirem endpoints compatíveis, renderiza o payload (snapshot com valor exato do momento, sem lista de `items`) e insere na `webhook_outbox` como `PENDING`.
6. O `commit` consolida todas as ações atômicas.

### 5.2 Processamento (Worker Polling)
1. `src/worker.ts` inicia em um `setInterval` de 2 segundos.
2. A cada loop, faz uma query bloqueante/ordenada (limitando o batch, e.g., 50 registros) filtrando: `status = PENDING` ou `(status = FAILED AND next_retry_at <= NOW())`. Ordena por `created_at ASC`.
3. Para cada registro, despacha a chamada HTTP:
   - Aplica timeout de 10s.
   - Computa a assinatura HMAC-SHA256 (valida a config da URL, gerando o `X-Signature`).
4. Se responder 2xx: Atualiza para `DELIVERED`, zera falhas, anota latência e loga o sucesso no `webhook_delivery_log`.
5. Se não responder 2xx ou timeout: Envia para fluxo de Retry/DLQ.

### 5.3 Retry e DLQ (Dead Letter Queue)
1. Na captura de uma falha de envio HTTP:
   - Incrementa o `attempts`.
   - Se `attempts >= 5`: Move o evento para `webhook_dead_letter` (salva o payload original e motivo do erro). O evento é removido logicamente da outbox principal.
   - Se `attempts < 5`: Agenda o `next_retry_at` no registro com base no backoff exponencial predeterminado (1m, 5m, 30m, 2h, 12h) e volta o status para `FAILED` temporariamente.
2. Registros da DLQ não são pegos pelo Polling, até que o Admin dispare o endpoint de replay, que reinjeta o evento na `webhook_outbox` como `PENDING`, zerando as falhas.

## 6. Contratos Públicos

O módulo será construído com endpoints padronizados REST. Todas as rotas estão montadas em `/api/v1/webhooks`.

### 6.1 `POST /api/v1/webhooks`
Cadastra um novo endpoint. O retorno único revelará o `secret`.
- **Payload:**
```json
{
  "customer_id": "c1f7b0e1-4c12-4c6e-8d8d-1e1b1b1b1b1b",
  "url": "https://api.atlas.com/webhooks/orders",
  "events": ["PAID", "SHIPPED", "DELIVERED"]
}
```
- **Response (201 Created):**
```json
{
  "id": "w1f7b0e1-...",
  "url": "https://api.atlas.com/webhooks/orders",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "secret": "whsec_2a...5f", 
  "active": true
}
```
*(Nota: a `secret` não será devolvida novamente nas requisições GET. É armazenada criptografada - e.g., via bcrypt).*

### 6.2 `GET /api/v1/webhooks?customer_id=xxx`
Lista endpoints ativos do cliente.
- **Response (200 OK):** *(Apenas retorna as configurações, sem a secret)*

### 6.3 `GET /api/v1/webhooks/:id/deliveries`
Consulta o histórico das últimas (máx 100) tentativas/sucessos.
- **Response (200 OK):**
```json
{
  "data": [
    {
      "id": "del_1...",
      "event_id": "evt_9...",
      "status": 200,
      "response_body": "{\"received\": true}",
      "latency_ms": 145,
      "created_at": "2026-08-13T10:00:00Z"
    }
  ],
  "meta": { "total": 1, "page": 1, "pageSize": 100 }
}
```

### 6.4 `POST /api/v1/admin/webhooks/dead-letter/:id/replay`
Endpoint administrativo de replay. Reinserir o evento da DLQ.
- **Header:** JWT de Role `ADMIN`.
- **Response:** `202 Accepted`

### 6.5 Payload do Webhook Disparado ao Cliente
Como o evento entregue ao cliente parecerá (Payload Snapshot Enxuto).
- **Headers enviados:**
  - `Content-Type`: `application/json`
  - `X-Event-Id`: `uuid-v4-unique`
  - `X-Signature`: `sha256=a1b2c3d4...`
  - `X-Timestamp`: `2026-08-13T10:00:05Z`
  - `X-Webhook-Id`: `id-da-configuracao`
- **Body JSON:**
```json
{
  "event_id": "evt_uuid_123",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-13T10:00:00Z",
  "order_id": "ord_uuid_456",
  "order_number": "ORD-000123",
  "from_status": "PENDING",
  "to_status": "PAID",
  "customer_id": "cust_uuid_789",
  "total_cents": 15000
}
```

## 7. Matriz de Erros Previstos (Padrão WEBHOOK_*)

| Código de Erro | Http Status | Motivo | Tratamento / Middleware |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Edição/deleção de config inexistente. | Responde 404 limpo. |
| `WEBHOOK_INVALID_URL` | 400 | URL não é `https://` (recusada pelo Zod schema). | Responde 400 (Bad Request). |
| `WEBHOOK_LIMIT_EXCEEDED` | 413 | Tamanho de Payload ultrapassou 64KB no pipeline do Worker. | Rejeita envio e envia p/ DLQ. |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | Tentativa de Replay de evento que não está na DLQ. | Responde 404 limpo. |
| `WEBHOOK_FORBIDDEN_ROLE` | 403 | Operador comum tenta chamar a rota Admin de Replay. | Responde 403 (Forbidden). |

## 8. Observabilidade

1. **Métricas:** O sistema poderá rastrear volume, via Prometheus/Datadog (logs parseados ou middleware nativo), para métricas vitais: latência do webhook, eventos caídos em DLQ, sucessos vs. falhas.
2. **Logs (Pino):**
   - Transição gravada no DB logará um nível `info` com metadados `{ event_id, order_id }`.
   - Disparo do worker gera logs `debug` no início da requisição e `info` no seu encerramento com `latency_ms` e `http_status`.
3. **Tracing (implícito):** O `X-Event-Id` pode atuar como um *Correlation ID* entre os serviços no futuro. O endpoint histórico grava as entregas de modo transparente.

## 9. Dependências e Compatibilidade

- **Prisma Client:** Para operar tabelas: `webhook_configs`, `webhook_outbox`, `webhook_delivery_log`, `webhook_dead_letter`.
- **Crypto / JsonWebToken:** Geração de chaves (HMAC-SHA256) nativa em `crypto.createHmac`. Validação de secret gerada localmente.
- O Node.js 20 nativo fornece API de `fetch()`, sugerindo dispensar bibliotecas extras como `axios` para o worker se ater à API nativa, mas compatível com os `Timeouts` estritos via `AbortController`.

## 10. Riscos e Mitigação

- **Risco de Gargalo na Outbox:** Devido ao polling frequente.
  - *Mitigação:* Queries paginadas limitadas (`take: 50`) e filtragem exclusiva em status PENDING, aliviadas pela criação de índices (`CREATE INDEX ix_outbox_status_created ON webhook_outbox(status, created_at)`). Eventos finalizados sofrem "archival" futuramente.
- **Risco de Re-entregas excessivas durante downtime:** 
  - *Mitigação:* A regra Exponencial (1m, 5m, 30m, 2h, 12h) atenua pressões em picos prolongados de indisponibilidade externa.
- **Risco de Tamanhos absurdos nos Eventos (> 64KB):**
  - *Mitigação:* A escolha pela "Payload Enxuta" (não trazendo a árvore completa de `Items` da order) fixa as notificações de mudança de status em poucos kilobytes, impossibilitando atingir a margem de 64KB por evento.

## 11. Critérios de Aceite Técnicos

- Endpoints CRUD testados com integração (Supertest), respondendo `2xx` com as estruturas corretas e JWT operando.
- Modificação no `OrderService` devidamente unificada à transação: A falha forçada no `publishWebhookEvent` impede a mudança no pedido e na ordem do histórico.
- Worker (`npm run worker`) logando as tentativas HTTP para endpoint fantasma/mocked, parando após 5 falhas e depositando na DLQ com os tempos de Next_Retry respeitados.
- Checagem do `X-Signature` via Hmac-Sha256 na porta do recebedor deve confirmar total equivalência ao recálculo do payload enviado usando a secret concedida no POST.