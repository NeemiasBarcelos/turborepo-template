# Changelog

Todas as mudanças notáveis na baseline deste template são registradas
aqui. Formato baseado em [Keep a Changelog](https://keepachangelog.com/),
versionamento conforme `docs/versioning.md`.

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
