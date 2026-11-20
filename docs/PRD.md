# PRD — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Produto** | Order Management System (OMS) |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Status** | Aprovado para design |
| **Owner** | Marcos (Product Manager) |
| **Tech Lead** | Larissa |
| **Data** | 20/11/2026 |
| **Documentos relacionados** | [RFC](RFC.md), [FDD](FDD.md), [ADRs](adrs/), [Tracker](TRACKER.md) |

---

## 1. Resumo e contexto da feature

**PRD-CTX-01** — O OMS vai passar a oferecer **webhooks outbound** para clientes B2B: quando o status de um pedido muda, a plataforma envia uma notificação HTTP assinada para o endpoint HTTPS cadastrado pelo cliente, em vez de o cliente precisar consultar `GET /orders` repetidamente.

**PRD-CTX-02** — A feature nasce de um pedido formal de três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — que querem ser notificados em tempo real sobre mudanças de status. Para eles, "tempo real" significa qualquer latência **abaixo de 10 segundos** ([09:00]–[09:02] Marcos).

**PRD-CTX-03** — A aplicação atual não possui eventos, filas ou webhooks. O escopo é **outbound**: apenas a plataforma envia para o cliente; o cliente não envia eventos para a plataforma ([09:02] Marcos/Sofia).

---

## 2. Problema e motivação

**PRD-PROB-01** — Hoje os clientes B2B fazem polling em `GET /orders` de tempos em tempos para descobrir se o status mudou. Isso deixa a integração lenta e cara para eles, além de gerar carga desnecessária na API ([09:00] Marcos).

**PRD-PROB-02** — A Atlas sinalizou que, se a entrega não acontecer até o fim do trimestre, pode migrar para o concorrente. Há, portanto, uma motivação comercial concreta além da melhoria técnica ([09:00] Marcos).

**PRD-PROB-03** — Os clientes não pediram garantia de ordering global; pediram saber, de forma confiável e rápida, quando cada pedido deles mudou ([09:14] Marcos). A solução pode priorizar simplicidade e entrega em até 10 segundos.

**PRD-PROB-04** — A mudança de status já é uma operação transacional pesada — atualiza pedido, histórico e estoque. Uma chamada HTTP síncrona no meio dessa transação acoplaria a disponibilidade do cliente à operação de pedidos, o que é inaceitável ([09:04] Bruno).

---

## 3. Público-alvo e cenários de uso

### 3.1 Público-alvo

| Público | Necessidade |
| --- | --- |
| **Times de integração dos clientes B2B** (Atlas, MaxDistribuição, Nova Cargo) | Receber eventos de mudança de status, validar autenticidade, deduplicar e consultar histórico |
| **Usuários operadores do OMS** | Cadastrar, editar, listar e remover configurações de webhook dos clientes |
| **Administradores** | Reprocessar eventos em DLQ com segurança e auditoria |
| **Suporte / operação** | Diagnosticar entregas, acompanhar falhas e orientar clientes |

### 3.2 Cenários de uso

| ID | Cenário |
| --- | --- |
| **PRD-UC-01** | O time de integração da Atlas cadastra um endpoint HTTPS para receber apenas `SHIPPED` e `DELIVERED`. Quando o pedido muda para `SHIPPED`, a plataforma envia o evento assinado em menos de 10 segundos. |
| **PRD-UC-02** | O cliente valida a assinatura HMAC-SHA256 com a secret do endpoint e confirma que o payload não foi adulterado. |
| **PRD-UC-03** | O cliente recebe o mesmo evento duas vezes por causa de um retry e deduplica pelo `X-Event-Id`. |
| **PRD-UC-04** | O cliente consulta o histórico das últimas 100 entregas para investigar falhas e ver o tempo de resposta do seu endpoint. |
| **PRD-UC-05** | O cliente rotaciona a secret e mantém a antiga válida por 24 horas enquanto migra seus sistemas. |
| **PRD-UC-06** | Um administrador reprocessa um evento que esgotou as tentativas e foi para a DLQ, com registro de quem executou o replay. |

