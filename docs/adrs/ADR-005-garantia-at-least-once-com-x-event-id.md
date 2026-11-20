# ADR-005 — Garantia at-least-once com X-Event-Id

## Status

Aceito

- **Data:** decisão registrada na reunião técnica de quinta-feira (ver `TRANSCRACAO.md`); ADR consolidado em 20/11/2026
- **Decisores:** Larissa (Tech Lead), Diego (Engenharia Sênior), Sofia (Engenharia de Segurança)
- **Contexto da decisão:** reunião técnica do Sistema de Webhooks de Notificação de Pedidos

## Contexto

O worker pode falhar depois de enviar a requisição e antes de marcar o evento como entregue (por exemplo, o cliente recebeu, mas a resposta se perdeu ou o processo caiu). Nesses casos, a retentativa pode entregar o mesmo evento duas vezes.

A reunião discutiu exactly-once versus at-least-once. Exactly-once exigiria coordenação dos dois lados e seria muito mais complexo; at-least-once com um identificador de evento resolve a grande maioria dos casos ([09:25] Diego). O padrão de mercado citado foi o de Stripe e GitHub ([09:25] Diego).

A decisão foi **at-least-once**, com um `event_id` enviado no header `X-Event-Id`, gerado quando o evento entra na outbox, para o cliente deduplicar do lado dele ([09:24]–[09:26] Diego/Larissa). O PM se comprometeu a documentar isso de forma destacada no portal de desenvolvedor ([09:26] Marcos).

## Decisão

1. A plataforma garante **at-least-once**: todo evento elegível será tentado pelo menos uma vez, podendo ser entregue mais de uma vez.
2. Cada evento recebe um `eventId` UUID **no momento da inserção na outbox**.
3. O `eventId` é enviado:
   - no header `X-Event-Id` ([09:25] Diego);
   - no campo `event_id` do payload ([09:43] Diego).
4. O cliente é responsável por **deduplicar** pelo `eventId` do lado dele. A plataforma não mantém uma janela de deduplicação por cliente.
5. O mesmo `eventId` é usado para todas as linhas de outbox geradas por uma mesma mudança de status (uma por endpoint interessado). A combinação `(webhookId, eventId)` é única na outbox; o cliente que tem vários endpoints pode usar `X-Webhook-Id` em conjunto com `X-Event-Id` se precisar de uma chave ainda mais específica.
6. **Exactly-once não será implementado nesta fase.**
7. A documentação para o cliente deve deixar explícito que duplicatas podem acontecer e como deduplicar ([09:26] Marcos).

## Alternativas Consideradas

### Alternativa A — Exactly-once

- **Trade-off que levou ao descarte:** exigiria coordenação dos dois lados, confirmação transacional e tratamento de falhas parciais; complexidade desproporcional para o benefício ([09:25] Diego).
- **Custo evitado:** protocolo de duas fases ou idempotência distribuída.

### Alternativa B — At-most-once

- **Trade-off que levou ao descarte:** poderia perder eventos em falhas transitórias, contrariando a necessidade de o cliente saber que o pedido mudou; a plataforma prefere entregar duas vezes a não entregar.
- **Custo evitado:** simplicidade, ao preço de perda de eventos.

### Alternativa C — Deduplicação server-side por cliente

- **Trade-off que levou ao descarte:** exigiria armazenar estado de deduplicação por cliente, definir janela e lidar com múltiplos endpoints; a reunião optou por transferir a responsabilidade ao cliente, como fazem Stripe e GitHub ([09:25] Diego).
- **Custo evitado:** estado e complexidade adicionais na plataforma.

## Consequências

### Positivas

- **Simplicidade e robustez:** não é necessário protocolo distribuído; a plataforma pode retentar sem medo de duplicar.
- **Padrão de mercado:** clientes já conhecem o padrão de deduplicação por `event_id`.
- **Rastreabilidade:** o mesmo `eventId` aparece no payload, no header, na outbox e no histórico de entregas.
- **Compatibilidade com retry:** a política de retry do ADR-003 pode ser aplicada sem coordenação adicional.

### Negativas / trade-offs

- **Duplicatas possíveis:** o cliente precisa implementar deduplicação; sem isso, pode processar o mesmo evento duas vezes.
- **Responsabilidade transferida:** a plataforma não garante exatamente uma entrega; a qualidade da integração depende do cliente.
- **Documentação obrigatória:** o portal precisa explicar o comportamento e a chave de deduplicação ([09:26] Marcos).
- **Chave por endpoint:** como a outbox é por endpoint, o cliente com vários cadastros deve considerar `(X-Webhook-Id, X-Event-Id)` se quiser deduplicar entre endpoints.

## Referências

- `TRANSCRACAO.md` — [09:24] Diego; [09:25] Bruno/Diego/Sofia; [09:26] Marcos/Larissa; [09:43] Diego; [09:44] Diego; [09:48] Larissa
- `prisma/schema.prisma` — modelo de outbox proposto no FDD
- `src/modules/orders/order.service.ts` — ponto de criação do evento
- `src/modules/webhooks/` — módulo proposto no FDD (a criar)
