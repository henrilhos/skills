# skills

My [Agent Skills](https://github.com/vercel-labs/skills). Private repo, consumed by the `skills` CLI.

## Install

```bash
npx skills add henrilhos/skills                    # all skills
npx skills add henrilhos/skills --skill commit     # just one
npx skills use henrilhos/skills --skill pr-status  # use without installing
npx skills update                                  # update installed skills
npx skills ls                                      # list what's installed
```

Private repos work without extra setup. The CLI reuses whatever git auth is already on the machine (credential helper, GitHub CLI, SSH). To force a specific method, export `GITHUB_TOKEN` or `GH_TOKEN`.

## Skills

| Skill | What it does |
| --- | --- |
| `back-review` | Review a backend PR (PHP/Laravel), by ID or PR URL |
| `commit` | Conventional commits, grouped by logical unit |
| `create-jira-task` | Create Jira issues in the DEV project, filling the required `customfield_11412` field |
| `front-review` | Frontend quality review (TS/React): security, performance, architecture |
| `open-pr` | Open an interactive PR with the repo template filled in |
| `pr-status` | Team's open PRs grouped by Jira column, formatted for Teams |
| `update-branch` | Update the current branch from base via merge, resolve conflicts, validate, push |

## Layout

```
skills/<name>/SKILL.md
```

One directory per skill, each with a `SKILL.md` (frontmatter `name` + `description`, markdown body). It's one of the layouts the CLI auto-discovers.

## Notes

- Every skill has `user-invocable: true`, so they're still reachable as `/<name>` in Claude Code, the same way `back-review`, `commit`, `front-review`, `open-pr`, `pr-status`, and `update-branch` worked before, as slash commands in `~/.claude/commands/`.
- `skills update` deletes the skill's directory and recopies it. Any file a skill writes inside its own directory gets destroyed on update, including anything in `.gitignore`. I checked this in practice, it's not theoretical. A skill that needs to keep state has to write outside `~/.claude/skills/<name>/`.
- `front-review` uses the `` !`command` `` context-injection syntax, inherited from when it was a slash command. If that ever stops being expanded, the agent still reads the instruction, it just loses the precomputed file list.