---

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Fonte |
| --- | --- | --- | --- | --- |
| **PRD-OBJ-01** | Entregar notificações em "tempo real" | Percentual de eventos elegíveis cuja primeira tentativa HTTP ocorre em até 10 segundos após o commit da mudança de status | **100%** | [09:02] Marcos, [09:10] Larissa |
| **PRD-OBJ-02** | Ativar os três clientes B2B solicitantes | Quantidade de clientes com pelo menos um webhook ativo e entrega bem-sucedida | **3 clientes** (Atlas, MaxDistribuição, Nova Cargo) até o fim de novembro | [09:00] Marcos, [09:45] Marcos |
| **PRD-OBJ-03** | Eliminar a dependência de polling manual | Quantidade de chamadas de polling necessárias para os clientes acompanharem mudanças de status | **0 chamadas de polling** por cliente integrado | [09:00]–[09:02] Marcos |
| **PRD-OBJ-04** | Garantir consistência entre status e evento | Quantidade de mudanças de status commitadas sem evento correspondente na outbox | **0 ocorrências** | [09:41] Diego |
| **PRD-OBJ-05** | Proteger as entregas | Percentual de entregas assinadas com HMAC-SHA256 e secret por endpoint | **100%** | [09:22] Sofia |

> **Nota sobre metas:** a meta de latência é expressa como 100% dos eventos em até 10 segundos porque o polling de 2 segundos foi aceito como suficiente para o pior caso ([09:10] Larissa). O acompanhamento deve considerar a latência de commit, o ciclo do worker e o tempo de resposta do cliente.

---

## 5. Escopo

### 5.1 Incluso no escopo

| ID | Item |
| --- | --- |
| **PRD-SCOPE-01** | Webhooks **outbound** de mudança de status de pedido ([09:02] Marcos) |
| **PRD-SCOPE-02** | Cadastro, edição, listagem e remoção de endpoints de webhook via API ([09:31]–[09:33] Marcos/Bruno) |
| **PRD-SCOPE-03** | Filtro de eventos por lista de status, por endpoint ([09:33] Marcos) |
| **PRD-SCOPE-04** | Histórico de entregas com sucesso/falha, payload, resposta e tempo ([09:34] Marcos) |
| **PRD-SCOPE-05** | Replay manual de DLQ por administrador ([09:18] Diego) |
| **PRD-SCOPE-06** | Rotação de secret com grace period de 24 horas ([09:21] Sofia) |

### 5.2 Fora de escopo

| ID | Item | Origem |
| --- | --- | --- |
| **PRD-OOS-01** | **Notificação por e-mail** quando o webhook do cliente falha repetidamente (ex.: 3 falhas seguidas). Fica para uma próxima fase, depois de medir o impacto. | [09:37] Larissa |
| **PRD-OOS-02** | **Rate limiting de envio** para evitar rajadas quando muitos pedidos mudam em pouco tempo. "Observar e decidir depois." | [09:38]–[09:39] Diego/Larissa |
| **PRD-OOS-03** | **Dashboard visual** para o cliente acompanhar webhooks. Painel é projeto separado do time de frontend; nesta fase, apenas endpoints. | [09:39]–[09:40] Marcos/Larissa |
| **PRD-OOS-04** | **Webhooks inbound** (cliente enviando eventos para a plataforma). O escopo é somente outbound. | [09:02] Marcos/Sofia |
| **PRD-OOS-05** | **Arquivamento de eventos entregues após 30 dias**. Fica fora desta feature. | [09:08] Diego |
| **PRD-OOS-06** | **Múltiplos workers e ordering global**. Nesta fase, um único worker e ordering apenas por `order_id`. | [09:12]–[09:13] Diego/Larissa |
| **PRD-OOS-07** | **Endurecimento de roles no CRUD**: nesta fase, qualquer role autenticada pode configurar webhooks; a restrição fica para depois. | [09:36]–[09:37] Sofia |

---

## 6. Requisitos funcionais

