# Tracker de Rastreabilidade

Mapeia cada item registrado no pacote de documentação à sua origem — um trecho da reunião ([`TRANSCRICAO.md`](../TRANSCRICAO.md)) ou um arquivo do código-fonte.

**Regra que este documento existe para garantir:** nenhum requisito, decisão ou restrição foi inventado. Se uma linha não tem origem identificável, o item correspondente foi removido ou reescrito nos documentos.

**Legenda da coluna Fonte:**
- `TRANSCRICAO` — localização no formato `[hh:mm] Nome`, referente à fala que originou o item.
- `CODIGO` — caminho de arquivo real do repositório.

**O que conta como "item identificável".** Cada requisito funcional e não funcional, decisão, alternativa descartada, questão em aberto, restrição, trade-off, contrato de endpoint, código de erro, ponto de integração com o código, dependência e risco registrado nos documentos. **Não** são contados separadamente: critérios de aceitação (PRD §11 e FDD §13), que são reformulações verificáveis de requisitos já rastreados nesta tabela; textos de contexto e seções de método. Decisões de implementação sem origem na reunião — como o `batchSize` default ou o limiar de recuperação de eventos travados — estão sinalizadas como tal no próprio FDD e por isso não aparecem aqui: elas não afirmam origem que não têm.

**Cobertura:** 228 itens rastreados — 182 (80%) com origem na transcrição, 46 (20%) com origem no código. As 85 referências distintas a falas da reunião foram verificadas contra o texto de `TRANSCRICAO.md`, e todos os caminhos de arquivo foram verificados contra o repositório.

---

## PRD — `docs/PRD.md`

### Contexto, problema e público

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram formalmente notificação em tempo real | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-02 | docs/PRD.md | Problema | Polling no `GET /orders` deixa a integração lenta e cara para os clientes | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-03 | docs/PRD.md | Restrição | Risco de churn: Atlas pode migrar para o concorrente se não entregarmos até o fim do trimestre | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-04 | docs/PRD.md | Restrição | "Tempo real" para os clientes significa qualquer coisa abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| PRD-CTX-05 | docs/PRD.md | Escopo | A feature é outbound apenas: o cliente recebe, não envia | TRANSCRICAO | `[09:02] Marcos` |
| PRD-CTX-06 | docs/PRD.md | Contexto | O sistema não tem hoje nenhum mecanismo de notificação externa, evento ou fila | CODIGO | `src/routes/index.ts` |
| PRD-UC-01 | docs/PRD.md | Cenário de uso | Onboarding: cliente cadastra URL e escolhe status ("só quero saber quando vira SHIPPED e DELIVERED") | TRANSCRICAO | `[09:33] Marcos` |
| PRD-UC-03 | docs/PRD.md | Cenário de uso | Cliente temporariamente indisponível; precedente de indisponibilidade de duas horas em manutenção planejada | TRANSCRICAO | `[09:16] Diego` |
| PRD-UC-04 | docs/PRD.md | Cenário de uso | Investigação de entrega: últimos 100 webhooks com sucesso/falha, payload, response e tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| PRD-UC-06 | docs/PRD.md | Cenário de uso | Resposta a vazamento de secret; precedente de cliente que vazou secret em log de aplicação | TRANSCRICAO | `[09:22] Diego` |

### Objetivos e métricas

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Latência p95 de entrega abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OBJ-01b | docs/PRD.md | Métrica | Alvo interno de ~2s no caminho feliz, derivado do ciclo de polling | TRANSCRICAO | `[09:09] Diego` |
| PRD-OBJ-02 | docs/PRD.md | Métrica | Redução do polling em `GET /orders` pelos clientes com webhook ativo (meta de 80% proposta pelo PM) | TRANSCRICAO | `[09:00] Marcos` |
| PRD-OBJ-03 | docs/PRD.md | Métrica | 100% dos eventos com destino terminal conhecido (entregue ou DLQ) | TRANSCRICAO | `[09:06] Diego` |
| PRD-OBJ-04 | docs/PRD.md | Métrica | Três clientes com webhook ativo até o fim de novembro | TRANSCRICAO | `[09:45] Marcos` |
| PRD-OBJ-05 | docs/PRD.md | Métrica | Não degradar a latência p95 de `PATCH /orders/:id/status` | TRANSCRICAO | `[09:04] Bruno` |
| PRD-OBJ-06 | docs/PRD.md | Métrica | Taxa de eventos em DLQ abaixo de 1% ao mês (meta proposta pelo PM) | TRANSCRICAO | `[09:18] Diego` |

