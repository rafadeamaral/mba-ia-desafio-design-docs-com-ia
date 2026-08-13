# Da Reunião ao Documento: Design Docs Gerados por IA

Este repositório é a entrega do desafio **"Da Reunião ao Documento"** do MBA. O conteúdo original do enunciado foi substituído pela documentação do processo de produção, conforme pedido no requisito 6.

📄 O enunciado original está preservado em [`docs/ENUNCIADO.md`](docs/ENUNCIADO.md).

---

## Sobre o desafio

O ponto de partida é um Order Management System real em Node.js + TypeScript, com Prisma sobre MySQL, módulos de autenticação, usuários, clientes, produtos e pedidos — e **nenhum** mecanismo de notificação externa, evento ou fila. Esse vácuo é o assunto de uma reunião técnica de 55 minutos entre tech lead, PM, dois engenheiros e uma engenheira de segurança, em que o time decide como construir um sistema de webhooks de notificação de mudança de status de pedido. Nada dessa reunião foi registrado além da transcrição literal em [`TRANSCRICAO.md`](TRANSCRICAO.md).

A tarefa é transformar essa transcrição, mais a leitura do código existente, em um pacote de documentação técnica acionável: PRD, RFC, FDD, sete ADRs e um tracker de rastreabilidade. A restrição que define o exercício é que **nada pode ser inventado** — todo requisito, decisão e restrição precisa ter origem identificável em uma fala da reunião ou em um arquivo do repositório. E a contrapartida, igualmente importante: o que a reunião **descartou ou adiou** não pode reaparecer como requisito. A reunião rejeita disparo síncrono, Redis Streams, trigger de banco, exactly-once, 3 tentativas de retry e DLQ como flag; e adia e-mail de aviso, dashboard, rate limiting, arquivamento da outbox e múltiplos workers. Reconhecer essa fronteira é metade do trabalho.

O papel do aluno aqui não é escrever os documentos à mão, e sim conduzir a IA: decidir o que produzir, formular os prompts, revisar criticamente cada entrega e corrigir até o resultado ficar consistente. A entrega é puramente documental — nenhum arquivo em `src/`, `prisma/` ou `tests/` foi alterado.

## Ferramentas de IA utilizadas

| Ferramenta | Papel no processo |
| --- | --- |
| **Claude Code (Claude Opus 5)** | Ferramenta principal. Usada em sessão de terminal com acesso direto ao repositório: leitura da transcrição, mapeamento do código-fonte, redação dos documentos e execução dos scripts de verificação. O acesso ao filesystem foi decisivo — em vez de descrever o código para a IA, ela leu os arquivos e citou linha e caminho reais |
| **Ferramentas de busca do Claude Code** (`Grep`, `Glob`, `Read`) | Mapeamento dos pontos de integração. `Glob **/*.ts` para o inventário de arquivos, `Grep requireRole` para achar o middleware e seu uso real, `Read` para os 15 arquivos que viraram referência nos documentos |
| **Bash / scripts de verificação** | O contraponto à IA. Scripts que validam mecanicamente as afirmações dos documentos contra a transcrição e o repositório (ver "Iterações e ajustes", item 6) |

## Workflow adotado

A ordem de produção foi deliberada: **das decisões para cima**, não do documento mais alto para baixo.

```
1. Leitura integral          TRANSCRICAO.md (324 linhas) + README do enunciado
        │
2. Mapeamento do código      Glob → inventário
        │                    Read → 15 arquivos: order.service.ts, order.status.ts,
        │                           app-error.ts, http-errors.ts, error.middleware.ts,
        │                           auth.middleware.ts, validate.middleware.ts,
        │                           request-logger.middleware.ts, logger/index.ts,
        │                           response.ts, env.ts, database.ts, server.ts,
        │                           app.ts, routes/index.ts, schema.prisma, package.json
        │
3. Triagem da transcrição    Decidido / Descartado / Adiado / Em aberto / Ruído
        │
4. ADRs (7)                  As decisões viram o esqueleto de tudo
        │
5. RFC                       Consolida as decisões + alternativas + questões em aberto
        │
6. FDD                       Detalha a implementação em cima das decisões fechadas
        │
7. PRD                       Por último entre os grandes: com o resto pronto, é consolidação
        │
8. TRACKER                   Varredura dos documentos prontos, item por item
        │
9. Verificação mecânica      Scripts validando timestamps e caminhos de arquivo
        │
10. README do processo
```

