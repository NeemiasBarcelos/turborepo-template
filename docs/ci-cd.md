# CI/CD

GitHub Actions para validação e para a API. A Vercel cuida do deploy do
web (`docs/deploy.md`).

| Workflow | Gatilho | Faz |
|---|---|---|
| `ci.yml` | `pull_request` para `main` | lint, typecheck, testes (API em branch Neon), build, E2E, docker build, título da PR |
| `neon-cleanup.yml` | `pull_request: closed` | apaga o branch Neon `ci/pr-<n>` |
| `api-build.yml` | `push` em `main` que afeta a API | publica a imagem da API no GHCR |
| `api-deploy.yml` | `api-build` concluído com sucesso, ou manual | migrate + deploy em staging, depois (com aprovação) em produção |

Convenções dos workflows (as mesmas dos templates):

- **Bun fixo** por `.bun-version` (`oven-sh/setup-bun` com
  `bun-version-file`). **Node fixo** por `.nvmrc` onde o Next roda
  (build e E2E).
- `bun install --frozen-lockfile` sempre.
- `permissions:` mínimas no topo (`contents: read`), elevadas por job só
  quando precisa.
- `HUSKY: 0` no `env:`.
- Actions de terceiros aparecem aqui por tag. Na instância, fixar por
  **SHA** e deixar o Dependabot atualizar.
- **Turbo com `--affected`** nos jobs de monorepo. Precisa de histórico
  (`fetch-depth: 0`) para comparar com `main`. Com `TURBO_TOKEN`/
  `TURBO_TEAM` configurados, o cache remoto encurta os jobs.

## `ci.yml`: validação de PR

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened, edited]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

env:
  HUSKY: 0
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ vars.TURBO_TEAM }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with: { bun-version-file: .bun-version }
      - run: bun install --frozen-lockfile
      - run: bun run lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: oven-sh/setup-bun@v2
        with: { bun-version-file: .bun-version }
      - run: bun install --frozen-lockfile
      - run: bunx turbo run typecheck --affected

  test-web:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: oven-sh/setup-bun@v2
        with: { bun-version-file: .bun-version }
      - uses: actions/setup-node@v4
        with: { node-version-file: .nvmrc }
      - run: bun install --frozen-lockfile
      - run: bunx turbo run test --affected --filter=!@repo/api

  api-test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write # comentário do schema diff
    env:
      NEON_API_KEY: ${{ secrets.NEON_API_KEY }}
      NEON_PROJECT_ID: ${{ vars.NEON_PROJECT_ID }}
      NEON_BRANCH: ci/pr-${{ github.event.pull_request.number }}
    services:
      redis:
        image: redis:7
        ports: ['6379:6379']
        options: >-
          --health-cmd "redis-cli ping" --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with: { bun-version-file: .bun-version }
      - run: bun install --frozen-lockfile
      - run: bun add -g neonctl@2.24

      # 1. branch da PR (cria ou renova a expiração)
      - run: bun run neon:branch "$NEON_BRANCH"

      # 2. migrations sobre a cópia de produção (banco app)
      - run: bun run neon:urls "$NEON_BRANCH" app --github-env
      - run: bun run db:migrate

      # 3. schema diff contra main, como comentário na PR
      - run: bun run neon:diff "$NEON_BRANCH" > schema-diff.md
      - uses: marocchino/sticky-pull-request-comment@v2
        with:
          header: neon-schema-diff
          path: schema-diff.md

      # 4-5. banco de teste limpo no mesmo branch, migrations do zero e testes
      - run: bun run neon:test-db "$NEON_BRANCH" app_test
      - run: bun run neon:urls "$NEON_BRANCH" app_test --github-env # sobrescreve as URLs do passo 2
      - run: bun run db:migrate
      - run: bun test --coverage
        working-directory: apps/api
        env:
          NODE_ENV: test
          REDIS_URL: redis://localhost:6379/1
          BETTER_AUTH_SECRET: ci-test-secret-not-a-real-secret-0123456789
          BETTER_AUTH_URL: http://localhost:3333

  build:
    runs-on: ubuntu-latest
    env:
      # o env é validado no build; valores fictícios válidos
      API_URL: http://localhost:3333
      NEXT_PUBLIC_APP_URL: http://localhost:3000
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: oven-sh/setup-bun@v2
        with: { bun-version-file: .bun-version }
      - uses: actions/setup-node@v4
        with: { node-version-file: .nvmrc }
      - run: bun install --frozen-lockfile
      - run: bunx turbo run build --affected

  e2e:
    needs: api-test # reusa o branch Neon já criado e migrado
    runs-on: ubuntu-latest
    env:
      NEON_API_KEY: ${{ secrets.NEON_API_KEY }}
      NEON_PROJECT_ID: ${{ vars.NEON_PROJECT_ID }}
      NEON_BRANCH: ci/pr-${{ github.event.pull_request.number }}
    services:
      redis:
        image: redis:7
        ports: ['6379:6379']
        options: >-
          --health-cmd "redis-cli ping" --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with: { bun-version-file: .bun-version }
      - uses: actions/setup-node@v4
        with: { node-version-file: .nvmrc }
      - run: bun install --frozen-lockfile
      - run: bun add -g neonctl@2.24
      - run: bun run neon:test-db "$NEON_BRANCH" app_e2e_test
      - run: bun run neon:urls "$NEON_BRANCH" app_e2e_test --github-env
      - run: bun run db:migrate
      - run: bunx playwright install --with-deps chromium
        working-directory: apps/web
      - run: bunx turbo run test:e2e --filter=@repo/web
        env:
          # API sobe em modo teste pelo webServer do Playwright
          NODE_ENV: test
          REDIS_URL: redis://localhost:6379/2
          BETTER_AUTH_SECRET: ci-test-secret-not-a-real-secret-0123456789
          BETTER_AUTH_URL: http://localhost:3333
          TRUSTED_ORIGINS: http://localhost:3000
          API_URL: http://localhost:3333
          NEXT_PUBLIC_APP_URL: http://localhost:3000
      - if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: apps/web/playwright-report/

  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - id: bun
        run: echo "version=$(cat .bun-version)" >> "$GITHUB_OUTPUT"
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          context: .
          file: apps/api/Dockerfile
          push: false
          build-args: BUN_VERSION=${{ steps.bun.outputs.version }}
          cache-from: type=gha,scope=api
          cache-to: type=gha,scope=api,mode=max

  pr-title:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: read
    steps:
      - uses: amannn/action-semantic-pull-request@v5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Pontos que a baseline fixa:

