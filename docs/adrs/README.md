# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do Sistema de Webhooks de Notificação de Pedidos.

Cada arquivo registra **uma** decisão arquitetural fechada, no formato MADR, com as seções Status, Contexto, Decisão, Alternativas Consideradas e Consequências (positivas e negativas). Todas as decisões têm origem na reunião técnica registrada em [`../../TRANSCRICAO.md`](../../TRANSCRICAO.md) e são referenciadas por timestamp e falante.

Nomenclatura: `ADR-NNN-titulo-em-kebab-case.md`.

## Índice

| ADR | Decisão | Status | Origem na transcrição |
| --- | --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL para publicação de eventos de pedido | Aceito | `[09:03]`–`[09:08]` |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado, lendo a outbox por polling de 2 segundos | Aceito | `[09:08]`–`[09:13]`, `[09:28]` |
| [ADR-003](ADR-003-retry-com-backoff-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela dedicada | Aceito | `[09:14]`–`[09:19]`, `[09:35]`–`[09:36]` |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | Aceito | `[09:19]`–`[09:24]`, `[09:44]` |
| [ADR-005](ADR-005-at-least-once-com-x-event-id.md) | Entrega at-least-once com idempotência delegada ao cliente via `X-Event-Id` | Aceito | `[09:24]`–`[09:26]`, `[09:44]` |
| [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso máximo dos padrões existentes do projeto no módulo de webhooks | Aceito | `[09:27]`–`[09:30]`, `[09:36]` |
| [ADR-007](ADR-007-snapshot-do-payload-na-outbox.md) | Payload renderizado como snapshot no momento da inserção na outbox | Aceito | `[09:43]`, `[09:51]`–`[09:52]` |

## Como os ADRs se relacionam com o resto do pacote

- O [RFC](../RFC.md) consolida estas decisões em uma proposta técnica única e lista o que ficou em aberto.
- O [FDD](../FDD.md) parte delas como premissas e detalha a implementação.
- O [TRACKER](../TRACKER.md) mapeia cada decisão à sua origem na transcrição ou no código.

## Decisões deliberadamente não promovidas a ADR

Alguns pontos técnicos foram discutidos e decididos na reunião, mas classificados pelos próprios participantes como não-arquiteturais. Estão especificados no FDD, não aqui:

- **TLS obrigatório na URL do webhook** — `[09:23] Sofia`: *"nem é decisão arquitetural, é só uma validação no schema Zod"*.
- **Teto de 64KB por payload** — `[09:24] Larissa`: *"não vejo como decisão arquitetural separada, é só requisito não funcional"*.
- **Timeout de 10s no HTTP call** — `[09:42] Diego`; parâmetro operacional da política de resiliência de [ADR-003](ADR-003-retry-com-backoff-e-dlq.md).
- **Conjunto de headers do request** (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`) — `[09:44]`; contrato detalhado no FDD, com a semântica de cada um herdada de [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) e [ADR-005](ADR-005-at-least-once-com-x-event-id.md).