**Por que ADRs primeiro.** As seis decisões principais da reunião são o esqueleto da feature. Escritas primeiro, elas viram a fonte única de verdade para os documentos seguintes: o RFC as consolida, o FDD parte delas como premissa e o PRD as traduz para linguagem de negócio. Produzir na ordem inversa produziria três descrições concorrentes da mesma decisão.

**Como a fronteira entre documentos foi mantida.** A regra aplicada foi de **altura**: o PRD fala em cliente e métrica de negócio, o RFC em abordagem e trade-off arquitetural, o ADR em uma decisão isolada com suas consequências, o FDD em payload, índice e assinatura de função. Quando um trecho aparecia em dois documentos, ele foi cortado do mais alto. Exemplo concreto: os endpoints estão **só** no FDD; o RFC diz "sete endpoints, contratos no FDD" e linka. A política de retry aparece como número em três documentos, mas com alturas diferentes — o PRD registra "retentativa automática com intervalos crescentes" como requisito, o ADR-003 justifica por que 5 e não 3, e o FDD dá a tabela de momentos exatos de cada tentativa.

**Como a interação foi organizada.** Uma sessão contínua, com o contexto do código e da transcrição carregado desde o início e mantido ao longo de toda a produção. Isso importa: o FDD conseguiu citar `src/shared/errors/http-errors.ts:27` e apontar um problema real na herança de `NotFoundError` porque o arquivo tinha sido lido de fato, não descrito.

## Prompts customizados

### Prompt 1 — Triagem da transcrição em cinco baldes

O prompt genérico ("gere um PRD a partir dessa transcrição") produz documento vazio porque trata toda fala como requisito. Este prompt força a IA a classificar antes de escrever, e a **separar o que não entra**:

```
Leia TRANSCRICAO.md integralmente antes de responder.

Classifique CADA fala com conteúdo técnico ou de produto em exatamente um destes cinco baldes:

  DECIDIDO   — o time fechou. Existe uma fala de ratificação explícita.
  DESCARTADO — foi proposto e rejeitado. Registre QUEM propôs e QUAL o
               argumento que matou a proposta.
  ADIADO     — reconhecido como válido, mas colocado fora desta fase.
               Registre a justificativa e, se houver, a condição de retomada.
  EM ABERTO  — levantado e não resolvido. Ninguém decidiu nem descartou.
  RUÍDO      — logística de call, saudação, explicação didática. Descarte.

Para cada item, produza: timestamp exato no formato [hh:mm], nome do falante,
citação literal do trecho decisivo, e uma frase sua resumindo.

REGRAS:
- Não agregue itens de falantes diferentes na mesma linha.
- Uma decisão ratificada por outra pessoa depois gera DUAS linhas
  (a proposta e a ratificação), não uma.
- Se você não conseguir apontar o timestamp, o item não existe. Não inclua.
- Ao final, liste separadamente tudo que caiu em DESCARTADO e ADIADO. Essa
  lista é o gabarito do "fora de escopo" e não pode aparecer como requisito
  em nenhum documento posterior.
```

### Prompt 2 — Ancoragem no código, com verificação obrigatória

Este prompt ataca o modo de falha mais comum em design doc gerado por IA: citar arquivo, classe ou método plausível que não existe no repositório.

