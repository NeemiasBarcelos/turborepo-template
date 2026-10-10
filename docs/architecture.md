# Arquitetura do monorepo

Este documento cobre só o que é **transversal** ao monorepo: layout,
workspaces, pipeline do Turborepo, pacotes compartilhados e como as peças
se conectam. A arquitetura interna de cada app continua documentada no
próprio app:

- `apps/web/docs/architecture.md`: frontend Next.js (baseline app-nextjs).
- `apps/api/docs/architecture.md`: API Bun/Elysia (baseline api-bun).

Quando uma regra daqui conflita com a doc de um app, **esta vence**. Cada
app lista seus desvios em "## Diferenças no monorepo" no próprio
`CLAUDE.md` e `docs/architecture.md`.

## Visão geral

```
                      ┌──────────────────────────┐
  browser ──────────▶ │ apps/web (Next.js)       │  Vercel
                      │ /api/* → rewrite         │
                      └────────────┬─────────────┘
                                   │ API_URL (server-side)
                      ┌────────────▼─────────────┐
                      │ apps/api (Elysia + Bun)  │  Docker: DigitalOcean,
                      │ /api/*, /health, /ready  │  Railway, Render…
                      └──────┬─────────────┬─────┘
                             │             │
                   DATABASE_URL (pooled)   REDIS_URL (rediss://)
                             │             │
                      ┌──────▼──────┐  ┌───▼────────────┐
                      │ Neon        │  │ Redis gerenc.  │
                      │ Postgres 16 │  └────────────────┘
                      └─────────────┘
```

- O **web** nunca fala com o banco nem com o Redis. Tudo passa pela API.
- O **browser** só fala com a origem do web. `/api/*` é reescrito pelo
  Next para `API_URL`, então os cookies do Better Auth são first-party e
  não há CORS no caminho normal (`apps/web/docs/architecture.md`,
  "### Mesma origem via rewrite").
- A **API** é a dona da regra de negócio, da autorização e do schema do
  banco (Drizzle). Ela é multi-tenant: tenant = organização do Better
  Auth (`apps/api/docs/architecture.md`, "### Multi-tenancy").
- O **Neon** é o único Postgres hospedado. Produção, staging e CI são
  branches do mesmo projeto Neon (`docs/neon.md`). Localmente o Postgres
  roda no Docker, com a mesma major (`docs/development.md`).

## Estrutura de pastas

```
.
├── apps/
│   ├── web/                    # @repo/web: Next.js 16 (Vercel)
│   │   ├── CLAUDE.md, AGENTS.md
│   │   ├── docs/               # baseline app-nextjs + diferenças no monorepo
│   │   ├── src/
│   │   ├── biome.json          # estende a raiz ("extends": "//")
│   │   ├── next.config.ts
│   │   ├── package.json
│   │   └── tsconfig.json       # estende @repo/tsconfig/nextjs.json
│   └── api/                    # @repo/api: Elysia + Bun (Docker)
│       ├── CLAUDE.md
│       ├── docs/               # baseline api-bun + diferenças no monorepo
│       ├── src/
│       ├── tests/
│       ├── Dockerfile          # build a partir da RAIZ do repo (docs/deploy.md)
│       ├── bunfig.toml         # [run] bun = true, preload de teste: só da API
│       ├── drizzle.config.ts
│       ├── package.json
│       └── tsconfig.json       # estende @repo/tsconfig/bun.json
├── packages/
│   ├── contracts/              # @repo/contracts: contrato web ↔ api
│   └── tsconfig/               # @repo/tsconfig: bases de tsconfig
├── scripts/
│   └── neon/                   # wrappers do neonctl (docs/neon.md)
├── docker/
│   └── postgres-init/          # cria app_test no Postgres local
├── docs/                       # docs transversais (este diretório)
├── .github/
│   ├── workflows/              # ci, neon-cleanup, api-build, api-deploy
│   └── dependabot.yml
├── .bun-version                # versão única do Bun (CI, Docker, local)
├── .nvmrc                      # Node do Next (mesma major da Vercel)
├── .dockerignore               # contexto de build da API é a raiz
├── biome.json                  # config raiz ("root": true)
├── bun.lock                    # ÚNICO lockfile do repositório
├── docker-compose.yml          # Postgres 16 + Redis 7 locais
├── package.json                # workspaces, packageManager, scripts turbo
└── turbo.json
```

