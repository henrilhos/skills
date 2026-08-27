---
name: cleanup-worktrees
description: "Removes git worktrees that have no uncommitted changes, plus any orphaned Docker containers left behind (matched via the compose working_dir label). Use after finishing work in a worktree, or when asked to clean up/remove worktrees or their leftover containers."
user-invocable: true
argument-hint: "[directory]"
---

# Remove finished git worktrees and their orphaned Docker containers

## Usage

```
/cleanup-worktrees              # px-torre-core
/cleanup-worktrees <directory>  # any other repo
```

## 1. Resolve the repository

- Directory given as an argument → use it directly.
- No directory → `AskUserQuestion`:
  - "px-torre-core" — `~/Code/PX-Center/Torre/`
  - "Current directory" — the working directory
  - (the user can also type a custom path via "Other")

## 2. List candidate worktrees

```bash
git -C <DIRECTORY> worktree list
```

The first line is the primary checkout — never a removal candidate. Every other line is a candidate.

## 3. Check each candidate for uncommitted work

```bash
git -C <path> status --short --branch
```

**Clean** means the branch header is the only line. Anything else (staged, unstaged, or untracked) makes it dirty — leave it alone and list it in the report; don't force past this.

## 4. Remove clean worktrees

```bash
git worktree remove <path>
```

No `--force`. If git still refuses (untracked files `status --short` didn't flag as an ignored-but-tracked case, a stale lock, etc.), stop and show the user the exact refusal instead of retrying with `--force`.

## 5. Find orphaned Docker containers

A container is orphaned when its compose `working_dir` label points at a path that no longer exists on disk — this catches containers from worktrees removed just now *and* any left over from worktrees removed some other way:

```bash
docker ps -a --format '{{.ID}}' | while read -r id; do
  wd=$(docker inspect "$id" --format '{{index .Config.Labels "com.docker.compose.project.working_dir"}}')
  [ -n "$wd" ] && [ ! -d "$wd" ] && \
    docker inspect "$id" --format '{{.Names}}  {{index .Config.Labels "com.docker.compose.project"}}  '"$wd"
done
```

Skip this step (and step 6) if `docker` isn't running — report that Docker was unavailable rather than erroring out.

Group the hits by compose project name and bring each one down (no `-v` — named volumes may be shared or hold data worth keeping):

```bash
docker compose -p <project> down
```

## 6. Verify no dangling volumes

```bash
docker volume ls -f dangling=true
```

Report what's left; don't remove volumes yourself — ownership of a dangling volume is ambiguous (could be shared, could hold data someone wants). Ask before deleting any.

## 7. Report

- Worktrees removed vs. skipped (dirty, with their `status --short` output).
- Docker projects brought down, with the container names and working dirs that made them orphans.
- Any dangling volumes found, unactioned.

## Guardrails

- Check `git status --short --branch` before every `git worktree remove` — never remove a worktree you haven't verified clean.
- Never pass `--force` to `git worktree remove`; a refusal means stop and ask, not override.
- Never pass `-v` to `docker compose down` here; that deletes volumes, and orphan status doesn't imply the volume is safe to lose.
- Never run `docker volume rm`; surface dangling volumes and let the user decide.
- Never touch the primary checkout (the first line of `git worktree list`).
