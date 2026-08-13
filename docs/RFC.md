# RFC — Sistema de Webhooks de Notificação de Pedidos

| | |
| --- | --- |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-08-13 |
| **Revisores** | Bruno (Eng. Pedidos), Diego (Eng. Plataforma), Sofia (Eng. Segurança), Marcos (PM) |
| **Fonte das decisões** | Reunião técnica de quinta-feira, 09:00 — [`TRANSCRICAO.md`](../TRANSCRICAO.md) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

> Larissa, no fechamento da call (`[09:50]`): *"Eu vou abrir o doc de design da feature e marcar uma sessão pro Bruno e o Diego revisarem comigo antes da gente começar a codar."* Este é esse documento.

---

## 1. TL;DR

Propomos entregar notificações outbound de mudança de status de pedido usando o **padrão Outbox no MySQL que já operamos**, sem introduzir nenhuma infraestrutura nova.

Quando `OrderService.changeStatus` commita, uma linha de evento é gravada **na mesma transação**. Um **processo Node separado** (`npm run worker`) lê essa tabela a cada 2 segundos e faz o `POST` no endpoint do cliente, assinado com **HMAC-SHA256** usando uma secret exclusiva daquele endpoint. Falhas são retentadas 5 vezes com backoff exponencial (1m/5m/30m/2h/12h) e, esgotadas as tentativas, o evento vai para uma **dead letter queue** com replay manual restrito a `ADMIN`.

A garantia é **at-least-once**: o cliente pode receber o mesmo evento duas vezes e deduplica pelo `X-Event-Id`.

O módulo entra como `src/modules/webhooks/`, seguindo os mesmos padrões dos módulos existentes, **sem nenhuma dependência nova no `package.json`**. Estimativa: **três sprints**, incluindo a revisão de segurança (`[09:46] Larissa`).

## 2. Contexto e problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente para ser notificados quando o status dos pedidos deles muda (`[09:00] Marcos`). Hoje eles fazem polling no `GET /orders`, o que Marcos descreveu como *"deixando a integração lenta e cara pra eles"*. A Atlas sinalizou risco de churn caso a entrega não saia até o fim do trimestre.

O sistema atual **não tem nenhum mecanismo de notificação externa, evento, fila ou webhook**. Esta é a primeira integração de saída do produto.

Dois parâmetros delimitam o desenho:

- **"Tempo real" aqui significa menos de 10 segundos.** Marcos apurou com os clientes (`[09:02]`): *"qualquer coisa abaixo de 10 segundos já é 'tempo real'"*. Não é um requisito de milissegundos.
- **A direção é apenas de saída.** Sofia confirmou o escopo antes de entrar em arquitetura (`[09:02]`) e Marcos respondeu: *"Só saindo da gente pra eles"* — o que elimina toda a complexidade de webhooks de entrada.

O ponto de acoplamento é `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126`), hoje uma transação que já faz três escritas: `update` da order, `create` no histórico e ajuste de `stockQuantity` dos produtos. O problema técnico central é **como acoplar a notificação a essa transação sem acoplar sua disponibilidade à de terceiros**.

## 3. Proposta técnica

### 3.1 Visão geral do fluxo

```
  API (src/server.ts)                          Worker (src/worker.ts)
  ───────────────────                          ──────────────────────
  PATCH /orders/:id/status
        │
        ▼
  changeStatus  ──── transação única ────┐
    · update order                        │
    · insert order_status_history         │      loop a cada 2s
    · debit/replenish stock               │            │
    · INSERT webhook_outbox (pendente) ───┘            ▼
                                              SELECT pendentes vencidos
                                              ordenados por created_at
                                                       │
                                                       ▼
                                              POST na URL do cliente
                                              (HMAC-SHA256, timeout 10s)
                                                   ╱        ╲
                                             sucesso        falha
                                                │              │
                                          marca entregue   agenda retry
                                                            (5 tentativas)
                                                                │
                                                          esgotou → DLQ
```

### 3.2 Os cinco pilares

**Outbox transacional no MySQL** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)). O evento é gravado dentro da mesma transação da mudança de status. Se a transação commitou, o evento existe; se deu rollback, o evento sumiu junto. Bruno colocou a regra de forma inegociável (`[09:40]`): *"Se a outbox falhar de inserir, rollback. Não pode ter caso de status mudar e evento não sair."* A integração no service se dá por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da transação corrente, em vez de injetar um repository inteiro no `OrderService` (`[09:41] Bruno` e Diego: *"função pura recebendo o tx"*).

**Worker separado em polling** ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)). Processo Node próprio, entrypoint `src/worker.ts` espelhando `src/server.ts`, com seu próprio `PrismaClient` apontando para o mesmo `DATABASE_URL`. Ciclo de 2 segundos, batch pequeno, ordenado por `created_at`. Single-worker, o que preserva ordering por `order_id` no caminho feliz.

**Resiliência** ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)). Timeout de 10 segundos por chamada; 5 tentativas com backoff 1m → 5m → 30m → 2h → 12h (≈15h de janela total); depois, `webhook_dead_letter` com payload, motivo e timestamp. Replay manual via `POST /admin/webhooks/dead-letter/:id/replay`, exigindo role `ADMIN` e registrando quem executou.

