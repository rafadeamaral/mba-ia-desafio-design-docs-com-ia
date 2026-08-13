# ADR-002 — Worker em processo separado, lendo a outbox por polling de 2 segundos

- **Status:** Aceito
- **Data:** 2026-08-13
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos), Marcos (PM)
- **Origem:** Reunião técnica `TRANSCRICAO.md`, blocos `[09:08]` a `[09:13]` e `[09:28]`
- **ADRs relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

---

## Contexto

Com a outbox decidida ([ADR-001](ADR-001-outbox-no-mysql.md)), sobra a pergunta que Larissa fez logo em seguida: *"como o worker lê isso?"* (`[09:08]`).

Duas restrições delimitam o espaço de solução:

1. **Latência.** Marcos apurou o requisito com os clientes: *"Pra eles, qualquer coisa abaixo de 10 segundos já é 'tempo real'"* (`[09:02]`). Não é um requisito de milissegundos.
2. **Banco.** O datasource é MySQL (`prisma/schema.prisma:6`), que não oferece o mecanismo de notificação assíncrona que tornaria uma abordagem reativa trivial.

Há ainda uma questão de topologia: onde esse leitor roda. A API hoje sobe por `src/server.ts`, que chama `app.listen` e registra handlers de `SIGINT`/`SIGTERM` para shutdown gracioso.

## Decisão

**O processamento da outbox roda em um processo Node separado da API, em loop de polling com intervalo de 2 segundos.**

Três partes:

- **Estratégia de leitura:** polling. *"A cada 2 segundos, busca os eventos pendentes mais antigos, processa, marca"* (`[09:09] Diego`). A busca é ordenada por `created_at` e limitada a um batch pequeno, usando os índices previstos em [ADR-001](ADR-001-outbox-no-mysql.md).
- **Topologia:** processo separado, não uma thread ou timer dentro da API. *"O worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker"* (`[09:11] Diego`).
- **Concorrência:** **um único worker**. Isso preserva ordering por `order_id` de graça, já que um leitor único consumindo por `created_at` entrega na ordem em que os eventos foram gerados (`[09:12] Diego`).

Sobre o encaixe no projeto, Larissa propôs e o time aceitou (`[09:11]`): uma nova entrypoint `src/worker.ts` espelhando o papel de `src/server.ts`, mais um script `npm run worker` no `package.json`. A lógica de processamento vive dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` (`[09:28] Bruno`).

O worker instancia **seu próprio `PrismaClient`**, apontando para o mesmo `DATABASE_URL`. Bruno justificou (`[09:30]`): *"PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node."* Isso é compatível com `src/config/database.ts`, que exporta tanto a factory `createPrismaClient()` quanto o singleton `prisma`.

## Alternativas Consideradas

### 1. Trigger no banco para notificar o worker

Bruno perguntou: *"Não dá pra usar trigger do banco pra ser mais reativo?"* (`[09:09]`).

**Descartada.** Diego explicou a limitação técnica (`[09:09]`): *"MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres. Trigger no banco a gente até tem, mas ela não notifica processo externo, ela só executa SQL. Pra avisar o worker, a gente teria que improvisar algo tipo escrever em arquivo ou bater num endpoint, fica esquisito."*

O trade-off: ganharíamos latência quase-zero em troca de um mecanismo de notificação improvisado, fora do Prisma, invisível para as migrations e difícil de testar. Como polling de 2s já cabe folgado no requisito de 10s (`[09:10] Marcos`: *"2 segundos serve, perfeito"*), o ganho não paga o custo.

### 2. Worker como timer dentro do processo da API

Rodar um `setInterval` dentro da mesma instância Express, evitando um segundo processo e um segundo deploy.

**Descartada.** Acopla o ciclo de vida da entrega ao da API: um restart, um deploy ou um crash do servidor HTTP derruba a entrega junto. Além disso, o processamento competiria por event loop com o tráfego de requisições. Diego vetou explicitamente (`[09:11]`).

## Consequências

### Positivas

- **Latência de entrega previsível e dentro do acordado.** Pior caso de ~2 segundos entre o commit da transação e o início do envio, contra os 10 segundos aceitos pelos clientes — margem confortável para o tempo de rede e o `timeout` de 10s do HTTP call (`[09:42] Diego`).
- **Isolamento de falha.** Deploy ou restart da API não interrompe a entrega, e vice-versa.
- **Ordering por `order_id` sai de graça** enquanto for single-worker: um pedido que vai `PAID → PROCESSING → SHIPPED` em sequência rápida chega ao cliente nessa ordem (`[09:12] Diego`).
- **Nenhuma dependência nova.** O loop é código Node comum, e a stack de conexão é a mesma da API.

### Negativas

- **Latência mínima não é zero.** Larissa registrou o custo explicitamente (`[09:10]`): *"A latência mínima vai ser 2 segundos no pior caso. Aceitamos."*
- **Polling gera carga de fundo constante no MySQL**, mesmo sem eventos — uma query a cada 2 segundos, 24/7. É barata por causa do índice de status, mas não é zero.
- **Single-worker é um ponto único de throughput e de disponibilidade.** Se o processo cair, a entrega para até ele voltar (os eventos não se perdem, ficam pendentes na outbox). Escalar para múltiplos workers quebra a garantia de ordering; Diego apontou os caminhos — *"dá pra particionar por order_id, ou usar lock pessimista"* — mas o time adiou (`[09:13]`).
- **A garantia de ordering é condicional e precisa ser documentada como limitação.** Larissa (`[09:13]`): *"Não é garantia de ordering global, só por order_id e enquanto for single-worker."* Marcos confirmou que isso não conflita com o pedido dos clientes (`[09:14]`).
- **Um artefato de deploy a mais.** O `npm run worker` precisa de processo, monitoramento e alarme próprios.
