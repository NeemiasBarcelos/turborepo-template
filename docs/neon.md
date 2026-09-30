# Neon

Todo projeto derivado deste template usa **Neon** como Postgres
hospedado: produção, staging e CI são branches de **um único projeto
Neon**. A CLI oficial (`neonctl`) é a forma de operar esses branches, à
mão ou no CI, sempre pelos wrappers de `scripts/neon/`.

As regras de conexão do api-bun continuam valendo e não se repetem aqui
(`apps/api/docs/architecture.md`, "### Neon"):

- `DATABASE_URL` pooled (host com `-pooler`) no runtime, com
  `prepare: false`.
- `DATABASE_URL_UNPOOLED` direta para migrations e `drizzle-kit`.
- `sslmode=require` e **sem** `channel_binding`.

Este documento cobre o que o monorepo acrescenta: modelo de branches,
setup com a CLI, branch por PR no CI, schema diff e operação.

## Modelo de branches

```
main  (default, produção)
 ├── staging          longo; API de staging e previews da Vercel
 ├── ci/pr-123        efêmero; testes e migrations da PR #123
 └── ci/pr-124        efêmero
```

| Branch | Criado por | Vida | Dados | Quem usa |
|---|---|---|---|---|
| `main` | criação do projeto | permanente | produção | API de produção |
| `staging` | setup, uma vez | permanente, resetado sob demanda | cópia de `main` no momento do reset | API de staging |
| `ci/pr-<n>` | CI, na primeira execução da PR | até fechar a PR, com teto de 7 dias (`--expires-at`) | cópia de `main` na criação | job `api-test` do CI |

Regras:

- **Nunca `drizzle-kit push` em branch Neon**, nem em `ci/pr-<n>`. Todo
  branch hospedado recebe schema só por `db:migrate` (regra do api-bun,
  estendida a todo branch).
- **Todo branch efêmero nasce com `--expires-at`.** A limpeza no
  `pull_request: closed` (`docs/ci-cd.md`) é o caminho normal. A
  expiração é a rede de segurança para PR abandonada ou job que falhou
  antes de apagar.
- **Nome de branch segue o padrão `<tipo>/<id>`** (`ci/pr-123`). O CI
  opera branches por nome, e os wrappers recusam apagar qualquer branch
  fora de `ci/`.
- **`main` e `staging` são protegidos** no console do Neon (*Protected
  branches*, se o plano permitir). Sem isso, pelo menos nunca rodar
  `neon:delete` ou `branches reset` contra eles fora do runbook
  ("## Operação").

## Setup do projeto (uma vez por instância)

1. **Instalar e autenticar a CLI** (local):

   ```bash
   bun add -g neonctl      # ou: brew install neonctl
   neonctl auth            # abre o browser; grava em ~/.config/neonctl
   ```

2. **Criar o projeto** no console do Neon, escolhendo **Postgres 16**, a
   mesma major do `docker-compose.yml` e do resto do template. O
   `neonctl projects create` não expõe a versão do Postgres, então a
   criação é pelo console (ou pela API do Neon). Região: a mesma (ou a
   mais próxima) do host da API.

3. **Renomear o banco padrão** para `app` (ou criar `app` e remover
   `neondb`). O nome precisa ser o mesmo em todos os branches, porque o
   CI e os wrappers usam `--database-name app`.

4. **Vincular o repositório** ao projeto, sem puxar env:

   ```bash
   neonctl link --project-id <project-id> --no-env-pull
   ```

   O `link` grava `.neon` na raiz (org, projeto, branch). O arquivo está
   no `.gitignore`: é contexto local de cada máquina. `--no-env-pull`
   porque o `link` escreveria `DATABASE_URL` num `.env` da raiz, que no
   monorepo não é lido por nenhum app (`docs/architecture.md`,
   "## Variáveis de ambiente"). `set-context` está deprecado: usar
   `link`/`checkout`.

5. **Criar o branch `staging`**:

   ```bash
   neonctl branches create --name staging --parent main
   ```

