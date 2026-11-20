# FDD — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Status** | Pronto para implementação |
| **Autor** | Larissa (Tech Lead) |
| **Revisores** | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança), Marcos (Produto) |
| **Última atualização** | 20/11/2026 |
| **Documentos relacionados** | [PRD](PRD.md), [RFC](RFC.md), [ADRs](adrs/) |
| **Escopo** | Especificação de implementação da feature de webhooks outbound de mudança de status |

---

## 1. Contexto e motivação técnica

**FDD-CTX-01** — O `OrderService.changeStatus` já executa uma transação que atualiza `orders`, insere em `order_status_history` e, conforme a transição, debita ou repõe `stock_quantity` dos produtos. Acrescentar uma chamada HTTP nessa transação faria um cliente lento bloquear mudanças de status de outros pedidos e impediria um rollback coerente caso o cliente estivesse fora do ar ([09:04] Bruno).

**FDD-CTX-02** — A aplicação não possui eventos, filas ou webhooks. A solução precisa criar o mecanismo de captura e entrega sem subir nova infraestrutura, usando o MySQL e o Prisma existentes ([09:06]–[09:07] Diego).

**FDD-CTX-03** — O requisito de latência é "abaixo de 10 segundos", e um polling de 2 segundos atende com folga ([09:02] Marcos, [09:10] Larissa).

---

## 2. Objetivos técnicos

| ID | Objetivo técnico |
| --- | --- |
| **FDD-OBJ-01** | Garantir atomicidade entre a mudança de status e o registro do evento na outbox: se a transação principal commitou, o evento existe; se deu rollback, o evento desaparece ([09:41] Diego). |
| **FDD-OBJ-02** | Processar eventos em worker separado, com polling de 2 segundos e ordering por `order_id` enquanto houver um único worker ([09:09]–[09:13] Diego). |
| **FDD-OBJ-03** | Implementar retry com backoff 1 m/5 m/30 m/2 h/12 h e DLQ em tabela separada, com replay manual ([09:15]–[09:18] Diego). |
| **FDD-OBJ-04** | Assinar entregas com HMAC-SHA256, secret por endpoint e rotação com grace period de 24 h ([09:19]–[09:22] Sofia). |
| **FDD-OBJ-05** | Garantir at-least-once com `X-Event-Id` para deduplicação no cliente ([09:24]–[09:26] Diego). |
| **FDD-OBJ-06** | Reutilizar os padrões existentes do projeto — módulos, `AppError`, Pino, error middleware, Zod, `requireRole` — sem nova biblioteca de log, validação ou erro ([09:27]–[09:30] Larissa). |

---

## 3. Escopo e exclusões

| ID | Item | Situação |
| --- | --- | --- |
| **FDD-ESC-01** | Modelos de configuração, outbox, deliveries e DLQ; módulo `src/modules/webhooks`; worker; CRUD; histórico; replay; HMAC; migração; testes | **Incluído** ([09:31]–[09:36] Marcos/Bruno) |
| **FDD-ESC-02** | Dashboard visual para o cliente | **Excluído** — apenas endpoints; painel é projeto separado do time de frontend ([09:40] Larissa) |
| **FDD-ESC-03** | E-mail de alerta em falhas recorrentes | **Excluído** — possível próxima fase após medir impacto ([09:37] Larissa) |
| **FDD-ESC-04** | Rate limiting de envio | **Excluído** — observar e decidir depois ([09:39] Larissa) |
| **FDD-ESC-05** | Arquivamento de eventos entregues após 30 dias | **Excluído** — fora do escopo desta feature ([09:08] Diego) |
| **FDD-ESC-06** | Webhooks inbound (cliente enviando para a plataforma) | **Excluído** — somente outbound ([09:02] Marcos) |
| **FDD-ESC-07** | Múltiplos workers e ordering global | **Excluído** — single-worker nesta fase; evolução futura ([09:13] Diego) |

---

## 4. Arquitetura e modelo de dados

### 4.1 Estrutura do módulo

**FDD-ARQ-01** — O módulo segue o padrão dos demais domínios, com os cinco arquivos principais e helpers específicos:

```text
src/modules/webhooks/                  (a criar)
├── webhook.controller.ts              # handlers HTTP
├── webhook.service.ts                 # regras de negócio do CRUD/rotação/replay
├── webhook.repository.ts              # acesso a dados via Prisma
├── webhook.routes.ts                  # rotas autenticadas /webhooks
├── webhook.admin.routes.ts            # rotas /admin/webhooks com requireRole
├── webhook.schemas.ts                 # schemas Zod
├── webhook.errors.ts                  # erros WEBHOOK_*
├── webhook.publisher.ts               # publishWebhookEvent(tx, ...)
├── webhook.payload.ts                 # builder do payload order.status_changed
├── webhook.signature.ts              # HMAC-SHA256
└── webhook.processor.ts               # ciclo do worker
```

**FDD-ARQ-02** — `src/worker.ts` (a criar) é apenas a entry point: cria o `PrismaClient`, chama o ciclo do processor a cada 2 segundos e trata shutdown. A lógica de processamento fica em `src/modules/webhooks/webhook.processor.ts` ([09:28] Bruno).

**FDD-ARQ-03** — A granularidade da outbox é **uma linha por endpoint de webhook interessado no status**. Se um cliente tem dois endpoints e ambos assinam `SHIPPED`, a mudança para `SHIPPED` gera duas linhas, cada uma com seu `webhookId`, permitindo retry, DLQ e histórico independentes. Se nenhum endpoint do cliente assina o status, nenhuma linha é criada ([09:33]–[09:34] Marcos/Bruno).

**FDD-ARQ-04** — `eventStatuses` é armazenado como **JSON** (array de strings do enum `OrderStatus`). O Prisma com MySQL não suporta listas escalares de enum; o filtro é feito em memória após carregar os endpoints ativos do cliente. O volume de endpoints por cliente é pequeno, então a abordagem é suficiente.

**FDD-ARQ-05** — Todas as novas entidades usam **UUID** como chave primária, seguindo o padrão do projeto ([09:51] Larissa).

**FDD-ARQ-06** — O payload é um **snapshot** gravado na inserção; o worker não relê o pedido ([09:52] Larissa — ADR-007).

**FDD-ARQ-07** — Índices obrigatórios: `webhook_outbox(status, createdAt)`, `webhook_outbox(status, nextAttemptAt)`, `webhook_outbox(orderId)`, `webhook_endpoints(customerId, active)`, `webhook_deliveries(webhookId, attemptedAt)` e `webhook_dead_letter(webhookId, failedAt)` ([09:08] Diego).

### 4.2 Modelo de dados proposto

> **Importante:** os modelos abaixo são **adições propostas** ao `prisma/schema.prisma`; a feature é documental e a implementação criará a migração correspondente.

**FDD-DATA-01** — Configuração de endpoint:

```prisma
model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(255)
  previousSecret          String?   @db.VarChar(255)
  previousSecretExpiresAt DateTime?
  eventStatuses           Json
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer    Customer           @relation(fields: [customerId], references: [id])
  outbox      WebhookOutbox[]
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}
```

**FDD-DATA-02** — Outbox transacional:

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  DELIVERED
  FAILED
}

