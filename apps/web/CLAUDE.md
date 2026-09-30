@AGENTS.md

# CLAUDE.md

> Baseado em app-nextjs v0.1.0, adaptado ao monorepo `turborepo-template`
> v0.1.0. Leia antes `../../CLAUDE.md` (regras da raiz, que vencem em caso
> de conflito) e "## Diferenças no monorepo" no fim deste arquivo.

Guia para o Claude ao trabalhar neste repositório. Este arquivo é a raiz;
regras específicas de arquitetura e convenções ficam em `/docs`. Leia os
arquivos referenciados quando a tarefa exigir.

## O que é este projeto

Template base para o **frontend Next.js** de produtos cujo backend é
derivado do template **api-bun** (multi-tenant: cada organização do Better
Auth é um tenant). Cada front real derivado deste template herda a stack e
as convenções abaixo; o domínio específico fica em `../../docs/domain.md` de
cada instância. Regra de negócio, persistência e autorização são da API,
nunca do front.

## Versão da baseline

Versionamento semântico próprio: ver `../../docs/versioning.md` para o que
conta como MAJOR/MINOR/PATCH e `../../docs/CHANGELOG.md` para o histórico.
Versão atual: **0.1.0** (compatível com api-bun `0.16.x`).

## Stack

- Next.js 16 (App Router, Turbopack, React Compiler), React 19
- TypeScript (strict, sem `any`)
- Tailwind CSS v4 + shadcn/ui
- TanStack Query (estado de servidor no client) + React Server Components
- ky (cliente HTTP, instâncias em `lib/api/`)
- nuqs (estado na URL)
- zustand (estado de UI, **só como exceção**, ver `docs/architecture.md`, "## Estado")
- React Hook Form + Zod (formulários, env e parse de resposta)
- Better Auth client + plugin `organization` (sessão e organização ativa do api-bun)
- Biome v2 (lint + format)
- Vitest + Testing Library + MSW; Playwright (E2E)
- Bun workspaces (package manager do monorepo; o Next roda em Node); deploy na Vercel

## Comandos essenciais

```bash
bun install          # SEMPRE na raiz do monorepo
bun run dev              # next dev (Turbopack) em http://localhost:3000
bun run build            # next build
bun run start            # next start (build de produção local)
bun run check            # biome check --write (lint + format + imports)
bun run lint             # biome check (sem escrever; o que o CI roda)
bun run typecheck        # next typegen && tsc --noEmit
bun run test             # vitest run
bun run test:watch       # vitest
bun run test:e2e         # playwright test
```

Precisa do api-bun rodando localmente (`API_URL`, ver
`docs/architecture.md`, "## Variáveis de ambiente").

## Regras não-negociáveis (baseline travada, ver ../../docs/versioning.md)

Mudar qualquer item desta lista é mudança de versão do template, não
ajuste de tarefa. Se uma tarefa parecer exigir quebrar uma regra daqui,
trate como proposta de mudança na baseline (`../../docs/versioning.md`), nunca
como exceção silenciosa.

- Antes de ler ou escrever código que usa API do Next.js, consultar
  `node_modules/next/dist/docs/` (Next 16: `proxy.ts` e não `middleware.ts`;
  `params`/`searchParams`/`cookies()`/`headers()` assíncronos).
- Nunca commitar sem `bun run lint` e `bun run typecheck` passando.
- Bun é o único package manager, com um só `bun.lock` na raiz do monorepo
  (`../../CLAUDE.md`). Nada de `package-lock.json`, `pnpm-lock.yaml` ou
  `yarn.lock`.
- Nunca usar `process.env` diretamente: sempre `env` (`lib/env.ts`, server)
  ou `clientEnv` (`lib/env.client.ts`), validados com Zod. Única exceção:
  `next.config.ts`, via `lib/env.schema.ts`.
- Nunca segredo em `NEXT_PUBLIC_*`.
- Todo request ao api-bun passa por `lib/api/` (`api` no browser,
  `serverApi()` no server). Nunca `fetch` solto nem ky importado numa
  feature. O browser só fala com a própria origem (`/api/*` via rewrite).
- Resposta da API é validada com Zod na fronteira, com os schemas de
  `@repo/contracts`. Tipos são inferidos dos schemas, nunca escritos à mão.
- Server Component por padrão; `"use client"` só na folha que precisa.
  Módulos só de server começam com `import 'server-only'`.
- Estado de servidor só em RSC ou TanStack Query, nunca copiado para
  `useState`, context ou zustand.
- Estado navegável (filtros, busca, paginação, aba) vai na URL via nuqs.
- zustand só para estado de UI, quando derivado/local/URL/context não
  resolvem, com justificativa na PR; store por módulo, nunca com dado da
  API ou de tenant.
- Formulário é React Hook Form + `zodResolver`; 400 com `issues` da API é
  mapeado de volta aos campos.
