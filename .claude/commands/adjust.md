---
description: Classifies a change request (decision, new scope or bug) and updates spec/tickets before any code
argument-hint: <description of the change>
disable-model-invocation: true
---
Request: $ARGUMENTS. The task ID is the `<slug>` of the current branch `<tipo>/<slug>`; the docs are in `tasks/<ID>/`.

1. Read the task's `spec.md` and `tickets.md` and classify the request:
   - **Decision**: changes or refines a D# (or creates one) without changing scope.
   - **New scope**: something not in the spec, or listed under "Fora de escopo". If it is large, or it is an incompatible API/contract change that must ship in a separate PR (`docs/deploy.md`, "## Ordem de deploy entre web e API"), suggest a separate task.
   - **Bug**: behavior contradicts the spec. Check whether it belongs to an open ticket or is a regression from a closed one.
2. Show the classification and the proposed diff:
   - decision: the revised D# row (old text struck through, Base `usuário`, date), plus items in open tickets it affects;
   - new scope: a new decision plus items in an open ticket or a new ticket, with workspaces and "Pronto quando:";
   - bug: an item in the open ticket it belongs to, or a new ticket `Tn.1: correção` if ticket Tn is closed, citing the violated D#.
3. Wait for approval and apply it to `spec.md`/`tickets.md` (in Portuguese), updating "Estado atual".
4. Only then deal with the code: a small fix inside the ticket in progress is implemented now and follows that ticket's test/commit steps; anything else waits for `/implement-ticket`.
