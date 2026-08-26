---
name: open-pr
description: "Abre um Pull Request interativo com o template do repositório preenchido automaticamente. Use quando o usuário pedir para abrir/criar um PR."
user-invocable: true
---

# Abre um Pull Request interativo com template preenchido automaticamente

## Uso

```
/open-pr
```

## O que este comando faz

1. Pergunta qual branch base usar (main, develop, etc.)
2. Pergunta qual é o card da tarefa no Jira pedindo para digitar algo como "EDC-315"
3. Verifica se a branch atual está no padrão com o nome do card (ex: `feature/EDC-315-descricao-curta`) e pergunta se deseja renomeá-la caso não esteja
4. Verifica se há commits não pushados e faz push se necessário
5. Analisa o diff entre a branch atual e a branch base com `git diff base...HEAD`
6. Analisa os commits com `git log base..HEAD --oneline`
7. Gera título seguindo o padrão de título de PR com [card do Jira] - descrição da mudança. Exemplo: "[QUAL-1234] - Descrição curta das mudanças"
8. Preenche o template de PR automaticamente:
   - **Escopo**: Resumo das mudanças baseado no diff
   - **Known issues**: Lista problemas conhecidos ou "Nenhum identificado"
   - **Evidências**: Placeholder para prints/vídeos
   - **Roteiro de testes**: Passos para testar baseado nas mudanças
   - **Pontos de impacto**: Áreas do sistema afetadas
9. Mostra preview do PR e pede confirmação do usuário
10. Cria o PR **sempre como draft e sem nenhuma label** usando `gh pr create --draft --base <branch> --title "..." --body "..." --assignee @me --reviewer matheusbusarellopx,AugustoBendlin,AbraoDaniel,lexdmm,tchiteu`
11. Pergunta ao usuário quais labels adicionais deseja adicionar, apresentando as opções abaixo (seleção múltipla, pode escolher nenhuma, uma ou ambas):
    - `ready-for-staging` - indica que o PR está pronto para deploy em staging
    - `ready-to-test` - indica que o PR está pronto para ser testado pelo QA
12. Adiciona as labels selecionadas usando `gh pr edit <pr-number> --add-label <labels>`
13. Retorna a URL do PR criado

## Template de PR do Projeto

O projeto usa um template em português com as seguintes seções:

- 🔖 Escopo
- 🚩 Known issues
- 📁 Evidências
- Roteiro de testes: Caminho feliz
- 💥 Pontos de impacto

## Formato do Body do PR

```markdown
## 🔖 Escopo

<!-- Descreva em detalhes o escopo das alterações, o problema que está sendo resolvido e como foi resolvido. Se aplicável, vincule o ticket relacionado. -->

## 🚩 Known issues

<!-- Liste quaisquer problemas conhecidos ou limitações introduzidas por esta alteração. Se não houver nenhum, escreva "N/A". -->

## 📁 Evidências

<!-- Prints, logs ou vídeos -->

## Roteiro de testes: Caminho feliz

<!-- Descreva brevemente o propósito e objetivo desse roteiro -->

### Critério(s) de aceitação

<!-- Liste todos os critérios que devem ser atingidos ao cumprir o roteiro de testes -->

## 💥 Pontos de impacto

<!-- Quais partes do sistema podem ter sido afetadas? -->
```

## Notas Importantes

- Sempre fazer push da branch antes de criar o PR
- O título deve ser conciso (máximo 72 caracteres)
- A descrição deve seguir o template em `.github/pull_request_template.md`
- Usar o idioma português no título e na descrição do PR, bem como no nome da branch
- **Sempre pedir confirmação antes de executar `gh pr create`**
- Verificar se o GitHub CLI (`gh`) está autenticado
- O PR é **sempre criado como draft** (flag `--draft`)
- Nenhuma label é adicionada automaticamente na criação do PR
- As labels `ready-for-staging` e `ready-to-test` são opcionais e perguntadas ao usuário após a criação do PR
