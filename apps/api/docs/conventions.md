# Convenções de código

> Baseado em api-bun v0.16.0, adaptado ao monorepo. Onde este arquivo
> conflita com a raiz, vale a raiz (`../../CLAUDE.md`) e as seções
> "## Diferenças no monorepo" de `CLAUDE.md` e `docs/architecture.md`.

## TypeScript

- `strict: true` sempre. Nunca usar `any` — usar `unknown` e narrowing.
- Preferir `type` a `interface`, exceto quando for para ser estendido por
  outro pacote.
- Nunca usar `enum` do TypeScript — usar `const` object + `as const` ou
  union de literais (regra de código; não confundir com `pgEnum` do
  Drizzle pra colunas de banco, ver `docs/architecture.md`).

## Nomenclatura

- Arquivos: `kebab-case.ts` (ex: `create-task.ts`, `create-task.service.ts`)
- Classes/Types: `PascalCase`
- Funções e variáveis: `camelCase`
- Tabelas do banco: `snake_case` no plural (ex: `releases`, `rights_holders`)
- Coluna de tenant: `organization_id` (`organizationId` no Drizzle), sempre
  via o mixin `tenantColumns` — nunca `tenant_id`, `org_id` ou variações.
- Rotas HTTP: `kebab-case` no plural (ex: `/rights-holders`)

## Imports

- Usar path alias `@` para imports internos, nunca caminho relativo longo
  (`../../../lib/x`).
- Nunca incluir extensão `.js` ou `.ts` no import.

```ts
// ✅ correto
import { createTask } from '@/modules/tasks/features/create-task/create-task.service';

// ❌ não fazer
import { createTask } from '../../../modules/tasks/features/create-task/create-task.service.ts';
```

Configuração do alias em `tsconfig.json` — **`paths` sem `baseUrl`**, não o
par `baseUrl` + `paths` que costumava ser necessário. Desde o TS 4.1,
`paths` funciona sozinho (relativo ao próprio `tsconfig.json`); o TS 6.0
descontinuou `baseUrl` como opção (será removido no TS 7.0), e o resolver
do Bun já lê `paths` sem `baseUrl` corretamente:

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

Nunca adicionar `baseUrl` pra "resolver" o aviso de depreciação — a
correção é tirar `baseUrl`, não silenciar o aviso com
`ignoreDeprecations`.

## Estrutura de uma feature (exemplo de padrão bom)

```ts
// create-task.ts
export const inputSchema = z.object({ title: z.string().min(1) });
export const outputSchema = z.object({ id: z.string(), title: z.string() });

export async function createTask(
  input: z.infer<typeof inputSchema>,
  ctx: TenantContext,
) {
  return createTaskService(input, ctx);
}
```

```ts
// create-task.service.ts — o tenant vem do ctx, nunca do input
export async function createTaskService(
  input: z.infer<typeof inputSchema>,
  ctx: TenantContext,
) {
  const existing = await db.query.tasks.findFirst({
    where: inTenant(tasks, ctx, eq(tasks.title, input.title)),
  });
  if (existing) throw new ConflictError('Task já existe com esse título');

  const [created] = await db
    .insert(tasks)
    .values({ ...input, organizationId: ctx.organizationId })
    .returning();
  return created;
}
```

Evitar (dois padrões ruins — regra de negócio dentro do arquivo da feature,
sem passar pelo service, **e** consulta sem escopo de tenant, que lê e
conflita com a task de qualquer organização):

```ts
// ❌ não fazer isso
export async function createTask(input: CreateTaskInput) {
  const existing = await db.query.tasks.findFirst({
    where: eq(tasks.title, input.title),
  });
  if (existing) throw new ConflictError('Task já existe');
  const [created] = await db.insert(tasks).values(input).returning();
  return created;
}
```