**Segurança** ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)). HMAC-SHA256 sobre o corpo, transportado em `X-Signature`. Secret única por endpoint, gerada por nós e devolvida ao cliente na criação. Rotação pela API com grace period de 24h. URL obrigatoriamente `https`, validado no schema Zod. Payload limitado a 64KB — acima disso, erro, não truncamento (`[09:23] Sofia`).

**Semântica de entrega** ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)). At-least-once. Cada evento carrega um UUID estável em `X-Event-Id`, gerado na inserção na outbox e preservado por todas as tentativas e replays. A deduplicação é do cliente. O payload é um **snapshot** renderizado no momento da inserção ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md)), não uma consulta ao estado atual do pedido.

### 3.3 Superfície de API

Sete endpoints, todos autenticados. CRUD de configuração (`POST`, `GET`, `PATCH`, `DELETE` em `/webhooks`), rotação de secret, histórico de entregas (`GET /webhooks/:id/deliveries`, pedido por Marcos em `[09:34]`) e o replay administrativo. Contratos, payloads e status codes estão no [FDD](FDD.md#6-contratos-públicos) — este RFC não os duplica.

Um ponto de escopo que gerou discussão: o `customer_id` **não** vem do JWT. Bruno levantou que o JWT atual é do usuário operador, não do cliente (`[09:32]`), e Larissa concluiu (`[09:32]`): *"é endpoint autenticado normal, e o customer_id é passado no body ou no path. Não vem do JWT."*

### 3.4 Encaixe no projeto existente

O módulo segue integralmente os padrões vigentes ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)): estrutura `controller/service/repository/routes/schemas` como em `src/modules/orders/`, erros herdando de `AppError` com prefixo `WEBHOOK_`, `errorMiddleware` sem alteração, `requireRole` reaproveitado, logger Pino sem substituto, UUID como chave primária. Larissa fechou o bloco como *"reuso máximo do que já existe"* (`[09:30]`).

## 4. Alternativas consideradas

| Alternativa | Por que foi levantada | Trade-off que motivou o descarte |
| --- | --- | --- |
| **Disparo síncrono no `changeStatus`** (`[09:03]`–`[09:06]`) | Caminho mais simples: chamar o webhook direto no service | Acopla a disponibilidade da nossa operação de pedidos à infra de terceiros. Bruno (`[09:04]`): *"qualquer cliente lento vai travar mudança de status pra outros pedidos"* — e cliente fora do ar não tem tratamento possível sem rollback de uma operação legítima. Diego (`[09:06]`): *"Síncrono está fora de questão."* |
| **Broker externo (Redis Streams)** (`[09:07]`) | Escalabilidade e desacoplamento reais, sem polling | Custo operacional desproporcional para o tamanho do time. Diego: *"Subir Redis Cluster pra isso é overengineering."* Além disso, publicar no Redis não participaria da transação SQL, reintroduzindo o dual-write que a outbox elimina. |
| **Trigger de banco para notificar o worker** (`[09:09]`) | Reatividade em vez de polling, latência quase zero | MySQL não tem `NOTIFY/LISTEN`; a trigger só executa SQL e não avisa processo externo. Diego: *"a gente teria que improvisar algo tipo escrever em arquivo ou bater num endpoint, fica esquisito."* Polling de 2s já cabe folgado no requisito de 10s. |
| **Exactly-once** (`[09:25]`) | Elimina a necessidade de o cliente deduplicar | Exigiria protocolo de confirmação dos dois lados, elevando muito a barreira de integração. Diego: *"At-least-once com event_id resolve 99% dos casos. Stripe faz assim, GitHub faz assim."* |
| **Retry com 3 tentativas** (`[09:16]`) | Proposta de Bruno, falha detectada mais cedo | Janela curta demais para indisponibilidade real de cliente. Diego: *"Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada"* — 3 tentativas matariam o evento em ~30 minutos. |
| **DLQ como flag `failed` na própria outbox** (`[09:17]`) | Uma tabela e uma migration a menos | Polui a leitura da tabela operacional com linhas permanentemente mortas. Diego preferiu tabela dedicada: *"Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento."* |

## 5. Questões em aberto

Cinco pontos foram levantados na reunião e **deliberadamente não decididos**. Precisam de posição antes ou durante a implementação.

**Q1 — Rate limiting de saída.** Diego (`[09:38]`): *"Se o cliente tem 50 pedidos mudando de status em um minuto, a gente bombardeia ele com 50 chamadas?"* Ele mesmo propôs não resolver agora (`[09:39]`): *"A gente observa e implementa se virar problema. Mas vale registrar como ponto em aberto."* Larissa registrou como *"observar e decidir depois"*. **Encaminhamento sugerido:** instrumentar a taxa de envio por endpoint desde o dia 1 para que a decisão tenha dado quando vier.

