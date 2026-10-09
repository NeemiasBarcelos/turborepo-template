---
description: Asks the user to describe the task in free text, then reads only the relevant code/data and records a compact context and decisions in spec.md
argument-hint: [ID] [topic]
disable-model-invocation: true
---
Arguments: $ARGUMENTS. The task ID is the `<slug>` of the branch `<tipo>/<slug>`. No ID: use the slug of the current branch; if the branch is not `<tipo>/<slug>` or `tasks/<slug>/` does not exist, ask.

Rules:
- Read-only on code and database. The database is only the local `app_dev` (SELECT queries, never writes); never point at a Neon branch (`main`, `staging`, `ci/pr-*`). The only file you may write is `tasks/<ID>/spec.md`.
- Run outside plan mode: if plan mode is on, warn that nothing can be recorded and ask the user to leave it.
- After the user's reply to step 1 (never before), read `spec.md`, `docs/domain.md` and, if the task touches one, its `docs/features/<name>.md`. Whatever the root/app `CLAUDE.md` or `docs/` already decides is neither a question nor a decision.
- Questions to the user and everything written to `spec.md` stay in Portuguese.
- Goal: save tokens. Read only what the task touches, and write the context so that `/create-tickets` and `/implement-ticket` do not need to re-read the code to know where things are.

Steps:
1. **Ask before reading anything.** Your first action, before opening `spec.md`, code or database: send a plain text message (no AskUserQuestion, no options, no guesses) asking the user to describe the task in their own words, then end the turn and wait. Do not suggest scope, screens or modules in that question. A topic in the arguments is only context for the question, never the answer. With the reply in hand, record it under "Objetivo" (summarized, nothing added) and anything the user excludes under "Fora de escopo".
2. **Read only what the reply points to:** the module in `packages/contracts/src/`, `apps/api/src/modules/<module>/`, `apps/web/src/features/<module>/` and the tables they use. Use Grep/Glob to locate; read excerpts, not whole trees.
3. **Write "Contexto"**: grouped by workspace (contracts, api, web) and tables, each item `path:line` + what it does in one line; data numbers with database (`app_dev`) and date. Only what is specific to this task. Fill in "Workspaces" in the header.
4. Anything the code or data answers on its own does not go to the user: record it directly as a decision, with Base `código:`/`dado:`/`doc:`.
5. For the rest, ask **one** question at a time, with what you found, the options and a recommendation. Wait for the answer, record it right away as a row in "Decisões" (Base `usuário`, today's date) and remove it from "Questões abertas". If an answer implies an incompatible API/contract change, note that it splits into PRs (`docs/deploy.md`, "## Ordem de deploy entre web e API").
6. Repeat until no questions are left or the user stops. At the end, list the new decisions and suggest `/create-tickets <ID>`.

Do not write code, do not write tickets.md and do not commit.
