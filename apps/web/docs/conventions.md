# Convenções de código

> Baseado em app-nextjs v0.1.0, adaptado ao monorepo. Onde este arquivo
> conflita com a raiz, vale a raiz (`../../CLAUDE.md`) e as seções
> "## Diferenças no monorepo" de `CLAUDE.md` e `docs/architecture.md`.

## TypeScript

- `strict: true` sempre. Nunca `any`: usar `unknown` e narrowing (ou o
  tipo inferido de um schema Zod).
- Preferir `type` a `interface`, exceto quando for para ser estendido.
- Nunca `enum` do TypeScript: usar `as const` + union de literais.
- `import type` para import só de tipo (o Biome já organiza).
- Tipos de dado da API são **inferidos do schema Zod**
  (`z.infer<typeof taskSchema>`), nunca escritos à mão em paralelo.
- `params`, `searchParams`, `cookies()` e `headers()` são `Promise` no
  Next 16: sempre `await`. Usar os helpers globais gerados pelo Next
  (`PageProps<'/rota'>`, `LayoutProps<'/rota'>`) para tipar páginas e
  layouts. Eles são gerados por `next dev`/`next build`/`next typegen`, por
  isso `bun run typecheck` roda `next typegen` antes do `tsc`.

## Nomenclatura

- Arquivos e pastas: `kebab-case` (`tasks-table.tsx`, `use-task-filters.ts`).
  Exceção: arquivos de convenção do Next (`page.tsx`, `layout.tsx`,
  `error.tsx`, `proxy.ts`, …).
- Componentes: `PascalCase` (`TasksTable`). Um componente exportado por
  arquivo, com o nome do arquivo em kebab-case.
- Hooks: `useCamelCase`, arquivo `use-*.ts`.
- Funções e variáveis: `camelCase`. Constantes de módulo realmente fixas:
  `SCREAMING_SNAKE_CASE`.
- Schemas Zod: `camelCase` + sufixo `Schema` (`createTaskSchema`); tipo
  inferido em `PascalCase` sem sufixo (`CreateTask`).
- Query options e keys: `<recurso>Query`, `<recurso>Keys`
  (`tasksQuery`, `taskKeys`).
- Módulos em `features/` usam **o mesmo nome** do módulo no api-bun
  (`tasks`, `rights-holders`).
- Rotas: `kebab-case` no plural (`/rights-holders`), espelhando a API
  sempre que fizer sentido.

## Imports

- Alias `@/` para tudo interno (`@/*` → `./src/*`), nunca `../../..`.
  Import relativo só dentro da mesma pasta (`./tasks-table-row`).
- Sem extensão `.ts`/`.tsx` no import.
- Sem barrel (`index.ts` re-exportando tudo) em `features/`: import direto
  do arquivo. Barrel atrapalha tree-shaking e mistura server e client no
  mesmo grafo.

## Componentes

- Server Component por padrão; `"use client"` só na folha (ver
  `docs/architecture.md`, "## Server e Client Components").
- Props tipadas com `type <Nome>Props = { … }`. Nada de `React.FC`.
- Sem `useMemo`/`useCallback`/`memo` defensivos: o React Compiler faz isso.
- Sem `useEffect` para derivar estado ou buscar dado. Dado vem de RSC ou
  TanStack Query; valor derivado é calculado no render.
- `key` estável (id da entidade), nunca índice de array em lista que muda.

## Estrutura de uma feature (exemplo de padrão bom)

```
src/features/tasks/
  api/
    queries.ts            # taskKeys + queryOptions (leitura)
    mutations.ts          # funções de escrita (create/update/delete)
  components/
    tasks-table.tsx       # "use client": useQuery + filtros via nuqs
    task-form.tsx         # "use client": RHF + zod
    task-status-badge.tsx # server-safe, sem estado
  hooks/
    use-task-filters.ts   # useQueryStates(tasksSearchParams)
  schemas/
    task.ts               # taskSchema, taskListSchema, createTaskSchema
  search-params.ts        # parsers nuqs (fonte única client + server)
```

## Formulários

