# AGENTS.md: Projeto de Agente (Claude Agent SDK + TypeScript)

Este arquivo tem duas partes:

* **Parte 1: Regras gerais.** Valem para **qualquer** agente construído neste repositório. Não mude ao criar outro agente.
* **Parte 2: Agente atual (`agente-estudos`).** Descreve o agente que está sendo construído agora. Ao criar outro agente, **substitua só esta parte** (e a skill em `skills/`).

---

# Parte 1: Regras gerais

## Papel

Você é um engenheiro de software especialista em IA, fazendo pair programming com quem está desenvolvendo o agente. Explique **o que** está fazendo e **por quê**, em português do Brasil, de forma curta e objetiva.

## Stack

* **Claude Agent SDK** ([docs.claude.com/en/api/agent-sdk/overview](https://docs.claude.com/en/api/agent-sdk/overview)): pacote `@anthropic-ai/claude-agent-sdk` em TypeScript, com Node.js 20 ou superior e `tsx` para rodar sem etapa de build. Sem LangChain nem outro framework de agentes.
* **`query()`:** recebe `prompt` e `options` e devolve um fluxo assíncrono de mensagens. Para manter a conversa entre turnos, reaproveite o `session_id` com a opção `resume`.
* **Agent Skills** ([agentskills.io/specification](https://agentskills.io/specification)): o SDK descobre skills em `.claude/skills/<nome>/SKILL.md` quando `settingSources: ["project"]` está ativo. A divulgação é progressiva:
  * **L1:** `name` e `description` no frontmatter do `SKILL.md`, sempre no contexto.
  * **L2:** o corpo do `SKILL.md`.
  * **L3:** arquivos de `references/` e `assets/`, lidos sob demanda com a ferramenta `Read`.
* **Ferramentas de arquivo:** `Read` e `Write` nativas do SDK. Ambas recebem `file_path` **absoluto**.
* **Autenticação local:** `ANTHROPIC_API_KEY` e `MODEL` no arquivo `.env` do projeto do agente. A API Key vem do [Claude Console](https://console.anthropic.com/settings/keys).

## Regras de construção

* **Configuração:** `options` do `query()` com `cwd` no projeto do agente (caminho absoluto calculado a partir do `agente.ts`), `settingSources: ["project"]`, `skills: [...]`, `tools` (restringe as ferramentas disponíveis) e `allowedTools` (aprova sem perguntar). Não ponha `Skill` em `allowedTools`: está depreciado, use a opção `skills`. TypeScript em modo `strict`.
* **Sem shell:** não inclua `Bash`, `Edit` nem `WebFetch` em `tools`, a menos que o agente precise deles.
* **Restrição de caminhos:** `Read` e `Write` não ficam presos a uma pasta por padrão. Restrinja com um hook `PreToolUse` que devolve `permissionDecision: "deny"` com uma mensagem clara.
* **Divulgação progressiva:** **não** cole o conteúdo de `references/` ou `assets/` no `systemPrompt`.
* **Empacotamento da skill:** a fonte fica em `skills/<nome>/` e a cópia em `<agente>/.claude/skills/<nome>/`. A pasta precisa ter **exatamente** o nome do campo `name` do frontmatter. Ao alterar uma skill, mantenha as duas cópias sincronizadas (`diff -r` para conferir).
* **Modelo:** leia da variável `MODEL`, com um padrão no código. Não deixe o nome do modelo fixo.
* **System prompt dinâmico:** monte o `systemPrompt` a cada turno. Inclua a data de hoje (para entender "segunda que vem") e qualquer caminho absoluto de que o agente precise.
* **Chat no terminal:** use `node:readline/promises` e trate o fim da entrada (EOF) sem lançar erro.
* **Dependências:** `npm` dentro da pasta do agente, com o `package-lock.json` versionado.
* **Simplicidade:** prefira código simples e legível a abstrações sofisticadas. Confirme o entendimento do pedido antes de gerar muito código.
* **Segredos:** **nunca** versione o `.env` nem pastas com dados pessoais, e nunca escreva a API Key em código, README, testes ou logs.
* **Fora de escopo:** deploy, interface web pública e infraestrutura (Terraform, CI/CD), a menos que sejam pedidos explicitamente.

## Debug UI local

O SDK não traz uma interface visual como o `adk web`. Para depurar, construa uma página **só para uso local**, seguindo estas regras:

* **Arquivos:** `config.ts` (opções do agente e hook, com a função `montarOpcoes(sessionId?)`), `servidor.ts` (servidor HTTP) e `public/index.html` (página). O `agente.ts` (terminal) e o `servidor.ts` importam as mesmas opções do `config.ts`, sem duplicar configuração.
* **Servidor:** use só `node:http`, sem framework. Escute **apenas em `127.0.0.1`**, porque a página usa a API Key do `.env`. A porta vem de `PORT` (padrão 3000).
* **Rotas:** `GET /` serve a página, `GET /api/info` devolve o modelo e `POST /api/chat` recebe `{ mensagem, sessionId }`. Rejeite mensagem vazia com `400`.
* **Streaming:** `POST /api/chat` responde em **NDJSON** (`application/x-ndjson`), uma mensagem do SDK por linha, enviada assim que chega. Erros durante o `query()` viram uma linha `{ "type": "erro_servidor", "mensagem": "..." }`.
* **Página, duas colunas:**
  * **Conversa:** chat com envio por Enter (Shift+Enter quebra a linha) e botão "Nova conversa", que zera o `sessionId`.
  * **Eventos:** linha do tempo do turno, com um item expansível por evento. Cada item mostra um resumo e o JSON completo.
* **Eventos que a página deve mostrar:**
  * `system` com `subtype: "init"`: modelo, ferramentas e skills carregadas.
  * `assistant`: blocos `text` vão para o chat; `tool_use` (nome e argumentos) e `thinking` vão para os eventos.
  * `user` com blocos `tool_result`: resultado de cada ferramenta, ligado ao `tool_use` por `tool_use_id`. Com `is_error`, mostre como erro. É aqui que aparece a negação do hook, com o motivo.
  * `result`: número de passos, duração, `total_cost_usd` e uso de tokens.
  * Ignore os outros tipos de mensagem (por exemplo `rate_limit_event`).
* **Segurança da página:** escreva o conteúdo vindo do agente com `textContent`, nunca com `innerHTML`.
* **Visual:** suporte a tema claro e escuro (`prefers-color-scheme`), layout que empilha as colunas em telas estreitas e erros em destaque.
* **Script:** `npm run web` roda `tsx --env-file=.env servidor.ts`.

## Como validar

* Rode `npx tsc --noEmit`.
* Teste com chamadas reais: um pedido incompleto (gating), um completo, um ajuste e uma tentativa de escrever fora da pasta permitida.
* Rode o chat com a entrada vinda de um pipe (`printf '...\n' | npm start`) para automatizar os testes.
* Confira com `git status` que nada sensível foi para o git.
* Para a debug UI, suba o servidor e teste com `curl -N -X POST http://127.0.0.1:3000/api/chat -H 'Content-Type: application/json' -d '{"mensagem":"..."}'`. Confirme que os eventos chegam linha a linha e que a negação do hook aparece como `tool_result` com `is_error`.
* Abra a página num navegador e confira o layout, o scroll, o tema escuro e o botão "Nova conversa". Os testes por `curl` não cobrem isso.

## Armadilhas já encontradas

* O agente pode ler o `SKILL.md` direto com `Read` em vez de chamar a ferramenta `Skill`. O fluxo funciona assim, então não trate isso como erro.
* `tools` e `allowedTools` têm papéis diferentes: a primeira define o que existe, a segunda o que dispensa confirmação.
* O `resume` entre turnos **ainda não foi testado**. Valide num chat com pelo menos dois turnos.

---

# Parte 2: Agente atual (`agente-estudos`)

## Goal

Construir o **`agente-estudos`**: um agente movido a skills que monta planos de estudo personalizados a partir da skill `skills/plano-de-estudos/`. O plano inclui revisão espaçada, checkpoints de autoavaliação e uma agenda em CSV que pode ser importada no Google Agenda. O agente salva essa agenda com a ferramenta `Write` e roda localmente, num chat no terminal.

## Context

* **Ambiente:** a ferramenta de código (Claude Code, `agy` etc.) roda na raiz do repositório, e o projeto do agente fica em `./agente-estudos/`.
* **L3 da skill:** `references/workflow.md`, `references/tecnicas.md`, `references/salvar_agenda.md`, `assets/plano_template.md` e `assets/exemplo_plano.md`.
* **Arquivos:** o agente usa `Write` para gravar `plano-de-estudos.csv` e `Read` para conferir o arquivo antes de reescrever no replanejamento.
* **Regra de gating da skill:** sem **objetivo**, **prazo**, **disponibilidade** e **nível atual**, o agente pergunta só o que falta e não lê nenhum arquivo L3. Com tudo em mãos, ele segue o `workflow.md` até entregar o plano no formato do `plano_template.md` e salvar o CSV com `Write`.
* **`.env`** (em `agente-estudos/`):
  ```env
  ANTHROPIC_API_KEY=<sua-api-key>
  MODEL=claude-sonnet-5-5
  ```
* **Estrutura do projeto:**
  ```text
  agente-estudos/
  ├── package.json
  ├── tsconfig.json
  ├── .env                      # não versionado
  ├── config.ts                 # opções do agente e hook (compartilhado)
  ├── agente.ts                 # chat no terminal
  ├── servidor.ts               # debug UI local (http://127.0.0.1:3000)
  ├── public/index.html         # página da debug UI
  ├── .claude/skills/plano-de-estudos/   # cópia da skill
  └── planos/                   # CSVs gerados, não versionado
  ```
* **Execução local:**
  ```bash
  cd agente-estudos
  npm install
  npm start                 # chat no terminal
  npm run web               # debug UI no navegador
  ```

## Constraints

* **Skill:** `skills: ["plano-de-estudos"]`, `tools: ["Read", "Write"]` e `allowedTools: ["Read", "Write"]`.
* **Hook de caminhos:**
  * `Write` só pode gravar dentro de `agente-estudos/planos/`; qualquer outro caminho é negado com a mensagem `Escrita fora da pasta de planos`.
  * `Read` só pode ler dentro de `planos/` e de `.claude/skills/`.
* **Pasta de planos no prompt:** o `systemPrompt` informa o caminho absoluto de `planos/`, que o `salvar_agenda.md` usa para montar o `file_path` de `<pasta de planos>/plano-de-estudos.csv`.
* **Dados pessoais:** a pasta `planos/` guarda dados de agenda e nunca é versionada.

## Done-when

- [ ] `agente-estudos/agente.ts` chama `query()` com `settingSources: ["project"]`, `skills`, `tools: ["Read", "Write"]` e o hook de restrição de caminhos.
- [ ] O modelo vem da variável `MODEL`, e o `systemPrompt` contém a data de hoje e o caminho da pasta `planos/`.
- [ ] `npm install` e `npm start` rodam sem erros, `npx tsc --noEmit` passa, e o chat responde no terminal.
- [ ] **Gating:** com a mensagem "Quero organizar meus estudos de JavaScript.", o agente pede só as informações que faltam e **não** usa `Read` em `references/` nem `assets/`.
- [ ] **Plano completo:** com todas as informações, o agente lê `workflow.md`, `tecnicas.md`, `plano_template.md` e `salvar_agenda.md`, entrega módulos, cronograma e checkpoints e chama `Write`, criando `agente-estudos/planos/plano-de-estudos.csv` com um CSV válido para importação no Google Agenda.
- [ ] **Replanejamento:** quando o usuário informa sessões perdidas, o agente devolve a tabela atualizada e sobrescreve o `plano-de-estudos.csv`, sem empilhar o conteúdo atrasado em um único dia.
- [ ] **Restrição de caminho:** um pedido para gravar fora de `planos/` é negado pelo hook.
- [ ] **Debug UI:** `npm run web` abre a página em `127.0.0.1`, mostra o chat e, ao lado, cada chamada de ferramenta com argumentos e resultado, incluindo a negação do hook.
- [ ] **Conversa em dois turnos:** o agente mantém o contexto entre duas perguntas seguidas (`resume`).
- [ ] Nenhum `.env`, API Key ou arquivo de `planos/` aparece em `git status` nem em arquivos versionados.
