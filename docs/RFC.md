# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Título** | Sistema de Webhooks de Notificação de Pedidos |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | Reunião técnica de quinta-feira (ver `TRANSCRICAO.md`); consolidação em 20/11/2026 |
| **Revisores** | Marcos (Product Manager), Bruno (Engenharia Pleno — Pedidos), Diego (Engenharia Sênior — Plataforma), Sofia (Engenharia de Segurança) |
| **ADRs relacionados** | [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-em-polling.md), [ADR-003](adrs/ADR-003-politica-de-retry-e-dlq.md), [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md), [ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md), [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md), [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) |

---

## 1. Resumo executivo (TL;DR)

**RFC-TLDR-01** — Propomos capturar eventos de mudança de status em uma **outbox transacional no MySQL**, dentro da mesma transação de `OrderService.changeStatus`, e entregá-los por um **worker separado em polling de 2 segundos**. As entregas são assinadas com **HMAC-SHA256** e secret por endpoint, com rotação e grace period de 24 horas. A garantia é **at-least-once**, com `X-Event-Id` para deduplicação no cliente. Falhas seguem retry com backoff **1 m / 5 m / 30 m / 2 h / 12 h** e, esgotadas as tentativas, vão para uma **DLQ** com replay manual por `ADMIN`. A solução reutiliza os padrões existentes do projeto e não introduz nova infraestrutura. Estimativa: **3 sprints**, incluindo pelo menos 2 dias úteis de revisão de segurança da Sofia.

**RFC-PLAN-01** — A entrega foi estimada em 3 sprints: modelagem de outbox/DLQ, worker e retry, CRUD e deliveries, integração no `order.service` e testes ponta a ponta, incluindo a revisão de segurança ([09:46] Larissa).

**RFC-PLAN-02** — A Sofia reservou pelo menos **2 dias úteis** para revisar HMAC e geração de secret antes do deploy ([09:46] Sofia).

---

## 2. Contexto e problema

**RFC-CTX-01** — Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram notificação em tempo real quando o status dos pedidos deles muda. Hoje eles fazem polling em `GET /orders`, o que deixa a integração lenta e cara para eles. Para os clientes, "tempo real" significa **abaixo de 10 segundos** ([09:00]–[09:02] Marcos).

**RFC-CTX-02** — A aplicação não possui eventos, filas ou webhooks. A transação de mudança de status já é pesada — atualiza `orders`, insere em `order_status_history` e altera estoque — e não pode ser acoplada à disponibilidade de um cliente externo ([09:04] Bruno). O escopo é **outbound**: apenas a plataforma envia para o cliente; o cliente não envia para a plataforma ([09:02] Marcos/Sofia).

**RFC-CTX-03** — A solução precisa cobrir configuração por cliente, filtro de status por endpoint, histórico de entregas, segurança das entregas, retry e operação de falhas. O detalhamento de contratos, payloads, erros e integração com o código fica no [FDD](FDD.md).

---

## 3. Proposta técnica

### 3.1 Captura atômica com outbox no MySQL

**RFC-PROP-01** — Quando o status muda, a mesma transação SQL que atualiza o pedido, o histórico e o estoque insere uma linha em `webhook_outbox` com o evento já renderizado. Se a transação commitou, o evento está registrado; se deu rollback, o evento some junto. Um worker separado lê a outbox e dispara as chamadas HTTP ([09:06] Diego, [09:40]–[09:41] Bruno/Diego).

**RFC-PROP-02** — A outbox é uma tabela no MySQL existente, com índice no status e em `created_at`. Não há Redis, RabbitMQ ou banco adicional. Essa escolha evita subir infraestrutura para um time pequeno ([09:07] Diego).

**RFC-PROP-03** — A granularidade é **uma linha de outbox por endpoint interessado no status**. Isso permite que cada endpoint tenha sua própria política de retry, DLQ, histórico e `X-Webhook-Id`. O filtro por status é aplicado **na inserção**: se nenhum endpoint do cliente quer aquele status, nenhuma linha é criada ([09:33]–[09:34] Marcos/Bruno).

**RFC-PROP-04** — O payload é um **snapshot** no momento da inserção, não uma renderização tardia. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou ([09:52] Larissa). O detalhamento do payload fica no FDD.

### 3.2 Processamento por worker separado

**RFC-PROP-05** — O worker roda em **processo separado** (`src/worker.ts`, a criar, com script `npm run worker`), com o mesmo banco e a mesma stack, mas com seu próprio `PrismaClient` por processo. Ele faz **polling a cada 2 segundos**, busca os eventos pendentes mais antigos em lote pequeno e os processa ([09:09]–[09:11] Diego/Larissa/Bruno, [09:30] Bruno).

**RFC-PROP-06** — Nesta fase roda **um único worker**. A ordering é implícita por `order_id` e por `created_at`; não há garantia de ordering global. Escalar para múltiplos workers exigiria particionamento por `order_id` ou lock pessimista, que é problema do futuro ([09:12]–[09:13] Diego/Larissa).

### 3.3 Retry, backoff e DLQ

