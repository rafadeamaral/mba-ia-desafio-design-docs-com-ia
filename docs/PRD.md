# PRD — Sistema de Webhooks de Notificação de Pedidos

| | |
| --- | --- |
| **Produto** | Order Management System (OMS) |
| **Feature** | Webhooks de notificação de mudança de status de pedido |
| **Status** | Aprovado para implementação |
| **Data** | 2026-08-13 |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Fonte** | Reunião técnica de quinta-feira, 09:00 — [`TRANSCRICAO.md`](../TRANSCRICAO.md) |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

> Este documento responde **"por que e o quê"**. A proposta técnica está no [RFC](RFC.md), as decisões nos [ADRs](adrs/) e o detalhamento de implementação no [FDD](FDD.md).

---

## 1. Resumo e contexto

O OMS passa a **notificar ativamente os clientes B2B** quando o status de um pedido deles muda, em vez de obrigá-los a consultar a API repetidamente em busca de novidades.

Na prática: o cliente cadastra pela nossa API uma URL `https` própria e escolhe quais status quer acompanhar. A cada mudança de status de um pedido dele, enviamos um `POST` assinado para essa URL, em geral em menos de 5 segundos. Se a URL do cliente estiver fora do ar, retentamos ao longo de aproximadamente 15 horas antes de desistir e registrar a falha para tratamento manual.

O sistema **não possui hoje nenhum mecanismo de notificação externa, evento ou fila**. Esta é a primeira integração de saída do produto, e por isso a feature carrega não só valor de negócio imediato, mas a fundação sobre a qual futuras notificações do OMS serão construídas.

## 2. Problema e motivação

### O problema do cliente

Três clientes B2B — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — abriram pedido formal para serem notificados em tempo real (`[09:00] Marcos`).

Hoje eles resolvem isso com polling. Marcos descreveu o custo (`[09:00]`): *"ficam batendo no GET /orders de tempos em tempos pra ver se mudou alguma coisa, e isso tá deixando a integração lenta e cara pra eles"*. O polling impõe ao cliente uma escolha ruim: consultar com frequência alta e gastar recursos dos dois lados, ou consultar com frequência baixa e descobrir tarde que o pedido mudou.

### A urgência comercial

Marcos registrou o risco (`[09:00]`): *"A Atlas chegou a sugerir que se a gente não entregar isso até fim do trimestre, eles podem migrar pro nosso concorrente."* O prazo comercial acordado é **fim de novembro** (`[09:45] Marcos`).

### O que "tempo real" significa aqui

Bruno questionou o requisito (`[09:01]`) e Marcos trouxe a resposta apurada com os próprios clientes (`[09:02]`): *"Pra eles, qualquer coisa abaixo de 10 segundos já é 'tempo real'. O importante é que não fique pendurado e eles tenham que ficar atualizando manualmente."*

Isso é decisivo para o escopo: a dor é **operacional**, não de baixa latência. O cliente precisa parar de perguntar, não precisa saber em milissegundos.

## 3. Público-alvo e cenários de uso

### Público-alvo

| Público | Necessidade |
| --- | --- |
| **Clientes B2B integrados via API** (Atlas Comercial, MaxDistribuição, Nova Cargo) | Reagir automaticamente a mudanças de status sem polling |
| **Time de suporte/operações interno** | Diagnosticar por que um cliente não recebeu uma notificação |
| **Administradores da plataforma** | Reprocessar entregas que falharam em definitivo |

Vale registrar quem **não** é público desta fase: o usuário final do painel. Marcos perguntou sobre um dashboard visual e Larissa respondeu (`[09:40]`): *"Não, agora não. Só endpoints. Painel é projeto separado do time de frontend."*

### Cenários de uso

**C1 — Onboarding de um cliente.** O cliente cadastra sua URL `https` e escolhe os status que quer ouvir — Marcos deu o exemplo (`[09:33]`): *"só quero saber quando vira SHIPPED e DELIVERED"*. Recebe na resposta a secret que usará para validar as requisições. A partir daí, é notificado automaticamente.

**C2 — Notificação no fluxo normal.** Um operador muda um pedido de `PROCESSING` para `SHIPPED` no OMS. Em segundos, o sistema do cliente recebe a notificação assinada e dispara sua própria rotina interna.