model WebhookOutbox {
  id                  String               @id @default(uuid()) @db.Char(36)
  webhookId           String               @db.Char(36)
  eventId             String               @db.Char(36)
  orderId             String               @db.Char(36)
  customerId          String               @db.Char(36)
  eventType           String               @db.VarChar(64)
  payload             String               @db.MediumText
  status              WebhookOutboxStatus  @default(PENDING)
  attempts            Int                  @default(0)
  nextAttemptAt       DateTime?
  processingStartedAt DateTime?
  lastError           String?              @db.Text
  createdAt           DateTime             @default(now())
  updatedAt           DateTime             @updatedAt

  webhook    WebhookEndpoint  @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  deliveries WebhookDelivery[]

  @@unique([webhookId, eventId])
  @@index([status, createdAt])
  @@index([status, nextAttemptAt])
  @@index([orderId])
  @@map("webhook_outbox")
}
```

**FDD-DATA-03** — Histórico de tentativas de entrega:

```prisma
model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  webhookId      String   @db.Char(36)
  outboxId       String   @db.Char(36)
  eventId        String   @db.Char(36)
  attempt        Int
  success        Boolean
  requestPayload String   @db.MediumText
  responseStatus Int?
  responseBody   String?  @db.MediumText
  durationMs     Int
  errorCode      String?  @db.VarChar(64)
  attemptedAt    DateTime @default(now())

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  outbox  WebhookOutbox   @relation(fields: [outboxId], references: [id], onDelete: Cascade)

  @@index([webhookId, attemptedAt])
  @@map("webhook_deliveries")
}
```

**FDD-DATA-04** — Dead Letter Queue:

```prisma
model WebhookDeadLetter {
  id            String    @id @default(uuid()) @db.Char(36)
  webhookId     String    @db.Char(36)
  outboxId      String    @db.Char(36)
  eventId       String    @db.Char(36)
  orderId       String    @db.Char(36)
  customerId    String    @db.Char(36)
  eventType     String    @db.VarChar(64)
  payload       String    @db.MediumText
  failureReason String    @db.Text
  failedAt      DateTime  @default(now())
  replayedAt    DateTime?
  replayedById  String?   @db.Char(36)

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, failedAt])
  @@map("webhook_dead_letter")
}
```

**FDD-DATA-05** — Relação no model `Customer` existente:

```prisma
model Customer {
  // ... campos atuais ...
  webhooks WebhookEndpoint[]
}
```

> A relação não cria coluna em `customers`; apenas permite o `include`/`where` relacional no Prisma. A migração deve ser criada com `prisma migrate dev --name add_webhook_tables`.

### 4.3 Decisões de modelagem

- **`payload` como texto:** o snapshot é serializado uma vez e armazenado como string. É essa string que o worker envia e assina, garantindo que o HMAC corresponda exatamente aos bytes entregues.
- **`MEDIUMTEXT`:** o limite de 64 KB é maior que o máximo de `TEXT` (65.535 bytes); `MEDIUMTEXT` evita truncamento silencioso.
- **`attempts`:** conta retries já realizados; começa em 0. A tentativa inicial não é retry. O valor máximo é 5 antes da DLQ.
- **`nextAttemptAt`:** `null` significa "pronto para processar agora"; após falha, recebe o próximo horário permitido.
- **`processingStartedAt`:** usado para detectar eventos presos em `PROCESSING` e reenfileirá-los.
- **Delete de webhook:** o `DELETE` remove a configuração e, por cascade, os eventos/entregas/DLQ associados. Isso evita reter dados de pedidos de um cliente que pediu remoção; a consequência é que o histórico daquele endpoint deixa de existir.

---

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

**FDD-FLOW-01** — O `OrderService.changeStatus` continua executando a transação atual. Ao final dela, antes do commit, chama `publishWebhookEvent(tx, order, fromStatus, toStatus)`. A função recebe o `Prisma.TransactionClient`, não o `PrismaClient` ([09:41] Bruno/Diego).

**FDD-FLOW-02** — Se a inserção na outbox falhar por qualquer motivo — inclusive payload acima de 64 KB — a exceção propaga e a transação inteira sofre rollback. Não existe estado em que o status mudou e o evento não foi registrado ([09:40] Bruno, [09:41] Diego).

**FDD-FLOW-03** — `publishWebhookEvent` é uma função pura em relação a HTTP: não faz chamada externa, não usa `fetch` e não depende de estado global. Ela apenas lê os endpoints ativos do cliente, filtra os interessados, monta o snapshot e insere as linhas.

**FDD-FLOW-04** — O filtro é aplicado na inserção: se nenhum endpoint ativo do cliente assina `toStatus`, a função retorna `null` e não insere nada ([09:34] Bruno).

Implementação de referência:

```ts
// src/modules/webhooks/webhook.publisher.ts (a criar)
import { randomUUID } from 'node:crypto';
import type { Order, OrderStatus, Prisma } from '@prisma/client';
import { logger } from '../../shared/logger/index.js';
import { buildOrderStatusChangedPayload } from './webhook.payload.js';
import { WebhookPayloadTooLargeError } from './webhook.errors.js';

const MAX_PAYLOAD_BYTES = 64 * 1024;

export type PublishedWebhookEvent = {
  eventId: string;
  webhookIds: string[];
};

export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Order,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<PublishedWebhookEvent | null> {
  const endpoints = await tx.webhookEndpoint.findMany({
    where: { customerId: order.customerId, active: true },
  });

  const interested = endpoints.filter((endpoint) =>
    (endpoint.eventStatuses as OrderStatus[]).includes(toStatus),
  );
  if (interested.length === 0) return null;

  const eventId = randomUUID();
  const timestamp = new Date().toISOString();
  const payload = buildOrderStatusChangedPayload({
    eventId,
    order,
    fromStatus,
    toStatus,
    timestamp,
  });
  const body = JSON.stringify(payload);

  if (Buffer.byteLength(body, 'utf8') > MAX_PAYLOAD_BYTES) {
    throw new WebhookPayloadTooLargeError(Buffer.byteLength(body, 'utf8'));
  }

  await tx.webhookOutbox.createMany({
    data: interested.map((endpoint) => ({
      webhookId: endpoint.id,
      eventId,
      orderId: order.id,
      customerId: order.customerId,
      eventType: 'order.status_changed',
      payload: body,
      status: 'PENDING',
    })),
  });

  return { eventId, webhookIds: interested.map((endpoint) => endpoint.id) };
}
```

**FDD-FLOW-05** — A integração no `OrderService.changeStatus` fica assim (trecho ilustrativo, preservando a lógica atual):

```ts
// src/modules/orders/order.service.ts
const { order: updated, published } = await this.prisma.$transaction(async (tx) => {
  // ... validações e estoque atuais ...

  await tx.order.update({ where: { id }, data: { status: to } });
  await tx.orderStatusHistory.create({
    data: {
      orderId: id,
      fromStatus: from,
      toStatus: to,
      changedById: userId,
      reason: input.reason ?? null,
    },
  });

  const published = await publishWebhookEvent(tx, order, from, to);

  const refreshed = await tx.order.findUnique({
    where: { id },
    include: {
      items: { include: { product: { select: { id: true, sku: true, name: true } } } },
      history: { orderBy: { changedAt: 'asc' } },
      customer: { select: { id: true, name: true, email: true } },
    },
  });

  return { order: refreshed!, published };
});