**RFC-PROP-07** — Falhas de entrega seguem retry com backoff **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas** após a tentativa inicial. Esgotados os retries, o evento é considerado falha permanente e vai para uma **DLQ em tabela separada**, com payload, motivo e timestamp ([09:15]–[09:18] Diego).

**RFC-PROP-08** — O replay da DLQ é **manual**, por endpoint administrativo, com role `ADMIN` e auditoria de quem executou. O evento volta para a outbox como pendente ([09:18] Diego, [09:36] Sofia).

### 3.4 Segurança das entregas

**RFC-PROP-09** — As entregas são assinadas com **HMAC-SHA256 sobre o corpo do request**, com **secret única por endpoint**. A secret é gerada pela plataforma, devolvida apenas na criação e rotacionável. Na rotação, a secret anterior continua válida por **24 horas** em paralelo ([09:19]–[09:22] Sofia).

**RFC-PROP-10** — A URL do webhook precisa ser **HTTPS**; cadastros com `http` são rejeitados na validação. O payload tem limite de **64 KB**, com erro se ultrapassar ([09:23]–[09:24] Sofia/Diego/Larissa).

### 3.5 Idempotência e ordering

**RFC-PROP-11** — A garantia é **at-least-once**: o cliente pode receber o mesmo evento duas vezes. Cada evento recebe um **UUID** no momento da inserção na outbox, enviado em `X-Event-Id` (e em `event_id` no payload), para o cliente deduplicar ([09:24]–[09:26] Diego).

**RFC-PROP-12** — **Exactly-once não será implementado nesta fase.** Exigiria coordenação dos dois lados e seria muito mais complexo; at-least-once com `event_id` resolve a grande maioria dos casos ([09:25] Diego).

### 3.6 API, operação e reuso

**RFC-PROP-13** — A feature expõe CRUD de configuração de webhooks, consulta de histórico de entregas, rotação de secret e replay administrativo de DLQ. Os endpoints são autenticados com o JWT existente; o replay exige `ADMIN`. O CRUD pode ser usado por qualquer role autenticada nesta fase ([09:31]–[09:37] Marcos/Bruno/Larissa/Sofia).

**RFC-PROP-14** — O módulo segue o padrão existente: `src/modules/webhooks` com controller, service, repository, routes e schemas; erros com prefixo `WEBHOOK_`; Pino; middleware de erro centralizado; Zod; `requireRole`. Não há nova biblioteca de log, validação ou erro ([09:27]–[09:30] Bruno/Larissa).

### 3.7 Visão geral do fluxo

```text
PATCH /api/v1/orders/:id/status
        │
        ▼
┌─────────────────────────────────────────────┐
│ Transação MySQL (OrderService.changeStatus) │
│  • atualiza orders                          │
│  • insere order_status_history              │
│  • atualiza estoque                         │
│  • insere webhook_outbox (snapshot)         │
└─────────────────────────────────────────────┘
        │ commit
        ▼
  webhook_outbox (PENDING)
        │ polling 2s
        ▼
  worker separado (src/worker.ts)
        │
        ├── HMAC-SHA256 + headers
        ├── POST https://cliente
        │
        ├── 2xx ──► DELIVERED
        └── falha ──► retry 1m/5m/30m/2h/12h
                         │
                         └── esgotou ──► webhook_dead_letter
                                              │
                                              └── replay ADMIN
```

---

## 4. Alternativas consideradas

| ID | Alternativa | Por que foi considerada | Trade-off que levou ao descarte |
| --- | --- | --- | --- |
| **RFC-ALT-01** | Chamada HTTP síncrona dentro de `changeStatus` | Simplicidade e entrega imediata | Cliente lento travaria a transação de status; cliente fora do ar não permitiria rollback coerente ([09:04] Bruno) |
| **RFC-ALT-02** | Redis Streams, RabbitMQ ou fila externa | Desacoplamento e entrega assíncrona | Exigiria subir e operar mais infraestrutura; overengineering para o tamanho do time ([09:07] Diego) |
| **RFC-ALT-03** | Trigger de banco para acordar o worker | Reatividade sem polling | MySQL não tem listener nativo; trigger só executa SQL e não notifica processo externo ([09:09] Diego) |
| **RFC-ALT-04** | Exactly-once | Garantia mais forte para o cliente | Exigiria coordenação dos dois lados e complexidade desproporcional; at-least-once com `event_id` resolve a maioria ([09:25] Diego) |
| **RFC-ALT-05** | Retry indefinido com backoff | Nunca desistir do cliente | Evento ficaria pendurado para sempre se o cliente desaparecesse; sem estado terminal ([09:15] Diego) |
| **RFC-ALT-06** | 3 tentativas em vez de 5 | Feedback mais rápido e menos espera | Não cobre indisponibilidade de 2 horas em manutenção planejada; 3 retries em ~30 min descartariam o cliente ([09:16] Diego) |
| **RFC-ALT-07** | Marcar falha na própria outbox, sem DLQ separada | Menos uma tabela e menos código | Outbox principal ficaria poluída; tabela separada é mais limpa e serve como evidência para debug/replay ([09:18] Diego) |

