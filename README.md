# Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre o desafio

Neste projeto, a tarefa foi construir um conjunto completo e maduro de especificações de design (PRD, RFC, FDD, ADRs) focados em uma arquitetura de notificação via **Webhooks Outbound**. Partindo do zero absoluto — sendo fornecidos apenas um repositório boilerplate (em Node.js) e uma transcrição crua de uma reunião remota de planejamento —, o objetivo foi extrair as entrelinhas e decisões arquiteturais de forma inteligente e traduzir esse caos conversacional em documentação técnica acionável para times de engenharia.

O desafio provou como podemos utilizar modelos de inteligência artificial de fronteira não apenas como meros geradores de texto, mas como **Maestros** (Agentes Autônomos de Análise), capazes de varrer transcrições verbais não estruturadas e ligar seus pontos a fragmentos exatos e isolados num código-fonte (referenciando middlewares, padrões de schema do Zod, estrutura modular MVC) sem sofrer alucinações.

## Ferramentas de IA utilizadas

- **Claude 5 (Fable 5) / Claude Code CLI:** O orquestrador central de todo o desafio. Empregado em um fluxo contínuo na CLI, lendo nativamente o terminal, inspecionando o repositório, lendo e compreendendo a transcrição. Foram aproveitados agentes background simultâneos (`Explore agent`) que se dividiram entre varrer o código, varrer o `TRANSCRICAO.md` e organizar mapas mentais antes mesmo de redigir a primeira linha documentada de PRDs ou ADRs.
- **Agent Tooling:** Uso assíncrono de instâncias virtuais do próprio Claude para ler arquivos concorrentemente e emitir "Sumários".

## Workflow adotado

A construção não foi linear (do PRD para o FDD), mas adotou uma cadeia de causa-e-efeito lógica baseada nas decisões concretas. O fluxo de trabalho foi o seguinte:

1. **Reconhecimento (Recon / Discovery):**
   Gatilho inicial que forçou a CLI a ler e entender o core do repositório (`package.json`, `.prisma`, `app.ts` e arquivos modulares como `order.service.ts`). O objetivo era aclimatar o Claude com os padrões da casa.

2. **Delegação Assíncrona via Agentes:**
   Para extrair a carga imensa da `TRANSCRICAO.md`, criei agentes em Background, ordenando: *"Leia as 200 linhas da transcrição e crie-me esqueletos puros propostos para ADR, FDD, PRD e RFC, focando excludentes lógicos"*. Eles geraram os recortes perfeitos de onde extrairíamos cada ponto no `TRACKER`.

3. **Arquitetura antes de Produto (ADRs -> RFC):**
   Com as "Decisões" puras mapeadas (Ex: *Outbox MySQL*, *HMAC-SHA256*, *Retry com Backoff*, etc.), os **7 ADRs** foram os primeiros a nascer, servindo de cimento sólido para o resto.
   A reboque dos ADRs, redigimos a **RFC** formalizando o direcionamento técnico.

4. **Especificação de Máquina (FDD):**
   Com o consenso montado na RFC, mergulhamos no **FDD**, criando ligações diretas do webhook recém desenhado com os arquivos pré-existentes listados na Fase 1 (`auth.middleware.ts`, `error.middleware.ts`, `$transaction`).

5. **A Capa do Livro (PRD e TRACKER):**
   Focamos no Business (PRD), transformando as amarrações em *Customer Goals*. E, por fim, catalogamos cada gota dos 4 passos anteriores na tabela central do `TRACKER.md`.

## Prompts customizados

A principal chave do sucesso nas iterações de Agentes foi isolar seus propósitos:

**Prompt 1 (Extração de Decisões e ADRs via Agente):**
```text
Based on TRANSCRICAO.md, identify the architectural decisions made and create an outline for 5-8 ADRs. Look for decisions regarding: Outbox Pattern, Polling vs Event Listeners for Worker, Retry Policy/DLQ, HMAC-SHA256 for Security, At-least-once delivery with X-Event-Id, Reusing existing AppError/Pino/Zod patterns. Make sure to identify alternatives and consequences for each. Output a markdown list of ADR titles and a brief summary of Context, Decision, Alternatives, and Consequences for each.
```

**Prompt 2 (Extração Refinada de FDD c/ Ligação de Código Real):**
```text
Analyze TRANSCRICAO.md and the source code (especially src/modules/orders/order.service.ts and src/middlewares/). Outline the FDD (Feature Design Document). Include details on the Webhook flow (outbox, worker, retry, dlq), the API contracts (create webhook, list webhooks, list deliveries, replay DLQ), Webhook Payload and Headers, Error Matrix with WEBHOOK_ prefix, and critically: **Integration with existing system** pointing to at least 4 specific files/patterns (e.g., how src/modules/orders/order.service.ts's changeStatus will be modified, reusing src/shared/errors/app-error.ts, src/middlewares/error.middleware.ts, and src/middlewares/auth.middleware.ts). Output a markdown outline.
```

## Iterações e ajustes

Apesar do modelo Fable 5 ser denso e profundo, o texto original requer um olhar cirúrgico para aparar sobras (as famosas "alucinações controladas"):

1. **Ajuste Fino na Abordagem Síncrona vs Assíncrona:** A IA, por vezes, num rascunho primário, mesclava o processamento assíncrono do Worker com o tempo de requisição atômico do `OrderService.changeStatus`, assumindo um delay na própria transação do banco. Tive que impor que ela focasse no modelo `Outbox` não para "salvar dados", mas para salvar "Snapshots" e descolar completamente a fila transacional do delay imposto pelo HTTP do endpoint alvo.
2. **Correção sobre E-mails:** Em uma primeira leitura, o rascunho inferiu que faríamos um sistema de fallback via e-mail. Isso contraria as decisões de finalização da transcrição (minuto [09:37] onde Larissa decide deixar p/ fase 2). Apontei rigidamente nos prompts subsequentes que o e-mail não entraria no PRD/FDD, forçando o enquadramento na seção `Out of Scope`.
3. **Limitação de Ações Manuais:** Ao detalhar o Endpoint de `Replay DLQ`, a IA precisou de uma refatoração no comando para relembrar que as roles do sistema já existiam (`requireRole('ADMIN')` provenientes do código atual) — ela precisou ser impedida de "inventar" um novo padrão de RBAC se havia um perfeitamente funcional em `src/middlewares`.

## Como navegar a entrega

Os arquivos gerados encontram-se estruturados em diretórios coerentes. Recomenda-se a leitura fluída através da seguinte rota (do baixo pro alto nível arquitetural e encerramento em alto nível de produto):

1. Vá à pasta `/docs/adrs/` e consuma as decisões nucleares (ADR-001 a ADR-007).
2. Siga para `/docs/RFC.md`, englobando os consensos perante a equipe e os trade-offs.
3. Atravesse o `/docs/FDD.md` (Design de Features), para entender o mapeamento mecânico que une o Webhook às veias atuais do Monolito.
4. Feche a compreensão de Produto/Business lendo o `/docs/PRD.md`.
5. Valide todas as afirmações contidas nos 4 passos acima checando a origem de cada sentença na tabela cruzada em `/docs/TRACKER.md`.