# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Status** | Rascunho para aprovação |
| **Data** | 2026-11-06 |
| **Responsável de produto** | Marcos (PM) |
| **Relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [TRACKER](TRACKER.md) |

> Itens com ID (ex.: `PRD-FR-01`) são rastreados no [TRACKER](TRACKER.md). Itens marcados **[Proposta]** são sugestões de redação/meta sem número ou decisão explícita na reunião.

## 1. Resumo e contexto

O Order Management System (OMS) passará a **notificar clientes B2B por webhook, em tempo real, sempre que o status de um pedido mudar**. Hoje o OMS não tem nenhum mecanismo de notificação externa; os clientes consultam `GET /orders` repetidamente. A feature é de saída (OMS → cliente) ([09:02] Marcos).

## 2. Problema e motivação

- Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram formalmente notificação em tempo real ([09:00] Marcos).
- O polling em `GET /orders` torna a integração lenta e cara para eles ([09:00] Marcos).
- Risco comercial: a Atlas sinalizou possível migração para o concorrente se não houver entrega até o fim do trimestre ([09:00] Marcos); a Atlas quer a feature para o fim de novembro ([09:45] Marcos).

## 3. Público-alvo e cenários de uso

**Público:** integradores técnicos de clientes B2B que cadastram webhooks pela API do OMS (autenticados com usuários que representam o cliente, [09:32] Marcos); operadores/ADMIN do OMS que acompanham falhas.

| Cenário | Descrição |
|---|---|
| C-01 | O cliente cadastra uma URL https e escolhe receber só `SHIPPED` e `DELIVERED` ([09:33] Marcos) |
| C-02 | Um pedido vira `SHIPPED`; o cliente recebe o POST assinado em menos de 10 s |
| C-03 | O cliente está em manutenção por 2 h; o sistema reenvia automaticamente ([09:16] Diego) |
| C-04 | Um cliente recebe o mesmo evento duas vezes e deduplica por `X-Event-Id` ([09:25] Diego) |
| C-05 | O cliente suspeita de secret vazada e a rotaciona, com 24 h de convivência ([09:21] Sofia) |
| C-06 | O cliente consulta as últimas 100 entregas para depurar ([09:34] Marcos) |
| C-07 | Um ADMIN reprocessa um evento que foi à DLQ ([09:18] Diego) |

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
|---|---|---|---|---|
| PRD-OBJ-01 | Notificação em "tempo real" | Tempo entre a mudança de status e a requisição ao cliente | **< 10 segundos** (polling de 2 s) | [09:02] Marcos, [09:10] Larissa |
| PRD-OBJ-02 | Entregar a tempo de reter os clientes | Feature em produção | Fim de novembro, em 3 sprints, com revisão de segurança incluída | [09:45] Marcos, [09:47] Larissa |
| PRD-OBJ-03 | Não perder eventos | Mudanças de status com webhook interessado e sem evento registrado | **0** (atomicidade na transação) | [09:40] Bruno |
| PRD-OBJ-04 | Cobrir indisponibilidade do cliente | Janela de retry automático | **~15 horas**, 5 tentativas | [09:17] Diego |
| PRD-OBJ-05 | Atender os clientes solicitantes **[Proposta]** | Clientes recebendo webhook | 3 de 3 (Atlas, MaxDistribuição, Nova Cargo) | [09:00] Marcos |

A meta de redução de chamadas a `GET /orders` não foi quantificada na reunião; sugere-se medir após o lançamento.

## 5. Escopo

### Incluso
Cadastro, edição, remoção e listagem de webhooks; filtro de status por webhook; entrega assinada com retry e DLQ; rotação de secret; histórico de entregas; replay administrativo de DLQ.

### Fora de escopo (descartado ou adiado na reunião)