if (published) {
  logger.info(
    { eventId: published.eventId, orderId: updated.id, webhooks: published.webhookIds.length },
    'webhook_event_enqueued',
  );
}

return updated;
```

> O log de enfileiramento é emitido **após o commit**, para não registrar um evento que sofreu rollback.

### 5.2 Processamento pelo worker

**FDD-FLOW-06** — O ciclo do worker:

1. Reenfileira eventos `PROCESSING` com `processingStartedAt` mais antigo que 5 minutos (recuperação de worker que caiu).
2. Busca eventos `PENDING` com `nextAttemptAt` nulo ou menor/igual a agora, ordenados por `createdAt` ascendente, em lote de 20.
3. Para cada evento, faz um "claim" atômico (`updateMany` de `PENDING` para `PROCESSING`); se outro worker já pegou, ignora.
4. Carrega o endpoint; se inativo, move para DLQ com motivo `WEBHOOK_INACTIVE`.
5. Assina o payload com a secret atual, monta os headers e faz `POST` para a URL do endpoint com timeout de 10 segundos.
6. Registra a tentativa em `webhook_deliveries`.
7. Em sucesso (HTTP 2xx), marca a outbox como `DELIVERED`.
8. Em falha, agenda retry ou move para DLQ (fluxo 5.3).
9. Dorme 2 segundos e repete.

Implementação de referência do loop:

```ts
// src/worker.ts (a criar)
import { createPrismaClient } from './config/database.js';
import { logger } from './shared/logger/index.js';
import { runWorkerCycle } from './modules/webhooks/webhook.processor.js';

const POLL_INTERVAL_MS = 2_000;
const prisma = createPrismaClient();
let stopping = false;

async function bootstrap(): Promise<void> {
  logger.info('webhook_worker_started');

  while (!stopping) {
    try {
      await runWorkerCycle(prisma);
    } catch (err) {
      logger.error({ err }, 'webhook_worker_cycle_failed');
    }
    await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL_MS));
  }

  await prisma.$disconnect();
  logger.info('webhook_worker_stopped');
}

process.on('SIGINT', () => { stopping = true; });
process.on('SIGTERM', () => { stopping = true; });

bootstrap().catch((err) => {
  logger.fatal({ err }, 'webhook_worker_bootstrap_failed');
  process.exit(1);
});
```

**FDD-FLOW-07** — O worker usa `fetch` global do Node 20 e `AbortController`/`AbortSignal.timeout(10_000)`. A resposta é lida como texto e limitada a **8 KB** para armazenamento em `webhook_deliveries.responseBody`; se for maior, é truncada com marcador `...[truncated]`. Redirecionamentos (`3xx`) são tratados como falha, não seguidos automaticamente.

### 5.3 Retry e backoff

**FDD-FLOW-08** — A política é: tentativa inicial + até **5 retries**, com atrasos de **1 m, 5 m, 30 m, 2 h e 12 h**. A janela total entre a primeira falha e a última tentativa é de **14 h 36 min**. Se o retry 5 falhar, o evento vai para a DLQ ([09:15]–[09:17] Diego/Larissa — ver nota de interpretação no ADR-003).

```ts
// src/modules/webhooks/webhook.processor.ts (a criar)
const RETRY_DELAYS_MS = [
  60_000,      // 1 minuto
  300_000,     // 5 minutos
  1_800_000,   // 30 minutos
  7_200_000,   // 2 horas
  43_200_000,  // 12 horas
];

// event.attempts = retries já realizados (0 na tentativa inicial)
const failedAttempts = event.attempts + 1;