Regras de estrutura:

- **Só dois tipos de workspace**: `apps/*` (deployáveis) e `packages/*`
  (bibliotecas internas). Nada de código compartilhado fora de
  `packages/`.
- **App nunca importa de outro app.** Nem por caminho relativo
  (`../../api/src/...`), nem declarando `@repo/api` como dependência do
  web. O que os dois precisam ver vai para `packages/contracts`.
- **Pacote nunca importa de app.** A direção é sempre
  `apps/* → packages/*`.
- **Docs de app ficam no app.** A pasta `docs/` da raiz não repete o que
  está em `apps/*/docs/`, só referencia.

## Workspaces e package manager

Bun workspaces. **Um único lockfile**: `bun.lock` na raiz.

```jsonc
// package.json (raiz)
{
  "name": "turborepo-template",
  "private": true,
  "packageManager": "bun@1.3.11",
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "check": "biome check --write",
    "lint": "biome check",
    "typecheck": "turbo run typecheck",
    "test": "turbo run test",
    "db:generate": "turbo run db:generate --filter=@repo/api",
    "db:migrate": "turbo run db:migrate --filter=@repo/api",
    "db:studio": "bun run --cwd apps/api db:studio",
    "neon:branch": "bun scripts/neon/branch.ts",
    "neon:urls": "bun scripts/neon/urls.ts",
    "neon:test-db": "bun scripts/neon/test-db.ts",
    "neon:diff": "bun scripts/neon/diff.ts",
    "neon:delete": "bun scripts/neon/delete.ts",
    "prepare": "husky"
  },
  "devDependencies": {
    "@biomejs/biome": "…",
    "@commitlint/cli": "…",
    "@commitlint/config-conventional": "…",
    "@types/bun": "…",
    "husky": "…",
    "turbo": "…",
    "typescript": "…"
  }
}
```

- `packageManager` fixa a versão do Bun para o Turborepo e para a Vercel.
  `.bun-version` tem **o mesmo valor** e é lido pelo `setup-bun` no CI e
  pelo `ARG BUN_VERSION` do Dockerfile. Ao subir o Bun, os dois mudam na
  mesma PR (`docs/checklists.md`).
- **Dependência interna** é declarada com `workspace:*`:
  `"@repo/contracts": "workspace:*"`.
- **Versão única por dependência compartilhada.** `zod`, `typescript` e
  `better-auth` precisam ter a mesma versão em todos os workspaces que os
  usam. Duas versões de `zod` quebram `instanceof` e a inferência de tipos
  entre `contracts` e os apps. Na dúvida, `bun pm ls <pacote>` mostra as
  cópias instaladas.
- Ferramentas usadas pelo repositório inteiro (Biome, Husky, commitlint,
  turbo, TypeScript) ficam no `package.json` da raiz. Dependências de
  runtime ficam no workspace que as usa.
- **Sem `bunfig.toml` na raiz com `[run] bun = true`.** Essa opção faz
  scripts com shebang de Node rodarem no runtime do Bun. No web isso
  colocaria o `next` no Bun, e a Vercel roda o Next em Node. A opção vive
  só em `apps/api/bunfig.toml`.

### Versões validadas

O `"…"` dos snippets não quer dizer "qualquer versão". As docs foram
validadas num scaffold real com as majors abaixo. Um scaffold novo instala
o `latest` de cada pacote e **confere a major com esta tabela**. Major
diferente da tabela (ex: `typescript@latest` virou o port nativo) é sinal
de que algum snippet pode estar desatualizado: ler o changelog da lib antes
de seguir e, se precisar de correção, registrar como achado para a
baseline.