| ID | Item | Situação | Origem |
|---|---|---|---|
| PRD-OOS-01 | E-mail ao cliente quando o webhook falha | Fora desta fase; talvez próxima, após medir impacto | [09:37] Larissa |
| PRD-OOS-02 | Rate limiting de saída por cliente | Observar e decidir depois | [09:39] Larissa |
| PRD-OOS-03 | Dashboard/painel visual do cliente | Fora; projeto separado do frontend | [09:40] Larissa |
| PRD-OOS-04 | Arquivamento de eventos entregues (~30 dias) | Fora desta feature | [09:08] Diego |
| PRD-OOS-05 | Múltiplos workers e ordenação global | Adiado; limitação conhecida | [09:13] Diego, [09:13] Larissa |
| PRD-OOS-06 | Webhooks de entrada (cliente → OMS) | Fora: só saída | [09:02] Marcos |
| PRD-OOS-07 | Exactly-once | Descartado em favor de at-least-once | [09:25] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-FR-01 | Cadastrar webhook via `POST` informando `url` e lista de status desejados; a secret é gerada pelo sistema e devolvida na criação | [09:31] Marcos |
| PRD-FR-02 | O `customer_id` é informado no body ou path; não vem do JWT | [09:32] Larissa |
| PRD-FR-03 | Editar (PATCH), remover (DELETE) e listar (GET) os webhooks de um customer | [09:33] Bruno |
| PRD-FR-04 | Escolher por webhook quais status receber; o filtro é aplicado na inserção na outbox | [09:33] Marcos, [09:34] Bruno |
| PRD-FR-05 | Consultar histórico de entregas (últimos 100, com sucesso/falha, payload, resposta e tempo) em `GET /webhooks/:id/deliveries` | [09:34] Marcos |
| PRD-FR-06 | Reprocessar evento da DLQ via `POST /admin/webhooks/dead-letter/:id/replay`, exclusivo de ADMIN, com registro de quem executou | [09:18] Diego, [09:36] Sofia |
| PRD-FR-07 | Rotacionar a secret pela API; a anterior segue válida por 24 h | [09:21] Sofia |
| PRD-FR-08 | Registrar um evento a cada mudança de status de pedido, atomicamente com a mudança | [09:40] Bruno |
| PRD-FR-09 | Assinar toda entrega com HMAC-SHA256 do corpo, usando secret exclusiva do endpoint | [09:20] Sofia, [09:21] Sofia |
| PRD-FR-10 | Reentregar com backoff 1m/5m/30m/2h/12h; esgotado, mover o evento para a DLQ | [09:17] Larissa |
| PRD-FR-11 | Enviar headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` | [09:44] Diego, [09:44] Sofia |
| PRD-FR-12 | Aceitar apenas URLs https | [09:23] Sofia |
| PRD-FR-13 | O CRUD de configuração é aberto a qualquer role autenticada | [09:36] Marcos, [09:37] Sofia |
| PRD-FR-14 | Payload JSON enxuto, sem `items` | [09:43] Diego |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-NFR-01 | Latência < 10 s fim a fim | [09:02] Marcos |
| PRD-NFR-02 | Entrega at-least-once, dedup do cliente por `X-Event-Id` | [09:24] Diego |
| PRD-NFR-03 | Timeout de 10 s por chamada | [09:42] Diego |
| PRD-NFR-04 | Payload máximo de 64 KB; acima, não é enviado | [09:23] Sofia, [09:24] Diego |
| PRD-NFR-05 | Secret única por endpoint | [09:21] Sofia |
| PRD-NFR-06 | TLS obrigatório | [09:23] Sofia |
| PRD-NFR-07 | Ordenação garantida só por `order_id` e com um único worker | [09:13] Larissa |
| PRD-NFR-08 | Worker em processo separado da API | [09:11] Diego |
| PRD-NFR-09 | Reuso dos padrões do projeto (módulos, `AppError`, Pino, códigos `WEBHOOK_*`) | [09:30] Larissa |
| PRD-NFR-10 | Revisão de segurança (≥2 dias úteis) antes do deploy | [09:46] Sofia |

## 8. Decisões e trade-offs principais

Cada decisão está detalhada no ADR correspondente.

| Decisão | Trade-off aceito | ADR |
|---|---|---|
| Outbox no MySQL | Sem infra nova, em troca de carga extra no MySQL e de a falha na outbox reverter o status | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| 5 tentativas com backoff e DLQ | Cobre ~15 h; além disso exige replay manual | [ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md) |
| HMAC-SHA256, secret por endpoint, rotação 24 h | Segurança por cliente, com custo de gerir secrets | [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` | Simplicidade, com dedup a cargo do cliente | [ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md) |
| Worker separado, polling de 2 s | Latência de até 2 s e sem ordenação global | [ADR-005](adrs/ADR-005-worker-separado-em-polling.md) |
| Reuso dos padrões do projeto | Herda limitações atuais | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Publicação dentro da transação | Atomicidade com acoplamento ao `OrderService` | [ADR-007](adrs/ADR-007-publish-webhook-event-na-transacao.md) |
| Payload enxuto e limites | Cliente consulta `GET /orders/:id` para detalhes | [ADR-008](adrs/ADR-008-payload-headers-e-limites-de-entrega.md) |