```
Você vai escrever a seção "Integração com o sistema existente" do FDD.

ANTES de escrever qualquer linha: use Read para abrir cada arquivo que você
pretende citar. Não escreva sobre arquivo que você não abriu nesta sessão.

Para cada ponto de integração, produza:
  1. O caminho real do arquivo e o número da linha do trecho relevante.
  2. O que aquele código faz HOJE, descrito a partir do que você leu —
     não a partir do que o nome do arquivo sugere.
  3. O que muda, com o diff conceitual mínimo.
  4. A fala da reunião ([hh:mm] Nome) que justifica a mudança.

PROIBIDO:
- Citar arquivo, classe, método ou campo que você não viu no repositório.
- Descrever comportamento "provável" de um trecho. Se precisou supor, leia.
- Inventar arquivos novos sem marcá-los explicitamente como A CRIAR.

OBRIGATÓRIO — sinalize ativamente as fricções, não as suavize:
Se um padrão existente NÃO acomodar bem o que a reunião decidiu, diga isso
em vez de fingir encaixe. Exemplos do tipo de coisa que quero ver:
"a classe X fixa o código de erro e não aceita override, então Y precisa
estender Z diretamente"; "a lista de campos redigidos no logger não cobre
o campo W, e isso vaza dado sensível".
```

### Prompt 3 — Auditoria adversarial antes de fechar

Rodado sobre os documentos já escritos, invertendo o papel da IA de autora para crítica:

```
Assuma o papel de revisor cético. Sua tarefa NÃO é elogiar nem melhorar a
escrita — é encontrar afirmação sem lastro.

Varra docs/PRD.md, docs/RFC.md, docs/FDD.md e docs/adrs/*.md e liste:

  A) Toda afirmação factual (número, prazo, limite, nome de campo, caminho
     de arquivo, código de erro) que NÃO consiga ser rastreada a um
     timestamp da transcrição ou a um arquivo do repositório.
  B) Todo item que apareça como requisito e que na verdade tenha sido
     DESCARTADO ou ADIADO na reunião.
  C) Toda contradição entre dois documentos do pacote, ou entre um documento
     e a transcrição.
  D) Todo trecho duplicado entre dois documentos — indicando de qual dos
     dois ele deve ser removido, pela regra de altura.

Para cada achado: cite o trecho, diga por que é problema e proponha a
correção (reescrever, mover de documento, marcar como premissa a ratificar,
ou remover).

Não invente achados para parecer produtivo. Se uma seção está bem ancorada,
diga que está e siga.
```

## Iterações e ajustes

Foram **quatro passadas principais** sobre o material — triagem, redação, auditoria crítica e verificação mecânica. Os ajustes mais relevantes:

**1. A primeira versão do payload inventava campos.** A transcrição diz, em `[09:43] Diego`: *"order_id, order_number, from_status, to_status, customer_id, e os campos básicos da order tipo total_cents"*. A geração inicial expandiu "campos básicos" para `subtotal_cents`, `discount_cents`, `notes` e `created_at` — plausível, todos existem no modelo `Order`, e **nenhum foi dito na reunião**. Correção: o exemplo do FDD ficou restrito a `total_cents`, que é o único nomeado, com uma nota explícita de que a inclusão dos demais é decisão de implementação a ratificar. É exatamente o caso em que preencher a coluna "Localização" do tracker se torna impossível e denuncia a invenção.

**2. `WEBHOOK_SECRET_REQUIRED` não fecha com o resto da reunião.** Bruno nomeia esse código literalmente em `[09:28]`. Mas Marcos diz em `[09:31]` que *"secret é gerada pela gente e devolvida na criação"* — ou seja, o cliente nunca informa uma secret, e o significado óbvio do código ("faltou a secret no request") é impossível. A primeira versão do FDD documentou o código com essa semântica óbvia e errada. Correção: o código foi mantido, com a única semântica coerente com as demais decisões (guarda de integridade no momento de assinar uma entrega de endpoint sem secret ativa) e uma nota pedindo confirmação da intenção original na revisão. Preferível a escolher em silêncio ou a omitir um código que foi explicitamente citado.

**3. `WebhookNotFoundError` não pode estender `NotFoundError`.** Escrever "os erros do módulo estendem a hierarquia existente" era fácil e soava certo. Ler `src/shared/errors/http-errors.ts:27` mostrou que `NotFoundError` fixa o código como `'NOT_FOUND'` e não aceita override — então estendê-la produziria `NOT_FOUND`, e não o `WEBHOOK_NOT_FOUND` decidido em `[09:28]` e `[09:29]`. Correção: o FDD passou a apontar o problema e a instruir herança direta de `AppError`. Esse achado só existe porque o arquivo foi aberto; nenhuma leitura da transcrição levaria a ele.