| Pacote | Versão validada | Onde | Observação |
|---|---|---|---|
| Bun | 1.3.11 | raiz | `.bun-version` + `packageManager` |
| Node | 24 | web (Vercel) | `.nvmrc` |
| `typescript` | 7.0.x | raiz | port nativo |
| `turbo` | 2.11.x | raiz | `agentGuidance: false` (ver abaixo) |
| `@biomejs/biome` | 2.5.x | raiz | `"preset": "recommended"` |
| `next` / `react` | 16.4.x / 19.3.x | web | sem `cacheComponents` |
| `tailwindcss` | 4.3.x | web | via `@tailwindcss/turbopack` |
| `shadcn` | 4.x | web | base `radix`, pacote `cn` |
| `ky` | 2.x | web | `baseUrl`, `error.data` |
| `@tanstack/react-query` | 5.x | web | |
| `react-hook-form` | 7.x | web | com `Field` do shadcn |
| `nuqs` | 2.x | web | |
| `vitest` | 5.x | web | |
| `msw` | 3.x | web | `onUnhandledFrame` |
| `jsdom` | 30.x | web | |
| `@testing-library/jest-dom` | 7.x | web | |
| `@playwright/test` | 1.64.x | web | |
| `elysia` | 1.4.x | api | `@elysia/cors`, `@elysia/openapi` 1.4 |
| `better-auth` / `auth` (CLI) | 1.7.x | api, web | CLI `auth` na **mesma** versão do core |
| `drizzle-orm` / `drizzle-kit` | 0.45.x / 0.31.x | api | |
| `postgres` | 3.4.x | api | |
| `pino` | 10.x | api | |
| `zod` | 4.x | todos | mesma versão nos três workspaces |

Ao subir uma major de propósito, a PR atualiza esta tabela e os snippets
afetados (`docs/checklists.md`, "## Subir Bun, Node ou `neonctl`").

## Pipeline do Turborepo

```jsonc
// turbo.json
{
  "$schema": "https://turborepo.com/schema.json",
  "ui": "tui",
  "agentGuidance": false,
  "globalDependencies": [".bun-version"],
  "globalEnv": ["NODE_ENV"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"],
      "env": ["API_URL", "NEXT_PUBLIC_*"]
    },
    "typecheck": {
      "dependsOn": ["^typecheck"]
    },
    "test": {
      "dependsOn": ["^build"],
      "env": [
        "DATABASE_URL",
        "DATABASE_URL_UNPOOLED",
        "REDIS_URL",
        "BETTER_AUTH_SECRET",
        "BETTER_AUTH_URL"
      ]
    },
    "test:e2e": {
      "dependsOn": ["^build"],
      "cache": false,
      "env": [
        "API_URL",
        "NEXT_PUBLIC_*",
        "DATABASE_URL",
        "DATABASE_URL_UNPOOLED",
        "REDIS_URL",
        "BETTER_AUTH_*",
        "TRUSTED_ORIGINS"
      ]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "db:generate": { "cache": false },
    "db:migrate": { "cache": false, "env": ["DATABASE_URL", "DATABASE_URL_UNPOOLED"] }
  }
}
```

- **`agentGuidance: false`.** O turbo 2.11+ cria (e recria) um
  `AGENTS.md` na raiz quando detecta um agente de IA. A raiz deste
  template já tem `CLAUDE.md` como guia único, então o recurso fica
  desligado. O `apps/web/AGENTS.md` (gerado pelo Next) continua versionado.
- **Lint e format não passam pelo turbo.** O Biome roda uma vez na raiz
  (`bun run lint`/`bun run check`), e é mais rápido assim do que N
  execuções por workspace.
- **`env` por task é obrigatório para variável que muda o resultado.** O
  turbo roda em *strict env mode*: a task só enxerga as variáveis
  declaradas em `env`/`globalEnv`. Variável lida no build sem estar
  declarada vira cache hit errado (build com `API_URL` antigo) ou falha
  de validação do Zod. Nova variável de build ou teste entra no
  `turbo.json` na mesma PR que entra no schema de env.
- **`db:*` nunca é cacheado.** Migration tem efeito colateral fora do repo.
- **`dev` é `persistent`.** `bun run dev` na raiz sobe web (:3000) e api
  (:3333) juntos na TUI do turbo.
- Pacotes em `packages/*` são "just-in-time" (ver abaixo) e não têm task
  `build`. O `dependsOn: ["^build"]` é inofensivo com eles e passa a valer
  se um dia algum pacote precisar de build.
- **Remote cache** (opcional): Vercel Remote Cache, com `TURBO_TOKEN` e
  `TURBO_TEAM` como secret/var do GitHub. Sem eles, o turbo usa só o cache
  local e o CI reconstrói tudo.

Filtros mais usados:

```bash
bun run dev --filter=@repo/api           # só a API
turbo run test --filter=@repo/web        # testes do web
turbo run build --affected               # só o que mudou em relação a main
turbo run typecheck --filter=...@repo/contracts  # contracts e quem depende dele
```

## Pacotes compartilhados

### `packages/tsconfig`

Bases de `tsconfig` sem código:

