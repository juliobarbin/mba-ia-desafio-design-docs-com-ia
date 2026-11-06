# Da Reunião ao Documento: Design Docs do Sistema de Webhooks de Notificação de Pedidos

> O enunciado original do desafio está no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia). Este README documenta o **processo de produção** do pacote de design docs.

## Sobre o desafio

A partir da transcrição de uma reunião técnica de ~55 minutos (`TRANSCRICAO.md`) e do código de um Order Management System em Node.js + TypeScript + Prisma/MySQL, o objetivo foi produzir a documentação técnica de uma feature ainda inexistente: **webhooks de notificação de mudança de status de pedidos**. O pacote contém PRD, RFC, FDD, 8 ADRs e um Tracker de rastreabilidade, de modo que o time possa começar a implementar sem depender da memória da reunião.

A entrega é puramente documental: `src/`, `prisma/`, `tests/` e configurações **não foram alterados**. A regra central foi que nenhum requisito, decisão ou restrição entra sem origem identificável na transcrição ou no código, e que o que foi descartado ou adiado na reunião (e-mail, rate limiting, dashboard, arquivamento, multi-worker) fica explicitamente **fora** do escopo.

## Ferramentas de IA utilizadas

- **Claude Code (modelo Sonnet 5.5)**: ferramenta principal. Leu o repositório e a transcrição, produziu os documentos, e rodou scripts de verificação (existência de arquivos, timestamps e falantes do tracker).
- **Skill `caveman` (modo de resposta)**: usada apenas para deixar as respostas do chat enxutas; **não** afeta o conteúdo dos documentos.

## Workflow adotado

1. **Fork/clone** do repositório base e leitura integral da `TRANSCRICAO.md`.
2. **Exploração do código** guiada pelos ganchos citados na reunião: `order.service.ts` (`changeStatus`), `order.status.ts`, `schema.prisma`, erros em `src/shared/errors`, middlewares de auth/erro/validação, `app.ts`, `routes/index.ts`, `server.ts`, logger Pino.
3. **ADRs primeiro** (ADR-001 a 008): cada decisão fechada na reunião, com alternativas e consequências.
4. **RFC** sobre as decisões: alternativas descartadas, questões em aberto, links para ADRs. Conciso, sem repetir o FDD.
5. **FDD**: modelo de dados, 7 endpoints + contrato de saída, fluxos, matriz `WEBHOOK_*`, resiliência, observabilidade e a seção "Integração com o sistema existente".
6. **PRD** por último, consolidando o que os outros documentos já fixavam.
7. **Tracker** montado ao final, varrendo os documentos; depois **validado por script**.
8. **Revisão** contra a checklist de critérios de aceite.

Convenção para não inventar: informação sem decisão explícita na reunião é marcada **[Proposta]** nos documentos e consolidada em `FDD-PROP-01` no tracker.

## Prompts customizados

**Prompt 1: classificação do que entra e do que não entra (filtragem dirigida)**

```text
Leia TRANSCRICAO.md inteira. Produza uma tabela com 4 categorias, citando sempre [hh:mm] Falante:
(a) decisões FECHADAS (alguém disse "decidido/fechado/anotado" ou houve consenso);
(b) requisitos funcionais explícitos;
(c) itens DESCARTADOS ou ADIADOS (e-mail, rate limiting, dashboard, arquivamento, multi-worker...);
(d) pontos AMBÍGUOS ou não decididos.
Não infira nada sem citação. Se algo parecer decisão mas ninguém fechou, coloque em (d).
Depois liste divergências entre a transcrição e o código (ex.: o que a fala assume que existe e não existe).
```

**Prompt 2: geração de ADR com rastreabilidade obrigatória**