### Requisitos funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook: url, lista de status; secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| PRD-RF-02 | docs/PRD.md | Requisito Funcional | Listar os webhooks de um customer | TRANSCRICAO | `[09:33] Bruno` |
| PRD-RF-03 | docs/PRD.md | Requisito Funcional | Editar webhook cadastrado (PATCH) | TRANSCRICAO | `[09:33] Bruno` |
| PRD-RF-04 | docs/PRD.md | Requisito Funcional | Remover webhook cadastrado (DELETE) | TRANSCRICAO | `[09:33] Bruno` |
| PRD-RF-05 | docs/PRD.md | Requisito Funcional | Filtro de status por endpoint: webhook só recebe os status que assinou | TRANSCRICAO | `[09:33] Marcos` |
| PRD-RF-05b | docs/PRD.md | Requisito Funcional | O filtro é aplicado na inserção da outbox; se ninguém assina, não insere | TRANSCRICAO | `[09:34] Bruno` |
| PRD-RF-06 | docs/PRD.md | Requisito Funcional | Evento gerado atomicamente com a mudança de status; falha no insert causa rollback | TRANSCRICAO | `[09:40] Bruno` |
| PRD-RF-07 | docs/PRD.md | Requisito Funcional | Entrega via POST com payload contendo event_id, event_type, timestamp, dados do pedido e status | TRANSCRICAO | `[09:43] Diego` |
| PRD-RF-08 | docs/PRD.md | Requisito Funcional | Requisição assinada com HMAC-SHA256 usando a secret do endpoint | TRANSCRICAO | `[09:20] Sofia` |
| PRD-RF-09 | docs/PRD.md | Requisito Funcional | Rotação de secret pela API com grace period de 24h para a secret anterior | TRANSCRICAO | `[09:21] Sofia` |
| PRD-RF-10 | docs/PRD.md | Requisito Funcional | Retentativa automática com intervalos crescentes até um limite | TRANSCRICAO | `[09:15] Diego` |
| PRD-RF-11 | docs/PRD.md | Requisito Funcional | Falha permanente registrada com payload, motivo e timestamp | TRANSCRICAO | `[09:18] Diego` |
| PRD-RF-12 | docs/PRD.md | Requisito Funcional | Reprocessamento manual por administrador, com registro do autor | TRANSCRICAO | `[09:18] Diego` |
| PRD-RF-13 | docs/PRD.md | Requisito Funcional | Consulta ao histórico de entregas do webhook | TRANSCRICAO | `[09:34] Marcos` |
| PRD-RF-14 | docs/PRD.md | Requisito Funcional | Identificador único e estável de evento para deduplicação pelo cliente | TRANSCRICAO | `[09:25] Diego` |
| PRD-RF-15 | docs/PRD.md | Requisito Funcional | Webhook pode ser ativado/desativado sem ser removido (estado ativo no cadastro) | TRANSCRICAO | `[09:21] Bruno` |

### Requisitos não funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por tentativa de entrega | TRANSCRICAO | `[09:42] Diego` |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Cinco tentativas por evento | TRANSCRICAO | `[09:15] Diego` |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Backoff de 1min, 5min, 30min, 2h, 12h | TRANSCRICAO | `[09:17] Diego` |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Payload limitado a 64KB; acima disso, erro em vez de truncamento | TRANSCRICAO | `[09:24] Diego` |
| PRD-RNF-05b | docs/PRD.md | Trade-off | Erro em vez de truncar: "se chegou nesse tamanho, tem algo errado" | TRANSCRICAO | `[09:23] Sofia` |
| PRD-RNF-06 | docs/PRD.md | Restrição | TLS obrigatório: URL precisa ser https; http é recusado na validação | TRANSCRICAO | `[09:23] Sofia` |
| PRD-RNF-07 | docs/PRD.md | Restrição | Secret única por endpoint, nunca global ("se vaza uma, vaza tudo") | TRANSCRICAO | `[09:21] Sofia` |
| PRD-RNF-08 | docs/PRD.md | Restrição | Garantia at-least-once; deduplicação é responsabilidade do cliente | TRANSCRICAO | `[09:24] Diego` |
| PRD-RNF-09 | docs/PRD.md | Restrição | Ordering garantido apenas por order_id e apenas enquanto single-worker | TRANSCRICAO | `[09:13] Larissa` |
| PRD-RNF-10 | docs/PRD.md | Restrição | Worker roda em processo separado da API | TRANSCRICAO | `[09:11] Diego` |
| PRD-RNF-11 | docs/PRD.md | Requisito Não Funcional | A geração do evento não pode degradar a mudança de status de forma perceptível | TRANSCRICAO | `[09:04] Bruno` |
| PRD-RNF-12 | docs/PRD.md | Requisito Não Funcional | Reprocessamentos manuais registram o autor, para auditoria | TRANSCRICAO | `[09:36] Sofia` |
| PRD-RNF-13 | docs/PRD.md | Restrição | Nenhuma infraestrutura nova; reuso do banco, logger e padrões existentes | TRANSCRICAO | `[09:30] Larissa` |

