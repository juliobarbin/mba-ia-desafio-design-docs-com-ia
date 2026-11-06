# ADR-002: Política de retry com backoff exponencial e DLQ em tabela separada

- **Status:** Aceito
- **Data:** 2026-11-06
- **Decisores:** Larissa, Diego, Bruno (Marcos concordou com a janela de ~15h)
- **Fonte:** TRANSCRICAO [09:14]–[09:19], [09:48]

## Contexto

Clientes podem ficar indisponíveis. Já houve cliente com indisponibilidade de duas horas por manutenção planejada ([09:16] Diego). O timeout da chamada HTTP é de 10 segundos e uma falha (timeout ou erro) é tratada como candidata a retry ([09:42] Diego). É preciso decidir quantas tentativas, qual progressão e o que fazer com eventos que esgotam as tentativas.

## Decisão

- **Backoff exponencial com teto de 5 tentativas**, progressão **1 min / 5 min / 30 min / 2 h / 12 h**, janela total próxima de 15 horas entre a primeira falha e a última tentativa ([09:17] Diego, decidido por Larissa [09:17]).
- Esgotadas as tentativas, o evento é considerado falha permanente e movido para uma **DLQ persistida em tabela separada, `webhook_dead_letter`**, com payload, motivo da falha e timestamp ([09:18] Diego).
- **Reprocessamento manual** via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. Exige role `ADMIN` (reuso de `requireRole`) e registra em log quem executou o replay, para auditoria ([09:18] Diego, [09:36] Sofia, [09:36] Larissa).

> Nota de interpretação: a transcrição diz "5 tentativas" e lista **5 intervalos**, e o total "quase 15 horas" (1+5+30+120+720 min ≈ 14,6 h) só fecha se houver a tentativa inicial mais 5 retries. Essa leitura está registrada como questão em aberto no [RFC](../RFC.md) (RFC-OQ-03) e deve ser confirmada pelo time antes do código.

## Alternativas Consideradas

1. **3 tentativas** — descartada: curto demais; uma indisponibilidade de manhã esgotaria as tentativas em ~30 min e perderia o evento ([09:16] Diego).
2. **Retry indefinido com backoff** — descartada: evento ficaria "pendurado pra sempre" se o cliente sumisse ([09:15] Diego).
3. **Marcar como `falhou` na própria outbox em vez de tabela DLQ** — descartada: tabela separada deixa a leitura da outbox principal mais limpa e serve de evidência para debug e reprocessamento ([09:18] Diego).
4. **Notificar o cliente por e-mail após falhas** — adiada para fase futura ([09:37] Larissa).

## Consequências

**Positivas**
- Cobre indisponibilidades planejadas de algumas horas sem intervenção humana.
- DLQ separada mantém a outbox enxuta e preserva evidência para suporte.
- Replay manual com auditoria dá controle operacional sem automatizar reenvios indevidos.

**Negativas / trade-offs**
- Um cliente fora por mais de ~15 h perde a entrega automática e depende de replay manual por um ADMIN.
- O evento tardio chega bem depois da mudança de status (até ~15 h) e possivelmente depois de eventos posteriores do mesmo pedido, tensionando a ordenação (ver ADR-005).
- Sem alerta ao cliente (e-mail adiado), a falha só é visível via histórico de entregas.