- Toda query key de dado de negócio começa pelo `organizationId`; trocar
  de organização limpa o cache do TanStack Query (`clear()`) e faz
  `router.refresh()`.
- Autorização é do api-bun. O front só esconde/desabilita
  (`can` de `@repo/contracts/permissions`) e sempre trata 403. `proxy.ts` só faz checagem
  otimista de cookie, nunca autorização. O front nunca lê `members.role`
  para decidir permissão.
- Sem `'use cache'`/`cacheComponents` para dado autenticado ou de tenant.
- Nunca logar cookie, token, header `Authorization` nem dado pessoal;
  nunca `console.log` em código de produção.
- Commits em Conventional Commits pelo hook `commit-msg` (Husky +
  commitlint); nunca `--no-verify`. Nunca push direto em `main`: todo merge
  passa por PR com squash.
- Nunca commitar ou dar push no repositório de onde a instância foi
  clonada (o template): antes do primeiro commit, confirmar o remote
  `origin` (`../../docs/git-workflow.md`).

## Leia antes de tarefas específicas

- Estrutura, integração com o api-bun, auth, estado, dados → `docs/architecture.md`
- Padrões de código, nomenclatura, formulários, estilo → `docs/conventions.md`
- Estratégia de testes (Vitest, MSW, Playwright) → `docs/testing.md`
- Branch, commits, PR e merge → `../../docs/git-workflow.md`
- CI (GitHub Actions) e deploy (Vercel) → `docs/ci-cd.md`
- Logging, `instrumentation.ts`, métricas → `docs/observability.md`
- Checklist de feature, de PR e de bump de versão → `../../docs/checklists.md`
- Mecanismo de versão da baseline → `../../docs/versioning.md`
- Glossário e decisões de produto desta instância → `../../docs/domain.md`
- Features complexas → `../../docs/features/<nome>.md`

## O que NÃO fazer

- Não introduzir biblioteca nova sem checar se já existe equivalente na
  stack (ex: não adicionar axios, SWR, Redux, Formik, outra lib de
  componentes).
- Não reimplementar no front regra de negócio ou validação que é da API:
  o schema do front é UX, quem decide é o api-bun.
- Não reimplementar endpoints do Better Auth (login, organização, convite):
  usar o `authClient`.
- Não importar de uma `features/<modulo>` dentro de outra.
- Não usar `useEffect` para buscar dado ou derivar estado.
- Não escrever `useMemo`/`useCallback`/`memo` defensivos (React Compiler).

## Diferenças no monorepo

O que muda em relação à baseline app-nextjs v0.1.0 (o resto deste arquivo
e de `docs/` vale como está):

- **"api-bun" nestas docs é o `apps/api` deste repositório.** As docs
  dele estão em `../api/CLAUDE.md` e `../api/docs/`.
- **Package manager**: Bun workspaces na raiz, não pnpm. `bun install` só
  na raiz, e os comandos acima rodam de dentro de `apps/web` com
  `bun run <script>` ou da raiz com `turbo run <task> --filter=@repo/web`.
  O Next continua em **Node** (local e Vercel). Nunca pôr
  `[run] bun = true` num `bunfig.toml` que alcance o web.
- **Contrato**: `lib/permissions.ts` e os tipos do corpo de erro deixam
  de ser cópia manual. Eles vêm de `@repo/contracts` (`permissions`,
  `errors`), e os schemas de resposta de cada módulo vêm de
  `@repo/contracts/<modulo>`. O `next.config.ts` declara
  `transpilePackages: ['@repo/contracts']`.
- **Biome**: `biome.json` do web estende o da raiz (`"extends": "//"`),
  com tabs, aspas simples e `semicolons: "asNeeded"`. Os exemplos em
  `docs/` mantêm o estilo original. O `bun run check` normaliza.
- **tsconfig**: estende `@repo/tsconfig/nextjs.json`, mantendo
  `paths` `@/*` → `./src/*` aqui.
- **Lint, format, Husky, commitlint, `.vscode/`, `.nvmrc`**: vivem na raiz
  (`../../docs/conventions.md`).
- **API local**: é o `apps/api` do mesmo repo. `bun run dev` na raiz sobe
  os dois.
- **E2E**: o Playwright sobe a API do monorepo como segundo `webServer`,
  contra banco `_test` (`docs/testing.md`, "## Diferenças no monorepo").
- **CI/CD e deploy**: os workflows ficam em `../../docs/ci-cd.md`, e a
  config da Vercel (Root Directory `apps/web`, install com Bun, build via
  turbo) em `../../docs/deploy.md`. `docs/ci-cd.md` deste app só guarda o
  que é específico do web.
- **Git workflow, versionamento, checklists, domínio, features**: só na
  raiz (`../../docs/`). Os links deste app já apontam para lá.

---

Versão da baseline: 0.1.0 (app-nextjs), dentro de turborepo-template 0.1.0.
Ver `../../docs/CHANGELOG.md`.