### Fora de escopo

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FE-01 | docs/PRD.md | Fora de Escopo | E-mail de aviso ao cliente quando o webhook falha — adiado para a próxima fase | TRANSCRICAO | `[09:37] Larissa` |
| PRD-FE-01b | docs/PRD.md | Fora de Escopo | Pedido original de e-mail após 3 falhas seguidas, feito pelo PM | TRANSCRICAO | `[09:37] Marcos` |
| PRD-FE-02 | docs/PRD.md | Fora de Escopo | Dashboard/painel visual para o cliente — projeto separado do time de frontend | TRANSCRICAO | `[09:40] Larissa` |
| PRD-FE-03 | docs/PRD.md | Fora de Escopo | Rate limiting de envio — "observa e implementa se virar problema" | TRANSCRICAO | `[09:39] Diego` |
| PRD-FE-04 | docs/PRD.md | Fora de Escopo | Webhooks de entrada (inbound) | TRANSCRICAO | `[09:02] Marcos` |
| PRD-FE-05 | docs/PRD.md | Fora de Escopo | Arquivamento de linhas entregues após ~30 dias | TRANSCRICAO | `[09:08] Diego` |
| PRD-FE-06 | docs/PRD.md | Fora de Escopo | Múltiplos workers e ordering global — "problema do futuro, não agora" | TRANSCRICAO | `[09:13] Diego` |
| PRD-FE-06b | docs/PRD.md | Restrição | Clientes nunca pediram garantia de ordering global | TRANSCRICAO | `[09:14] Marcos` |
| PRD-FE-07 | docs/PRD.md | Fora de Escopo | Endurecimento da autorização do CRUD — "mais pra frente a gente pode endurecer" | TRANSCRICAO | `[09:37] Sofia` |

### Dependências, riscos e validação

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-DEP-01 | docs/PRD.md | Dependência | Módulo de pedidos: a geração do evento acontece dentro da mudança de status | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-DEP-02 | docs/PRD.md | Dependência | Cadastro de clientes: todo webhook pertence a um customer existente | CODIGO | `prisma/schema.prisma` |
| PRD-DEP-03 | docs/PRD.md | Dependência | Autenticação e roles: o replay exige role ADMIN | TRANSCRICAO | `[09:36] Larissa` |
| PRD-DEP-04 | docs/PRD.md | Dependência | MySQL serve como armazenamento e como fila | TRANSCRICAO | `[09:08] Larissa` |
| PRD-DEP-05 | docs/PRD.md | Dependência | Revisão de segurança de Sofia: dois dias úteis antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| PRD-DEP-06 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor, sob responsabilidade do PM | TRANSCRICAO | `[09:26] Marcos` |
| PRD-DEP-07 | docs/PRD.md | Dependência | Capacidade de engenharia estimada em três sprints, com revisão incluída | TRANSCRICAO | `[09:46] Larissa` |
| PRD-R-01 | docs/PRD.md | Risco | Degradação da transação de mudança de status (prob. média / impacto alto) | TRANSCRICAO | `[09:04] Bruno` |
| PRD-R-02 | docs/PRD.md | Risco | Vazamento de secret (prob. baixa / impacto alto) | TRANSCRICAO | `[09:22] Diego` |
| PRD-R-03 | docs/PRD.md | Risco | Cliente não implementa dedup e processa duplicatas (prob. média / impacto médio) | TRANSCRICAO | `[09:25] Sofia` |
| PRD-R-04 | docs/PRD.md | Risco | Worker cai silenciosamente (prob. média / impacto alto) | TRANSCRICAO | `[09:11] Diego` |
| PRD-R-05 | docs/PRD.md | Risco | Atraso na revisão de segurança compromete o prazo comercial | TRANSCRICAO | `[09:47] Larissa` |
| PRD-R-06 | docs/PRD.md | Risco | Cliente lento ou volumoso atrasa entregas dos demais | TRANSCRICAO | `[09:38] Diego` |
| PRD-R-07 | docs/PRD.md | Risco | Crescimento da outbox sem política de expurgo | TRANSCRICAO | `[09:08] Diego` |
| PRD-CA-04 | docs/PRD.md | Critério de Aceitação | Falha no registro do evento impede a mudança de status | TRANSCRICAO | `[09:40] Bruno` |
| PRD-CA-15 | docs/PRD.md | Critério de Aceitação | Sequência rápida de mudanças no mesmo pedido chega em ordem | TRANSCRICAO | `[09:12] Larissa` |
| PRD-TEST-01 | docs/PRD.md | Estratégia de Teste | Integração no order.service e testes ponta a ponta previstos na estimativa | TRANSCRICAO | `[09:46] Larissa` |
| PRD-TEST-02 | docs/PRD.md | Estratégia de Teste | Revisão de segurança focada em HMAC e geração de secret | TRANSCRICAO | `[09:46] Sofia` |
| PRD-TEST-03 | docs/PRD.md | Estratégia de Teste | Padrão de teste de integração com Vitest e Supertest já estabelecido | CODIGO | `tests/orders.test.ts` |

---

