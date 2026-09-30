# Desenvolvimento local

## Pré-requisitos

| Ferramenta | Versão | Por quê |
|---|---|---|
| Bun | a de `.bun-version` | package manager do monorepo, runtime da API |
| Node.js | a de `.nvmrc` (24 LTS) | o `next` roda em Node, como na Vercel |
| Docker | qualquer recente | Postgres 16 e Redis 7 locais |
| `neonctl` | a fixada abaixo | operar branches do Neon (`docs/neon.md`) |
| Vercel CLI | opcional | `vercel env pull`, `vercel link` |

Versão do `neonctl` usada pelo CI e pelos devs: **2.24.x**. Subir a
versão é PR que muda esta linha e o passo de instalação do `ci.yml`.

## Primeira vez

```bash
bun install                                  # instala todos os workspaces; ativa o Husky
docker compose up -d                         # Postgres 16 (app_dev, app_test) + Redis 7
cp apps/api/.env.example apps/api/.env.local
cp apps/web/.env.example apps/web/.env.local
bun run db:migrate                           # migrations no app_dev
bun run dev                                  # web :3000 + api :3333 (TUI do turbo)
```

`neonctl auth` e `neonctl link` (`docs/neon.md`, "## Setup do projeto")
só são necessários para quem vai operar branches do Neon. O dia a dia
local não precisa do Neon.

## `docker-compose.yml`

O mesmo do api-bun (`apps/api/docs/docker.md`), movido para a raiz e com
os bancos renomeados:

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16 # mesma major do projeto Neon
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_dev
    ports: ['127.0.0.1:5432:5432']
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./docker/postgres-init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres']

  redis:
    image: redis:7
    ports: ['127.0.0.1:6379:6379']
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']

volumes:
  pgdata:
```

```sql
-- docker/postgres-init/01-create-test-db.sql
CREATE DATABASE app_test;
```

- Portas só em `127.0.0.1`, como no api-bun.
- Nomes de banco: `app_dev` (dev), `app_test` (testes). São os mesmos
  nomes lógicos que o CI usa no branch Neon (`app`, `app_test`), então a
  guarda de teste da API vale igual nos dois lugares.
- O web não usa nenhum dos dois containers.

## Variáveis de ambiente locais

Cada app tem os próprios arquivos (`docs/architecture.md`,
"## Variáveis de ambiente"). Valores locais:

```bash
# apps/api/.env.local
DATABASE_URL=postgres://postgres:postgres@localhost:5432/app_dev
REDIS_URL=redis://localhost:6379/0
BETTER_AUTH_SECRET=<32+ caracteres: openssl rand -base64 32>
BETTER_AUTH_URL=http://localhost:3333
TRUSTED_ORIGINS=http://localhost:3000
```

```bash
# apps/api/.env.test (commitado)
NODE_ENV=test
DATABASE_URL=postgres://postgres:postgres@localhost:5432/app_test
REDIS_URL=redis://localhost:6379/1
BETTER_AUTH_SECRET=test-secret-not-a-real-secret-0123456789
BETTER_AUTH_URL=http://localhost:3333
```

```bash
# apps/web/.env.local
API_URL=http://localhost:3333
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

`vercel env pull apps/web/.env.local` (com o projeto já vinculado via
`vercel link --cwd apps/web`) traz as de Development da Vercel.

## Comandos do dia a dia

Tudo a partir da **raiz**:

```bash
bun run dev                              # web + api
bun run dev --filter=@repo/api           # só a API
bun run check                            # Biome com --write (repo inteiro)
bun run lint                             # Biome sem escrever (o que o CI roda)
bun run typecheck                        # tsc em todos os workspaces
bun run test                             # testes de todos os workspaces
turbo run test --filter=@repo/api        # só a API (bun test)
turbo run test:e2e --filter=@repo/web    # Playwright (precisa da API no ar)
bun run db:generate                      # gera migration a partir do schema Drizzle
bun run db:migrate                       # aplica migrations no app_dev
bun run db:studio                        # Drizzle Studio
bun add <pacote> --cwd apps/web          # dependência de um workspace
bun add -d <pacote>                      # devDependency da raiz (tooling)
```

- `turbo` fica nas devDependencies da raiz. `bunx turbo …` ou `bun run
  turbo …` funcionam sem instalar global.
- Nunca rodar `bun install` dentro de `apps/*`: sempre na raiz. Um
  `bun.lock` aparecendo em `apps/*` é erro e não entra no commit.
- Testes da API também rodam direto no workspace (`bun test` em
  `apps/api`), útil para `--watch` ou filtrar por arquivo.

## E2E local

O Playwright do web sobe o `next start` e precisa de uma API real com
banco de teste:

```bash
docker compose up -d
(cd apps/api && NODE_ENV=test bun run db:migrate)   # migrations no app_test (.env.test)
turbo run test:e2e --filter=@repo/web
```

A configuração do Playwright sobe a API em modo teste (`bun --env-file
.env.test src/index.ts` em `apps/api`) como segundo `webServer`. Detalhes
em `apps/web/docs/testing.md`, "## Diferenças no monorepo".

## Problemas comuns

- **`next` rodando no Bun** (erros estranhos de runtime no web): existe
  um `bunfig.toml` com `[run] bun = true` fora de `apps/api`. Remover.
- **Tipos do `contracts` não atualizam no web**: reiniciar o TS server do
  editor. O pacote é JIT e não tem build para esquecer.
- **Cache do turbo com env velho**: variável nova não declarada em `env`
  no `turbo.json` (`docs/architecture.md`, "## Pipeline do Turborepo").
  `turbo run build --force` confirma o diagnóstico.
- **Guarda de teste abortando**: o `DATABASE_URL` do shell sobrepõe o
  `.env.test`. Rodar `env | grep DATABASE_URL` e limpar.
