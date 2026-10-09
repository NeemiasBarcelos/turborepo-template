---
description: Creates tasks/<ID>/ (spec.md, tickets.md, log.md) linking branch, workspaces, tickets and logs
argument-hint: <slug | tipo/slug> <title>
disable-model-invocation: true
---
Arguments: $ARGUMENTS (the first token is the task ID; the rest is the title).

The task ID is the `<slug>` of the branch `<tipo>/<slug>` (`docs/git-workflow.md`). The first token may be `<slug>` or `<tipo>/<slug>`; keep only the slug, which must match `^[a-z0-9-]+$`.

1. No valid ID: try the current branch (`git branch --show-current`); if it matches `^(feat|fix|refactor|perf|style|test|docs|chore|ci)/[a-z0-9-]+$`, use its slug. Otherwise ask and stop. No title: ask for it. If `tasks/<ID>/` already exists: warn and stop, never overwrite.
2. Check the branch: the current branch must be `<tipo>/<ID>`. If it is not, warn and suggest `/new-branch <tipo>/<ID>`; do not create the branch.
3. Create, from `tasks/_templates/`, filling in ID, title, branch and today's date:
   - `tasks/<ID>/spec.md` (leave "Workspaces" as `a definir no /investigate`)
   - `tasks/<ID>/tickets.md` (header and "Estado atual" only; replace the example tickets with `_A definir no /create-tickets._`)
   - `tasks/<ID>/log.md` (header only, no entries)
4. Do not fill in "Objetivo", "Contexto" or "Decisões" and do not commit (the task docs go in with the first ticket's commit). End by suggesting `/investigate <ID>`.

`fix/*` branches follow the same flow as features (investigate → create-tickets → implement-ticket → close-task); they usually have a single ticket. Task docs are written in Portuguese, following the templates.