if (failedAttempts > RETRY_DELAYS_MS.length) {
  await moveToDeadLetter(prisma, event, lastError);
} else {
  await prisma.webhookOutbox.update({
    where: { id: event.id },
    data: {
      status: 'PENDING',
      attempts: failedAttempts,
      nextAttemptAt: new Date(Date.now() + RETRY_DELAYS_MS[event.attempts]!),
      processingStartedAt: null,
      lastError,
    },
  });
}
```

**FDD-FLOW-09** — Cada tentativa gera uma linha em `webhook_deliveries` com `attempt` de 1 a 6 (1 = tentativa inicial; 6 = retry 5). O histórico do cliente mostra sucesso/falha, payload, resposta e tempo de resposta ([09:34] Marcos).

### 5.4 DLQ e replay

**FDD-FLOW-10** — Quando os retries se esgotam, o worker:

1. Cria uma linha em `webhook_dead_letter` com payload, motivo, `webhookId`, `eventId`, `orderId`, `customerId`, `eventType` e timestamp.
2. Marca a linha da outbox como `FAILED` e grava o último erro.
3. Emite o log `webhook_dead_letter_created`.

O replay:

1. Exige `ADMIN` ([09:36] Sofia).
2. Localiza a DLQ por id; se não existir, retorna `WEBHOOK_DEAD_LETTER_NOT_FOUND`.
3. Se o endpoint estiver inativo, retorna `WEBHOOK_INACTIVE`.
4. Em transação, atualiza a outbox original para `PENDING`, zera `attempts`, limpa `nextAttemptAt`, `processingStartedAt` e `lastError`.
5. Marca a DLQ com `replayedAt` e `replayedById` (auditoria).
6. Retorna `202 Accepted` com `outboxEventId` e `status: "PENDING"` ([09:18] Diego).

### 5.5 Rotação de secret

**FDD-FLOW-11** — A rotação:

1. Gera uma nova secret opaca (sugestão: `randomBytes(32).toString('hex')`).
2. Move a secret atual para `previousSecret` e define `previousSecretExpiresAt = agora + 24h`.
3. Grava a nova em `secret`.
4. Retorna a nova secret **apenas nessa resposta**.
5. O worker passa a assinar novas entregas com a nova secret; o cliente pode validar com a nova ou com a anterior durante 24h.
6. Após a expiração, a secret anterior pode ser limpa de forma oportunista (na próxima rotação ou no worker), sem afetar a nova ([09:21] Sofia).

### 5.6 Recuperação de eventos presos

**FDD-FLOW-12** — No início de cada ciclo, o worker executa:

```ts
await prisma.webhookOutbox.updateMany({
  where: {
    status: 'PROCESSING',
    processingStartedAt: { lt: new Date(Date.now() - 5 * 60_000) },
  },
  data: { status: 'PENDING', processingStartedAt: null },
});
```

Isso cobre o caso de o worker cair entre o claim e a conclusão. O limiar de 5 minutos é um parâmetro inicial de implementação, ajustável.

### 5.7 Exclusão de configuração

**FDD-FLOW-13** — `DELETE /api/v1/webhooks/:id` remove a configuração. Por cascade, os registros de outbox, deliveries e DLQ daquele endpoint também são removidos. Eventos já entregues deixam de compor o histórico do cliente. A decisão evita reter dados de pedidos de um cliente que pediu remoção; se o produto quiser preservar histórico após a exclusão, será necessário evoluir para soft delete em fase futura.

---

## 6. Contratos públicos

### 6.0 Convenções

**FDD-CONTRATO-01** — Todos os endpoints ficam sob `/api/v1`, prefixo já montado em `src/app.ts` (`app.use('/api/v1', buildApiRouter(controllers))`).

**FDD-CONTRATO-02** — Todas as rotas exigem `Authorization: Bearer <jwt>`. O CRUD pode ser usado por qualquer role autenticada (`ADMIN` ou `OPERATOR`), conforme decidido ([09:36]–[09:37] Sofia). O replay exige `ADMIN` ([09:36] Larissa).

**FDD-CONTRATO-03** — Erros seguem o formato do middleware existente:

```json
{
  "error": {
    "code": "WEBHOOK_NOT_FOUND",
    "message": "Webhook <id> not found",
    "details": { "id": "<id>" }
  }
}
```

**FDD-CONTRATO-04** — Listagens usam o helper `paginated()` de `src/shared/http/response.ts`:

```json
{
  "data": [],
  "pagination": { "page": 1, "pageSize": 20, "total": 0, "totalPages": 0 }
}
```

### 6.1 `POST /api/v1/webhooks` — criar configuração

**FDD-CONTRATO-05** — Cria um endpoint de webhook. A secret é gerada pela plataforma e devolvida **apenas nesta resposta** ([09:31] Marcos).

**Request**

```http
POST /api/v1/webhooks
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "customerId": "8a7b6c5d-4e3f-2a1b-9c8d-7e6f5a4b3c2d",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "eventStatuses": ["SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`**

```json
{
  "id": "3f1c0a8e-6d7b-4f3a-9a1e-2b7c4d5e6f70",
  "customerId": "8a7b6c5d-4e3f-2a1b-9c8d-7e6f5a4b3c2d",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "eventStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "9f2c4d6e8a0b1c3d5e7f9a2b4c6d8e0f1a3b5c7d9e2f4a6b8c0d1e3f5a7b9c1d",
  "createdAt": "2026-11-20T12:00:00.000Z",
  "updatedAt": "2026-11-20T12:00:00.000Z"
}
```

**Status codes**

| Código | Significado |
| --- | --- |
| `201` | Webhook criado |
| `400` | Payload inválido (`VALIDATION_ERROR` ou `WEBHOOK_INVALID_URL`) |
| `401` | JWT ausente/inválido (`UNAUTHORIZED`) |
| `404` | `customerId` inexistente (`NOT_FOUND`) |
| `500` | Erro inesperado |

**Regras**

- `customerId` é obrigatório no corpo; **não** vem do JWT, porque o JWT atual representa um usuário operador, não o cliente ([09:32]–[09:33] Marcos/Bruno/Larissa).
- `url` deve ser `https://`; `http://` retorna `WEBHOOK_INVALID_URL` ([09:23] Sofia).
- `eventStatuses` deve conter pelo menos um status válido do enum `OrderStatus`; a lista é o filtro de eventos daquele endpoint ([09:33] Marcos).
- `active` começa `true`.

### 6.2 `GET /api/v1/webhooks` — listar configurações de um cliente

**FDD-CONTRATO-06** — Lista os webhooks de um `customerId`, com paginação ([09:33] Bruno).

**Request**

```http
GET /api/v1/webhooks?customerId=8a7b6c5d-4e3f-2a1b-9c8d-7e6f5a4b3c2d&page=1&pageSize=20
Authorization: Bearer <jwt>
```

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "3f1c0a8e-6d7b-4f3a-9a1e-2b7c4d5e6f70",
      "customerId": "8a7b6c5d-4e3f-2a1b-9c8d-7e6f5a4b3c2d",
      "url": "https://hooks.atlascomercial.com.br/oms/orders",
      "eventStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-20T12:00:00.000Z",
      "updatedAt": "2026-11-20T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

**Status codes:** `200`, `400` (query inválida), `401`, `500`.

**Regra:** a secret **nunca** é retornada em listagens ([09:31] Marcos).

### 6.3 `PATCH /api/v1/webhooks/:id` — editar configuração

**FDD-CONTRATO-07** — Edita `url`, `eventStatuses` e `active` ([09:33] Bruno).

**Request**

```http
PATCH /api/v1/webhooks/3f1c0a8e-6d7b-4f3a-9a1e-2b7c4d5e6f70
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "eventStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false
}
```

**Response `200 OK`**

```json
{
  "id": "3f1c0a8e-6d7b-4f3a-9a1e-2b7c4d5e6f70",
  "customerId": "8a7b6c5d-4e3f-2a1b-9c8d-7e6f5a4b3c2d",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "eventStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false,
  "createdAt": "2026-11-20T12:00:00.000Z",
  "updatedAt": "2026-11-20T12:05:00.000Z"
}
```

**Status codes:** `200`, `400` (`VALIDATION_ERROR`/`WEBHOOK_INVALID_URL`), `401`, `404` (`WEBHOOK_NOT_FOUND`), `500`.

**Regras:** a secret não é editável por este endpoint; para trocar, usar a rotação. Campos ausentes permanecem inalterados.

### 6.4 `DELETE /api/v1/webhooks/:id` — remover configuração

**FDD-CONTRATO-08** — Remove o webhook e, por cascade, seus eventos e históricos ([09:33] Bruno).

**Request**

```http
DELETE /api/v1/webhooks/3f1c0a8e-6d7b-4f3a-9a1e-2b7c4d5e6f70
Authorization: Bearer <jwt>
```

**Response `204 No Content`** — sem corpo.

**Status codes:** `204`, `401`, `404` (`WEBHOOK_NOT_FOUND`), `500`.

### 6.5 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

**FDD-CONTRATO-09** — Gera nova secret e mantém a anterior válida por 24h ([09:21] Sofia).

**Request**

```http
POST /api/v1/webhooks/3f1c0a8e-6d7b-4f3a-9a1e-2b7c4d5e6f70/rotate-secret
Authorization: Bearer <jwt>
```

**Response `200 OK`**

```json
{
  "id": "3f1c0a8e-6d7b-4f3a-9a1e-2b7c4d5e6f70",
  "secret": "1a3b5c7d9e2f4a6b8c0d1e3f5a7b9c1d2e4f6a8b0c2d4e6f8a1b3c5d7e9f1a3b",
  "previousSecretExpiresAt": "2026-11-21T12:00:00.000Z"
}
```

**Status codes:** `200`, `401`, `404` (`WEBHOOK_NOT_FOUND`), `500`.

**Regra:** a nova secret só é retornada nesta resposta; não há endpoint para consultá-la novamente.

### 6.6 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

**FDD-CONTRATO-10** — Retorna as tentativas de entrega do endpoint, das mais recentes para as mais antigas. O padrão é `pageSize=100`, refletindo o pedido dos "últimos 100" ([09:34] Marcos).

**Request**

```http
GET /api/v1/webhooks/3f1c0a8e-6d7b-4f3a-9a1e-2b7c4d5e6f70/deliveries?page=1&pageSize=100
Authorization: Bearer <jwt>
```

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "8b9c0d1e-2f3a-4b5c-6d7e-8f9a0b1c2d3e",
      "eventId": "1e2d3c4b-5a6f-7e8d-9c0b-1a2f3e4d5c6b",
      "attempt": 1,
      "success": false,
      "requestPayload": {
        "event_id": "1e2d3c4b-5a6f-7e8d-9c0b-1a2f3e4d5c6b",
        "event_type": "order.status_changed",
        "timestamp": "2026-11-20T12:34:56.789Z",
        "order_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
        "order_number": "ORD-000123",
        "from_status": "PAID",
        "to_status": "SHIPPED",
        "customer_id": "8a7b6c5d-4e3f-2a1b-9c8d-7e6f5a4b3c2d",
        "total_cents": 45900
      },
      "responseStatus": 503,
      "responseBody": "{\"error\":\"service unavailable\"}",
      "durationMs": 412,
      "errorCode": null,
      "attemptedAt": "2026-11-20T12:34:57.201Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

**Status codes:** `200`, `400` (query inválida), `401`, `404` (`WEBHOOK_NOT_FOUND`), `500`.

**Regras:** `pageSize` máximo 100; a resposta inclui o payload da tentativa, o status HTTP do cliente, o corpo (truncado em 8 KB) e o tempo de resposta.

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay de DLQ

**FDD-CONTRATO-11** — Recoloca um evento da DLQ na outbox como pendente. Exige `ADMIN` e registra o autor ([09:18] Diego, [09:36] Sofia).

**Request**

```http
POST /api/v1/admin/webhooks/dead-letter/5c6d7e8f-9a0b-1c2d-3e4f-5a6b7c8d9e0f/replay
Authorization: Bearer <jwt-de-admin>
```

**Response `202 Accepted`**

```json
{
  "deadLetterId": "5c6d7e8f-9a0b-1c2d-3e4f-5a6b7c8d9e0f",
  "outboxEventId": "7d8e9f0a-1b2c-3d4e-5f6a-7b8c9d0e1f2a",
  "eventId": "1e2d3c4b-5a6f-7e8d-9c0b-1a2f3e4d5c6b",
  "webhookId": "3f1c0a8e-6d7b-4f3a-9a1e-2b7c4d5e6f70",
  "status": "PENDING",
  "replayedAt": "2026-11-20T13:00:00.000Z"
}
```

**Status codes:** `202`, `401`, `403` (`FORBIDDEN`), `404` (`WEBHOOK_DEAD_LETTER_NOT_FOUND`), `422` (`WEBHOOK_INACTIVE`), `500`.

### 6.8 Contrato de saída — entrega ao cliente

**FDD-CONTRATO-12** — O worker envia `POST` para a URL cadastrada, com os headers abaixo ([09:44] Diego/Sofia):

| Header | Valor |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (mesmo `event_id` do payload) |
| `X-Webhook-Id` | UUID do endpoint de webhook |
| `X-Timestamp` | Timestamp ISO 8601 do envio, em UTC |
| `X-Signature` | HMAC-SHA256 do corpo exato, em hexadecimal minúsculo (a confirmar com Sofia) |

**Corpo**

```json
{
  "event_id": "1e2d3c4b-5a6f-7e8d-9c0b-1a2f3e4d5c6b",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-20T12:34:56.789Z",
  "order_id": "0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "SHIPPED",
  "customer_id": "8a7b6c5d-4e3f-2a1b-9c8d-7e6f5a4b3c2d",
  "total_cents": 45900
}
```

**Semântica**

- `event_id`: UUID gerado na inserção da outbox; chave de deduplicação.
- `event_type`: sempre `order.status_changed` nesta fase.
- `timestamp`: momento da mudança de status (não do envio).
- `from_status` / `to_status`: transição que gerou o evento.
- `order_number`, `customer_id`, `total_cents`: dados básicos do pedido. **Itens não são enviados** ([09:43] Diego).
- Sucesso: qualquer status HTTP `2xx` dentro de 10 segundos.
- Falha: timeout, erro de rede, `3xx` ou qualquer `4xx`/`5xx`.
- A assinatura é calculada sobre a string exata do corpo; o worker não deve re-serializar o objeto.

**Cálculo da assinatura**

```ts
// src/modules/webhooks/webhook.signature.ts (a criar)
import { createHmac } from 'node:crypto';

export function signPayload(secret: string, body: string): string {
  return createHmac('sha256', secret).update(body, 'utf8').digest('hex');
}
```

---

## 7. Matriz de erros previstos

### 7.1 Erros de domínio (`WEBHOOK_*`)

| ID | Código | HTTP | Quando ocorre | Onde é gerado | Efeito |
| --- | --- | --- | --- | --- | --- |
| **FDD-ERR-01** | `WEBHOOK_NOT_FOUND` | `404` | `id` de webhook inexistente em GET/PATCH/DELETE/rotate/deliveries | `webhook.service.ts` | Resposta de erro padrão; nenhuma alteração |
| **FDD-ERR-02** | `WEBHOOK_INVALID_URL` | `400` | URL malformada ou com esquema diferente de `https:` | `webhook.service.ts` | Cadastro/edição rejeitado |
| **FDD-ERR-03** | `WEBHOOK_SECRET_REQUIRED` | `400` | Invariante de criação/rotação: secret gerada vazia ou ausente antes de persistir | `webhook.service.ts` | Operação abortada; nada é gravado |
| **FDD-ERR-04** | `WEBHOOK_INVALID_EVENT_STATUS` | `400` | `eventStatuses` vazio, com status desconhecido ou duplicado | `webhook.schemas.ts` / `webhook.service.ts` | Cadastro/edição rejeitado |
| **FDD-ERR-05** | `WEBHOOK_PAYLOAD_TOO_LARGE` | `422` | Snapshot do evento ultrapassa 64 KB na inserção | `webhook.publisher.ts` | Rollback da transação de mudança de status |
| **FDD-ERR-06** | `WEBHOOK_INACTIVE` | `422` | Replay de DLQ para endpoint inativo | `webhook.service.ts` | Replay recusado |
| **FDD-ERR-07** | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | `404` | `id` de DLQ inexistente ou outbox associada ausente | `webhook.service.ts` | Resposta de erro padrão |
| **FDD-ERR-08** | `WEBHOOK_DELIVERY_TIMEOUT` | interno | Chamada HTTP excede 10 segundos | `webhook.processor.ts` | Registra falha, agenda retry ou DLQ |
| **FDD-ERR-09** | `WEBHOOK_DELIVERY_FAILED` | interno | Resposta não-2xx, erro de rede ou redirecionamento | `webhook.processor.ts` | Registra falha, agenda retry ou DLQ |

**FDD-ERR-10** — Erros de infraestrutura reutilizados sem alteração no middleware: `VALIDATION_ERROR` (`400`, Zod), `UNAUTHORIZED` (`401`, `authenticate`), `FORBIDDEN` (`403`, `requireRole`), `NOT_FOUND` (`404`, recurso como `Customer`), `CONFLICT` (`409`, Prisma `P2002`) e `INTERNAL_SERVER_ERROR` (`500`). O formato é sempre o de `src/middlewares/error.middleware.ts`.

### 7.2 Classes de erro propostas

```ts
// src/modules/webhooks/webhook.errors.ts (a criar)
import { AppError } from '../../shared/errors/index.js';

export class WebhookNotFoundError extends AppError {
  constructor(id: string) {
    super(`Webhook ${id} not found`, 404, 'WEBHOOK_NOT_FOUND', { id });
  }
}

export class WebhookInvalidUrlError extends AppError {
  constructor(url: string) {
    super('Webhook URL must use https', 400, 'WEBHOOK_INVALID_URL', { url });
  }
}

export class WebhookSecretRequiredError extends AppError {
  constructor() {
    super('Webhook secret is required', 400, 'WEBHOOK_SECRET_REQUIRED');
  }
}

export class WebhookInvalidEventStatusError extends AppError {
  constructor(statuses: string[]) {
    super('Invalid webhook event status filter', 400, 'WEBHOOK_INVALID_EVENT_STATUS', {
      statuses,
    });
  }
}

export class WebhookPayloadTooLargeError extends AppError {
  constructor(size: number) {
    super('Webhook payload exceeds 64KB', 422, 'WEBHOOK_PAYLOAD_TOO_LARGE', {
      size,
      limit: 65_536,
    });
  }
}

export class WebhookInactiveError extends AppError {
  constructor(webhookId: string) {
    super('Webhook endpoint is inactive', 422, 'WEBHOOK_INACTIVE', { webhookId });
  }
}

export class WebhookDeadLetterNotFoundError extends AppError {
  constructor(id: string) {
    super(`Webhook dead letter ${id} not found`, 404, 'WEBHOOK_DEAD_LETTER_NOT_FOUND', { id });
  }
}
```

> `WEBHOOK_SECRET_REQUIRED` é uma invariante de segurança: o serviço nunca deve persistir um endpoint sem secret. A secret é gerada pela plataforma; o cliente não a envia.

---

## 8. Estratégias de resiliência

| ID | Estratégia | Implementação |
| --- | --- | --- |
| **FDD-RES-01** | **Timeout de 10 segundos** | `AbortSignal.timeout(10_000)` ou `AbortController` com timer; timeout conta como falha ([09:42] Diego) |
| **FDD-RES-02** | **Retry com backoff** | 5 retries com 1 m/5 m/30 m/2 h/12 h; `attempts` e `nextAttemptAt` na outbox ([09:17] Larissa) |
| **FDD-RES-03** | **DLQ** | Tabela separada com payload, motivo, timestamp; replay manual `ADMIN` ([09:18] Diego) |
| **FDD-RES-04** | **At-least-once + dedup** | `X-Event-Id` e `event_id`; cliente deduplica ([09:25] Diego) |
| **FDD-RES-05** | **Ordering por `order_id`** | Single worker, processamento por `createdAt` ascendente; sem garantia global ([09:12] Diego) |
| **FDD-RES-06** | **Recuperação de `PROCESSING` preso** | Reenfileirar eventos com `processingStartedAt` > 5 min no início do ciclo |
| **FDD-RES-07** | **Sem fallback por e-mail nesta fase** | Falha permanente fica na DLQ para replay; e-mail é próxima fase ([09:37] Larissa) |
| **FDD-RES-08** | **Sem rate limiting nesta fase** | Observar e decidir depois; risco de rajadas assumido ([09:39] Larissa) |

---

## 9. Observabilidade

### 9.1 Logs

**FDD-OBS-01** — Reutilizar o Pino de `src/shared/logger/index.ts`; não criar logger novo ([09:29] Bruno). Eventos estruturados recomendados:

| Evento | Campos principais | Quando emitir |
| --- | --- | --- |
| `webhook_event_enqueued` | `eventId`, `orderId`, `webhooks`, `toStatus` | Após o commit da transação de status |
| `webhook_delivery_attempt` | `eventId`, `webhookId`, `attempt`, `statusCode`, `durationMs`, `success` | A cada tentativa HTTP |
| `webhook_retry_scheduled` | `eventId`, `webhookId`, `attempt`, `nextAttemptAt` | Após falha com retry |
| `webhook_dead_letter_created` | `eventId`, `webhookId`, `reason` | Ao esgotar retries |
| `webhook_dead_letter_replayed` | `deadLetterId`, `outboxEventId`, `adminUserId` | No replay |
| `webhook_worker_cycle_failed` | `err`, `durationMs` | Exceção no ciclo |
| `webhook_worker_started` / `webhook_worker_stopped` | `env`, `pid` | Ciclo de vida do worker |
| `webhook_payload_rejected` | `orderId`, `size`, `limit` | Payload > 64 KB |

**FDD-OBS-02** — Nunca logar `secret`, `previousSecret`, `X-Signature` ou o payload completo com dados de pedido. Estender a lista de `redact` de `src/shared/logger/index.ts` com caminhos como `*.secret`, `*.previousSecret`, `req.headers['x-signature']` e `signature` ([09:22] Sofia).

### 9.2 Métricas

**FDD-OBS-03** — A stack atual não possui biblioteca de métricas. As métricas abaixo são emitidas como **logs estruturados** e podem ser derivadas por um coletor externo. Se o time adotar Prometheus/OpenTelemetry, usar os mesmos nomes:

| Métrica (evento) | Tipo | Labels | Uso |
| --- | --- | --- | --- |
| `webhook_event_enqueued` | counter | `toStatus` | Volume de eventos gerados |
| `webhook_delivery_attempt` | counter + histogram | `success`, `statusCode`, `attempt` | Taxa de sucesso e latência |
| `webhook_retry_scheduled` | counter | `attempt` | Volume de retries |
| `webhook_dead_letter_created` | counter | `reason` | Falhas permanentes |
| `webhook_dead_letter_replayed` | counter | `adminUserId` | Operação de replay |
| `webhook_outbox_backlog` | gauge | `status` | Atraso e acúmulo |
| `webhook_worker_cycle` | histogram | `processed` | Saúde do worker |
| `webhook_payload_rejected` | counter | `reason` | Payloads recusados |

### 9.3 Tracing e correlação

**FDD-OBS-04** — A API já propaga `X-Request-Id` via `src/middlewares/request-logger.middleware.ts`. No worker não há request HTTP de entrada; a correlação é feita por:

- `eventId` — chave principal do evento, presente no payload, no header `X-Event-Id`, na outbox, nas deliveries e na DLQ.
- `webhookId` — identifica o endpoint e o cadastro.
- `orderId` — correlaciona com os logs de pedidos.
- `requestId` — presente nos logs de replay e das chamadas administrativas.

**FDD-OBS-05** — Não há tracing distribuído na stack atual. Se OpenTelemetry for adotado no futuro, propagar `traceparent` no request de saída; nesta fase, `eventId` é o identificador de rastreio ponta a ponta.

### 9.4 Auditoria do replay

**FDD-OBS-06** — O replay grava `replayedById` na DLQ e emite `webhook_dead_letter_replayed` com `adminUserId`. Isso atende ao requisito de auditoria de quem executou o replay ([09:36] Sofia).

---

## 10. Dependências e compatibilidade

| ID | Dependência | Detalhe |
| --- | --- | --- |
| **FDD-DEP-01** | Node.js >= 20 | Já declarado em `package.json` (`engines.node`); `fetch` global e `AbortSignal.timeout` disponíveis |
| **FDD-DEP-02** | MySQL 8 + Prisma 5.22 | Já usados pela aplicação; nova migração adiciona as tabelas de webhook |
| **FDD-DEP-03** | Dependências existentes | Nenhuma nova biblioteca: `crypto` é built-in; `uuid`, `zod`, `pino`, `express`, `jsonwebtoken` já estão no projeto |
| **FDD-DEP-04** | Worker em processo separado | Requer deploy/supervisão do novo `src/worker.ts`; script `npm run worker` ([09:11] Larissa) |
| **FDD-DEP-05** | Revisão de segurança | Pelo menos 2 dias úteis da Sofia antes do deploy ([09:46] Sofia) |
| **FDD-DEP-06** | Compatibilidade de API | Nenhuma rota existente muda de contrato; `changeStatus` mantém resposta e status codes; apenas passa a inserir na outbox |

**FDD-DEP-07** — O `changeStatus` retorna o mesmo `OrderWithRelations` de hoje. A única mudança observável é o efeito colateral na outbox e a possibilidade de erro `WEBHOOK_PAYLOAD_TOO_LARGE` em casos extremos.

**FDD-DEP-08** — O worker usa o mesmo `DATABASE_URL` e a mesma stack, mas **não** compartilha a instância de `PrismaClient` com a API, porque cada processo tem a sua ([09:30] Bruno).

---

## 11. Integração com o sistema existente

### 11.1 Mapa de integração

| ID | Caminho existente | Como o módulo de webhooks se integra |
| --- | --- | --- |
| **FDD-INT-01** | `src/modules/orders/order.service.ts` | `changeStatus` chama `publishWebhookEvent(tx, order, from, to)` dentro da transação existente; rollback em falha; resposta inalterada |
| **FDD-INT-02** | `src/modules/orders/order.status.ts` | Usa `OrderStatus`, `canTransition` e as transições existentes como fonte dos status elegíveis; não altera a máquina de estados |
| **FDD-INT-03** | `src/shared/errors/http-errors.ts` e `src/shared/errors/app-error.ts` | Novos erros `WEBHOOK_*` estendem `AppError`/subclasses, seguindo o padrão de `InsufficientStockError` e `InvalidStatusTransitionError` |
| **FDD-INT-04** | `src/middlewares/error.middleware.ts` | Nenhuma alteração: o middleware já trata `AppError` e serializa `{ error: { code, message, details } }` |
| **FDD-INT-05** | `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas; `requireRole('ADMIN')` no replay, reutilizando a união `AuthUser['role']` |
| **FDD-INT-06** | `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService` e `WebhookController`; `AppDependencies` continua recebendo apenas `prisma` |
| **FDD-INT-07** | `src/routes/index.ts` | Monta `/webhooks` e `/admin/webhooks` no `buildApiRouter`; `Controllers` ganha `webhooks` |
| **FDD-INT-08** | `src/config/database.ts` | Worker usa `createPrismaClient()` para obter sua própria instância |
| **FDD-INT-09** | `src/server.ts` | Padrão de bootstrap, logs e shutdown (`SIGINT`/`SIGTERM`) replicado em `src/worker.ts` |
| **FDD-INT-10** | `src/shared/logger/index.ts` | Reuso do Pino; extensão da lista de `redact` para secret/assinatura |
| **FDD-INT-11** | `prisma/schema.prisma` | Novos models `WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery`, `WebhookDeadLetter` e enum `WebhookOutboxStatus`; relação `Customer.webhooks` |
| **FDD-INT-12** | `package.json` | Novos scripts `worker` e `worker:start`; nenhuma dependência nova |
| **FDD-INT-13** | `tests/orders.test.ts` e `tests/helpers/factories.ts` | Reuso do padrão Supertest/Vitest e das factories para testes de integração do CRUD e da transação |
| **FDD-INT-14** | `tests/setup.ts` | Incluir a limpeza das novas tabelas antes de `customer`/`user` para manter o isolamento dos testes |

### 11.2 `order.service.ts` — extensão do `changeStatus`

O método atual já faz tudo dentro de `this.prisma.$transaction(async (tx) => { ... })`. A extensão é inserir a chamada ao publisher **antes do retorno**, usando o mesmo `tx`. Não se deve usar `this.prisma` dentro do publisher, pois isso sairia da transação e quebraria a atomicidade ([09:40]–[09:41] Bruno/Diego).

```ts
// src/modules/orders/order.service.ts
import { publishWebhookEvent } from '../webhooks/webhook.publisher.js';

// dentro de changeStatus, após order.update e orderStatusHistory.create:
const published = await publishWebhookEvent(tx, order, from, to);
```

O `order` carregado no início da transação (`tx.order.findUnique({ where: { id }, include: { items: true } })`) já contém `id`, `orderNumber`, `customerId`, `totalCents` e os demais campos usados no payload. Não é necessário reler o pedido para montar o snapshot.

### 11.3 `order.status.ts` — status e transições

O módulo de webhooks usa o enum `OrderStatus` do Prisma e as transições de `order.status.ts` como referência. O filtro `eventStatuses` aceita apenas valores do enum; a mudança de status continua validada por `canTransition` antes de qualquer inserção na outbox. Nenhuma transição nova é criada.

### 11.4 Erros existentes — reuso do padrão

`src/shared/errors/app-error.ts` define `AppError` com `statusCode`, `errorCode` e `details`. `src/shared/errors/http-errors.ts` já traz `BadRequestError`, `ValidationError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `InvalidStatusTransitionError` e `InsufficientStockError`.

Os erros de webhook devem seguir o mesmo formato. Exemplo de integração com o padrão existente:

```ts
// src/modules/webhooks/webhook.errors.ts (a criar)
import { AppError } from '../../shared/errors/index.js';

export class WebhookNotFoundError extends AppError {
  constructor(id: string) {
    super(`Webhook ${id} not found`, 404, 'WEBHOOK_NOT_FOUND', { id });
  }
}
```

O `error.middleware.ts` não precisa saber dessas classes: como todas estendem `AppError`, o `instanceof AppError` já as captura e serializa o `errorCode` ([09:29] Bruno).

### 11.5 `auth.middleware.ts` — autenticação e autorização

- `authenticate` já valida o JWT e popula `req.user` com `{ id, email, role }`.
- O CRUD de webhooks usa `router.use(authenticate)`.
- O replay usa `router.use(authenticate, requireRole('ADMIN'))`.
- O `requireRole` já lança `ForbiddenError('Insufficient permissions')` (código `FORBIDDEN`, HTTP 403), que é o comportamento esperado para o replay ([09:36] Larissa/Sofia).

### 11.6 `app.ts` e `routes/index.ts` — registro do módulo

Em `src/app.ts`, `buildControllers` deve instanciar o módulo e devolvê-lo em `Controllers`:

```ts
// src/app.ts (trecho proposto)
const webhookRepository = new WebhookRepository(prisma);
const webhookService = new WebhookService(webhookRepository, prisma);
const webhookController = new WebhookController(webhookService);

return {
  auth: authController,
  users: userController,
  customers: customerController,
  products: productController,
  orders: orderController,
  webhooks: webhookController,
};
```

Em `src/routes/index.ts`:

```ts
// src/routes/index.ts (trecho proposto)
router.use('/webhooks', buildWebhookRouter(controllers.webhooks));
router.use('/admin/webhooks', buildWebhookAdminRouter(controllers.webhooks));
```

### 11.7 `config/database.ts` e `server.ts` — worker

O worker deve usar `createPrismaClient()` de `src/config/database.ts` para criar sua própria instância e replicar o padrão de shutdown de `src/server.ts`:

```ts
// src/worker.ts (trecho proposto)
import { createPrismaClient } from './config/database.js';
import { logger } from './shared/logger/index.js';

const prisma = createPrismaClient();

process.on('SIGINT', () => void shutdown('SIGINT'));
process.on('SIGTERM', () => void shutdown('SIGTERM'));

async function shutdown(signal: string): Promise<void> {
  logger.info({ signal }, 'webhook_worker_shutdown_initiated');
  await prisma.$disconnect();
  process.exit(0);
}
```

### 11.8 `prisma/schema.prisma` — migração

Adicionar o enum e os quatro models da seção 4.2, além da relação `webhooks WebhookEndpoint[]` em `Customer`. Gerar a migração com:

```bash
npx prisma migrate dev --name add_webhook_tables
```

A migração não altera colunas existentes; apenas cria tabelas, índices e FKs. O nome das tabelas segue o padrão snake_case com `@@map`.

### 11.9 `package.json` — scripts do worker

```json
{
  "scripts": {
    "worker": "tsx watch --env-file=.env src/worker.ts",
    "worker:start": "node --env-file=.env dist/worker.js"
  }
}
```

`tsconfig.build.json` já inclui `src/**/*.ts`, então `dist/worker.js` será gerado pelo `npm run build`.

### 11.10 `logger/index.ts` — redaction

A lista atual já cobre `authorization`, `password`, `passwordHash`, `token` e `accessToken`. Acrescentar caminhos para secret e assinatura:

```ts
const redactPaths = [
  'req.headers.authorization',
  'req.headers.cookie',
  'req.headers["x-signature"]',
  '*.password',
  '*.passwordHash',
  '*.token',
  '*.accessToken',
  '*.secret',
  '*.previousSecret',
  'signature',
];
```

### 11.11 `tests/` — padrão de testes e limpeza

- `tests/orders.test.ts` usa `getTestApp()`, `bootstrapAuthenticatedUser()`, `createTestCustomer()` e `createTestProduct()`; o mesmo padrão serve para os testes de webhooks.
- `tests/helpers/factories.ts` concentra factories e login; adicionar factories de webhook se necessário.
- `tests/setup.ts` limpa tabelas em ordem; incluir `webhookDelivery`, `webhookDeadLetter`, `webhookOutbox` e `webhookEndpoint` **antes** de `customer` e `user` para respeitar as FKs.

---

## 12. Critérios de aceite técnicos

| ID | Critério | Como verificar |
| --- | --- | --- |
| **FDD-ACC-01** | Migração cria as quatro tabelas e índices sem alterar colunas existentes | `prisma migrate dev` + inspeção do SQL gerado |
| **FDD-ACC-02** | `changeStatus` insere uma linha de outbox por endpoint ativo interessado no status | Teste de integração com 2 endpoints e status distintos |
| **FDD-ACC-03** | Falha na inserção da outbox causa rollback da mudança de status | Teste com payload > 64 KB ou constraint simulada; status e estoque inalterados |
| **FDD-ACC-04** | Nenhum evento é inserido quando nenhum endpoint quer o status | Teste com endpoints filtrando outros status |
| **FDD-ACC-05** | Worker processa pendentes em ordem de `createdAt`, single-worker, a cada 2 s | Teste com fake timers e ordem de eventos |
| **FDD-ACC-06** | Retry segue 1 m/5 m/30 m/2 h/12 h e DLQ após o quinto retry | Teste unitário da função de agendamento |
| **FDD-ACC-07** | DLQ é criada com payload e motivo; replay exige `ADMIN` e audita `replayedById` | Teste de integração com tokens `ADMIN`/`OPERATOR` |
| **FDD-ACC-08** | HMAC-SHA256 confere com a secret atual; rotação mantém a anterior por 24 h | Teste de assinatura e rotação com relógio controlado |
| **FDD-ACC-09** | URL `http://` é rejeitada; payload > 64 KB é rejeitado | Testes de contrato e de publisher |
| **FDD-ACC-10** | Histórico retorna últimas 100 tentativas com payload, resposta e tempo | Teste de paginação e ordenação descendente |
| **FDD-ACC-11** | Logs incluem `eventId`, `webhookId`, `attempt`, `statusCode`, `durationMs`; segredos não aparecem | Inspeção de logs em teste e revisão de redaction |
| **FDD-ACC-12** | Testes existentes de pedidos continuam passando | `npm test` |

---

## 13. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| **FDD-RISK-01** | Inserção extra na transação de status aumentar contenção/latência | Média | Médio | Inserção simples em tabela indexada; medir latência do `changeStatus` antes/depois; rollback consistente |
| **FDD-RISK-02** | Crescimento da outbox/deliveries e degradação do worker | Média | Alto | Índices, lote pequeno, monitoramento de backlog; arquivamento após 30 dias em fase futura ([09:08] Diego) |
| **FDD-RISK-03** | Worker cair com eventos em `PROCESSING` | Média | Alto | Reenfileirar `PROCESSING` com mais de 5 min; logs e alerta de backlog |
| **FDD-RISK-04** | Vazamento de secret/assinatura em log ou resposta | Média | Alto | Redaction no Pino, secret só na criação/rotação, revisão de segurança ([09:22] Sofia) |
| **FDD-RISK-05** | Duplicidade de entregas confundir o cliente | Média | Médio | `X-Event-Id` e `event_id`; documentação no portal ([09:26] Marcos) |
| **FDD-RISK-06** | Rajadas de eventos sem rate limiting | Média | Médio | Observar volume real; rate limiting fica para fase futura ([09:39] Larissa) |
| **FDD-RISK-07** | Payload acima de 64 KB bloquear uma mudança de status | Baixa | Médio | Payload enxuto sem itens; limite explícito e erro `WEBHOOK_PAYLOAD_TOO_LARGE` ([09:24] Larissa) |

---

## 14. Plano de entrega e validação

| ID | Etapa | Conteúdo |
| --- | --- | --- |
| **FDD-PLAN-01** | Sprint 1 | Modelagem/migração, módulo e CRUD, publisher na transação, testes unitários e de integração básicos |
| **FDD-PLAN-02** | Sprint 2 | Worker, polling, retry/backoff, DLQ, replay, HMAC e rotação de secret |
| **FDD-PLAN-03** | Sprint 3 | Histórico de deliveries, testes ponta a ponta, observabilidade, hardening e revisão de segurança da Sofia (2 dias úteis) |

A estimativa total é de **3 sprints**, incluindo a revisão de segurança ([09:46] Larissa, [09:46] Sofia).

---

## 15. Referências

- `TRANSCRICAO.md` — reunião técnica completa.
- [PRD](PRD.md) — problema, público, escopo e métricas.
- [RFC](RFC.md) — proposta técnica e alternativas.
- [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md) — outbox no MySQL.
- [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) — worker separado e polling.
- [ADR-003](adrs/ADR-003-politica-de-retry-e-dlq.md) — retry e DLQ.
- [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md) — HMAC e secret por endpoint.
- [ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md) — at-least-once.
- [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) — reuso dos padrões.
- [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) — snapshot do payload.
- `docs/TRACKER.md` — rastreabilidade.