| ID | Requisito | Prioridade |
| --- | --- | --- |
| **PRD-FR-01** | O cliente deve conseguir **cadastrar** um webhook via `POST`, informando `customerId`, `url` e a lista de status que deseja receber. A secret é gerada pela plataforma e devolvida na criação. | Must |
| **PRD-FR-02** | O cliente deve conseguir **editar** um webhook via `PATCH`, alterando URL, lista de status e estado ativo. | Must |
| **PRD-FR-03** | O cliente deve conseguir **remover** um webhook via `DELETE`. | Must |
| **PRD-FR-04** | O cliente deve conseguir **listar** os webhooks de um `customerId` via `GET`, com paginação. | Must |
| **PRD-FR-05** | Cada endpoint deve ter um **filtro de eventos por status** (por exemplo, "só quero `SHIPPED` e `DELIVERED`"). | Must |
| **PRD-FR-06** | Quando o status de um pedido muda, a plataforma deve **enviar um evento outbound** com payload definido, desde que exista pelo menos um endpoint interessado. | Must |
| **PRD-FR-07** | O filtro de status deve ser aplicado **na inserção do evento**: se nenhum webhook do cliente quer aquele status, nenhum evento é criado. | Must |
| **PRD-FR-08** | O cliente deve conseguir consultar o **histórico das últimas 100 entregas** do endpoint, com sucesso/falha, payload, resposta e tempo de resposta. | Must |
| **PRD-FR-09** | Um administrador deve conseguir fazer **replay manual** de um evento em DLQ, recolocando-o como pendente. | Must |
| **PRD-FR-10** | O cliente deve conseguir **rotacionar a secret**, mantendo a anterior válida por 24 horas em paralelo. | Must |
| **PRD-FR-11** | As entregas devem ser assinadas com **HMAC-SHA256** sobre o corpo do request, usando secret **por endpoint**. | Must |
| **PRD-FR-12** | Cada evento deve carregar um **`X-Event-Id`** único (UUID) para o cliente deduplicar entregas repetidas. | Must |
| **PRD-FR-13** | Falhas de entrega devem ser **retentadas com backoff** e, esgotadas as tentativas, movidas para uma **DLQ**. | Must |

---

## 7. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| **PRD-NFR-01** | **Latência:** a primeira tentativa de entrega deve ocorrer em até **10 segundos** após o commit da mudança de status, no pior caso ([09:02] Marcos, [09:10] Larissa). |
| **PRD-NFR-02** | **Segurança de transporte:** a URL do webhook deve ser **HTTPS**; cadastros com `http://` devem ser recusados ([09:23] Sofia). |
| **PRD-NFR-03** | **Limite de payload:** o evento deve ter no máximo **64 KB**; se ultrapassar, deve ocorrer erro, sem truncamento ([09:24] Larissa). |
| **PRD-NFR-04** | **Timeout de entrega:** a chamada HTTP do worker deve ter timeout de **10 segundos**; cliente que não responder nesse prazo é tratado como falha ([09:42] Diego). |
| **PRD-NFR-05** | **Garantia de entrega:** **at-least-once**; duplicatas são possíveis e o cliente deve deduplicar pelo `X-Event-Id` ([09:24]–[09:25] Diego). |
| **PRD-NFR-06** | **Ordering:** garantia apenas **por `order_id`** e enquanto houver um único worker; não há garantia de ordering global entre pedidos ([09:12]–[09:13] Diego/Larissa). |
| **PRD-NFR-07** | **Autenticação e autorização:** CRUD exige JWT autenticado; replay de DLQ exige role `ADMIN` e auditoria de quem executou ([09:36] Larissa/Sofia). |
| **PRD-NFR-08** | **Reuso de stack:** a feature deve usar MySQL, Prisma, Pino, Zod, Express e JWT já existentes, sem nova infraestrutura ([09:07] Diego, [09:30] Larissa). |
| **PRD-NFR-09** | **Isolamento do worker:** o processamento deve rodar em **processo separado** da API, para não ser perdido em restart da API ([09:11] Diego). |
| **PRD-NFR-10** | **Payload enxuto:** o evento não deve incluir os itens do pedido; se o cliente quiser detalhes, consulta `GET /orders/:id` ([09:43] Diego). |