```
packages/tsconfig/
├── package.json        # "name": "@repo/tsconfig", "private": true
├── base.json           # strict, noUncheckedIndexedAccess, isolatedModules, skipLibCheck
├── nextjs.json         # extends base; jsx, lib dom, plugin next, moduleResolution bundler
└── bun.json            # extends base; types bun, moduleResolution bundler
```

```jsonc
// apps/api/tsconfig.json
{
  "extends": "@repo/tsconfig/bun.json",
  "compilerOptions": {
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src", "tests", "drizzle.config.ts"]
}
```

- `paths` fica **no app**, nunca na base: `@/*` resolve relativo ao
  `tsconfig` que o declara, e cada app tem o próprio `src/`.
- Sem `baseUrl` (decisão herdada do api-bun, deprecação do TS 6).

### `packages/contracts`

O contrato entre web e api. Até aqui ele era **copiado à mão** entre os
dois templates, com a regra "ao mudar no api-bun, mudar no front no mesmo
ciclo". No monorepo, a cópia acaba: os dois apps importam deste pacote.

```
packages/contracts/
├── package.json
├── tsconfig.json          # estende @repo/tsconfig/base.json, types: ["bun"]
└── src/
    ├── errors.ts          # apiErrorSchema, ApiErrorBody
    ├── permissions.ts     # permissions, Role, Action, can()
    ├── permissions.test.ts
    ├── roles.ts           # moduleRoles (valores do enum de role por módulo)
    ├── modules.ts         # módulos com role própria: ['users', ...]
    ├── session.ts         # sessionSchema (resposta de /api/auth/get-session)
    ├── users.ts           # contrato base do módulo de administração de roles
    ├── organizations.ts   # transfer-ownership
    └── <modulo>.ts        # schemas Zod de request/response de cada módulo
```

```jsonc
// packages/contracts/package.json
{
  "name": "@repo/contracts",
  "private": true,
  "type": "module",
  "exports": {
    "./errors": "./src/errors.ts",
    "./permissions": "./src/permissions.ts",
    "./roles": "./src/roles.ts",
    "./modules": "./src/modules.ts",
    "./*": "./src/*.ts"
  },
  "scripts": {
    "typecheck": "tsc --noEmit",
    "test": "bun test"
  },
  "dependencies": {
    "zod": "…"
  },
  "devDependencies": {
    "@repo/tsconfig": "workspace:*",
    "@types/bun": "…"
  }
}
```

```jsonc
// packages/contracts/tsconfig.json
{
  "extends": "@repo/tsconfig/base.json",
  "compilerOptions": {
    // os testes do pacote usam bun:test; o código de src/ não depende do Bun
    "types": ["bun"]
  },
  "include": ["src"]
}
```

Sem `types: ["bun"]` e `@types/bun`, o `tsc --noEmit` do pacote falha em
`permissions.test.ts` com `TS2307: Cannot find module 'bun:test'`.

Pacote **just-in-time**: exporta TypeScript direto, sem build. O Bun
executa TS nativamente. O Next transpila o pacote se ele estiver em
`transpilePackages: ['@repo/contracts']` no `next.config.ts`. No bundle
de produção da API, o `bun build` empacota o pacote junto com o app.

```ts
// packages/contracts/src/errors.ts
import { z } from 'zod/v4'

export const apiErrorSchema = z.object({
	statusCode: z.number().int(),
	error: z.string(),
	message: z.string(),
	issues: z.array(z.object({ path: z.string(), message: z.string() })).optional(),
})

export type ApiErrorBody = z.infer<typeof apiErrorSchema>
```

```ts
// packages/contracts/src/roles.ts
export const moduleRoles = ['user', 'editor', 'manager', 'admin'] as const
export type ModuleRole = (typeof moduleRoles)[number]
```

```ts
// packages/contracts/src/modules.ts
/**
 * Módulos que têm role própria em `user_module_roles`. Mesmo nome em
 * apps/api/src/modules, apps/web/src/features e packages/contracts/src.
 * `users` é o módulo de administração (roles por módulo).
 */
export const modules = ['users'] as const
export type Module = (typeof modules)[number]
```

```ts
// apps/api/src/db/schema/roles.ts
import { moduleRoles } from '@repo/contracts/roles'
import { modules } from '@repo/contracts/modules'
import { pgEnum } from 'drizzle-orm/pg-core'

export const moduleRole = pgEnum('module_role', moduleRoles)
export const moduleName = pgEnum('module_name', modules)
```

