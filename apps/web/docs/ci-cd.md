# CI/CD

> Baseado em app-nextjs v0.1.0, adaptado ao monorepo. Onde este arquivo
> conflita com a raiz, vale a raiz (`../../CLAUDE.md`) e as seções
> "## Diferenças no monorepo" de `CLAUDE.md` e `docs/architecture.md`.

No monorepo, o `ci.yml` é um só para o repositório inteiro e a
configuração da Vercel muda (Root Directory, install com Bun, build via
turbo). As duas coisas estão documentadas na raiz:

- Workflows, jobs do web (`test-web`, `build`, `e2e`), branch protection
  e Dependabot → `../../docs/ci-cd.md`.
- Projeto na Vercel, variáveis por ambiente, rollback e ordem de deploy
  com a API → `../../docs/deploy.md`.

Este arquivo guarda só o que é específico do web e continua valendo.

## Build no CI precisa de env válido

O `next build` valida o env com Zod (`docs/architecture.md`, "## Variáveis
de ambiente"). O job `build` exporta valores fictícios válidos
(`API_URL=http://localhost:3333`, `NEXT_PUBLIC_APP_URL=http://localhost:3000`).
Toda variável nova do web entra:

- no schema;
- no `.env.example`;
- em `env` da task `build` no `turbo.json`;
- nos três ambientes da Vercel;
- no `env:` do job `build`, se o schema exigir.

## `NEXT_PUBLIC_*` é fixada no build

Mudou na Vercel, precisa de redeploy. O turbo também inclui essas
variáveis no hash da task `build` (`NEXT_PUBLIC_*` em `env`), então um
valor novo nunca reaproveita um build em cache com o valor velho.

## Preview × API de staging

Todo preview do web usa a API de staging, que roda a última versão de
`main` contra o branch Neon `staging`. Consequência: **um preview de PR
não enxerga mudança de API da mesma PR**. A mudança de API é validada
pelos testes do CI (branch Neon `ci/pr-<n>`) e pelo E2E, que sobe a API
da própria PR. Tela que depende de endpoint novo só é navegável no
preview depois que a API chega em staging (`../../docs/deploy.md`,
"## Ordem de deploy entre web e API").

## Major de dependência do web

Major de `next`, `react`, `tailwindcss`, `zod` ou `@tanstack/react-query`
nunca entra por merge automático do Dependabot. Ler o guia de upgrade e,
se afetar algo documentado aqui, tratar como mudança de baseline
(`../../docs/versioning.md`).
