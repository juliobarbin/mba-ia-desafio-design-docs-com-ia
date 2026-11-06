# TRACKER — Rastreabilidade dos documentos

Mapeia cada item dos documentos ao trecho da [TRANSCRICAO](../TRANSCRICAO.md) ou ao código.
Total de linhas: **206**; Fonte TRANSCRICAO: **167** (81%); Fonte CODIGO: **39**.

Itens marcados **[Proposta]** nos documentos são detalhamento de implementação sem decisão explícita na reunião (por exemplo, critério de sucesso 2xx, nomes de tabelas auxiliares e métricas): ficam consolidados em `FDD-PROP-01` e devem ser validados na sessão de revisão ([09:50] Larissa).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Clientes B2B Atlas, MaxDistribuição e Nova Cargo pedem notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling em GET /orders, integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | Risco de a Atlas migrar para concorrente sem entrega até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Restrição | Fluxo apenas de saída (OMS para cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-05 | docs/PRD.md | Contexto | OMS atual não possui eventos, filas ou webhooks | CODIGO | src/server.ts |
| PRD-CTX-06 | docs/PRD.md | Contexto | Clientes cadastram via API com usuários do sistema que os representam | TRANSCRICAO | [09:32] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Notificação em menos de 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Entrega em 3 sprints para fim de novembro | TRANSCRICAO | [09:47] Larissa |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Zero mudanças de status sem evento (atomicidade) | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Janela de retry de ~15 h com 5 tentativas | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | Atender os 3 clientes solicitantes | TRANSCRICAO | [09:00] Marcos |
| PRD-C-01 | docs/PRD.md | Cenário | Cliente escolhe só SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-C-03 | docs/PRD.md | Cenário | Cliente em manutenção de 2 h recebe reenvio | TRANSCRICAO | [09:16] Diego |
| PRD-C-04 | docs/PRD.md | Cenário | Cliente deduplica por X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-C-05 | docs/PRD.md | Cenário | Rotação de secret com 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-C-06 | docs/PRD.md | Cenário | Cliente consulta últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-C-07 | docs/PRD.md | Cenário | ADMIN reprocessa evento da DLQ | TRANSCRICAO | [09:18] Diego |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail ao cliente na falha, fora desta fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Rate limiting de saída, observar | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Dashboard visual fora de escopo | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues fora da feature | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Multi-worker e ordenação global adiados | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Webhooks de entrada fora de escopo | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com url e status; secret gerada e devolvida | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | customer_id no body/path, não no JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | PATCH, DELETE e GET de webhooks por customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Filtro de status por webhook, aplicado na inserção | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | GET /webhooks/:id/deliveries com últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Replay de DLQ por ADMIN com log de auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace de 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Evento registrado atomicamente a cada mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 com secret por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Retry 1m/5m/30m/2h/12h e DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:45] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Somente URLs https | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | CRUD aberto a qualquer role autenticada | TRANSCRICAO | [09:37] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Payload enxuto sem items | TRANSCRICAO | [09:43] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | At-least-once com dedup por X-Event-Id | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por chamada | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Payload máximo de 64 KB | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Ordenação só por order_id e com worker único | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Reuso de padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança de ≥2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-01 | docs/PRD.md | Dependência | Marcos documenta at-least-once e integração no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-02 | docs/PRD.md | Dependência | Sofia deve ser agendada para revisão de segurança | TRANSCRICAO | [09:49] Sofia |
| PRD-DEP-03 | docs/PRD.md | Dependência | Sessão de revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| PRD-DEP-04 | docs/PRD.md | Dependência | Painel do cliente depende do time de frontend, fora de escopo | TRANSCRICAO | [09:40] Larissa |
| PRD-RSK-01 | docs/PRD.md | Risco | Prazo de 3 sprints vs fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-RSK-02 | docs/PRD.md | Risco | Falha na outbox reverte a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-RSK-03 | docs/PRD.md | Risco | Vazamento de secret, já ocorrido em cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RSK-04 | docs/PRD.md | Risco | Cliente processa duplicados | TRANSCRICAO | [09:25] Sofia |
| PRD-RSK-05 | docs/PRD.md | Risco | Cliente lento com worker único | TRANSCRICAO | [09:42] Diego |
| PRD-RSK-06 | docs/PRD.md | Risco | Crescimento da outbox sem arquivamento | TRANSCRICAO | [09:08] Diego |
| PRD-RSK-07 | docs/PRD.md | Risco | Bombardeio sem rate limit em picos | TRANSCRICAO | [09:38] Diego |
| PRD-TST-01 | docs/PRD.md | Estratégia de testes | Testes de integração seguem o padrão de tests/orders.test.ts | CODIGO | tests/orders.test.ts |
| RFC-META-01 | docs/RFC.md | Metadados | Revisores: Marcos, Bruno, Diego e Sofia (participantes da reunião) | TRANSCRICAO | [09:00] Larissa |
| RFC-PROP-01 | docs/RFC.md | Decisão | Insert de outbox via publishWebhookEvent(tx) dentro de changeStatus | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker src/worker.ts com polling de 2 s | TRANSCRICAO | [09:11] Larissa |
| RFC-PROP-03 | docs/RFC.md | Decisão | Retry, backoff e DLQ | TRANSCRICAO | [09:17] Larissa |
| RFC-PROP-04 | docs/RFC.md | Decisão | HMAC-SHA256, secret por endpoint e rotação 24 h | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-05 | docs/RFC.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | changeStatus já executa transação pesada (pedido, histórico, estoque) | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-02 | docs/RFC.md | Contexto | Transação existente em OrderService.changeStatus | CODIGO | src/modules/orders/order.service.ts |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | HTTP síncrono no OrderService | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams ou fila equivalente | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco para acionar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | 3 tentativas ou retry indefinido | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-OQ-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída em aberto | TRANSCRICAO | [09:39] Larissa |
| RFC-OQ-03 | docs/RFC.md | Questão em aberto | Contagem das tentativas ambígua (5 tentativas vs 5 intervalos) | TRANSCRICAO | [09:17] Diego |
| RFC-OQ-04 | docs/RFC.md | Questão em aberto | Assinatura durante o grace period não definida | TRANSCRICAO | [09:22] Sofia |
| RFC-OQ-05 | docs/RFC.md | Questão em aberto | Evento na criação do pedido não discutido | TRANSCRICAO | [09:40] Bruno |
| RFC-OQ-06 | docs/RFC.md | Questão em aberto | Escalar para múltiplos workers adiado | TRANSCRICAO | [09:13] Diego |
| RFC-OQ-07 | docs/RFC.md | Questão em aberto | Endurecer autorização do CRUD no futuro | TRANSCRICAO | [09:37] Sofia |
| RFC-OQ-02 | docs/RFC.md | Questão em aberto | Proteção da secret em repouso não discutida; secret é campo de texto no schema | CODIGO | prisma/schema.prisma |
| RFC-IMP-01 | docs/RFC.md | Impacto | Registro do novo módulo em app.ts e routes/index.ts | CODIGO | src/app.ts |
| RFC-RSK-01 | docs/RFC.md | Risco | Prazo de 3 sprints para fim de novembro | TRANSCRICAO | [09:46] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox no MySQL, atomicidade com a mudança de status | TRANSCRICAO | [09:08] Larissa |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h e DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, grace de 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004 | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-005 | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Worker separado em polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso dos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-publish-webhook-event-na-transacao.md | Decisão | publishWebhookEvent(tx, ...) dentro da transação | TRANSCRICAO | [09:41] Bruno |
| ADR-008 | docs/adrs/ADR-008-payload-headers-e-limites-de-entrega.md | Decisão | Payload sem items, headers, timeout 10 s e limite de 64 KB | TRANSCRICAO | [09:44] Diego |
| ADR-001-a | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | UUID como chave da outbox, padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-001-b | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Payload renderizado como snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-001-c | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Índices em status e created_at; arquivamento fora de escopo | TRANSCRICAO | [09:08] Diego |
| ADR-001-d | docs/adrs/ADR-001-outbox-no-mysql.md | Referência a código | IDs UUID Char(36) em todos os modelos | CODIGO | prisma/schema.prisma |
| ADR-002-a | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | Replay manual por endpoint admin | TRANSCRICAO | [09:18] Diego |
| ADR-002-b | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Restrição | Já houve cliente com 2 h de indisponibilidade planejada | TRANSCRICAO | [09:16] Diego |
| ADR-003-a | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | Rotação de secret via API | TRANSCRICAO | [09:21] Sofia |
| ADR-003-b | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Restrição | HTTPS obrigatório por validação Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-003-c | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Referência a código | Logger sem redação do campo secret | CODIGO | src/shared/logger/index.ts |
| ADR-004-a | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Decisão | UUID do evento gerado na entrada da outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-a | docs/adrs/ADR-005-worker-separado-em-polling.md | Restrição | Ordenação por order_id apenas com um worker | TRANSCRICAO | [09:13] Larissa |
| ADR-005-b | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | PrismaClient próprio no worker | TRANSCRICAO | [09:30] Bruno |
| ADR-005-c | docs/adrs/ADR-005-worker-separado-em-polling.md | Referência a código | Entry point atual da API usado como modelo para o worker | CODIGO | src/server.ts |
| ADR-006-a | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo em src/modules/webhooks | TRANSCRICAO | [09:27] Bruno |
| ADR-006-b | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Erros WEBHOOK_* estendendo AppError | TRANSCRICAO | [09:28] Bruno |
| ADR-006-c | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Referência a código | Hierarquia AppError e erros HTTP existentes | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-d | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Referência a código | errorMiddleware trata AppError, Zod e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-e | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Referência a código | requireRole e authenticate reaproveitados | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-007-a | docs/adrs/ADR-007-publish-webhook-event-na-transacao.md | Restrição | Falha na outbox causa rollback do status | TRANSCRICAO | [09:40] Bruno |
| ADR-007-b | docs/adrs/ADR-007-publish-webhook-event-na-transacao.md | Decisão | Sem injeção do repository inteiro no OrderService | TRANSCRICAO | [09:41] Diego |
| ADR-007-c | docs/adrs/ADR-007-publish-webhook-event-na-transacao.md | Referência a código | $transaction de changeStatus com update e histórico | CODIGO | src/modules/orders/order.service.ts |
| ADR-008-a | docs/adrs/ADR-008-payload-headers-e-limites-de-entrega.md | Restrição | Acima de 64 KB não se envia (erro, sem truncar) | TRANSCRICAO | [09:23] Sofia |
| ADR-008-b | docs/adrs/ADR-008-payload-headers-e-limites-de-entrega.md | Decisão | X-Webhook-Id adicionado a pedido da Segurança | TRANSCRICAO | [09:44] Sofia |
| ADR-008-c | docs/adrs/ADR-008-payload-headers-e-limites-de-entrega.md | Decisão | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-OT-01 | docs/FDD.md | Objetivo técnico | Nenhum evento perdido nem de transação revertida | TRANSCRICAO | [09:06] Diego |
| FDD-OT-02 | docs/FDD.md | Objetivo técnico | Latência < 10 s com polling de 2 s | TRANSCRICAO | [09:09] Diego |
| FDD-OT-03 | docs/FDD.md | Objetivo técnico | Entrega at-least-once, assinada, com retry e DLQ | TRANSCRICAO | [09:24] Diego |
| FDD-EXC-01 | docs/FDD.md | Exclusão | Fora do escopo: e-mail, rate limiting, dashboard, arquivamento | TRANSCRICAO | [09:37] Larissa |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | Tabela webhook_outbox com índices em status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | Tabela webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | Config de webhook: url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | Status da outbox: pendente, processando, falhou, entregue | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-05 | docs/FDD.md | Modelo de dados | Tabela de entregas (payload, resposta, tempo) derivada do endpoint de deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-06 | docs/FDD.md | Referência a código | Padrão de modelos Prisma com UUID, @@map e @@index | CODIGO | prisma/schema.prisma |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato público | POST /webhooks, secret devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato público | GET /webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato público | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato público | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato público | POST rotate-secret, geração de nova secret pela API | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato público | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato público | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato público | Contrato de saída: headers e corpo do POST ao cliente | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato público | Formato de resposta paginada e de erro reaproveitado | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato público | DELETE devolve 204 como em pedidos | CODIGO | src/modules/orders/order.controller.ts |
| FDD-FLX-01 | docs/FDD.md | Fluxo | Criação do evento na outbox dentro da mesma transação | TRANSCRICAO | [09:40] Bruno |
| FDD-FLX-02 | docs/FDD.md | Fluxo | Filtro de status na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| FDD-FLX-03 | docs/FDD.md | Fluxo | Polling a cada 2 s com ordem por created_at | TRANSCRICAO | [09:12] Diego |
| FDD-FLX-04 | docs/FDD.md | Fluxo | Retry com backoff e 5 tentativas | TRANSCRICAO | [09:17] Larissa |
| FDD-FLX-05 | docs/FDD.md | Fluxo | Esgotadas as tentativas, evento vai para DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-06 | docs/FDD.md | Fluxo | Replay reinsere o evento como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-07 | docs/FDD.md | Fluxo | Bootstrap e shutdown do worker modelados em server.ts | CODIGO | src/server.ts |
| FDD-ERR-01 | docs/FDD.md | Matriz de erros | WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Matriz de erros | Prefixo WEBHOOK_ para todo o módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-03 | docs/FDD.md | Matriz de erros | URL http recusada com erro de validação | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-04 | docs/FDD.md | Matriz de erros | Payload acima de 64 KB não é enviado | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-05 | docs/FDD.md | Matriz de erros | Timeout de 10 s tratado como falha e retry | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-06 | docs/FDD.md | Matriz de erros | Formato de resposta de erro do errorMiddleware | CODIGO | src/middlewares/error.middleware.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout 10 s, retry 5x, DLQ | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Worker separado isola falhas da API | TRANSCRICAO | [09:11] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | Ordenação só com um worker; escala futura por order_id ou lock | TRANSCRICAO | [09:13] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs Pino, com log de quem fez o replay | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Pino já existe no projeto; redação de campos sensíveis via redactPaths | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | requestId do pino-http liga o evento à requisição | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent dentro da $transaction | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Enum OrderStatus e transições usados para validar os eventos | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Erros WEBHOOK_* estendem AppError e erros HTTP | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Reexport dos novos erros | CODIGO | src/shared/errors/index.ts |
| FDD-INT-05 | docs/FDD.md | Integração | errorMiddleware sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | authenticate e requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | validate com schemas Zod | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | buildControllers instancia o módulo | CODIGO | src/app.ts |
| FDD-INT-09 | docs/FDD.md | Integração | buildApiRouter monta /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-10 | docs/FDD.md | Integração | worker.ts modelado no bootstrap atual | CODIGO | src/server.ts |
| FDD-INT-11 | docs/FDD.md | Integração | PrismaClient próprio do worker via createPrismaClient | CODIGO | src/config/database.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Variáveis de ambiente novas no schema Zod | CODIGO | src/config/env.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Pino reutilizado | CODIGO | src/shared/logger/index.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Novos modelos Prisma | CODIGO | prisma/schema.prisma |
| FDD-INT-15 | docs/FDD.md | Integração | Script npm run worker | CODIGO | package.json |
| FDD-INT-16 | docs/FDD.md | Integração | Limpeza de novas tabelas em testes futuros | CODIGO | tests/setup.ts |
| FDD-INT-17 | docs/FDD.md | Integração | Reuso de requireRole no replay | TRANSCRICAO | [09:36] Larissa |
| FDD-INT-18 | docs/FDD.md | Integração | Módulo em src/modules/webhooks e src/worker.ts | TRANSCRICAO | [09:11] Larissa |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node >= 20, Prisma 5.22, Zod, Pino | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | MySQL 8 do docker-compose | CODIGO | docker-compose.yml |
| FDD-DEP-03 | docs/FDD.md | Dependência | Revisão de segurança de 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-CA-01 | docs/FDD.md | Critério de aceite | Replay só por ADMIN, com auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-02 | docs/FDD.md | Critério de aceite | Rotação: secret antiga expira em 24 h | TRANSCRICAO | [09:22] Sofia |
| FDD-RSK-01 | docs/FDD.md | Risco | Prazo de 3 sprints | TRANSCRICAO | [09:46] Larissa |
| FDD-RSK-02 | docs/FDD.md | Risco | Vazamento de secret em log de cliente | TRANSCRICAO | [09:22] Diego |
| FDD-PROP-01 | docs/FDD.md | Proposta (sem origem direta) | Critério de sucesso 2xx, reset de PROCESSING no boot, tamanho do lote: detalhamento de implementação | CODIGO | src/modules/orders/order.service.ts |
| FDD-OT-04 | docs/FDD.md | Objetivo técnico | Reuso integral dos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| FDD-OT-05 | docs/FDD.md | Objetivo técnico | Worker em processo separado, mesmo banco e stack | TRANSCRICAO | [09:11] Diego |
| PRD-CA-01 | docs/PRD.md | Critério de aceite | Cadastro com URL https devolve a secret; http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-02 | docs/PRD.md | Critério de aceite | Mudança de status gera 1 evento por endpoint interessado e nenhum sem interessados | TRANSCRICAO | [09:34] Bruno |
| PRD-CA-03 | docs/PRD.md | Critério de aceite | Cliente recebe evento em menos de 10 s com headers e assinatura | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-04 | docs/PRD.md | Critério de aceite | Reentrega conforme backoff e, esgotado, evento na DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-CA-05 | docs/PRD.md | Critério de aceite | ADMIN reprocessa DLQ; OPERATOR recebe 403; log do executor | TRANSCRICAO | [09:36] Sofia |
| PRD-CA-06 | docs/PRD.md | Critério de aceite | Secret antiga deixa de valer após 24 h da rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-07 | docs/PRD.md | Critério de aceite | Histórico mostra últimas 100 entregas com resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-08 | docs/PRD.md | Critério de aceite | Nada fora de escopo (e-mail, rate limiting, painel) presente | TRANSCRICAO | [09:37] Larissa |
| FDD-CA-03 | docs/FDD.md | Critério de aceite técnico | Evento entregue em < 10 s (p95) sem falha | TRANSCRICAO | [09:02] Marcos |
| FDD-CA-04 | docs/FDD.md | Critério de aceite técnico | Corpo assinado com HMAC-SHA256 e headers conforme contrato | TRANSCRICAO | [09:44] Diego |
| FDD-CA-05 | docs/FDD.md | Critério de aceite técnico | Falhas seguem 1m/5m/30m/2h/12h e vão à DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-CA-06 | docs/FDD.md | Critério de aceite técnico | Replay só por ADMIN, registra userId e reinsere como pendente | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-07 | docs/FDD.md | Critério de aceite técnico | URL http rejeitada; payload > 64 KB não enviado | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-08 | docs/FDD.md | Critério de aceite técnico | Secret anterior expira em 24 h após rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-09 | docs/FDD.md | Critério de aceite técnico | Parar a API não interrompe o worker | TRANSCRICAO | [09:11] Diego |
| FDD-CA-10 | docs/FDD.md | Critério de aceite técnico | Erros do módulo com prefixo WEBHOOK_ e formato do errorMiddleware | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-WEBHOOK_CUSTOMER_NOT_FOUND | docs/FDD.md | Matriz de erros | WEBHOOK_CUSTOMER_NOT_FOUND: customerId inexistente no cadastro (análogo ao NotFoundError('Customer') de pedidos) | CODIGO | src/modules/orders/order.service.ts |
| FDD-ERR-WEBHOOK_DEAD_LETTER_NOT_FOUND | docs/FDD.md | Matriz de erros | WEBHOOK_DEAD_LETTER_NOT_FOUND: Id de DLQ inexistente no replay (derivado do endpoint de replay) | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-WEBHOOK_DELIVERY_FAILED | docs/FDD.md | Matriz de erros | WEBHOOK_DELIVERY_FAILED: Resposta não-2xx ou erro de rede vira falha e retry (critério 2xx é proposta) | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-WEBHOOK_DELIVERY_TIMEOUT | docs/FDD.md | Matriz de erros | WEBHOOK_DELIVERY_TIMEOUT: Sem resposta em 10 s é falha e vai para retry | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-WEBHOOK_ENDPOINT_INACTIVE | docs/FDD.md | Matriz de erros | WEBHOOK_ENDPOINT_INACTIVE: Endpoint removido ou inativo antes do envio (derivado do estado ativo e do DELETE) | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-WEBHOOK_OUTBOX_INSERT_FAILED | docs/FDD.md | Matriz de erros | WEBHOOK_OUTBOX_INSERT_FAILED: Falha no insert da outbox causa rollback da mudança de status | TRANSCRICAO | [09:40] Bruno |
| FDD-ERR-WEBHOOK_PAYLOAD_TOO_LARGE | docs/FDD.md | Matriz de erros | WEBHOOK_PAYLOAD_TOO_LARGE: Payload acima de 64 KB não é enviado | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-WEBHOOK_REPLAY_NOT_ALLOWED | docs/FDD.md | Matriz de erros | WEBHOOK_REPLAY_NOT_ALLOWED: Replay de evento já reprocessado (proposta, derivada do replay manual) | TRANSCRICAO | [09:18] Diego |
