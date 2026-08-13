# ADR-003 — Retry com backoff exponencial (5 tentativas) e DLQ em tabela dedicada

- **Status:** Aceito
- **Data:** 2026-08-13
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos), Marcos (PM), Sofia (Eng. Segurança)
- **Origem:** Reunião técnica `TRANSCRICAO.md`, blocos `[09:14]` a `[09:19]`, `[09:35]` e `[09:36]`
- **ADRs relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

---

## Contexto

O endpoint de destino é infraestrutura de terceiros e vai falhar. Larissa colocou a pergunta assim: *"Vamos pra retry. Se o cliente tá offline, o que a gente faz?"* (`[09:14]`).

Duas subdecisões estavam abertas: **quantas vezes e com que espaçamento retentar**, e **o que fazer com o evento que esgotou as tentativas**.

O contexto empírico veio de Diego: já houve cliente com indisponibilidade de duas horas em manutenção planejada (`[09:16]`), o que descarta janelas de retry curtas.

## Decisão

### Política de retry

**5 tentativas, com backoff exponencial de 1 minuto → 5 minutos → 30 minutos → 2 horas → 12 horas** (`[09:17] Diego`, ratificado por Larissa em `[09:17]`).

A janela total entre a primeira falha e a última tentativa é de aproximadamente 15 horas. Marcos aceitou o número do ponto de vista de negócio (`[09:17]`): *"Se um cliente meu cair por 15 horas, ele já tá com problema sério dele. Acho aceitável."*

Uma tentativa é considerada falha quando o endpoint responde status HTTP de erro, quando a conexão falha, ou quando estoura o **timeout de 10 segundos** do HTTP call (`[09:42] Diego`).

### Dead Letter Queue

Esgotadas as 5 tentativas, o evento é movido para uma **tabela dedicada `webhook_dead_letter`**, contendo o payload, o motivo da falha e o timestamp (`[09:18] Diego`).

### Reprocessamento

O reprocessamento é **manual, via endpoint administrativo** `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente (`[09:18]` e `[09:35] Diego`).

O endpoint **exige role `ADMIN`**. Sofia foi explícita (`[09:36]`): *"Mexer em fila de entrega de notificação não é coisa de operador. E o endpoint de admin tem que logar quem fez o replay, pra auditoria."* Larissa fechou reaproveitando o que já existe: *"role ADMIN obrigatório no replay e a gente reaproveita o `requireRole` que já existe"* (`[09:36]`) — o middleware está em `src/middlewares/auth.middleware.ts:49` e já é usado assim em `src/modules/users/user.routes.ts:15`.

## Alternativas Consideradas

### 1. Três tentativas, mais agressivo

Bruno propôs: *"3 não é melhor? Mais agressivo."* (`[09:16]`).

**Descartada.** Diego rebateu com o caso concreto (`[09:16]`): *"3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria. Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada."* O trade-off — menos linhas retidas na outbox e falha detectada mais cedo — não compensa descartar eventos de um cliente que estava apenas em janela de manutenção.

### 2. Retry indefinido com backoff

Nunca desistir, apenas espaçar cada vez mais.

**Descartada.** Diego (`[09:15]`): *"Algumas pessoas defendem retry indefinido com backoff, mas isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu."* O trade-off é entregabilidade máxima versus uma outbox que nunca drena e um sinal de erro que nunca aparece. Cinco tentativas dão um ponto de corte claro em que o problema vira visível.

### 3. Marcar o evento como `failed` na própria outbox, sem tabela separada

Larissa levantou a opção diretamente (`[09:17]`): *"Faz numa tabela separada ou marca como 'failed' na própria outbox?"*

**Descartada.** Diego (`[09:18]`): *"Eu fazia uma tabela `webhook_dead_letter` separada, com a payload, motivo da falha e timestamp. Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento."* O trade-off é uma tabela e uma migration a mais em troca de manter a outbox operacional enxuta — o worker varre só o que ainda tem chance de ser entregue, e o índice de status não fica poluído por linhas mortas permanentes.

### 4. Reprocessamento automático da DLQ

Não foi proposta na reunião; registrada aqui como alternativa plausível e conscientemente não adotada. Reprocessar automaticamente reintroduz o problema do retry indefinido por outra porta e remove o momento de inspeção humana que justifica a existência da DLQ como *evidence* (`[09:18] Diego`). O replay é deliberadamente manual e auditado.

## Consequências

### Positivas

- **Cobre indisponibilidades reais de cliente**, inclusive manutenções planejadas de várias horas, sem intervenção nossa.
- **A outbox drena.** Todo evento tem um destino terminal — entregue ou DLQ —, então a tabela operacional não acumula lixo indefinidamente.
- **A DLQ é uma fila de trabalho legível.** Payload, motivo e timestamp em um só lugar dão ao time o suficiente para diagnosticar sem abrir log.
- **Recuperação sem deploy.** Corrigido o problema do lado do cliente, um `POST` de replay recoloca o evento em circulação.
- **Rastreabilidade da intervenção.** Exigir `ADMIN` e logar o autor do replay atende o requisito de auditoria de Sofia (`[09:36]`).

### Negativas

- **Um evento pode levar até ~15 horas para ser entregue.** É bastante distante da promessa de "tempo real" do caminho feliz. Aceito conscientemente por Marcos (`[09:17]`), mas precisa estar documentado no portal do desenvolvedor.
- **Retry combinado com at-least-once amplifica duplicatas.** Um cliente que processa a requisição mas falha ao responder (timeout, por exemplo) vai receber o mesmo evento de novo. É exatamente o cenário que [ADR-005](ADR-005-at-least-once-com-x-event-id.md) endereça com `X-Event-Id`.
- **Backoff longo com single-worker exige que o agendamento seja por evento, não por bloqueio do loop.** O worker não pode "dormir" esperando um retry de 12 horas; a próxima tentativa precisa ser um campo de data no registro, filtrado na query de leitura. Consequência de projeto herdada de [ADR-002](ADR-002-worker-separado-em-polling.md).
- **Replay manual não escala.** Se uma falha sistêmica nossa mandar centenas de eventos para a DLQ, o replay item a item vira gargalo operacional. Aceito por ora, dado o volume esperado de três clientes B2B.
- **Retry preserva a ordem de tentativa, não a ordem de entrega.** Se o evento A entra em backoff de 30 minutos e o evento B do mesmo pedido é entregue no minuto seguinte, o cliente recebe B antes de A. A garantia de ordering de [ADR-002](ADR-002-worker-separado-em-polling.md) vale para o caminho feliz; em cenário de retry, ela não se sustenta.
