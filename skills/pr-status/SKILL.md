---
name: pr-status
description: "Lists the team's open PRs grouped by Jira column, in Brazilian Portuguese, ready to paste into Teams"
user-invocable: true
argument-hint: "[--author <handle>]... [--repo <owner>/<name>]..."
---

# Team's open PR status, grouped by Jira column

Prints a status message in the chat, ready to paste into Teams. The message itself is Brazilian Portuguese; nothing is saved to a file or copied to the clipboard — the user copies it from the chat UI.

## Usage

```
/pr-status
/pr-status --author <handle>
/pr-status --repo <owner>/<name>
/pr-status --author <handle> --repo <owner>/<name> --author <outro>
```

No arguments uses the defaults below. `--author` and `--repo` are repeatable and **add to** the defaults, never replace them.

## Defaults

### Authors (GitHub handle → display name)

| Handle               | Name                  |
| -------------------- | --------------------- |
| `henrilhos`          | Henrique de Castilhos |
| `matheusbusarellopx` | Matheus Busarello     |
| `AugustoBendlin`     | Augusto Bendlin       |
| `AbraoDaniel`        | Daniel Abrão          |
| `lexdmm`             | Daniel Menescal       |
| `luiza-liebl`        | Luiza Liebl           |

### Repos (full name → short display name)

| Repo                            | Short name  |
| ------------------------------- | ----------- |
| `px-center/px-painel`           | Painel      |
| `px-center/px-torre-core`       | Torre       |
| `px-center/px-mobile-motorista` | Mobile      |
| `px-center/px-docs`             | Docs        |
| `px-center/px-event-store`      | Event Store |

A handle or repo passed via `--author`/`--repo` that isn't in these tables uses the raw handle, or the repo name without its owner, as its display name — never guess one.

## 1. Parse arguments

Extract every `--author <handle>` and `--repo <owner>/<name>` from `$ARGUMENTS` (both repeatable) and merge with the Defaults above into the final author list and repo list.

## 2. Fetch open PRs

One `gh search prs` call per author, with all repos passed together via repeated `--repo`. Run every author's call **in parallel** — multiple `Bash` tool calls in the same message:

```bash
gh search prs \
  --repo <repo1> --repo <repo2> ... \
  --state open --author <handle> \
  --json number,title,author,repository,url --limit 100
```

## 3. Extract Jira keys

Match each PR title against `\[?([A-Z]{2,5}-\d+)\]?` — covers `DEV-6345`, `FFC-123`, `QUAL-1234`. A title with no match goes into the "Sem chave JIRA" bucket.

## 4. Enrich via GraphQL

Single `gh api graphql` call, aliased per PR — not one call per PR. For each PR add an alias:

```graphql
pr_<idx>: repository(owner: "...", name: "...") {
  pullRequest(number: N) { ... }
}
```

querying:

- `isDraft`
- `mergeable` (`MERGEABLE` / `CONFLICTING` / `UNKNOWN`)
- `reviewDecision` (`APPROVED` / `CHANGES_REQUESTED` / `REVIEW_REQUIRED` / `null`)
- `reviews(last: 100) { nodes { state author { login } submittedAt } }` — to count unique approvals: dedup by `author.login`, keep only each reviewer's most recent review, count how many are `APPROVED`

Index results by `(owner/repo, number)` to join with the list from step 2.

`mergeStateStatus` is deliberately left out — `BEHIND`/`BLOCKED`/`UNSTABLE` would add noise with no clear use yet; revisit if that changes.

## 5. Look up Jira status

Deferred — load schemas before calling:

```
ToolSearch("select:mcp__claude_ai_Atlassian__getAccessibleAtlassianResources,mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql")
```

- `getAccessibleAtlassianResources` → `cloudId`.
- `searchJiraIssuesUsingJql`:
  - `cloudId`: from above
  - `jql`: `key in (DEV-X, DEV-Y, ...)` — every key at once
  - `fields`: `["summary", "status"]`
  - `maxResults`: `100`
- Auth/expiration error from the MCP call → **stop and ask** the user to run `/mcp` and re-authenticate `claude.ai Atlassian (2)`. Don't work around it.
- A key with no matching issue (mentioned in a PR title but nonexistent, or no permission) → "Sem status no Jira" bucket, not an error.

## 6. Column emoji and pt-br name

Emoji from `statusCategory.key`, not the column name — this works for any board:

| `statusCategory.key` | Emoji |
| -------------------- | ----- |
| `indeterminate`      | 🟡    |
| `new`                | 🔵    |
| `done`               | 🟢    |
| no status / no key   | ⚪    |

pt-br translation of the column name (fallback: the Jira column's own name, if it's not in this table):

| Jira column            | pt-br             |
| ---------------------- | ----------------- |
| `Code Review`          | Revisão de Código |
| `TESTING.` / `Testing` | Em Teste          |
| `Ready to Test`        | Pronto para Teste |
| `Waiting Deploy`       | Aguardando Deploy |
| `In Progress`          | Em Andamento      |
| `Done`                 | Concluído         |

## 7. Build PR badges

From the step 4 enrichment, concatenate with `·` in this order, omitting any that don't apply. No badges apply → no suffix on the bullet.

1. **Draft**: `🚧 draft` if `isDraft == true`
2. **Approvals**: `✅ N` (the unique-approval count from step 4); omit if `N == 0`
3. **Changes requested**: `🔴 changes requested` if `reviewDecision == "CHANGES_REQUESTED"`
4. **Conflict**: `⚠️ conflitos` if `mergeable == "CONFLICTING"`

## 8. Group and sort

**Column order**:

1. Revisão de Código (🟡)
2. Pronto para Teste (🔵)
3. Em Teste (🟡)
4. Aguardando Deploy (🔵)
5. Any other column, sorted by category (🟡 before 🔵 before 🟢)
6. Sem status no Jira (⚪)
7. Sem chave JIRA (⚪)

**Within a column**, group by Jira key (alphanumeric order). For each key:

- Bold header: `**CHAVE** — <Jira summary> (<name>)`
- One bullet per PR under that key: `- <ShortRepo>: <url>`, followed by the step 7 badges if any

`<name>` is the Jira assignee's display name when one is mapped, otherwise the author of the first PR found under that key.

## 9. Report

Print, exactly as shown:

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

- Blank line between columns, and between Jira keys within the same column.
- An empty column has no header at all.
- Zero PRs found in total → print only `Nenhum PR aberto encontrado para o time.` and stop there.
- Separator between the URL and the badges: `·` (two spaces, middle dot, two spaces). No badges → no separator.

## Environment

- `gh` authenticated with access to the repos.
- MCP `claude.ai Atlassian` connected.