Cada módulo de negócio novo entra no array `modules` (e gera migration do
enum `module_name`) na mesma PR que cria o módulo.

#### Contrato base do scaffold

Todo scaffold já nasce com estes arquivos, porque a API da baseline expõe
os endpoints de administração de roles e de transferência de dono
(`apps/api/docs/architecture.md`, "### Gerenciamento de roles e membros"):

```ts
// packages/contracts/src/users.ts
import { z } from 'zod/v4'
import { modules } from './modules'
import { moduleRoles } from './roles'

export const moduleSchema = z.enum(modules)
export const moduleRoleSchema = z.enum(moduleRoles)

export const userModuleRoleSchema = z.object({
	userId: z.uuid(),
	module: moduleSchema,
	role: moduleRoleSchema,
})
export type UserModuleRole = z.infer<typeof userModuleRoleSchema>

export const listUserRolesResponseSchema = z.object({
	items: z.array(userModuleRoleSchema),
})

export const userRoleParamsSchema = z.object({
	userId: z.uuid(),
	module: moduleSchema,
})

export const assignUserRoleRequestSchema = z.object({
	role: moduleRoleSchema,
})

/** Roles do usuário logado na organização ativa, por módulo (o web usa com `can`). */
export const myRolesResponseSchema = z.object({
	organizationId: z.uuid(),
	isOwner: z.boolean(),
	roles: z.record(moduleSchema, moduleRoleSchema),
})
export type MyRoles = z.infer<typeof myRolesResponseSchema>
```

```ts
// packages/contracts/src/organizations.ts
import { z } from 'zod/v4'

/** Módulo `organizations`: o que não é endpoint do plugin do Better Auth. */
export const transferOwnershipRequestSchema = z.object({
	userId: z.uuid(),
})

export const transferOwnershipResponseSchema = z.object({
	organizationId: z.uuid(),
	ownerUserId: z.uuid(),
})
```

```ts
// packages/contracts/src/session.ts
import { z } from 'zod/v4'

/**
 * Resposta de GET /api/auth/get-session (Better Auth), só com os campos que
 * o web usa. Sem sessão, o endpoint devolve `null`: o web valida com
 * `sessionSchema.nullable()`.
 */
export const sessionSchema = z.object({
	session: z.object({ activeOrganizationId: z.string().nullish() }),
	user: z.object({ id: z.string(), name: z.string(), email: z.string() }),
})
export type Session = z.infer<typeof sessionSchema>
```

**O que entra** em `contracts`:

- Formato do corpo de erro (`{ statusCode, error, message, issues? }`).
- Matriz de permissão (`permissions`, `Role`, `Action`, `can`). A função
  `resolveRole`, que consulta o banco, **fica na API**.
- Valores de enum que aparecem na borda da API (roles por módulo, status
  de entidades). O `pgEnum` da API passa a consumir o array do
  `contracts`, e não o contrário.
- Schemas Zod de **request e response** que atravessam a rede, um arquivo
  por módulo, com o mesmo nome do módulo nos dois apps.
- O schema mínimo da sessão (`session.ts`): é resposta da API ao web,
  mesmo vindo do Better Auth, e o web a valida na fronteira.

**O que não entra**:

- Schema Drizzle, `db`, `TenantContext`, qualquer coisa que importe
  `drizzle-orm`, `postgres`, `better-auth` server ou `elysia`. O web
  importa o `contracts`, e isso não pode puxar driver de banco para o
  bundle do browser. A única dependência de runtime permitida é `zod`.
- Schema de formulário do web (é UX, pode ser mais restritivo ou ter
  campos só de tela) e regra de negócio da API. O formulário pode
  **derivar** do schema do contrato (`.pick`, `.extend`), mas vive em
  `apps/web/src/features/<modulo>/schemas/`.
- Qualquer `import` com alias `@/`. Pacote JIT é compilado no contexto do
  app consumidor, e `@/` apontaria para o `src/` do app errado. Dentro de
  `packages/*` só há import relativo ou de pacote.

A API usa os schemas do `contracts` como `outputSchema` e como base do
`inputSchema` das features. O web usa os mesmos schemas no `.parse` da
fronteira (`apps/web/docs/architecture.md`, "### Validação de resposta").
Uma mudança de contrato que quebra o outro lado aparece no `typecheck`
da PR, não em produção.

