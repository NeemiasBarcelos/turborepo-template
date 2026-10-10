# CLAUDE.md

> Baseado em api-bun v0.16.0, adaptado ao monorepo `turborepo-template`
> v0.3.0. Leia antes `../../CLAUDE.md` (regras da raiz, que vencem em caso
> de conflito) e "## Diferenças no monorepo" no fim deste arquivo.

Guia para o Claude ao trabalhar neste repositório. Este arquivo é a raiz —
regras específicas de domínio, arquitetura e convenções ficam em `/docs`.
Leia os arquivos referenciados quando a tarefa exigir.

## O que é este projeto

Template base para APIs TypeScript **multi-tenant** (cada organização é um
tenant; não existe modo single-tenant). Cada API real derivada deste
template herda a stack e as convenções abaixo; o domínio de negócio específico é
definido em `../../docs/domain.md` de cada instância.

## Versão da baseline

Este template segue versionamento semântico próprio — ver
`../../docs/versioning.md` para o que conta como mudança MAJOR/MINOR/PATCH e
`../../docs/CHANGELOG.md` para o histórico. Versão atual: **0.16.0**.

## Stack

- Bun (runtime + package manager + bundler + test runner)
- TypeScript (strict mode, sem `any` implícito)
- Elysia (HTTP layer)
- Drizzle ORM + PostgreSQL (Neon em produção, staging e CI; Postgres do Docker local)
- Zod (validação — via Standard Schema do Elysia, não é o validador nativo)
- Better Auth (autenticação, session e organizações — cada organização é um tenant)
- Redis (cache, rate limit e secondary storage da sessão do Better Auth)
- Biome v2 (lint + format — substitui ESLint/Prettier)

## Comandos essenciais

```bash
bun install           # SEMPRE na raiz do monorepo
docker compose up -d  # na raiz: sobe Postgres + Redis localmente
bun dev               # bun --watch src/index.ts — reload automático, sem build
bun run build         # bun build src/index.ts --outdir dist --target bun
bun test              # test runner nativo do Bun
bun run check         # lint + format (Biome)
bun run typecheck     # tsc --noEmit (checagem de tipos)
bun run db:generate   # gerar migration a partir do schema Drizzle
bun run db:migrate    # aplicar migrations
bun run db:studio     # abrir Drizzle Studio para acessar/visualizar o banco
```

## Regras não-negociáveis (baseline travada — ver ../../docs/versioning.md)

Mudar qualquer item desta lista é uma mudança de versão do template, não
um ajuste de tarefa. Se uma tarefa parecer exigir quebrar uma regra
daqui, trate como proposta de mudança na baseline (processo em
`../../docs/versioning.md`), nunca como exceção silenciosa.

- Nunca commitar sem rodar `bun run check` antes (na raiz: o Biome é único).
- Toda feature nova precisa de `inputSchema`/`outputSchema` (Zod) e teste
  correspondente.
- Nunca usar `drizzle-kit push` fora de ambiente local (staging, CI e
  produção incluídos) — sempre `generate` + `migrate`.
- Nunca acessar o banco fora do `<action>.service.ts` — não existe camada de
  repository neste projeto; a feature (`<action>.ts`) nunca faz query direta.
- Nunca usar `process.env` diretamente — sempre pelo helper `env` validado
  com Zod (`lib/env.ts`; o tooling de CLI, ex: `drizzle.config.ts`, usa o
  módulo mínimo `lib/env-tooling.ts`, com a mesma regra).
- Erros de negócio são exceptions tipadas, nunca `throw new Error("string")`.
- Commits seguem Conventional Commits e passam pelo hook `commit-msg`
  (Husky + commitlint) — nunca commitar com `--no-verify`.
- Nunca push direto em `main` — todo merge passa por PR, mesmo solo.
- Nunca logar segredo (senha, token, header `Authorization`) — usar o
  redact do logger estruturado, nunca `console.log` em produção.
- Migrations em produção rodam **só** no job de CD, com a URL direta do
  Neon (`DATABASE_URL_UNPOOLED`) — nunca no start do container nem pelo
  pooler (`docs/architecture.md`, "### Neon"; `docs/ci-cd.md`).
- `bun test` nunca roda fora de banco/Redis de teste: `tests/setup.ts`
  (preload no `bunfig.toml`) aborta se `NODE_ENV` não for `test`, se o banco
  não terminar em `_test` ou se o Redis estiver no índice `0`
  (`docs/testing.md`).
- Toda API é multi-tenant (tenant = organização do Better Auth): toda
  tabela de negócio tem `organization_id NOT NULL` e todo acesso passa por
  `inTenant(table, ctx, …)` com o `TenantContext` — nunca consulta ou altera
  por `id` sozinho, e recurso de outro tenant responde 404
  (`docs/architecture.md`, "### Multi-tenancy").