**Q2 — Escala para múltiplos workers e ordering.** Single-worker é um teto de throughput e um ponto único de falha. Diego apontou os caminhos (`[09:13]`) — *"dá pra particionar por order_id, ou usar lock pessimista"* — mas classificou como *"problema do futuro, não agora"*. A limitação de ordering foi aceita e documentada (`[09:13] Larissa`): garantia apenas por `order_id` e apenas enquanto for single-worker. **Encaminhamento sugerido:** definir o gatilho quantitativo que reabre essa discussão.

**Q3 — Retenção e arquivamento da outbox.** Diego mencionou arquivar linhas entregues após ~30 dias, mas colocou explicitamente **fora do escopo desta feature** (`[09:08]`). Com a decisão de snapshot ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md)), cada linha carrega o JSON completo, o que acelera o crescimento. **Encaminhamento sugerido:** dívida técnica com dono e prazo, não um "depois" indefinido.

**Q4 — Onde trafega o `customer_id`: body ou path.** Larissa deixou a alternativa aberta na própria frase que resolveu a questão do JWT (`[09:32]`): *"o customer_id é passado no body ou no path"*. **Encaminhamento sugerido:** decidir na revisão deste RFC; o [FDD](FDD.md) assume `customerId` no body na criação e como query param na listagem, e essa premissa precisa ser ratificada.

**Q5 — Endurecimento da autorização do CRUD.** Marcos perguntou se o CRUD de configuração pode ser feito por qualquer role autenticada e Sofia respondeu (`[09:37]`): *"Por enquanto sim. Mais pra frente a gente pode endurecer."* Só o replay de DLQ exige `ADMIN`. **Encaminhamento sugerido:** reavaliar após o onboarding dos três primeiros clientes, dado que qualquer `OPERATOR` pode hoje alterar a URL de destino de webhooks de qualquer customer.

## 6. Impacto e riscos

### Impacto no sistema existente

| Área | Impacto |
| --- | --- |
| `src/modules/orders/order.service.ts` | Alteração no caminho mais crítico do sistema: `changeStatus` ganha uma leitura (quais webhooks escutam este status) e um insert na mesma transação |
| `prisma/schema.prisma` | Três modelos novos (`webhook_endpoints`, `webhook_outbox`, `webhook_dead_letter`) e uma tabela de histórico de entregas; nenhuma alteração nos modelos existentes |
| `src/shared/logger/index.ts` | Única alteração obrigatória em arquivo compartilhado: estender `redactPaths` para cobrir secrets de webhook |
| Deploy | Um artefato novo — o processo do worker, com seu próprio monitoramento e alarme |
| `package.json` | Nenhuma dependência nova; um script `worker` |

### Riscos principais

| Risco | Severidade | Mitigação |
| --- | --- | --- |
| Degradação da transação de `changeStatus`, que já faz três escritas acopladas | Alta — afeta a operação central do produto | Filtrar na inserção: se nenhum webhook do customer escuta aquele status, nem insere (`[09:34] Bruno`). Snapshot é serialização em memória, não I/O adicional |
| Vazamento de secrets em log | Alta | `redactPaths` estendido; revisão de segurança dedicada de Sofia antes do deploy (`[09:46]`) |
| Cliente sem dedup processa eventos duplicados | Média | Documentação destacada no portal do desenvolvedor, assumida por Marcos (`[09:26]`) |
| Crescimento não controlado da outbox | Média — degrada com o tempo, não no dia 1 | Índices em status e `created_at` (`[09:08] Diego`); resolução depende de **Q3** |
| Worker cai e ninguém percebe | Média | Métricas de idade do evento pendente mais antigo e alarme sobre elas (ver [FDD](FDD.md#9-observabilidade)) |
| Revisão de segurança no caminho crítico do cronograma | Média | Sofia pediu **dois dias úteis** reservados antes do deploy (`[09:46]`); precisa entrar no planejamento da terceira sprint, não depois dela |

### Cronograma

Larissa estimou **três sprints**, com a revisão de Sofia incluída (`[09:46]`): modelagem de outbox e DLQ (1 sprint), worker e retry (1 sprint), CRUD e deliveries (½), integração no `order.service` e testes ponta a ponta (½), mais HMAC, schemas e validações. O compromisso comercial com a Atlas é **fim de novembro** (`[09:45] Marcos`).

## 7. Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL para publicação de eventos de pedido |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, lendo a outbox por polling de 2 segundos |
| [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela dedicada |
| [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) | Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h |
| [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) | Entrega at-least-once com idempotência delegada ao cliente via `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso máximo dos padrões existentes do projeto no módulo de webhooks |
| [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md) | Payload renderizado como snapshot no momento da inserção na outbox |

## 8. O que se pede aos revisores

1. **Bruno e Diego:** validar a assinatura de `publishWebhookEvent(tx, order, fromStatus, toStatus)` e o impacto real na transação de `changeStatus`.
2. **Sofia:** confirmar que HMAC-SHA256 com grace period de 24h e o teto de 64KB cobrem o que você precisa, e agendar os dois dias de revisão de código antes do deploy.
3. **Marcos:** confirmar que o histórico de entregas e a limitação de ordering atendem ao que foi prometido aos três clientes.
4. **Todos:** posicionamento sobre **Q4** (body vs. path), que bloqueia o contrato dos endpoints no FDD.