## RFC — `docs/RFC.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-META-01 | docs/RFC.md | Metadado | Larissa como autora: abriria o doc de design e marcaria sessão de revisão | TRANSCRICAO | `[09:50] Larissa` |
| RFC-META-02 | docs/RFC.md | Metadado | Bruno e Diego como revisores da sessão de design | TRANSCRICAO | `[09:50] Larissa` |
| RFC-PROP-01 | docs/RFC.md | Decisão | Padrão outbox no MySQL, transação atômica com a mudança de status | TRANSCRICAO | `[09:48] Larissa` |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker separado em polling de 2 segundos | TRANSCRICAO | `[09:48] Larissa` |
| RFC-PROP-03 | docs/RFC.md | Decisão | Retry com backoff, 5 tentativas, depois DLQ em tabela separada | TRANSCRICAO | `[09:48] Larissa` |
| RFC-PROP-04 | docs/RFC.md | Decisão | HMAC-SHA256, secret por endpoint, rotação com grace period de 24h | TRANSCRICAO | `[09:48] Larissa` |
| RFC-PROP-05 | docs/RFC.md | Decisão | Idempotência por X-Event-Id, garantia at-least-once | TRANSCRICAO | `[09:48] Larissa` |
| RFC-PROP-06 | docs/RFC.md | Decisão | Padrões do projeto reaproveitados; módulo em src/modules/webhooks | TRANSCRICAO | `[09:48] Larissa` |
| RFC-PROP-07 | docs/RFC.md | Decisão | CRUD autenticado normal; replay de DLQ exige role ADMIN | TRANSCRICAO | `[09:48] Larissa` |
| RFC-PROP-08 | docs/RFC.md | Decisão | `publishWebhookEvent(tx, order, fromStatus, toStatus)` como ponto de integração | TRANSCRICAO | `[09:41] Bruno` |
| RFC-PROP-09 | docs/RFC.md | Trade-off | Função pura recebendo o tx, em vez de injetar o repository no OrderService | TRANSCRICAO | `[09:41] Diego` |
| RFC-PROP-10 | docs/RFC.md | Restrição | `customer_id` não vem do JWT: o JWT atual é do usuário operador | TRANSCRICAO | `[09:32] Bruno` |
| RFC-ALT-01 | docs/RFC.md | Alternativa | Disparo síncrono no changeStatus — descartado: cliente lento trava mudanças de outros pedidos | TRANSCRICAO | `[09:04] Bruno` |
| RFC-ALT-01b | docs/RFC.md | Alternativa | Síncrono "está fora de questão"; não há resposta razoável se o cliente estiver fora do ar | TRANSCRICAO | `[09:06] Diego` |
| RFC-ALT-02 | docs/RFC.md | Alternativa | Redis Streams — descartado: subir Redis Cluster é overengineering para o time | TRANSCRICAO | `[09:07] Diego` |
| RFC-ALT-02b | docs/RFC.md | Alternativa | Broker externo levantado como opção, exigiria subir mais infraestrutura | TRANSCRICAO | `[09:07] Larissa` |
| RFC-ALT-03 | docs/RFC.md | Alternativa | Trigger de banco — descartado: MySQL não tem NOTIFY/LISTEN e trigger não avisa processo externo | TRANSCRICAO | `[09:09] Diego` |
| RFC-ALT-03b | docs/RFC.md | Alternativa | Proposta original de usar trigger para ser mais reativo | TRANSCRICAO | `[09:09] Bruno` |
| RFC-ALT-04 | docs/RFC.md | Alternativa | Exactly-once — descartado: exigiria coordenação dos dois lados | TRANSCRICAO | `[09:25] Diego` |
| RFC-ALT-05 | docs/RFC.md | Alternativa | Três tentativas — descartado: mataria o evento em ~30 minutos | TRANSCRICAO | `[09:16] Diego` |
| RFC-ALT-05b | docs/RFC.md | Alternativa | Proposta original de 3 tentativas, mais agressiva | TRANSCRICAO | `[09:16] Bruno` |
| RFC-ALT-06 | docs/RFC.md | Alternativa | DLQ como flag "failed" na própria outbox — descartado em favor de tabela dedicada | TRANSCRICAO | `[09:18] Diego` |
| RFC-ALT-06b | docs/RFC.md | Alternativa | Pergunta que abriu a alternativa: tabela separada ou flag na outbox | TRANSCRICAO | `[09:17] Larissa` |
| RFC-ALT-07 | docs/RFC.md | Alternativa | Retry indefinido com backoff — descartado: evento fica pendurado para sempre | TRANSCRICAO | `[09:15] Diego` |
| RFC-Q1 | docs/RFC.md | Questão em Aberto | Rate limiting de saída: 50 mudanças em um minuto geram 50 chamadas | TRANSCRICAO | `[09:38] Diego` |
| RFC-Q1b | docs/RFC.md | Questão em Aberto | Registrado como "observar e decidir depois" | TRANSCRICAO | `[09:39] Larissa` |
| RFC-Q2 | docs/RFC.md | Questão em Aberto | Escala para múltiplos workers: particionar por order_id ou lock pessimista | TRANSCRICAO | `[09:13] Diego` |
| RFC-Q2b | docs/RFC.md | Questão em Aberto | Pergunta que abriu o ponto: e se algum dia quisermos escalar | TRANSCRICAO | `[09:13] Bruno` |
| RFC-Q3 | docs/RFC.md | Questão em Aberto | Retenção/arquivamento da outbox declarado fora do escopo, sem dono nem prazo | TRANSCRICAO | `[09:08] Diego` |
| RFC-Q4 | docs/RFC.md | Questão em Aberto | `customer_id` no body ou no path — não decidido | TRANSCRICAO | `[09:32] Larissa` |
| RFC-Q5 | docs/RFC.md | Questão em Aberto | Endurecimento da autorização do CRUD adiado | TRANSCRICAO | `[09:37] Sofia` |
| RFC-Q5b | docs/RFC.md | Questão em Aberto | Pergunta que abriu o ponto: CRUD pode ser qualquer role autenticada | TRANSCRICAO | `[09:36] Marcos` |
| RFC-IMP-01 | docs/RFC.md | Impacto | Alteração no caminho crítico: changeStatus ganha leitura e insert na mesma transação | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-IMP-02 | docs/RFC.md | Impacto | Novos modelos no schema; nenhum modelo existente alterado | CODIGO | `prisma/schema.prisma` |
| RFC-IMP-03 | docs/RFC.md | Impacto | Única alteração obrigatória em arquivo compartilhado: `redactPaths` do logger | CODIGO | `src/shared/logger/index.ts` |
| RFC-IMP-04 | docs/RFC.md | Impacto | Nenhuma dependência nova; apenas um script `worker` | CODIGO | `package.json` |
| RFC-RISK-01 | docs/RFC.md | Risco | Degradação da transação que já faz três escritas acopladas | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-CRON-01 | docs/RFC.md | Restrição | Estimativa de três sprints, revisão de segurança incluída | TRANSCRICAO | `[09:46] Larissa` |
| RFC-CRON-02 | docs/RFC.md | Restrição | Compromisso comercial: entrega até o fim de novembro | TRANSCRICAO | `[09:45] Marcos` |
| RFC-CRON-03 | docs/RFC.md | Restrição | Detalhe da estimativa por bloco (outbox/DLQ, worker/retry, CRUD, integração, HMAC) | TRANSCRICAO | `[09:46] Larissa` |

