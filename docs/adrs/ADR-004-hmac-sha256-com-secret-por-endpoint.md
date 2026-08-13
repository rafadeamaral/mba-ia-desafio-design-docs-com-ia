# ADR-004 — Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

- **Status:** Aceito
- **Data:** 2026-08-13
- **Decisores:** Sofia (Eng. Segurança), Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos)
- **Origem:** Reunião técnica `TRANSCRICAO.md`, blocos `[09:19]` a `[09:24]` e `[09:44]`
- **ADRs relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)

---

## Contexto

Sofia abriu o bloco de segurança com o enquadramento do problema (`[09:19]`): *"a gente tá expondo eventos com dados de pedidos pra um endpoint fora da nossa infra. O cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio."*

São duas propriedades distintas a garantir no destino: **autenticidade** (a requisição partiu de nós) e **integridade** (o corpo não foi alterado em trânsito). E há um terceiro problema, de blast radius: se o material criptográfico for compartilhado entre clientes, o vazamento de um compromete todos.

Diego trouxe o precedente concreto que motivou a rotação (`[09:22]`): *"A gente já teve cliente que vazou secret em log de aplicação dele uma vez."*

## Decisão

**Assinatura HMAC-SHA256 sobre o corpo do request, com secret única por endpoint de webhook e suporte a rotação com grace period de 24 horas** (`[09:22] Sofia`).

Os quatro elementos da decisão:

1. **Algoritmo: HMAC-SHA256.** Bruno perguntou qual algoritmo e Sofia respondeu (`[09:20]`): *"SHA-256. HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso."* O critério não foi força criptográfica bruta, e sim disponibilidade de implementação do lado do cliente.

2. **Transporte da assinatura: header `X-Signature`.** *"A gente assina o payload com uma secret compartilhada entre nós e o cliente, manda a assinatura num header tipo `X-Signature`. Cliente verifica do lado dele"* (`[09:20] Sofia`). Junto vai `X-Timestamp` com o timestamp do envio, para que o cliente possa detectar replay attack se quiser (`[09:44] Diego`).

3. **Escopo da secret: por endpoint, não global.** Sofia (`[09:21]`): *"cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo."* A secret é gerada por nós e devolvida ao cliente na criação do webhook (`[09:31] Marcos`); o registro de configuração guarda `url + secret + customer_id + estado ativo` (`[09:21] Bruno`).

4. **Rotação com janela de sobreposição de 24h.** Sofia (`[09:21]`): *"a secret tem que ser rotacionável. Endpoint pro cliente conseguir pedir nova secret pela API. Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele. Depois disso, a antiga morre."*

**Escopo desta decisão.** A validação de TLS obrigatório (URL tem que ser `https`, `http` é recusado) foi discutida no mesmo bloco, mas Sofia mesma classificou como não-arquitetural (`[09:23]`): *"nem é decisão arquitetural, é só uma validação no schema Zod."* O mesmo vale para o teto de 64KB de payload, que Larissa classificou como requisito não funcional e não como decisão separada (`[09:24]`). Ambos estão especificados no FDD, não aqui.

## Alternativas Consideradas

### 1. Secret global da plataforma

Uma única secret nossa, usada para assinar tudo o que sai para todos os clientes.

**Descartada.** É a alternativa que Sofia nomeou e rejeitou na mesma frase (`[09:21]`): *"Senão se vaza uma, vaza tudo."* O trade-off é operacional — uma secret global seria trivial de armazenar e rotacionar de um lado só, sem tabela, sem endpoint de rotação, sem grace period. O custo é um blast radius total: o vazamento no log de aplicação de um cliente (cenário que já ocorreu, `[09:22] Diego`) permitiria forjar eventos para todos os demais.

### 2. Rotação sem grace period (corte seco)

Trocar a secret e invalidar a anterior imediatamente.

**Descartada por implicar downtime de integração no cliente.** É a razão explícita da janela de 24h (`[09:21] Sofia`): dar ao cliente tempo de migrar os sistemas dele. O trade-off aceito é que, durante 24 horas, existem **duas** secrets válidas para o mesmo endpoint — uma superfície de ataque temporariamente dobrada, em troca de rotação sem quebra. Sem a janela, o incentivo prático seria o cliente nunca rotacionar.

### 3. mTLS ou assinatura assimétrica

Não foi levantada na reunião; registrada como alternativa plausível e não adotada. Ambas eliminariam o segredo compartilhado, mas exigiriam gestão de certificados ou de chaves públicas do lado do cliente. Isso colide frontalmente com o critério que Sofia usou para escolher HMAC-SHA256: *"todo cliente sério tem biblioteca pra isso"* (`[09:20]`). Para integrações B2B com clientes de maturidade técnica variada, a barreira de adoção seria alta demais.

## Consequências

### Positivas

- **Autenticidade e integridade verificáveis no destino** sem que o cliente precise de nenhuma infraestrutura além de uma função de HMAC da biblioteca padrão.
- **Blast radius contido.** O comprometimento da secret de um endpoint não afeta nenhum outro cliente nem nenhum outro endpoint do mesmo cliente.
- **Resposta a incidente sem interrupção.** Cliente que vazou a secret rotaciona pela API e migra em até 24 horas, sem perder entregas nesse intervalo.
- **Baixo atrito de adoção**, que é o que faz a proteção efetivamente ser usada pelos clientes em vez de ignorada.
- **Defesa opcional contra replay.** O `X-Timestamp` permite ao cliente rejeitar requisições antigas se quiser (`[09:44] Diego`), sem impor essa complexidade a quem não quer.

### Negativas

- **Passamos a armazenar material criptográfico sensível.** A tabela de configuração de webhooks guarda secrets, o que cria uma superfície nova: a proteção precisa alcançar backups, dumps e logs. O logger Pino já tem lista de `redactPaths` em `src/shared/logger/index.ts:4` (que cobre `*.password`, `*.token`, `*.accessToken`) — a lista precisará ser estendida para os campos de secret de webhook.
- **A verificação depende inteiramente do cliente.** Nada nos garante que o cliente confere a assinatura; se ele ignorar o `X-Signature`, a proteção não existe na prática. Cabe a Marcos documentar isso no portal do desenvolvedor (`[09:26]`, `[09:40]`).
- **Complexidade permanente no caminho de verificação.** Durante o grace period, a lógica de assinatura precisa lidar com duas secrets simultâneas, e o expurgo da antiga após 24h precisa de mecanismo próprio.
- **Bloqueio de deploy por revisão de segurança.** Sofia condicionou a subida (`[09:46]`): *"Reservem pelo menos dois dias úteis pra eu revisar o código de segurança antes do deploy. HMAC e geração de secret eu quero olhar com calma."* Isso é caminho crítico no cronograma de três sprints.
- **A secret é exibida uma única vez, na criação** (`[09:31] Marcos`), o que exige tratamento cuidadoso na resposta HTTP e no portal — um erro de manuseio do cliente nesse momento força uma rotação imediata.