---

## 8. Decisões e trade-offs principais

| ID | Decisão | Trade-off |
| --- | --- | --- |
| **PRD-DEC-01** | Usar **outbox transacional no MySQL** em vez de chamada HTTP síncrona no `changeStatus` | Ganha-se atomicidade e isolamento da disponibilidade do cliente; aceita-se latência de até ~2 segundos e uma escrita adicional na transação ([09:04] Bruno, [09:06]–[09:08] Diego) |
| **PRD-DEC-02** | Garantir **at-least-once** em vez de exactly-once | Simplicidade e robustez; o cliente precisa deduplicar e pode receber o mesmo evento mais de uma vez ([09:25] Diego) |
| **PRD-DEC-03** | Usar **DLQ em tabela separada** em vez de marcar falha na outbox | Operação e auditoria melhores; mais uma tabela e um fluxo de replay para manter ([09:18] Diego) |
| **PRD-DEC-04** | Usar **secret por endpoint** em vez de secret global | Limita o impacto de um vazamento; aumenta a gestão de secrets e a complexidade de rotação ([09:21] Sofia) |
| **PRD-DEC-05** | Rodar **um único worker** nesta fase | Simplicidade e ordering por `order_id`; sem escala horizontal e com throughput limitado ([09:12]–[09:13] Diego) |
| **PRD-DEC-06** | Deixar **e-mail, rate limiting e dashboard fora desta fase** | Foco e prazo; o cliente não terá alerta proativo nem painel, e rajadas não serão limitadas ([09:37]–[09:40] Larissa) |

---

## 9. Dependências

| ID | Dependência | Impacto |
| --- | --- | --- |
| **PRD-DEP-01** | **MySQL e Prisma existentes** | A outbox, DLQ, deliveries e configuração usam o banco atual, sem nova infraestrutura ([09:07] Diego) |
| **PRD-DEP-02** | **JWT e roles existentes** (`ADMIN`/`OPERATOR`) | O CRUD e o replay reutilizam a autenticação atual; não há novo provedor de identidade ([09:36] Larissa) |
| **PRD-DEP-03** | **Worker em processo separado** | Requer deploy e supervisão de um novo processo, com o mesmo banco e a mesma stack ([09:30] Bruno) |
| **PRD-DEP-04** | **Endpoint HTTPS do cliente** | O cliente precisa expor uma URL pública HTTPS, validar HMAC e deduplicar por `X-Event-Id` ([09:20] Sofia, [09:25] Diego) |
| **PRD-DEP-05** | **Revisão de segurança da Sofia** | Pelo menos 2 dias úteis antes do deploy, com foco em HMAC e geração de secret ([09:46] Sofia) |
| **PRD-DEP-06** | **Prazo comercial da Atlas** | Fim de novembro; a estimativa é de 3 sprints incluindo a revisão de segurança ([09:45]–[09:46] Marcos/Larissa) |

---

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| **PRD-RISK-01** | Cliente indisponível por longos períodos; evento termina na DLQ sem alerta proativo | Média | Alto | Retry por ~15 horas, DLQ com replay manual; e-mail fica para fase futura ([09:15]–[09:18] Diego, [09:37] Larissa) |
| **PRD-RISK-02** | Crescimento da outbox e atraso do worker, aumentando a latência | Média | Alto | Índices em status/`created_at`, leitura em lote pequeno, monitoramento; arquivamento após 30 dias em fase futura ([09:08] Diego) |
| **PRD-RISK-03** | Vazamento de secret ou assinatura em log | Média | Alto | Secret por endpoint, rotação com grace de 24 h, redaction no Pino, revisão de segurança ([09:21]–[09:22] Sofia) |
| **PRD-RISK-04** | Prazo de 3 sprints + revisão de segurança não atender ao fim de novembro da Atlas | Alta | Alto | Escopo enxuto (sem dashboard/e-mail/rate limiting), revisão da Sofia agendada, acompanhamento semanal ([09:45]–[09:47] Marcos/Larissa/Sofia) |
| **PRD-RISK-05** | Duplicidade de entregas confundir o cliente | Média | Médio | `X-Event-Id` e documentação destacada no portal de desenvolvedor ([09:26] Marcos) |
| **PRD-RISK-06** | Payload anormalmente grande falhar a mudança de status | Baixa | Médio | Payload enxuto sem itens, limite de 64 KB com erro explícito ([09:43] Diego, [09:24] Larissa) |

