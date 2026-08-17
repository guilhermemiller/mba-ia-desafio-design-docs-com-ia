# RFC — Request for Comments: Order Status Change Webhook System

**Status:** Proposed  
**Author:** Engenharia (Desafio Design Docs)  
**Date:** 2026-08-13  
**Reviewers:** Larissa (Tech Lead), Marcos (PM), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior), Sofia (Eng. Segurança)

## 1. Resumo Executivo (TL;DR)

Esta RFC propõe a arquitetura para um sistema de Webhooks Outbound que notificará clientes B2B em tempo real (<10s) sobre mudanças de status em seus pedidos. A solução adota o **Padrão Outbox transacional no MySQL**, um **worker separado em polling**, política de **retry com backoff exponencial e Dead Letter Queue (DLQ)**, e segurança via **assinatura HMAC-SHA256**. Esta abordagem reaproveita a infraestrutura existente (Node.js, Prisma, MySQL), garantindo entrega at-least-once sem adicionar complexidade operacional de message brokers externos.

## 2. Contexto e Problema

Três clientes B2B estratégicos (Atlas Comercial, MaxDistribuição, Nova Cargo) solicitaram notificações em tempo real de mudanças de status de pedidos. Atualmente, eles fazem polling no endpoint `GET /orders`, o que é ineficiente, sobrecarrega a API e atrasa a integração. A Atlas ameaçou migrar para um concorrente caso a feature não seja entregue até o final do trimestre (novembro).

**Requisitos e Restrições Atuais:**
- O sistema atual (Node.js + Express + Prisma + MySQL) possui uma arquitetura de monolito modular e não tem infraestrutura de mensageria (RabbitMQ, Kafka, etc).
- A mudança de status de pedido ocorre dentro de uma transação complexa do Prisma no `OrderService` (atualiza pedido, insere histórico, debita/repõe estoque).
- Não podemos bloquear essa transação com chamadas HTTP síncronas para os clientes.
- A latência aceitável para os clientes é inferior a 10 segundos.
- O sistema será apenas outbound (nós enviamos, não recebemos webhooks).

## 3. Proposta Técnica

Propomos implementar um **Padrão Outbox (Outbox Pattern)** operando sobre nosso banco MySQL existente, suportado por um worker background dedicado.

### 3.1 Padrão Outbox Transacional
Dentro da mesma transação do Prisma que altera o status do pedido (`OrderService.changeStatus`), inseriremos o evento na tabela `webhook_outbox`. 
- **Garantia:** Se a transação falhar (estoque insuficiente, erro de banco), o evento não é registrado. Se comitar, o evento está garantido.
- **Snapshot:** O payload JSON completo será renderizado e salvo no momento da inserção, garantindo um registro imutável do estado do pedido naquele exato momento.

### 3.2 Worker de Processamento
Um processo Node.js independente (`src/worker.ts`), executando na mesma base de código mas isolado da API, fará **polling a cada 2 segundos** na tabela `webhook_outbox`.
- Busca os eventos mais antigos com status `PENDENTE`.
- Processa o envio HTTP (limite de 10s de timeout).
- Atualiza o status para `ENTREGUE` ou `FALHOU`.

### 3.3 Resiliência (Retry e DLQ)
Para lidar com indisponibilidades dos clientes:
- **Backoff Exponencial:** 5 tentativas nos intervalos: 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas (janela total de ~15 horas).
- **Dead Letter Queue (DLQ):** Após a 5ª falha, o evento é movido para uma tabela separada (`webhook_dead_letter`) com o payload, motivo do erro e timestamp.
- **Replay Manual:** Um endpoint administrativo (`POST /admin/webhooks/dead-letter/:id/replay`) permitirá reprocessar eventos da DLQ após o cliente resolver sua indisponibilidade.
- **Garantia de Entrega:** At-least-once. Forneceremos um UUID único no header `X-Event-Id` para o cliente deduplicar os eventos do lado dele.

### 3.4 Segurança
- **Assinatura HMAC-SHA256:** O payload será assinado com um segredo (secret) único por endpoint configurado, enviado no header `X-Signature`.
- **Rotação de Segredos:** Os clientes poderão rotacionar o segredo via API. O segredo antigo terá um *grace period* de 24 horas, período em que tanto o novo quanto o antigo funcionarão.
- **TLS Obrigatório:** As URLs cadastradas devem obrigatoriamente ser HTTPS.

### 3.5 Integração com o Código Base
A solução reusará amplamente os padrões estabelecidos no projeto:
- Módulo isolado em `src/modules/webhooks`.
- Classes de erro estendendo `AppError` com prefixo `WEBHOOK_`.
- Validação via middlewares existentes e schemas Zod.
- Logger Pino centralizado.