```tsx
// src/features/tasks/components/task-form.tsx
'use client';

import { zodResolver } from '@hookform/resolvers/zod';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useForm } from 'react-hook-form';
import { createTask } from '@/features/tasks/api/mutations';
import { taskKeys } from '@/features/tasks/api/queries';
import { type CreateTask, createTaskSchema } from '@/features/tasks/schemas/task';
import { applyApiIssues } from '@/lib/api/errors';

export function TaskForm({ organizationId }: { organizationId: string }) {
  const queryClient = useQueryClient();
  const form = useForm<CreateTask>({
    resolver: zodResolver(createTaskSchema),
    defaultValues: { title: '' },
  });

  const mutation = useMutation({
    mutationFn: createTask,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: taskKeys.all(organizationId) }),
    onError: (error) => applyApiIssues(error, form.setError), // 400 com issues → campos
  });

  return <form onSubmit={form.handleSubmit((values) => mutation.mutate(values))}>{/* FormField… */}</form>;
}
```

- `defaultValues` sempre completos (inputs controlados do shadcn quebram
  com `undefined`).
- Botão de submit desabilitado com `mutation.isPending`.
- `applyApiIssues` (em `lib/api/errors.ts`) é o único lugar que traduz
  `issues` do api-bun para `setError`.

## Estilo (Tailwind + shadcn)

- Só classes Tailwind e tokens do tema. Sem CSS modules, sem `style={{}}`
  para cor, espaçamento ou tipografia.
- Combinar classes com `cn()` (`lib/utils.ts`), nunca concatenação de
  string.
- Variantes de componente com `cva` (padrão do shadcn), não ternários
  espalhados em `className`.
- Componente do shadcn precisa de ajuste? Editar em `components/ui/`. Se o
  ajuste for específico de um módulo, criar um wrapper em
  `features/<modulo>/components/`.

## Acessibilidade (mínimo obrigatório)

- Todo input com `<Label>` associado (o `FormField` do shadcn já faz).
- Botão só com ícone tem `aria-label` (ou `<span className="sr-only">`).
- Interação por teclado funcionando. Não trocar `<button>` por `<div
  onClick>`.
- Imagens com `alt` (vazio se decorativa); usar `next/image`.
- `lang` do `<html>` coerente com o idioma da instância.

## Erros

- Chamada à API lança `ApiError` (ver `docs/architecture.md`, "### Erros
  da API"). Nunca `throw new Error("string")` para erro esperado de
  negócio.
- Nunca engolir erro (`catch {}` vazio). Ou trata e informa o usuário, ou
  deixa subir para `error.tsx`.

## Testes

Ver `docs/testing.md`. Nome de arquivo: `<arquivo>.test.ts(x)` ao lado do
código testado; E2E em `e2e/<fluxo>.spec.ts`.

## Commits

Conventional Commits, ver `../../docs/git-workflow.md`.

## Lint e formatação

Biome v2 (`biome.json`), com os domínios `next` e `react` ligados e
`organizeImports`. Scripts:

```bash
bun run check        # biome check --write (lint + format + imports)
bun run lint         # biome check (sem escrever — é o que o CI roda)
```

Nunca desligar regra com `biome-ignore` sem comentário explicando o
motivo na mesma linha.

### Setup do VSCode

Extensão oficial do Biome (`biomejs.biome`) como formatter padrão, e a do
Tailwind CSS IntelliSense. `.vscode/settings.json` commitado:

```json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.biome": "explicit",
    "source.organizeImports.biome": "explicit"
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "files.associations": { "*.css": "tailwindcss" }
}
```

E `.vscode/extensions.json` recomendando `biomejs.biome` e
`bradlc.vscode-tailwindcss`.

## Diferenças no monorepo

- **Biome**: `apps/web/biome.json` estende a raiz (`"extends": "//"`) e só
  liga os domínios `next`/`react` e as diretivas do Tailwind. O estilo de
  formatação é o da raiz (tabs, aspas simples, sem ponto e vírgula
  obrigatório). `bun run check`/`bun run lint` rodam **na raiz**
  (`../../docs/conventions.md`, "## Biome").
- **VSCode**: `.vscode/` fica na raiz do monorepo, e o editor é aberto na
  raiz. Acrescentar ao `settings.json` da raiz o
  `"files.associations": { "*.css": "tailwindcss" }` acima.
- **Imports de outro workspace**: só `@repo/contracts/<subpath>`, nunca
  caminho relativo para fora de `apps/web` (`../../docs/conventions.md`,
  "## Imports entre workspaces").
- **Nome de módulo**: o mesmo em `features/<modulo>`,
  `packages/contracts/src/<modulo>.ts` e `apps/api/src/modules/<modulo>`.
