# Deploy

Web e API deployam **separados**, cada um no seu lugar:

| App | Onde | Gatilho | Ambientes |
|---|---|---|---|
| `apps/web` | Vercel (integração Git) | push em qualquer branch → preview; `main` → produção | Preview, Production |
| `apps/api` | Container Docker em host à escolha (DigitalOcean, Railway, Render…) | `api-build.yml` → `api-deploy.yml` (`docs/ci-cd.md`) | staging (automático), production (com aprovação) |

O banco é sempre Neon: `staging` e `main` (`docs/neon.md`).

## Web na Vercel

### Configuração do projeto (uma vez por instância)

- **Importar o repositório da instância** (nunca o do template).
- **Root Directory**: `apps/web`. Manter ligado *Include files outside
  the root directory in the Build Step*, que é o padrão e é necessário
  porque o web importa `packages/*`.
- **Framework preset**: Next.js.
- **Install Command**: `bun install --frozen-lockfile`. A Vercel detecta o
  `bun.lock` e usa o Bun. Declarar explicitamente evita surpresa se a
  detecção mudar.
- **Build Command**: `turbo run build --filter=@repo/web`. Assim o
  `^build` e o cache remoto (se configurado) valem também na Vercel.
- **Node.js Version**: a mesma major do `.nvmrc`.
- **Production Branch**: `main`.
- **Pular builds sem mudança**: habilitar *Skip deployments* para
  projetos não afetados (Settings → Build and Deployment → Root
  Directory). Onde não se aplicar, *Ignored Build Step* com
  `npx turbo-ignore`. Mudança só em `apps/api` não deve gerar deploy do
  web, mas mudança em `packages/contracts` deve.
- **Deployment Checks**: exigir o `ci.yml` verde antes de promover para
  produção (Settings → Deployment Checks), para que o merge com CI
  quebrado não publique.
- `transpilePackages: ['@repo/contracts']` no `next.config.ts`
  (`docs/architecture.md`, "### `packages/contracts`").

### Variáveis por ambiente

| Variável | Development (local) | Preview | Production |
|---|---|---|---|
| `API_URL` | `http://localhost:3333` | API de **staging** | API de produção |
| `NEXT_PUBLIC_APP_URL` | `http://localhost:3000` | `https://$VERCEL_BRANCH_URL` | domínio de produção |

O resto (rewrite, `NEXT_PUBLIC_*` fixada no build, `vercel env pull`)
segue `apps/web/docs/ci-cd.md`.

**Preview × `TRUSTED_ORIGINS` de staging**: a URL de preview muda por
branch e a API de staging valida `Origin`. Use o domínio de preview fixo
por branch (`<projeto>-git-<branch>-<time>.vercel.app`) ou um domínio
customizado de preview, e registre a escolha em `docs/domain.md`.
Nunca liberar `*` em `TRUSTED_ORIGINS`.

### Rollback do web

*Instant Rollback* no painel da Vercel. Depois, reverter o commit em
`main` por PR.

## API em container

### Dockerfile

A imagem é construída a partir da **raiz do repositório** (contexto
`.`), com o Dockerfile em `apps/api/Dockerfile`. O contexto precisa ser a
raiz porque o `bun.lock` e o `packages/contracts` estão lá.

```dockerfile
# apps/api/Dockerfile
# contexto de build: raiz do monorepo
# docker build -f apps/api/Dockerfile --build-arg BUN_VERSION=$(cat .bun-version) -t api .
# o default espelha .bun-version (muda junto, docs/checklists.md); o CI sempre passa o arg
ARG BUN_VERSION=1.3.11

FROM oven/bun:${BUN_VERSION} AS build
ENV HUSKY=0
WORKDIR /repo

# 1. manifests primeiro (cache de camada do install)
COPY package.json bun.lock ./
COPY apps/api/package.json apps/api/
COPY apps/web/package.json apps/web/
COPY packages/contracts/package.json packages/contracts/
COPY packages/tsconfig/package.json packages/tsconfig/
RUN bun install --frozen-lockfile --filter @repo/api

# 2. só o código que a API usa
COPY packages/ packages/
COPY apps/api/ apps/api/
RUN bun run --cwd apps/api build

FROM oven/bun:${BUN_VERSION}-slim AS runtime
ENV NODE_ENV=production
WORKDIR /app
RUN useradd --system --create-home appuser
COPY --from=build --chown=appuser /repo/apps/api/dist ./dist
USER appuser
EXPOSE 3333
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD ["bun", "-e", "fetch('http://localhost:'+(process.env.PORT||3333)+'/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
CMD ["bun", "dist/index.js"]
```