## Comunicação web → api

Mantida como no app-nextjs:

- O browser chama `/api/*` na origem do web. O `rewrites()` do
  `next.config.ts` manda para `${API_URL}/api/*`.
- O server do Next (RSC, route handlers, server actions) chama
  `serverApi()`, que repassa o cookie da request para `API_URL`.
- A API serve tudo sob `/api` (Better Auth em `/api/auth/*`) e lista a
  origem do web em `TRUSTED_ORIGINS`.

| Ambiente | Web | `API_URL` do web | `TRUSTED_ORIGINS` da API |
|---|---|---|---|
| Local | `http://localhost:3000` | `http://localhost:3333` | `http://localhost:3000` |
| CI (E2E) | `http://localhost:3000` | `http://localhost:3333` | `http://localhost:3000` |
| Preview (Vercel) | URL de preview | API de **staging** | domínio de preview fixo (`docs/deploy.md`) |
| Produção | domínio de produção | API de produção | domínio de produção |

## Ambientes e branches do Neon

| Ambiente | Web | API | Branch Neon |
|---|---|---|---|
| Local | `bun run dev` | `bun run dev` | nenhum: Postgres do Docker (`app_dev`, `app_test`) |
| CI de PR | build + E2E no job | testes no job | `ci/pr-<n>` (efêmero, `docs/neon.md`) |
| Staging | previews da Vercel | deploy automático de `main` | `staging` (longo, filho de `main`) |
| Produção | Vercel, `main` | deploy de `main` com aprovação | `main` (default do projeto) |

Detalhes em `docs/neon.md` (branches), `docs/ci-cd.md` (workflows) e
`docs/deploy.md` (Vercel e host da API).

## Variáveis de ambiente

Cada app valida o próprio env com Zod, como nos templates
(`apps/web/src/lib/env*.ts`, `apps/api/src/lib/env.ts`). O monorepo não
cria env compartilhado, e cada app tem os próprios arquivos:

```
apps/web/.env.local        # gitignored: API_URL, NEXT_PUBLIC_APP_URL
apps/web/.env.example      # commitado
apps/api/.env.local        # gitignored: DATABASE_URL, REDIS_URL, BETTER_AUTH_*…
apps/api/.env.test         # commitado: banco app_test, Redis índice 1
apps/api/.env.example      # commitado
```

- Nunca `.env` na raiz. O Bun carrega `.env*` do **diretório de trabalho**
  e o turbo roda cada task no diretório do workspace, então um `.env` na
  raiz não é lido pelos apps e só confunde.
- O `neonctl link` grava `DATABASE_URL` num `.env` por padrão. No
  monorepo, usar sempre `--no-env-pull` (`docs/neon.md`).
- Variável de CI e de deploy (`NEON_API_KEY`, `DATABASE_URL_UNPOOLED` de
  produção…) vive no GitHub, na Vercel ou no host da API, nunca no repo.

## Decisões registradas

### Bun como package manager do monorepo inteiro

**Decisão**: Bun workspaces com um único `bun.lock` para web, api e
pacotes. O web continua rodando o Next em **Node** (local e Vercel). O Bun
só instala dependências e dispara scripts.
**Contexto**: o app-nextjs escolheu pnpm e proíbe `bun.lock`. O api-bun
usa Bun para tudo e proíbe outro lockfile. Num monorepo só cabe um
package manager. Bun já é obrigatório para a API (runtime, bundler, test
runner), a Vercel detecta `bun.lock` e instala com Bun, e o turbo suporta
Bun workspaces.
**Alternativas consideradas**: pnpm workspaces com a API rodando no Bun
(rejeitado: dois binários obrigatórios em dev, CI e Docker, e o
Dockerfile da API precisaria de pnpm só para instalar); manter os dois
apps em repos separados (rejeitado: é o cenário que obriga copiar o
contrato à mão).

### Contrato em `packages/contracts`, não copiado nem gerado

**Decisão**: erro, permissões, enums de borda e schemas Zod de
request/response vivem em `@repo/contracts`, importados pelos dois apps.
**Contexto**: nos templates separados, `lib/permissions.ts` do front era
cópia manual da matriz da API e os schemas de resposta eram reescritos no
front. No monorepo, a divergência vira erro de `typecheck` na mesma PR.
**Alternativas consideradas**: cliente gerado do OpenAPI (adiado: o
`/openapi` fica desligado em produção e a geração adiciona um passo de
build que o import direto dispensa); exportar o schema Drizzle para o web
(rejeitado: puxaria driver e detalhes de persistência para o bundle do
front).

