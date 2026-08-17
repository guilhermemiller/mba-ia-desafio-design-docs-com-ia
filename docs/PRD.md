# PRD — Product Requirement Document: Sistema de Webhooks de Notificação de Pedidos

## 1. Resumo e Contexto da Feature

O sistema atual do Order Management System (OMS) demanda de integrações *event-driven* com os clientes empresariais. Atualmente, recebemos reclamações constantes de três clientes B2B estratégicos (Atlas Comercial, MaxDistribuição e Nova Cargo) sobre a ausência de notificações *push*. Eles são obrigados a praticar *polling* periódico chamando a rota `GET /orders` para descobrir mudanças de status, acarretando custos infraestruturais desnecessários para ambos os lados e latência inaceitável. Para neutralizar a ameaça de cancelamento (ex: cliente Atlas avisando migração caso não exista entrega até o final do trimestre), desenvolveremos a infraestrutura de **Outbound Webhooks**.

A solução adotará a inserção instantânea de eventos de mudança (`Order Status Changed`) numa fila local do nosso banco MySQL (*Outbox Pattern*), sendo despachada aos clientes via *worker background* assíncrono para garantir não impactar a transação do banco, resiliência de tentativa/erro e isolamento.

## 2. Problema e Motivação

**O problema:** Os sistemas B2B dos nossos clientes não ficam sabendo em tempo real quando as ordens de compra evoluem (Ex: `PENDING` para `PAID` ou `SHIPPED`).
**O impacto:** Lentidão, overhead nas integrações via *polling* ativo (alto RPM desperdiçado sem novidades na payload). A Atlas estipulou limite de fim do ano (novembro).
**O valor:** Promover agilidade e tempo-real (*near real-time*, <10s) na atualização das entregas e processamentos, satisfazendo parceiros-chave e tornando nossa API atrativa/competitiva a longo prazo.

## 3. Público-Alvo e Cenários de Uso

- **Atores principais (Clientes B2B):** Desenvolvedores/Sistemas das companhias que integram conosco. Eles farão o cadastro da sua URL de Webhook, copiarão a `Secret` de assinatura, e criarão, nos seus sistemas, a aceitação do nosso POST com a atualização do pedido.
- **Atores Internos (Suporte Técnico / Admin):** Operadores avançados e desenvolvedores da nossa plataforma usarão painéis (via API ou Banco Direto) para observar a *Dead Letter Queue* e fazer *replay* das requisições após confirmarem que o cliente B2B corrigiu seu servidor externo offline.

**Cenário Típico:**
A *Atlas Comercial* cria um endpoint `https://api.atlas.com/oms-hook` nas suas configurações via nossa API. Quando a transportadora preenche uma ordem no nosso OMS atualizando-a para `DELIVERED`, em até 2 segundos, o servidor da Atlas recebe o POST com a *payload* reduzida avisando a entrega, e com o HMAC batendo para comprovar a segurança, deduzindo do fluxo o longo *polling* a cada minuto.

## 4. Objetivos e Métricas de Sucesso

- **Velocidade (Latência):** Entregar pelo menos 95% das mensagens em até **10 segundos** de latência após a efetiva troca do status do pedido no OMS.
- **Resiliência (Success Rate):** Ter uma taxa de entrega definitiva em endpoints saudáveis acima de 99.5%, permitindo que os retries programados supram *downtimes* normais da internet.
- **Prazo:** Feature finalizada e validada internamente (incluindo segurança) em um *timebox* de 3 Sprints (novembro, focando o deadline imposto pela Atlas).

## 5. Escopo (In Scope & Out of Scope)

### In Scope
- Estrutura completa de gerenciamento (CRUD) das URLs de webhook pelo cliente, listagem e vinculação aos `status` de pedido escolhidos.
- Sistema transacional no core atual para anexar o Snapshot JSON da payload da notificação no mesmo ciclo do status, garantindo 100% de coerência atômica.
- Worker isolado da API (`src/worker.ts`), com *polling* (busca) a cada 2s para enviar requisições sem atrasar o sistema central.
- Tentativas progressivas (*Backoff Exponencial*) de: 1min, 5min, 30min, 2h e 12h, seguidas de isolamento num DLQ (*Dead Letter Queue*).
- Autenticidade baseada em assinaturas criptográficas rigorosas HMAC-SHA256, com secret gerado na hora (por endpoint) e rotação de secret (grace-period 24h).
- Resgate/replay de DLQ de forma manual (Admin REST Route).
- Histórico de entrega de mensagens (endpoint consumível pelo cliente).

