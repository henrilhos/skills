---
name: pr-status
description: "Lista PRs abertos do time agrupados por coluna do Jira (pt-br) para compartilhar no Teams"
user-invocable: true
argument-hint: "[--author <handle>]... [--repo <owner>/<name>]..."
---

# Status de PRs abertos do time, agrupado por coluna do Jira

Gera uma mensagem em pt-br, pronta para colar no Teams, listando os PRs abertos
do time agrupados pela coluna do board do Jira em que a tarefa correspondente
está. A mensagem é apenas impressa no chat — nada é salvo em arquivo nem
copiado para clipboard.

## Uso

```
/pr-status
/pr-status --author <handle>
/pr-status --repo <owner>/<name>
/pr-status --author <handle> --repo <owner>/<name> --author <outro>
```

Sem argumentos usa os defaults abaixo. `--author` e `--repo` são repetíveis e
**somam** aos defaults (não substituem).

## Defaults

### Autores (handle GitHub → nome para exibição)

- `henrilhos` → Henrique de Castilhos
- `matheusbusarellopx` → Matheus Busarello
- `AugustoBendlin` → Augusto Bendlin
- `AbraoDaniel` → Daniel Abrão
- `paulovictor237` → Paulo Victor Duarte
- `luiza-liebl` → Luiza Liebl

### Repos (nome completo → nome curto para exibição)

- `px-center/px-painel` → Painel
- `px-center/px-torre-core` → Torre
- `px-center/px-mobile-motorista` → Mobile
- `px-center/px-docs` → Docs
- `px-center/px-event-store` → Event Store

Para handles ou repos passados via `--author` / `--repo` que não estejam nessa
tabela: usar o próprio handle como nome de exibição e o nome do repo
(sem o owner) como nome curto.

## O que este comando faz

1. **Parse de `$ARGUMENTS`**: extrai flags `--author <handle>` e
   `--repo <owner>/<name>` (ambas repetíveis). Combina com os defaults para
   formar a lista final de autores e repos.

2. **Busca de PRs abertos** via `gh search prs` — uma chamada por autor, com
   todos os repos juntos via `--repo` repetido. Executar todas as chamadas
   **em paralelo** numa mesma mensagem (vários `Bash` tools de uma vez):

   ```bash
   gh search prs \
     --repo <repo1> --repo <repo2> ... \
     --state open --author <handle> \
     --json number,title,author,repository,url --limit 100
   ```

3. **Extração de chaves Jira dos títulos dos PRs** com o regex
   `\[?([A-Z]{2,5}-\d+)\]?`. Cobre formatos comuns (`DEV-6345`, `FFC-123`,
   `QUAL-1234`). PRs sem match vão para o bucket "Sem chave JIRA".

4. **Enriquecimento dos PRs via uma única chamada GraphQL** —
   `gh api graphql` com aliased queries por PR, uma chamada para todos. Para
   cada PR montar um alias `pr_<idx>: repository(owner: "...", name: "...") {
pullRequest(number: N) { ... } }` consultando estes campos:
   - `isDraft`
   - `mergeable` (`MERGEABLE` / `CONFLICTING` / `UNKNOWN`)
   - `reviewDecision` (`APPROVED` / `CHANGES_REQUESTED` / `REVIEW_REQUIRED` / `null`)
   - `reviews(last: 100) { nodes { state author { login } submittedAt } }`
     para calcular approves únicos (dedup por `author.login`, considerar
     somente o review mais recente de cada reviewer; contar quantos têm
     `state == APPROVED`)

   Indexar os resultados pelo par `(owner/repo, number)` para juntar com a
   lista do passo 2.

   > Nota: `mergeStateStatus` foi propositalmente omitido nesta iteração
   > (BEHIND/BLOCKED/UNSTABLE adicionam ruído e podem voltar depois).

5. **Lookup do status no Jira via MCP `claude.ai Atlassian`**:
   - Chamar `mcp__claude_ai_Atlassian__getAccessibleAtlassianResources`
     para obter o `cloudId`.
   - Chamar `mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql` com:
     - `cloudId`: o retornado acima
     - `jql`: `key in (DEV-X, DEV-Y, ...)` (todas as chaves de uma vez)
     - `fields`: `["summary", "status"]`
     - `maxResults`: `100`
   - Se a chamada do MCP retornar erro de autenticação/expiração, **parar e
     pedir** ao usuário para rodar `/mcp` e reautenticar
     `claude.ai Atlassian (2)` antes de continuar. Não tentar contornar.
   - Se uma chave da JQL não retornar issue (foi mencionada num PR mas não
     existe / sem permissão), tratar como "Sem status no Jira".