- **`api-test` roda contra o Neon, não contra service container.** A
  sequência (branch → migrate em `app` → schema diff → `app_test` limpo
  → migrate do zero → `bun test`) e o motivo estão em `docs/neon.md`,
  "## Branch por PR no CI". O Redis continua service container, no índice
  `1`, que passa na guarda de teste do api-bun.
- **E2E usa outro banco no mesmo branch** (`app_e2e_test`) e outro índice
  do Redis (`/2`). Assim ele nunca disputa dados com o `api-test` quando
  os dois rodam em paralelo em re-runs. Os nomes terminam em `_test`, que
  a guarda exige.
- **URLs do Neon só via `neon:urls --github-env`.** O script mascara os
  valores e grava no `$GITHUB_ENV`. As variáveis valem a partir do passo
  seguinte, e uma gravação posterior sobrescreve a anterior (por isso o
  passo 5 troca `app` por `app_test`). Nunca `echo` de URL no YAML.
- **`api-test` e `e2e` não usam `--affected`.** Criar o branch é barato,
  e rodar sempre garante que o check obrigatório existe em toda PR.
  Instância com muitas PRs só de web pode condicionar o job a mudanças em
  `apps/api/**` ou `packages/**`. Nesse caso, o job pulado precisa
  continuar satisfazendo a branch protection (job pulado por `if:` conta
  como sucesso).
- **PR de fork não recebe secrets**: `api-test` e `e2e` falham nelas por
  design (`docs/neon.md`, "### Dados de produção no CI").
- `lint` roda o Biome uma vez na raiz, sem turbo.

## `neon-cleanup.yml`

```yaml
# .github/workflows/neon-cleanup.yml
name: Neon cleanup
on:
  pull_request:
    types: [closed]

permissions:
  contents: read

jobs:
  delete-branch:
    runs-on: ubuntu-latest
    env:
      NEON_API_KEY: ${{ secrets.NEON_API_KEY }}
      NEON_PROJECT_ID: ${{ vars.NEON_PROJECT_ID }}
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with: { bun-version-file: .bun-version }
      - run: bun install --frozen-lockfile
      - run: bun add -g neonctl@2.24
      - run: bun run neon:delete "ci/pr-${{ github.event.pull_request.number }}"
```

Falhou? O `--expires-at` apaga o branch em até 7 dias.

## `api-build.yml`: imagem da API

Igual ao `build.yml` do api-bun (`apps/api/docs/ci-cd.md`), com três
diferenças: filtro de path, contexto na raiz e nome de imagem com
sufixo `-api`.

```yaml
# .github/workflows/api-build.yml
name: API build
on:
  push:
    branches: [main]
    paths:
      - apps/api/**
      - packages/**
      - bun.lock
      - .bun-version
      - .github/workflows/api-*.yml
  workflow_dispatch:

permissions:
  contents: read
  packages: write

env:
  HUSKY: 0

jobs:
  image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - id: bun
        run: echo "version=$(cat .bun-version)" >> "$GITHUB_OUTPUT"
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v6
        with:
          images: ghcr.io/${{ github.repository }}-api
          tags: |
            type=sha,prefix=sha-,format=long
            type=raw,value=latest
      - uses: docker/build-push-action@v6
        with:
          context: .
          file: apps/api/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: BUN_VERSION=${{ steps.bun.outputs.version }}
          cache-from: type=gha,scope=api
          cache-to: type=gha,scope=api,mode=max
```

