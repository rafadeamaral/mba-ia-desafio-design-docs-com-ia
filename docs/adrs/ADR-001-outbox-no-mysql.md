# ADR-001 — Padrão Outbox no MySQL para publicação de eventos de pedido

- **Status:** Aceito
- **Data:** 2026-08-13
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos)
- **Origem:** Reunião técnica `TRANSCRICAO.md`, blocos `[09:03]` a `[09:08]`
- **ADRs relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-007](ADR-007-snapshot-do-payload-na-outbox.md)

---

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para ser notificados quando o status dos pedidos deles muda, em vez de ficarem fazendo polling no `GET /orders` (`[09:00] Marcos`).

A mudança de status hoje acontece em `OrderService.changeStatus`, dentro de uma única transação Prisma (`src/modules/orders/order.service.ts:131`) que executa três escritas acopladas:

1. `tx.order.update` — muda o status do pedido;
2. `tx.orderStatusHistory.create` — grava a auditoria da mudança;
3. `debitStock` / `replenishStock` — ajusta `stockQuantity` dos produtos do pedido.

Bruno resumiu o risco de acoplar uma chamada HTTP a essa transação: *"a transação de mudança de status hoje já é pesada (...) se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos"* (`[09:04]`). E, se o cliente estiver fora do ar, não existe resposta razoável: *"o que a gente faz, dá rollback na mudança de status? Não dá"* (`[09:04]`).

O problema central, portanto, não é "como mandar HTTP", e sim **como garantir que o evento e a mudança de status tenham o mesmo destino**: se o status mudou, o evento tem que existir; se a transação abortou, o evento não pode existir.

## Decisão

Adotamos o **padrão Outbox persistido no MySQL já existente**.

Na mesma transação de `changeStatus`, além do `update` da order e do `create` do histórico, inserimos uma linha em uma tabela `webhook_outbox` contendo o evento a ser entregue. Um processo separado lê essa tabela e faz as chamadas HTTP.

Como Diego descreveu (`[09:06]`): *"Garante que se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto. Não tem inconsistência possível."*

Consequências diretas desta decisão:

- **Nenhuma infraestrutura nova.** O broker é a tabela; o storage é o MySQL que já roda em produção via `datasource db { provider = "mysql" }` (`prisma/schema.prisma:5`).
- **A inserção na outbox é obrigatória dentro da transação.** Se o insert falhar, a transação inteira dá rollback e a mudança de status não acontece (`[09:40] Bruno`: *"Se a outbox falhar de inserir, rollback. Não pode ter caso de status mudar e evento não sair"*).
- **A tabela é indexada por status de processamento e por `created_at`**, para que o leitor busque apenas os pendentes mais antigos em batch (`[09:08] Diego`).
- **A chave primária é UUID**, seguindo o padrão de todos os modelos existentes — `@id @default(uuid()) @db.Char(36)` em `prisma/schema.prisma` (`[09:51] Larissa`: *"UUID, segue o padrão do resto do projeto. Tudo é uuid"*).

## Alternativas Consideradas

### 1. Disparo síncrono dentro de `changeStatus`

Chamar o webhook do cliente diretamente no meio da transação, logo após o `tx.order.update`.

**Descartada.** Acopla a latência e a disponibilidade da nossa mudança de status à infraestrutura de terceiros. Um cliente lento degrada a operação de pedidos para *todos* os outros; um cliente fora do ar não tem tratamento possível sem rollback de uma operação de negócio legítima. Diego foi categórico: *"Síncrono está fora de questão"* (`[09:06]`).

### 2. Broker externo (Redis Streams ou similar)

Publicar o evento em um Redis Streams e ter consumidores lendo de lá.

**Descartada por custo operacional desproporcional.** Larissa levantou a opção (`[09:07]`) e Diego respondeu: *"a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve"* (`[09:07]`). O trade-off é real — um broker dedicado escalaria melhor e dispensaria polling — mas o volume esperado (eventos de mudança de status de pedidos de três clientes B2B) não justifica introduzir um novo componente com seu próprio ciclo de vida, monitoramento e modo de falha. Além disso, publicar no Redis *não* participaria da transação SQL, reintroduzindo exatamente o problema de consistência dual-write que a outbox resolve.

## Consequências

### Positivas

- **Atomicidade real entre mudança de status e registro do evento.** Não existe estado em que o pedido mudou de status e o evento se perdeu, nem o inverso — é a propriedade que motivou a decisão.
- **Zero infraestrutura adicional.** Reaproveita o MySQL, o `PrismaClient` e as migrations que o time já opera.
- **A outbox vira trilha de auditoria de entrega**, útil para debug e para alimentar o endpoint de histórico de entregas pedido por Marcos (`[09:34]`).
- **Falha do processo entregador não perde eventos.** O estado vive no banco, não em memória.

### Negativas

- **Custo de escrita adicional na transação mais crítica do sistema.** Todo `changeStatus` passa a ter um `INSERT` a mais — e, com o filtro de eventos por webhook, uma leitura a mais para decidir se insere. Mitigação: só inserimos se algum webhook ativo do customer escutar aquele status (`[09:34] Bruno`: *"Se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela"*).
- **A tabela cresce indefinidamente sem uma política de retenção.** Diego previu arquivar linhas entregues após ~30 dias, mas declarou explicitamente que isso está **fora do escopo desta feature** (`[09:08]`). Fica como dívida conhecida e como ponto em aberto no RFC.
- **Não há push: a leitura é por polling**, o que impõe uma latência mínima. Essa consequência é tratada em [ADR-002](ADR-002-worker-separado-em-polling.md).
- **Sem broker, escalar horizontalmente exige trabalho manual** (particionamento por `order_id` ou lock pessimista), problema que o time conscientemente adiou (`[09:13] Diego`: *"isso é problema do futuro, não agora"*).