**C3 — Cliente temporariamente indisponível.** A URL do cliente está fora do ar por manutenção. O sistema retenta ao longo de horas — Diego citou o precedente (`[09:16]`): *"Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada"* — e entrega assim que o cliente volta, sem intervenção nossa.

**C4 — Investigação de entrega.** O cliente afirma não ter recebido uma notificação. Marcos definiu a necessidade (`[09:34]`): *"o cliente precisa conseguir ver o histórico de entregas. Tipo 'esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta'."*

**C5 — Recuperação de falha permanente.** Um cliente ficou fora do ar tempo demais e eventos esgotaram as tentativas. Corrigido o problema, um administrador reprocessa manualmente os eventos que falharam.

**C6 — Resposta a vazamento de secret.** O cliente suspeita que sua secret vazou. Diego citou o precedente real (`[09:22]`): *"A gente já teve cliente que vazou secret em log de aplicação dele uma vez."* O cliente pede uma nova secret pela API e tem 24 horas para migrar seus sistemas, sem perder nenhuma notificação nesse intervalo.

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| **OBJ-01** | Entregar notificações dentro do que os clientes consideram tempo real | Latência p95 entre a mudança de status e o `POST` recebido pelo cliente | **< 10 segundos** — limite declarado por Marcos (`[09:02]`). O desenho técnico (polling de 2s, `[09:09] Diego`) mira **≤ 5 segundos** no caminho feliz |
| **OBJ-02** | Eliminar o polling como forma de integração | Redução de chamadas a `GET /orders` pelos clientes com webhook ativo | **≥ 80%** de queda nos 30 dias após o onboarding de cada cliente |
| **OBJ-03** | Não perder eventos | Eventos entregues + eventos em DLQ ÷ eventos gerados | **100%** — todo evento tem destino terminal conhecido |
| **OBJ-04** | Reter os clientes que motivaram a feature | Atlas, MaxDistribuição e Nova Cargo com webhook ativo em produção | **3 de 3** até o fim de novembro (`[09:45] Marcos`) |
| **OBJ-05** | Não degradar a operação de pedidos | Latência p95 de `PATCH /orders/:id/status` | Sem aumento perceptível frente à linha de base pré-feature |
| **OBJ-06** | Manter as falhas permanentes sob controle | Eventos movidos para DLQ ÷ eventos gerados | **< 1%** ao mês, excluídas janelas de indisponibilidade declaradas pelo cliente |

> As metas de OBJ-02, OBJ-03 e OBJ-06 são propostas do PM para tornar os objetivos verificáveis; não foram debatidas na reunião. OBJ-01 e OBJ-04 vêm diretamente da transcrição.

## 5. Escopo

### 5.1 Incluso

- Cadastro, consulta, edição e remoção de configurações de webhook via API (`[09:31]`, `[09:33]`).
- Seleção, por endpoint, de quais status de pedido geram notificação (`[09:33] Marcos`).
- Entrega assíncrona e automática das notificações, com assinatura criptográfica.
- Retentativas automáticas em caso de falha do cliente.
- Registro de falhas permanentes e reprocessamento manual por administrador.
- Consulta ao histórico de entregas pelo cliente (`[09:34] Marcos`).
- Rotação de secret pela API, com janela de convivência de 24h (`[09:21] Sofia`).
- Documentação de integração no portal do desenvolvedor, sob responsabilidade de Marcos (`[09:26]`, `[09:40]`).

### 5.2 Fora de escopo

Itens **explicitamente descartados ou adiados** na reunião:

| # | Item | Decisão | Origem |
| --- | --- | --- | --- |
| **FE-01** | **Notificação por e-mail ao cliente quando o webhook dele falha** | Adiado para fase futura. Marcos pediu (`[09:37]`): *"Tem como avisar o cliente quando o webhook dele tá com problema? Tipo se ele falhou 3 vezes seguidas, mandar email pra ele."* Larissa: *"Não. Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto."* | `[09:37]` |
| **FE-02** | **Dashboard/painel visual para o cliente** | Descartado nesta fase. Larissa (`[09:40]`): *"Não, agora não. Só endpoints. Painel é projeto separado do time de frontend."* | `[09:40]` |
| **FE-03** | **Rate limiting de envio para o cliente** | Adiado, com observação ativa. Diego levantou o cenário de 50 pedidos mudando em um minuto (`[09:38]`) e concluiu (`[09:39]`): *"A gente observa e implementa se virar problema. Mas vale registrar como ponto em aberto."* | `[09:38]`–`[09:39]` |
| **FE-04** | **Webhooks de entrada (inbound)** | Fora de escopo desde a definição. Sofia perguntou (`[09:02]`) e Marcos respondeu: *"Só saindo da gente pra eles. Eles querem receber, não mandar."* | `[09:02]` |
| **FE-05** | **Arquivamento/expurgo de eventos antigos** | Explicitamente fora. Diego (`[09:08]`): *"Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature."* | `[09:08]` |
| **FE-06** | **Garantia de ordering global e múltiplos workers** | Adiado. Diego (`[09:13]`): *"dá pra particionar por order_id, ou usar lock pessimista. Mas isso é problema do futuro, não agora."* Marcos confirmou que não conflita com o pedido dos clientes (`[09:14]`): *"Os clientes nunca pediram garantia de ordering global."* | `[09:12]`–`[09:14]` |
| **FE-07** | **Autorização granular por cliente no CRUD de webhooks** | Adiado. Sofia (`[09:37]`): *"Por enquanto sim. Mais pra frente a gente pode endurecer."* Apenas o reprocessamento exige `ADMIN`. | `[09:36]`–`[09:37]` |

## 6. Requisitos funcionais

| # | Requisito | Origem |
| --- | --- | --- |
| **RF-01** | O cliente deve poder **cadastrar** um webhook informando a URL de destino, o cliente ao qual ele pertence e a lista de status que deseja receber. A secret é gerada pela plataforma e devolvida na criação. | `[09:31] Marcos` |
| **RF-02** | O cliente deve poder **listar** os webhooks cadastrados de um cliente. | `[09:33] Bruno` |
| **RF-03** | O cliente deve poder **editar** um webhook cadastrado. | `[09:33] Bruno` |
| **RF-04** | O cliente deve poder **remover** um webhook cadastrado. | `[09:33] Bruno` |
| **RF-05** | Cada webhook deve ter um **filtro de status**: só recebe notificação dos status que assinou. O filtro é aplicado no momento da geração do evento — se nenhum webhook do cliente assina aquele status, nenhum evento é gerado. | `[09:33] Marcos`, `[09:34] Bruno` |
| **RF-06** | O sistema deve **gerar um evento de notificação a cada mudança de status de pedido**, de forma atômica com a própria mudança: se o registro do evento falhar, a mudança de status não acontece. | `[09:40] Bruno`, `[09:41] Diego` |
| **RF-07** | O sistema deve **entregar a notificação** via `POST` na URL cadastrada, com o payload contendo identificador do evento, tipo, timestamp, dados do pedido e os status de origem e destino. | `[09:43] Diego` |
| **RF-08** | Toda requisição enviada deve ser **assinada** com HMAC-SHA256 usando a secret exclusiva daquele webhook, permitindo ao cliente verificar autenticidade e integridade. | `[09:20]`–`[09:22] Sofia` |
| **RF-09** | O cliente deve poder **rotacionar a secret** pela API. A secret anterior permanece válida por 24 horas para permitir a migração dos sistemas dele. | `[09:21] Sofia` |
| **RF-10** | O sistema deve **retentar automaticamente** entregas que falharem, com intervalos crescentes, até um limite de tentativas. | `[09:15]`–`[09:17]` |
| **RF-11** | Esgotadas as tentativas, o evento deve ser **registrado como falha permanente**, preservando payload, motivo e momento da falha. | `[09:18] Diego` |
| **RF-12** | Um **administrador** deve poder **reprocessar** manualmente um evento que falhou em definitivo, recolocando-o na fila de entrega. A operação registra quem a executou. | `[09:18]`, `[09:35]`, `[09:36]` |
| **RF-13** | O cliente deve poder **consultar o histórico de entregas** de um webhook, com sucesso/falha, payload, resposta recebida e tempo de resposta. | `[09:34] Marcos` |
| **RF-14** | Cada notificação deve carregar um **identificador único e estável de evento**, permitindo ao cliente descartar entregas duplicadas. | `[09:25] Diego` |
| **RF-15** | Cada webhook deve poder ser **ativado ou desativado** sem ser removido. | `[09:21] Bruno` |

## 7. Requisitos não funcionais

