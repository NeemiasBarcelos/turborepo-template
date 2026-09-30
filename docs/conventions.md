# Convenções do monorepo

Convenções de código **dentro** de cada app continuam em
`apps/web/docs/conventions.md` e `apps/api/docs/conventions.md`. Este
arquivo cobre o que atravessa workspaces.

## Nomes de workspace e de pastas

- Workspaces: `@repo/<nome>`, com `<nome>` igual à pasta (`apps/web` →
  `@repo/web`, `packages/contracts` → `@repo/contracts`).
- Pastas em kebab-case. Pacote novo em `packages/` só quando **dois ou
  mais** workspaces precisam do código. Código usado por um app só fica
  no app.
- Nome de módulo de negócio é **o mesmo** em todo lugar:
  `apps/api/src/modules/<modulo>`, `apps/web/src/features/<modulo>`,
  `packages/contracts/src/<modulo>.ts` e o escopo do commit.

## Imports entre workspaces

```ts
// ✅ apps/web/src/features/tasks/api/list-tasks.ts
import { taskListResponseSchema } from '@repo/contracts/tasks'

// ✅ apps/api/src/modules/tasks/features/list-tasks.ts
import { taskListResponseSchema } from '@repo/contracts/tasks'

// ❌ caminho relativo atravessando workspace
import { taskListResponseSchema } from '../../../../packages/contracts/src/tasks'

// ❌ app importando de outro app
import { tasks } from '@repo/api/src/db/schema/tasks'

// ❌ alias @/ dentro de packages/* (resolve para o src/ do app consumidor)
import { moduleRoles } from '@/roles'
```

- Import de pacote sempre por **subpath** declarado em `exports`
  (`@repo/contracts/tasks`), nunca pela raiz com barrel. Isso segue a
  regra "sem barrels" dos dois apps e mantém o bundle do web enxuto.
- Novo arquivo em `packages/contracts/src/` já é exportado pelo curinga
  `"./*"`. Arquivo interno do pacote, que não deve ser importado de fora,
  fica em `src/internal/` e não tem entrada em `exports`.
- Dependência de workspace declarada no `package.json` de quem usa
  (`"@repo/contracts": "workspace:*"`). Import sem dependência declarada
  funciona por acidente no Bun (hoisting) e quebra no Docker com
  `--filter`.

## Biome

Um `biome.json` na raiz (`docs/architecture.md`, "### Biome único na
raiz, no estilo do api-bun"):

```jsonc
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/2.4.2/schema.json",
  "root": true,
  "vcs": { "enabled": true, "clientKind": "git", "useIgnoreFile": true },
  "files": {
    "includes": ["**", "!**/dist", "!**/.next", "!**/.turbo", "!apps/api/src/db/migrations"]
  },
  "formatter": { "enabled": true, "indentStyle": "tab", "lineWidth": 100 },
  "javascript": {
    "formatter": { "quoteStyle": "single", "semicolons": "asNeeded" }
  },
  "linter": {
    "enabled": true,
    "rules": { "recommended": true }
  },
  "assist": { "actions": { "source": { "organizeImports": "on" } } }
}
```

```jsonc
// apps/web/biome.json
{
  "root": false,
  "extends": "//",
  "linter": {
    "domains": { "next": "recommended", "react": "recommended" }
  },
  "css": { "parser": { "tailwindDirectives": true } }
}
```

- A versão do `$schema` acompanha a do `@biomejs/biome` da raiz. Ao subir
  o Biome, rodar `bunx biome migrate --write`.
- `apps/api` não precisa de `biome.json` próprio: herda a raiz.
- Exemplos de código copiados das docs dos templates podem estar em outro
  estilo (o app-nextjs usa aspas duplas). O `bun run check` normaliza, e
  a doc não é reescrita só por isso.

## TypeScript

- Todo workspace estende uma base de `@repo/tsconfig`
  (`docs/architecture.md`, "### `packages/tsconfig`").
- `typecheck` existe em todo workspace com código (`tsc --noEmit`; no
  web, `next typegen && tsc --noEmit`).
- Não usar *project references* (`composite`). Os pacotes são JIT e o
  `tsc` de cada app já enxerga o fonte do `contracts`.

## Scripts de workspace

Todo workspace expõe os mesmos nomes de script quando se aplicam, para
o turbo orquestrar:

| Script | web | api | contracts |
|---|---|---|---|
| `dev` | `next dev` | `bun --watch src/index.ts` | — |
| `build` | `next build` | `bun build src/index.ts --outdir dist --target bun` | — |
| `start` | `next start` | `bun dist/index.js` | — |
| `typecheck` | `next typegen && tsc --noEmit` | `tsc --noEmit` | `tsc --noEmit` |
| `test` | `vitest run` | `bun test` | `bun test` |
| `test:e2e` | `playwright test` | — | — |
| `db:generate` / `db:migrate` / `db:studio` | — | drizzle-kit | — |

- Lint e format **não** são scripts de workspace: `bun run lint` e
  `bun run check` rodam o Biome na raiz.
- Script novo que o CI precisa rodar entra no `turbo.json` como task.

## Arquivos de configuração

- **Na raiz**: `biome.json`, `turbo.json`, `.bun-version`, `.nvmrc`,
  `docker-compose.yml`, `.dockerignore`, `commitlint.config.mjs`,
  `.husky/`, `.github/`, `.vscode/`.
- **No app**: tudo que é do framework (`next.config.ts`,
  `drizzle.config.ts`, `bunfig.toml`, `vitest.config.mts`,
  `playwright.config.ts`, `components.json`, `.env*`).
- Nunca duplicar config da raiz dentro de um app (segundo `biome.json`
  com `"root": true`, segundo `.husky/`).

## `.gitignore`

```gitignore
node_modules
.turbo
dist
.next
coverage
playwright-report
test-results
.env*
!.env.example
!apps/api/.env.test
.neon
.vercel
```

## VSCode

```jsonc
// .vscode/settings.json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.biome": "explicit",
    "source.organizeImports.biome": "explicit"
  },
  "typescript.tsdk": "node_modules/typescript/lib"
}
```

```jsonc
// .vscode/extensions.json
{ "recommendations": ["biomejs.biome", "bradlc.vscode-tailwindcss"] }
```

Abrir o editor **na raiz** do monorepo, não em `apps/*`. É a raiz que tem
o `biome.json` com `"root": true` e o `typescript` único.