- Toda feature de negócio tem teste de isolamento entre tenants: a
  organização B não lê, altera nem exclui dado da organização A
  (`docs/testing.md`).
- Roles de negócio são por módulo (`user`, `editor`, `manager`, `admin`,
  em `user_module_roles`), dentro da organização. O dono é identificado só
  pela flag `members.is_owner` (única por organização) e vale como `admin`
  em qualquer módulo; a aplicação nunca lê `members.role`. Nenhuma role
  atravessa organizações — não existe role global nem "super admin" na API. Chave de Redis de dado de negócio leva o prefixo
  `t:<organization_id>:`; rate limit autenticado é por
  `(organization_id, user_id)`.
- Redis nunca é fonte de verdade — cache e sessão sempre reconstruíveis a
  partir do Postgres.
- Nunca commitar ou dar push no repositório de onde a instância foi
  clonada (o template) — antes do primeiro commit, confirmar o remote
  `origin` (`../../docs/git-workflow.md`) e resolver isso primeiro.

## Leia antes de tarefas específicas

- Decisões de arquitetura e camadas → `docs/architecture.md`
- Padrões de código e nomenclatura → `docs/conventions.md`
- Estratégia de testes → `docs/testing.md`
- Branch, commits, PR e merge → `../../docs/git-workflow.md`
- Pipeline de CI/CD → `docs/ci-cd.md`
- Dockerfile e infra local (Postgres/Redis) → `docs/docker.md`
- Logging estruturado e OpenTelemetry → `docs/observability.md`
- Checklist de nova feature, de PR e de bump de versão → `../../docs/checklists.md`
- Mecanismo de versão da baseline → `../../docs/versioning.md`
- Glossário e regras do domínio desta instância → `../../docs/domain.md`
- Features complexas → `../../docs/features/<nome>.md`

## O que NÃO fazer

- Não introduzir nova biblioteca sem checar se já existe equivalente na stack.
- Não duplicar lógica de validação entre camada HTTP e camada de service.
- Não deixar `console.log` em código de produção — usar o logger configurado.
- Não criar caminho sem tenant para dado de negócio: nada de modo
  single-tenant, tabela de negócio sem `organization_id`, rota de negócio
  sem `tenant: true`, nem role ou flag global que atravesse organizações.
  Precisar de um papel entre tenants é proposta de mudança de baseline
  (`../../docs/versioning.md`), não exceção local.

## Diferenças no monorepo

O que muda em relação à baseline api-bun v0.16.0 (o resto deste arquivo
e de `docs/` vale como está):

- **Workspace**: este app é `@repo/api` em `apps/api`. `bun install`,
  `docker compose` e o Biome (`bun run check`/`lint`) rodam **na raiz**.
  Os scripts do app (`dev`, `build`, `typecheck`, `db:*`, `bun test`)
  rodam aqui dentro ou pela raiz via turbo (`--filter=@repo/api`).
- **`bunfig.toml`** (`[run] bun = true` + preload de teste) fica em
  `apps/api/`, nunca na raiz (colocaria o Next no runtime do Bun).
- **Contrato**: a matriz de permissão (`permissions`, `Role`, `Action`,
  `can`), o corpo de erro e os valores de enum que aparecem na borda vêm
  de `@repo/contracts`. `resolveRole`, schema Drizzle e `TenantContext`
  continuam aqui. O `pgEnum` consome o array do contracts
  (`docs/architecture.md`, "## Diferenças no monorepo").
- **Bancos locais**: `app_dev` e `app_test` (em vez de `api_bun_*`), no
  `docker-compose.yml` da raiz (`../../docs/development.md`).
- **Testes no CI**: contra o banco `app_test` dentro do branch Neon da
  PR (`ci/pr-<n>`), não contra service container. A guarda `_test`
  continua igual (`../../docs/neon.md`, "## Branch por PR no CI").
- **Staging**: além de produção, existe a API de staging (branch Neon
  `staging`), deployada antes de produção pelo mesmo workflow.
- **Dockerfile**: `apps/api/Dockerfile`, com contexto de build na **raiz**
  do monorepo (`../../docs/deploy.md`, "### Dockerfile").
- **Workflows**: `ci.yml`, `api-build.yml` e `api-deploy.yml` na raiz
  (`../../docs/ci-cd.md`). `docs/ci-cd.md` deste app só guarda o que é
  específico da API.
- **Git workflow, versionamento, checklists, domínio, features**: só na
  raiz (`../../docs/`). Os links deste app já apontam para lá.

---

Versão da baseline: 0.16.0 (api-bun), dentro de turborepo-template
0.3.0. Ver `../../docs/CHANGELOG.md`.
