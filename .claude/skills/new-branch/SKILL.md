---
name: new-branch
description: Pergunta o nome da branch (<tipo>/<slug>) se não vier no argumento, derruba o dev local (bun run dev, next dev, bun --watch), atualiza a main, cria a branch e sobe docker compose + bun run dev. Usar quando o usuário pedir pra criar/começar uma branch ou task nova, ou retomar o ambiente de uma task.
---

# Nova branch + ambiente local

Um repositório só (monorepo Turborepo). Todos os comandos rodam a partir da raiz. Branching segue `docs/git-workflow.md`: branch curta a partir de `main`, nomeada `<tipo>/<slug>`. O `<slug>` é o ID da task (`tasks/<slug>/`).

## 0. Remote e nome da branch

1. `git remote -v`. Se `origin` ainda aponta para o repositório do template (`turborepo-template`) e este diretório é uma instância, avisar e parar (`docs/git-workflow.md`, "## Remote ao derivar do template"). No próprio template, seguir.
2. Nome da branch:
   - **Com argumento** (atalho): `feat/tasks-bulk-actions`. Se for válido, não perguntar de novo e mostrar o nome no resumo do passo 3, para o usuário conferir antes de qualquer git.
   - **Sem argumento, ou argumento inválido:** AskUserQuestion "Qual o tipo da branch?" com os tipos mais comuns (`feat`, `fix`, `refactor`, `chore`). O "Other" cobre `perf`, `style`, `test`, `docs` e `ci`. Depois pedir o slug em texto (minúsculas, números e hífens, ex.: `tasks-bulk-actions`) e montar o nome.
   - Validar contra `^(feat|fix|refactor|perf|style|test|docs|chore|ci)/[a-z0-9-]+$`. Se não bater, mostrar o que veio e perguntar de novo antes de seguir.

## 1. Derrubar o dev local

Só uma instância de dev por vez: duas disputam as portas 3000/3333 e o banco `app_dev`.

1. Listar candidatos:

   ```bash
   ps -axo pid,pgid,command | grep -E "turbo run dev|turbo dev|next dev|next-server|next-router-worker|bun --watch|bun run dev" | grep -v grep
   lsof -nP -iTCP:3333 -iTCP:3000 -sTCP:LISTEN
   ```

   Considerar só processos deste repositório (caminho no comando, ou `lsof -p <pid> | grep cwd` apontando para a raiz ou para `apps/*`). Mostrar ao usuário a lista do que vai ser encerrado.

2. Encerrar por process group, como um Ctrl+C. Um SIGINT só no PID do turbo não basta: processo em background ignora SIGINT e os filhos continuam rodando.

   ```bash
   kill -INT -- -<PGID>     # pra cada PGID encontrado
   sleep 5
   ps -g <PGID>             # conferir que o grupo esvaziou
   ```

   Se sobrar processo: `kill -TERM <pid>`. Se ainda sobrar depois de alguns segundos: `kill -KILL <pid>`.

3. Confirmar que `lsof -nP -iTCP:3333 -iTCP:3000 -sTCP:LISTEN` voltou vazio antes de seguir.

Os containers do `docker compose` (Postgres e Redis) ficam no ar: são compartilhados entre branches.

## 2. Atualizar a main

Seguir `.claude/skills/update-main/SKILL.md` (mudanças locais não são descartadas; avisos de `bun.lock` e migrations sem executar nada sem confirmação).

## 3. Criar a branch

```bash
B=<tipo>/<slug>
git rev-parse --verify --quiet "refs/heads/$B" && echo "existe local"
git ls-remote --heads origin "$B" | grep -q . && echo "existe no remoto"
```

- Se já existir (local ou remoto): perguntar se é só fazer checkout dela (com `git pull` se for remota) ou abortar.
- Se não existir: `git checkout -b "$B"`. **Sem push**; o push fica pro `/close-task`.

## 4. Subir o ambiente

Só depois do scaffold (`package.json` na raiz). Sem scaffold, pular e avisar.

1. `docker compose up -d` e esperar o healthcheck do Postgres e do Redis (`docker compose ps`).
2. Checar migrations pendentes só com leitura: comparar `drizzle.__drizzle_migrations` do `app_dev` com `apps/api/drizzle/meta/_journal.json`. Se faltar alguma, sugerir `bun run db:migrate`. Não rodar sem confirmação.
3. `bun run dev` em background (Bash com `run_in_background: true`). Acompanhar até a API responder em `:3333` e o web em `:3000` (`lsof -nP -iTCP:3333 -iTCP:3000 -sTCP:LISTEN`).

O dev local nunca usa o Neon (`docs/development.md`). Nunca apontar `DATABASE_URL` local para o branch `main` ou `staging`.

## 5. Resumo final

Reportar:

- branch atual e resultado do pull (já atualizado / N commits / erro);
- avisos de `package.json`/`bun.lock`/migrations;
- processos encerrados;
- status do ambiente (containers e portas no ar, ou erro).

Se `tasks/<slug>/` ainda não existir, sugerir `/new-task <slug> <título>`.
