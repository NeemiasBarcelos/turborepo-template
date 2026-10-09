---
description: Stops the local dev, checks every ticket is closed, commits leftovers, runs the CI gates locally and opens the PR
argument-hint: [ID]
disable-model-invocation: true
---
Arguments: $ARGUMENTS. The task ID is the `<slug>` of the branch `<tipo>/<slug>`. No ID: infer it from the current branch.

1. **Stop the local dev:** `bun run dev`/turbo, `next dev`, `bun --watch`, using step 1 of `.claude/skills/new-branch/SKILL.md` (SIGINT to the process group, confirm 3000/3333 are closed). Do not start it again at the end. The `docker compose` containers stay up.
2. **Branch:** the current branch must be `<tipo>/<ID>`. Otherwise, stop.
3. **Tickets:** every ticket in `tickets.md` must be `[x]` with a commit. If any is open, list it and ask: implement it now (`/implement-ticket`), move it to "Fora de escopo" in `spec.md`, or abort.
4. **Leftovers:** `git status`. If anything is uncommitted (including the `tasks/<ID>/` bookkeeping), show the diff summary, propose the Conventional Commits message and **wait for explicit approval** before committing. Never `--no-verify`.
5. **Gates**, from the root (stop at the first failure, show the error and ask how to proceed):
   - `bun run lint`, `bun run typecheck`, `bun run test`;
   - `turbo run build --affected`;
   - if `apps/api` or `packages/contracts` changed against `main`: check that the Docker daemon is running (`docker info`; if not, ask the user to start it), then `docker build -f apps/api/Dockerfile -t api:<ID> .` and, after a successful build, `docker image rm api:<ID>`.
6. **PR:** draft the title in Conventional Commits (it becomes the squash commit on `main`) and the body: objective from `spec.md`, ticket list with commits, how it was verified, pending items. Follow `docs/checklists.md` "## Pull Request": if `apps/api` or `packages/contracts` changed, answer the two compatibility questions from `docs/deploy.md` ("## Ordem de deploy entre web e API"); if there is a migration, say so and that the Neon schema diff comment must be checked. End with the PR attribution line. Show them and **wait for explicit approval**. Once approved: `git push -u origin <tipo>/<ID>` and `gh pr create --base main --head <tipo>/<ID> --title … --body …`.
7. **Record:** the PR link in the `spec.md` header (`**PR:**`) and the "Fechamento" entry at the end of `log.md` (tickets, gates, PR). Update "Estado atual" in `tickets.md` to "Concluída". Show the commit message and, after approval, commit and push this bookkeeping to the same branch.
8. Reminders in the final summary: merge is squash only after CI is green; migrations reach staging/production only through `api-deploy.yml` (never by hand); the web publishes before the API; docs left to update (`docs/domain.md`, `docs/features/`).
