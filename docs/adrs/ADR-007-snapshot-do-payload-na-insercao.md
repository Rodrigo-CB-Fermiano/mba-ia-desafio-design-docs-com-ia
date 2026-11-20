# ADR-007 — Snapshot do payload na inserção da outbox

## Status

Aceito

- **Data:** decisão registrada na reunião técnica de quinta-feira (ver `TRANSCRICAO.md`); ADR consolidado em 20/11/2026
- **Decisores:** Larissa (Tech Lead), Diego (Engenharia Sênior), Bruno (Engenharia Pleno)
- **Contexto da decisão:** reunião técnica do Sistema de Webhooks de Notificação de Pedidos

## Contexto

Ao inserir um evento na outbox, havia duas opções: guardar o payload já renderizado (snapshot) ou guardar apenas referências como `order_id` e renderizar o payload na hora do envio.

Se o payload fosse renderizado no envio, o evento refletiria o estado **atual** do pedido, e não o estado no momento da mudança de status. Se o pedido mudasse novamente antes da entrega, o cliente receberia um evento de `PAID` com dados de um pedido já `SHIPPED`, por exemplo. A reunião considerou esse comportamento esquisito e preferiu o snapshot ([09:51]–[09:52] Bruno/Larissa/Diego).

## Decisão

1. O payload é **renderizado no momento da inserção** na outbox e gravado como snapshot serializado.
2. O worker **não relê o pedido** para montar o payload; ele envia exatamente o snapshot gravado.
3. O `timestamp` do payload representa o momento da mudança de status, não o momento do envio.
4. O snapshot inclui apenas os campos definidos para o evento `order.status_changed`: `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`; **não inclui os itens do pedido** ([09:43] Diego).
5. O snapshot é serializado uma única vez e armazenado como texto; é essa string que o worker envia e assina com HMAC, garantindo que a assinatura corresponda exatamente aos bytes entregues.
6. O limite de 64 KB é verificado no momento da inserção; se o snapshot ultrapassar, a transação de mudança de status sofre rollback ([09:24] Larissa).

## Alternativas Consideradas

### Alternativa A — Guardar apenas `order_id` e renderizar no envio

- **Trade-off que levou ao descarte:** o evento refletiria o estado atual do pedido, não o estado no momento da transição; poderia gerar eventos incoerentes e dificultar auditoria ([09:52] Larissa).
- **Custo evitado:** menos armazenamento, ao preço de consistência histórica.

### Alternativa B — Guardar apenas o diff/transição e deixar o cliente buscar o pedido

- **Trade-off:** o cliente teria que fazer uma chamada extra para `GET /orders/:id` e poderia ver um estado diferente do evento; além disso, aumentaria a carga na API.
- **Custo evitado:** payload menor, ao preço de mais latência e acoplamento.

### Alternativa C — Snapshot no envio, mas com versão do pedido

- **Trade-off:** exigiria versionamento ou histórico consultável no momento do envio; complexidade maior do que gravar o snapshot no evento.
- **Custo evitado:** menos duplicação de dados, ao preço de consultas e controle de versão.

## Consequências

### Positivas

- **Consistência histórica:** o evento representa exatamente o estado do pedido quando o status mudou.
- **Determinismo:** o worker não depende de o pedido ainda existir ou estar no mesmo estado no momento do envio.
- **Assinatura confiável:** como o snapshot textual é o mesmo enviado, o HMAC corresponde aos bytes reais.
- **Auditoria:** o payload original fica preservado na outbox, nas deliveries e na DLQ.
- **Menos consultas no worker:** o processamento não precisa reler `orders`, `order_items` ou `customers`.

### Negativas / trade-offs

- **Duplicação de dados:** o mesmo conteúdo pode existir na outbox, nas deliveries e na DLQ.
- **Evolução de schema:** eventos antigos mantêm o formato antigo; se o payload evoluir, o cliente pode receber formatos diferentes. Nesta fase não há versionamento explícito de payload.
- **Dados sensíveis persistidos:** o snapshot contém dados do pedido e do cliente; a retenção e a proteção precisam ser tratadas (o arquivamento após 30 dias ficou fora desta fase).
- **Limite de 64 KB:** se o snapshot ultrapassar, a mudança de status falha por rollback; é um efeito colateral deliberado para garantir consistência.

## Referências

- `TRANSCRICAO.md` — [09:43] Diego; [09:24] Larissa; [09:51] Diego/Larissa; [09:52] Bruno/Larissa/Diego
- `src/modules/orders/order.service.ts` — dados disponíveis no momento da transição
- `prisma/schema.prisma` — modelo `Order` e campos usados no payload
- `src/modules/webhooks/` — módulo proposto no FDD (a criar)
