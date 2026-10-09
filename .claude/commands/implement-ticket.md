---
description: Asks which ticket to do, implements it, tests, runs lint/typecheck/test, logs and commits (after approval)
argument-hint: [ID] [Tn]
disable-model-invocation: true
---
Arguments: $ARGUMENTS. The task ID is the `<slug>` of the branch `<tipo>/<slug>`. No ID: infer it from the current branch. A token like `T3` is the ticket.

1. **Branch:** `git branch --show-current` must be `<tipo>/<ID>`. Otherwise, stop.
2. **Which ticket:** if no ticket came in the arguments, ask with AskUserQuestion listing the open tickets in `tickets.md` (title + "Pronto quando:"), recommending the first open one.
3. **Read only:** `spec.md`, "Estado atual" and the chosen ticket, plus the `CLAUDE.md` of each app the ticket touches. Read a previous log entry only if this ticket depends on it. Use the "Contexto" paths instead of searching again.
4. Summarize in a few lines what will be done, then **implement** only that ticket's items. If something outside the spec comes up (new decision, new scope, bug outside the ticket): stop and suggest `/adjust`; do not resolve it on your own. Tick the items in `tickets.md` as they are done.
5. **Test:**
   - "Pronto quando:" check: `bun test` for API logic, Vitest for web logic, browser smoke on `bun run dev` for screens.
   - Migration in the ticket: generated with `bun run db:generate`, reviewed (expand/contract) and applied only to the local `app_dev` with `bun run db:migrate`, after the user's ok. Never `drizzle-kit push`.
   - Validation pipeline, from the root: `bun run lint`, `bun run typecheck`, then `turbo run test --filter=<each workspace touched>`. API tests only run against `app_test` (guard in `apps/api/docs/testing.md`).
   - If anything fails, fix it inside the ticket's scope and run again. Never commit with a red pipeline.
6. **Log:** append the ticket's entry to `log.md` (template `tasks/_templates/log.md`, in Portuguese, up to ~15 lines). Findings that change a decision go to `spec.md` via `/adjust`, not to the log.
7. **Commit:** show the files to stage (the ticket's code plus `tasks/<ID>/`) and the Conventional Commits message `<tipo>(<escopo>): <description>` (scope by business module, then workspace, then infra area; `docs/git-workflow.md`), and **wait for explicit approval**. Once approved: `git add <files>` and `git commit -m "…"` (the Husky `pre-commit` and `commit-msg` hooks run; never `--no-verify`). Record the hash in the ticket (`**Commit:**`) and in the log entry, update "Estado atual" (open count, next ticket) and fold that bookkeeping into the next ticket's commit, or `/close-task`'s leftovers. Never push.
8. Do not start the next ticket. Suggest `/clear` and then `/implement-ticket <ID>`, or `/close-task <ID>` if no ticket is left open.