6. **Criar uma API key** (console → Account/Organization settings → API
   keys; de organização quando o projeto é de uma org) e cadastrar no
   GitHub:

   | Onde | Nome | Valor |
   |---|---|---|
   | Repository secret | `NEON_API_KEY` | API key do Neon |
   | Repository variable | `NEON_PROJECT_ID` | id do projeto (`neonctl projects list`) |
   | Environment `staging`, secret | `DATABASE_URL_UNPOOLED` | `bun run neon:urls staging` (linha `UNPOOLED`) |
   | Environment `production`, secret | `DATABASE_URL_UNPOOLED` | `bun run neon:urls main` (linha `UNPOOLED`) |

   A URL **pooled** de `staging` e de `main` vai para as variáveis de
   runtime do host da API (`docs/deploy.md`), não para o GitHub.

7. **Conferir** uma URL com o script avulso do api-bun
   (`apps/api/docs/architecture.md`, "### Neon"), contra `staging`.

## Wrappers em `scripts/neon/`

O CI e os desenvolvedores não chamam `neonctl` solto para operar
branches. Os wrappers garantem:

- idempotência (rodar duas vezes não falha);
- `--expires-at` em todo branch efêmero;
- remoção de `channel_binding` das URLs;
- recusa em apagar branch fora de `ci/`.

Autenticação: `NEON_API_KEY` no ambiente (CI) ou a credencial do
`neonctl auth` (local). Projeto: `NEON_PROJECT_ID` no ambiente ou o
`.neon` do `link`.

| Script | Uso | Faz |
|---|---|---|
| `neon:branch <nome>` | `bun run neon:branch ci/pr-123` | Cria o branch a partir de `main` com `--expires-at` de 7 dias, se ainda não existir. Se existir, só renova a expiração. |
| `neon:urls <branch> [db] [--github-env]` | `bun run neon:urls ci/pr-123 app_test` | Imprime `DATABASE_URL=` (pooled) e `DATABASE_URL_UNPOOLED=` (direta) do banco `db` (padrão `app`), sem `channel_binding`. Com `--github-env` (CI), mascara os valores e grava no `$GITHUB_ENV` em vez de imprimir. |
| `neon:test-db <branch> [db]` | `bun run neon:test-db ci/pr-123` | Apaga e recria o banco `db` (padrão `app_test`) no branch. Recusa nome sem sufixo `_test`. |
| `neon:diff <branch>` | `bun run neon:diff ci/pr-123` | `neonctl branches schema-diff main <branch> --database app` e escreve o resultado em Markdown (para o comentário da PR). |
| `neon:delete <branch>` | `bun run neon:delete ci/pr-123` | Apaga o branch se existir. Recusa qualquer nome fora de `ci/`. |

Exemplo do formato (o scaffold escreve os cinco no mesmo padrão):

```ts
// scripts/neon/branch.ts
import { $ } from 'bun'

const name = Bun.argv[2]
if (!name?.startsWith('ci/')) {
	console.error('uso: bun run neon:branch ci/<id>')
	process.exit(1)
}

const project = process.env.NEON_PROJECT_ID ? ['--project-id', process.env.NEON_PROJECT_ID] : []
const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString()

const exists = await $`neonctl branches get ${name} ${project} -o json`.nothrow().quiet()

if (exists.exitCode === 0) {
	await $`neonctl branches set-expiration ${name} --expires-at ${expiresAt} ${project}`.quiet()
} else {
	await $`neonctl branches create --name ${name} --parent main --expires-at ${expiresAt} ${project} -o json`.quiet()
}
```