**4. Um risco de segurança que a reunião não viu.** Sofia decidiu secret por endpoint e Bruno decidiu reusar o logger Pino "sem botar nada novo" (`[09:29]`). As duas decisões são boas isoladamente e se chocam na prática: `src/shared/logger/index.ts:4` redige `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`, mas **não** `*.secret`. Reusar o logger como está vaza secrets de webhook em log. Correção: registrado como a única alteração obrigatória em arquivo compartilhado (FDD §11.5), como consequência negativa do ADR-006 e como risco R-02 no PRD. Achado do código, não da transcrição.

**5. A garantia de ordering se contradizia entre dois ADRs.** O ADR-002 afirma que single-worker preserva a ordem por `order_id`. O ADR-003 estabelece backoff de até 12 horas. Juntos: se o evento A entra em backoff e o evento B do mesmo pedido é entregue no minuto seguinte, o cliente recebe B antes de A — a garantia do ADR-002 não sobrevive ao mecanismo do ADR-003. A primeira versão afirmava a garantia sem ressalva nos dois documentos. Correção: a limitação virou consequência negativa explícita do ADR-003, e o ADR-002 passou a qualificar a garantia como válida no caminho feliz.

**6. Verificação mecânica em vez de confiança.** Antes de fechar, dois scripts foram rodados contra os documentos prontos:

```bash
# Toda referência [hh:mm] Nome do tracker existe mesmo na transcrição?
grep -o '`\[09:[0-9][0-9]\] [A-Za-zí]*`' docs/TRACKER.md | tr -d '`' | sort -u \
  | while read -r r; do grep -qF "$r" TRANSCRICAO.md || echo "FALTA: $r"; done