---

## 11. Critérios de aceitação

| ID | Critério |
| --- | --- |
| **PRD-ACC-01** | É possível criar, editar, listar e remover webhooks autenticados por JWT, com `customerId` explícito e filtro de status por endpoint. |
| **PRD-ACC-02** | Toda mudança de status elegível gera evento na outbox na mesma transação; se a outbox falhar, o status e o estoque não são alterados. |
| **PRD-ACC-03** | Um evento elegível é entregue ao endpoint do cliente em até 10 segundos após o commit, no pior caso. |
| **PRD-ACC-04** | O cliente consegue validar a assinatura HMAC-SHA256 com a secret do endpoint; a rotação mantém a secret anterior válida por 24 horas. |
| **PRD-ACC-05** | Falhas seguem a política de retry com backoff e, esgotadas as tentativas, o evento aparece na DLQ. |
| **PRD-ACC-06** | O replay de DLQ exige `ADMIN`, recoloca o evento como pendente e registra quem executou. |
| **PRD-ACC-07** | O histórico retorna as últimas 100 entregas com sucesso/falha, payload, resposta e tempo de resposta. |
| **PRD-ACC-08** | URL `http://` é recusada e payload acima de 64 KB é rejeitado com erro. |
| **PRD-ACC-09** | Os itens fora de escopo (e-mail, rate limiting, dashboard, inbound, arquivamento, multi-worker, endurecimento de roles) **não** são entregues nesta fase. |

---

## 12. Estratégia de testes e validação

| ID | Estratégia |
| --- | --- |
| **PRD-TEST-01** | **Testes unitários** para cálculo de retry/backoff, assinatura HMAC, validação de URL HTTPS, filtro de status e montagem do payload. |
| **PRD-TEST-02** | **Testes de integração** com Supertest para os contratos de CRUD, rotação de secret, histórico e autorização `ADMIN`/`OPERATOR`, reutilizando o padrão de `tests/orders.test.ts` e `tests/auth.test.ts`. |
| **PRD-TEST-03** | **Testes do worker** com servidor HTTP mock: sucesso 2xx, erro 5xx, timeout de 10 s, retry com relógio controlado e transição para DLQ ([09:17] Larissa). |
| **PRD-TEST-04** | **Teste ponta a ponta**: mudança de status → outbox → worker → endpoint mock, verificando payload, headers, assinatura e `X-Event-Id` ([09:40] Bruno). |
| **PRD-TEST-05** | **Testes de segurança**: adulteração de payload, assinatura inválida, rotação com janela de 24 h, rejeição de `http://` e payload > 64 KB ([09:46] Sofia). |
| **PRD-TEST-06** | **Regressão**: a suíte existente de pedidos (`tests/orders.test.ts`) deve continuar passando, garantindo que `changeStatus` mantém o comportamento externo. |

**Validação com o cliente:** antes do rollout geral, validar com pelo menos um dos três clientes B2B um endpoint de homologação, cobrindo cadastro, recebimento, verificação de assinatura e deduplicação.

---

## 13. Referências

- `TRANSCRICAO.md` — reunião técnica completa.
- [RFC](RFC.md) — proposta técnica, alternativas e questões em aberto.
- [FDD](FDD.md) — contratos, fluxos, erros e integração com o código.
- [ADRs](adrs/) — decisões arquiteturais.
- [Tracker](TRACKER.md) — rastreabilidade de cada item à transcrição ou ao código.