6. **Emoji da coluna** vem do `statusCategory.key` do Jira (não hardcoded por
   nome de coluna — funciona para qualquer board):
   - `indeterminate` → 🟡
   - `new` → 🔵
   - `done` → 🟢
   - sem status / sem chave → ⚪

7. **Tradução do nome da coluna para pt-br** (defaults; fallback = nome
   original do Jira se não houver tradução cadastrada):
   - `Code Review` → `Revisão de Código`
   - `TESTING.` / `Testing` → `Em Teste`
   - `Ready to Test` → `Pronto para Teste`
   - `Waiting Deploy` → `Aguardando Deploy`
   - `In Progress` → `Em Andamento`
   - `Done` → `Concluído`

8. **Badges do PR** (construídos a partir do enriquecimento do passo 4 —
   concatenar com `·` na ordem abaixo, omitindo os que não se aplicam):
   - **Draft**: `🚧 draft` se `isDraft == true`
   - **Approvals**: `✅ N` onde N é a contagem de approves únicos (calculada
     no passo 4). Omitir se N == 0.
   - **Changes requested**: `🔴 changes requested` se `reviewDecision ==
"CHANGES_REQUESTED"`
   - **Conflito**: `⚠️ conflitos` se `mergeable == "CONFLICTING"`

   Se nenhum badge se aplica, não adicionar sufixo no bullet.

9. **Agrupamento e ordenação**:
   - **Por coluna**, nesta ordem:
     1. Revisão de Código (🟡)
     2. Pronto para Teste (🔵)
     3. Em Teste (🟡)
     4. Aguardando Deploy (🔵)
     5. Outras colunas, ordenadas por categoria (🟡 antes de 🔵 antes de 🟢)
     6. Sem status no Jira (⚪)
     7. Sem chave JIRA (⚪)
   - **Dentro de cada coluna**, agrupar por chave Jira (ordem alfanumérica
     da chave). Para cada chave, mostrar:
     - Negrito: `**CHAVE** — <summary do Jira> (<nome do autor>)`
     - Bullets com cada PR daquela chave: `- <RepoCurto>: <url>` seguido dos
       badges do passo 8 (se houver)
   - O `<nome do autor>` é o do assignee do Jira se houver mapeamento, senão
     o do autor do primeiro PR encontrado.

10. **Formato final da mensagem** (imprimir no chat, exatamente assim):

    ```
    **Status dos PRs abertos** 📋

    **<emoji> <Coluna pt-br>**

    **CHAVE** — <summary do Jira> (<nome>)
    - <RepoCurto>: <url>  ·  <badges>
    - <RepoCurto>: <url>  ·  <badges>

    **CHAVE** — <summary do Jira> (<nome>)
    - <RepoCurto>: <url>

    **<emoji> <próxima coluna>**

    ...

    **⚪ Sem chave JIRA**
    - <título do PR> (<nome>): <url>  ·  <badges>
    ```

    - Linha em branco entre colunas e entre chaves Jira dentro da mesma coluna.
    - Se uma coluna estiver vazia, omitir o cabeçalho dela.
    - Se nenhum PR for encontrado no total, imprimir apenas:
      `Nenhum PR aberto encontrado para o time.`
    - Separador entre URL e badges: `·` (dois espaços, middle dot, dois
      espaços). Sem badges, omitir o separador.

## Notas importantes

- A mensagem é **apenas exibida no chat**. Não salvar arquivo, não rodar
  `pbcopy`. O usuário copia da própria UI.
- Executar as chamadas `gh search prs` (uma por autor) em paralelo.
- Não inventar status do Jira — se vier vazio, usar o bucket "Sem status no
  Jira" com ⚪.
- Pressupostos do ambiente:
  - `gh` autenticado com acesso aos repos
  - MCP `claude.ai Atlassian` conectado
- Para autores/repos extras não cadastrados, mostrar o handle/nome de repo
  cru — não tentar deduzir display names.