# Todo caminho de arquivo citado nos documentos existe mesmo no repositório?
grep -ohE '(src|prisma|tests)/[A-Za-z0-9._/-]+\.(ts|prisma)' docs/*.md docs/adrs/*.md \
  | sort -u | while read -r p; do [ -f "$p" ] || echo "NAO EXISTE: $p"; done
```

O primeiro validou as 85 referências distintas a falas — todas conferem. O segundo apontou seis caminhos inexistentes; todos eram arquivos que a feature vai **criar** (`src/worker.ts`, `src/modules/webhooks/*`, `tests/webhooks.test.ts`), não citações falsas. Ajuste feito: uma convenção de leitura no início da §11 do FDD separando explicitamente "existe hoje" de "a criar", para que a distinção não dependa de o leitor inferir.

**7. Corte de duplicação entre RFC e FDD.** A primeira versão do RFC trazia payloads de exemplo e a matriz de erros — conteúdo que pertence ao FDD e que estourava a orientação de 2 a 4 páginas. Correção: o RFC passou a apenas nomear a superfície ("sete endpoints, todos autenticados") e linkar. O mesmo tratamento foi dado às consequências das decisões, que ficaram nos ADRs em vez de repetidas no RFC.

**Uma decisão de método que vale registrar.** Vários pontos da reunião ficaram genuinamente sem resposta. A tentação é a IA escolher uma e seguir, e o resultado parece mais completo. Preferimos o oposto: as cinco questões em aberto viraram uma seção nomeada do RFC, cada uma com o timestamp em que foi levantada e um encaminhamento sugerido. A mais consequente é a Q4 — Larissa deixa em aberto, em `[09:32]`, se o `customer_id` vai no body ou no path. O FDD assume body, mas marca a premissa como pendente de ratificação e diz quais contratos mudam se a revisão decidir o contrário. Um documento que esconde o que não foi decidido é pior que um que declara a pendência.

## Como navegar a entrega

```
.
├── README.md                    ← este documento (processo de produção)
├── TRANSCRICAO.md               ← fonte primária, não alterada
├── docs/
│   ├── ENUNCIADO.md             ← enunciado original do desafio
│   ├── PRD.md                   ← por que e o quê (produto/negócio)
│   ├── RFC.md                   ← como propomos resolver (arquitetura)
│   ├── FDD.md                   ← como construir (implementação)
│   ├── TRACKER.md               ← de onde veio cada coisa (transversal)
│   └── adrs/
│       ├── README.md            ← índice dos ADRs
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-worker-separado-em-polling.md
│       ├── ADR-003-retry-com-backoff-e-dlq.md
│       ├── ADR-004-hmac-sha256-com-secret-por-endpoint.md
│       ├── ADR-005-at-least-once-com-x-event-id.md
│       ├── ADR-006-reuso-dos-padroes-do-projeto.md
│       └── ADR-007-snapshot-do-payload-na-outbox.md
└── src/ prisma/ tests/          ← código do OMS, não alterado
```

### Ordem de leitura sugerida

| # | Documento | Por quê |
| --- | --- | --- |
| 1 | [`docs/PRD.md`](docs/PRD.md) | Entra pelo problema: quem pediu, por quê, o que entra e o que ficou de fora |
| 2 | [`docs/RFC.md`](docs/RFC.md) | A proposta técnica em uma página de altura. O TL;DR da §1 dá o desenho inteiro |
| 3 | [`docs/adrs/`](docs/adrs/) | Cada decisão isolada, com a alternativa descartada e o trade-off. O [índice](docs/adrs/README.md) diz qual ADR responde qual pergunta |
| 4 | [`docs/FDD.md`](docs/FDD.md) | O detalhe de implementação. A §11 "Integração com o sistema existente" é a que amarra tudo ao código real |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | A auditoria. Serve para checar qualquer afirmação dos anteriores contra a fonte |

### Atalhos por interesse

- **Quero ver o encaixe no código:** [FDD §11](docs/FDD.md#11-integração-com-o-sistema-existente) e [ADR-006](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md).
- **Quero ver o que a reunião rejeitou:** [RFC §4](docs/RFC.md#4-alternativas-consideradas) para as alternativas descartadas, [PRD §5.2](docs/PRD.md#52-fora-de-escopo) para o que ficou fora.
- **Quero ver o que ainda está indefinido:** [RFC §5](docs/RFC.md#5-questões-em-aberto).
- **Quero conferir se algo foi inventado:** [`docs/TRACKER.md`](docs/TRACKER.md) — 228 itens, cada um com timestamp ou caminho de arquivo.

## Validação contra os critérios de aceite

O enunciado original está preservado em [`docs/ENUNCIADO.md`](docs/ENUNCIADO.md) **sem alterações**, inclusive com as caixas de seleção vazias — ele é a referência do que foi pedido, não o registro do que foi feito. Este é o registro.

### PRD — [`docs/PRD.md`](docs/PRD.md)

- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias do requisito 1 — **12 de 12**
- [x] Identifica no mínimo 8 requisitos funcionais — **15** (RF-01 a RF-15)
- [x] Pelo menos 1 objetivo com métrica e meta quantitativa — **6** (OBJ-01: latência p95 < 10s; OBJ-02: queda ≥ 80% no polling; OBJ-06: DLQ < 1%/mês)
- [x] "Fora de escopo" com pelo menos 2 itens descartados ou adiados — **7** (FE-01 a FE-07)
- [x] "Riscos" com pelo menos 2 riscos com probabilidade, impacto e mitigação — **7** (R-01 a R-07)

### RFC — [`docs/RFC.md`](docs/RFC.md)

- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias do requisito 2 — **8**, mais os metadados no cabeçalho
- [x] "Alternativas consideradas" com pelo menos 2 descartadas, cada uma com o trade-off — **6**
- [x] "Questões em aberto" com pelo menos 2 pontos adiados ou não decididos — **5** (Q1 a Q5)
- [x] Referencia com link pelo menos 2 ADRs — **7**
- [x] Conciso, 2 a 4 páginas — **2.242 palavras**

### FDD — [`docs/FDD.md`](docs/FDD.md)

- [x] Arquivo existe e está em Markdown
- [x] Contém todas as seções obrigatórias do requisito 3 — **14 de 14**
- [x] "Contratos públicos" com pelo menos 4 endpoints HTTP com payload de exemplo (request e response) e status codes — **7 de 7 endpoints completos**
- [x] Matriz de erros com prefixo `WEBHOOK_` — **19 códigos**
- [x] "Integração com o sistema existente" referencia pelo menos 4 caminhos de arquivo reais — **16 arquivos existentes**, em 12 pontos de integração
- [x] "Observabilidade" cita métricas, logs e tracing — as três subseções

### ADRs — [`docs/adrs/`](docs/adrs/)

- [x] Entre 5 e 8 arquivos no formato `ADR-NNN-titulo-em-kebab-case.md` — **7**
- [x] Cada ADR com Status, Contexto, Decisão, Alternativas Consideradas, Consequências — **7 de 7**
- [x] Cobre pelo menos 5 das 6 decisões principais — **as 6**, mais o snapshot do payload (ADR-007)
- [x] Pelo menos 1 ADR referencia arquivos, módulos ou classes do código base — **6 dos 7**; o [ADR-006](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md) é integralmente sobre isso

### Tracker — [`docs/TRACKER.md`](docs/TRACKER.md)

- [x] Segue o formato de tabela do requisito 5 — colunas ID, Documento, Tipo, Conteúdo, Fonte, Localização
- [x] Pelo menos 80% dos itens identificáveis têm linha correspondente — **228 itens**; o critério de contagem está declarado no topo do arquivo
- [x] Pelo menos 70% das linhas com Fonte = `TRANSCRICAO` e timestamp válido — **80%** (182 linhas); as **85 referências distintas foram verificadas** contra `TRANSCRICAO.md`
- [x] Pelo menos 5 linhas com Fonte = `CODIGO` e caminho real — **46**

### README — [`README.md`](README.md)

- [x] Contém todas as seções obrigatórias do requisito 6 — **6 de 6**
- [x] Lista pelo menos 1 ferramenta de IA — **3**
- [x] Mostra pelo menos 2 prompts customizados em blocos de código — **3**
- [x] Descreve pelo menos 2 iterações ou ajustes concretos — **7**

### Consistência geral

- [x] Nenhum requisito, decisão ou restrição contradiz a transcrição ou o código
- [x] Nenhum arquivo de código mencionado é inexistente no repositório — os 6 caminhos que ainda não existem são os arquivos **a criar** pela feature, identificados como tais na convenção de leitura do início da §11 do FDD
- [x] Nenhuma alteração em `src/`, `prisma/`, `tests/` ou configurações

### Como reproduzir a verificação

Os itens que dependem de contagem ou de conferência contra as fontes foram checados por script, não a olho:

```bash
# 1. Toda referência [hh:mm] Nome do tracker existe na transcrição?
grep -o '`\[09:[0-9][0-9]\] [A-Za-zí]*`' docs/TRACKER.md | tr -d '`' | sort -u \
  | while read -r r; do grep -qF "$r" TRANSCRICAO.md || echo "FALTA: $r"; done

# 2. Todo caminho de arquivo citado existe no repositório?
grep -ohE '(src|prisma|tests)/[A-Za-z0-9._/-]+\.(ts|prisma)' docs/*.md docs/adrs/*.md \
  | sort -u | while read -r p; do [ -f "$p" ] || echo "A CRIAR: $p"; done

# 3. Cada endpoint do FDD tem request, response e tabela de status codes?
for n in 1 2 3 4 5 6 7; do
  sec=$(sed -n "/^### 6\.$n /,/^### 6\.$((n+1)) \|^## 7\./p" docs/FDD.md)
  echo "6.$n -> Request=$(echo "$sec" | grep -c '\*\*Request\*\*')" \
       "Response=$(echo "$sec" | grep -c '\*\*Response `')" \
       "Status=$(echo "$sec" | grep -c '^| Status | Quando |')"
done

# 4. Nenhum link markdown interno quebrado?
for f in README.md docs/*.md docs/adrs/*.md; do d=$(dirname "$f")
  grep -oE '\]\(([^)#]+\.md)(#[^)]*)?\)' "$f" | sed -E 's/^\]\(//; s/\)$//; s/#.*$//' \
    | sort -u | while read -r l; do [ -e "$d/$l" ] || echo "QUEBRADO: $f -> $l"; done
done
```

Foi a rodada 3 desses scripts que revelou que apenas **1** dos 7 endpoints tinha request *e* response de exemplo, contra os 4 exigidos pelo critério. A correção está no commit `e51a744`.