---

## FDD — `docs/FDD.md`

### Modelo de dados e fluxos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-DADOS-01 | docs/FDD.md | Decisão | Quatro estados da outbox: pendente, processando, falhou, entregue | TRANSCRICAO | `[09:08] Diego` |
| FDD-DADOS-02 | docs/FDD.md | Decisão | Índices em status e created_at; worker lê só os pendentes em batch pequeno | TRANSCRICAO | `[09:08] Diego` |
| FDD-DADOS-03 | docs/FDD.md | Decisão | UUID como chave primária, seguindo o padrão do projeto | TRANSCRICAO | `[09:51] Larissa` |
| FDD-DADOS-04 | docs/FDD.md | Restrição | Padrão real de PK no schema: `@id @default(uuid()) @db.Char(36)` | CODIGO | `prisma/schema.prisma` |
| FDD-DADOS-05 | docs/FDD.md | Decisão | Tabela de configuração guarda url, secret, customer_id e estado ativo | TRANSCRICAO | `[09:21] Bruno` |
| FDD-DADOS-06 | docs/FDD.md | Decisão | Tabela `webhook_dead_letter` separada com payload, motivo e timestamp | TRANSCRICAO | `[09:18] Diego` |
| FDD-DADOS-07 | docs/FDD.md | Restrição | Enum `OrderStatus` reusado nos campos from/to do evento | CODIGO | `prisma/schema.prisma` |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Inserção na outbox dentro da transação de changeStatus; falha causa rollback | TRANSCRICAO | `[09:40] Bruno` |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Sem a transação, perde-se a garantia inteira | TRANSCRICAO | `[09:41] Diego` |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Filtro de status aplicado na inserção, não no envio | TRANSCRICAO | `[09:34] Bruno` |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Worker em polling: a cada 2s busca os pendentes mais antigos, processa, marca | TRANSCRICAO | `[09:09] Diego` |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Processamento sequencial preserva ordering por order_id em single-worker | TRANSCRICAO | `[09:12] Diego` |
| FDD-FLOW-06 | docs/FDD.md | Fluxo | Retry: 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Diego` |
| FDD-FLOW-07 | docs/FDD.md | Fluxo | Após esgotar as tentativas, o evento vai para a DLQ | TRANSCRICAO | `[09:15] Diego` |
| FDD-FLOW-08 | docs/FDD.md | Fluxo | Replay recoloca o evento na outbox como pendente | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLOW-09 | docs/FDD.md | Fluxo | Payload gravado como snapshot na inserção, não renderizado no envio | TRANSCRICAO | `[09:52] Larissa` |
| FDD-FLOW-10 | docs/FDD.md | Decisão | Confirmação da decisão de snapshot pelos dois engenheiros | TRANSCRICAO | `[09:52] Diego` |
| FDD-FLOW-11 | docs/FDD.md | Fluxo | Bootstrap do worker espelha o de `src/server.ts` (SIGINT/SIGTERM, `$disconnect`) | CODIGO | `src/server.ts` |
| FDD-FLOW-12 | docs/FDD.md | Restrição | Worker instancia PrismaClient próprio via factory existente | CODIGO | `src/config/database.ts` |

### Contratos públicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /api/v1/webhooks` — cadastro; secret devolvida apenas nesta resposta | TRANSCRICAO | `[09:31] Marcos` |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /api/v1/webhooks?customerId=` — listagem dos webhooks de um customer | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /api/v1/webhooks/:id` — edição | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /api/v1/webhooks/:id` — remoção | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `POST /api/v1/webhooks/:id/rotate-secret` — rotação com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | `GET /api/v1/webhooks/:id/deliveries` — últimas 100 entregas com payload, response e duração | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` — replay, exige ADMIN | TRANSCRICAO | `[09:35] Diego` |
| FDD-CONTRATO-08 | docs/FDD.md | Restrição | Prefixo `/api/v1` em todas as rotas, conforme o app existente | CODIGO | `src/app.ts` |
| FDD-CONTRATO-09 | docs/FDD.md | Restrição | Envelope de erro `{ error: { code, message, details } }` | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-CONTRATO-10 | docs/FDD.md | Restrição | Envelope paginado `{ data, pagination }` reusado nas listagens | CODIGO | `src/shared/http/response.ts` |
| FDD-CONTRATO-11 | docs/FDD.md | Restrição | Defaults de paginação (page 1, pageSize 20, teto 100) herdados do módulo de orders | CODIGO | `src/modules/orders/order.schemas.ts` |
| FDD-CONTRATO-12 | docs/FDD.md | Restrição | `204 No Content` no DELETE, seguindo o controller de orders | CODIGO | `src/modules/orders/order.controller.ts` |
| FDD-HEADER-01 | docs/FDD.md | Contrato | Headers de saída: X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | `[09:44] Diego` |
| FDD-HEADER-02 | docs/FDD.md | Contrato | Header adicional X-Webhook-Id, para clientes com vários cadastros | TRANSCRICAO | `[09:44] Sofia` |
| FDD-HEADER-03 | docs/FDD.md | Contrato | X-Signature transporta o HMAC; cliente verifica do lado dele | TRANSCRICAO | `[09:20] Sofia` |
| FDD-PAYLOAD-01 | docs/FDD.md | Contrato | Campos do payload: event_id, event_type, timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, total_cents | TRANSCRICAO | `[09:43] Diego` |
| FDD-PAYLOAD-02 | docs/FDD.md | Trade-off | `items` deliberadamente ausente para não inflar o payload | TRANSCRICAO | `[09:43] Diego` |
| FDD-PAYLOAD-03 | docs/FDD.md | Trade-off | Payload enxuto confirmado pelo time | TRANSCRICAO | `[09:44] Bruno` |
| FDD-PAYLOAD-04 | docs/FDD.md | Restrição | Campos `subtotalCents`/`discountCents` existem no modelo Order e ficam como decisão a ratificar | CODIGO | `prisma/schema.prisma` |

