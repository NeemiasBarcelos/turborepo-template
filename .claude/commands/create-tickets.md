---
description: Slices the task into small tickets with checkboxes in tasks/<ID>/tickets.md
argument-hint: [ID]
disable-model-invocation: true
---
Arguments: $ARGUMENTS. The task ID is the `<slug>` of the branch `<tipo>/<slug>`. No ID: infer it from the current branch.

1. Read `spec.md`. Do not re-read code the "Contexto" already maps; open a file only if a ticket's shape depends on it. If an open question would change the slicing, list it and suggest `/investigate` first.
2. Build the tickets:
   - **Small:** each ticket fits in one session and becomes **one commit**. If a ticket needs more than ~8 items, split it.
   - **T1 = a thin end-to-end slice**: the smallest path that crosses every workspace the task touches (contracts → api → web) and can be seen working. E.g. the read-only listing with its schema in `contracts`, the API route and the screen.
   - **Each risk in its own ticket** (migration, multi-tenancy isolation, permissions, side effects, concurrency, bulk writes), with **tests written first** (failing before the code): `bun test` in the API (`apps/api/docs/testing.md`, including the tenant isolation test), Vitest/MSW in the web (`apps/web/docs/testing.md`).
   - Use `docs/checklists.md` "## Feature ponta a ponta" as the list of pieces, not as an order.
   - An incompatible API/contract change splits into separate PRs (`docs/deploy.md`, "## Ordem de deploy entre web e API"): flag it and suggest a separate task for the second PR.
   - Each ticket states its workspaces (`contracts`, `api`, `web`), the decisions (D#) it implements, `[ ]` items, an observable **"Pronto quando:"** and `**Commit:** —`.
   - The last ticket covers end-to-end browser verification and any docs (`docs/domain.md` glossary, `docs/features/<name>.md`), when the task needs them.
3. Show the list (title, workspaces and "Pronto quando:" per ticket) and wait for approval. Only then write `tasks/<ID>/tickets.md` (template `tasks/_templates/tickets.md`, in Portuguese), with "Estado atual" pointing to T1.
4. Do not write code. End by suggesting `/implement-ticket <ID>`.
