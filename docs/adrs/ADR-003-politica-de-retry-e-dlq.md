# ADR-003 — Política de retry com backoff exponencial e DLQ

## Status

Aceito

- **Data:** decisão registrada na reunião técnica de quinta-feira (ver `TRANSCRICAO.md`); ADR consolidado em 20/11/2026
- **Decisores:** Larissa (Tech Lead), Diego (Engenharia Sênior), Bruno (Engenharia Pleno)
- **Contexto da decisão:** reunião técnica do Sistema de Webhooks de Notificação de Pedidos

## Contexto

O cliente pode estar temporariamente indisponível — manutenção planejada, deploy, indisponibilidade de rede ou erro transitório. A entrega precisa sobreviver a falhas temporárias, mas não pode ficar pendurada para sempre se o cliente desapareceu.

A reunião considerou retry indefinido, 3 tentativas e 5 tentativas. Retry indefinido foi descartado porque o evento ficaria pendurado eternamente. Três tentativas foram consideradas pouco: um cliente com indisponibilidade de duas horas em manutenção planejada seria descartado depois de três retries em cerca de 30 minutos ([09:15]–[09:16] Diego/Bruno). A decisão foi **5 tentativas** com backoff **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas**, cobertando quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego/Larissa).

Também foi decidido que, esgotadas as tentativas, o evento vai para uma **DLQ em tabela separada**, com payload, motivo da falha e timestamp, em vez de apenas marcar como `failed` na outbox ([09:18] Diego). O replay é manual, por endpoint administrativo, e exige role `ADMIN` com auditoria de quem executou ([09:18] Diego, [09:35]–[09:36] Diego/Larissa/Sofia).

## Decisão

1. A primeira tentativa de entrega acontece assim que o worker pega o evento.
2. Em caso de falha, o worker agenda **até 5 retries** após a tentativa inicial, com os seguintes intervalos:

   | Retry | Espera após a falha anterior |
   | --- | --- |
   | 1 | 1 minuto |
   | 2 | 5 minutos |
   | 3 | 30 minutos |
   | 4 | 2 horas |
   | 5 | 12 horas |

3. A janela total entre a primeira falha e a última tentativa é de **14 h 36 min** (1 m + 5 m + 30 m + 2 h + 12 h), coerente com a fala de "quase 15 horas" ([09:17] Diego).
4. Se o retry 5 também falhar, o evento é considerado falha permanente: a linha da outbox é marcada como `FAILED` e é criado um registro em `webhook_dead_letter` com payload, motivo da falha, `webhookId`, `eventId`, `orderId` e timestamp.
5. O replay é manual pelo endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, exige role `ADMIN` e registra quem executou o replay para auditoria ([09:36] Sofia).
6. O replay recoloca o evento na outbox como `PENDING`, zera o contador de tentativas e preserva o payload original.
7. **Não haverá retry indefinido** nem fallback por e-mail nesta fase. E-mail de alerta ao cliente é próximo passo, após medir o impacto ([09:37] Larissa).

> **Nota de interpretação:** a transcrição registra "5 tentativas" e também lista cinco intervalos de espera. Este ADR adota **5 retries após a tentativa inicial**, totalizando até 6 chamadas HTTP. Essa leitura é a única compatível com a janela de "quase 15 horas" e com a afirmação de que 5 tentativas cobrem "12 ou 24 horas" ([09:15]–[09:17] Diego). O RFC registra a mesma leitura.

## Alternativas Consideradas

### Alternativa A — Sem retry (tentativa única)

- **Trade-off que levou ao descarte:** qualquer indisponibilidade momentânea do cliente perderia o evento permanentemente; não atende à expectativa de confiabilidade da integração.
- **Custo evitado:** simplicidade de implementação, ao preço de perda de eventos.

### Alternativa B — 3 tentativas

- **Trade-off que levou ao descarte:** não cobre indisponibilidades de algumas horas; um cliente com manutenção planejada de 2 horas seria descartado em cerca de 30 minutos ([09:16] Diego).
- **Custo evitado:** menos tempo de espera, ao preço de menos resiliência.

### Alternativa C — Retry indefinido com backoff

- **Trade-off que levou ao descarte:** o evento ficaria pendurado para sempre se o cliente tivesse desaparecido; sem estado terminal claro ([09:15] Diego).
- **Custo evitado:** fila infinita e eventos sem desfecho.

### Alternativa D — Marcar falha na própria outbox, sem tabela de DLQ

- **Trade-off que levou ao descarte:** a outbox principal ficaria mais poluída para leitura; a tabela separada é mais limpa e serve como evidência para debug e reprocessamento ([09:18] Diego).
- **Custo evitado:** menos uma tabela, ao preço de pior operação e auditoria.

### Alternativa E — Backoff linear ou intervalo fixo

- **Trade-off:** não foi a opção escolhida; um intervalo fixo não acomoda falhas curtas e longas com a mesma eficiência. O backoff exponencial foi o padrão definido na reunião ([09:15] Diego).

## Consequências

### Positivas

- **Resiliência a falhas temporárias:** cobre desde indisponibilidades curtas até manutenções de algumas horas.
- **Estado terminal claro:** todo evento termina como `DELIVERED` ou `FAILED`/DLQ; não fica pendurado indefinidamente.
- **Auditoria e reprocessamento:** a DLQ guarda payload, motivo e timestamp; o replay é controlado e auditado.
- **Previsibilidade operacional:** a progressão de backoff é conhecida e documentada.

### Negativas / trade-offs

- **Demora até a DLQ:** um cliente fora do ar pode levar cerca de 15 horas para ter o evento considerado falho permanente.
- **Duplicidade:** retries podem gerar entregas duplicadas; o cliente precisa deduplicar por `X-Event-Id` (ADR-005).
- **Mais uma tabela e um fluxo operacional:** DLQ e replay exigem monitoramento e processo de suporte.
- **Sem alerta proativo:** nesta fase não há e-mail nem notificação ao cliente quando ele falha repetidamente; isso foi adiado ([09:37] Larissa).
- **Custo de armazenamento:** cada tentativa gera registro de delivery e a DLQ preserva o payload.

## Referências

- `TRANSCRICAO.md` — [09:15] Diego; [09:16] Bruno/Diego; [09:17] Diego/Larissa/Marcos; [09:18] Diego/Bruno; [09:35]–[09:36] Diego/Larissa/Sofia; [09:37] Larissa; [09:48] Larissa
- `src/modules/orders/order.service.ts` — origem dos eventos
- `prisma/schema.prisma` — modelos de outbox e DLQ propostos no FDD
- `src/shared/errors/http-errors.ts` — padrão de erros de domínio
- `src/middlewares/auth.middleware.ts` — `requireRole` para o replay