### Erros, resiliência e observabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-ERRO-01 | docs/FDD.md | Restrição | Prefixo `WEBHOOK_` para todos os códigos de erro do módulo | TRANSCRICAO | `[09:29] Larissa` |
| FDD-ERRO-02 | docs/FDD.md | Contrato | Códigos nomeados na reunião: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERRO-03 | docs/FDD.md | Restrição | Convenção de código de erro derivada de INSUFFICIENT_STOCK e INVALID_STATUS_TRANSITION | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERRO-04 | docs/FDD.md | Restrição | Classe base `AppError` com statusCode, errorCode e details | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-ERRO-05 | docs/FDD.md | Contrato | `WEBHOOK_PAYLOAD_TOO_LARGE` derivado do teto de 64KB | TRANSCRICAO | `[09:24] Diego` |
| FDD-ERRO-06 | docs/FDD.md | Contrato | `WEBHOOK_INVALID_URL` derivado da exigência de https | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERRO-07 | docs/FDD.md | Trade-off | Semântica de `WEBHOOK_SECRET_REQUIRED` ajustada: a secret nunca é informada pelo cliente | TRANSCRICAO | `[09:31] Marcos` |
| FDD-RESIL-01 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s; cliente que não responde é tratado como falha | TRANSCRICAO | `[09:42] Diego` |
| FDD-RESIL-02 | docs/FDD.md | Decisão | Sem fallback: e-mail de aviso foi adiado | TRANSCRICAO | `[09:37] Larissa` |
| FDD-RESIL-03 | docs/FDD.md | Restrição | Shutdown gracioso replicando o padrão de handlers de sinal da API | CODIGO | `src/server.ts` |
| FDD-OBS-01 | docs/FDD.md | Restrição | Logger Pino já no projeto; nenhuma dependência nova de observabilidade | TRANSCRICAO | `[09:29] Bruno` |
| FDD-OBS-02 | docs/FDD.md | Restrição | `redactPaths` atual não cobre `secret` — extensão obrigatória | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-03 | docs/FDD.md | Requisito Não Funcional | Log de auditoria do replay: quem executou | TRANSCRICAO | `[09:36] Sofia` |
| FDD-OBS-04 | docs/FDD.md | Restrição | Correlação via `X-Request-Id` já gerado por requisição | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-OBS-05 | docs/FDD.md | Métrica | Latência de entrega medida contra o limite de 10s | TRANSCRICAO | `[09:02] Marcos` |

