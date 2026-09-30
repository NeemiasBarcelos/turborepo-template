# Docker

> Baseado em api-bun v0.16.0, adaptado ao monorepo. Onde este arquivo
> conflita com a raiz, vale a raiz (`../../CLAUDE.md`) e as seções
> "## Diferenças no monorepo" de `CLAUDE.md` e `docs/architecture.md`.

Dois usos distintos, não confundir:

- **Dockerfile** (`apps/api/Dockerfile`): só para *produção* (e staging).
  Desenvolvimento roda nativo via `bun run dev`, sem Docker no caminho.
- **`docker-compose.yml`** (na **raiz** do monorepo): só para *infra
  local* (Postgres + Redis). Não roda a aplicação.

## Dockerfile (produção)

O Dockerfile completo está em `../../docs/deploy.md`, "### Dockerfile".
Em relação ao api-bun, mudam três coisas:

- **Contexto de build é a raiz do monorepo**, porque o `bun.lock` e o
  `packages/contracts` estão lá:

  ```bash
  # a partir da raiz
  docker build -f apps/api/Dockerfile --build-arg BUN_VERSION=$(cat .bun-version) -t api .
  ```

- **Install filtrado**: copia os `package.json` de todos os workspaces
  (o `--frozen-lockfile` exige) e roda
  `bun install --frozen-lockfile --filter @repo/api`.
- **`.dockerignore` na raiz**, excluindo o código do web e mantendo só o
  `apps/web/package.json`.

As decisões do api-bun continuam todas valendo:

- **`bun build` empacota as dependências** (inclusive o
  `@repo/contracts`). O runtime só leva `dist/`, sem `node_modules`.
- **`ENV NODE_ENV=production` só no estágio de runtime.**
- **Sem migrations na imagem.** Migration roda no CD, com a URL direta do
  Neon (`docs/ci-cd.md`).
- **Versão do Bun fixa** (`.bun-version` na raiz, `ARG BUN_VERSION`).
- **`HUSKY=0` no build.**
- **`HEALTHCHECK` em `/health`** (liveness, sem tocar no banco),
  executado pelo próprio Bun (a imagem `slim` não tem `curl`).
- **`CMD` em forma exec**: o Bun é o PID 1 e recebe o `SIGTERM`
  (`docs/architecture.md`, "## Shutdown gracioso").
- **Usuário não-root** (`appuser`).
- **Segredos nunca entram na imagem.** Configuração vem do runtime do
  host.

## `docker-compose.yml` (infra local)

Movido para a raiz, com os bancos `app_dev` e `app_test`. Conteúdo e
script de init em `../../docs/development.md`, "## `docker-compose.yml`".
Continuam valendo:

- Portas publicadas só em `127.0.0.1`.
- `postgres:16`, a **mesma major** do projeto Neon.
- O banco de teste é criado por script em `docker/postgres-init/`, que
  só roda na **primeira** inicialização do volume. Se o volume já
  existia, criar à mão (`docker compose exec postgres psql -U postgres -c
  'CREATE DATABASE app_test;'`) ou recriar com `docker compose down -v`
  (apaga os dados de dev).
- Redis único. O isolamento do teste vem do índice (`/1` no
  `.env.test`), e o `flushdb()` do teste nunca atinge as chaves de dev
  (índice `0`).
