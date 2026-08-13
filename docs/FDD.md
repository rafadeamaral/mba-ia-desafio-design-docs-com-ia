# FDD — Sistema de Webhooks de Notificação de Pedidos

| | |
| --- | --- |
| **Documento** | Feature Design Document |
| **Status** | Pronto para implementação |
| **Data** | 2026-08-13 |
| **Autores** | Larissa (Tech Lead), com Bruno (Eng. Pedidos) e Diego (Eng. Plataforma) |
| **Premissas** | As decisões dos [ADRs 001–007](adrs/) estão fechadas e são tratadas aqui como dadas |
| **Documentos relacionados** | [PRD](PRD.md) · [RFC](RFC.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

> Este documento responde **"como construir"**. O "por quê" está nos [ADRs](adrs/); a proposta em nível de arquitetura e as questões em aberto estão no [RFC](RFC.md).

---

## 1. Contexto e motivação técnica

O OMS hoje muda o status de um pedido em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126`), dentro de uma única transação Prisma que executa quatro operações acopladas: valida a transição contra a máquina de estados de `src/modules/orders/order.status.ts`, ajusta `stockQuantity` dos produtos quando aplicável, atualiza `orders` e insere em `order_status_history`.

O sistema **não possui nenhum mecanismo de notificação externa, evento, fila ou webhook**. Não há cliente HTTP de saída, não há processo de background, não há tabela de eventos.

O desafio técnico é acoplar a geração do evento a essa transação — para não haver estado em que o status mudou e a notificação se perdeu — sem acoplar a **disponibilidade** da operação de pedidos à infraestrutura de terceiros. A solução é a outbox transacional ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) com entrega assíncrona por um processo separado ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

## 2. Objetivos técnicos

| # | Objetivo | Verificável por |
| --- | --- | --- |
| OT-01 | Garantir que o registro do evento e a mudança de status compartilhem o mesmo destino transacional | Teste de integração forçando erro no insert da outbox e verificando que o status **não** mudou |
| OT-02 | Manter a latência do caminho feliz abaixo de 10s entre o commit e o `POST` no cliente | Métrica `webhook_delivery_latency_seconds` (p95) |
| OT-03 | Não aumentar a latência p95 de `PATCH /orders/:id/status` de forma perceptível | Comparação antes/depois no log `http_request` de `src/middlewares/request-logger.middleware.ts` |
| OT-04 | Entregar todo evento ou registrá-lo na DLQ — nenhum evento em estado indeterminado | Query de auditoria: `outbox(PENDING) + outbox(DELIVERED) + dead_letter` = total gerado |
| OT-05 | Não introduzir nenhuma dependência nova em `package.json` | Diff do `package.json` contendo apenas o script `worker` |
| OT-06 | Todo erro do módulo trafegar pelo `errorMiddleware` existente sem alteração nele | Testes de contrato verificando o envelope `{ error: { code, message, details } }` |

## 3. Escopo e exclusões

### Dentro do escopo

- Modelagem das tabelas de configuração, outbox, tentativas de entrega e DLQ.
- Módulo `src/modules/webhooks/` com CRUD de configuração, rotação de secret, histórico de entregas e replay administrativo.
- Entrypoint `src/worker.ts` e o processador de outbox.
- Assinatura HMAC-SHA256, geração e rotação de secret.
- Alteração pontual em `OrderService.changeStatus` para publicar o evento.
- Extensão de `redactPaths` em `src/shared/logger/index.ts`.

### Fora do escopo

| Exclusão | Origem |
| --- | --- |
| Webhooks de entrada (inbound) | `[09:02] Marcos`: *"Só saindo da gente pra eles"* |
| Notificação por e-mail ao cliente quando o webhook dele falha | `[09:37] Larissa`: *"Email tá fora de escopo dessa fase"* |
| Dashboard/painel visual para o cliente | `[09:40] Larissa`: *"Não, agora não. Só endpoints"* |
| Rate limiting de saída por endpoint | `[09:39] Diego`: *"A gente observa e implementa se virar problema"* |
| Arquivamento/expurgo de linhas entregues da outbox | `[09:08] Diego`: *"fora do escopo dessa feature"* |
| Múltiplos workers e ordering global | `[09:13] Diego`: *"problema do futuro, não agora"* |
| Autorização granular por customer no CRUD | `[09:37] Sofia`: *"Por enquanto sim. Mais pra frente a gente pode endurecer"* |

## 4. Modelo de dados

Quatro modelos novos em `prisma/schema.prisma`. Todos seguem o padrão do arquivo: `@id @default(uuid()) @db.Char(36)` ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md), `[09:51] Larissa`).

```prisma
enum WebhookEventStatus {
  PENDING      // aguardando processamento
  PROCESSING   // retirado pelo worker no ciclo atual
  FAILED       // tentativa falhou, aguardando próxima janela de retry
  DELIVERED    // 2xx recebido do cliente
}

model WebhookEndpoint {
  id                     String   @id @default(uuid()) @db.Char(36)
  customerId             String   @db.Char(36)
  url                    String   @db.VarChar(2048)     // obrigatoriamente https
  secret                 String   @db.VarChar(255)      // secret ativa
  previousSecret         String?  @db.VarChar(255)      // válida durante o grace period
  previousSecretExpiresAt DateTime?                     // now + 24h na rotação
  subscribedStatuses     Json                           // ex.: ["SHIPPED","DELIVERED"]
  active                 Boolean  @default(true)
  createdAt              DateTime @default(now())
  updatedAt              DateTime @updatedAt

  customer   Customer                @relation(fields: [customerId], references: [id])
  events     WebhookOutboxEvent[]
  deliveries WebhookDeliveryAttempt[]
  deadLetters WebhookDeadLetter[]

  @@index([customerId])
  @@index([active])
  @@map("webhook_endpoints")
}

model WebhookOutboxEvent {
  id            String             @id @default(uuid()) @db.Char(36)  // == X-Event-Id
  endpointId    String             @db.Char(36)
  orderId       String             @db.Char(36)
  eventType     String             @db.VarChar(64)      // "order.status_changed"
  fromStatus    OrderStatus?
  toStatus      OrderStatus
  payload       Json                                    // snapshot renderizado (ADR-007)
  status        WebhookEventStatus @default(PENDING)
  attempts      Int                @default(0)
  nextAttemptAt DateTime           @default(now())
  lastError     String?            @db.VarChar(500)
  createdAt     DateTime           @default(now())
  deliveredAt   DateTime?

  endpoint WebhookEndpoint @relation(fields: [endpointId], references: [id])

  @@index([status, nextAttemptAt])   // query principal do worker
  @@index([createdAt])
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDeliveryAttempt {
  id             String   @id @default(uuid()) @db.Char(36)
  eventId        String   @db.Char(36)
  endpointId     String   @db.Char(36)
  attemptNumber  Int
  responseStatus Int?                                   // null quando timeout/erro de conexão
  responseBody   String?  @db.Text                      // truncado
  durationMs     Int
  error          String?  @db.VarChar(500)
  attemptedAt    DateTime @default(now())

  endpoint WebhookEndpoint @relation(fields: [endpointId], references: [id])

  @@index([eventId])
  @@index([endpointId, attemptedAt])   // GET /webhooks/:id/deliveries
  @@map("webhook_delivery_attempts")
}

model WebhookDeadLetter {
  id             String    @id @default(uuid()) @db.Char(36)
  originalEventId String   @db.Char(36)                 // preservado para o replay (ADR-005)
  endpointId     String    @db.Char(36)
  payload        Json
  failureReason  String    @db.VarChar(500)
  attempts       Int
  createdAt      DateTime  @default(now())
  replayedAt     DateTime?
  replayedById   String?   @db.Char(36)                 // auditoria exigida por Sofia [09:36]

  endpoint WebhookEndpoint @relation(fields: [endpointId], references: [id])

  @@index([endpointId])
  @@index([createdAt])
  @@map("webhook_dead_letter")
}
```

**Notas de modelagem:**

- Os quatro estados da outbox são exatamente os que Diego nomeou (`[09:08]`): *"pendente, processando, falhou, entregue"*.
- O índice composto `(status, nextAttemptAt)` atende diretamente a query do worker; Diego previu índice em status e em `created_at` (`[09:08]`), e `nextAttemptAt` entra porque o backoff exige filtro temporal.
- **Uma linha de outbox por endpoint interessado.** Se dois webhooks do mesmo customer escutam `SHIPPED`, a mudança gera duas linhas, com dois `X-Event-Id` distintos. Isso mantém `X-Event-Id` único por entrega lógica e viabiliza a dedup do cliente.
- `previousSecret` + `previousSecretExpiresAt` implementam o grace period de 24h ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

Ocorre dentro da transação existente de `changeStatus` (`src/modules/orders/order.service.ts:131`), após o insert em `order_status_history` e antes do `findUnique` final.

```
changeStatus(id, input, userId)
└── prisma.$transaction(async (tx) => {
      … validações e escritas atuais, sem alteração …
      await tx.orderStatusHistory.create({ … })

      ── NOVO ──
      await publishWebhookEvent(tx, order, from, to)
        │
        ├─ 1. SELECT endpoints ativos do customer cujo subscribedStatuses contenha `to`
        │       └─ nenhum? retorna sem escrever nada  ......................... [09:34] Bruno
        │
        ├─ 2. para cada endpoint:
        │       ├─ renderiza o payload JSON (snapshot) ....................... ADR-007
        │       ├─ valida tamanho ≤ 64KB
        │       │     └─ excede? lança WebhookPayloadTooLargeError → rollback . [09:23] Sofia
        │       └─ INSERT webhook_outbox (status=PENDING, nextAttemptAt=now())
        │
        └─ 3. erro em qualquer ponto → exceção propaga → rollback da transação
                                                                     ......... [09:40] Bruno
      … findUnique final e return, sem alteração …
   })
```

A assinatura foi definida por Bruno (`[09:41]`): `publishWebhookEvent(tx, order, fromStatus, toStatus)`. Diego endossou a forma (`[09:41]`): *"função pura recebendo o tx. Não precisa injetar repository inteiro."* Ela vive em `src/modules/webhooks/webhook.publisher.ts` e é importada diretamente pelo `OrderService` — **não** entra no construtor.

**Por que o filtro acontece aqui e não no envio:** Diego perguntou (`[09:34]`) e Bruno respondeu: *"Na inserção. Se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela."*

### 5.2 Processamento pelo worker

```
src/worker.ts  (npm run worker)
└── bootstrap: createPrismaClient() próprio + logger Pino + handlers SIGINT/SIGTERM
    │                                          espelhando src/server.ts
    └── loop infinito, ciclo de 2s ............................... [09:09] Diego
        │
        ├─ 1. SELECT * FROM webhook_outbox
        │       WHERE status IN ('PENDING','FAILED') AND nextAttemptAt <= NOW()
        │       ORDER BY createdAt ASC
        │       LIMIT <batchSize>                     ← batch pequeno [09:08]
        │
        ├─ 2. marca o batch como PROCESSING (evita reprocesso se o ciclo demorar)
        │
        ├─ 3. para cada evento, sequencialmente (single-worker preserva
        │      ordering por order_id) ............................. [09:12] Diego
        │      │
        │      ├─ carrega o endpoint; inativo? → marca DELIVERED=false e descarta
        │      │    com log WEBHOOK_ENDPOINT_INACTIVE (não consome tentativa)
        │      ├─ assina: HMAC-SHA256(payload, endpoint.secret) ..... ADR-004
        │      ├─ POST na url com os headers da seção 7, timeout 10s . [09:42] Diego
        │      ├─ grava WebhookDeliveryAttempt (status, body truncado, durationMs)
        │      │
        │      ├─ resposta 2xx  → status=DELIVERED, deliveredAt=now()
        │      └─ demais casos  → seção 5.3
        │
        └─ 4. dorme até completar o ciclo de 2s
```

O worker **não consulta as tabelas de domínio**. Ele lê a outbox e envia o `payload` gravado, consequência direta de [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md).

**Recuperação de eventos travados em `PROCESSING`:** se o processo morrer entre o passo 2 e o 3, linhas ficam presas. O worker, no startup, promove de volta para `PENDING` toda linha em `PROCESSING` com `updatedAt` anterior a um limiar (sugestão: 5 minutos). Isso é seguro porque a entrega é at-least-once ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)) — no pior caso o cliente recebe duplicado e deduplica pelo `X-Event-Id`.

### 5.3 Retry

Falha é: status HTTP fora da faixa 2xx, erro de conexão, ou estouro do timeout de 10 segundos.

```
falha na tentativa N
│
├─ N < 5 ?
│    ├─ SIM → status = FAILED
│    │        attempts = N
│    │        lastError = <motivo truncado em 500 chars>
│    │        nextAttemptAt = now() + BACKOFF[N]
│    │
│    └─ NÃO → seção 5.4 (DLQ)
│
└─ BACKOFF = [1min, 5min, 30min, 2h, 12h] ................. [09:17] Diego
             ↑ após 1ª  ↑2ª  ↑3ª   ↑4ª  ↑5ª falha
```

| Tentativa | Momento aproximado após a 1ª falha |
| --- | --- |
| 2ª | +1 min |
| 3ª | +6 min |
| 4ª | +36 min |
| 5ª | +2h36 |
| DLQ | +14h36 |

Janela total de ~15 horas, aceita por Marcos (`[09:17]`): *"Se um cliente meu cair por 15 horas, ele já tá com problema sério dele. Acho aceitável."*

O agendamento é **por registro**, via `nextAttemptAt`. O worker nunca bloqueia esperando um backoff — ele apenas não seleciona o evento até a janela vencer. Isso é obrigatório dado o single-worker de [ADR-002](adrs/ADR-002-worker-separado-em-polling.md).

### 5.4 Dead Letter Queue e replay

```
5ª tentativa falhou
│
└─ transação:
     ├─ INSERT webhook_dead_letter (originalEventId, endpointId, payload,
     │                              failureReason, attempts=5)
     └─ UPDATE webhook_outbox SET status='FAILED'   ← permanece como trilha
                                                       histórica, fora da
                                                       janela de seleção

POST /api/v1/admin/webhooks/dead-letter/:id/replay   (requireRole('ADMIN'))
│
└─ transação:
     ├─ valida: DLQ existe e replayedAt IS NULL (senão WEBHOOK_ALREADY_REPLAYED)
     ├─ INSERT webhook_outbox com id = originalEventId  ← X-Event-Id preservado,
     │      status='PENDING', attempts=0, nextAttemptAt=now()          ADR-005
     └─ UPDATE webhook_dead_letter SET replayedAt=now(), replayedById=req.user.id
                                                            auditoria [09:36] Sofia
```

O `originalEventId` é reutilizado deliberadamente: um cliente que já havia recebido e processado aquele evento antes de a resposta se perder consegue deduplicar o replay pelo mesmo `X-Event-Id`.

## 6. Contratos públicos

Todos os endpoints ficam sob o prefixo `/api/v1`, montado em `src/app.ts:67`, e passam pelo middleware `authenticate` (`src/middlewares/auth.middleware.ts:27`), seguindo o padrão de `src/modules/orders/order.routes.ts:14`.

Erros usam o envelope produzido por `src/middlewares/error.middleware.ts:14`:

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found", "details": {} } }
```

> **Premissa a ratificar (Q4 do [RFC](RFC.md#5-questões-em-aberto)):** Larissa deixou aberto se o `customerId` vem no body ou no path (`[09:32]`). Este documento assume **body na criação, query param na listagem**. Se a revisão decidir por path, os contratos 6.1 e 6.2 mudam de forma.

### 6.1 `POST /api/v1/webhooks` — cadastrar webhook

Roles: qualquer usuário autenticado (`[09:37] Sofia`).

**Request**

```http
POST /api/v1/webhooks
Authorization: Bearer <jwt>
Content-Type: application/json
```
```json
{
  "customerId": "8f14e45f-ceea-467a-9f4f-1e2b3c4d5e6f",
  "url": "https://api.atlascomercial.com.br/integracoes/oms/webhook",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`**

```json
{
  "id": "3b241101-e2bb-4255-8caf-4136c566a962",
  "customerId": "8f14e45f-ceea-467a-9f4f-1e2b3c4d5e6f",
  "url": "https://api.atlascomercial.com.br/integracoes/oms/webhook",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9f2c1a7b4e8d63f05a1c9e2b7d4f8a06c3e5b1d9",
  "createdAt": "2026-08-13T14:02:11.000Z"
}
```

**Semântica.** A `secret` é gerada por nós e **retornada apenas nesta resposta** (`[09:31] Marcos`: *"secret é gerada pela gente e devolvida na criação"*). Nenhum outro endpoint a devolve; perdida, só resta rotacionar. `subscribedStatuses` aceita valores do enum `OrderStatus` de `prisma/schema.prisma:16`. A `url` precisa ser `https` (`[09:23] Sofia`).

| Status | Quando |
| --- | --- |
| `201` | Criado |
| `400` | `WEBHOOK_INVALID_URL` (não-https ou malformada), `WEBHOOK_INVALID_EVENT_TYPE`, `VALIDATION_ERROR` |
| `401` | Sem token válido |
| `404` | `NOT_FOUND` — `customerId` inexistente |
| `409` | `WEBHOOK_URL_ALREADY_REGISTERED` |

### 6.2 `GET /api/v1/webhooks?customerId=<uuid>` — listar webhooks de um customer

Pedido por Bruno (`[09:33]`): *"GET pra listar os webhooks de um customer"*.

**Response `200 OK`** — mesmo formato paginado de `src/shared/http/response.ts`, usado hoje por `OrderService.list`:

```json
{
  "data": [
    {
      "id": "3b241101-e2bb-4255-8caf-4136c566a962",
      "customerId": "8f14e45f-ceea-467a-9f4f-1e2b3c4d5e6f",
      "url": "https://api.atlascomercial.com.br/integracoes/oms/webhook",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotatedAt": null,
      "createdAt": "2026-08-13T14:02:11.000Z",
      "updatedAt": "2026-08-13T14:02:11.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

**Semântica.** O campo `secret` **nunca** aparece em listagem ou consulta. Query params `page` e `pageSize` seguem os defaults de `listOrdersQuerySchema` (`src/modules/orders/order.schemas.ts:23`): 1 e 20, com teto de 100.

### 6.3 `PATCH /api/v1/webhooks/:id` — editar webhook

**Request**

```json
{ "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"], "active": false }
```

**Response `200 OK`** — o recurso atualizado, no mesmo formato de 6.2.

**Semântica.** Campos aceitos: `url`, `subscribedStatuses`, `active`. A `secret` **não** é editável por aqui — rotação tem endpoint próprio (6.5). Desativar (`active: false`) interrompe novas inserções na outbox, mas **não** cancela eventos já enfileirados.

| Status | Quando |
| --- | --- |
| `200` | Atualizado |
| `400` | `WEBHOOK_INVALID_URL`, `WEBHOOK_INVALID_EVENT_TYPE` |
| `404` | `WEBHOOK_NOT_FOUND` |

### 6.4 `DELETE /api/v1/webhooks/:id` — remover webhook

**Response `204 No Content`**, sem corpo — mesmo padrão de `OrderController.delete` (`src/modules/orders/order.controller.ts:48`).

**Semântica.** Remoção lógica: o registro é marcado como removido e para de receber eventos, mas o histórico de entregas e as linhas de DLQ associadas são preservados. Erro `404 WEBHOOK_NOT_FOUND` se não existir.

### 6.5 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar a secret

Decidido por Sofia (`[09:21]`): *"Endpoint pro cliente conseguir pedir nova secret pela API."*

**Response `200 OK`**

```json
{
  "id": "3b241101-e2bb-4255-8caf-4136c566a962",
  "secret": "whsec_5d8e2f1a9c6b47e30f2a8d1c5b9e4f7a2c6d0b83",
  "previousSecretExpiresAt": "2026-08-14T14:40:02.000Z",
  "rotatedAt": "2026-08-13T14:40:02.000Z"
}
```

**Semântica.** A secret anterior permanece válida por **24 horas** (`[09:21] Sofia`). Durante a janela, o worker assina com a **nova** secret; a antiga existe para que o cliente possa validar requisições em trânsito e migrar seus sistemas gradualmente. Expirado o prazo, `previousSecret` é limpo.

| Status | Quando |
| --- | --- |
| `200` | Rotacionada |
| `404` | `WEBHOOK_NOT_FOUND` |
| `409` | `WEBHOOK_ROTATION_IN_PROGRESS` — já existe rotação com grace period ativo |

### 6.6 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Pedido por Marcos (`[09:34]`): *"esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta"*.

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "a1f3c9d2-77b4-4e18-9c3a-5d6e7f801b2c",
      "eventId": "c8d0f3e4-1a2b-4c5d-8e9f-0a1b2c3d4e5f",
      "attemptNumber": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 342,
      "error": null,
      "attemptedAt": "2026-08-13T14:07:03.221Z",
      "payload": {
        "event_id": "c8d0f3e4-1a2b-4c5d-8e9f-0a1b2c3d4e5f",
        "event_type": "order.status_changed",
        "order_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
        "from_status": "PROCESSING",
        "to_status": "SHIPPED"
      }
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

**Semântica.** Ordenado por `attemptedAt` decrescente. `pageSize` default 100, alinhado ao pedido de Marcos. `responseBody` é truncado no armazenamento para não inflar a tabela. Uma tentativa que estourou timeout tem `responseStatus: null` e `error` preenchido. Retorna `404 WEBHOOK_NOT_FOUND` se o webhook não existir.

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar da DLQ

Endpoint exatamente como Diego nomeou (`[09:35]`). **Exige role `ADMIN`**, via `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts:49`), no mesmo padrão de `src/modules/users/user.routes.ts:15`.

**Response `202 Accepted`**

```json
{
  "deadLetterId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "eventId": "c8d0f3e4-1a2b-4c5d-8e9f-0a1b2c3d4e5f",
  "status": "PENDING",
  "replayedAt": "2026-08-13T15:12:44.000Z",
  "replayedBy": "0b3f8a21-6c4d-4e97-b2a5-8d1e3f7c0a94"
}
```

**Semântica.** `202` e não `200` porque a operação apenas reenfileira: a entrega ocorrerá no próximo ciclo do worker, em até 2 segundos. O `eventId` retornado é o **original**, preservado para que a dedup do cliente funcione. A operação é registrada em `replayedById` e emitida no log Pino como evento auditável (`[09:36] Sofia`).

| Status | Quando |
| --- | --- |
| `202` | Reenfileirado |
| `401` | Sem token |
| `403` | `FORBIDDEN` — role diferente de `ADMIN` |
| `404` | `WEBHOOK_DEAD_LETTER_NOT_FOUND` |
| `409` | `WEBHOOK_ALREADY_REPLAYED` |

## 7. Contrato de saída (a requisição que o cliente recebe)

### Headers

Conjunto fechado em `[09:44]`, com o `X-Webhook-Id` sugerido por Sofia no mesmo bloco.

| Header | Conteúdo | Origem |
| --- | --- | --- |
| `Content-Type` | `application/json` | `[09:44] Diego` |
| `X-Event-Id` | UUID do evento, estável entre tentativas e replays | `[09:25]`, `[09:44] Diego` |
| `X-Signature` | `sha256=<hex>` do HMAC-SHA256 do corpo, com a secret do endpoint | `[09:20] Sofia` |
| `X-Timestamp` | ISO 8601 do envio; permite ao cliente detectar replay attack | `[09:44] Diego` |
| `X-Webhook-Id` | UUID do cadastro de webhook, para clientes com vários endpoints | `[09:44] Sofia` |

### Payload

```json
{
  "event_id": "c8d0f3e4-1a2b-4c5d-8e9f-0a1b2c3d4e5f",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-13T14:06:58.104Z",
  "order_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "order_number": "ORD-000482",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "8f14e45f-ceea-467a-9f4f-1e2b3c4d5e6f",
  "total_cents": 149900
}
```

O conjunto de campos é o que Diego especificou (`[09:43]`). Ele mencionou *"os campos básicos da order tipo total_cents"* — `total_cents` é o único nomeado, e o exemplo acima se restringe a ele; a inclusão de `subtotal_cents` e `discount_cents` (presentes em `prisma/schema.prisma:79`) fica como decisão de implementação a ratificar na revisão.

**Ausência deliberada de `items`.** Diego (`[09:43]`): *"Não manda items pra não inflar. Se o cliente quiser detalhes, ele bate no GET /orders/:id depois."* Bruno concordou (`[09:44]`): *"mantém payload enxuto"*.

**O corpo enviado é byte a byte igual ao `payload` gravado na outbox** ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md)), o que torna a assinatura estável entre tentativas. A serialização precisa ser determinística — o JSON assinado e o JSON enviado têm de ser exatamente a mesma string.

### Verificação pelo cliente

```
assinatura_esperada = "sha256=" + hex(HMAC_SHA256(secret, corpo_bruto_do_request))
```

Comparação deve usar função de tempo constante. Durante o grace period de rotação, o cliente pode ter duas secrets válidas e deve aceitar se qualquer uma delas conferir.

### Resposta esperada do cliente

Qualquer status `2xx` é sucesso. Qualquer outra coisa — incluindo `3xx` (não seguimos redirects) — é falha e entra em retry. O corpo da resposta é ignorado, apenas registrado em `webhook_delivery_attempts` para debug.

## 8. Matriz de erros

Todos os códigos usam o prefixo `WEBHOOK_`, decidido por Larissa (`[09:29]`): *"Prefixo WEBHOOK_ pra tudo do módulo"*. As classes herdam da hierarquia de `src/shared/errors/http-errors.ts`, o que faz o `errorMiddleware` produzir a resposta correta sem nenhuma alteração.

| Código | HTTP | Classe base | Quando ocorre | `details` |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `NotFoundError` | Webhook inexistente ou removido | — |
| `WEBHOOK_INVALID_URL` | 400 | `ValidationError` | URL malformada ou com esquema diferente de `https` | `{ url, reason }` |
| `WEBHOOK_SECRET_REQUIRED` | 422 | `UnprocessableEntityError` | Tentativa de assinar entrega de endpoint sem secret ativa (estado inconsistente) | `{ endpointId }` |
| `WEBHOOK_INVALID_EVENT_TYPE` | 400 | `ValidationError` | `subscribedStatuses` contém valor fora do enum `OrderStatus` | `{ invalid: [...] }` |
| `WEBHOOK_URL_ALREADY_REGISTERED` | 409 | `ConflictError` | Já existe webhook ativo com a mesma URL para o customer | `{ customerId, url }` |
| `WEBHOOK_ROTATION_IN_PROGRESS` | 409 | `ConflictError` | Rotação pedida com grace period ainda ativo | `{ previousSecretExpiresAt }` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `UnprocessableEntityError` | Payload renderizado excede 64KB | `{ sizeBytes, limitBytes: 65536 }` |
| `WEBHOOK_ENDPOINT_INACTIVE` | 409 | `ConflictError` | Operação sobre webhook com `active: false` | `{ endpointId }` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `NotFoundError` | Registro de DLQ inexistente | — |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | `ConflictError` | DLQ já reprocessada (`replayedAt` preenchido) | `{ replayedAt, replayedById }` |
| `WEBHOOK_DELIVERY_TIMEOUT` | — | interno (worker) | Cliente não respondeu em 10s | `{ eventId, endpointId, timeoutMs: 10000 }` |
| `WEBHOOK_DELIVERY_FAILED` | — | interno (worker) | Resposta não-2xx ou erro de conexão | `{ eventId, responseStatus }` |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — | interno (worker) | 5ª tentativa falhou; evento movido para DLQ | `{ eventId, attempts: 5 }` |

**Notas.**

- `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` foram nomeados literalmente por Bruno (`[09:28]`). Os demais seguem o mesmo padrão, derivado das regras decididas na reunião e da convenção de `src/shared/errors/http-errors.ts`.
- **Sobre `WEBHOOK_SECRET_REQUIRED`:** como a secret é sempre gerada por nós (`[09:31] Marcos`), o cliente nunca a informa — logo o código não pode significar "faltou a secret no request". A semântica adotada acima é a única coerente com as demais decisões: guarda de integridade no momento da assinatura. Vale confirmar na revisão se era essa a intenção de Bruno.
- Os três últimos códigos não são erros HTTP de API: são rótulos de log e de `lastError`/`failureReason`, para que a busca no log e a busca na tabela usem o mesmo vocabulário.

## 9. Estratégias de resiliência

| Mecanismo | Configuração | Origem |
| --- | --- | --- |
| Timeout HTTP | 10s por tentativa | `[09:42] Diego` |
| Tentativas | 5 no total | `[09:15]`–`[09:16]` |
| Backoff | 1m, 5m, 30m, 2h, 12h | `[09:17] Diego` |
| Destino terminal | `webhook_dead_letter` | `[09:18] Diego` |
| Recuperação | Replay manual, role `ADMIN` | `[09:18]`, `[09:36]` |
| Teto de payload | 64KB, erro (não truncamento) | `[09:23]`, `[09:24]` |
| Batch por ciclo | Pequeno, ordenado por `createdAt` | `[09:08] Diego` |
| Isolamento de processo | Worker fora da API | `[09:11] Diego` |

**Contenção de blast radius entre clientes.** Como o worker é single-thread e processa sequencialmente, um endpoint lento que sempre estoura os 10s consome 10 segundos de cada ciclo. Com um batch de N eventos desse cliente, o ciclo passa a durar N×10s e atrasa os demais. Mitigações a considerar na implementação: limitar a quantos eventos do mesmo endpoint entram em um batch, ou desativar temporariamente endpoints com falhas consecutivas. **Não foi decidido na reunião** — está relacionado a Q1 do [RFC](RFC.md#5-questões-em-aberto) e deve ser levado à revisão.

**Fallback.** Não há. Se o cliente não recebe após 5 tentativas, o evento fica na DLQ aguardando ação humana. E-mail de aviso foi explicitamente adiado (`[09:37] Larissa`).

**Shutdown gracioso.** O worker replica o padrão de `src/server.ts:13`: ao receber `SIGINT`/`SIGTERM`, termina o batch corrente, para de selecionar novos eventos, executa `prisma.$disconnect()` e sai. Eventos ainda em `PROCESSING` são recuperados no próximo startup (seção 5.2).

## 10. Observabilidade

Toda a instrumentação usa o Pino já configurado em `src/shared/logger/index.ts`. Bruno foi explícito (`[09:29]`): *"o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo."*

### Logs

Eventos estruturados, seguindo a convenção `snake_case` de `server_started` e `http_request` já usada no projeto:

| Evento | Nível | Campos |
| --- | --- | --- |
| `worker_started` | info | `pollIntervalMs`, `batchSize` |
| `webhook_event_enqueued` | info | `eventId`, `endpointId`, `orderId`, `toStatus` |
| `webhook_delivery_attempt` | info | `eventId`, `endpointId`, `attemptNumber`, `responseStatus`, `durationMs` |
| `webhook_delivery_failed` | warn | `eventId`, `attemptNumber`, `errorCode`, `nextAttemptAt` |
| `webhook_moved_to_dlq` | error | `eventId`, `endpointId`, `attempts`, `failureReason` |
| `webhook_dlq_replayed` | warn | `deadLetterId`, `eventId`, `replayedById` — **auditoria exigida por Sofia** (`[09:36]`) |
| `webhook_secret_rotated` | warn | `endpointId`, `previousSecretExpiresAt` |

**Alteração obrigatória em `src/shared/logger/index.ts`:** a lista `redactPaths` (linha 4) cobre `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`, mas **não** `secret`. Sem estender a lista, uma secret de webhook vaza em log ao serializar o objeto do endpoint. Adicionar `*.secret` e `*.previousSecret`.

**Nada de payload em log.** O payload contém dados de pedido do cliente; ele já vive na outbox e no histórico de entregas. Logs referenciam `eventId`.

### Métricas

Sem stack de métricas no projeto hoje, as três primeiras podem sair de queries sobre as próprias tabelas, e as demais de contadores expostos pelo worker:

| Métrica | Tipo | Para que serve |
| --- | --- | --- |
| `webhook_outbox_pending_total` | gauge | Tamanho da fila. Crescimento sustentado = worker parado ou throughput insuficiente |
| `webhook_outbox_oldest_pending_age_seconds` | gauge | **Sinal primário de saúde.** Deve ficar abaixo de ~4s no caminho feliz; alarme acima de 60s |
| `webhook_dead_letter_total` | counter | Falhas permanentes; alarme em qualquer incremento |
| `webhook_delivery_latency_seconds` | histogram | Latência ponta a ponta (commit → 2xx). Verifica OT-02 contra os 10s de `[09:02]` |
| `webhook_delivery_duration_seconds` | histogram | Duração do `POST`, por endpoint. Identifica o cliente lento da seção 9 |
| `webhook_delivery_attempts_total{result}` | counter | `success` / `failure` / `timeout`, por endpoint |
| `webhook_worker_cycle_duration_seconds` | histogram | Deve ficar bem abaixo dos 2s de ciclo; se ultrapassar, o polling está saturado |

### Tracing

Não há tracing distribuído instalado no projeto. Duas medidas de correlação, ambas sem dependência nova:

1. **`X-Event-Id` como identificador de correlação.** O mesmo UUID aparece na linha da outbox, em toda entrada de log do worker, em todas as tentativas de entrega e no header recebido pelo cliente. Isso permite reconstruir o ciclo de vida completo de um evento com um único `grep` — inclusive quando o cliente abre um chamado citando o `X-Event-Id` que recebeu.
2. **Propagação do `requestId` da API para o evento.** `src/middlewares/request-logger.middleware.ts:6` já gera um `X-Request-Id` por requisição. Gravar esse valor na linha da outbox liga a requisição `PATCH /orders/:id/status` que originou o evento à entrega que ocorreu segundos depois, em outro processo.

Se o time adotar OpenTelemetry no futuro, esses dois campos são os pontos naturais de enxerto de `traceparent`.

## 11. Integração com o sistema existente

Esta seção mapeia cada ponto de contato entre o módulo de webhooks e o código atual.

> **Convenção de leitura.** Todos os caminhos citados abaixo **existem hoje no repositório** e foram verificados, salvo os seis arquivos a serem criados pela feature, sempre identificados como novos: `src/worker.ts`, `src/modules/webhooks/webhook.publisher.ts`, `src/modules/webhooks/webhook.worker.ts`, `src/modules/webhooks/webhook.errors.ts`, `src/modules/webhooks/webhook.schemas.ts` e `tests/webhooks.test.ts` — mais os arquivos padrão do módulo (`webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`).

### 11.1 `src/modules/orders/order.service.ts` — **a alteração crítica**

Bruno classificou assim (`[09:40]`): *"a alteração crítica é dentro do service de orders, no método changeStatus"*.

O método `changeStatus` (linha 126) abre uma transação (linha 131) que hoje executa, nesta ordem: `findUnique` da order com `items`, validação de transição via `canTransition`, `debitStock`/`replenishStock`, `tx.order.update` (linha 158), `tx.orderStatusHistory.create` (linha 159) e um `findUnique` final que monta o retorno (linha 169).

**Alteração:** uma chamada a `publishWebhookEvent(tx, order, from, to)` entre o `orderStatusHistory.create` e o `findUnique` final.

```ts
      await tx.orderStatusHistory.create({ /* … código atual … */ });

      // NOVO — ADR-001: evento na mesma transação
      await publishWebhookEvent(tx, order, from, to);

      const refreshed = await tx.order.findUnique({ /* … código atual … */ });
```

A função recebe o `TxClient` — tipo já declarado no arquivo (`type TxClient = Prisma.TransactionClient`, linha 24) e usado por `debitStock` e `replenishStock`. Diego endossou a forma (`[09:41]`): *"função pura recebendo o tx. Não precisa injetar repository inteiro"*, o que evita mexer no construtor de `OrderService` (linha 27) e, por consequência, na montagem em `buildControllers` (`src/app.ts:42`).

Qualquer exceção lançada dentro de `publishWebhookEvent` aborta a transação inteira — comportamento exigido por Bruno (`[09:40]`) e que já é como `InsufficientStockError` se comporta hoje.

**Não alterado:** `create` (linha 50), `list`, `getById`, `delete`. A criação de pedido gera `PENDING`, mas a reunião tratou exclusivamente de *mudança* de status; nenhum participante pediu evento na criação.

### 11.2 `src/shared/errors/http-errors.ts` e `app-error.ts` — reuso da hierarquia de erros

Bruno (`[09:28]`): *"a gente já tem um padrão. Tem classe `AppError`, classes específicas tipo `InsufficientStockError`, `InvalidStatusTransitionError`. (…) Quero seguir igual pra webhook."*

O módulo cria `src/modules/webhooks/webhook.errors.ts` estendendo as classes existentes, exatamente como `InvalidStatusTransitionError` estende `ConflictError` (linha 45) e `InsufficientStockError` estende `UnprocessableEntityError` (linha 55):

```ts
export class WebhookNotFoundError extends NotFoundError {
  constructor() { super('Webhook'); }   // produz 404 / NOT_FOUND
}

export class WebhookPayloadTooLargeError extends UnprocessableEntityError {
  constructor(sizeBytes: number) {
    super('Webhook payload exceeds the 64KB limit',
          'WEBHOOK_PAYLOAD_TOO_LARGE',
          { sizeBytes, limitBytes: 65536 });
  }
}
```

> **Ajuste necessário em `WebhookNotFoundError`:** `NotFoundError` (linha 27) fixa o código como `NOT_FOUND` e não aceita override. Para emitir `WEBHOOK_NOT_FOUND` conforme decidido (`[09:28]` Bruno, `[09:29]` Larissa), a classe deve estender `AppError` diretamente — `super('Webhook not found', 404, 'WEBHOOK_NOT_FOUND')` — em vez de `NotFoundError`. Nenhuma alteração no arquivo compartilhado é necessária.

### 11.3 `src/middlewares/error.middleware.ts` — nenhuma alteração

Bruno (`[09:29]`): *"O middleware de erro centralizado já trata AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada."* Confirmado no código: a primeira condição (linha 15) é `err instanceof AppError`, e a resposta é montada a partir de `err.statusCode`, `err.errorCode` e `err.details`. Todo erro `WEBHOOK_*` que herde de `AppError` já sai no formato correto.

O tratamento de `Prisma.PrismaClientKnownRequestError` (linha 37) também vale de graça: um `P2002` na constraint de URL única vira `409 CONFLICT` sem código nosso — embora seja preferível validar antes e devolver o código específico `WEBHOOK_URL_ALREADY_REGISTERED`.

### 11.4 `src/middlewares/auth.middleware.ts` — `requireRole` no replay de DLQ

Larissa (`[09:36]`): *"role ADMIN obrigatório no replay e a gente reaproveita o `requireRole` que já existe."*

O middleware está na linha 49 e produz `ForbiddenError` (403) quando o role não confere. O uso é idêntico ao de `src/modules/users/user.routes.ts:15`:

```ts
router.post(
  '/admin/webhooks/dead-letter/:id/replay',
  requireRole('ADMIN'),
  validate({ params: deadLetterIdParamSchema }),
  controller.replayDeadLetter,
);
```

O `authenticate` (linha 27) popula `req.user` com `{ id, email, role }`, e é de `req.user.id` que sai o `replayedById` da auditoria pedida por Sofia (`[09:36]`).

**Ponto de atenção do RFC (Q5):** os únicos roles existentes são `ADMIN` e `OPERATOR` (linha 9). Não há granularidade por customer, então hoje qualquer `OPERATOR` autenticado pode alterar o webhook de qualquer cliente. Sofia aceitou por ora (`[09:37]`).

### 11.5 `src/shared/logger/index.ts` — extensão obrigatória de `redactPaths`

A lista da linha 4 cobre `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`. **Não cobre `secret`.** Como o objeto `WebhookEndpoint` carrega `secret` e `previousSecret`, logar o registro sem estender a lista vaza material criptográfico. É a **única alteração obrigatória em arquivo compartilhado** que o reuso não elimina.

### 11.6 `src/config/database.ts` — `PrismaClient` próprio no worker

Bruno (`[09:30]`): *"Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node."*

O arquivo já expõe a factory `createPrismaClient()` (linha 4) além do singleton `prisma` (linha 10). O `src/worker.ts` chama a factory; a API continua usando o singleton via `src/server.ts:3`. Nenhuma alteração no arquivo.

### 11.7 `src/server.ts` → `src/worker.ts` — nova entrypoint

Larissa (`[09:11]`): *"Tem espaço pra ser uma entry-point nova no projeto. Tipo o que a gente já tem em `src/server.ts`, criar um `src/worker.ts` e um script `npm run worker`."*

`src/worker.ts` espelha a estrutura de `src/server.ts`: função `bootstrap()`, handlers de `SIGINT`/`SIGTERM` (linha 20), `prisma.$disconnect()` no shutdown (linha 16) e `logger.fatal` + `process.exit(1)` no catch do bootstrap (linha 24). No lugar do `app.listen`, entra o loop de polling.

Em `package.json` (scripts, linha 10), seguindo o padrão de `dev` e `start`:

```json
"worker": "tsx watch --env-file=.env src/worker.ts",
"worker:start": "node --env-file=.env dist/worker.js"
```

### 11.8 `src/routes/index.ts` e `src/app.ts` — registro do módulo

`buildApiRouter` (`src/routes/index.ts:21`) recebe um objeto `Controllers` e monta um router por domínio. Adicionar `webhooks: WebhookController` ao tipo `Controllers` (linha 13) e `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))`. Em `src/app.ts`, `buildControllers` (linha 26) instancia repository → service → controller na mesma ordem dos demais módulos (linhas 42–44) e devolve o novo controller no objeto de retorno (linha 46).

### 11.9 `src/middlewares/validate.middleware.ts` e schemas Zod

O `validate({ body, query, params })` (linha 11) converte `ZodError` em `ValidationError`. `src/modules/webhooks/webhook.schemas.ts` segue o formato de `src/modules/orders/order.schemas.ts`: `z.object` por operação, `z.string().uuid()` para ids e `z.nativeEnum(OrderStatus)` para os status assinados, exatamente como na linha 26 daquele arquivo.

A validação de TLS obrigatório vive aqui — Sofia (`[09:23]`): *"é só uma validação no schema Zod"*:

```ts
url: z.string().url().max(2048)
  .refine((u) => u.startsWith('https://'), { message: 'Webhook URL must use https' }),
```

### 11.10 `prisma/schema.prisma` — novos modelos

Os quatro modelos da seção 4 entram no arquivo existente. `WebhookEndpoint` referencia `Customer` (linha 40), que ganha a relação inversa `webhooks WebhookEndpoint[]`. O enum `OrderStatus` (linha 16) é reusado em `fromStatus`/`toStatus`. Nenhum modelo existente muda de forma; a migration é puramente aditiva.

### 11.11 `src/shared/http/response.ts` — paginação

`GET /webhooks` e `GET /webhooks/:id/deliveries` usam o helper `paginated()` (linha 22), produzindo o mesmo envelope `{ data, pagination }` que `OrderService.list` retorna hoje (`src/modules/orders/order.service.ts:41`).

### 11.12 `tests/` — testes de integração

`tests/orders.test.ts` e `tests/helpers/factories.ts` já estabelecem o padrão de teste de integração com Vitest e Supertest (`vitest.config.ts`). Os testes novos ficam em `tests/webhooks.test.ts`, com as factories estendidas para criar endpoints de webhook.

## 12. Dependências e compatibilidade

### Dependências

**Nenhuma nova em `package.json`.** Decisão de [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md).

| Necessidade | Solução | Situação |
| --- | --- | --- |
| Geração de UUID | `uuid` 11.0.3 | Já em `dependencies`, usado em `src/middlewares/request-logger.middleware.ts:2` |
| HMAC-SHA256 | `node:crypto` | Nativo (`createHmac`) |
| Geração de secret | `node:crypto` | Nativo (`randomBytes`) |
| Cliente HTTP com timeout | `fetch` + `AbortSignal.timeout(10_000)` | Nativo no Node ≥ 20; `engines` exige `>=20` (`package.json:7`) |
| Comparação em tempo constante | `node:crypto` | Nativo (`timingSafeEqual`) |
| Validação | `zod` 3.23.8 | Já em `dependencies` |
| Log | `pino` 9.5.0 | Já em `dependencies` |

### Configuração

`src/config/env.ts` valida o ambiente com Zod e **encerra o processo se faltar variável** (linha 22). As novas entram no mesmo schema, todas com default para não quebrar deploys existentes:

```ts
WEBHOOK_POLL_INTERVAL_MS:  z.coerce.number().int().positive().default(2000),
WEBHOOK_BATCH_SIZE:        z.coerce.number().int().positive().default(20),
WEBHOOK_HTTP_TIMEOUT_MS:   z.coerce.number().int().positive().default(10000),
WEBHOOK_MAX_ATTEMPTS:      z.coerce.number().int().positive().default(5),
WEBHOOK_MAX_PAYLOAD_BYTES: z.coerce.number().int().positive().default(65536),
```

Os defaults reproduzem os valores decididos na reunião. `DATABASE_URL` é compartilhada entre API e worker (`[09:30] Bruno`).

### Compatibilidade

- **Retrocompatível.** Nenhum contrato existente muda. Clientes que continuarem fazendo polling em `GET /orders` seguem funcionando.
- **Migration aditiva.** Sem alteração de coluna existente, sem backfill, sem downtime.
- **Deploy independente.** API e worker sobem separadamente. Se o worker não subir, eventos se acumulam como `PENDING` e são entregues quando ele voltar — nada se perde.
- **Ordem de deploy:** migration → API → worker. Se o worker subir antes da API, ele simplesmente não encontra eventos.
- **Rollback:** desligar o worker interrompe as entregas sem afetar a API. Reverter a alteração em `order.service.ts` interrompe a geração de eventos. As tabelas podem permanecer vazias sem impacto.

## 13. Critérios de aceite técnicos

| # | Critério | Como verificar |
| --- | --- | --- |
| CAT-01 | Erro na inserção da outbox faz rollback da mudança de status | Teste forçando falha em `publishWebhookEvent`; asserção de que `orders.status` e `order_status_history` não mudaram |
| CAT-02 | Commit bem-sucedido sempre produz uma linha de outbox por endpoint interessado | Teste com 2 endpoints assinando `SHIPPED` e 1 assinando só `DELIVERED`: transição para `SHIPPED` gera exatamente 2 linhas |
| CAT-03 | Nenhuma linha é criada quando nenhum endpoint assina o status | Teste com endpoint assinando só `DELIVERED` e transição `PENDING → PAID` |
| CAT-04 | O worker entrega em ≤ 2s do commit no caminho feliz | Teste de integração com servidor HTTP local, medindo `createdAt → attemptedAt` |
| CAT-05 | Falha agenda a próxima tentativa conforme a tabela de backoff | Teste unitário do cálculo: tentativas 1–5 produzem +1m, +5m, +30m, +2h, +12h |
| CAT-06 | 5ª falha move o evento para `webhook_dead_letter` e não gera 6ª tentativa | Teste com servidor sempre retornando 500 |
| CAT-07 | `X-Signature` confere com HMAC-SHA256 do corpo bruto usando a secret do endpoint | Teste comparando com assinatura calculada independentemente |
| CAT-08 | Rotação mantém a secret anterior por 24h e depois a expurga | Teste com relógio controlado, verificando `previousSecretExpiresAt` |
| CAT-09 | URL `http://` é recusada com `WEBHOOK_INVALID_URL` / 400 | Teste de contrato via Supertest |
| CAT-10 | Payload acima de 64KB gera `WEBHOOK_PAYLOAD_TOO_LARGE` e não é enviado | Teste com payload artificialmente inflado |
| CAT-11 | Replay de DLQ com role `OPERATOR` retorna 403 `FORBIDDEN` | Teste de contrato com JWT de operador |
| CAT-12 | Replay preserva o `X-Event-Id` original | Teste comparando o `eventId` da DLQ com o da nova linha de outbox |
| CAT-13 | Replay grava `replayedById` com o usuário autenticado | Asserção sobre a linha de DLQ pós-replay |
| CAT-14 | Todo erro `WEBHOOK_*` retorna o envelope `{ error: { code, message } }` | Testes de contrato cobrindo cada código da seção 8 |
| CAT-15 | `secret` nunca aparece em `GET /webhooks`, `PATCH` ou em log | Asserção sobre corpo de resposta e sobre a saída do logger com `redactPaths` estendido |
| CAT-16 | Sequência rápida de transições no mesmo pedido é entregue em ordem | Teste com `PAID → PROCESSING → SHIPPED` e verificação da ordem de recebimento |
| CAT-17 | Eventos presos em `PROCESSING` voltam a `PENDING` no restart do worker | Teste inserindo linha `PROCESSING` antiga e iniciando o worker |
| CAT-18 | O payload entregue é idêntico ao gravado na outbox, em todas as tentativas | Comparação byte a byte entre `webhook_outbox.payload` e o corpo recebido |

## 14. Riscos e mitigação

| # | Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| RT-01 | A escrita extra degrada a transação de `changeStatus`, o caminho mais crítico do sistema | Média | Alto | Filtro na inserção evita escrita quando ninguém assina (`[09:34] Bruno`); snapshot é CPU, não I/O; medir p95 de `PATCH /orders/:id/status` antes e depois (OT-03) |
| RT-02 | Secret vaza em log ao serializar o endpoint | Média | Alto | Estender `redactPaths` em `src/shared/logger/index.ts` (seção 11.5); nunca logar o objeto do endpoint inteiro; revisão de Sofia antes do deploy (`[09:46]`) |
| RT-03 | Um cliente lento monopoliza o worker e atrasa os demais | Média | Médio | Timeout de 10s limita o dano por evento; instrumentar `webhook_delivery_duration_seconds` por endpoint; relacionado a Q1 do [RFC](RFC.md#5-questões-em-aberto) |
| RT-04 | Worker cai e ninguém percebe até o cliente reclamar | Média | Alto | `webhook_outbox_oldest_pending_age_seconds` com alarme acima de 60s (seção 10) |
| RT-05 | Serialização não determinística quebra a assinatura entre tentativas | Baixa | Alto | Armazenar e enviar a mesma string; CAT-18 cobre |
| RT-06 | Outbox cresce sem limite e degrada a query do worker | Alta (longo prazo) | Médio | Índice `(status, nextAttemptAt)`; retenção é Q3 do [RFC](RFC.md#5-questões-em-aberto), com dono e prazo |
| RT-07 | Cliente não implementa dedup e processa eventos duplicados | Média | Médio | Documentação destacada no portal, assumida por Marcos (`[09:26]`); `X-Event-Id` em header e no corpo |
| RT-08 | Revisão de segurança atrasa o deploy no fim do cronograma | Média | Médio | Reservar os dois dias úteis de Sofia dentro da terceira sprint (`[09:46]`), não depois dela |
| RT-09 | Evento fica preso em `PROCESSING` após crash do worker | Baixa | Médio | Recuperação no startup (seção 5.2), segura porque a entrega é at-least-once; CAT-17 cobre |
