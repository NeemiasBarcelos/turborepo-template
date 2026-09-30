# CI/CD

> Baseado em api-bun v0.16.0, adaptado ao monorepo. Onde este arquivo
> conflita com a raiz, vale a raiz (`../../CLAUDE.md`) e as seções
> "## Diferenças no monorepo" de `CLAUDE.md` e `docs/architecture.md`.

No monorepo, os workflows são do repositório inteiro e ficam
documentados na raiz:

| api-bun (repo separado) | turborepo-template | Onde |
|---|---|---|
| `ci.yml`, job `test` com service container de Postgres | `ci.yml`, job `api-test` em branch Neon `ci/pr-<n>` | `../../docs/ci-cd.md` |
| `ci.yml`, job `docker` | `ci.yml`, job `docker` (contexto na raiz, `-f apps/api/Dockerfile`) | idem |
| `build.yml` | `api-build.yml` (filtro de path, imagem `<repo>-api`) | idem |
| `deploy.yml` (migrate + deploy em produção) | `api-deploy.yml` (staging automático, depois produção com aprovação) | idem |
| Dependabot `bun` + `github-actions` | igual, na raiz | idem |

Este arquivo guarda só as regras da API que continuam valendo como
estão.

## O que a baseline da API fixa (inalterado)

- **Ordem**: build da imagem → migrate → deploy. Agora a ordem se repete
  por ambiente: staging, depois produção.
- **Migration só no CD, nunca no container.** O job de migrate usa a URL
  **direta** do Neon (`DATABASE_URL_UNPOOLED`, `docs/architecture.md`,
  "### Neon") e recebe **só esse secret**, do Environment do GitHub
  (`staging` ou `production`). O tooling lê a env por
  `lib/env-tooling.ts`.
- **`concurrency` sem cancelamento** no deploy: uma migration por vez,
  nunca interrompida.
- **Migration antes da imagem nova ⇒ compatível com a versão antiga**
  (*expand/contract*). No monorepo isso vale também para o web, que
  publica antes da API (`../../docs/deploy.md`, "## Ordem de deploy entre
  web e API").
- **Migrations só andam para frente.** Rollback de schema é migration
  nova ou restore do Neon (`../../docs/neon.md`, "### Restore de
  produção"), nunca editar ou apagar migration aplicada.
- **Segredos de runtime** (`DATABASE_URL` pooled, `REDIS_URL`,
  `BETTER_AUTH_*`…) vivem no host da API, por ambiente, não no GitHub.
- **Imagem com tag imutável** `sha-<commit>`, que é a que o deploy usa.
  Nome minúsculo via `docker/metadata-action`.
- **Bun fixo** por `.bun-version` (agora na raiz do monorepo), o mesmo
  no CI e no `BUN_VERSION` do Docker.

## Env do job de teste

O `api-test` exporta, além das URLs do branch Neon (via `neon:urls
--github-env`), as variáveis que `lib/env.ts` exige, com valores de
teste:

```yaml
env:
  NODE_ENV: test
  REDIS_URL: redis://localhost:6379/1
  BETTER_AUTH_SECRET: ci-test-secret-not-a-real-secret-0123456789
  BETTER_AUTH_URL: http://localhost:3333
```

O banco é `app_test` e o Redis usa o índice `1`. É a mesma convenção do
`.env.test`, então a guarda de `tests/setup.ts` passa no CI pelo mesmo
motivo que passa localmente (`docs/testing.md`).