---

## 5. Questões em aberto

| ID | Questão | Situação | Próximo passo |
| --- | --- | --- | --- |
| **RFC-QO-01** | **Rate limiting de envio**: se o cliente tiver 50 pedidos mudando em um minuto, a plataforma bombardeará o cliente com 50 chamadas? | Não decidido; "observar e decidir depois" ([09:38]–[09:39] Diego/Larissa) | Medir volume real após o lançamento e avaliar limite por endpoint/cliente em fase futura |
| **RFC-QO-02** | **Escala horizontal e ordering**: como garantir ordering por `order_id` com múltiplos workers? | Adiado; single-worker nesta fase ([09:13] Diego/Larissa) | Avaliar particionamento por `order_id` ou lock pessimista quando o volume exigir |
| **RFC-QO-03** | **E-mail de alerta ao cliente** em falhas recorrentes (ex.: 3 seguidas) | Fora de escopo desta fase; possível próxima fase após medir impacto ([09:37] Larissa) | Medir taxa de falhas e reavaliar |
| **RFC-QO-04** | **Arquivamento de eventos entregues após 30 dias** | Fora de escopo desta feature ([09:08] Diego) | Definir política de retenção e job de arquivamento em fase futura |
| **RFC-QO-05** | **Endurecimento de roles no CRUD**: hoje qualquer role autenticada pode configurar webhooks | Adiado; "mais pra frente a gente pode endurecer" ([09:36]–[09:37] Sofia) | Reavaliar quando houver requisito de segregação de funções |

---

## 6. Impacto e riscos

| ID | Impacto / risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| **RFC-RISK-01** | Cliente indisponível por longos períodos; evento termina em DLQ sem alerta proativo | Média | Alto | Retry por ~15 h, DLQ com replay manual; e-mail fica para fase futura ([09:15]–[09:18] Diego, [09:37] Larissa) |
| **RFC-RISK-02** | Crescimento da outbox e atraso do worker | Média | Alto | Índices em status/`created_at`, leitura em lote pequeno, monitoramento; arquivamento futuro ([09:08] Diego) |
| **RFC-RISK-03** | Vazamento de secret ou assinatura em log | Média | Alto | Secret por endpoint, rotação com grace de 24 h, redaction no Pino, revisão de segurança ([09:21]–[09:22] Sofia) |
| **RFC-RISK-04** | Prazo de 3 sprints + revisão de segurança vs. expectativa da Atlas para fim de novembro | Alta | Alto | Escopo enxuto (sem dashboard/e-mail/rate limiting), revisão da Sofia agendada, acompanhamento semanal ([09:45]–[09:47] Marcos/Larissa/Sofia) |
| **RFC-RISK-05** | Duplicidade de entregas confundir o cliente | Média | Médio | `X-Event-Id`, documentação destacada no portal ([09:26] Marcos) |
| **RFC-RISK-06** | Payload anormalmente grande falhar a transação de status | Baixa | Médio | Payload enxuto sem itens, limite de 64 KB com erro explícito ([09:43] Diego, [09:24] Larissa) |

**Impactos positivos esperados:** eliminação do polling manual dos clientes, integração mais rápida e barata, previsibilidade operacional com DLQ e replay, e base reutilizável para futuras notificações.

**Impactos operacionais:** necessidade de rodar e monitorar um novo processo (worker), nova migração de banco e acompanhamento da DLQ.

---

## 7. Decisões relacionadas

| ID | ADR | Decisão |
| --- | --- | --- |
| **RFC-ADR-01** | [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md) | Outbox transacional no MySQL existente |
| **RFC-ADR-02** | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2 s, single-worker |
| **RFC-ADR-03** | [ADR-003](adrs/ADR-003-politica-de-retry-e-dlq.md) | Retry com backoff 1 m/5 m/30 m/2 h/12 h e DLQ com replay |
| **RFC-ADR-04** | [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação de 24 h |
| **RFC-ADR-05** | [ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` para deduplicação |
| **RFC-ADR-06** | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes (módulos, AppError, Pino, Zod, `requireRole`) |
| **RFC-ADR-07** | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload na inserção da outbox |

---

## 8. Não objetivos desta fase

- Webhooks **inbound** (cliente enviando para a plataforma) ([09:02] Marcos).
- Dashboard visual para o cliente; apenas endpoints ([09:40] Larissa).
- Notificação por e-mail em falhas recorrentes ([09:37] Larissa).
- Rate limiting de envio; observar e decidir depois ([09:39] Larissa).
- Arquivamento automático de eventos entregues após 30 dias ([09:08] Diego).
- Múltiplos workers e ordering global ([09:13] Diego).

---

## 9. Referências

- `TRANSCRICAO.md` — reunião técnica completa.
- [PRD](PRD.md) — problema, público, escopo e métricas.
- [FDD](FDD.md) — contratos, fluxos, erros e integração com o código.
- `docs/adrs/` — decisões arquiteturais detalhadas.
- `docs/TRACKER.md` — rastreabilidade de cada item à transcrição ou ao código.
