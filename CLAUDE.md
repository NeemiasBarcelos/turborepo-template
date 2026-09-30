# CLAUDE.md

Guia para o Claude ao trabalhar neste repositório. Este arquivo é a raiz
do monorepo. Regras transversais ficam em `/docs`, e cada app tem o
próprio `CLAUDE.md` com as regras dele. Leia os arquivos referenciados
quando a tarefa exigir.

## O que é este projeto

Template base para produtos em **monorepo Turborepo** com dois apps:

- `apps/web`: frontend **Next.js**, deploy na **Vercel** (baseline
  app-nextjs).
- `apps/api`: API **Bun/Elysia** multi-tenant em Docker, num host como
  DigitalOcean, Railway ou Render (baseline api-bun).

O banco é sempre **Neon**, operado pelo `neonctl`. O contrato entre os
apps vive em `packages/contracts`. Cada produto derivado herda a stack e
as regras abaixo. O domínio fica em `docs/domain.md` de cada instância.

**Trabalhando dentro de um app?** Leia também `apps/web/CLAUDE.md` ou
`apps/api/CLAUDE.md`. As regras de lá valem, com os ajustes de "##
Diferenças no monorepo". Quando conflitarem com este arquivo, este vence.

## Versão da baseline

Versionamento semântico próprio: ver `docs/versioning.md` para o que
conta como MAJOR/MINOR/PATCH e `docs/CHANGELOG.md` para o histórico.
Versão atual: **0.1.0** (embute app-nextjs `0.1.0` e api-bun `0.16.0`).

## Stack

- Turborepo + Bun workspaces (um único `bun.lock`)
- `apps/web`: Next.js 16, React 19, Tailwind v4, shadcn/ui, TanStack
  Query, Better Auth client (Node na Vercel)
- `apps/api`: Bun, Elysia, Drizzle ORM + postgres.js, Better Auth +
  `organization`, Redis, Zod
- `packages/contracts`: Zod, sem outras dependências de runtime
- `packages/tsconfig`: bases de `tsconfig`
- Neon (Postgres 16) + `neonctl`; Postgres/Redis locais via Docker
- Biome v2 (um `biome.json` na raiz), Husky + commitlint
- GitHub Actions; Vercel (web); GHCR + host de containers (api)

## Comandos essenciais

Sempre a partir da raiz:

```bash
bun install                            # instala todos os workspaces
docker compose up -d                   # Postgres 16 + Redis locais
bun run dev                            # web :3000 + api :3333
bun run lint                           # Biome sem escrever (o que o CI roda)
bun run check                          # Biome com --write
bun run typecheck                      # tsc em todos os workspaces
bun run test                           # testes de todos os workspaces
turbo run test --filter=@repo/api      # testes de um workspace
turbo run build --affected             # build do que mudou
bun run db:generate                    # migration a partir do schema Drizzle
bun run db:migrate                     # aplica migrations (app_dev local)
bun run neon:branch ci/pr-<n>          # wrappers do neonctl (docs/neon.md)
```

## Regras não-negociáveis (baseline travada, ver docs/versioning.md)

Mudar qualquer item desta lista é mudança de versão do template, não
ajuste de tarefa. Se uma tarefa parecer exigir quebrar uma regra daqui
(ou dos `CLAUDE.md` dos apps), trate como proposta de mudança na
baseline (`docs/versioning.md`), nunca como exceção silenciosa.

- Bun é o único package manager. Um só lockfile, `bun.lock` na raiz:
  sem `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock` nem `bun.lock`
  dentro de `apps/*`. `bun install` só na raiz.
- App nunca importa de outro app. Código compartilhado vive em
  `packages/*`, importado por subpath declarado em `exports`, com a
  dependência `workspace:*` declarada.
- O contrato web ↔ api (erro, permissões, enums de borda, schemas de
  request/response) vive **só** em `packages/contracts`. Nunca copiar
  para dentro de um app. O `contracts` só depende de `zod` e nunca
  importa driver de banco, Elysia, Better Auth server nem alias `@/`.
- Nunca commitar sem `bun run lint` e `bun run typecheck` passando.
- Variável de ambiente só pelos módulos de env validados com Zod de cada
  app. Variável que afeta build ou teste é declarada em `env` da task no
  `turbo.json`. Nunca `.env` na raiz.
- Neon é o único Postgres hospedado. Produção, staging e CI são branches
  do mesmo projeto (`main`, `staging`, `ci/pr-<n>`).
- Branch Neon é operado pelos wrappers `bun run neon:*`
  (`scripts/neon/`). Todo branch efêmero nasce com `--expires-at`.
  Connection string nunca aparece em log nem vai para o repositório.
- Nunca `drizzle-kit push` em branch Neon (nenhum, nem de CI). Schema
  hospedado só muda por `db:migrate`.
- Migrations de staging e produção rodam **só** no `api-deploy.yml`,
  antes do deploy, com `DATABASE_URL_UNPOOLED` do Environment. Nunca no
  start do container nem pelo pooler. Auto-deploy por Git do host da API
  fica desligado.
- Mudança de API ou contrato é compatível com o outro lado **em
  produção**. O web publica antes da API. Mudança incompatível é
  dividida em PRs (`docs/deploy.md`, "## Ordem de deploy entre web e
  API").
- `bun test` da API só roda contra banco `_test` e Redis de índice ≠ 0,
  local ou no branch Neon de CI (guarda do api-bun).
- Commits em Conventional Commits pelo hook `commit-msg`. Nunca
  `--no-verify`. Nunca push direto em `main`: todo merge passa por PR com
  squash.
- Nunca commitar ou dar push no repositório de onde a instância foi
  clonada (o template): antes do primeiro commit, confirmar o remote
  `origin` (`docs/git-workflow.md`).

## Leia antes de tarefas específicas

- Estrutura do monorepo, turbo, pacotes, contrato, decisões → `docs/architecture.md`
- Branches do Neon, `neonctl`, CI no Neon, staging, restore → `docs/neon.md`
- Setup local, Docker, env, comandos → `docs/development.md`
- Vercel, Dockerfile da API, host, ordem de deploy → `docs/deploy.md`
- Workflows do GitHub Actions, secrets → `docs/ci-cd.md`
- Imports entre workspaces, Biome, tsconfig, scripts → `docs/conventions.md`
- Branch, commits, PR e merge → `docs/git-workflow.md`
- Checklists (feature ponta a ponta, PR, setup, bump) → `docs/checklists.md`
- Mecanismo de versão e sincronização com app-nextjs/api-bun → `docs/versioning.md`
- Glossário, módulos e infra desta instância → `docs/domain.md`
- Features complexas → `docs/features/<nome>.md`
- Qualquer coisa dentro de `apps/web` → `apps/web/CLAUDE.md`
- Qualquer coisa dentro de `apps/api` → `apps/api/CLAUDE.md`

## O que NÃO fazer

- Não introduzir biblioteca nova sem checar se já existe equivalente na
  stack, nem com versão diferente da já usada em outro workspace.
- Não criar pacote em `packages/` para código que só um app usa.
- Não colocar schema Drizzle, `TenantContext` ou regra de negócio no
  `contracts`.
- Não pôr `[run] bun = true` num `bunfig.toml` fora de `apps/api` (o Next
  rodaria no runtime do Bun).
- Não criar branch Neon à mão sem expiração, nem apontar teste ou script
  local para `main`.
- Não rodar `neonctl link` sem `--no-env-pull`.

---

Versão da baseline: 0.1.0. Ver `docs/CHANGELOG.md`.