## 4. Alternativas Consideradas

Foram discutidas e descartadas as seguintes alternativas durante a reunião técnica:

### Alternativa 1: Chamada HTTP Síncrona na Transação
**Descrição:** Disparar o webhook diretamente do `OrderService` assim que o status mudar.
**Trade-off e Descarte:** Rejeitada pois acopla o tempo de resposta da nossa transação de banco de dados ao tempo de resposta do servidor do cliente. Se o cliente estiver lento ou fora do ar, nossa transação de banco (que altera estoque) travaria ou sofreria rollback, afetando toda a plataforma.

### Alternativa 2: Mensageria Externa (Redis Streams / Kafka)
**Descrição:** Publicar o evento em um broker de mensagens em vez de usar uma tabela MySQL.
**Trade-off e Descarte:** Rejeitada (considerada *overengineering*). Nossa equipe é pequena e não temos essa infraestrutura hoje. Manter a coesão no MySQL garante atomicidade transacional sem o problema de *dual-write* e sem custo de setup e monitoramento de nova infraestrutura.

### Alternativa 3: Retry Indefinido
**Descrição:** Tentar reenviar o evento para sempre, sem teto de tentativas.
**Trade-off e Descarte:** Rejeitada pois um endpoint permanentemente fora do ar acumularia lixo infinito em nossa fila, consumindo recursos do worker. O limite de 5 tentativas (~15h) é suficiente para cobrir janelas de manutenção; após isso, a DLQ assume.

### Alternativa 4: Trigger no Banco (Notificação Push)
**Descrição:** Usar triggers do MySQL para notificar o worker em vez de polling.
**Trade-off e Descarte:** Rejeitada pois o MySQL não possui funcionalidade nativa equivalente ao `NOTIFY/LISTEN` do PostgreSQL. Qualquer workaround (como escrever em arquivos via UDF) seria frágil. Polling de 2 segundos atende facilmente a restrição de <10s.

## 5. Questões em Aberto

Os seguintes pontos foram identificados, mas sua decisão final foi postergada para fases futuras:

1. **Rate Limiting de Saída:** Se um cliente tiver muitos pedidos alterados simultaneamente (ex: processamento em lote), nosso worker poderá disparar dezenas de requisições por segundo contra o endpoint dele. Decidimos **observar primeiro** e implementar limitação de taxa (throttling) no futuro, se virar um problema.
2. **Ordering Global em Cenário de Multi-Workers:** A abordagem atual com um único worker garante que os eventos de um mesmo pedido sejam entregues em ordem (FIFO pelo `created_at`). Se no futuro precisarmos escalar para múltiplos workers em paralelo, essa garantia se perde. Soluções como particionamento por `order_id` ficam para o futuro.
3. **Notificação de Falhas por Email:** Foi sugerido avisar o cliente por email caso o endpoint dele falhe repetidamente. Definido como **fora de escopo para a V1**, podendo ser adicionado após medirmos o volume na DLQ.

## 6. Impacto e Riscos

- **Risco Operacional:** O worker é um novo processo Node.js independente que precisa de monitoramento (se ele cair silenciosamente, eventos se acumulam no banco).
- **Impacto no Banco de Dados:** A tabela `webhook_outbox` crescerá rápido. Será necessária uma política de limpeza de eventos entregues (archival), embora o escopo inicial suporte o volume.
- **Segurança (HMAC):** Erros na implementação do HMAC ou no grace period da rotação de segredos podem causar falhas de integração. A Engenheira de Segurança revisará a PR com 2 dias de antecedência.

## 7. Decisões Relacionadas

As minúcias técnicas desta RFC estão documentadas nos seguintes Architecture Decision Records (ADRs):

- [ADR-001: Padrão Outbox Transacional no MySQL](./adrs/ADR-001-outbox-pattern-mysql.md)
- [ADR-002: Worker Separado em Polling](./adrs/ADR-002-worker-polling-strategy.md)
- [ADR-003: Política de Retry e DLQ](./adrs/ADR-003-retry-policy-dlq.md)
- [ADR-004: Segurança HMAC-SHA256](./adrs/ADR-004-hmac-sha256-webhook-security.md)
- [ADR-005: Garantia At-Least-Once com X-Event-Id](./adrs/ADR-005-at-least-once-delivery-x-event-id.md)
- [ADR-006: Reuso de Padrões do Projeto](./adrs/ADR-006-reuse-existing-patterns.md)
- [ADR-007: Snapshot de Payload na Inserção](./adrs/ADR-007-snapshot-payload-at-insertion.md)