### Out of Scope (Fora de Escopo / Adiados)
- **Envio de alertas por Email (Fallback):** Criar alertas ao e-mail do parceiro caso o Webhook caia na DLQ. Postergado para a Fase 2, visando medir o real impacto antes de alocar recursos.
- **Painel Front-End/Dashboard Visual para o Cliente:** Postergado. Atenderemos provisoriamente via endpoints REST (documentados no *Developer Portal*), para o cliente operar as configs pelas suas próprias ferramentas/APIs.
- **Throttling e Limitação de Requisições de Saída:** O controle estrito e filas restritas por "X Requests per Second" enviadas para o cliente B2B foram postergados. Começaremos reativamente.
- **Garantia Globally-Ordered / Exactly-Once:** Optaremos por *At-least-once* (cliente precisará deduplicar recebimentos com o uuid fixo `X-Event-Id`).

## 6. Requisitos Funcionais

1. **Gestão de Webhooks:**
   - **(RF-01)** O sistema deverá permitir que clientes criem endpoints fornecendo `url` (HTTPS obrigatório) e array `events` com o status do pedido (ex: `['PAID', 'SHIPPED']`).
   - **(RF-02)** A resposta de criação deverá exibir exclusivamente uma `secret` randômica segura gerada pelo server, jamais mostrada repetidamente nas requisições GET por questões de segurança.
   - **(RF-03)** Clientes podem atualizar (PATCH) ou remover (DELETE) webhooks e obter lista paginada (GET) das configurações ativas.
2. **Ciclo do Evento:**
   - **(RF-04)** Uma mudança em um Pedido (Via `OrderService`) criará imediatamente uma linha `PENDING` na `webhook_outbox`.
   - **(RF-05)** O evento inserido conterá os dados imutáveis (*snapshot*) referentes à transição exata do pedido.
3. **Despacho Assíncrono:**
   - **(RF-06)** O processo de envio varrerá o banco, efetuando POST REST às URLs, e portará as chaves customizadas de envio: `X-Event-Id`, `X-Signature` e `X-Timestamp`.
   - **(RF-07)** Se ocorrerem erros (4xx/5xx/Timeout de 10s), o fluxo agenda o *next retry* conforme a escadinha Exponencial configurada (1m...12h) e reincrementa as tentativas até o teto de 5.
4. **DLQ e Histórico:**
   - **(RF-08)** Mensagens estagnadas após 5 falhas devem compor a Dead Letter Queue persistindo logs valiosos (erro e resposta original de falha).
   - **(RF-09)** Usuários com role JWT `ADMIN` podem disparar requisição manual ativando a reinserção do evento morto para a fila viva.
   - **(RF-10)** Clientes poderão solicitar a Rotação de Segredo, validando tanto o segredo velho quanto novo por até 24h de sobreposição.

## 7. Requisitos Não Funcionais

- **Garantia At-Least-Once (RNF-01):** Sem perda de notificação caso haja travamentos. Uso do header `X-Event-Id` único por evento gerado na Outbox para deduplicação no frontend do parceiro.
- **Alta Latência Limiar (RNF-02):** Processamento leve, onde o *Worker Node.js* consulta a fila via *Polling* de não menos e não mais de 2 segundos.
- **Isolamento de Erros e Padrões de Projeto (RNF-03):** Aproveitar 100% o ferramental estabelecido no projeto: `Zod` (schemas estritos p/ as requisições Webhook), Logger universal `Pino`, Central Error Middleware tratando `AppError` com códigos customizados com prefixo `WEBHOOK_`.
- **Carga (RNF-04):** Os corpos (Payloads) não podem ultrapassar 64KB por notificação para evitar saturação. Os detalhes completos de "itens do pedido" serão excluídos; o cliente, se quiser, invoca o GET via ID.
- **Segurança HTTPS Mandatory (RNF-05):** Cadastro de URL não protegida via TLS resultará em erro 400 pelo schema de validação.

