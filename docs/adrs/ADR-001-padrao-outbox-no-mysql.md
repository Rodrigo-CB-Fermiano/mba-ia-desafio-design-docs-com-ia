# ADR-001 — Padrão Outbox no MySQL

## Status

Aceito

- **Data:** decisão registrada na reunião técnica de quinta-feira (ver `TRANSCRICAO.md`); ADR consolidado em 20/11/2026
- **Decisores:** Larissa (Tech Lead), Diego (Engenharia Sênior), Bruno (Engenharia Pleno)
- **Contexto da decisão:** reunião técnica do Sistema de Webhooks de Notificação de Pedidos

## Contexto

A mudança de status de um pedido já executa uma transação pesada no `OrderService.changeStatus`: atualiza a tabela `orders`, insere em `order_status_history` e, dependendo da transição, decrementa ou repõe `stock_quantity` dos produtos do pedido.

A aplicação não possui eventos, filas ou qualquer mecanismo de notificação externa. A proposta discutida precisa notificar clientes B2B quando o status muda, sem acoplar a disponibilidade do cliente à transação de pedido.

Uma chamada HTTP síncrona dentro de `changeStatus` foi descartada: um cliente lento bloquearia a mudança de status de outros pedidos e não faria sentido dar rollback na mudança de status porque o cliente está fora do ar ([09:04] Bruno). Também foi descartado subir Redis Streams ou outra infraestrutura de fila para um time pequeno ([09:07] Diego).

## Decisão

Adotar o **padrão Outbox transacional no MySQL existente**:

1. Criar a tabela `webhook_outbox` no mesmo banco MySQL da aplicação.
2. Inserir a linha de evento na `webhook_outbox` **dentro da mesma transação SQL** que atualiza `orders`, `order_status_history` e estoque.
3. Se a transação principal commitou, o evento está registrado; se ela deu rollback, o evento desaparece junto.
4. Um worker separado lê a outbox e realiza as chamadas HTTP.
5. A tabela tem índice no campo de status (`PENDING`, `PROCESSING`, `DELIVERED`, `FAILED`) e em `created_at`, e o worker lê apenas pendentes em lote pequeno.
6. As linhas entregues serão arquivadas após 30 dias em fase futura; esse arquivamento **não** faz parte desta feature ([09:08] Diego).

A granularidade adotada é **uma linha de outbox por endpoint de webhook interessado no status**. Isso permite que cada endpoint tenha sua própria política de retry, DLQ, histórico e `X-Webhook-Id`, sem fan-out implícito no worker. A decisão é uma derivação de implementação das falas sobre múltiplos endpoints e filtro por status ([09:33] Marcos, [09:44] Sofia).

O `id` da outbox é UUID, seguindo o padrão do projeto ([09:51] Larissa). O payload é gravado como snapshot no momento da inserção ([09:52] Larissa) — ver ADR-007.

## Alternativas Consideradas

### Alternativa A — Chamada HTTP síncrona dentro de `changeStatus`

- **Trade-off que levou ao descarte:** qualquer cliente lento travaria a transação de mudança de status; se o cliente estivesse fora do ar, não haveria resposta coerente (não se dá rollback na mudança de status por causa do cliente) ([09:04] Bruno).
- **Custo evitado:** acoplamento da disponibilidade da plataforma à disponibilidade de cada cliente.

### Alternativa B — Redis Streams, RabbitMQ ou outra fila externa

- **Trade-off que levou ao descarte:** exigiria subir e operar mais infraestrutura; para o tamanho do time, seria overengineering ([09:07] Diego).
- **Custo evitado:** complexidade operacional e novos componentes para monitorar.

### Alternativa C — Trigger de banco para notificar processo externo

- **Trade-off que levou ao descarte:** MySQL não tem listener nativo como `NOTIFY`/`LISTEN` do PostgreSQL; uma trigger só executa SQL e não notifica um processo externo ([09:09] Diego).
- **Custo evitado:** soluções improvisadas de trigger gravando arquivo ou batendo em endpoint.

### Alternativa D — Outbox em banco separado

- **Trade-off que levou ao descarte:** quebraria a atomicidade com a transação de pedidos, que é exatamente o requisito central; exigiria coordenação distribuída.
- **Custo evitado:** complexidade de consistência entre dois bancos.

## Consequências

### Positivas

- **Atomicidade:** não existe estado em que o status mudou e o evento não foi registrado, nem o contrário.
- **Desacoplamento:** a disponibilidade do cliente não afeta a transação de pedido.
- **Simplicidade operacional:** usa o MySQL e o Prisma já existentes, sem nova infraestrutura.
- **Auditoria:** a outbox preserva o payload snapshot e o histórico de tentativas.
- **Evolução:** a mesma tabela permite retry, DLQ e replay manual.

### Negativas / trade-offs

- **Escrita adicional:** toda mudança de status com pelo menos um endpoint interessado passa a fazer uma inserção extra dentro da transação.
- **Latência de entrega:** a entrega não é imediata; o worker em polling de 2 segundos impõe latência mínima de até 2 segundos, aceita porque o requisito é abaixo de 10 segundos.
- **Crescimento da tabela:** a outbox e o histórico de entregas crescem com o volume; o arquivamento após 30 dias ficou fora desta fase.
- **Dependência do worker:** sem o worker em execução, os eventos acumulam e não são entregues.
- **Granularidade por endpoint:** uma mudança de status pode gerar várias linhas (uma por endpoint interessado), aumentando o volume.

## Referências

- `TRANSCRICAO.md` — [09:04] Bruno; [09:06] Diego; [09:07] Larissa/Diego; [09:08] Diego; [09:40] Bruno; [09:41] Diego; [09:51] Larissa; [09:52] Larissa
- `src/modules/orders/order.service.ts` — transação atual de `changeStatus`
- `src/modules/orders/order.status.ts` — máquina de estados
- `prisma/schema.prisma` — modelos e banco MySQL
- `prisma/migrations/20260519182739_init/migration.sql` — convenções de nomes e tipos
- `src/config/database.ts` — criação do `PrismaClient`
