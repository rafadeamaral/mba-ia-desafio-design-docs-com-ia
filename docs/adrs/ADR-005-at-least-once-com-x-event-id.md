# ADR-005 — Entrega at-least-once com idempotência delegada ao cliente via `X-Event-Id`

- **Status:** Aceito
- **Data:** 2026-08-13
- **Decisores:** Diego (Eng. Plataforma), Larissa (Tech Lead), Sofia (Eng. Segurança), Marcos (PM)
- **Origem:** Reunião técnica `TRANSCRICAO.md`, blocos `[09:24]` a `[09:26]`, `[09:44]` e `[09:51]`
- **ADRs relacionados:** [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)

---

## Contexto

A combinação de outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) com retry ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)) produz inevitavelmente entregas duplicadas. O caso clássico: o worker envia, o cliente processa com sucesso, mas a resposta se perde ou estoura o timeout de 10 segundos — do nosso lado aquilo é falha, e o evento entra em retry. O cliente recebe o mesmo evento duas vezes.

Diego trouxe isso à mesa antes que virasse bug em produção (`[09:24]`): *"a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado."*

Bruno fez a pergunta operacional que define a decisão (`[09:25]`): *"E como ele diferencia?"*

## Decisão

**A garantia de entrega é at-least-once, e a deduplicação é responsabilidade do cliente, viabilizada por um identificador único de evento transmitido no header `X-Event-Id`.**

- O identificador é um **UUID gerado no momento em que o evento entra na outbox** (`[09:25] Diego`), o que o torna estável ao longo de todas as tentativas de entrega daquele evento — inclusive após um replay de DLQ. Ele é a própria chave primária da linha da outbox, seguindo o padrão UUID do projeto (`[09:51] Larissa`).
- O valor é enviado tanto no header `X-Event-Id` quanto no campo `event_id` do corpo JSON (`[09:43]` e `[09:44] Diego`).
- Junto vai o **`X-Webhook-Id`**, com o id do cadastro de webhook, sugerido por Sofia (`[09:44]`): *"pra cliente que tem vários conseguir saber qual cadastro caiu naquele envio."*
- A obrigação do cliente — guardar os `event_id` já processados e descartar repetidos — precisa ser documentada de forma destacada no portal do desenvolvedor. Marcos assumiu (`[09:26]`): *"Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes, sem problema."*

## Alternativas Consideradas

### 1. Exactly-once

Garantir que cada evento chegue exatamente uma vez ao cliente.

**Descartada por custo de coordenação.** Diego (`[09:25]`): *"Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos."*

O trade-off: exactly-once real exigiria um protocolo de confirmação de dois lados — o cliente precisaria expor algum mecanismo de ack idempotente e nós precisaríamos manter estado de confirmação por evento e por endpoint. Isso transformaria uma integração de "receba um POST" em um protocolo proprietário, elevando drasticamente a barreira de adoção para os três clientes B2B que motivaram a feature. Na prática, o problema não seria eliminado, apenas movido: mesmo com ack, uma falha na entrega do ack reintroduz a ambiguidade.

Diego ancorou a escolha no mercado (`[09:25]`): *"é o padrão de mercado. Stripe faz assim, GitHub faz assim."*

### 2. Deduplicação do nosso lado, antes do envio

Manter uma tabela de eventos já entregues por endpoint e checar antes de cada envio.

**Não foi levantada na reunião**; registrada como alternativa plausível e não adotada. Ela não resolve o problema real: o caso duplicado nasce justamente quando *não sabemos* se a entrega chegou (timeout, resposta perdida). Uma checagem nossa só evitaria reenvios que já sabemos ter sido bem-sucedidos — e esses já não seriam reenviados de qualquer forma, já que o evento seria marcado como entregue. Adicionaria estado e uma query por envio sem eliminar nenhuma duplicata real.

## Consequências

### Positivas

- **Nenhum evento é perdido.** Na dúvida, reenviamos. Essa é a propriedade que os clientes efetivamente precisam: saber que o pedido mudou, mesmo que fiquem sabendo duas vezes.
- **Contrato simples e familiar.** Clientes que já integram com Stripe ou GitHub reconhecem o padrão e provavelmente já têm código de dedup (`[09:25] Diego`).
- **O `event_id` é estável entre tentativas**, o que faz a dedup funcionar também no cenário de replay manual de DLQ ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)) — o evento reprocessado carrega o mesmo identificador do original.
- **`X-Event-Id` é chave de correlação também para nós.** O mesmo identificador aparece na outbox, no log estruturado do worker e no histórico de entregas, tornando o rastreio de uma entrega ponta a ponta trivial durante o suporte.
- **`X-Webhook-Id` desambigua múltiplos cadastros do mesmo cliente** (`[09:44] Sofia`).

### Negativas

- **Transferimos complexidade para o cliente.** Sofia apontou isso na hora (`[09:25]`): *"Isso joga responsabilidade pro cliente."* Diego reconheceu — *"Joga, mas é o padrão de mercado"* — e a mitigação acordada é documentação, não código.
- **Cliente que não implementar dedup vai processar eventos repetidos**, com consequências que podem ser sérias no domínio dele (ex.: disparar duas vezes uma rotina de expedição). Não temos como detectar nem prevenir isso do nosso lado.
- **Não há garantia de exatidão de entrega para conciliação.** O histórico de entregas (`GET /webhooks/:id/deliveries`, `[09:34] Marcos`) mostra o que *tentamos* enviar e a resposta que recebemos, não o que o cliente efetivamente processou.
- **A responsabilidade precisa ser comunicada antes do onboarding**, não depois do primeiro incidente. Isso cria uma dependência de documentação de produto no caminho crítico da entrega da feature.