O filtro `paths` inclui `packages/**` porque o `contracts` vai empacotado
no bundle da API. Mudança só em `apps/web` não gera imagem.

## `api-deploy.yml`: migrate e deploy

Mesma estrutura do `deploy.yml` do api-bun (migrate com URL direta,
`concurrency` sem cancelamento, deploy depois), aplicada duas vezes:
**staging** automático e **production** com aprovação (*required
reviewers* no Environment `production` do GitHub).

```yaml
# .github/workflows/api-deploy.yml
name: API deploy
on:
  workflow_run:
    workflows: [API build]
    types: [completed]
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: api-deploy
  cancel-in-progress: false # nunca cancelar uma migration no meio

env:
  HUSKY: 0
  SHA: ${{ github.event.workflow_run.head_sha || github.sha }}

jobs:
  migrate-staging:
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
        with: { ref: '${{ env.SHA }}' }
      - uses: oven-sh/setup-bun@v2
        with: { bun-version-file: .bun-version }
      - run: bun install --frozen-lockfile
      - run: bun run db:migrate
        env:
          DATABASE_URL_UNPOOLED: ${{ secrets.DATABASE_URL_UNPOOLED }}

  deploy-staging:
    needs: migrate-staging
    runs-on: ubuntu-latest
    environment: staging
    steps:
      # Específico do host (docs/deploy.md, "### Host da API"):
      # apontar api-staging para ghcr.io/<owner>/<repo>-api:sha-<SHA>
      - run: echo "deploy staging sha-$SHA"

  migrate-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production # aprovação manual acontece aqui
    steps:
      - uses: actions/checkout@v4
        with: { ref: '${{ env.SHA }}' }
      - uses: oven-sh/setup-bun@v2
        with: { bun-version-file: .bun-version }
      - run: bun install --frozen-lockfile
      - run: bun run db:migrate
        env:
          DATABASE_URL_UNPOOLED: ${{ secrets.DATABASE_URL_UNPOOLED }}

  deploy-production:
    needs: migrate-production
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: echo "deploy production sha-$SHA"
```

- **`DATABASE_URL_UNPOOLED` por Environment**: em `staging` é a URL
  direta do branch Neon `staging`, em `production` a do `main`. Nenhum
  outro segredo da API entra nesses jobs (`lib/env-tooling.ts`).
- **`db:migrate` na raiz** passa pelo turbo, que só repassa as variáveis
  declaradas em `env` da task (`docs/architecture.md`, "## Pipeline do
  Turborepo"). `DATABASE_URL_UNPOOLED` já está lá.
- **Staging sempre antes de produção.** Uma migration que quebra em
  staging para o fluxo antes de tocar `main`.
- Rollback e ordem web × API: `docs/deploy.md`.

## Branch protection

Checks obrigatórios em `main`: **`lint`**, **`typecheck`**,
**`test-web`**, **`api-test`**, **`build`**, **`e2e`**, **`docker`** e
**`pr-title`**. Configuração de merge (só squash, título da PR como
mensagem, apagar branch) em `docs/git-workflow.md`.

## Secrets e variáveis

| Nome | Tipo | Escopo | Usado por |
|---|---|---|---|
| `NEON_API_KEY` | secret | repositório | `ci.yml`, `neon-cleanup.yml` |
| `NEON_PROJECT_ID` | variable | repositório | idem |
| `TURBO_TOKEN` / `TURBO_TEAM` | secret / variable | repositório | cache remoto (opcional) |
| `DATABASE_URL_UNPOOLED` | secret | Environment `staging` e `production` | `api-deploy.yml` (migrate) |
| tokens do host | secret | Environment `staging` e `production` | `api-deploy.yml` (deploy) |

Variáveis de runtime da API ficam no host. As do web ficam na Vercel.

## Atualização de dependências

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: bun
    directory: /
    schedule: { interval: weekly }
    groups:
      minor-and-patch:
        update-types: [minor, patch]
    commit-message: { prefix: 'chore(deps)' }
  - package-ecosystem: github-actions
    directory: /
    schedule: { interval: weekly }
    commit-message: { prefix: 'ci(deps)' }
```

- Um único ecossistema `bun` na raiz cobre todos os workspaces (um
  lockfile).
- Major de `next`, `react`, `elysia`, `drizzle-orm`, `better-auth`,
  `zod`, `turbo` ou `@biomejs/biome` nunca entra por merge automático. Se
  afetar algo documentado, é mudança de baseline (`docs/versioning.md`).
- Bun, Node e `neonctl` sobem por PR manual (`.bun-version` +
  `packageManager`, `.nvmrc`, versão no `ci.yml` e em
  `docs/development.md`).
