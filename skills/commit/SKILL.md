---
name: commit
description: "Create well-formatted, atomic git commits in English with conventional commit messages, splitting into separate commits by default."
user-invocable: true
allowed-tools: Bash
---

# Commit

## Usage

```
/commit
```

## 1. Stage

`git status`. If nothing is staged, stage everything modified and new with `git add`. If files are already staged, commit only those.

## 2. Review and split

`git diff` the staged changes. Default to splitting: commit the smallest coherent unit that still makes sense on its own, and start a new commit as soon as the next change belongs to a different concern — unrelated part of the codebase, different change type (feat/fix/refactor tangled together), or just a second logical step. Only keep changes together when splitting them would leave a commit that doesn't build or doesn't make sense alone.

## 3. Commit

`<type>: <description>`, imperative mood, present tense ("add", not "added"), first line under 72 characters.

| Type | Use for |
| --- | --- |
| `feat` | new feature |
| `fix` | bug fix |
| `docs` | documentation |
| `style` | formatting, no code change |
| `refactor` | neither fixes a bug nor adds a feature |
| `perf` | performance improvement |
| `test` | adding or fixing tests |
| `chore` | build process, tooling |

## Guardrails

- Write the commit message in English, regardless of the language the user is writing in.
- Never add "Generated with Claude Code" or "Co-Authored-By: Claude" to the commit message.
- Verify the code is lint-clean, builds, and docs are updated before committing.
