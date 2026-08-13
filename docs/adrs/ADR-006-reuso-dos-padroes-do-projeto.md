# ADR-006 — Reuso máximo dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data:** 2026-08-13
- **Decisores:** Bruno (Eng. Pedidos), Larissa (Tech Lead), Diego (Eng. Plataforma)
- **Origem:** Reunião técnica `TRANSCRICAO.md`, blocos `[09:27]` a `[09:30]`, `[09:36]` e `[09:51]`
- **ADRs relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)

> Este é o ADR que ancora o pacote no código existente. Todos os caminhos citados abaixo foram verificados no repositório.

---

## Contexto

Larissa abriu o bloco pedindo a Bruno que descrevesse os padrões da codebase (`[09:27]`), e Bruno respondeu com o inventário (`[09:27]`): *"A gente tem um padrão claro na codebase. Cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas."*

A questão em jogo era se o módulo de webhooks — que traz elementos genuinamente novos ao sistema (um processo worker, chamadas HTTP de saída, material criptográfico) — justificaria abrir exceções aos padrões vigentes, ou se caberia inteiro dentro deles.

O sistema não tem hoje **nenhum** mecanismo de notificação externa, evento ou fila. É a primeira vez que o projeto fala com fora.

## Decisão

Larissa fechou o bloco assim (`[09:30]`): *"reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros."*

Em concreto, os pontos de reuso decididos:

| Padrão | Onde já existe | Como o módulo de webhooks adere |
| --- | --- | --- |
| Estrutura modular | `src/modules/orders/` (controller, service, repository, routes, schemas) | Novo `src/modules/webhooks/` com a mesma composição (`[09:27] Bruno`) |
| Hierarquia de erros | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` | Erros do módulo estendem `AppError` / `ConflictError` / `NotFoundError`, como `InvalidStatusTransitionError` e `InsufficientStockError` fazem hoje |
| Códigos de erro em SCREAMING_SNAKE | `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION` | Prefixo `WEBHOOK_` para todo o módulo: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` (`[09:28] Bruno`, `[09:29] Larissa`) |
| Tratamento centralizado de erro | `src/middlewares/error.middleware.ts` | Nenhuma mudança necessária — já trata `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError` (`[09:29] Bruno`) |
| Validação de entrada | `src/middlewares/validate.middleware.ts` + schemas Zod por módulo | `webhook.schemas.ts` no mesmo formato de `src/modules/orders/order.schemas.ts` |
| Autorização por role | `requireRole` em `src/middlewares/auth.middleware.ts:49`, usado em `src/modules/users/user.routes.ts:15` | `requireRole('ADMIN')` no endpoint de replay de DLQ (`[09:36] Larissa`) |
| Logging | Pino em `src/shared/logger/index.ts` | Mesmo logger, sem dependência nova (`[09:29] Bruno`) |
| Entrypoint de processo | `src/server.ts` | `src/worker.ts` espelhando a estrutura de bootstrap e shutdown (`[09:11] Larissa`) |
| Identificadores | `@id @default(uuid()) @db.Char(36)` em todos os modelos de `prisma/schema.prisma` | UUID nas novas tabelas (`[09:51] Larissa`) |

Bruno foi explícito quanto a não introduzir nada novo no eixo de observabilidade (`[09:29]`): *"o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo. O middleware de erro centralizado já trata AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada."*

**Exceção deliberada:** o worker instancia seu próprio `PrismaClient` em vez de importar o singleton `prisma` de `src/config/database.ts:10`. Diego levantou a questão (`[09:29]`) e Bruno respondeu (`[09:30]`): *"Separado. PrismaClient é por processo. Mesmo banco, mesma `DATABASE_URL`, mas instância nova porque é outro processo Node."* O arquivo já expõe a factory `createPrismaClient()` exatamente para esse tipo de uso, então nem essa exceção exige código novo de infraestrutura.

## Alternativas Consideradas

### 1. Tratar webhooks como subsistema com padrões próprios