O `organization_id` nunca vem do body nem da URL: o cliente não escolhe o
tenant de uma escrita. Ele vem do `ctx`, montado pelo macro `tenant: true`
a partir da sessão (ver `docs/architecture.md`, "### Tenant: organização
ativa e contexto").

## Erros

- Exceptions de domínio tipadas (`ValidationError`, `UnauthorizedError`,
  `ForbiddenError`, `NotFoundError`, `ConflictError`,
  `TooManyRequestsError`) definidas em `lib/errors.ts` — status de cada
  uma em `docs/architecture.md`, "Tratamento de erros".
- Um error handler global no plugin do Elysia (`.onError`) converte para
  status HTTP — services nunca sabem de status code.

## Validação

- Todo input validado via `inputSchema` (Zod) no arquivo da feature.
- Reaproveitar o `inputSchema`/`outputSchema` como tipo do service
  (`z.infer<typeof inputSchema>`), nunca duplicar tipo manualmente.

## Testes

- Testes de service: usar banco de teste dedicado, limpo com
  `resetDatabase()` no `afterEach` (ver `docs/testing.md`, "Banco de
  teste"), sem repository para mockar.
- Testes de feature: subir instância Elysia em memória (`.handle(request)`),
  testar contrato HTTP.
- Nome do arquivo de teste espelha o arquivo testado: `x.service.ts` →
  `x.service.test.ts`.

## Commits

Conventional Commits (`tipo(escopo): descrição`). Validado automaticamente
pelo hook `commit-msg` (Husky + commitlint) — não é convenção "de boa
vontade", é bloqueada por tooling; nunca commitar com `--no-verify` para
contornar.

Tabela de tipos, exemplos reais e configuração de Husky/commitlint em
`../../docs/git-workflow.md`.

## Lint e formatação

- Biome v2 cuida de lint + format. Rodar `bun run check` antes de commitar.
- Aspas simples (`'`) em vez de duplas.
- Ponto e vírgula só quando necessário para evitar ambiguidade de ASI —
  usar `asNeeded`, não `always`.
- Indentação: 1 tab (não espaço).
- Organizar imports automaticamente (assist do Biome ativado).
- Não sobrescrever regras do Biome sem justificativa registrada aqui.
- Os snippets em `docs/*.md` são ilustrativos: usam `;` e 2 espaços por
  legibilidade na doc e **não** seguem o formatter. O código real do
  projeto é sempre o que passa em `bun run check`.

Configuração equivalente em `biome.json`. `files.includes` tira do Biome
o que é gerado: as migrations do `drizzle-kit` (`!` — não formata nem lint,
mas segue indexado) e o `dist` do build (`!!` — fora de qualquer operação);
sem isso, `bun run check` reclama de arquivo que ninguém escreve à mão.

```json
{
  "files": {
    "includes": ["**", "!**/src/db/migrations", "!!**/dist"]
  },
  "formatter": {
    "indentStyle": "tab"
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "semicolons": "asNeeded"
    }
  },
  "assist": {
    "actions": {
      "source": {
        "organizeImports": "on"
      }
    }
  }
}
```

### Setup do VSCode

`.vscode/settings.json` (commitado no repo, pra não depender de config manual
em cada máquina — mesmo trabalhando sozinho, evita perder tempo reconfigurando
após reinstalar o VSCode):

```json
{
	"prettier.enable": false,
	"eslint.enable": false,

	"biome.lsp.bin": "./node_modules/.bin/biome",

	"editor.defaultFormatter": "biomejs.biome",
	"editor.formatOnSave": true,
	"editor.codeActionsOnSave": {
		"source.organizeImports.biome": "explicit",
		"source.fixAll.biome": "explicit"
	},

	"[typescript]": {
		"editor.defaultFormatter": "biomejs.biome"
	},
	"[javascript]": {
		"editor.defaultFormatter": "biomejs.biome"
	}
}
```

Requer a extensão `biomejs.biome` instalada no VSCode. Prettier e ESLint
desabilitados explicitamente evita conflito de formatter quando a extensão
estiver instalada por hábito/dependência de outro projeto.

## Diferenças no monorepo

- **Biome**: um `biome.json` só, na raiz do monorepo, com este mesmo
  estilo (tabs, aspas simples, `semicolons: "asNeeded"`). A API não tem
  `biome.json` próprio. `bun run check` roda na raiz. O ignore de
  `src/db/migrations` vira `apps/api/src/db/migrations` na raiz
  (`../../docs/conventions.md`, "## Biome").
- **VSCode**: `.vscode/` na raiz do monorepo.
- **Imports de outro workspace**: só `@repo/contracts/<subpath>`. O alias
  `@/` continua sendo só do `src/` da API e nunca aparece dentro de
  `packages/*`.
- **Zod**: `packages/contracts` também importa de `zod/v4`, com a mesma
  versão de `zod` da API (uma só no monorepo).
- **Nome de módulo**: o mesmo em `src/modules/<modulo>`,
  `packages/contracts/src/<modulo>.ts` e `apps/web/src/features/<modulo>`,
  e é o escopo do commit (`../../docs/git-workflow.md`).
