---
name: create-jira-task
description: "Cria tarefas no Jira sempre no projeto DEV (Execução de Contratos), lembrando de preencher o campo obrigatório customfield_11412 (\"Projeto\"). Use quando o usuário pedir para criar uma ou mais tarefas/issues no Jira, colocá-las sob um épico, ou vincular tarefas (bloqueia / é bloqueada por). Cobre o mapeamento do prefixo [TORRE]/[PAINEL] para o campo Projeto e a criação de links entre issues."
user-invocable: true
---

# Criar tarefa no Jira (projeto DEV)

Cria issues no Jira via MCP Atlassian, sempre no projeto **DEV** (Execução de
Contratos), preenchendo o campo obrigatório **Projeto** (`customfield_11412`).

## Constantes

- **cloudId**: `6a9359ed-0c0c-48ca-b3c5-950d9391b7fd` (site `motoristapx.atlassian.net`)
- **projectKey**: `DEV`
- **issueTypeName**: `Tarefa` (id `10002`) — este é o padrão. Outros tipos do
  projeto: `História`, `Bug`, `Idea`, `Incidente`, `Vulnerabilidade`, `Post mortem`, `Epic`.

## ⚠️ Campo obrigatório: Projeto (`customfield_11412`)

A criação **falha** sem esse campo (erro `"Projeto: Projeto é necessário."`). É
um multicheckbox — passe via `additional_fields` como array de `{id}`:

```json
{ "customfield_11412": [{ "id": "11269" }] }
```

Opções:

| Valor       | id    |
|-------------|-------|
| Painel      | 11269 |
| Torre       | 11270 |
| App         | 11271 |
| Integrações | 11396 |
| Falcon      | 11429 |
| Outros      | 11495 |

**Mapeamento por prefixo do título**: `[PAINEL]` → Painel (11269),
`[TORRE]` → Torre (11270), `[APP]` → App (11271). Se o título não tiver prefixo
claro, pergunte ao usuário qual projeto usar.

## Passos

1. **Carregue as ferramentas** (são deferred — carregue os schemas antes de chamar):

   ```
   ToolSearch("select:mcp__claude_ai_Atlassian__createJiraIssue,mcp__claude_ai_Atlassian__createIssueLink")
   ```

   Se precisar validar o épico ou os campos: adicione
   `mcp__claude_ai_Atlassian__getJiraIssue` e
   `mcp__claude_ai_Atlassian__getJiraIssueTypeMetaWithFields`.

2. **Crie cada tarefa** com `createJiraIssue`. Chamadas independentes podem ir em
   paralelo no mesmo bloco:

   ```
   cloudId: 6a9359ed-0c0c-48ca-b3c5-950d9391b7fd
   projectKey: DEV
   issueTypeName: Tarefa
   summary: <título>
   parent: <DEV-XXXX>            # só quando for filha de um épico
   description: <opcional>       # markdown por padrão
   additional_fields: { "customfield_11412": [{ "id": "<id>" }] }
   ```

3. **Vínculos entre issues** (quando houver "bloqueada por" / "bloqueia"), com
   `createIssueLink` e tipo `Blocks`. A direção é: `inwardIssue` = quem bloqueia,
   `outwardIssue` = quem é bloqueada.

   Ex.: "#2 é bloqueada por #1" → `inwardIssue: <#1>`, `outwardIssue: <#2>`.

## Observações

- `reporter`, `project`, `summary`, `issuetype` são obrigatórios; `reporter` já
  vem com default (usuário autenticado) e os demais são preenchidos pelos passos acima.
- Ao terminar, retorne uma tabela com chave, título e link
  (`https://motoristapx.atlassian.net/browse/<KEY>`) de cada tarefa criada.
