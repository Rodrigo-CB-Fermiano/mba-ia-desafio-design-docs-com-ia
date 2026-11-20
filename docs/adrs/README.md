# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do Sistema de Webhooks de Notificação de Pedidos.

Cada arquivo segue o formato MADR e a nomenclatura `ADR-NNN-titulo-em-kebab-case.md`.

| ADR | Decisão |
| --- | --- |
| [ADR-001](ADR-001-padrao-outbox-no-mysql.md) | Padrão Outbox transacional no MySQL existente |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado com polling de 2 segundos |
| [ADR-003](ADR-003-politica-de-retry-e-dlq.md) | Política de retry com backoff exponencial e DLQ |
| [ADR-004](ADR-004-autenticacao-hmac-sha256-por-endpoint.md) | Autenticação HMAC-SHA256 com secret por endpoint |
| [ADR-005](ADR-005-garantia-at-least-once-com-x-event-id.md) | Garantia at-least-once com `X-Event-Id` |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes do projeto |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload na inserção da outbox |

A rastreabilidade de cada decisão está em [`docs/TRACKER.md`](../TRACKER.md).