## 8. Decisões e Trade-offs Principais

- **MySQL no Padrão Outbox Vs. Mensagerias Externas (Kafka, Redis):** Trade-off positivo pela alta coesão e velocidade (elimina custos operacionais pesados em Cloud para um projeto que resolve tudo atomicamente via `$transaction`).
- **Segredo Individual por Endpoint Vs. Global:** Optado pela granularidade. Caso o parceiro Nova Cargo vaze seu próprio webhook, os webhooks da Atlas Comercial estão a salvo.
- **Polling Loop (2s) Vs. Trigger/Listener MySQL:** Dada as restrições nativas do MySQL, um loop inteligente otimizado em memória/DB consome menos tração do que gambiarras envolvendo leitura cruzada de triggers ou replicação no código da infra Node.
- **At-Least-Once com `X-Event-Id` Vs. Exactly-Once:** Optado por transferir a responsabilidade de deduplicação ao cliente parceiro. Soluções como Exactly-Once requerem commits de duas fases (*2PC*) que oneram sistemas não-distribuídos puros ou requerem filas robustas de handshake.

## 9. Dependências

- Operações via Prisma (ORM). Necessita migrações de novas tabelas (`webhook_configs`, `webhook_outbox`, `webhook_delivery_log`, `webhook_dead_letter`).
- Middleware de Autorização `auth.middleware.ts` para capturar a hierarquia JWT, protegendo as chamadas base dos Webhooks CRUD e isolando a requisição `ADMIN` p/ Replays de DLQ.

## 10. Riscos e Mitigação

- **R-01: Gargalo do Worker travando devido a respostas excessivamente lentas.**
  - **Mitigação:** Aplicação de um *timeout HTTP hard limit* de 10 segundos no momento de chamada do `fetch` na requisição externa. Falhou, entra direto na tabela de Retry/Next_Run.
- **R-02: Vazamento de Secret na Interface.**
  - **Mitigação:** Restrição absoluta de leitura; A chaves de acesso são persistidas apenas embaralhadas, mostradas ao dono uma única vez no momento do Cadastro/Re-Rotação. Revisão da Engenheira de Segurança antes do Deploy (Agendado).
- **R-03: Excesso do Volume Outbox (Base gorda).**
  - **Mitigação:** Processamento filtrado por indexação de Datas, eliminando o consumo das "Antigas". A expurgação (`Archival`) pode ser gerada posteriormente sem colidir com as sprints.

## 11. Critérios de Aceitação

- **CA-01:** O webhook é criado via POST JSON para HTTPS com resposta 201 e secret única retornada em plain text.
- **CA-02:** Um pedido que evolui para SHIPPED insere imediatamente o evento transacional imutável via API no MySQL. O worker acorda em ~2s, envia via requisição REST e registra no `webhook_delivery_log` em < 10s.
- **CA-03:** Enviar intencionalmente o tráfego a uma URL inacessível/fantasma forçará as 5 tentativas programadas de retentativas. Uma sexta passagem no Polling ignora o evento atirando-o no registro da `webhook_dead_letter`.
- **CA-04:** Enviar Header Assinado (`X-Signature` HMAC-SHA256) validando o Body na ponta contrária usando a secret gerada com 100% de match.
- **CA-05:** Endpoints Admin são unicamente passíveis de uso portando um JWT válido associado à Tag Role: `ADMIN`.
- **CA-06:** Rotação de chaves responde a requisições autênticas usando a assinatura antiga e a assinatura nova simultaneamente num intervalo limite de 24 horas.

## 12. Estratégia de Testes e Validação

- **Testes Unitários:** Para a formatação das tabelas e criação de Segredos via Crypto Hmac; Teste das regras de Backoff exponencial isoladamente (Simulando 1 a 5 tentativas caindo na linha exata de agendamento em cronologia).
- **Testes de Integração:** Uso estrito do Supertest integrado ao Vitest para disparar fluxo contínuo "Cria Webhook -> Update de Ordem -> Varredura no BD verificando Output Assíncrono -> Validar Header de Entrega simulada".
- **Revisão de Segurança (Manual/Check):** 2 dias antes do Deploy, engenheira averigua os limites de Payload, verificação do Segredo Hasheado por fora e as premissas de rotatividade com grace period.