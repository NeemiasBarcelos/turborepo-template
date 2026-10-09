---
name: update-main
description: Coloca o repositório na branch main e roda git pull, avisando quando vierem mudanças em dependências (package.json, bun.lock) ou migrations da API. Usar quando o usuário pedir pra atualizar/sincronizar a main.
---

# Atualizar a main

Um repositório só (monorepo Turborepo). Todos os comandos rodam a partir da raiz.

## Passos

1. Checar mudanças locais antes de trocar de branch:

   ```bash
   echo "== $(git branch --show-current)"
   git status --porcelain
   ```

   Se houver mudanças, **não** fazer stash, reset nem descartar nada. Mostrar o status e perguntar ao usuário como proceder.

2. Atualizar a main, guardando o HEAD anterior para detectar o que mudou:

   ```bash
   git checkout main && before=$(git rev-parse HEAD) && git pull \
     && git diff --name-only "$before" HEAD -- package.json '*/package.json' '*/*/package.json' bun.lock apps/api/drizzle/
   ```

   Se o `checkout` ou o `pull` falhar, parar, mostrar o erro e o `git status` e perguntar.

3. Se o diff listar `package.json` ou `bun.lock`, avisar e sugerir `bun install` (só na raiz). Se listar `apps/api/drizzle/`, avisar e sugerir `bun run db:migrate`, que aplica no `app_dev` local. Não executar nada disso sem confirmação do usuário.

4. Reportar a branch atual e o resumo do pull: já atualizado, N commits novos ou erro.
