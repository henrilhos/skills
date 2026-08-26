# skills

Minhas [Agent Skills](https://github.com/vercel-labs/skills) — repo privado, consumido pelo CLI `skills`.

## Instalar

```bash
npx skills add henrilhos/skills                    # todas as skills
npx skills add henrilhos/skills --skill commit     # só uma
npx skills use henrilhos/skills --skill pr-status  # usa sem instalar
npx skills update                                  # atualiza as já instaladas
npx skills ls                                      # lista o que está instalado
```

Repo privado funciona sem configuração extra: o CLI reaproveita a autenticação de git que já existe na máquina (credential helper → GitHub CLI → SSH). Se precisar forçar, exporte `GITHUB_TOKEN` ou `GH_TOKEN`.

## Skills

| Skill | O que faz |
| --- | --- |
| `back-review` | Review de PR de backend (PHP/Laravel), por ID ou URL do PR |
| `brag` | Relatório de conquistas a partir de PRs, commits, sessões do Claude Code, Asana, Slack e Sentry |
| `commit` | Commits em conventional commits, agrupados por unidade lógica |
| `create-jira-task` | Cria issues no Jira no projeto DEV, com o campo obrigatório `customfield_11412` |
| `front-review` | Review de qualidade de frontend (TS/React): segurança, performance e arquitetura |
| `open-pr` | Abre PR interativo com o template do repo preenchido |
| `pr-status` | PRs abertos do time agrupados por coluna do Jira, formatado para o Teams |
| `update-branch` | Atualiza a branch atual com a base via merge, resolve conflitos, valida e dá push |

## Layout

```
skills/<nome>/SKILL.md
```

Um diretório por skill, cada um com `SKILL.md` (frontmatter `name` + `description`, corpo em markdown). É um dos layouts que o CLI descobre automaticamente.

## Notas

- Todas as skills têm `user-invocable: true`, então continuam acessíveis como `/<nome>` no Claude Code — é assim que `back-review`, `commit`, `front-review`, `open-pr`, `pr-status` e `update-branch` viviam antes, como slash commands em `~/.claude/commands/`.
- `brag` lê `config.json` do próprio diretório da skill. O `config.json` aqui já vem preenchido; `config.json.example` é o template genérico que o `SKILL.md` referencia.
- `skills/brag/impact.md` está no `.gitignore` de propósito. É estado local que a skill vai acrescentando ao longo do tempo, e versionar aqui faria `skills update` sobrescrever entradas mais novas da máquina. Vale backup separado.
- `front-review` usa a sintaxe `` !`comando` `` de injeção de contexto, herdada de quando era slash command. Se em algum momento ela deixar de ser expandida, o agente ainda lê a instrução — só perde o pré-cálculo da lista de arquivos.
