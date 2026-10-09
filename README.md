# turborepo-template

Template base para produtos em **monorepo Turborepo** com:

- **`apps/web`**: frontend Next.js 16 na **Vercel**, derivado do
  template app-nextjs.
- **`apps/api`**: API Bun/Elysia multi-tenant em **Docker**, num host
  como DigitalOcean, Railway ou Render, derivada do template
  [api-bun](https://github.com/NeemiasBarcelos/api-bun).
- **Neon** como Postgres hospedado, com um branch por PR no CI operado
  pelo `neonctl`.
- **`packages/contracts`** como contrato único entre web e API.

Bun workspaces, Biome v2, GitHub Actions. Cada produto real é derivado
deste template e herda a stack e as regras abaixo.

## Status atual

Este repositório contém a **baseline documentada** do monorepo (regras,
arquitetura, Neon, desenvolvimento, deploy, CI/CD, convenções, git
workflow, versionamento, fluxo de tasks do Claude Code), versão atual
**0.2.0** (`docs/CHANGELOG.md`),
embutindo app-nextjs `0.1.0` e api-bun `0.16.0`.

O scaffold de código **ainda não foi gerado**: não existem
`package.json`, `turbo.json`, `apps/*/src` nem workflows. Os comandos
abaixo descrevem o estado depois do scaffold.

## Pré-requisitos

- Bun (versão de `.bun-version`, depois do scaffold).
- Node.js 24 LTS (o Next roda em Node, como na Vercel).
- Docker (Postgres 16 e Redis locais).
- `neonctl` (`bun add -g neonctl` ou `brew install neonctl`) e uma conta
  no [Neon](https://neon.com).
- Para deploy: um projeto na [Vercel](https://vercel.com), um host de
  containers para a API e um Redis gerenciado com TLS.

## Como começar

**Agora, com o repositório neste estado** (só documentação):

1. Ler `CLAUDE.md`, que tem as regras não-negociáveis.
2. Percorrer a documentação pelo mapa abaixo, na ordem sugerida.

**Depois que o scaffold existir**:

```bash
bun install
docker compose up -d
cp apps/api/.env.example apps/api/.env.local
cp apps/web/.env.example apps/web/.env.local
bun run db:migrate
bun run dev                  # web http://localhost:3000, api http://localhost:3333
```

Para operar o Neon (setup da instância, staging, branches):

```bash
neonctl auth
neonctl link --project-id <project-id> --no-env-pull
```

Ver `docs/neon.md`, "## Setup do projeto".

## Prompt inicial para o Claude

Depois de clonar este repositório para começar um produto novo (ver
`docs/versioning.md` para o que "derivar do template" significa), cole o
prompt abaixo no Claude, dentro do novo repositório, para gerar o
scaffold técnico real a partir da baseline documentada:

```
Leia CLAUDE.md e todos os arquivos em docs/ deste repositório, e depois apps/web/CLAUDE.md, apps/web/AGENTS.md, apps/web/docs/, apps/api/CLAUDE.md e apps/api/docs/, antes de fazer qualquer coisa. Eles definem a stack, a arquitetura e as regras não-negociáveis. Onde a doc de um app conflitar com a raiz, a raiz vence, e as seções "Diferenças no monorepo" dizem como. Para qualquer API do Next.js, consulte node_modules/next/dist/docs/ da versão instalada. Para as outras libs (Turborepo, Bun, Elysia, Drizzle, Better Auth, neonctl), consulte a documentação atual (Context7 ou --help) em vez de confiar na memória.

Antes de gerar qualquer arquivo, me pergunte:
(1) o nome do produto e se os workspaces ficam com o escopo @repo/ ou outro;
(2) quais métodos de login estarão habilitados (email+senha, OAuth e quais provedores, magic link…);
(3) como é o fluxo de organização: quem pode criar organização, como o convite é aceito, o que o usuário vê sem nenhuma organização;
(4) idioma(s) da interface e se haverá tema claro/escuro;
(5) qual host vai rodar a API (DigitalOcean, Railway, Render, outro) e qual provedor de Redis;
(6) se o projeto Neon já existe (id e região) e se a produção terá dado pessoal que exija branch de CI --schema-only.
São decisões de produto e de infra: não presuma nem deixe de fora sem perguntar.

Só depois de eu responder, gere o scaffold técnico inicial seguindo exatamente o que está documentado:
- Raiz: package.json (workspaces, packageManager, scripts), turbo.json, biome.json, .bun-version, .nvmrc, .gitignore, .dockerignore, docker-compose.yml, docker/postgres-init/, commitlint.config.mjs, .husky/, .vscode/ (docs/architecture.md, docs/conventions.md, docs/development.md)
- packages/tsconfig (base.json, nextjs.json, bun.json) e packages/contracts (errors, permissions, roles, modules), com typecheck e teste de can()
- apps/api: o scaffold do api-bun (apps/api/docs/*), adaptado por "Diferenças no monorepo": tsconfig estendendo @repo/tsconfig, enums do @repo/contracts, Dockerfile com contexto na raiz (docs/deploy.md), .env.example e .env.test com app_dev/app_test
- apps/web: o scaffold do app-nextjs (apps/web/docs/*), adaptado por "Diferenças no monorepo": sem pnpm, transpilePackages com @repo/contracts, permissões e erro vindos do @repo/contracts, Playwright subindo a API do monorepo
- scripts/neon/: branch.ts, urls.ts, test-db.ts, diff.ts, delete.ts (docs/neon.md)
- .github/workflows/: ci.yml, neon-cleanup.yml, api-build.yml, api-deploy.yml, e .github/dependabot.yml (docs/ci-cd.md), com o passo de deploy do host escolhido

Depois de gerar os arquivos acima:
- Rode bun install, docker compose up -d, bun run db:migrate, bun run lint, bun run typecheck, bun run test e turbo run build, e corrija até passarem. Rode também o docker build da API a partir da raiz.
- Atualize README.md e CLAUDE.md para refletirem esta instância, não o template: troque a seção "Status atual" do README pelo estado real e adicione no topo do CLAUDE.md a linha "> Baseado no template turborepo-template vX.Y.Z" com a versão documentada agora, conforme docs/versioning.md. Registre as respostas às perguntas acima em docs/domain.md.
- Antes de commitar qualquer coisa, rode git remote -v. Se o remote origin ainda aponta para o repositório do template (turborepo-template) de onde você clonou, PARE e me pergunte: criar um repositório novo no GitHub para esta instância (e trocar o origin), ou adicionar um remoto novo. Nunca commite nem dê push para o repositório de origem do template (docs/git-workflow.md).
- Só depois disso, siga docs/git-workflow.md para commitar o scaffold: crie uma branch, rode bun run lint e bun run typecheck, commit em Conventional Commits, e abra PR para main. Nunca deixe os arquivos gerados como mudança não commitada direto em main.
- Me passe a lista do que eu preciso configurar fora do repositório (Neon, secrets e environments do GitHub, Vercel, host da API, Redis), seguindo "## Setup de uma instância nova" de docs/checklists.md.

Não invente domínio de negócio: docs/domain.md está vazio de propósito. Não crie módulos de feature reais ainda, só a base técnica (auth, tenant, erro, health, layout, providers, contrato base). Se alguma decisão não estiver clara na documentação, pare e me pergunte em vez de assumir.
```

Depois que o scaffold rodar, vale um segundo prompt pedindo **uma feature
de exemplo ponta a ponta**: contrato em `packages/contracts`, módulo na
API com migration e teste de isolamento, listagem com filtro na URL e
formulário de criação no web. É o teste de que a documentação sozinha
basta para gerar código consistente nos três workspaces, e de que o CI
cria o branch Neon, comenta o schema diff e roda os testes nele.

## Mapa da documentação

Ordem sugerida de leitura:

| Doc | Cobre |
|---|---|
| `CLAUDE.md` | Regras não-negociáveis, stack e comandos do monorepo, em resumo denso escrito para guiar um agente de IA |
| `docs/architecture.md` | Layout, Bun workspaces, pipeline do turbo, `packages/contracts` e `tsconfig`, comunicação web → api, ambientes, decisões registradas |
| `docs/neon.md` | Modelo de branches, setup com `neonctl`, wrappers `scripts/neon/`, branch por PR no CI, staging, restore |
| `docs/development.md` | Pré-requisitos, Docker local, env por app, comandos do dia a dia, E2E local |
| `docs/deploy.md` | Vercel (Root Directory, envs), Dockerfile da API a partir da raiz, hosts, ordem de deploy web × API, rollback |
| `docs/ci-cd.md` | `ci.yml`, `neon-cleanup.yml`, `api-build.yml`, `api-deploy.yml`, secrets, branch protection, Dependabot |
| `docs/conventions.md` | Nomes de workspace, imports entre pacotes, Biome, TypeScript, scripts, VSCode |
| `docs/git-workflow.md` | Branch, commits com escopo de módulo/workspace, PR, merge, Husky |
| `docs/versioning.md` | Versão do template, baselines embutidas e sincronização com app-nextjs/api-bun |
| `docs/checklists.md` | Feature ponta a ponta, PR, setup de instância, sincronização, bump |
| `tasks/how-to-use.md` + `.claude/` | Fluxo de tasks do Claude Code: skills `new-branch`/`update-main` e comandos `/new-task` a `/close-task` |
| `docs/domain.md` | Domínio e infra: vazio no template, preenchido por cada instância |
| `docs/features/` | Docs de features complexas: vazio no template |
| `apps/web/CLAUDE.md` + `apps/web/docs/` | Baseline app-nextjs com as diferenças no monorepo |
| `apps/api/CLAUDE.md` + `apps/api/docs/` | Baseline api-bun com as diferenças no monorepo |
| `SECURITY.md` | Como reportar vulnerabilidade no template |

## Versionamento

Este é um **template versionado**, não um app pronto: a baseline segue
SemVer próprio (`docs/versioning.md`), com histórico em
`docs/CHANGELOG.md`. Uma instância derivada herda a baseline na versão
vigente quando foi criada.

## Licença

MIT. Ver `LICENSE`.
