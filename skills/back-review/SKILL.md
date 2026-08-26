---
name: back-review
description: "Review de código de Pull Requests do backend (PHP/Laravel). Use com ID do PR ou URL. Ex: /back-review 4359 ou /back-review 4359 /caminho/do/repo"
user-invocable: true
allowed-tools: Read, Bash, Grep, Glob
argument-hint: "[pr-id] | [pr-url]"
---

# Code Quality Review

Skill para realizar code review de Pull Requests focado em código backend (PHP/Laravel).

## Uso

```
/back-review <PR_ID>
/back-review <PR_ID> <DIRETÓRIO>
/back-review <PR_URL>
/back-review <PR_URL> <DIRETÓRIO>
```

### Exemplos

- `/back-review 4359` - Pergunta qual repositório usar
- `/back-review 4359 /home/user/code/px-torre-core` - Usa o diretório especificado
- `/back-review https://github.com/px-center/px-torre-core/pull/4359` - Pergunta qual repositório usar

## Protocolo de Seleção do Repositório

### IMPORTANTE: Antes de executar qualquer comando `gh`, você DEVE determinar o diretório do repositório.

**Regra de seleção:**

1. **Se o diretório foi passado nos argumentos** → Use o diretório especificado
2. **Se NÃO foi passado diretório** → Use a ferramenta `AskUserQuestion` para perguntar:

```
Pergunta: "Qual repositório deseja usar para o code review?"
Opções:
  - "px-torre-core" (descrição: "Repositório principal do backend - /home/augustobendlin/code/px-torre-core")
  - "Diretório atual" (descrição: "Usar o diretório de trabalho atual")
```

**Nota:** O usuário pode selecionar "Other" para digitar um caminho customizado.

**Mapeamento de respostas:**

- Se escolher "px-torre-core" → use `/home/augustobendlin/code/px-torre-core`
- Se escolher "Diretório atual" → use o diretório de trabalho atual
- Se escolher "Other" e digitar um caminho → use o caminho digitado

## Configuração Importante

**SEMPRE** use `GITHUB_TOKEN=` antes de comandos `gh` para garantir autenticação correta.
**SEMPRE** execute os comandos dentro do diretório do repositório selecionado usando `cd <DIRETÓRIO> &&`.

```bash
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr view <PR_ID>
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr diff <PR_ID>
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr checks <PR_ID>
```

## Protocolo de Review

### 1. Coleta de Informações

Execute os seguintes comandos para obter contexto completo do PR (substitua `<DIRETÓRIO>` pelo caminho selecionado):

```bash
# Informações gerais do PR
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr view <PR_ID> --json title,body,author,baseRefName,headRefName,files,additions,deletions,changedFiles,reviews,comments,state

# Diff completo do PR
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr diff <PR_ID>

# Status dos checks (CI/CD)
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr checks <PR_ID>

# Comentários existentes (extraia owner/repo do repositório)
cd <DIRETÓRIO> && GITHUB_TOKEN= gh api repos/{owner}/{repo}/pulls/<PR_ID>/comments
```

### 2. Checklist de Análise Backend

#### Qualidade de Código PHP

- [ ] **Nomenclatura**: Variáveis, métodos e classes seguem convenções do projeto (camelCase para variáveis/métodos, PascalCase para classes)
- [ ] **Type hints**: Parâmetros e retornos com tipos declarados quando possível
- [ ] **Formatação**: Código segue configuração do Pint (Laravel preset, braces na mesma linha)
- [ ] **Imports**: Organizados e sem imports não utilizados
- [ ] **Comentários**: Código autoexplicativo, sem comentários óbvios ou desatualizados
- [ ] **Carbon::now()**: Linhas novas ou editadas devem usar `Carbon::now()` em vez do helper `now()`. Não validar código legado não tocado no PR — apenas linhas adicionadas ou modificadas. Se uma linha modificada usa `now()`, sugerir atualização para `Carbon::now()`

#### Padrões Laravel

- [ ] **Eloquent**: Uso correto de relacionamentos, evitar N+1 queries
- [ ] **Validação**: Form Requests ou validação inline quando apropriado
- [ ] **Services**: Lógica de negócio em Services, não em Controllers
- [ ] **Repositories**: Acesso a dados através de Repositories quando existe o padrão no módulo
- [ ] **Jobs/Events**: Operações assíncronas quando necessário
- [ ] **Migrations**: Reversíveis (método down), tipos corretos de colunas

#### Segurança

- [ ] **SQL Injection**: Uso de query builder/Eloquent, não queries raw sem binding
- [ ] **Mass Assignment**: Campos $fillable/$guarded definidos corretamente
- [ ] **Autorização**: Policies/Gates aplicados quando necessário
- [ ] **Dados sensíveis**: Sem credenciais, tokens ou dados sensíveis hardcoded
- [ ] **Validação de entrada**: Input do usuário sempre validado