### Integração com o sistema existente

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-INT-01 | docs/FDD.md | Integração | `changeStatus`: chamada a `publishWebhookEvent(tx, ...)` entre o insert de histórico e o findUnique final | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-01b | docs/FDD.md | Integração | Bruno identificou changeStatus como "a alteração crítica" | TRANSCRICAO | `[09:40] Bruno` |
| FDD-INT-02 | docs/FDD.md | Integração | Erros do módulo estendem a hierarquia existente; ajuste necessário em `WebhookNotFoundError` | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INT-03 | docs/FDD.md | Integração | Error middleware trata AppError, Zod e Prisma sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-03b | docs/FDD.md | Integração | "Vai pegar nossos erros sem precisar mudar nada" | TRANSCRICAO | `[09:29] Bruno` |
| FDD-INT-04 | docs/FDD.md | Integração | `requireRole('ADMIN')` no endpoint de replay, reusando o middleware existente | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-04b | docs/FDD.md | Integração | Decisão de reaproveitar o requireRole existente | TRANSCRICAO | `[09:36] Larissa` |
| FDD-INT-05 | docs/FDD.md | Integração | Padrão de uso do requireRole já aplicado em outro módulo | CODIGO | `src/modules/users/user.routes.ts` |
| FDD-INT-06 | docs/FDD.md | Integração | Extensão de `redactPaths` para cobrir `secret` e `previousSecret` | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-07 | docs/FDD.md | Integração | Worker usa `createPrismaClient()`; API segue com o singleton | CODIGO | `src/config/database.ts` |
| FDD-INT-07b | docs/FDD.md | Integração | PrismaClient é por processo: mesmo banco, instância nova | TRANSCRICAO | `[09:30] Bruno` |
| FDD-INT-08 | docs/FDD.md | Integração | `src/worker.ts` como nova entrypoint espelhando `src/server.ts`, com script `npm run worker` | TRANSCRICAO | `[09:11] Larissa` |
| FDD-INT-09 | docs/FDD.md | Integração | Lógica de processamento em `webhook.worker.ts` dentro do módulo | TRANSCRICAO | `[09:28] Bruno` |
| FDD-INT-10 | docs/FDD.md | Integração | Registro do router de webhooks no agregador de rotas | CODIGO | `src/routes/index.ts` |
| FDD-INT-11 | docs/FDD.md | Integração | Montagem repository → service → controller em `buildControllers` | CODIGO | `src/app.ts` |
| FDD-INT-12 | docs/FDD.md | Integração | Validação Zod via middleware `validate({ body, query, params })` | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-13 | docs/FDD.md | Integração | Validação de https no schema Zod, não como decisão arquitetural | TRANSCRICAO | `[09:23] Sofia` |
| FDD-INT-14 | docs/FDD.md | Integração | Schemas do módulo seguindo o formato do módulo de orders | CODIGO | `src/modules/orders/order.schemas.ts` |
| FDD-INT-15 | docs/FDD.md | Integração | Estrutura modular controller/service/repository/routes/schemas | TRANSCRICAO | `[09:27] Bruno` |
| FDD-INT-16 | docs/FDD.md | Integração | Modelos novos adicionados ao schema Prisma existente | CODIGO | `prisma/schema.prisma` |
| FDD-INT-17 | docs/FDD.md | Integração | Máquina de estados permite sequências rápidas entre status | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-18 | docs/FDD.md | Integração | Testes novos seguindo o padrão de factories existente | CODIGO | `tests/helpers/factories.ts` |
| FDD-DEP-01 | docs/FDD.md | Dependência | Nenhuma dependência nova; `uuid`, `zod` e `pino` já presentes; Node >= 20 | CODIGO | `package.json` |
| FDD-DEP-02 | docs/FDD.md | Dependência | Novas variáveis de ambiente no schema Zod com defaults | CODIGO | `src/config/env.ts` |
| FDD-DEP-03 | docs/FDD.md | Restrição | DATABASE_URL compartilhada entre API e worker | TRANSCRICAO | `[09:30] Bruno` |
| FDD-CAT-16 | docs/FDD.md | Critério de Aceite | Sequência PAID → PROCESSING → SHIPPED entregue em ordem | TRANSCRICAO | `[09:12] Larissa` |

---