```ts
// scripts/neon/urls.ts
import { appendFile } from 'node:fs/promises'
import { $ } from 'bun'

const args = Bun.argv.slice(2)
const githubEnv = args.includes('--github-env')
const [branch, database = 'app'] = args.filter((a) => !a.startsWith('--'))
if (!branch) {
	console.error('uso: bun run neon:urls <branch> [database] [--github-env]')
	process.exit(1)
}

const project = process.env.NEON_PROJECT_ID ? ['--project-id', process.env.NEON_PROJECT_ID] : []

// o neonctl sempre acrescenta channel_binding=require; o postgres.js não o implementa
// (apps/api/docs/architecture.md, "### Neon")
const clean = (raw: string) => {
	const url = new URL(raw.trim())
	url.searchParams.delete('channel_binding')
	url.searchParams.set('sslmode', 'require')
	return url.toString()
}

const pooled = clean(await $`neonctl cs ${branch} --database-name ${database} --pooled ${project}`.text())
const direct = clean(await $`neonctl cs ${branch} --database-name ${database} ${project}`.text())
const lines = `DATABASE_URL=${pooled}\nDATABASE_URL_UNPOOLED=${direct}\n`

if (githubEnv && process.env.GITHUB_ENV) {
	// máscara antes de qualquer outro uso: o valor nunca aparece no log do job
	console.log(`::add-mask::${pooled}`)
	console.log(`::add-mask::${direct}`)
	await appendFile(process.env.GITHUB_ENV, lines)
} else {
	process.stdout.write(lines)
}
```

- Os scripts são **tooling**, então podem usar `process.env` e `console`
  direto. As regras de env e de logger valem para o código dos apps.
- Os comandos acima foram conferidos contra o `neonctl` 2.24
  (`branches create --expires-at`, `branches set-expiration`,
  `branches schema-diff [base] [compare] --database`,
  `databases create --branch --name`, `cs --pooled --database-name`). Ao
  subir a CLI, conferir com `neonctl <comando> --help`.
- O CI instala a CLI com versão fixa (`bun add -g neonctl@<versão>`), a
  mesma que os devs usam, registrada em `docs/development.md`.

## Branch por PR no CI

O job `api-test` do `ci.yml` (`docs/ci-cd.md`) faz, a cada push na PR:

1. `bun run neon:branch ci/pr-<n>`: cria o branch a partir de `main` (1ª
   execução) ou renova a expiração.
2. `bun run db:migrate` contra o banco **`app`** do branch, que é uma
   cópia de produção. Uma migration que falha com dado real (ex:
   `NOT NULL` sem default numa tabela populada) quebra aqui, antes do
   merge.
3. `bun run neon:diff ci/pr-<n>`: schema diff entre `main` e o branch,
   postado como comentário na PR. Sem diferença, o comentário diz isso.
4. `bun run neon:test-db ci/pr-<n>`: recria `app_test` vazio dentro do
   mesmo branch.
5. `bun run neon:urls ci/pr-<n> app_test --github-env`,
   `bun run db:migrate` (do zero, prova que o histórico de migrations
   aplica num banco vazio) e `bun test`.

Por que dois bancos no mesmo branch: `app` valida a migration contra
dados reais e alimenta o schema diff. `app_test` é o banco que os testes
truncam, e o sufixo `_test` mantém válida a guarda do `tests/setup.ts`
do api-bun, que aborta fora de banco `_test`
(`apps/api/docs/testing.md`). Os testes nunca tocam `app`.

Quando a PR é **fechada** (merge ou não), o workflow `neon-cleanup.yml`
roda `bun run neon:delete ci/pr-<n>`.

### Dados de produção no CI

O branch `ci/pr-<n>` é uma cópia de produção, e quem tem o
`NEON_API_KEY` já tem acesso a ela. Mesmo assim:

- **PRs de fork não recebem secrets** do GitHub, então não criam branch.
  O job de teste da API falha nelas por design, e um mantenedor roda a
  validação a partir de uma branch do próprio repo.
- Se a produção guarda dado pessoal que não pode sair de `main` (LGPD,
  contrato), a instância troca para **`--schema-only`** no
  `neon:branch` e registra a decisão em `docs/domain.md`. Branch
  schema-only não tem dados, então o passo 2 deixa de pegar migration
  que falha só com dado real. Esse risco passa a ser coberto em staging.
- Logs de CI nunca imprimem connection string. O `neon:urls` mascara
  (`::add-mask::`) com `--github-env`, e os jobs nunca fazem `echo` de `DATABASE_URL`.