#### Performance

- [ ] **Queries**: Sem N+1, uso de eager loading (with/load)
- [ ] **Cache**: Uso apropriado quando há dados frequentemente acessados
- [ ] **Indexes**: Migrations adicionam indexes para colunas de busca/filtro
- [ ] **Paginação**: Listagens grandes com paginação

#### Testes

- [ ] **Cobertura**: Testes para novos métodos/funcionalidades
- [ ] **Casos de borda**: Testes para edge cases
- [ ] **Fixtures**: Factories e seeders quando necessário

#### Arquitetura do Projeto

- [ ] **Localização**: Arquivo no diretório correto (Services/, Models/, etc.)
- [ ] **Responsabilidade única**: Classes/métodos com propósito bem definido
- [ ] **DTOs**: Uso de DTOs para transferência de dados complexos
- [ ] **Enums**: Uso de Enums para valores constantes (PHP 8.1+)

### 3. Análise de Impacto

Verificar:

- Quais módulos/features são afetados
- Se há breaking changes
- Se a migration é segura para rollback
- Dependências entre PRs (se houver)

### 4. Formato do Feedback

> **REGRA OBRIGATÓRIA — sempre cite o arquivo:** Toda sugestão, problema crítico ou ressalva DEVE começar com o cabeçalho `### caminho/do/arquivo.php:linha — título`, identificando o arquivo (e a linha quando possível) onde a inconsistência foi encontrada. Isso vale inclusive para pontos que não estão no diff (ex.: model relacionado, trait herdada) — nesse caso cite o arquivo de origem do fato. **Nunca** descreva um ponto de review sem dizer em qual arquivo ele está.

Estruture o review da seguinte forma:

````markdown
## 📋 Resumo do PR

**Título**: [título do PR]
**Autor**: [autor]
**Branch**: [head] → [base]
**Arquivos alterados**: [número]
**Adições/Remoções**: +[additions] / -[deletions]

## ✅ Pontos Positivos

- [Lista do que está bem implementado]

## ⚠️ Sugestões de Melhoria

### [arquivo:linha] - [título da sugestão]

**Problema**: [descrição]

**Sugestão**:

```php
// código sugerido
```
````

**Motivo**: [justificativa]

## 🔴 Problemas Críticos (se houver)

### [arquivo:linha] - [título do problema]

**Problema**: [descrição do problema]
**Impacto**: [qual o risco]
**Correção necessária**: [o que precisa ser corrigido]

## 📊 Checklist

- [x] Qualidade de código
- [x] Padrões Laravel
- [ ] Testes (pendente)
- [x] Segurança

## 🎯 Veredicto

- [ ] ✅ **Aprovado** - Pronto para merge
- [ ] 🔄 **Aprovado com ressalvas** - Pode fazer merge, mas considere as sugestões
- [ ] ⏳ **Mudanças solicitadas** - Necessário ajustes antes do merge
- [ ] 🚫 **Bloqueado** - Problemas críticos que impedem o merge

````

## Comandos Úteis Durante o Review

Lembre-se de sempre usar `cd <DIRETÓRIO> &&` antes dos comandos:

```bash
# Ver arquivo específico na branch do PR
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr diff <PR_ID> -- <caminho/do/arquivo>

# Ver checks do CI
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr checks <PR_ID>

# Listar arquivos alterados
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr view <PR_ID> --json files -q '.files[].path'

# Ver reviews existentes
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr view <PR_ID> --json reviews

# Adicionar comentário no PR
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr comment <PR_ID> --body "Comentário do review"

# Aprovar PR
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr review <PR_ID> --approve --body "LGTM! ✅"

# Solicitar mudanças
cd <DIRETÓRIO> && GITHUB_TOKEN= gh pr review <PR_ID> --request-changes --body "Por favor, verifique..."
````

## Padrões de Código do Projeto

### Pint (Formatação)

- Preset: Laravel
- Braces: mesma linha (Allman style para functions/classes)
- Array syntax: short (`[]` em vez de `array()`)
- Imports: ordenados
- Sem trailing comma em multiline

### PHPStan

- Level: 0 (com baseline)
- Paths: app/

## Notas

- Sempre leia o código completo do diff antes de dar feedback
- Considere o contexto do projeto e padrões já estabelecidos
- Seja construtivo e específico nas sugestões
- Priorize problemas de segurança e bugs sobre estilo
- Verifique se testes existentes continuam passando
- **Todo ponto de review deve indicar o arquivo (e linha) onde foi encontrado** — sem exceção, mesmo para arquivos fora do diff