## 9. Dependências

- Marcos documentar no portal de desenvolvedor o modelo at-least-once e como integrar ([09:26] Marcos, [09:40] Marcos).
- Sofia: agendar revisão de segurança antes do deploy ([09:49] Sofia).
- Sessão de revisão do design com Bruno e Diego antes de codar ([09:50] Larissa).
- Técnicas: MySQL e Prisma existentes, sem infraestrutura nova; novo processo `worker` a ser operado junto da API ([09:11] Diego).
- Painel do cliente depende do time de frontend, fora desta feature ([09:40] Larissa).

## 10. Riscos e mitigação

| ID | Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|---|
| PRD-RSK-01 | Prazo de 3 sprints não cumprir o fim de novembro e a Atlas migrar para o concorrente | Média | Alto | Priorizar outbox, worker e HMAC; reservar os 2 dias de revisão da Sofia; comunicar prazo aos clientes ([09:47] Marcos) |
| PRD-RSK-02 | Falha ao inserir na outbox reverte a mudança de status | Baixa | Alto | Insert simples e testes ponta a ponta; comportamento intencional ([09:40] Bruno) |
| PRD-RSK-03 | Vazamento de secret (cliente já vazou em log) | Média | Alto | Secret por endpoint, rotação com grace, redação em logs, revisão de segurança ([09:22] Diego) |
| PRD-RSK-04 | Cliente processa eventos duplicados por não deduplicar | Média | Médio | Documentação destacada no portal; `X-Event-Id` ([09:26] Marcos) |
| PRD-RSK-05 | Cliente lento atrasa outros por worker único | Média | Médio | Timeout de 10 s; monitorar idade do evento mais antigo |
| PRD-RSK-06 | Crescimento da outbox sem arquivamento | Média | Baixo | Índices em status e `created_at` ([09:08] Diego); arquivamento em feature futura |
| PRD-RSK-07 | Bombardeio de um cliente em picos (sem rate limit) | Baixa | Médio | Observar e decidir depois ([09:39] Diego) |

## 11. Critérios de aceitação

- CA-01: Cadastro com URL https e lista de status retorna a secret uma única vez; URL `http` é recusada.
- CA-02: Toda mudança de status de pedido com webhook interessado gera 1 evento por endpoint; sem interessados, nenhum evento.
- CA-03: Cliente recebe o evento em menos de 10 s no cenário sem falha, com os 4 headers de identificação e assinatura.
- CA-04: Evento indisponível é reentregue conforme 1m/5m/30m/2h/12h e, esgotado, aparece na DLQ.
- CA-05: ADMIN reprocessa DLQ; OPERATOR recebe 403; o log registra quem executou.
- CA-06: Após rotação, a secret antiga deixa de valer em 24 h.
- CA-07: `GET /webhooks/:id/deliveries` mostra as últimas 100 entregas com resultado, payload, resposta e tempo.
- CA-08: Nenhuma funcionalidade fora de escopo (e-mail, rate limiting, painel) está presente.

## 12. Estratégia de testes e validação

- **Integração (Vitest + Supertest, padrão de `tests/orders.test.ts`):** `changeStatus` gera/rollback de outbox; filtro de status; CRUD; autorização ADMIN; validação https.
- **Unitários:** assinatura HMAC, cálculo de backoff, validação de tamanho 64 KB, máquina de estados da outbox.
- **Ponta a ponta com receptor simulado:** sucesso, timeout (10 s), 5xx, indisponibilidade e esgotamento até a DLQ; replay.
- **Resiliência:** reiniciar a API sem afetar o worker; crash do worker com evento `PROCESSING`.
- **Segurança:** revisão manual de Sofia de HMAC, geração e rotação de secret, e de vazamento em logs.
- **Validação com clientes:** homologação com ao menos um dos 3 clientes antes do lançamento **[Proposta]**.
