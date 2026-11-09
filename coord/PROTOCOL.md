# Coordination protocol: coordinator ↔ cloud worker

Two Claude sessions work on this repository:

- **Coordinator:** a local session on the user's PC. It plans the work, writes tasks, reviews results and merges them.
- **Worker:** a cloud session. It picks up tasks, does the work and reports back.

GitHub is the only channel between the two sessions.

## Ownership rules

| What | Who writes it | Where |
|---|---|---|
| Task files `coord/tasks/NNN-slug.md` | Coordinator only | `main` |
| Merges into `main` | Coordinator only | `main` |
| Work and the report for a task | Worker only | branch `task/NNN-slug` |

**The worker never pushes to `main` and never force-pushes.** Keeping `main` single-writer prevents push conflicts between the two sessions.

## Task file format (`coord/tasks/NNN-slug.md`)

```markdown
---
id: NNN
status: open        # open | done | cancelled  (only the coordinator changes it)
---
# Title

What to do, which files, what done looks like.
```

## Worker cycle (run every few minutes)

1. Run `git fetch origin --prune`, then `git checkout main && git reset --hard origin/main`.
2. A task is **pending** when both of these are true:
   - its file in `coord/tasks/` has `status: open`
   - **no** remote branch `origin/task/NNN-slug` exists yet
3. Take the lowest-numbered pending task:
   1. `git checkout -b task/NNN-slug origin/main`
   2. Do the work and commit it.
   3. Write `coord/reports/NNN-slug.md` (format below) and commit it **last**.
   4. `git push -u origin task/NNN-slug`
4. If nothing is pending, do nothing. Don't commit and don't push.

If a task's `task/` branch already exists, the task has been taken. Never redo it. If the coordinator wants changes, it writes a **new** task.

## Report format (`coord/reports/NNN-slug.md`)

```markdown
---
id: NNN
result: done        # done | partial | blocked
---
## Summary
What was done, in a few lines.

## Changes
Files touched, and why.

## Notes / blockers
Questions for the coordinator, anything not verified, anything that needs the user.
```

A report is the worker's only voice, so it should be complete and honest. Say what was tested and what wasn't.

## Coordinator cycle

1. `git fetch origin --prune`, then look for `origin/task/*` branches that contain a `coord/reports/*.md` file.
2. Read each report, review the diff, then merge it into `main` or write a follow-up task.
3. Once a task is handled, set its task file to `status: done` on `main` and push. Only after that, delete the `task/` branch on GitHub. In that order, the worker never sees the task as pending again.