## ADRs — `docs/adrs/`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox no MySQL: evento inserido na mesma transação da mudança de status | TRANSCRICAO | `[09:06] Diego` |
| ADR-001b | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Ratificação: "Tá decidido então: outbox em MySQL" | TRANSCRICAO | `[09:08] Larissa` |
| ADR-001c | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Transação atual já faz update da order, insert no histórico e ajuste de estoque | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Polling em loop a cada 2 segundos | TRANSCRICAO | `[09:09] Diego` |
| ADR-002b | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker como processo separado da API | TRANSCRICAO | `[09:11] Diego` |
| ADR-002c | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Latência mínima de 2 segundos aceita explicitamente | TRANSCRICAO | `[09:10] Larissa` |
| ADR-002d | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | 2 segundos validado pelo PM como suficiente | TRANSCRICAO | `[09:10] Marcos` |
| ADR-002e | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Worker conecta no mesmo banco e usa o mesmo Prisma | TRANSCRICAO | `[09:11] Bruno` |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Backoff exponencial com teto de tentativas, depois DLQ | TRANSCRICAO | `[09:15] Diego` |
| ADR-003b | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Cinco tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Larissa` |
| ADR-003c | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | DLQ em tabela dedicada com payload, motivo e timestamp | TRANSCRICAO | `[09:18] Diego` |
| ADR-003d | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Replay manual via endpoint admin, com role ADMIN | TRANSCRICAO | `[09:36] Sofia` |
| ADR-003e | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | Janela de ~15 horas aceita pelo PM | TRANSCRICAO | `[09:17] Marcos` |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace de 24h | TRANSCRICAO | `[09:22] Sofia` |
| ADR-004b | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Escolha do SHA-256 pelo critério de disponibilidade de biblioteca no cliente | TRANSCRICAO | `[09:20] Sofia` |
| ADR-004c | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Contexto | Necessidade de autenticidade e integridade no destino | TRANSCRICAO | `[09:19] Sofia` |
| ADR-004d | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Contexto | Precedente de cliente que vazou secret em log | TRANSCRICAO | `[09:22] Diego` |
| ADR-004e | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | `redactPaths` existente não cobre `secret` | CODIGO | `src/shared/logger/index.ts` |
| ADR-004f | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Dependência | Revisão de segurança de dois dias antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once; cliente pode receber duplicado | TRANSCRICAO | `[09:24] Diego` |
| ADR-005b | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | UUID em `X-Event-Id` gerado na entrada da outbox | TRANSCRICAO | `[09:25] Diego` |
| ADR-005c | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de dedup transferida ao cliente | TRANSCRICAO | `[09:25] Sofia` |
| ADR-005d | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | Ratificação da decisão de at-least-once com X-Event-Id | TRANSCRICAO | `[09:26] Larissa` |
| ADR-005e | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Dependência | Documentação destacada no portal do desenvolvedor | TRANSCRICAO | `[09:26] Marcos` |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, schemas Zod, códigos de erro | TRANSCRICAO | `[09:30] Larissa` |
| ADR-006b | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | Estrutura modular existente com controller, service, repository, routes e schemas | CODIGO | `src/modules/orders/order.repository.ts` |
| ADR-006c | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | Nenhuma dependência nova de observabilidade | TRANSCRICAO | `[09:29] Bruno` |
| ADR-006d | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | `uuid` já é dependência e já é usado no projeto | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| ADR-006e | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Trade-off | Roles existentes limitados a ADMIN e OPERATOR, sem granularidade por customer | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Decisão | Payload renderizado no momento da inserção (snapshot) | TRANSCRICAO | `[09:52] Larissa` |
| ADR-007b | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Contexto | Pergunta que originou a decisão: payload renderizado ou só order_id | TRANSCRICAO | `[09:51] Bruno` |
| ADR-007c | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Decisão | Confirmação: "snapshot na inserção" | TRANSCRICAO | `[09:52] Diego` |
| ADR-007d | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Trade-off | Sem `items` no payload; detalhe fica no GET /orders/:id | TRANSCRICAO | `[09:43] Diego` |
| ADR-007e | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Restrição | Transições rápidas permitidas pela máquina de estados criam a janela de divergência | CODIGO | `src/modules/orders/order.status.ts` |

---

## Itens deliberadamente não documentados

Pontos mencionados na reunião que **não** viraram requisito, decisão ou seção nos documentos — registrados aqui para deixar explícito que foram avaliados e descartados, e não esquecidos.

| Item | Por que não entrou | Localização |
| --- | --- | --- |
| Explicação didática do que é o padrão outbox | Contexto de reunião, não conteúdo de documento | `[09:06] Bruno` |
| Atraso do Diego na call e reorganização da agenda | Irrelevante para o produto | `[09:05] Diego` |
| Confirmação de que o time fecharia a call | Encerramento, sem conteúdo técnico | `[09:53] Larissa` |
| Menção a Stripe e GitHub como referências | Justificativa retórica da decisão; citada no ADR-005 como argumento, não como requisito | `[09:25] Diego` |
| Truncamento de payload acima do limite | Alternativa levantada e rejeitada na mesma fala; registrada como trade-off, não como comportamento | `[09:23] Sofia` |