```text
Escreva docs/adrs/ADR-00N-<kebab>.md no formato MADR com Status, Contexto, Decisão,
Alternativas Consideradas e Consequências (positivas E negativas, com trade-off explícito).
Regras: (1) cada afirmação da seção Decisão termina com [hh:mm] Falante da transcrição;
(2) alternativas só se foram discutidas na reunião, ou marcadas como "plausível, não discutida";
(3) cite arquivos reais do repo (verifique com ls antes); (4) não invente números, nomes de campos
ou bibliotecas; se faltar informação, registre como questão em aberto em vez de decidir.
```

**Prompt 3: auditoria do Tracker**

```text
Para cada linha de docs/TRACKER.md, verifique: se Fonte=CODIGO o caminho existe; se Fonte=TRANSCRICAO
o par [hh:mm] Falante existe literalmente em TRANSCRICAO.md. Depois varra PRD/RFC/FDD/ADRs procurando
timestamps, caminhos e códigos WEBHOOK_* citados que não estejam na transcrição, no código ou marcados [Proposta].
```

## Iterações e ajustes

Foram **três ciclos** principais (geração, auditoria, correção); o que mudou concretamente:

1. **Falha de payload > 64 KB.** A primeira versão do FDD verificava o limite dentro de `publishWebhookEvent`, ou seja, um evento anômalo faria **rollback da mudança de status** do pedido, algo que a reunião não decidiu e que seria desproporcional. Corrigi para verificar no **envio** (worker): não envia, `WEBHOOK_PAYLOAD_TOO_LARGE`, sem retry, vai à DLQ, coerente com "a gente não envia" ([09:23] Sofia).
2. **Ambiguidade das "5 tentativas".** A fala lista 5 intervalos (1m/5m/30m/2h/12h) e fala em ~15 h, mas diz "5 tentativas". A soma só fecha com tentativa inicial + 5 retries. Em vez de escolher em silêncio, registrei no ADR-002 e como questão em aberto (RFC-OQ-03).
3. **Observabilidade sem origem.** A reunião só cita logs Pino e auditoria do replay, e o código não tem métricas nem tracing. Em vez de apresentá-las como requisito, marquei métricas/tracing como **[Proposta]** no FDD, ancorei em logs e `requestId` existentes e registrei que `redactPaths` do logger não cobre `secret`.
4. **Auditoria do Tracker por script.** O script de validação apontou que uma citação no ADR-008 atribuía `[09:45]` a "Sofia/Diego"; na transcrição, a fala de 09:45 é do Diego. Corrigido. Todas as linhas do tracker têm caminho existente ou par `[hh:mm] Falante` real.
5. **Lacunas que a transcrição deixa:** proteção da secret em repouso, assinatura durante o grace de 24 h e evento na criação do pedido viraram questões em aberto no RFC em vez de decisões fabricadas.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [`docs/PRD.md`](docs/PRD.md): por que e o quê (problema, escopo, métricas, riscos)
2. [`docs/RFC.md`](docs/RFC.md): proposta técnica, alternativas e questões em aberto
3. [`docs/adrs/`](docs/adrs/): decisões isoladas
   - [ADR-001 Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
   - [ADR-002 Retry, backoff e DLQ](docs/adrs/ADR-002-retry-backoff-e-dlq.md)
   - [ADR-003 HMAC-SHA256 e secret por endpoint](docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
   - [ADR-004 At-least-once e X-Event-Id](docs/adrs/ADR-004-at-least-once-com-x-event-id.md)
   - [ADR-005 Worker separado em polling](docs/adrs/ADR-005-worker-separado-em-polling.md)
   - [ADR-006 Reuso dos padrões do projeto](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
   - [ADR-007 publishWebhookEvent na transação](docs/adrs/ADR-007-publish-webhook-event-na-transacao.md)
   - [ADR-008 Payload, headers e limites](docs/adrs/ADR-008-payload-headers-e-limites-de-entrega.md)
4. [`docs/FDD.md`](docs/FDD.md): como construir, em detalhe
5. [`docs/TRACKER.md`](docs/TRACKER.md): de onde veio cada item

A transcrição original permanece em [`TRANSCRICAO.md`](TRANSCRICAO.md), sem alterações.
