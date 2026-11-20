# Tracker de Rastreabilidade

Este tracker mapeia os itens registrados em `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/` às suas origens na transcrição (`TRANSCRICAO.md`) ou no código-fonte.

**Critérios de leitura:**

- **Fonte `TRANSCRICAO`:** a localização segue o formato `[hh:mm] Nome`, exatamente como na transcrição.
- **Fonte `CODIGO`:** a localização é um caminho de arquivo real do repositório.
- Itens marcados como derivação de implementação usam a fala que originou a decisão e são explicados no documento correspondente.
- As linhas estão agrupadas por documento de origem.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Webhooks outbound para notificar mudança de status de pedidos | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Três clientes B2B pedem notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | Escopo somente outbound; cliente não envia para a plataforma | TRANSCRICAO | [09:02] Marcos |
| PRD-PROB-01 | docs/PRD.md | Problema | Clientes fazem polling em GET /orders; integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Problema | Atlas pode migrar para concorrente se não houver entrega até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-03 | docs/PRD.md | Problema | Clientes não exigem ordering global; querem saber mudanças por pedido | TRANSCRICAO | [09:14] Marcos |
| PRD-PROB-04 | docs/PRD.md | Problema | Transação de status é pesada; chamada HTTP síncrona é inviável | TRANSCRICAO | [09:04] Bruno |
| PRD-UC-01 | docs/PRD.md | Cenário de uso | Cadastro de endpoint e recebimento de evento em até 10s | TRANSCRICAO | [09:00] Marcos |
| PRD-UC-02 | docs/PRD.md | Cenário de uso | Cliente valida assinatura HMAC-SHA256 | TRANSCRICAO | [09:20] Sofia |
| PRD-UC-03 | docs/PRD.md | Cenário de uso | Cliente deduplica entregas repetidas pelo X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-UC-04 | docs/PRD.md | Cenário de uso | Cliente consulta histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-UC-05 | docs/PRD.md | Cenário de uso | Cliente rotaciona secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-UC-06 | docs/PRD.md | Cenário de uso | Administrador reprocessa evento de DLQ com auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | 100% dos eventos elegíveis com primeira tentativa em até 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Ativar 3 clientes B2B até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Eliminar chamadas de polling dos clientes integrados | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Zero mudanças de status sem evento correspondente na outbox | TRANSCRICAO | [09:41] Diego |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | 100% das entregas assinadas com HMAC-SHA256 | TRANSCRICAO | [09:22] Sofia |
| PRD-SCOPE-01 | docs/PRD.md | Escopo | Webhooks outbound de mudança de status | TRANSCRICAO | [09:02] Marcos |
| PRD-SCOPE-02 | docs/PRD.md | Escopo | CRUD de endpoints de webhook via API | TRANSCRICAO | [09:31] Marcos |
| PRD-SCOPE-03 | docs/PRD.md | Escopo | Filtro de eventos por lista de status por endpoint | TRANSCRICAO | [09:33] Marcos |
| PRD-SCOPE-04 | docs/PRD.md | Escopo | Histórico de entregas com payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-SCOPE-05 | docs/PRD.md | Escopo | Replay manual de DLQ por administrador | TRANSCRICAO | [09:18] Diego |
| PRD-SCOPE-06 | docs/PRD.md | Escopo | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail de alerta em falhas recorrentes adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Rate limiting de envio será observado e decidido depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Dashboard visual é projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Webhooks inbound não entram nesta fase | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues após 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Múltiplos workers e ordering global | TRANSCRICAO | [09:13] Diego |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Endurecimento de roles no CRUD fica para depois | TRANSCRICAO | [09:37] Sofia |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Criar webhook com customerId, url e eventStatuses; secret gerada e devolvida | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar webhook via PATCH | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remover webhook via DELETE | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer via GET | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status por endpoint | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Enviar evento outbound quando o status muda | TRANSCRICAO | [09:43] Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Filtrar eventos na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Consultar últimas 100 entregas com sucesso, falha, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Replay manual de evento em DLQ | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotacionar secret mantendo a anterior válida por 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Assinar entregas com HMAC-SHA256 e secret por endpoint | TRANSCRICAO | [09:22] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Enviar X-Event-Id único para deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Retry com backoff e DLQ após esgotar tentativas | TRANSCRICAO | [09:15] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Primeira tentativa de entrega em até 10s no pior caso | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | URL do webhook deve ser HTTPS; http é recusado | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Payload máximo de 64 KB com erro se ultrapassar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s na chamada HTTP do worker | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Ordering por order_id enquanto single-worker; sem garantia global | TRANSCRICAO | [09:12] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | CRUD autenticado; replay exige ADMIN e auditoria | TRANSCRICAO | [09:36] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Reuso de MySQL, Prisma, Pino, Zod, Express e JWT | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Payload enxuto, sem itens do pedido | TRANSCRICAO | [09:43] Diego |
| PRD-DEC-01 | docs/PRD.md | Decisão | Outbox MySQL em vez de chamada síncrona; trade-off de latência | TRANSCRICAO | [09:04] Bruno |
| PRD-DEC-02 | docs/PRD.md | Trade-off | At-least-once em vez de exactly-once; dedup no cliente | TRANSCRICAO | [09:25] Diego |
| PRD-DEC-03 | docs/PRD.md | Trade-off | DLQ em tabela separada em vez de falha na outbox | TRANSCRICAO | [09:18] Diego |
| PRD-DEC-04 | docs/PRD.md | Trade-off | Secret por endpoint em vez de secret global | TRANSCRICAO | [09:21] Sofia |
| PRD-DEC-05 | docs/PRD.md | Trade-off | Single-worker nesta fase, sem escala horizontal | TRANSCRICAO | [09:12] Diego |
| PRD-DEC-06 | docs/PRD.md | Trade-off | E-mail, rate limiting e dashboard fora desta fase | TRANSCRICAO | [09:39] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | MySQL e Prisma existentes | TRANSCRICAO | [09:07] Diego |
| PRD-DEP-02 | docs/PRD.md | Dependência | JWT e roles ADMIN/OPERATOR existentes | TRANSCRICAO | [09:36] Larissa |
| PRD-DEP-03 | docs/PRD.md | Dependência | Worker separado com mesmo banco e stack | TRANSCRICAO | [09:30] Bruno |
| PRD-DEP-04 | docs/PRD.md | Dependência | Cliente precisa expor HTTPS, validar HMAC e deduplicar | TRANSCRICAO | [09:20] Sofia |
| PRD-DEP-05 | docs/PRD.md | Dependência | Revisão de segurança da Sofia por 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-06 | docs/PRD.md | Dependência | Prazo comercial da Atlas no fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente indisponível por longos períodos e evento em DLQ | TRANSCRICAO | [09:17] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Crescimento da outbox e atraso do worker | TRANSCRICAO | [09:08] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret ou assinatura em log | TRANSCRICAO | [09:22] Sofia |
| PRD-RISK-04 | docs/PRD.md | Risco | Prazo de 3 sprints e revisão de segurança vs. deadline | TRANSCRICAO | [09:46] Larissa |
| PRD-RISK-05 | docs/PRD.md | Risco | Duplicidade de entregas confundir o cliente | TRANSCRICAO | [09:26] Marcos |
| PRD-RISK-06 | docs/PRD.md | Risco | Payload grande falhar a mudança de status | TRANSCRICAO | [09:24] Larissa |
| PRD-ACC-01 | docs/PRD.md | Critério de Aceitação | CRUD de webhooks autenticado com customerId e filtro de status | TRANSCRICAO | [09:33] Bruno |
| PRD-ACC-02 | docs/PRD.md | Critério de Aceitação | Evento na outbox na mesma transação; rollback em falha | TRANSCRICAO | [09:41] Diego |
| PRD-ACC-03 | docs/PRD.md | Critério de Aceitação | Entrega em até 10s após o commit | TRANSCRICAO | [09:10] Larissa |
| PRD-ACC-04 | docs/PRD.md | Critério de Aceitação | HMAC verificável e rotação com 24h de grace | TRANSCRICAO | [09:22] Sofia |
| PRD-ACC-05 | docs/PRD.md | Critério de Aceitação | Retry com backoff e DLQ após esgotar | TRANSCRICAO | [09:17] Larissa |
| PRD-ACC-06 | docs/PRD.md | Critério de Aceitação | Replay exige ADMIN e registra autor | TRANSCRICAO | [09:36] Sofia |
| PRD-ACC-07 | docs/PRD.md | Critério de Aceitação | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-ACC-08 | docs/PRD.md | Critério de Aceitação | http recusado e payload acima de 64 KB rejeitado | TRANSCRICAO | [09:24] Larissa |
| PRD-ACC-09 | docs/PRD.md | Critério de Aceitação | Itens fora de escopo não entregues nesta fase | TRANSCRICAO | [09:48] Larissa |
| PRD-TEST-01 | docs/PRD.md | Estratégia de Teste | Testes unitários de retry, HMAC, URL, filtro e payload | CODIGO | tests/orders.test.ts |
| PRD-TEST-02 | docs/PRD.md | Estratégia de Teste | Testes de integração com Supertest para contratos e roles | CODIGO | tests/auth.test.ts |
| PRD-TEST-03 | docs/PRD.md | Estratégia de Teste | Testes do worker com HTTP mock, timeout e relógio controlado | TRANSCRICAO | [09:17] Larissa |
| PRD-TEST-04 | docs/PRD.md | Estratégia de Teste | Teste ponta a ponta status, outbox, worker e endpoint | TRANSCRICAO | [09:40] Bruno |
| PRD-TEST-05 | docs/PRD.md | Estratégia de Teste | Testes de segurança de HMAC, rotação, http e payload | TRANSCRICAO | [09:46] Sofia |
| PRD-TEST-06 | docs/PRD.md | Estratégia de Teste | Regressão da suíte existente de pedidos | CODIGO | tests/orders.test.ts |
| RFC-TLDR-01 | docs/RFC.md | Resumo Executivo | Outbox MySQL, worker polling, retry/DLQ, HMAC e at-least-once | TRANSCRICAO | [09:48] Larissa |
| RFC-PLAN-01 | docs/RFC.md | Plano | Estimativa de 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| RFC-PLAN-02 | docs/RFC.md | Plano | Reserva de 2 dias úteis para revisão de segurança | TRANSCRICAO | [09:46] Sofia |
| RFC-CTX-01 | docs/RFC.md | Contexto | Clientes B2B fazem polling e precisam de latência abaixo de 10s | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Contexto | Transação de status é pesada e não há eventos ou filas | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-03 | docs/RFC.md | Contexto | Escopo outbound; detalhes de implementação ficam no FDD | TRANSCRICAO | [09:02] Marcos |
| RFC-PROP-01 | docs/RFC.md | Proposta | Inserir evento na outbox dentro da transação de status | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-02 | docs/RFC.md | Proposta | Outbox no MySQL existente, sem nova infraestrutura | TRANSCRICAO | [09:07] Diego |
| RFC-PROP-03 | docs/RFC.md | Proposta | Uma linha de outbox por endpoint interessado e filtro na inserção | TRANSCRICAO | [09:34] Bruno |
| RFC-PROP-04 | docs/RFC.md | Proposta | Payload snapshot no momento da inserção | TRANSCRICAO | [09:52] Larissa |
| RFC-PROP-05 | docs/RFC.md | Proposta | Worker separado com polling de 2s | TRANSCRICAO | [09:09] Diego |
| RFC-PROP-06 | docs/RFC.md | Proposta | Single-worker e ordering implícita por order_id | TRANSCRICAO | [09:12] Diego |
| RFC-PROP-07 | docs/RFC.md | Proposta | Retry com backoff 1m/5m/30m/2h/12h e DLQ | TRANSCRICAO | [09:15] Diego |
| RFC-PROP-08 | docs/RFC.md | Proposta | Replay manual de DLQ com role ADMIN e auditoria | TRANSCRICAO | [09:18] Diego |
| RFC-PROP-09 | docs/RFC.md | Proposta | HMAC-SHA256 com secret por endpoint e rotação de 24h | TRANSCRICAO | [09:20] Sofia |
| RFC-PROP-10 | docs/RFC.md | Proposta | HTTPS obrigatório e limite de payload de 64 KB | TRANSCRICAO | [09:23] Sofia |
| RFC-PROP-11 | docs/RFC.md | Proposta | At-least-once com X-Event-Id para deduplicação | TRANSCRICAO | [09:25] Diego |
| RFC-PROP-12 | docs/RFC.md | Proposta | Exactly-once não será implementado nesta fase | TRANSCRICAO | [09:25] Diego |
| RFC-PROP-13 | docs/RFC.md | Proposta | CRUD autenticado e replay ADMIN | TRANSCRICAO | [09:36] Larissa |
| RFC-PROP-14 | docs/RFC.md | Proposta | Reuso de módulos, AppError, Pino, Zod e requireRole | TRANSCRICAO | [09:30] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa | Chamada HTTP síncrona no changeStatus descartada | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa | Redis Streams ou fila externa descartada | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa | Trigger de banco para acordar worker descartada | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa | Retry indefinido descartado | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa | Três tentativas descartadas em favor de cinco | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-07 | docs/RFC.md | Alternativa | Marcar falha na outbox sem DLQ separada descartado | TRANSCRICAO | [09:18] Diego |
| RFC-QO-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-QO-02 | docs/RFC.md | Questão em Aberto | Escala multi-worker e ordering por order_id | TRANSCRICAO | [09:13] Diego |
| RFC-QO-03 | docs/RFC.md | Questão em Aberto | E-mail de alerta em falhas recorrentes adiado | TRANSCRICAO | [09:37] Larissa |
| RFC-QO-04 | docs/RFC.md | Questão em Aberto | Arquivamento de eventos após 30 dias adiado | TRANSCRICAO | [09:08] Diego |
| RFC-QO-05 | docs/RFC.md | Questão em Aberto | Endurecimento de roles no CRUD adiado | TRANSCRICAO | [09:37] Sofia |
| RFC-RISK-01 | docs/RFC.md | Risco | Cliente indisponível e evento em DLQ sem alerta proativo | TRANSCRICAO | [09:17] Diego |
| RFC-RISK-02 | docs/RFC.md | Risco | Crescimento da outbox e atraso do worker | TRANSCRICAO | [09:08] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Vazamento de secret ou assinatura | TRANSCRICAO | [09:22] Sofia |
| RFC-RISK-04 | docs/RFC.md | Risco | Prazo de 3 sprints e revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| RFC-RISK-05 | docs/RFC.md | Risco | Duplicidade de entregas | TRANSCRICAO | [09:26] Marcos |
| RFC-RISK-06 | docs/RFC.md | Risco | Payload grande falhar a transação de status | TRANSCRICAO | [09:24] Larissa |
| RFC-ADR-01 | docs/RFC.md | Decisão Relacionada | Link para ADR-001, outbox no MySQL | TRANSCRICAO | [09:48] Larissa |
| RFC-ADR-02 | docs/RFC.md | Decisão Relacionada | Link para ADR-002, worker separado em polling | TRANSCRICAO | [09:48] Larissa |
| RFC-ADR-03 | docs/RFC.md | Decisão Relacionada | Link para ADR-003, retry e DLQ | TRANSCRICAO | [09:48] Larissa |
| RFC-ADR-04 | docs/RFC.md | Decisão Relacionada | Link para ADR-004, HMAC-SHA256 por endpoint | TRANSCRICAO | [09:48] Larissa |
| RFC-ADR-05 | docs/RFC.md | Decisão Relacionada | Link para ADR-005, at-least-once com X-Event-Id | TRANSCRICAO | [09:48] Larissa |
| RFC-ADR-06 | docs/RFC.md | Decisão Relacionada | Link para ADR-006, reuso dos padrões existentes | TRANSCRICAO | [09:48] Larissa |
| RFC-ADR-07 | docs/RFC.md | Decisão Relacionada | Link para ADR-007, snapshot do payload | TRANSCRICAO | [09:48] Larissa |
| FDD-CTX-01 | docs/FDD.md | Contexto | Transação de status é pesada; HTTP síncrono bloquearia pedidos | TRANSCRICAO | [09:04] Bruno |
| FDD-CTX-02 | docs/FDD.md | Contexto | Aplicação não possui eventos, filas ou webhooks | TRANSCRICAO | [09:06] Diego |
| FDD-CTX-03 | docs/FDD.md | Contexto | Polling de 2s atende ao requisito de menos de 10s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-01 | docs/FDD.md | Objetivo Técnico | Atomicidade entre mudança de status e evento na outbox | TRANSCRICAO | [09:41] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo Técnico | Worker separado, polling de 2s e ordering por order_id | TRANSCRICAO | [09:09] Diego |
| FDD-OBJ-03 | docs/FDD.md | Objetivo Técnico | Retry com backoff e DLQ com replay | TRANSCRICAO | [09:15] Diego |
| FDD-OBJ-04 | docs/FDD.md | Objetivo Técnico | HMAC-SHA256 com secret por endpoint e rotação | TRANSCRICAO | [09:19] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Objetivo Técnico | At-least-once com X-Event-Id | TRANSCRICAO | [09:24] Diego |
| FDD-OBJ-06 | docs/FDD.md | Objetivo Técnico | Reuso dos padrões existentes sem novas bibliotecas | TRANSCRICAO | [09:27] Larissa |
| FDD-ESC-01 | docs/FDD.md | Escopo | Inclui modelos, módulo, worker, CRUD, histórico, replay e HMAC | TRANSCRICAO | [09:31] Marcos |
| FDD-ESC-02 | docs/FDD.md | Exclusão | Dashboard visual fora do escopo | TRANSCRICAO | [09:40] Larissa |
| FDD-ESC-03 | docs/FDD.md | Exclusão | E-mail de alerta fora do escopo | TRANSCRICAO | [09:37] Larissa |
| FDD-ESC-04 | docs/FDD.md | Exclusão | Rate limiting fora do escopo | TRANSCRICAO | [09:39] Larissa |
| FDD-ESC-05 | docs/FDD.md | Exclusão | Arquivamento após 30 dias fora do escopo | TRANSCRICAO | [09:08] Diego |
| FDD-ESC-06 | docs/FDD.md | Exclusão | Webhooks inbound fora do escopo | TRANSCRICAO | [09:02] Marcos |
| FDD-ESC-07 | docs/FDD.md | Exclusão | Múltiplos workers e ordering global fora do escopo | TRANSCRICAO | [09:13] Diego |
| FDD-ARQ-01 | docs/FDD.md | Arquitetura | Módulo src/modules/webhooks com controller, service, repository, routes e schemas | TRANSCRICAO | [09:27] Bruno |
| FDD-ARQ-02 | docs/FDD.md | Arquitetura | src/worker.ts como entry point e processor no módulo | TRANSCRICAO | [09:28] Bruno |
| FDD-ARQ-03 | docs/FDD.md | Arquitetura | Uma linha de outbox por endpoint interessado no status | TRANSCRICAO | [09:33] Marcos |
| FDD-ARQ-04 | docs/FDD.md | Arquitetura | eventStatuses em JSON por limitação do Prisma com MySQL | CODIGO | prisma/schema.prisma |
| FDD-ARQ-05 | docs/FDD.md | Arquitetura | UUID como chave primária das novas entidades | TRANSCRICAO | [09:51] Larissa |
| FDD-ARQ-06 | docs/FDD.md | Arquitetura | Payload snapshot na inserção, sem reler o pedido no envio | TRANSCRICAO | [09:52] Larissa |
| FDD-ARQ-07 | docs/FDD.md | Arquitetura | Índices em status, createdAt, orderId, customerId e tentativas | TRANSCRICAO | [09:08] Diego |
| FDD-DATA-01 | docs/FDD.md | Modelo de Dados | Modelo WebhookEndpoint com url, secret, previousSecret, eventStatuses e active | TRANSCRICAO | [09:21] Sofia |
| FDD-DATA-02 | docs/FDD.md | Modelo de Dados | Modelo WebhookOutbox com status, attempts, nextAttemptAt e payload | TRANSCRICAO | [09:08] Diego |
| FDD-DATA-03 | docs/FDD.md | Modelo de Dados | Modelo WebhookDelivery com payload, responseStatus, responseBody e durationMs | TRANSCRICAO | [09:34] Marcos |
| FDD-DATA-04 | docs/FDD.md | Modelo de Dados | Modelo WebhookDeadLetter com payload, motivo, timestamp e replay | TRANSCRICAO | [09:18] Diego |
| FDD-DATA-05 | docs/FDD.md | Modelo de Dados | Relação Customer.webhooks no schema Prisma | CODIGO | prisma/schema.prisma |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | changeStatus chama publishWebhookEvent dentro da transação | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Falha na outbox provoca rollback da transação de status | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | publishWebhookEvent é função pura que recebe o tx client | TRANSCRICAO | [09:41] Bruno |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Filtro de status aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Integração de referência no OrderService.changeStatus | TRANSCRICAO | [09:41] Bruno |
| FDD-FLOW-06 | docs/FDD.md | Fluxo | Ciclo do worker: claim, envio, marcação e espera de 2s | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-07 | docs/FDD.md | Fluxo | fetch com timeout de 10s e truncamento do responseBody | TRANSCRICAO | [09:42] Diego |
| FDD-FLOW-08 | docs/FDD.md | Fluxo | Retry com 1m/5m/30m/2h/12h e DLQ após o quinto retry | TRANSCRICAO | [09:17] Larissa |
| FDD-FLOW-09 | docs/FDD.md | Fluxo | Cada tentativa gera registro em webhook_deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-FLOW-10 | docs/FDD.md | Fluxo | DLQ com payload, motivo e replay transacional | TRANSCRICAO | [09:18] Diego |
| FDD-FLOW-11 | docs/FDD.md | Fluxo | Rotação de secret com previousSecret válido por 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-FLOW-12 | docs/FDD.md | Fluxo | Reenfileirar PROCESSING antigo para recuperar worker caído | TRANSCRICAO | [09:11] Diego |
| FDD-FLOW-13 | docs/FDD.md | Fluxo | DELETE remove configuração e históricos por cascade | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | Base path /api/v1 já montado no app | CODIGO | src/app.ts |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | Todas as rotas exigem JWT; CRUD qualquer role; replay ADMIN | TRANSCRICAO | [09:36] Larissa |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | Formato de erro com code, message e details | CODIGO | src/middlewares/error.middleware.ts |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | Paginação padrão com data e pagination | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks com request, response 201 e secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks com customerId e paginação | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id para url, eventStatuses e active | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id com resposta 204 | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret com grace de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries com últimas 100 tentativas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay com 202 | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | Headers e payload outbound order.status_changed | TRANSCRICAO | [09:44] Diego |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND para id inexistente | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL para URL não HTTPS | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED como invariante de criação e rotação | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_INVALID_EVENT_STATUS para filtro inválido | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE acima de 64 KB | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-06 | docs/FDD.md | Erro | WEBHOOK_INACTIVE para replay de endpoint inativo | TRANSCRICAO | [09:21] Sofia |
| FDD-ERR-07 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND para DLQ inexistente | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-08 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT para chamada acima de 10s | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-09 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_FAILED para resposta não-2xx ou erro de rede | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-10 | docs/FDD.md | Erro | Erros de infraestrutura reutilizados no formato do middleware | CODIGO | src/middlewares/error.middleware.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10s na chamada HTTP | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | 5 retries com backoff e nextAttemptAt | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-03 | docs/FDD.md | Resiliência | DLQ separada com replay manual | TRANSCRICAO | [09:18] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | At-least-once com dedup pelo cliente | TRANSCRICAO | [09:25] Diego |
| FDD-RES-05 | docs/FDD.md | Resiliência | Ordering por order_id com single-worker | TRANSCRICAO | [09:12] Diego |
| FDD-RES-06 | docs/FDD.md | Resiliência | Reenfileirar PROCESSING com mais de 5 minutos | TRANSCRICAO | [09:11] Diego |
| FDD-RES-07 | docs/FDD.md | Resiliência | Sem fallback por e-mail nesta fase | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-08 | docs/FDD.md | Resiliência | Sem rate limiting nesta fase | TRANSCRICAO | [09:39] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados com Pino e eventos do worker | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Nunca logar secret, previousSecret ou X-Signature | TRANSCRICAO | [09:22] Sofia |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Métricas emitidas como logs estruturados | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Correlação por eventId, webhookId, orderId e requestId | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Sem tracing distribuído; eventId é a chave de rastreio | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-06 | docs/FDD.md | Observabilidade | Auditoria do replay com replayedById e adminUserId | TRANSCRICAO | [09:36] Sofia |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node.js >= 20 e fetch global | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | MySQL 8 e Prisma 5.22 | CODIGO | prisma/schema.prisma |
| FDD-DEP-03 | docs/FDD.md | Dependência | Nenhuma nova biblioteca; crypto é built-in | CODIGO | package.json |
| FDD-DEP-04 | docs/FDD.md | Dependência | Deploy e supervisão do worker separado | TRANSCRICAO | [09:11] Larissa |
| FDD-DEP-05 | docs/FDD.md | Dependência | Revisão de segurança de 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| FDD-DEP-06 | docs/FDD.md | Compatibilidade | Nenhuma rota existente muda de contrato | CODIGO | src/app.ts |
| FDD-DEP-07 | docs/FDD.md | Compatibilidade | changeStatus mantém resposta e status codes | CODIGO | src/modules/orders/order.service.ts |
| FDD-DEP-08 | docs/FDD.md | Dependência | Worker usa PrismaClient próprio por processo | TRANSCRICAO | [09:30] Bruno |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent dentro da transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Usa OrderStatus e canTransition existentes | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Novos erros WEBHOOK_* estendem AppError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Middleware de erro existente trata AppError sem mudança | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate em todas as rotas e requireRole no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | buildControllers registra WebhookRepository, Service e Controller | CODIGO | src/app.ts |
| FDD-INT-07 | docs/FDD.md | Integração | buildApiRouter monta /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Worker usa createPrismaClient para instância própria | CODIGO | src/config/database.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Worker replica bootstrap e shutdown do server | CODIGO | src/server.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Reuso do Pino e extensão da lista de redact | CODIGO | src/shared/logger/index.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Novos models e relação Customer.webhooks no schema | CODIGO | prisma/schema.prisma |
| FDD-INT-12 | docs/FDD.md | Integração | Scripts worker e worker:start no package.json | CODIGO | package.json |
| FDD-INT-13 | docs/FDD.md | Integração | Reuso do padrão Supertest e factories nos testes | CODIGO | tests/orders.test.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Limpeza das novas tabelas antes de customer e user | CODIGO | tests/setup.ts |
| FDD-ACC-01 | docs/FDD.md | Critério Técnico | Migração cria tabelas e índices sem alterar colunas existentes | CODIGO | prisma/schema.prisma |
| FDD-ACC-02 | docs/FDD.md | Critério Técnico | Uma linha de outbox por endpoint ativo interessado | TRANSCRICAO | [09:33] Marcos |
| FDD-ACC-03 | docs/FDD.md | Critério Técnico | Falha na outbox causa rollback de status e estoque | TRANSCRICAO | [09:40] Bruno |
| FDD-ACC-04 | docs/FDD.md | Critério Técnico | Nenhum evento quando nenhum endpoint quer o status | TRANSCRICAO | [09:34] Bruno |
| FDD-ACC-05 | docs/FDD.md | Critério Técnico | Worker processa em ordem de createdAt, single-worker, a cada 2s | TRANSCRICAO | [09:10] Larissa |
| FDD-ACC-06 | docs/FDD.md | Critério Técnico | Retry segue 1m/5m/30m/2h/12h e DLQ após o quinto retry | TRANSCRICAO | [09:17] Larissa |
| FDD-ACC-07 | docs/FDD.md | Critério Técnico | DLQ criada e replay exige ADMIN com auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-ACC-08 | docs/FDD.md | Critério Técnico | HMAC confere e rotação mantém secret anterior por 24h | TRANSCRICAO | [09:22] Sofia |
| FDD-ACC-09 | docs/FDD.md | Critério Técnico | URL http é rejeitada e payload acima de 64 KB é rejeitado | TRANSCRICAO | [09:24] Larissa |
| FDD-ACC-10 | docs/FDD.md | Critério Técnico | Histórico retorna últimas 100 tentativas com payload e resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-ACC-11 | docs/FDD.md | Critério Técnico | Logs incluem eventId, webhookId, attempt, statusCode e durationMs | CODIGO | src/shared/logger/index.ts |
| FDD-ACC-12 | docs/FDD.md | Critério Técnico | Testes existentes de pedidos continuam passando | CODIGO | tests/orders.test.ts |
| FDD-RISK-01 | docs/FDD.md | Risco | Inserção extra na transação aumentar contenção | TRANSCRICAO | [09:04] Bruno |
| FDD-RISK-02 | docs/FDD.md | Risco | Crescimento da outbox e degradação do worker | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Worker cair com eventos em PROCESSING | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Vazamento de secret ou assinatura em log | TRANSCRICAO | [09:22] Sofia |
| FDD-RISK-05 | docs/FDD.md | Risco | Duplicidade de entregas confundir o cliente | TRANSCRICAO | [09:25] Diego |
| FDD-RISK-06 | docs/FDD.md | Risco | Rajadas de eventos sem rate limiting | TRANSCRICAO | [09:39] Larissa |
| FDD-RISK-07 | docs/FDD.md | Risco | Payload acima de 64 KB bloquear mudança de status | TRANSCRICAO | [09:24] Larissa |
| FDD-PLAN-01 | docs/FDD.md | Plano | Sprint 1: modelagem, CRUD, publisher e testes básicos | TRANSCRICAO | [09:46] Larissa |
| FDD-PLAN-02 | docs/FDD.md | Plano | Sprint 2: worker, retry, DLQ, replay, HMAC e rotação | TRANSCRICAO | [09:46] Larissa |
| FDD-PLAN-03 | docs/FDD.md | Plano | Sprint 3: deliveries, e2e, observabilidade, hardening e revisão de segurança | TRANSCRICAO | [09:46] Sofia |
| ADR-001 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Decisão | Outbox transacional no MySQL existente | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado com polling de 2s | TRANSCRICAO | [09:09] Diego |
| ADR-003 | docs/adrs/ADR-003-politica-de-retry-e-dlq.md | Decisão | Retry com backoff 1m/5m/30m/2h/12h e DLQ | TRANSCRICAO | [09:17] Larissa |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso dos padrões existentes do projeto | TRANSCRICAO | [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Snapshot do payload na inserção da outbox | TRANSCRICAO | [09:52] Larissa |

---

## Cobertura

- **Total de itens rastreados:** 260.
- **Fonte `TRANSCRICAO`:** 226 linhas (86,9%), todas com timestamp no formato `[hh:mm] Nome`.
- **Fonte `CODIGO`:** 34 linhas (13,1%), todas apontando para arquivos existentes no repositório.
- **Cobertura dos documentos:** todos os itens identificáveis com ID no PRD, RFC, FDD e ADRs possuem linha correspondente nesta tabela.
- **Observação:** itens marcados como derivação de implementação (por exemplo, uma linha de outbox por endpoint e a recuperação de `PROCESSING` preso) usam a fala que originou a decisão e estão explicados no documento correspondente.
