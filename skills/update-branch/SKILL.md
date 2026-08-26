---
name: update-branch
description: "Atualiza a branch atual com a branch base (merge), resolve conflitos, valida e faz push"
user-invocable: true
argument-hint: "[branch-base] (padrão: main)"
---

# Atualiza a branch atual com a branch base, resolve conflitos, valida e faz push

## Uso

```
/update-branch           # usa main como base
/update-branch develop   # usa outra branch como base
```

A branch base é `$ARGUMENTS` quando informada; caso contrário, `main`.

## O que este comando faz

1. `git fetch origin <base>` e mostra os commits que vão entrar (`git log --oneline HEAD..origin/<base>`)
2. Avisa quando a base já contém trabalho equivalente ao da branch atual (ex.: o card irmão já foi mergeado em `main`) — isso é o principal previsor de conflito
3. `git merge origin/<base>`
4. Resolve os conflitos marcados (ver critério de resolução abaixo)
5. **Procura conflitos semânticos que o merge automático escondeu** — passo obrigatório, ver seção própria
6. Roda a validação do projeto e só segue com exit code 0
7. Commita o merge (`git commit --no-edit`, mantendo a mensagem padrão de merge)
8. Faz push, tratando rejeição e erro transitório conforme as seções abaixo
9. Confirma no final que remote e local apontam para o mesmo SHA

## Critério de resolução de conflitos

Cada lado costuma representar uma intenção diferente, e não versões concorrentes da mesma linha. O padrão é **combinar as duas intenções**, não escolher um lado.

Antes de resolver, entender o que cada lado quis fazer:

- `git log --oneline HEAD..origin/<base> -- <arquivo>` para ver o que a base mudou nele
- `git log --oneline origin/<base>..HEAD -- <arquivo>` para ver o que a branch atual mudou
- Ler as assinaturas reais (tipos de props, helpers, parâmetros) antes de escrever a resolução, em vez de deduzir pelo trecho conflitado

Depois de resolver, verificar que nada essencial foi perdido: as duas features precisam continuar presentes no arquivo final.

Nunca deixar marcador de conflito para trás:

```bash
grep -rln '^<<<<<<< \|^>>>>>>> \|^======= ' src || echo none
```

## Conflitos semânticos escondidos — não pular

O Git resolve por linha, não por significado. Um arquivo pode ser marcado como `Auto-merging` (sem conflito) e ainda assim quebrar, porque o merge pegou a linha de um lado que era incompatível com o uso do outro lado.

Caso real que já aconteceu neste repo: a base tornou um tipo privado (`type X` em vez de `export type X`) porque nada fora do módulo usava; a branch atual importava esse tipo. O merge aceitou a linha da base sem marcar conflito, e o erro só apareceu no `type:check`.

Por isso:

- **A validação é o detector, não uma formalidade.** Rodar a validação completa antes de commitar o merge — ela é o que encontra essa classe de problema.
- Checar os arquivos que apareceram como `Auto-merging` e tocam a mesma área da feature, não só os conflitados.
- Ao corrigir, verificar qual lado estava certo antes de mudar: `git show HEAD:<arquivo>` e `git show origin/<base>:<arquivo>`. A correção é restaurar a intenção que a branch atual precisa, não inventar uma terceira.

## Validação

Rodar o script de validação do projeto e conferir o exit code — a saída é longa e paralela, então ler só o final engana:

```bash
npm run validate >/dev/null 2>&1; echo "validate exit: $status"
```

Quando existir erro, filtrar o que importa em vez de despejar a saída inteira:

```bash
npm run validate 2>&1 | grep -E 'error TS|✖|Tests:|Test Suites:|ERROR' | head -20
```

Warnings de lint pré-existentes (que não vieram do merge) não bloqueiam — erros e testes quebrados bloqueiam. Se um teste falhar, reportar a falha com a saída, sem maquiar.

## Push rejeitado como non-fast-forward

Significa que a branch remota tem commits que não existem localmente — alguém empurrou trabalho ali. Investigar antes de qualquer coisa:

```bash
git fetch origin <branch-atual>
git rev-list --count HEAD..origin/<branch-atual>   # commits só no remote
git rev-list --count origin/<branch-atual>..HEAD   # commits só no local
git merge-base --is-ancestor origin/<branch-atual> HEAD; echo "ancestor: $status"
git --no-pager log --oneline --no-decorate HEAD..origin/<branch-atual> | cat
```

Contar com `rev-list --count` e confirmar com `merge-base --is-ancestor`. Saída de `git log` passando por pipe/filtro já apareceu vazia de forma enganosa nesta sessão, indicando "nada no remote" quando havia dois commits — não confiar em lista vazia sem o count concordar.

Havendo commits só no remote, integrar com `git merge origin/<branch-atual>`, resolver, validar de novo e então empurrar.

**Nunca fazer `git push --force` ou `--force-with-lease` por conta própria neste fluxo.** Force descarta o trabalho que está no remote. Se por algum motivo o force parecer necessário, parar, explicar o que seria descartado (com a lista de commits) e pedir decisão explícita do usuário.

## Erro transitório do GitHub

`remote: fatal error in commit_refs` (e parecidos com `remote rejected ... (failure)`) é falha do lado do servidor, não problema de ref. Tentar o push mais uma vez sem alterar nada. Persistindo depois da segunda tentativa, reportar ao usuário em vez de ficar repetindo.

## Notas Importantes

- **Nunca** usar `git push --force` sem aprovação explícita do usuário
- **Nunca** usar `git rebase` para atualizar a branch aqui — este comando é merge; a branch já é compartilhada e provavelmente tem PR aberto
- Não usar flag interativa (`-i`): não é suportado neste ambiente
- Manter a mensagem padrão do commit de merge (`git commit --no-edit`); não escrever mensagem própria para merge
- **NUNCA** adicionar "Generated with Claude Code" nem "Co-Authored-By: Claude" em nenhum commit
- Mensagem de commit sempre em inglês
- Validar antes de commitar o merge, não depois do push
- Ao terminar, relatar: quais conflitos existiam e como cada um foi resolvido, quais conflitos semânticos apareceram só na validação, o resultado da validação e o SHA final em local e remote
- Havendo conflito cuja resolução correta seja ambígua (as duas intenções são de fato incompatíveis), parar e perguntar ao usuário em vez de escolher um lado no escuro
