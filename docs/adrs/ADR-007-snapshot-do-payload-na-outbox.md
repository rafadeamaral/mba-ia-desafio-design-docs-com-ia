# ADR-007 — Payload renderizado como snapshot no momento da inserção na outbox

- **Status:** Aceito
- **Data:** 2026-08-13
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pedidos), Diego (Eng. Plataforma)
- **Origem:** Reunião técnica `TRANSCRICAO.md`, blocos `[09:43]`, `[09:51]` e `[09:52]`
- **ADRs relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)

---

## Contexto

Esta decisão foi tomada no fechamento da call, depois que PM e Segurança já haviam saído. Bruno colocou a última dúvida (`[09:51]`): *"o evento da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?"*

A pergunta é sobre **quando** o conteúdo do evento é materializado, e existe uma janela real entre os dois momentos: o evento é inserido no commit da transação de `changeStatus`, mas só é enviado quando o worker o pega no ciclo de polling seguinte ([ADR-002](ADR-002-worker-separado-em-polling.md)) — e, em cenário de retry, isso pode ser até ~15 horas depois ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)).

Nesse intervalo o pedido pode ter mudado de novo. A máquina de estados em `src/modules/orders/order.status.ts:3` permite sequências rápidas como `PENDING → PAID → PROCESSING → SHIPPED`, e Larissa já havia citado esse cenário no bloco de ordering (`[09:12]`).

O conteúdo do evento foi definido por Diego (`[09:43]`): `event_id`, `event_type` (`order.status_changed`), timestamp ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents`. Deliberadamente **sem os itens do pedido**: *"Não manda items pra não inflar. Se o cliente quiser detalhes, ele bate no GET /orders/:id depois"* (`[09:43] Diego`).

## Decisão

**O payload JSON completo é renderizado no momento da inserção na outbox, dentro da transação de `changeStatus`, e persistido na linha. O worker envia exatamente o que está gravado, sem reconsultar o pedido.**

Larissa decidiu (`[09:52]`): *"Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito."* Diego concordou — *"snapshot na inserção"* — e Bruno fechou: *"Beleza, snapshot. Decidido."*

O evento passa a ser um **registro imutável de um fato histórico**: "o pedido X foi de `from_status` para `to_status` neste instante", com o estado que o pedido tinha naquele instante. Não é uma consulta ao estado atual.

## Alternativas Consideradas

### 1. Guardar apenas `order_id` e renderizar no momento do envio

A alternativa que Bruno explicitou na pergunta (`[09:51]`). A outbox guardaria referência mínima (`order_id`, `from_status`, `to_status`) e o worker faria um `findUnique` da order antes de cada envio, montando o JSON na hora.

**Descartada.** O problema é que o payload deixaria de corresponder ao evento que ele afirma representar. Um pedido que foi `PAID` às 10:00 e `SHIPPED` às 10:05, com o evento de `PAID` em retry por indisponibilidade do cliente, seria entregue às 22:00 dizendo `to_status: PAID` mas carregando `total_cents` e demais campos do estado atual. Larissa chamou isso de *"caso esquisito"* (`[09:52]`); em termos de contrato, é um evento internamente inconsistente.

O trade-off descartado é real: renderizar na hora economizaria armazenamento (a outbox guardaria dezenas de bytes em vez do JSON inteiro) e permitiria corrigir o formato do payload retroativamente para eventos ainda não entregues. Perdeu para a correção semântica.

### 2. Snapshot com os itens do pedido incluídos

Materializar o payload completo, incluindo o array de `items` com produtos e quantidades.

**Descartada no mesmo bloco em que o formato foi definido** (`[09:43] Diego`): *"Não manda items pra não inflar."* Um pedido com muitos itens produziria payloads grandes, multiplicados por cada webhook cadastrado e por cada tentativa de retry — e há um teto rígido de 64KB por payload, acima do qual o envio é recusado com erro (`[09:23]` Sofia, `[09:24]` Diego e Larissa). Bruno endossou: *"mantém payload enxuto"* (`[09:44]`). O caminho para quem precisa de detalhe é o `GET /orders/:id`.

## Consequências

### Positivas

- **O evento é semanticamente correto em qualquer momento da entrega.** Um evento de `PAID` entregue 15 horas depois ainda descreve o pedido como ele era quando virou `PAID`.
- **Retry e replay são reproduzíveis.** Todas as tentativas de um mesmo evento enviam bytes idênticos, o que faz a assinatura HMAC ([ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)) ser estável e torna o histórico de entregas comparável entre tentativas.
- **O worker não depende das tabelas de domínio.** Ele lê a outbox e faz HTTP; não consulta `orders`, `order_items` ou `customers`. Isso reduz o acoplamento entre o processo de entrega e o schema do domínio, e elimina carga de leitura extra no banco a cada envio.
- **Nenhum risco de vazar estado futuro.** Um pedido cancelado depois não faz o evento antigo passar a expor `CANCELLED` a um cliente que jamais deveria ver aquele status por aquele evento.
- **O histórico de entregas pedido por Marcos (`[09:34]`) mostra o payload real enviado**, não uma reconstrução aproximada.

### Negativas

- **A outbox fica maior.** Cada linha carrega o JSON completo em vez de uma referência. Mitigado pela decisão de não incluir `items` (`[09:43] Diego`) e pelo teto de 64KB, mas agrava o problema de crescimento da tabela já registrado em [ADR-001](ADR-001-outbox-no-mysql.md) — e a política de arquivamento continua fora do escopo desta feature (`[09:08] Diego`).
- **O payload é imutável depois de gravado.** Um bug no formato do evento não pode ser corrigido para os eventos já enfileirados; a correção só vale para os próximos. Eventos malformados presos em retry precisariam ser tratados manualmente pela DLQ.
- **Duplicação de dados entre `orders` e `webhook_outbox`.** Os campos do pedido existem em dois lugares, com valores que podem divergir legitimamente ao longo do tempo. É intencional, mas exige que quem lê a outbox para debug entenda que aquilo é histórico, não estado atual.
- **A renderização acontece dentro da transação crítica de `changeStatus`** (`src/modules/orders/order.service.ts:131`), somando trabalho de serialização ao caminho quente. É trabalho de CPU, não de I/O, e ocorre uma vez por webhook interessado no status — mas está dentro da transação e conta para o tempo de lock.