| # | Requisito | Valor | Origem |
| --- | --- | --- | --- |
| **RNF-01** | Latência de entrega no caminho feliz | Abaixo de 10 segundos; alvo de ~2s pelo desenho | `[09:02] Marcos`, `[09:09] Diego` |
| **RNF-02** | Timeout de cada tentativa de entrega | 10 segundos | `[09:42] Diego` |
| **RNF-03** | Número de tentativas por evento | 5 | `[09:15]`–`[09:16]` |
| **RNF-04** | Intervalos entre tentativas | 1min, 5min, 30min, 2h, 12h (≈15h de janela) | `[09:17] Diego` |
| **RNF-05** | Tamanho máximo do payload | 64KB. Acima disso, **erro** — não truncamento. Sofia (`[09:23]`): *"Eu sou a favor de erra. Se chegou nesse tamanho, tem algo errado"* | `[09:23]`–`[09:24]` |
| **RNF-06** | Transporte | TLS obrigatório: URL de webhook precisa ser `https`; `http` é recusado na validação | `[09:23] Sofia` |
| **RNF-07** | Isolamento de secret | Uma secret por endpoint, nunca global. Sofia (`[09:21]`): *"Senão se vaza uma, vaza tudo"* | `[09:21] Sofia` |
| **RNF-08** | Garantia de entrega | At-least-once. O cliente pode receber o mesmo evento mais de uma vez e é responsável pela deduplicação | `[09:24]`–`[09:26]` |
| **RNF-09** | Ordenação | Garantida por pedido, não globalmente, e apenas enquanto houver um único processador. Registrada como limitação conhecida | `[09:12]`–`[09:13]` |
| **RNF-10** | Isolamento operacional | A entrega roda em processo separado da API: reinício de um não afeta o outro | `[09:11] Diego` |
| **RNF-11** | Impacto na operação existente | A geração do evento não pode degradar de forma perceptível a mudança de status de pedido | `[09:04] Bruno` |
| **RNF-12** | Auditoria | Reprocessamentos manuais registram o autor. Sofia (`[09:36]`): *"o endpoint de admin tem que logar quem fez o replay, pra auditoria"* | `[09:36] Sofia` |
| **RNF-13** | Reuso de stack | Nenhuma infraestrutura nova; a feature usa o banco, o logger e os padrões já existentes | `[09:07] Diego`, `[09:29]`–`[09:30]` |

## 8. Decisões e trade-offs principais

As decisões estão registradas individualmente nos [ADRs](adrs/). Aqui, apenas o que um leitor de produto precisa entender:

| Decisão | Trade-off aceito |
| --- | --- |
| **Entrega assíncrona, não síncrona** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) | Ganhamos que um cliente lento ou fora do ar não trava a operação de pedidos de ninguém. Pagamos com a impossibilidade de confirmar a entrega no mesmo instante da mudança de status. Bruno (`[09:04]`): *"qualquer cliente lento vai travar mudança de status pra outros pedidos"* |
| **Usar o banco existente em vez de uma fila dedicada** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) | Ganhamos tempo de entrega e zero custo operacional novo. Pagamos com um teto de escala menor. Diego (`[09:07]`): *"a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering"* |
| **Verificação a cada 2 segundos em vez de disparo imediato** ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)) | Ganhamos simplicidade. Pagamos com até 2 segundos de latência mínima — irrelevante frente aos 10 segundos aceitos. Larissa (`[09:10]`): *"A latência mínima vai ser 2 segundos no pior caso. Aceitamos"* |
| **5 tentativas ao longo de ~15h** ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)) | Ganhamos cobertura de indisponibilidades reais de cliente. Pagamos com eventos que podem chegar muito depois do fato. Marcos aceitou (`[09:17]`): *"Se um cliente meu cair por 15 horas, ele já tá com problema sério dele"* |
| **At-least-once com dedup do cliente** ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)) | Ganhamos que nenhum evento se perde e a integração continua simples. Pagamos transferindo uma responsabilidade ao cliente. Sofia apontou (`[09:25]`): *"Isso joga responsabilidade pro cliente."* Diego: *"Joga, mas é o padrão de mercado. Stripe faz assim, GitHub faz assim"* |
| **Secret por endpoint com rotação** ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)) | Ganhamos contenção total do estrago em caso de vazamento. Pagamos com mais complexidade de gestão e com o armazenamento de material sensível |
| **Payload enxuto, sem itens do pedido** ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md)) | Ganhamos entregas leves e rápidas. Pagamos com uma consulta adicional para o cliente que precisa de detalhe. Diego (`[09:43]`): *"Se o cliente quiser detalhes, ele bate no GET /orders/:id depois"* |