## Staging

- A API de staging roda contra `staging` (URL pooled no host). O deploy
  de staging roda `db:migrate` com o `DATABASE_URL_UNPOOLED` do
  Environment `staging` antes do deploy, igual a produção
  (`docs/ci-cd.md`, `api-deploy.yml`).
- **Resetar staging** traz de volta os dados atuais de produção:

  ```bash
  neonctl branches reset staging --parent
  ```

  O reset troca o schema de `staging` pelo de `main`, então apaga
  migrations aplicadas em staging e ainda não em produção. Logo depois do
  reset, rodar o workflow de deploy de staging (`workflow_dispatch`), que
  reaplica as migrations pendentes.
- Criar conta, organização e dados de teste em staging é permitido.
  Depois de um reset, eles somem.

## Desenvolvimento local contra um branch

O padrão local é o Postgres do Docker (`docs/development.md`). Para
depurar com dados reais, crie um branch descartável a partir de
`staging`, nunca de `main`, e aponte o `.env.local` da API para ele:

```bash
neonctl branches create --name dev/<seu-nome> --parent staging \
  --expires-at <data em RFC 3339>
bun run neon:urls dev/<seu-nome>          # copiar para apps/api/.env.local
```

- **Nunca rodar `bun test` com essa URL.** O banco `app` não termina em
  `_test` e a guarda aborta, que é justamente para isso.
- Apagar o branch ao terminar (`neonctl branches delete dev/<seu-nome>`).
  O `neon:delete` só apaga `ci/`, de propósito.

## Operação

### Restore de produção

Neon guarda histórico por um período que depende do plano (*restore
window*). Rollback de schema continua sendo migration nova (api-bun). O
restore é para **incidente de dados** (delete acidental, migration
destrutiva):

```bash
# 1. inspecionar o passado sem tocar em main
neonctl branches create --name incident/<data> --parent main@2026-09-30T12:00:00Z
neonctl psql incident/<data>

# 2. se for restaurar main inteiro (preserva o estado atual num branch)
neonctl branches restore main ^self@2026-09-30T12:00:00Z --preserve-under-name main-before-restore
```

- O restore de `main` é decisão de incidente, com a API de produção
  parada ou em modo manutenção. Nunca rodar por script de CI.
- Depois de restaurar, conferir que a tabela de migrations do Drizzle
  bate com o código em produção.

### Limpeza de branches esquecidos

```bash
neonctl branches list -o json    # conferir expires_at dos ci/*
```

Branch `ci/*` sem PR aberta pode ser apagado com `bun run neon:delete`.

### Limites do plano

Número de branches, horas de compute e restore window dependem do plano
do Neon. A instância registra em `docs/domain.md` o plano contratado. Se
o limite de branches for baixo, reduzir a expiração do `neon:branch`.

## Decisões registradas

### Um projeto Neon por instância, ambientes como branches

**Decisão**: produção, staging e CI são branches do mesmo projeto Neon.
**Contexto**: branch é cópia copy-on-write instantânea, e é o que torna
barato testar migrations contra dados reais em cada PR e resetar staging
a partir de produção. Com projetos separados, nada disso funciona sem
dump/restore.
**Alternativas consideradas**: um projeto por ambiente (rejeitado: perde
branching entre ambientes); staging fora do Neon (rejeitado: o template
usa o Neon em todo lugar hospedado).

### Wrappers `scripts/neon/*.ts` em vez de `neonctl` solto ou actions do Neon

**Decisão**: CI e devs operam branches pelos scripts `neon:*`, que
chamam o `neonctl`.
**Contexto**: os mesmos comandos rodam local e no CI, e as regras
(expiração, limpeza de URL, só apagar `ci/`) ficam num lugar só e
testável.
**Alternativas consideradas**: actions oficiais do Neon
(`create-branch-action`, `delete-branch-action`, `schema-diff-action`)
(válidas, mas o comportamento fica no YAML e não roda localmente);
`neonctl` direto no YAML (rejeitado: duplicaria as regras em cada
workflow).
