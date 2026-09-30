# Checklists

Os checklists dos templates de origem foram fundidos aqui: no monorepo,
os apps não têm `checklists.md` próprio. Os passos 2 e 3 de "## Feature
ponta a ponta" resumem os checklists de feature do api-bun e do
app-nextjs.

## Feature ponta a ponta

1. **Contrato** (`packages/contracts/src/<modulo>.ts`)
   - [ ] Schemas Zod de request e response do módulo, com o mesmo nome do
         módulo na API e no web.
   - [ ] Enum novo que aparece na borda (status, tipo) exportado como
         array `as const` e consumido pelo `pgEnum` da API.
   - [ ] Nenhum import de `drizzle-orm`, `elysia`, `better-auth` server
         ou `@/` no pacote.
   - [ ] Mudança incompatível (remover/renomear campo, estreitar enum)?
         Planejar em duas PRs (`docs/deploy.md`, "## Ordem de deploy
         entre web e API").
2. **API** (`apps/api/src/modules/<modulo>/`)
   - [ ] Schema Drizzle com `organization_id` + `tenantColumns`, acesso
         só por `inTenant` (`apps/api/docs/architecture.md`,
         "### Multi-tenancy").
   - [ ] Migration gerada (`bun run db:generate`), revisada e
         *expand/contract*. Nunca `push`.
   - [ ] `inputSchema`/`outputSchema` da feature baseados nos schemas do
         `contracts`.
   - [ ] Permissão por `resolveRole` + `can` (do `contracts`).
   - [ ] Testes, incluindo isolamento entre tenants
         (`apps/api/docs/testing.md`).
3. **Web** (`apps/web/src/features/<modulo>/`)
   - [ ] Resposta validada com o schema do `contracts` na fronteira.
   - [ ] Query keys começando por `organizationId`, estado no lugar certo
         (`apps/web/docs/architecture.md`, "## Estado").
   - [ ] Ações escondidas por `can` do `contracts`, 403 tratado mesmo
         assim.
   - [ ] Testes obrigatórios de `apps/web/docs/testing.md`, com mocks MSW
         no formato real da API.
4. **Documentação**
   - [ ] Feature complexa? `docs/features/<nome>.md` (um doc só para os
         dois lados).
   - [ ] Termo novo de negócio no glossário de `docs/domain.md`.

## Pull Request

- [ ] Título em Conventional Commits, escopo pelo módulo ou workspace
      (`docs/git-workflow.md`).
- [ ] `bun run lint`, `bun run typecheck` e `bun run test` passando
      localmente.
- [ ] CI verde: `lint`, `typecheck`, `test-web`, `api-test`, `build`,
      `e2e`, `docker`, `pr-title`.
- [ ] Mudou `apps/api` ou `packages/contracts`? As duas perguntas de
      compatibilidade respondidas na descrição.
- [ ] Tem migration? Schema diff do Neon conferido no comentário da PR.
- [ ] Navegado no preview da Vercel (quando muda o web), inclusive em
      largura de celular.
- [ ] Sem `console.log`, sem `any`, sem `process.env` fora dos módulos de
      env e de `scripts/`.
- [ ] Variável de ambiente nova:
  - [ ] no schema de env do app;
  - [ ] no `.env.example`;
  - [ ] em `env` da task certa no `turbo.json`;
  - [ ] na Vercel (3 ambientes) ou no host da API (staging e
        production).
- [ ] Workspace novo? Manifesto copiado no `apps/api/Dockerfile` e
      entrada nas tabelas de `docs/architecture.md`.
- [ ] Dependência nova: checado que não existe equivalente na stack e
      que a versão bate com a dos outros workspaces.
- [ ] Mudou "Regras não-negociáveis" ou decisão estrutural? Então é bump
      de baseline (abaixo), não PR comum.

## Setup de uma instância nova

- [ ] `git remote -v` resolvido (`docs/git-workflow.md`).
- [ ] Projeto Neon criado (Postgres 16), banco `app`, branch `staging`,
      `neonctl link --no-env-pull` (`docs/neon.md`).
- [ ] Secrets e variables do GitHub cadastrados (`docs/ci-cd.md`,
      "## Secrets e variáveis").
- [ ] Environments `staging` e `production` no GitHub, `production` com
      required reviewers.
- [ ] Projeto na Vercel com Root Directory `apps/web` e envs de Preview
      apontando para a API de staging (`docs/deploy.md`).
- [ ] Serviços `api-staging` e `api-production` no host, com auto-deploy
      por Git **desligado** e health check em `/health`.
- [ ] Redis gerenciado (`rediss://`) para staging e produção.
- [ ] Domínio de preview fixo em `TRUSTED_ORIGINS` da API de staging.
- [ ] Branch protection de `main` configurada.
- [ ] Decisões de infra (host, região, plano do Neon, domínio) em
      `docs/domain.md`.

## Sincronizar com app-nextjs ou api-bun

- [ ] CHANGELOG do template de origem lido desde a versão embutida.
- [ ] Arquivos alterados copiados para `apps/<app>/`, com as seções
      "## Diferenças no monorepo" preservadas.
- [ ] Conflitos com regras do monorepo resolvidos a favor do monorepo e
      registrados em "Diferenças no monorepo".
- [ ] Cabeçalhos `> Baseado em <template> vX.Y.Z` atualizados em todos os
      arquivos copiados.
- [ ] `docs/CHANGELOG.md` com as versões embutidas novas e bump deste
      template.

## Bump de versão da baseline

- [ ] Decisão registrada em "## Decisões registradas" do doc certo
      (Decisão / Contexto / Alternativas).
- [ ] Nível do bump decidido por `docs/versioning.md` (lembrar da
      convenção pré-1.0).
- [ ] Entrada nova em `docs/CHANGELOG.md` com data, seções (Adicionado /
      Alterado / Removido / Corrigido / Contexto), aviso de breaking
      quando for o caso e as versões embutidas de app-nextjs e api-bun.
- [ ] Versão atualizada em **todos** os lugares:
  - [ ] `CLAUDE.md`, seção "Versão da baseline"
  - [ ] `CLAUDE.md`, rodapé
  - [ ] `README.md`, seção "Status atual"
- [ ] `grep -rn "0\.[0-9]*\.[0-9]*" CLAUDE.md README.md` sem versão antiga
      sobrando.
- [ ] Commit `docs(<escopo>): …` (com `!` se breaking) via PR.

## Subir Bun, Node ou `neonctl`

- [ ] Bun: `.bun-version` **e** `packageManager` do `package.json` da
      raiz, com o mesmo valor.
- [ ] Node: `.nvmrc` e a versão configurada na Vercel.
- [ ] `neonctl`: versão no `ci.yml`, no `neon-cleanup.yml` e em
      `docs/development.md`. Comandos de `docs/neon.md` conferidos com
      `--help`.
- [ ] CI verde, incluindo `docker`.