## 9. Dependências

### Internas

| Dependência | Natureza | Observação |
| --- | --- | --- |
| Módulo de pedidos | Bloqueante | A geração do evento acontece dentro da mudança de status. É a alteração mais sensível da feature (`[09:40] Bruno`) |
| Cadastro de clientes | Bloqueante | Cada webhook pertence a um cliente existente |
| Autenticação e roles | Bloqueante | O reprocessamento exige role `ADMIN` (`[09:36] Larissa`) |
| Banco de dados MySQL | Bloqueante | Serve tanto de armazenamento quanto de fila (`[09:08] Larissa`) |

### Externas

| Dependência | Natureza | Observação |
| --- | --- | --- |
| Endpoints HTTPS dos clientes | Fora do nosso controle | A disponibilidade deles determina a taxa de sucesso. É o que motiva toda a política de retry |
| Implementação de deduplicação pelo cliente | Fora do nosso controle | Sem ela, o cliente processa eventos repetidos (RNF-08) |

### De equipe e processo

| Dependência | Observação |
| --- | --- |
| **Revisão de segurança por Sofia antes do deploy** | Bloqueante e no caminho crítico. Sofia (`[09:46]`): *"Reservem pelo menos dois dias úteis pra eu revisar o código de segurança antes do deploy. HMAC e geração de secret eu quero olhar com calma"* |
| **Documentação no portal do desenvolvedor** | Sob responsabilidade de Marcos (`[09:26]`, `[09:40]`). Precisa estar pronta antes do onboarding do primeiro cliente, sobretudo a parte de deduplicação |
| **Capacidade de engenharia: três sprints** | Estimativa de Larissa (`[09:46]`), incluindo a revisão de segurança |

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| **R-01** | A geração do evento degrada a performance da mudança de status, que é a operação central do produto | Média | **Alto** — afeta todos os pedidos, inclusive de clientes sem webhook | Nenhum evento é gerado quando nenhum webhook assina aquele status (`[09:34] Bruno`); medir a latência de `PATCH /orders/:id/status` antes e depois (OBJ-05); ponto de atenção declarado na revisão técnica com Bruno e Diego |
| **R-02** | Vazamento de secret de webhook por log, backup ou dump | Baixa | **Alto** — permite forjar eventos para aquele cliente | Secret por endpoint contém o estrago a um único cliente (`[09:21] Sofia`); rotação disponível pela API com 24h de convivência; revisão de segurança dedicada antes do deploy (`[09:46] Sofia`) |
| **R-03** | Cliente não implementa deduplicação e processa eventos repetidos, com efeito colateral no negócio dele | Média | Médio | Documentação destacada no portal, assumida por Marcos (`[09:26]`); identificador de evento em local previsível e estável; validação junto aos três primeiros clientes no onboarding |
| **R-04** | O processo de entrega cai e ninguém percebe até um cliente reclamar | Média | **Alto** — silencioso, pode durar horas | Monitoramento da idade do evento pendente mais antigo, com alarme (ver [FDD](FDD.md#10-observabilidade)); rodar em processo separado evita que um problema na API derrube a entrega (`[09:11] Diego`) |
| **R-05** | Atraso na revisão de segurança compromete o prazo comercial com a Atlas | Média | Médio | Agendar os dois dias de Sofia dentro da terceira sprint, não depois; Larissa já incluiu a revisão na estimativa (`[09:47]`) |
| **R-06** | Um cliente com muitos pedidos ou endpoint lento atrasa as entregas dos demais | Média | Médio | Timeout de 10s por tentativa limita o dano; medir duração por endpoint; rate limiting fica como ponto em aberto para ser decidido com dado (FE-03, `[09:39] Diego`) |
| **R-07** | O volume de eventos armazenados cresce sem política de expurgo e degrada o desempenho ao longo do tempo | Alta (longo prazo) | Médio | Não degrada no dia 1; o desenho prevê índices adequados; a política de retenção precisa virar dívida técnica com dono e prazo, não um "depois" indefinido (FE-05, `[09:08] Diego`) |

## 11. Critérios de aceitação

A feature é considerada pronta quando:

| # | Critério |
| --- | --- |
| **CA-01** | Um cliente consegue cadastrar, listar, editar e remover webhooks pela API, e recebe a secret na criação |
| **CA-02** | Um webhook só recebe notificações dos status que assinou |
| **CA-03** | Uma mudança de status de pedido gera notificação para todos os webhooks ativos que assinam aquele status |
| **CA-04** | Se o registro do evento falhar, a mudança de status **não** é efetivada — não existe pedido que mudou de status sem evento correspondente |
| **CA-05** | A notificação chega ao cliente em menos de 10 segundos no caminho feliz (OBJ-01) |
| **CA-06** | Toda requisição enviada carrega assinatura verificável com a secret daquele endpoint, e um cliente com a secret correta consegue validá-la |
| **CA-07** | Uma URL `http` é recusada no cadastro |
| **CA-08** | Uma entrega que falha é retentada conforme a política de RNF-03 e RNF-04, e o evento não é perdido |
| **CA-09** | Esgotadas as tentativas, o evento aparece na lista de falhas permanentes com payload e motivo |
| **CA-10** | Um administrador consegue reprocessar um evento que falhou, e o reprocessamento fica registrado com o autor |
| **CA-11** | Um usuário sem perfil de administrador **não** consegue reprocessar |
| **CA-12** | O cliente consegue consultar o histórico das últimas 100 entregas de um webhook, com sucesso/falha, payload, resposta e tempo de resposta |
| **CA-13** | O cliente consegue rotacionar a secret, e a anterior continua funcionando por 24 horas |
| **CA-14** | Nenhuma secret aparece em resposta de listagem/consulta nem em log |
| **CA-15** | Uma sequência rápida de mudanças no mesmo pedido chega ao cliente na ordem correta |
| **CA-16** | O portal do desenvolvedor documenta o formato do payload, os headers, a verificação da assinatura e — de forma destacada — a necessidade de deduplicação |

Critérios técnicos verificáveis por teste automatizado estão detalhados no [FDD](FDD.md#13-critérios-de-aceite-técnicos).

## 12. Estratégia de testes e validação

### Antes do deploy

| Camada | O que valida |
| --- | --- |
| **Testes unitários** | Cálculo do backoff, geração e verificação da assinatura, validação de URL, filtro de status, cálculo do tamanho do payload |
| **Testes de integração** | A atomicidade entre evento e mudança de status (CA-04) é o teste mais importante da feature: forçar falha no registro do evento e verificar que o pedido **não** mudou de status |
| **Testes de contrato de API** | Todos os endpoints, incluindo códigos de erro, status HTTP e ausência da secret nas respostas (CA-14) |
| **Teste ponta a ponta com servidor de destino simulado** | Ciclo completo: mudança de status → geração → entrega → confirmação. Inclui os cenários de falha: destino retornando 500, destino lento estourando o timeout, destino recusando conexão |
| **Teste de ordenação** | Sequência `PAID → PROCESSING → SHIPPED` no mesmo pedido, verificando a ordem de recebimento (CA-15) |
| **Revisão de segurança** | Conduzida por Sofia, com foco em HMAC e geração de secret, com dois dias úteis reservados (`[09:46]`) |

Larissa incluiu explicitamente *"integração no order.service e testes ponta a ponta"* na estimativa de sprints (`[09:46]`).

### Validação em produção

1. **Onboarding faseado.** Começar com um cliente — a Atlas, que é a mais urgente — antes de habilitar os três.
2. **Observação da latência real** contra a meta de OBJ-01 durante as primeiras semanas.
3. **Acompanhamento da taxa de falhas permanentes** (OBJ-06) para distinguir problema nosso de indisponibilidade do cliente.
4. **Medição do polling residual** em `GET /orders` para verificar OBJ-02, que é o sinal de que o cliente efetivamente migrou de padrão de integração.
5. **Coleta de dado para as decisões adiadas.** A taxa de envio por endpoint alimenta a decisão sobre rate limiting (FE-03); o volume acumulado alimenta a decisão sobre retenção (FE-05).

### Confirmação com o cliente

Marcos assumiu a comunicação (`[09:47]`): *"Atlas vai gostar. Eu confirmo prazo com eles."* A validação de negócio se fecha quando os três clientes estiverem com webhook ativo e tiverem reduzido o polling (OBJ-02 e OBJ-04).
