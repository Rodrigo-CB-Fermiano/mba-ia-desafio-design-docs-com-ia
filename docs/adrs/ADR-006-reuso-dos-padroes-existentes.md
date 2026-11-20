# ADR-006 — Reuso dos padrões existentes do projeto

## Status

Aceito

- **Data:** decisão registrada na reunião técnica de quinta-feira (ver `TRANSCRICAO.md`); ADR consolidado em 20/11/2026
- **Decisores:** Larissa (Tech Lead), Bruno (Engenharia Pleno), Diego (Engenharia Sênior)
- **Contexto da decisão:** reunião técnica do Sistema de Webhooks de Notificação de Pedidos

## Contexto

O repositório já tem convenções consolidadas: cada domínio é um módulo em `src/modules` com `controller`, `service`, `repository`, `routes` e `schemas`; os erros de domínio estendem `AppError` e usam códigos como `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`; o logger é Pino; o middleware de erro centralizado trata `AppError`, `ZodError` e erros conhecidos do Prisma; a autenticação é JWT com `requireRole`; e o `PrismaClient` é criado por processo.

O time é pequeno e a feature precisa entrar sem introduzir um novo framework, um novo logger, um novo formato de erro ou uma nova camada de validação. A decisão foi **reuso máximo** do que já existe ([09:30] Larissa).

## Decisão

O módulo de webhooks seguirá os mesmos padrões do restante da aplicação:

1. **Estrutura modular:** criar `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts` e `webhook.schemas.ts` ([09:27] Bruno).
2. **Erros:** criar classes específicas que estendem `AppError` (ou subclasses existentes), com códigos prefixados por `WEBHOOK_` — por exemplo `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` ([09:28]–[09:29] Bruno/Larissa).
3. **Logger:** reutilizar o Pino já configurado em `src/shared/logger/index.ts`; não introduzir outro logger ([09:29] Bruno).
4. **Middleware de erro:** não alterar o contrato de `src/middlewares/error.middleware.ts`; ele já trata `AppError` e produz `{ error: { code, message, details } }` ([09:29] Bruno).
5. **Validação:** usar Zod nos schemas do módulo, seguindo o padrão de `order.schemas.ts`, `customer.schemas.ts` etc. ([09:30] Larissa).
6. **Autorização:** usar `authenticate` em todas as rotas e `requireRole('ADMIN')` no replay de DLQ, reutilizando `src/middlewares/auth.middleware.ts` ([09:36] Larissa).
7. **Banco:** reutilizar Prisma e MySQL; o worker cria seu próprio `PrismaClient` por processo, via `createPrismaClient()` ([09:30] Bruno/Diego).
8. **IDs:** UUID em todas as novas entidades, seguindo o padrão do projeto ([09:51] Larissa).
9. **Paginação:** reutilizar `paginated()` de `src/shared/http/response.ts` para listagens e histórico.
10. **Nomes de tabela:** usar `@@map` em snake_case, como `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`, seguindo o padrão de `orders`, `order_status_history` etc.

## Alternativas Consideradas

### Alternativa A — Microserviço separado para webhooks

- **Trade-off que levou ao descarte:** exigiria novo deploy, novo banco ou comunicação entre serviços, nova autenticação e nova observabilidade; o time é pequeno e a feature é um módulo do OMS ([09:07] Diego, [09:30] Larissa).
- **Custo evitado:** simplicidade operacional e consistência de domínio.

### Alternativa B — Novo formato de erro ou nova biblioteca de validação

- **Trade-off que levou ao descarte:** quebraria a consistência da API e exigiria adaptar o middleware central; o formato atual já atende ([09:29] Bruno).
- **Custo evitado:** padronização e menos código novo.

### Alternativa C — Logger próprio no worker

- **Trade-off que levou ao descarte:** duplicaria configuração, redaction e formato de logs; o Pino já está no projeto inteiro ([09:29] Bruno).
- **Custo evitado:** consistência de observabilidade.

### Alternativa D — Repositório de webhook injetado no `OrderService`

- **Trade-off que levou ao descarte:** criar dependência circular ou injetar um repositório inteiro só para enfileirar um evento; a reunião preferiu uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da transação ([09:41] Bruno/Diego).
- **Custo evitado:** acoplamento desnecessário e dificuldade de testar a transação.

## Consequências

### Positivas

- **Consistência:** a API de webhooks se comporta como o restante do OMS.
- **Menos código e menos dependências:** nenhuma nova biblioteca de log, validação ou erro.
- **Facilidade de onboarding:** quem já conhece os módulos de pedidos entende o módulo de webhooks.
- **Aproveitamento do middleware:** erros `WEBHOOK_*` já saem no formato padrão sem alteração no handler central.
- **Testabilidade:** o padrão de Supertest/Vitest existente pode ser reutilizado.

### Negativas / trade-offs

- **Acoplamento às convenções:** mudanças futuras nos padrões do projeto afetam o módulo.
- **Crescimento do módulo:** o módulo de webhooks concentra CRUD, worker, assinatura e payload; é preciso disciplina para não virar um "mini-framework".
- **Limitações do padrão atual:** o middleware de erro e o formato de paginação foram pensados para CRUDs simples; o histórico de entregas precisará de parâmetros próprios.
- **Sem isolamento de deploy:** o worker é separado em processo, mas compartilha o mesmo banco e a mesma stack; um incidente de banco afeta ambos.

## Referências

- `TRANSCRICAO.md` — [09:27] Bruno; [09:28] Bruno/Diego; [09:29] Bruno/Larissa; [09:30] Bruno/Diego/Larissa; [09:36] Larissa; [09:41] Bruno/Diego; [09:51] Larissa
- `src/modules/orders/order.controller.ts` — padrão de controller
- `src/modules/orders/order.service.ts` — padrão de service e transação
- `src/modules/orders/order.repository.ts` — padrão de repository
- `src/modules/orders/order.routes.ts` — padrão de rotas
- `src/modules/orders/order.schemas.ts` — padrão de schemas Zod
- `src/shared/errors/app-error.ts` — classe base de erros
- `src/shared/errors/http-errors.ts` — erros específicos existentes
- `src/middlewares/error.middleware.ts` — tratamento centralizado
- `src/middlewares/auth.middleware.ts` — `authenticate` e `requireRole`
- `src/shared/logger/index.ts` — Pino
- `src/shared/http/response.ts` — paginação
- `src/config/database.ts` — `createPrismaClient()`
- `prisma/schema.prisma` — convenções de modelagem