Dado que o módulo introduz um processo separado, I/O de saída e criptografia — coisas que nenhum outro módulo faz —, seria defensável desenhá-lo como um subsistema à parte, com sua própria hierarquia de erros e seu próprio formato de log.

**Descartada.** O trade-off seria liberdade de desenho no módulo novo em troca de fragmentação permanente da codebase: dois formatos de resposta de erro, dois vocabulários de log, dois lugares para procurar quando algo quebra. O `errorMiddleware` (`src/middlewares/error.middleware.ts:14`) só produz a resposta padronizada `{ error: { code, message, details } }` para exceções que herdam de `AppError`; sair do padrão significaria ou duplicar esse middleware, ou entregar 500 genérico para erros de negócio do módulo. Larissa fechou pelo reuso sem que a alternativa chegasse a ser defendida por alguém.

### 2. Introduzir uma biblioteca de fila/jobs pronta

Usar algo como BullMQ ou Agenda para o loop de processamento e o agendamento de retry, em vez de escrever o loop.

**Não foi levantada nominalmente na reunião**, mas foi decidida por implicação: BullMQ exigiria Redis, exatamente a infraestrutura que Diego rejeitou como *"overengineering"* para o tamanho do time (`[09:07]`). Uma biblioteca com backend SQL seria menos invasiva, mas traria seu próprio schema, seu próprio vocabulário de estado e uma dependência a mais para manter — contra uma decisão que Bruno resumiu como *"não vamos botar nada novo"* (`[09:29]`).

## Consequências

### Positivas

- **Curva de entrada zero para o time.** Quem já mexeu em `src/modules/orders/` sabe onde ficam controller, service, repository, routes e schemas no módulo de webhooks.
- **O error middleware funciona sem alteração.** Erros `WEBHOOK_*` que herdem de `AppError` já saem no formato `{ error: { code, message, details } }` com o status HTTP correto, sem nenhuma linha nova em `src/middlewares/error.middleware.ts`.
- **Nenhuma dependência nova em `package.json`.** O `uuid` (para os identificadores de evento) e o `crypto` nativo do Node (para o HMAC de [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)) já cobrem o necessário — `uuid` está nas dependências e já é usado em `src/middlewares/request-logger.middleware.ts:2`.
- **Superfície de revisão de segurança menor.** Sofia revisa a lógica nova (HMAC e geração de secret, `[09:46]`), não uma stack inteira de infraestrutura.
- **Consistência de contrato para o cliente da API.** Os endpoints de webhook respondem com o mesmo envelope de erro e o mesmo formato paginado (`src/shared/http/response.ts`) dos demais recursos.

### Negativas

- **O padrão de módulo não cobre bem o worker.** A tríade controller/service/repository pressupõe um ciclo request-response; o processador de outbox não tem request. Ele entra como um arquivo adicional dentro do módulo (`webhook.worker.ts` ou `webhook.processor.ts`, `[09:28] Bruno`), o que é uma acomodação, não um encaixe natural.
- **Escrever o loop de polling e o agendamento de backoff à mão** significa reimplementar comportamento que bibliotecas de fila já resolvem — incluindo casos de borda como worker que morre no meio do processamento de um batch.
- **A lista de `redactPaths` do logger precisa ser estendida.** `src/shared/logger/index.ts:4` cobre `password`, `passwordHash`, `token` e `accessToken`, mas não `secret` — reusar o logger como está vazaria secrets de webhook em log. É uma alteração pequena, mas obrigatória, e a única mudança em arquivo compartilhado que o reuso não elimina.
- **Reuso do `requireRole` fixa o modelo de autorização atual.** Os roles disponíveis são apenas `ADMIN` e `OPERATOR` (`src/middlewares/auth.middleware.ts:9`), o que basta para a decisão de Sofia sobre o replay (`[09:36]`), mas não permite granularidade por customer. Sofia registrou que o endurecimento fica para depois (`[09:37]`): *"Por enquanto sim. Mais pra frente a gente pode endurecer."*