### Pacotes internos just-in-time, sem build

**Decisão**: `packages/*` exportam `.ts` direto por `exports`. Não têm
`dist/` nem task `build`.
**Contexto**: são dois consumidores e ambos entendem TS (Bun nativo, Next
via `transpilePackages`). Um build intermediário só adiciona watch mode e
ordem de tasks.
**Alternativas consideradas**: pacote compilado com `tsc`/`tsup`
(rejeitado enquanto não houver consumidor fora do monorepo).

### Biome único na raiz, no estilo do api-bun

**Decisão**: um `biome.json` na raiz (`"root": true`) com tabs, aspas
simples e `semicolons: "asNeeded"`. O `apps/web/biome.json` estende a
raiz (`"extends": "//"`) só para ligar os domínios `next`/`react` e o
parser de diretivas do Tailwind.
**Contexto**: os templates tinham estilos opostos (web: 2 espaços e aspas
duplas; api: tabs e aspas simples). O `contracts` é importado pelos dois
e precisa de um estilo só. O api-bun declara o estilo explicitamente, e o
app-nextjs usava os defaults.
**Alternativas consideradas**: um estilo por app com configs aninhadas
independentes (rejeitado: o diff do `contracts` mudaria de estilo
conforme quem formatou); adotar o estilo do web (rejeitado: mais
arquivos de exemplo do api-bun precisariam mudar).

### Postgres local no Docker, não branch Neon por desenvolvedor

**Decisão**: desenvolvimento local usa o Postgres 16 do
`docker-compose.yml` (`app_dev` e `app_test`). O Neon é usado no CI
(branch por PR), no staging e em produção.
**Contexto**: os testes da API truncam tabelas a cada teste e o guard
exige banco `_test`. Contra o Postgres local isso é rápido e funciona
offline. Branch Neon por dev adicionaria latência de rede a cada teste e
custo de compute.
**Alternativas consideradas**: branch `dev/<nome>` por desenvolvedor via
`neonctl` (rejeitado como padrão; continua possível por instância, e
`docs/neon.md` mostra como apontar o local para um branch quando for
preciso depurar com dados reais).

### Testes do CI em branch Neon, não em service container

**Decisão**: o job de teste da API no CI cria (ou reaproveita) o branch
`ci/pr-<n>` no Neon, cria o banco `app_test` dentro dele e roda migrations
e testes lá. Só o Redis continua como service container.
**Contexto**: o objetivo é testar contra o mesmo Postgres de produção
(versão, extensões, TLS, pooler) e, no mesmo branch, aplicar as migrations
sobre uma cópia do schema e dos dados de produção antes do merge. Um
service container `postgres:16` não pega migration que falha com dado
real, nem diferença de configuração do Neon.
**Alternativas consideradas**: service container como no api-bun
(rejeitado pelo motivo acima); um branch fixo `ci` compartilhado por todas
as PRs (rejeitado: PRs simultâneas se atropelam no truncate).

### Previews da Vercel apontam para a API de staging

**Decisão**: todo preview do web usa `API_URL` = API de staging, que roda
a última versão de `main` contra o branch Neon `staging`. Não existe API
por PR.
**Contexto**: preview completo por PR (API + branch Neon por preview)
depende do host da API (Railway e Render têm ambientes de PR,
DigitalOcean não da mesma forma) e multiplica serviços rodando. A mudança
de API de uma PR é validada pelos testes no CI, contra o branch
`ci/pr-<n>`.
**Alternativas consideradas**: API efêmera por PR ligada ao preview da
Vercel (adiado: vale por instância quando o host oferece ambientes de PR,
registrando em `docs/domain.md`).

### Escopo `@repo/` para os workspaces

**Decisão**: workspaces se chamam `@repo/web`, `@repo/api`,
`@repo/contracts`, `@repo/tsconfig`.
**Contexto**: é a convenção dos exemplos do Turborepo. Os pacotes são
`private` e nunca publicados, então o escopo não precisa ser o nome do
produto.
**Alternativas consideradas**: escopo com o nome do produto (permitido
por instância, desde que troque em todos os `package.json` e imports de
uma vez).

