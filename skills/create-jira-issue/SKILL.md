---
name: create-jira-issue
description: "Creates Jira issues, always in the DEV project (Contract Execution), filling the required Projeto field (customfield_11412). Use when asked to create one or more Jira tasks/issues, put them under an epic, or link issues (blocks / is blocked by). Covers the [TORRE]/[PAINEL] title-prefix mapping to the Projeto field and issue linking."
user-invocable: true
---

# Create Jira task (DEV project)

Creates issues in Jira via the Atlassian MCP, always in the **DEV** project (Contract Execution — projectKey `DEV`, cloudId `6a9359ed-0c0c-48ca-b3c5-950d9391b7fd`, site `motoristapx.atlassian.net`).

## 1. Load the tools

Deferred — load schemas before calling:

```
ToolSearch("select:mcp__claude_ai_Atlassian__createJiraIssue,mcp__claude_ai_Atlassian__createIssueLink")
```

Add `mcp__claude_ai_Atlassian__getJiraIssue` and `mcp__claude_ai_Atlassian__getJiraIssueTypeMetaWithFields` when the epic or the field metadata needs checking.

## 2. Create each task

Write `summary` and `description` in Brazilian Portuguese, regardless of the language the user is writing in.

`createJiraIssue`, one independent call per task — batch them in parallel:

```
cloudId: 6a9359ed-0c0c-48ca-b3c5-950d9391b7fd
projectKey: DEV
issueTypeName: Tarefa
summary: <title>
parent: <DEV-XXXX>            # only when it's a child of an epic
description: <optional>       # markdown by default
additional_fields: { "customfield_11412": [{ "id": "<id>" }] }
```

`issueTypeName` defaults to `Tarefa` (id `10002`). Other types available in the project: `História`, `Bug`, `Idea`, `Incidente`, `Vulnerabilidade`, `Post mortem`, `Epic`.

**Required field — Projeto (`customfield_11412`)**: creation fails without it (error `"Projeto: Projeto é necessário."`). It's a multi-checkbox — pass it as an array of `{id}`:

| Value | id |
|---|---|
| Painel | 11269 |
| Torre | 11270 |
| App | 11271 |
| Integrações | 11396 |
| Falcon | 11429 |
| Outros | 11495 |

Map it from the title prefix: `[PAINEL]` → Painel (11269), `[TORRE]` → Torre (11270), `[APP]` → App (11271). No clear prefix → ask the user which project to use.

## 3. Link issues

When a task is "blocked by" / "blocks" another, use `createIssueLink` with type `Blocks`. `inwardIssue` is the blocker, `outwardIssue` is the blocked one.

E.g. "#2 is blocked by #1" → `inwardIssue: <#1>`, `outwardIssue: <#2>`.

## Report

`reporter`, `project`, `summary`, `issuetype` are required; `reporter` defaults to the authenticated user, the rest come from the steps above.

When done, return a table with key, title, and link (`https://motoristapx.atlassian.net/browse/<KEY>`) for each task created.
