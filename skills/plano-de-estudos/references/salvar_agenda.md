# Salvar a Agenda em Arquivo

O agente salva a agenda num arquivo CSV usando as ferramentas nativas `Write` e `Read` do Claude Agent SDK. A pessoa importa o arquivo gerado no Google Agenda.

---

## Ferramentas disponíveis

| Ferramenta | Para que serve | Argumentos |
|---|---|---|
| `Write` | Cria o arquivo ou **sobrescreve** o arquivo inteiro | `file_path` (caminho absoluto), `content` (texto) |
| `Read` | Lê um arquivo e devolve o conteúdo com o número de cada linha | `file_path` (caminho absoluto), `offset` e `limit` (opcionais) |

Use **só** essas duas. Não tente executar comandos nem editar trechos: o CSV é sempre gravado inteiro, de uma vez.

## Regras de caminho

* A instrução do sistema informa o caminho absoluto da **pasta de planos**. Grave sempre em `<pasta de planos>/plano-de-estudos.csv`.
* Não grave em nenhum outro lugar. O agente recusa escritas fora da pasta de planos com a mensagem `Escrita fora da pasta de planos`.
* Use sempre o mesmo nome de arquivo. No replanejamento, a versão nova substitui a anterior.

## Como gravar a agenda

1. Monte o CSV completo seguindo as regras de `assets/plano_template.md`.
2. Antes de chamar a ferramenta, confira o checklist:
   - [ ] A primeira linha é exatamente `Subject,Start Date,Start Time,End Date,End Time,Description`.
   - [ ] Há uma linha por sessão, sem linhas em branco, e nenhuma sessão cai em dia que a pessoa não informou.
   - [ ] Datas em `MM/DD/YYYY` e horários em `HH:MM AM/PM`.
   - [ ] Toda `Description` está entre aspas duplas.
   - [ ] O conteúdo **não** tem a cerca ```` ```csv ```` nem texto explicativo, só o CSV.
3. Chame `Write` com `file_path` igual a `<pasta de planos>/plano-de-estudos.csv` e `content` igual ao CSV, terminando com uma quebra de linha.

Exemplo de chamada (com a pasta de planos `/app/planos`):

```json
{
  "file_path": "/app/planos/plano-de-estudos.csv",
  "content": "Subject,Start Date,Start Time,End Date,End Time,Description\n📚 Python: variáveis e entrada/saída,11/02/2026,08:00 PM,11/02/2026,09:00 PM,\"Revisão: — | Técnica: Aprender fazendo | Entregável: script de nome e idade\"\n"
}
```

## Como interpretar a resposta

| Resposta da ferramenta | O que fazer |
|---|---|
| Mensagem de sucesso (`File created successfully` ou `has been updated`) | Sucesso. Informe na seção 5 do template o caminho e o número de sessões. |
| `Escrita fora da pasta de planos` | O caminho estava errado. Tente de novo **uma vez** com `<pasta de planos>/plano-de-estudos.csv`. |
| Qualquer outro erro | Não insista. Diga em uma frase que não foi possível salvar o arquivo e mostre o CSV num bloco ```` ```csv ```` na seção 5. |

Nunca diga que salvou o arquivo sem ter recebido a mensagem de sucesso.

## Replanejamento

1. Se precisar lembrar do plano atual, chame `Read` com `file_path` igual a `<pasta de planos>/plano-de-estudos.csv`.
2. A resposta vem com o número de cada linha na frente (ex.: `     1→Subject,Start Date,...`). Esses números e a seta **não** fazem parte do arquivo: remova-os antes de reaproveitar qualquer linha.
3. Se a resposta indicar que o arquivo não existe, monte o CSV a partir da tabela de sessões da conversa.
4. Monte o CSV novo **inteiro**, já com os ajustes, e grave com `Write` no mesmo caminho. A versão anterior é substituída.
5. Na resposta, diga o que mudou e confirme que o arquivo foi atualizado.
