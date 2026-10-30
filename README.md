# 🧠 Projeto de Agente: Agente de Estudos com Claude Agent SDK

Um agente de estudos feito com o **Claude Agent SDK** (TypeScript). Você diz o que quer aprender e até quando, e ele monta um plano com revisão espaçada e checkpoints de autoavaliação. No final, salva uma agenda em CSV para importar no Google Agenda.

O comportamento vem da skill [`plano-de-estudos`](skills/plano-de-estudos/SKILL.md). As regras do projeto estão no [AGENTS.md](AGENTS.md).

## Como funciona

- Se faltar **objetivo**, **prazo**, **disponibilidade** ou **nível atual**, o agente pergunta só o que falta.
- Com tudo em mãos, ele segue o `workflow.md` da skill, entrega módulos, cronograma e checkpoints, e grava `planos/plano-de-estudos.csv` com a ferramenta `Write`.
- Se você perder sessões, ele replaneja e sobrescreve o CSV.
- O agente só lê e escreve em `planos/` e nas pastas das skills. Não tem acesso ao shell.

## Pré-requisitos

- [Node.js](https://nodejs.org/) 20 ou superior
- Uma chave de API da Anthropic, criada em [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys)
- Uma ferramenta de código com IA para construir o agente a partir do AGENTS.md (por exemplo, o Claude Code ou o `agy`)

## Construir o agente

Na raiz do repositório, abra a sua ferramenta de código e peça:

```text
Crie o agente-estudos seguindo o AGENTS.md. Copie a skill skills/plano-de-estudos para agente-estudos/.claude/skills/ e crie o agente.ts, o package.json e o tsconfig.json. Sem configuração de deploy.
```

## Configurar a API Key

```bash
cd agente-estudos
read -s -p "Cole sua API Key: " KEY && echo

cat << EOF > .env
ANTHROPIC_API_KEY=${KEY}
MODEL=claude-sonnet-5-5
EOF

npm install
```

O `.env` não é versionado.

## Rodar e testar

```bash
npm start
```

Teste estes três cenários:

1. **Pedido incompleto:** o agente deve fazer perguntas antes de montar o plano.
    ```text
    Quero organizar meus estudos de JavaScript.
    ```
2. **Pedido completo:** o agente deve entregar o plano e salvar o CSV.
    ```text
    Quero aprender JavaScript para fazer meu primeiro projeto de portfólio em 5 semanas. Começo segunda que vem. Posso estudar segunda, quarta e sexta, 2h por dia às 19h. Já sei HTML e CSS, mas nunca programei.
    ```
3. **Ajuste:** o agente deve reorganizar o plano e sobrescrever o CSV.
    ```text
    Perdi as sessões de quarta e sexta da primeira semana. Ajusta o plano?
    ```

O arquivo gerado fica em `agente-estudos/planos/plano-de-estudos.csv`. Para usar, importe no [Google Agenda](https://calendar.google.com/): **Configurações → Importar e exportar → Importar**.

## Estrutura

```text
.
├── AGENTS.md                      # regras do projeto e do agente
├── README.md
├── skills/plano-de-estudos/       # fonte da skill
└── agente-estudos/                # criado a partir do AGENTS.md
    ├── agente.ts
    ├── .claude/skills/            # cópia da skill
    └── planos/                    # CSVs gerados, não versionado
```

Ao alterar a skill, atualize as duas cópias.

## Licença e origem

A skill `plano-de-estudos` e a estrutura inicial deste projeto vieram de um laboratório da Imersão Dev da Alura. Antes de publicar o repositório, confira com o autor original se há alguma condição de uso e, se quiser, adicione um arquivo `LICENSE` seu.

## Referências

- [Claude Agent SDK: visão geral](https://docs.claude.com/en/api/agent-sdk/overview)
- [Agent Skills no SDK](https://docs.claude.com/en/api/agent-sdk/skills) · [Especificação Agent Skills](https://agentskills.io/specification)
- [Claude Console: API Keys](https://console.anthropic.com/settings/keys)