### Imagem da API sem `turbo prune`

**Decisão**: o Dockerfile da API copia os `package.json` dos workspaces,
roda `bun install --frozen-lockfile --filter @repo/api` e empacota com
`bun build`. Não usa `turbo prune --docker` (`docs/deploy.md`).
**Contexto**: o `turbo prune` gera um `bun.lock` reduzido que tem issues
abertas no Turborepo (o `bun install --frozen-lockfile` reescreve o
lockfile podado). Como o `bun build` já empacota tudo e o runtime só leva
`dist/`, o ganho de cache do prune é pequeno.
**Alternativas consideradas**: `turbo prune @repo/api --docker`
(reavaliar quando o suporte a `bun.lock` estabilizar; a troca fica
restrita ao Dockerfile).

### Fluxo de tasks do Claude Code versionado no template

**Decisão**: o template distribui `.claude/skills/` (`new-branch`,
`update-main`), `.claude/commands/` (`/new-task`, `/investigate`,
`/create-tickets`, `/implement-ticket`, `/adjust`, `/close-task`) e
`tasks/_templates/` + `tasks/how-to-use.md`. Cada task vive em
`tasks/<slug>/` (spec, tickets e log), com `<slug>` igual ao da branch
`<tipo>/<slug>`, e é versionada junto com o código.
**Contexto**: o fluxo foi provado num workspace com vários repos
(new-music). Aqui é adaptado a um repo só: branch no padrão de
`docs/git-workflow.md`, gates do CI (`lint`, `typecheck`, `test`,
`build --affected`, `docker build` da API) e Conventional Commits. Os
comandos reforçam as regras não-negociáveis (sem push nem PR sem ok,
nunca `--no-verify`, banco local só, migration nunca por `push`).
**Alternativas consideradas**: branch `task-N` como no workspace de
origem (rejeitado: diverge da convenção `<tipo>/<slug>` do commitlint e
do título da PR); `tasks/` fora do git (rejeitado: num repo só, spec e
decisões servem de histórico na própria PR).

### Tabela de majors validadas, não versões soltas

**Decisão**: `docs/architecture.md`, "### Versões validadas", fixa a
major de cada dependência com que as docs foram conferidas num scaffold
real. Scaffold novo instala o `latest` e compara com a tabela.
**Contexto**: na primeira rodada de scaffold (v0.2.0), `typescript@latest`
já era o 7 (port nativo), e `vitest` 5, `msw` 3, `ky` 2, `jsdom` 30,
Better Auth 1.7 e `shadcn` 4 tinham mudado APIs que os snippets usavam.
Sem referência de versão, ninguém sabia se um erro era bug da doc ou da
lib.
**Alternativas consideradas**: fixar versões exatas nos snippets
(rejeitado: envelhece a cada release e a instância herdaria versões
velhas); "sempre latest" sem tabela (rejeitado: é o cenário que gerou os
achados).

### `agentGuidance: false` no `turbo.json`

**Decisão**: o turbo não gera o `AGENTS.md` da raiz.
**Contexto**: o turbo 2.11 cria (e recria se for apagado) um `AGENTS.md`
na raiz quando detecta um agente. A raiz já tem o `CLAUDE.md` como guia
único, e um segundo arquivo de instruções gerado pela ferramenta
competiria com ele. O `apps/web/AGENTS.md`, gerado pelo Next, continua
versionado e importado pelo `apps/web/CLAUDE.md`.
**Alternativas consideradas**: versionar o `AGENTS.md` da raiz e
importá-lo no `CLAUDE.md` (rejeitado: o conteúdo só manda ler as docs
empacotadas do turbo, o que o `CLAUDE.md` já cobre).

### Schema da sessão no `contracts`

**Decisão**: `sessionSchema` (resposta de `GET /api/auth/get-session`)
vive em `packages/contracts/src/session.ts`, só com os campos que o web
usa.
**Contexto**: o snippet de `auth-server.ts` do web citava um "tipo
exportado do schema de sessão" que não existia em lugar nenhum. A
resposta vem do Better Auth, mas atravessa a rede da API para o web, e a
regra não-negociável põe todo schema de resposta no `contracts`.
**Alternativas consideradas**: schema no `apps/web/src/lib/auth-server.ts`
(rejeitado: abriria exceção à regra do contrato); usar o tipo inferido do
client do Better Auth (rejeitado: tipo não valida em runtime).
