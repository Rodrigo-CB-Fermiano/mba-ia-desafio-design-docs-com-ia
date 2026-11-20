# ADR-002 — Worker em processo separado com polling de 2 segundos

## Status

Aceito

- **Data:** decisão registrada na reunião técnica de quinta-feira (ver `TRANSCRICAO.md`); ADR consolidado em 20/11/2026
- **Decisores:** Larissa (Tech Lead), Diego (Engenharia Sênior), Bruno (Engenharia Pleno)
- **Contexto da decisão:** reunião técnica do Sistema de Webhooks de Notificação de Pedidos

## Contexto

Definido o padrão Outbox no MySQL (ADR-001), é preciso decidir **quem** lê a outbox e **como** esse consumidor é executado.

O MySQL não possui um mecanismo nativo de notificação para processos externos equivalente ao `NOTIFY`/`LISTEN` do PostgreSQL. Uma trigger de banco executa SQL, mas não acorda um processo Node ([09:09] Diego). Além disso, o worker não pode rodar dentro da mesma instância da API: se a API reiniciar, o worker seria perdido junto ([09:11] Diego).

A latência mínima aceita para a entrega é de 2 segundos no pior caso, compatível com o requisito de "abaixo de 10 segundos" ([09:10] Larissa). Os clientes não pediram garantia de ordering global; pediram saber se cada pedido mudou ([09:14] Marcos).

## Decisão

1. Criar uma **entry point separada** `src/worker.ts`, seguindo o padrão de bootstrap de `src/server.ts`.
2. Adicionar um script `npm run worker` no `package.json`.
3. O worker roda em **processo Node separado** da API.
4. O worker usa o **mesmo banco** (`DATABASE_URL`) e a mesma stack, mas cria sua **própria instância de `PrismaClient`**, porque `PrismaClient` é por processo ([09:30] Bruno). A criação deve reutilizar `createPrismaClient()` de `src/config/database.ts`.
5. O worker executa um **loop de polling a cada 2 segundos**, busca os eventos pendentes mais antigos em lote pequeno e processa em ordem de `created_at`.
6. Nesta fase, roda **um único worker**. A ordering é garantida **por `order_id`** e apenas enquanto houver um worker; não há garantia de ordering global entre pedidos diferentes ([09:12]–[09:13] Diego).
7. A lógica de processamento fica no módulo de webhooks, em `src/modules/webhooks/webhook.processor.ts`; `src/worker.ts` é apenas o bootstrap e o loop ([09:28] Bruno).
8. A evolução para múltiplos workers (particionamento por `order_id` ou lock pessimista) fica para o futuro ([09:13] Diego).

## Alternativas Consideradas

### Alternativa A — Worker dentro da mesma instância da API (`setInterval` no `server.ts`)

- **Trade-off que levou ao descarte:** se a API reiniciar, o worker morre junto; além disso, o processamento competiria com o tráfego HTTP da API ([09:11] Diego).
- **Custo evitado:** perda de eventos em deploy/restart e acoplamento de carga.

### Alternativa B — Trigger de banco para acordar o worker

- **Trade-off que levou ao descarte:** MySQL não tem listener nativo; a trigger não notifica processo externo e exigiria improvisar arquivo ou endpoint ([09:09] Diego).
- **Custo evitado:** solução frágil e não portável.

### Alternativa C — Fila externa (Redis Streams, RabbitMQ, SQS)

- **Trade-off que levou ao descarte:** subiria mais infraestrutura para um time pequeno; o outbox no MySQL existente resolve com menos complexidade ([09:07] Diego).
- **Custo evitado:** operação de um novo componente e duplicidade de mecanismos de persistência.

### Alternativa D — Múltiplos workers em paralelo desde o início

- **Trade-off que levou ao descarte:** perderia a ordering implícita por `order_id`; exigiria particionamento ou lock pessimista, que é problema do futuro ([09:13] Diego).
- **Custo evitado:** complexidade prematura de concorrência.

## Consequências

### Positivas

- **Isolamento:** restart da API não interrompe o worker; restart do worker não derruba a API.
- **Simplicidade:** polling de 2 segundos atende ao requisito de latência sem nova infraestrutura.
- **Ordering previsível por pedido:** com um único worker e ordenação por `created_at`, os eventos de um mesmo `order_id` são processados na ordem em que entraram na outbox.
- **Operação independente:** o worker pode ser escalado verticalmente ou monitorado separadamente.

### Negativas / trade-offs

- **Mais um processo para operar:** deploy, supervisão, logs e alertas precisam cobrir o worker.
- **Latência mínima de até 2 segundos:** aceita conscientemente; não é entrega síncrona.
- **Sem escala horizontal nesta fase:** um único worker limita throughput e mantém a garantia de ordering apenas por `order_id`.
- **Eventos presos em `PROCESSING`:** se o worker cair no meio do processamento, será necessário um mecanismo de recuperação (reenfileirar `PROCESSING` antigos), detalhado no FDD.
- **Dependência operacional:** se o worker não estiver rodando, os eventos acumulam na outbox e não são entregues.

## Referências

- `TRANSCRICAO.md` — [09:09] Diego; [09:10] Larissa; [09:11] Diego/Larissa/Bruno; [09:12]–[09:13] Diego/Larissa; [09:14] Marcos; [09:28] Bruno; [09:30] Bruno
- `src/server.ts` — padrão de bootstrap e shutdown
- `src/config/database.ts` — `createPrismaClient()`
- `package.json` — scripts e engine Node
- `src/modules/orders/order.service.ts` — transação que alimenta a outbox
