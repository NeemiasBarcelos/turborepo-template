# Changelog

Todas as mudanças notáveis na baseline deste template são registradas
aqui. Formato baseado em [Keep a Changelog](https://keepachangelog.com/),
versionamento conforme `docs/versioning.md`.

## [0.3.0] - 2026-10-10

Embute app-nextjs `0.1.0` e api-bun `0.16.0` (com correções à frente da
origem, listadas abaixo).

### Adicionado

- `docs/architecture.md`, "### Versões validadas": tabela de majors com
  que as docs foram conferidas (TypeScript 7, Next 16.4, Better Auth 1.7,
  ky 2, MSW 3, Vitest 5, Biome 2.5, shadcn 4…) e checklist "## Subir major
  de uma dependência".
- Contrato base do scaffold em `packages/contracts`: `modules = ['users']`
  (vira `pgEnum('module_name')`), `users.ts` (roles por módulo,
  `myRolesResponseSchema`), `organizations.ts` (transfer-ownership) e
  `session.ts` (`sessionSchema`).
- API: rotas fixas do módulo `users` (`GET /api/users/me/roles`,
  `GET /api/users/roles`, `PUT|DELETE /api/users/:userId/roles/:module`) e
  `POST /api/organizations/transfer-ownership` como parte da base técnica.
- API: rate limit de referência (`lib/rate-limit.ts`, plugin por IP no
  grupo `/api`, chave `(org, user)` no macro `tenant`), 120/min por IP e
  300/min por usuário/org, fail-open, `/api/auth/*` fora.
- API: `verification.storeInDatabase: true` ao lado de
  `session.storeSessionInDatabase: true`.
- API: padrão "organização pessoal no cadastro" como opção de instância
  (criada no `session.create.before`).
- API: passo a passo para regenerar o schema do Better Auth.
- Web: rotas e shell de base (`organizations/select` fora dos grupos,
  `OrganizationSwitcher`, `ThemeToggle`, `SignOutButton`, home com
  `GET /api/users/me/roles`).
- `docs/development.md`, "## Geradores sem TTY": comandos sem prompt para
  `create-next-app`, `shadcn init` e `auth generate`.
- `docs/deploy.md`: referência de deploy no Render e credencial do GHCR
  no host (também no checklist de setup).
- `turbo.json` com `"agentGuidance": false`.
- Biome: overrides de `scripts/**` (`noUndeclaredEnvVars`) e
  `apps/web/src/components/ui/**` (código do shadcn).

### Alterado

- API: `src/app.ts` monta os módulos sob `/api` e é exportado sem
  `.listen()`; `src/index.ts` só sobe o servidor e registra o shutdown.
- API: `authHandlerPlugin` (`.all('/api/auth/*')`) separado do
  `authPlugin` (só macros), e o IP do cliente repassado ao Better Auth por
  um header interno (`x-internal-client-ip`) que o cliente não forja.
- API: logger `silent` em teste e `pino-pretty` só em `development`;
  `onnotice` do postgres.js pelo logger.
- API: `afterEach` global (`resetDatabase` + `FLUSHDB`) no
  `tests/setup.ts`, depois da guarda; `createTenant` com timestamps.
- Web: sessão validada com `sessionSchema` do contracts; formulários com
  `Field` + `Controller`; shadcn com base Radix e preset Nova; E2E do
  scaffold só com fluxos de auth.
- Prompt inicial (`README.md`): base técnica inclui os módulos `users` e
  `organizations`, verificação inclui o E2E, aponta para os geradores sem
  TTY e para a tabela de versões.
- `.env.example` com opcionais comentadas e aviso de gerar
  `BETTER_AUTH_SECRET`; `TRUSTED_ORIGINS` no `.env.test`.
- `apps/api/Dockerfile`: `ARG BUN_VERSION` com default igual ao
  `.bun-version` (e item no checklist "Subir Bun").
- Checklist de bump inclui o cabeçalho e o rodapé de `apps/*/CLAUDE.md`.

### Corrigido

- `apps/*/CLAUDE.md` diziam "turborepo-template v0.1.0" depois do bump
  para 0.2.0.
- Better Auth: CLI `auth` (mesma versão do core) no lugar de
  `@better-auth/cli` 1.4; removida a instrução de acrescentar a coluna
  `issuer` em `accounts`, que quebra todo insert no core 1.7.3+.
- `.mount(auth.handler)` sem caminho engolia os 404 fora do formato de
  erro do contrato.
- ky 2: `baseUrl` no lugar de `prefixUrl`, `beforeError({ error })` e
  `error.data`.
- MSW 3: `onUnhandledFrame` (a opção antiga é ignorada em silêncio),
  `listen` no topo do setup, shim de `Request` para URL relativa no jsdom,
  `cleanup()` e mock de `next/navigation`.
- `create-next-app` 16.4 liga `cacheComponents`: lista do que remover
  depois de gerar e `next.config.ts` completo.
- Biome 2.5: `"preset": "recommended"`.
- `packages/contracts`: `types: ["bun"]` e `@types/bun` para o
  `permissions.test.ts` passar no typecheck.
- `docs/development.md`: o que fazer com porta 5432/6379 ocupada.

### Pendente de subir para a origem

Correções genéricas feitas aqui primeiro (decisão desta versão), ainda
não incorporadas nos templates de origem. Cada app lista os trechos em
"### Correções à frente de …" de `apps/<app>/docs/architecture.md`:

- **api-bun** (→ próxima versão): CLI `auth` e regeneração do schema, sem
  `issuer`; `verification.storeInDatabase`; `authHandlerPlugin` separado
  e IP interno; rate limit de referência; organização pessoal; `onnotice`;
  logger em teste; helpers e `afterEach` global de teste; `.env.example`
  com opcionais comentadas.
- **app-nextjs** (→ próxima versão): ky 2; shadcn 4 (`Field`, `cn`,
  `-b radix -p nova`); limpeza pós `create-next-app` 16.4; sessão
  validada com Zod; rotas e shell de base; setup do Vitest com MSW 3.

### Contexto

Primeira rodada de scaffold a partir da v0.2.0 (38 achados em
SCAFFOLD-NOTES, numa instância de teste). As docs assumiam versões de
lib anteriores às atuais e deixavam lacunas (prefixo `/api`, rate limit,
org pessoal, rotas base do web, setup de teste) que o scaffold teve de
resolver por conta própria. MINOR porque acrescenta regras e padrões
novos sem remover nenhuma regra não-negociável.

## [0.2.0] - 2026-10-09

Embute app-nextjs `0.1.0` e api-bun `0.16.0`.

### Adicionado

- Fluxo de tasks do Claude Code (`tasks/how-to-use.md`):
  - skills `.claude/skills/new-branch` (branch `<tipo>/<slug>` a partir
    da `main` atualizada + ambiente local) e `.claude/skills/update-main`;
  - comandos `.claude/commands/` `new-task`, `investigate`,
    `create-tickets`, `implement-ticket`, `adjust` e `close-task`;
  - `tasks/_templates/` (`spec.md`, `tickets.md`, `log.md`). Cada task
    fica em `tasks/<slug>/`, versionada.
- Decisão registrada em `docs/architecture.md`, referências em
  `CLAUDE.md`, `README.md` e `docs/git-workflow.md`.

### Contexto

Adaptado do fluxo usado no workspace new-music (vários repos, pnpm,
branches `task-N`) para um repo só, com a convenção de branch, commits e
gates de CI deste template.

## [0.1.0] - 2026-09-30

Embute app-nextjs `0.1.0` e api-bun `0.16.0`.

### Adicionado

- Baseline documentada inicial do monorepo:
  - `README.md`, `CLAUDE.md`, `SECURITY.md`, `LICENSE`;
  - `docs/` com architecture, neon, development, deploy, ci-cd,
    conventions, git-workflow, versioning, checklists, domain e
    features/.
- Monorepo Turborepo com Bun workspaces e um único `bun.lock`:
  `apps/web` (Next.js 16, Vercel), `apps/api` (Elysia + Bun, Docker),
  `packages/contracts` e `packages/tsconfig`.
- `packages/contracts` como fonte única do contrato web ↔ api (erro,
  matriz de permissão, enums de borda, schemas Zod de request/response),
  substituindo as cópias manuais dos templates separados.
- Neon como Postgres hospedado único: branches `main` (produção),
  `staging` e `ci/pr-<n>` (efêmero por PR, com `--expires-at`), operados
  pelo `neonctl` através dos wrappers `scripts/neon/*.ts`.
- CI com testes da API em branch Neon por PR (banco `app_test` dentro do
  branch), migrations validadas sobre cópia de produção e schema diff
  comentado na PR.
- Deploy da API em dois estágios (staging automático, produção com
  aprovação), com migrations antes de cada deploy e imagem construída a
  partir da raiz do monorepo.
- Docs de `apps/web` e `apps/api` copiadas das baselines de origem, com
  "## Diferenças no monorepo".

### Contexto

Os templates `app-nextjs` e `api-bun` já existiam como repositórios
separados e só documentados (sem scaffold completo). Este template junta
os dois num monorepo e resolve os conflitos entre eles:

- package manager: pnpm × Bun → Bun;
- estilo do Biome: web × api → estilo do api-bun;
- contrato: copiado à mão → pacote compartilhado.

Os conflitos estão registrados em `docs/architecture.md`, "## Decisões
registradas". O scaffold de código **ainda não foi gerado**. Ver o
prompt inicial no `README.md`.