- **O `package.json` do web é copiado mesmo sem ser instalado.** O
  `bun install --frozen-lockfile` confere o lockfile contra **todos** os
  workspaces declarados, e faltar um manifesto faz o lockfile "mudar" e
  o build falhar. Workspace novo em `apps/*` ou `packages/*` exige uma
  linha nova aqui (`docs/checklists.md`).
- **`--filter @repo/api`** instala só as dependências da API e dos
  pacotes que ela usa. O Next e o resto do web não entram no estágio de
  build.
- O resto das decisões (bundle sem `node_modules`, `NODE_ENV` só no
  runtime, não-root, `HEALTHCHECK` em `/health`, `CMD` exec, segredos só
  em runtime, migrations fora da imagem) é o do api-bun e está em
  `apps/api/docs/docker.md`.
- **`ARG BUN_VERSION` tem default** igual ao `.bun-version`. Sem ele, o
  build emite `InvalidDefaultArgInFrom` e, sem o `--build-arg`, falha com
  "nome de imagem inválido". O CI e o comando documentado continuam
  passando o arg; o default só evita o erro críptico. Ao subir o Bun, o
  default muda na mesma PR (`docs/checklists.md`, "## Subir Bun, Node ou
  `neonctl`").
- Por que não `turbo prune`: `docs/architecture.md`, "### Imagem da API
  sem `turbo prune`".

```
# .dockerignore (raiz: o contexto é o repo inteiro)
**/node_modules
**/dist
**/.next
**/.turbo
.git
.github
.husky
.vscode
docs
**/docs
**/tests
**/e2e
**/*.md
**/.env*
apps/web/*
!apps/web/package.json
docker
```

`apps/web/*` com a exceção do `package.json` mantém o código do front
fora do contexto e deixa o manifesto que o install exige.

### Host da API

A baseline fixa **o que** o host precisa fazer, não **qual** host. O
padrão é **imagem pronta**: o CI publica `ghcr.io/<owner>/<repo>-api` com
a tag `sha-<commit>` e o `api-deploy.yml` manda o host rodar essa tag,
depois das migrations.

**Regra para qualquer host: desligar o auto-deploy por Git do host.** Se
o host builda e publica sozinho a cada push em `main`, a API nova pode
subir antes do job `migrate`, e a ordem migrate → deploy deixa de
existir (`apps/api/docs/ci-cd.md`, `deploy.yml`).

Requisitos que valem para todos:

- Health check HTTP em `/health` (liveness). Onde o host distingue
  readiness, usar `/ready`.
- Porta: a de `PORT` (3333 por padrão).
- Variáveis de runtime da API (`DATABASE_URL` **pooled** do branch Neon
  do ambiente, `REDIS_URL` `rediss://`, `BETTER_AUTH_SECRET`,
  `BETTER_AUTH_URL`, `TRUSTED_ORIGINS`…) cadastradas no host, por
  ambiente. `DATABASE_URL_UNPOOLED` **não** vai para o host: só o job de
  migration usa.
- Dois serviços/apps: `api-staging` e `api-production`.
- Região igual (ou próxima) à do projeto Neon e do Redis.

Notas por host (conferir na doc atual do host ao configurar; interfaces
mudam):

| Host | Como apontar para a imagem | Deploy pelo CI |
|---|---|---|
| **DigitalOcean App Platform** | Componente *Service* com fonte *Container image* (GHCR ou DOCR) | `doctl apps update <app-id> --spec` com a tag nova, ou `doctl apps create-deployment` |
| **Railway** | Serviço com fonte *Docker image* | Railway CLI (`railway`) com `RAILWAY_TOKEN` do projeto, ou redeploy pela API |
| **Render** | *Web Service* do tipo *Existing image* | *Deploy hook* com `?imgURL=` apontando para a tag nova |
| **VM com Docker** (Droplet etc.) | `docker compose` com a imagem | SSH + `docker compose pull && docker compose up -d` |

Credencial de registry: a imagem do GHCR é **privada** por padrão. O host
precisa de uma credencial com `read:packages` (PAT clássico ou fine-grained
de uma conta de serviço) cadastrada como *registry credential*, ou o pull
falha. Tornar o pacote público só se a instância decidir isso
explicitamente.

#### Referência: Render

Validado na primeira instância. Cada serviço (`api-staging`,
`api-production`) é um *Web Service* com fonte *Existing image* e a
credencial do GHCR. O *deploy hook* de cada serviço vira o secret
`RENDER_DEPLOY_HOOK_URL` do Environment correspondente:

```yaml
# .github/workflows/api-deploy.yml, job deploy-staging (deploy-production igual)
- name: Deploy no Render
  env:
    RENDER_DEPLOY_HOOK_URL: ${{ secrets.RENDER_DEPLOY_HOOK_URL }}
  run: |
    image="ghcr.io/${GITHUB_REPOSITORY,,}-api:sha-$SHA"
    curl --fail --silent --show-error --output /dev/null -X POST \
      --get --data-urlencode "imgURL=$image" "$RENDER_DEPLOY_HOOK_URL"
    echo "deploy de staging disparado: sha-$SHA"
```

- A URL do hook já traz `?key=<segredo>`: o `--get --data-urlencode`
  acrescenta `&imgURL=…` codificado. Nunca imprimir a URL (sem `-v`, sem
  `echo`).
- `${GITHUB_REPOSITORY,,}` põe o nome em minúsculas, como o GHCR exige.
- `--fail` faz o job falhar se o Render recusar o hook.

A escolha do host, os ids/tokens usados e o passo real do job `deploy`
vão para `docs/domain.md` da instância. Os tokens ficam como secrets dos
Environments `staging`/`production` do GitHub.

Se a instância preferir que o host **builde a partir do Git** (Railway e
Render suportam Dockerfile path + contexto de build), apontar Dockerfile
para `apps/api/Dockerfile` e contexto para a raiz, e ainda assim manter o
auto-deploy desligado, disparando o deploy pelo `api-deploy.yml` depois
do `migrate`.

### Rollback da API

Reapontar o serviço para a tag `sha-<commit>` anterior (a imagem
continua no GHCR). Migration não volta junto: por isso toda migration é
*expand/contract* e a versão anterior da API continua funcionando com o
schema novo (`apps/api/docs/ci-cd.md`).

## Ordem de deploy entre web e API

Merge em `main` dispara **dois caminhos independentes**:

```
merge em main ─┬─▶ Vercel: build do web ─▶ produção (minutos)
               └─▶ api-build ─▶ api-deploy: migrate+deploy staging ─▶ [aprovação] ─▶ migrate+deploy production
```

O web chega em produção **antes** da API, porque a API espera aprovação.
Então:

- **Uma PR pode alterar web e API juntos só se o web continuar
  funcionando com a API que já está em produção.** Exemplos: campo novo
  opcional que o web só exibe se vier; endpoint novo usado por tela
  escondida atrás de flag.
- **Senão, são duas PRs**: a primeira muda a API (e `contracts`, de forma
  compatível) e vai até produção. A segunda muda o web.
- **Remoção** segue *expand/contract*: o web deixa de usar primeiro, a
  API remove depois.
- `packages/contracts` com mudança **incompatível** (remover campo,
  estreitar enum) nunca entra junto com o código que depende dela do
  outro lado. O `typecheck` pega a quebra dentro do repo, mas não pega a
  diferença de versão entre o que está no ar em cada lado.

Resumo para a descrição da PR: se ela muda `apps/api` ou
`packages/contracts`, responder "o web em produção funciona com esta API?"
e "esta API funciona com o web em produção?". As duas respostas precisam
ser sim.
