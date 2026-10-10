# Arquitetura

> Baseado em app-nextjs v0.1.0, adaptado ao monorepo. Onde este arquivo
> conflita com a raiz, vale a raiz (`../../CLAUDE.md`) e as seções
> "## Diferenças no monorepo" de `CLAUDE.md` e `docs/architecture.md`.

Este template é o **frontend** das APIs derivadas do template
[`api-bun`](https://github.com/NeemiasBarcelos/api-bun). Tudo o que é regra
de negócio, persistência, autenticação e autorização mora na API; o front
renderiza, coleta input e chama a API. Os conceitos de tenant, organização
ativa e roles por módulo vêm de lá — ler `../api/docs/architecture.md` (o api-bun deste monorepo)
("### Multi-tenancy", "## Autenticação e autorização") antes de mexer em
auth ou em qualquer dado de negócio aqui.

> Next.js **16** tem mudanças que quebram o que costuma estar em material
> de treino e tutoriais antigos: `middleware.ts` virou **`proxy.ts`**,
> `params`/`searchParams`/`cookies()`/`headers()` são **assíncronos**,
> Turbopack é o bundler padrão (dev e build) e `next lint` foi removido. A
> referência é sempre `node_modules/next/dist/docs/` da versão instalada
> (ver `AGENTS.md`).

## Estrutura de pastas

```
src/
  app/                      # SÓ roteamento: layouts, pages, loading, error, not-found
    (auth)/                 # route group das telas públicas (sign-in, sign-up, accept-invitation/[id])
    (app)/                  # route group autenticado — layout chama requireTenant() e monta o shell
      layout.tsx            # shell: OrganizationSwitcher, ThemeToggle, SignOutButton
      page.tsx              # home: saudação + info de dono (GET /api/users/me/roles)
      [modulo]/…            # uma pasta de rota por módulo de negócio
    organizations/select/   # FORA dos grupos: só requireSession() (ver "### Rotas e shell de base")
    layout.tsx              # root layout: fontes, <Providers>
    providers.tsx           # "use client": QueryClientProvider + NuqsAdapter + Toaster
    globals.css             # Tailwind v4 + tokens de tema do shadcn
  features/
    <modulo>/               # ex: tasks, members, organizations
      components/           # componentes do módulo (server e client)
      hooks/                # hooks client do módulo (useTasksFilters, …)
      api/                  # funções de chamada à API + queryOptions do TanStack Query
      schemas/              # schemas Zod (forms e parse de resposta)
      search-params.ts      # parsers nuqs do módulo (quando houver)
      store.ts              # store zustand do módulo (exceção — ver "## Estado")
  components/
    ui/                     # componentes do shadcn/ui (gerados pela CLI, editáveis)
    <compartilhados>.tsx    # componentes reutilizados por mais de um módulo
  lib/
    env.schema.ts           # schema Zod do env de server (sem server-only)
    env.ts                  # env validado com Zod (server)
    env.client.ts           # env público (NEXT_PUBLIC_*) validado com Zod
    api/
      client.ts             # instância ky do browser
      server.ts             # instância ky do server (repassa cookies)
      errors.ts             # ApiError + parse do corpo de erro (schema de @repo/contracts/errors)
    auth-client.ts          # createAuthClient do Better Auth + organizationClient
    auth-server.ts          # getSession()/requireSession() para RSC e server actions
    query-client.ts         # makeQueryClient / getQueryClient
    utils.ts                # cn() e helpers genéricos
  proxy.ts                  # checagem OTIMISTA de sessão (redirect), nada além
tests/
  setup.ts                  # setup do Vitest (jest-dom, MSW)
  mocks/                    # handlers MSW que imitam o api-bun
e2e/                        # specs do Playwright
```

O scaffold do `create-next-app` deixou `app/` na raiz e o alias `@/*` →
`./*`. O scaffold da baseline **move tudo para `src/`** e troca o alias
para `@/*` → `./src/*` (ver "### `src/` + features por módulo" em
"## Decisões registradas").

Regra de dependência entre pastas:

- `app/` importa de `features/`, `components/` e `lib/`. Nunca o contrário.
- `features/<a>` **não importa** de `features/<b>`. O que for compartilhado
  sobe para `components/` ou `lib/`.
- `components/ui/` não conhece domínio: nada de chamada à API nem regra de
  negócio ali dentro.
- `lib/` não importa de `features/` nem de `app/`.

O Biome não valida isso sozinho; é revisão de PR (`../../docs/checklists.md`).

### Onde fica cada coisa de uma rota

Uma `page.tsx` é fina: lê `params`/`searchParams` (com `await`), carrega o
que precisa no server e compõe componentes de `features/`. Lógica de tela,
formulário, tabela e afins vivem em `features/<modulo>/components/`.

```tsx
// src/app/(app)/tasks/page.tsx
import type { SearchParams } from 'nuqs/server';
import { TasksTable } from '@/features/tasks/components/tasks-table';
import { loadTasksSearchParams } from '@/features/tasks/search-params';
import { requireTenant } from '@/lib/auth-server';

export default async function TasksPage({ searchParams }: { searchParams: Promise<SearchParams> }) {
  const tenant = await requireTenant();
  const filters = await loadTasksSearchParams(searchParams);
  return <TasksTable organizationId={tenant.organizationId} initialFilters={filters} />;
}
```

## Server e Client Components

- **Server Component é o padrão.** `"use client"` só no componente folha
  que realmente precisa de estado, efeito, evento ou API do browser, nunca
  num layout ou página inteira "por garantia".
- Componente client não importa código que só roda no server
  (`lib/env.ts`, `lib/api/server.ts`, `lib/auth-server.ts`). Esses módulos
  começam com `import 'server-only'`, então o build quebra se isso
  acontecer.
- Props que atravessam a fronteira server → client precisam ser
  serializáveis (sem função, classe, `Date` só se o consumidor aceitar
  string).
- **React Compiler** está ligado (`reactCompiler: true`): não escrever
  `useMemo`/`useCallback`/`memo` por reflexo. Só quando houver um motivo
  medido e comentado.

## Comunicação com o api-bun

### Mesma origem via rewrite

O browser **nunca** chama o api-bun direto em outra origem. O
`next.config.ts` faz rewrite de `/api/:path*` para a API:

```ts
// next.config.ts
import type { NextConfig } from 'next';
import { serverEnvSchema } from './src/lib/env.schema';

const env = serverEnvSchema.parse(process.env);

const nextConfig: NextConfig = {
  reactCompiler: true,
  async rewrites() {
    return [{ source: '/api/:path*', destination: `${env.API_URL}/api/:path*` }];
  },
};

export default nextConfig;
```

Com isso, os cookies de sessão do Better Auth são **first-party** do
domínio do front, sem CORS nem cookie `SameSite=None`, e o mesmo código
funciona em localhost, preview da Vercel e produção. O api-bun precisa
servir tudo debaixo de `/api` (o Better Auth já monta em `/api/auth/…`) e
ter a origem do front em `TRUSTED_ORIGINS`, porque o Better Auth valida o
`Origin` das requisições mutáveis.

> Se o api-bun da instância não usa o prefixo `/api` nas rotas de negócio,
> ajustar o `destination` do rewrite. A regra é "o browser só fala com a
> própria origem", não o formato exato do prefixo.

### Cliente HTTP: ky

Toda chamada ao api-bun passa por uma das duas instâncias ky de
`lib/api/`. Nunca `fetch` solto nem `ky` importado direto numa feature.

```ts
// src/lib/api/client.ts — browser
import ky from 'ky';
import { toApiError } from '@/lib/api/errors';

export const api = ky.create({
  baseUrl: '/api/', // relativo: resolvido contra a origem da página
  credentials: 'include',
  retry: { limit: 1, methods: ['get'] },
  hooks: { beforeError: [toApiError] },
});
```

```ts
// src/lib/api/server.ts — RSC, server actions e route handlers
import 'server-only';
import ky from 'ky';
import { headers } from 'next/headers';
import { toApiError } from '@/lib/api/errors';
import { env } from '@/lib/env';

export async function serverApi() {
  const cookie = (await headers()).get('cookie') ?? '';
  return ky.create({
    baseUrl: `${env.API_URL}/api/`,
    headers: { cookie },
    retry: 0,
    hooks: { beforeError: [toApiError] },
  });
}
```

No server a chamada vai **direto** a `API_URL` (rede interna, sem passar
pelo rewrite), repassando o cookie da requisição original. Por isso
`serverApi()` é uma função: ela lê o cookie do request atual e não pode ser
um singleton de módulo.

Snippets validados com o **ky 2**. Em relação ao ky 1: `prefixUrl` virou
`prefix`/`baseUrl` (a baseline usa `baseUrl`, como o ky recomenda, com a
barra final e caminhos sem barra inicial: `api.get('users/me/roles')`); o
hook `beforeError` recebe um objeto de estado (`({ error })`), não o erro;
e o corpo do erro vem pré-lido em `error.data` (`error.response.json()` não
funciona mais).

### Erros da API

O api-bun responde erro sempre como
`{ statusCode, error, message, issues? }` (ver "## Tratamento de erros" no
api-bun). `lib/api/errors.ts` converte o `HTTPError` do ky numa `ApiError`
tipada com esses campos, e o resto do front só conhece `ApiError`:

```ts
// src/lib/api/errors.ts (trecho)
import { apiErrorSchema } from '@repo/contracts/errors';
import { type BeforeErrorState, isHTTPError } from 'ky';

export function toApiError({ error }: BeforeErrorState): Error {
  if (!isHTTPError(error)) return error;
  const parsed = apiErrorSchema.safeParse(error.data); // corpo já lido pelo ky 2
  if (parsed.success) return new ApiError(parsed.data);
  return new ApiError({
    statusCode: error.response.status,
    error: 'HttpError',
    message: 'Não foi possível completar a requisição',
  });
}
```


| Status | Tratamento no front |
|---|---|
| 400 com `issues` | `setError` do React Hook Form campo a campo (ver `docs/conventions.md`) |
| 401 | sessão expirou: redireciona para `/sign-in?next=<rota atual>` |
| 403 | sem permissão ou sem organização ativa: mensagem, sem retry |
| 404 | `notFound()` no server; estado vazio no client. Inclui recurso de outro tenant, e o front **não** tenta distinguir |
| 409 / 429 | toast com `message` da API |
| 5xx / rede | `error.tsx` da rota (server) ou toast + retry (client) |

Mensagem exibida ao usuário vem do `message` da API ou de texto do próprio
front, nunca do stack trace.

## Busca de dados

### Server first, TanStack Query no client

- **Leitura inicial de página**: no Server Component, via `serverApi()`.
  Sem waterfall: disparar em paralelo o que for independente
  (`Promise.all`) e usar `<Suspense>` + `loading.tsx` para streaming.
- **Dado que muda com interação no client** (paginação, filtros
  instantâneos, polling, refetch após mutation): **TanStack Query**. Quando
  a página também renderiza no server, prefetch no RSC e
  `HydrationBoundary` + `dehydrate` para o client não refazer a request
  (guia *Advanced Server Rendering* do TanStack Query).
- **Mutações**: `useMutation` chamando o `api` do browser, com
  `invalidateQueries` das chaves afetadas no `onSuccess`. Server Actions
  ficam para mutações simples de formulário sem estado de cache no client.
  Não misturar as duas coisas para o mesmo recurso.

`QueryClient`: um por request no server e um singleton no browser
(`lib/query-client.ts`, padrão do guia do TanStack Query), com `staleTime`
padrão acima de zero (60 s) para não refazer no client logo após hidratar.

### Chaves de query com tenant

Toda query de dado de negócio inclui o `organizationId` como **primeiro
segmento** da chave, e as chaves de um módulo nascem de uma fábrica única:

```ts
// src/features/tasks/api/queries.ts
import { queryOptions } from '@tanstack/react-query';
import { api } from '@/lib/api/client';
import { taskListSchema, type TaskFilters } from '@/features/tasks/schemas/task';

export const taskKeys = {
  all: (orgId: string) => [orgId, 'tasks'] as const,
  list: (orgId: string, filters: TaskFilters) => [...taskKeys.all(orgId), 'list', filters] as const,
};

export const tasksQuery = (orgId: string, filters: TaskFilters) =>
  queryOptions({
    queryKey: taskKeys.list(orgId, filters),
    queryFn: async () => taskListSchema.parse(await api.get('tasks', { searchParams: filters }).json()),
  });
```

Ao **trocar a organização ativa**, o front chama o endpoint do Better Auth,
dá `queryClient.clear()` e `router.refresh()`. Nada de dado da organização
anterior sobrevive em cache, e o prefixo por `organizationId` garante que,
mesmo com um bug nessa limpeza, uma chave nunca serve dado da outra.

### Validação de resposta

Resposta do api-bun é validada com Zod (`schema.parse`) na fronteira, em
`features/<modulo>/api/`. O componente recebe tipo inferido do schema,
nunca `unknown` nem um tipo escrito à mão que "deveria" bater com a API.

### Cache do Next.js

`cacheComponents` fica **desligado** e nada usa `'use cache'` na baseline:
praticamente todo dado é por sessão e por tenant, e cache compartilhado de
dado autenticado é a forma mais fácil de vazar dado entre organizações (ver
"### Cache Components desligado" em "## Decisões registradas"). Página
pública e estática (landing, termos) pode usar cache, isolada e registrada
na PR.

## Estado

A pergunta é sempre **"onde este estado deve morar?"**, respondida nesta
ordem. Vence a primeira opção que resolve:

1. **Derivado**: dá para calcular a partir de props ou de outro estado?
   Então não é estado, é uma variável.
2. **Local (`useState`/`useReducer`)**: só um componente (e seus filhos
   diretos) usa.
3. **URL (nuqs)**: o estado precisa sobreviver a refresh, ser
   compartilhável por link ou funcionar com voltar/avançar: filtros, busca,
   paginação, ordenação, aba ativa, id do item aberto num drawer.
4. **Estado do servidor (TanStack Query / RSC)**: veio da API. Nunca é
   copiado para `useState`, context ou zustand.
5. **Context**: valor estável compartilhado por uma subárvore (tema, sessão
   já carregada, configuração do módulo). Evitar para valor que muda com
   frequência.
6. **zustand**: estado **de UI**, client-only, compartilhado por
   componentes distantes na árvore, que muda com frequência e onde context
   causaria re-render amplo ou prop drilling pesado. Ex: estado de um
   editor, seleção em massa numa tabela grande, painel lateral controlado
   de vários pontos.

Usar zustand exige justificar na PR por que 1–5 não bastam. Regras quando
usar:

- Store **por módulo** (`features/<modulo>/store.ts`), nunca um store
  global único.
- Nunca guardar dado da API nem nada que identifique tenant. Trocar de
  organização não pode deixar resíduo em store.
- Com SSR, store criado por árvore (provider + `createStore`), não
  singleton de módulo, se o estado inicial depender do request.
- Seletores (`useStore((s) => s.x)`) em vez de pegar o store inteiro.

### nuqs

`NuqsAdapter` (`nuqs/adapters/next/app`) fica em `app/providers.tsx`.
Parsers de cada módulo moram em `features/<modulo>/search-params.ts` e são
a **fonte única**: o mesmo objeto alimenta `useQueryStates` no client e
`createLoader`/`createSearchParamsCache` (de `nuqs/server`) no server.

```ts
// src/features/tasks/search-params.ts
import { createLoader, parseAsInteger, parseAsString, parseAsStringLiteral } from 'nuqs/server';

export const tasksSearchParams = {
  q: parseAsString.withDefault(''),
  page: parseAsInteger.withDefault(1),
  status: parseAsStringLiteral(['open', 'done'] as const),
};

export const loadTasksSearchParams = createLoader(tasksSearchParams);
```

Atualizações são `shallow` por padrão (não re-renderizam o server). Usar
`shallow: false` só quando o Server Component depende do parâmetro, e
combinar com `startTransition` para o estado de carregamento.

## Formulários

React Hook Form + `zodResolver`, com o schema Zod em
`features/<modulo>/schemas/`. O mesmo schema tipa o formulário e valida o
input antes de enviar. Componentes de formulário são os do shadcn/ui 4:
`Field`, `FieldLabel`, `FieldError` (`components/ui/field.tsx`), ligados ao
RHF por `Controller` (o antigo `Form`/`FormField` não existe mais no
shadcn 4). Erro 400 com `issues` do api-bun é mapeado de
volta para os campos (`setError`). Detalhes e exemplo em
`docs/conventions.md`.

O schema do front **não substitui** a validação da API: é UX. Quem decide o
que é válido é o `inputSchema` do api-bun.

## Autenticação e tenant

### Better Auth client

```ts
// src/lib/auth-client.ts
import { organizationClient } from 'better-auth/client/plugins';
import { createAuthClient } from 'better-auth/react';

export const authClient = createAuthClient({
  // mesma origem: o rewrite de /api/* entrega para o api-bun
  plugins: [organizationClient()],
});
```

Login, logout, cadastro, criar organização, convidar, aceitar convite e
trocar a organização ativa usam **os métodos do `authClient`**
(`signIn.email`, `organization.setActive`, …), que batem nos endpoints do
plugin no api-bun. O front não reimplementa nada disso.

### Sessão no server

```ts
// src/lib/auth-server.ts
import 'server-only';
import { redirect } from 'next/navigation';
import { cache } from 'react';
import { sessionSchema } from '@repo/contracts/session';
import { serverApi } from '@/lib/api/server';

export const getSession = cache(async () => {
  const api = await serverApi();
  return sessionSchema.nullable().parse(await api.get('auth/get-session').json());
});

export async function requireSession() {
  const session = await getSession();
  if (!session) redirect('/sign-in');
  return session;
}

export async function requireTenant() {
  const session = await requireSession();
  const organizationId = session.session.activeOrganizationId;
  if (!organizationId) redirect('/organizations/select');
  return { ...session, organizationId };
}
```

A resposta é validada com o `sessionSchema` de `@repo/contracts/session`
(`.nullable()`: sem sessão, o endpoint devolve `null`), como qualquer outra
resposta da API. O tipo `Session` sai do schema, nunca escrito à mão.

`cache()` do React deduplica a chamada dentro do mesmo render. O layout de
`(app)/` chama `requireTenant()`, e páginas que precisam dos dados da
sessão chamam de novo sem custo extra.

### `proxy.ts`: só otimista

`src/proxy.ts` checa **apenas a presença** do cookie de sessão
(`getSessionCookie` de `better-auth/cookies`) e redireciona para
`/sign-in` quem claramente não está logado. Não valida sessão, não chama a
API, não decide permissão. A doc do Next e a do Better Auth dizem o mesmo:
proxy não é camada de autorização. A checagem real é o `requireSession()`
no layout e, acima de tudo, a própria API.

### Rotas e shell de base

O scaffold, antes de qualquer módulo de negócio, já tem:

- **`(auth)/`**: `sign-in` (com `?next=` validado para só aceitar caminho
  interno), `sign-up` e `accept-invitation/[id]`.
- **`src/app/organizations/select/page.tsx`**, **fora** dos route groups:
  só `requireSession()`. É para onde `requireTenant()` redireciona quem não
  tem organização ativa. Dentro de `(app)/` ela entraria em loop, porque o
  layout de `(app)/` chama `requireTenant()`.
- **Shell do `(app)/layout.tsx`**: `requireTenant()`, cabeçalho com
  `OrganizationSwitcher` (`features/organizations/components/`),
  `ThemeToggle` e `SignOutButton` (`components/`).
- **Home (`(app)/page.tsx`)**: saudação com o nome do usuário e se ele é
  dono da organização ativa, via `GET /api/users/me/roles`. Sem módulos de
  negócio, é o que prova que sessão, tenant e contrato estão ligados.

A troca de organização segue a regra não-negociável (cache limpo +
refresh):

```tsx
// src/features/organizations/components/organization-switcher.tsx (trecho)
async function switchTo(organizationId: string) {
  if (organizationId === activeOrganizationId) return;
  await authClient.organization.setActive({ organizationId });
  queryClient.clear(); // nada da organização anterior sobrevive em cache
  router.refresh();
}
```

### Autorização na UI

**Quem autoriza é o api-bun.** O front só **esconde ou desabilita** o que o
usuário não pode fazer, para não oferecer um botão que vai dar 403.

- A matriz de permissão (`user`, `editor`, `manager`, `admin` → ações) e
  `can(role, action)` vêm de `@repo/contracts/permissions`, a mesma que a
  API usa. Não existe cópia no web.
- A role do usuário em cada módulo vem de `GET /api/users/me/roles`
  (`myRolesResponseSchema` de `@repo/contracts/users`: `isOwner` e a role
  por módulo; o dono é `admin` em todos os módulos). Fica em
  `features/users/api/` (query + versão server). O front **nunca** lê `members.role` do plugin para
  decidir permissão, mesma regra da API.
- Um 403 da API é sempre tratado como possível (ver "### Erros da API"),
  mesmo quando a UI escondeu o botão.

## Variáveis de ambiente

Nunca ler `process.env` direto no código. Dois módulos, ambos com Zod:

```ts
// src/lib/env.schema.ts — só o schema, sem `server-only` (next.config.ts importa)
import { z } from 'zod';

export const serverEnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  API_URL: z.url('API_URL is required'), // api-bun, sem barra final
});
```

```ts
// src/lib/env.ts — só server
import 'server-only';
import { serverEnvSchema } from '@/lib/env.schema';

export const env = serverEnvSchema.parse(process.env);
```

```ts
// src/lib/env.client.ts — pode ir para o browser
import { z } from 'zod';

const schema = z.object({
  NEXT_PUBLIC_APP_URL: z.url(),
});

// NEXT_PUBLIC_* é inlined no build: precisa ser referenciado por nome literal
export const clientEnv = schema.parse({
  NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
});
```

> `next.config.ts` roda fora do bundle e não pode importar `server-only`
> (nem depender do alias `@`), por isso importa `env.schema.ts` por caminho
> relativo e faz o próprio `parse`. É o **único** lugar fora de `lib/env*.ts`
> que toca `process.env`.

Regras:

- `NEXT_PUBLIC_*` é **público**: vai no bundle do browser. Nunca segredo,
  token ou URL interna.
- Valor de `NEXT_PUBLIC_*` é fixado no **build**. Mudar na Vercel exige
  novo deploy.
- Arquivos: `.env.example` (commitado, todas as chaves sem valor real) e
  `.env.local` (não commitado). O `.gitignore` já ignora `.env*`; o
  scaffold acrescenta `!.env.example`.

| Variável | Onde | Uso |
|---|---|---|
| `API_URL` | server | URL do api-bun (`http://localhost:3333` local) |
| `NEXT_PUBLIC_APP_URL` | client | URL pública do front (links absolutos, redirects de auth) |

## Tratamento de erros na UI

- `error.tsx` por route group (`(app)/error.tsx`, `(auth)/error.tsx`) e
  `global-error.tsx` na raiz. Todos Client Components, com botão de tentar
  de novo.
- `not-found.tsx` na raiz e onde o módulo precisar de 404 próprio. Chamado
  via `notFound()` quando a API responde 404.
- Erro de mutação no client: toast (`sonner`, do shadcn) com a `message`
  da `ApiError`.
- Nunca `console.log` em código de produção (ver `docs/observability.md`).

## UI: Tailwind v4 + shadcn/ui

- Tailwind **v4**: config em CSS (`@import "tailwindcss"`, `@theme` em
  `globals.css`), sem `tailwind.config.js`.
- shadcn/ui instalado pela CLI, com `components.json` commitado. O `init`
  do shadcn 4 pergunta base e preset e trava sem TTY; a baseline fixa
  **base Radix, preset Nova**: `bunx shadcn@latest init -b radix -p nova
  --no-monorepo -y` (o `components.json` sai com `"style": "radix-nova"`).
- O shadcn 4 troca `clsx` + `tailwind-merge` pelo pacote `cn`
  (`lib/utils.ts` é só `export { cn } from 'cn'`) e passa a ser
  dependência de runtime (`@import "shadcn/tailwind.css"` no
  `globals.css`). Não reinstalar `clsx`/`tailwind-merge`.
- `shadcn add` pode gerar código que o Biome recommended acusa (ex:
  `field.tsx`). O `biome.json` da raiz tem um override para
  `src/components/ui/**` (`../../docs/conventions.md`, "## Biome"). Os componentes vivem em
  `src/components/ui/` e **podem** ser editados. Não são dependência
  opaca.
- Cor, raio, fonte: só via tokens CSS do tema (`bg-background`,
  `text-muted-foreground`, …), nunca hex solto em classe.
- Dark mode por classe (`.dark`), com `next-themes` se a instância oferecer
  alternância manual.

## Decisões registradas

Formato: **Decisão** / **Contexto** / **Alternativas consideradas**.

### pnpm como único package manager

> **Substituída no monorepo** por "### Bun como package manager do
> monorepo inteiro" (`../../docs/architecture.md`). Mantida abaixo como
> histórico da baseline app-nextjs.

**Decisão**: pnpm, versão fixada em `packageManager` do `package.json` e
ativada via Corepack. Só existe `pnpm-lock.yaml`.
**Contexto**: instalação rápida, `node_modules` estrito (dependência não
declarada quebra em vez de funcionar por acaso) e suporte nativo na Vercel.
O `create-next-app` gerou `package-lock.json`, que é removido no scaffold.
**Alternativas**: npm (lento, hoisting frouxo); Bun (usado no api-bun,
mas o runtime aqui é Node na Vercel e Bun como só package manager traz
pouco ganho e mais uma variável).

### Deploy na Vercel

**Decisão**: Vercel, pela integração Git (preview por PR, produção em
`main`). Sem Dockerfile no front.
**Contexto**: é a plataforma de referência do Next; ISR, imagens, proxy e
streaming funcionam sem configuração. O api-bun continua em container,
separado.
**Alternativas**: container próprio com `output: 'standalone'` (mais
controle, mas assume operação de CDN, cache e imagens que a Vercel já
resolve).

### Mesma origem via rewrite, não CORS

**Decisão**: o browser chama `/api/*` na origem do front; `rewrites()`
entrega ao api-bun. No server, chamada direta a `API_URL`.
**Contexto**: cookie de sessão first-party, sem `SameSite=None`, sem
depender de subdomínio compartilhado e sem CORS pré-flight em toda
mutação.
**Alternativas**: chamar a API em outra origem com CORS + cookies
cross-site (frágil com bloqueio de cookie de terceiros); `crossSubDomainCookies`
do Better Auth (exige front e API no mesmo domínio-pai, o que não vale para
preview da Vercel); BFF com route handlers reimplementando cada rota
(duplicação).

### ky como cliente HTTP

**Decisão**: ky, em duas instâncias (`lib/api/client.ts` e
`lib/api/server.ts`).
**Contexto**: API sobre `fetch` (funciona igual em RSC, server actions e
browser), `prefixUrl`, `hooks` para padronizar erro, retry e
`searchParams` tipados, com bundle pequeno.
**Alternativas**: `fetch` puro (cada chamada repete tratamento de erro e
serialização); axios (XHR no browser, fora do modelo de `fetch` do Next);
cliente gerado do OpenAPI do api-bun (vale reavaliar quando o contrato
estabilizar, mas o `/openapi` do api-bun fica desligado em produção).

### TanStack Query para estado de servidor no client

**Decisão**: TanStack Query para toda leitura/escrita da API no client;
RSC para a leitura inicial com prefetch + hidratação.
**Contexto**: cache, dedupe, invalidação e estados de loading/erro sem
reimplementar; chave com `organizationId` dá isolamento de tenant no cache.
**Alternativas**: só RSC + Server Actions (sem cache client para interação
rica); SWR (menos recursos de mutação e invalidação).

### nuqs para estado em URL

**Decisão**: todo estado que representa "o que estou vendo" (filtro,
página, aba, busca) vai para a URL via nuqs.
**Contexto**: link compartilhável, refresh sem perder estado,
voltar/avançar coerentes, e os mesmos parsers tipados no client e no
server.
**Alternativas**: `useSearchParams` + `router.push` à mão (sem tipo, com
parse espalhado); guardar em `useState`/zustand (perde no refresh).

### zustand só como exceção para estado de UI

**Decisão**: zustand entra na stack, mas é o **último** degrau da
hierarquia de estado (ver "## Estado") e cada uso é justificado na PR.
**Contexto**: React (estado local, URL, context) resolve a maioria dos
casos; zustand resolve bem o caso específico de estado de UI
compartilhado e de alta frequência. Liberar como padrão vira store global
com dado de servidor duplicado.
**Alternativas**: só React (casos pontuais ficam com prop drilling ou
context re-renderizando demais); Redux Toolkit (cerimônia maior para o
mesmo problema); Jotai (bom, mas duas libs de estado seria demais).

### React Hook Form + Zod

**Decisão**: RHF + `@hookform/resolvers/zod`, com componentes `Form` do
shadcn.
**Contexto**: input não controlado (desempenho), integração pronta com
shadcn e o mesmo Zod usado no api-bun e no parse de resposta.
**Alternativas**: `useActionState` + Server Actions puros (bom para forms
simples, fraco para validação client em tempo real e campos dinâmicos);
TanStack Form.

### shadcn/ui + Tailwind v4

**Decisão**: shadcn/ui (código copiado para `components/ui`) sobre Tailwind
v4.
**Contexto**: componentes acessíveis (Radix), totalmente editáveis, sem
lock-in de lib de componentes; Tailwind já vem do `create-next-app`.
**Alternativas**: MUI/Chakra/Mantine (dependência opaca, tema próprio,
brigam com Tailwind).

### Biome em vez de ESLint/Prettier

**Decisão**: Biome v2 com os domínios `next` e `react` (`biome.json`, já no
scaffold do `create-next-app`).
**Contexto**: mesma ferramenta do api-bun; o Next 16 removeu `next lint`,
então não há integração nativa a perder.
**Alternativas**: ESLint flat config + `eslint-config-next` + Prettier
(duas ferramentas, mais lento).

### React Compiler ligado

**Decisão**: `reactCompiler: true` (vem do `create-next-app`).
**Contexto**: memoização automática; código sem `useMemo`/`useCallback`
defensivos.
**Alternativas**: desligado, com memoização manual.

### Cache Components desligado

**Decisão**: `cacheComponents` desligado; nenhum `'use cache'` em dado
autenticado.
**Contexto**: o app é quase todo dado por sessão/tenant. Cache
compartilhado nesse contexto é risco de vazamento entre organizações, e o
ganho é pequeno porque a API já cacheia no Redis com chave por tenant.
**Alternativas**: ligar e marcar dinâmico o que é por usuário (fácil de
errar em silêncio). Reavaliar se a instância tiver área pública grande.

### `src/` + features por módulo

**Decisão**: código em `src/`; `app/` só roteia, `features/<modulo>/`
concentra o código do módulo.
**Contexto**: espelha os módulos do api-bun (mesmos nomes), separa
roteamento de implementação e evita que `app/` vire uma árvore com
componentes misturados a rotas.
**Alternativas**: tudo colocado em `app/` junto das rotas (mistura
roteamento com implementação); pastas por tipo (`components/`, `hooks/`
globais), que espalham um módulo pela árvore inteira.

## Diferenças no monorepo

- **Estrutura**: este app é o workspace `@repo/web` em `apps/web`. A
  árvore de `src/` acima vale como está, exceto `lib/permissions.ts`, que
  não existe (vem de `@repo/contracts/permissions`).
- **Contrato compartilhado**: schemas de resposta, corpo de erro,
  permissões e enums vêm de `@repo/contracts`
  (`../../docs/architecture.md`, "### `packages/contracts`"). Os schemas
  em `features/<modulo>/schemas/` são **de formulário** e podem derivar
  dos do contrato (`.pick`, `.extend`). A validação de resposta em
  `features/<modulo>/api/` usa o schema do contrato direto.
- **`next.config.ts`** completo da baseline (acrescenta
  `transpilePackages` e o loader do Tailwind ao de "### Mesma origem via
  rewrite"):

  ```ts
  // apps/web/next.config.ts
  import type { NextConfig } from 'next'
  import { serverEnvSchema } from './src/lib/env.schema'

  const env = serverEnvSchema.parse(process.env)

  const nextConfig: NextConfig = {
  	reactCompiler: true,
  	// cacheComponents fica desligado: dado por sessão/tenant
  	transpilePackages: ['@repo/contracts'],
  	turbopack: {
  		// Tailwind v4 via Turbopack, sem PostCSS (como o create-next-app 16.4 gera)
  		rules: { '*.css': { loaders: ['@tailwindcss/turbopack'], as: '*.css' } },
  	},
  	async rewrites() {
  		return [{ source: '/api/:path*', destination: `${env.API_URL}/api/:path*` }]
  	},
  }

  export default nextConfig
  ```

- **Depois do `create-next-app`** (gerado fora e copiado, ver
  `../../docs/development.md`, "## Geradores sem TTY"), remover o que a
  versão 16.4 traz a mais e que conflita com a baseline:
  - `cacheComponents: true` e `partialPrefetching: true` do
    `next.config.ts` (ver "### Cache Components desligado");
  - `biome.json` próprio (2 espaços): trocar pelo que estende a raiz
    (`../../docs/conventions.md`, "## Biome");
  - `.gitignore` próprio (o da raiz cobre, com `*.tsbuildinfo` e
    `next-env.d.ts`);
  - `@biomejs/biome` e `typescript` do `package.json` do app (versão
    única, na raiz);
  - `package-lock.json`/`bun.lock` gerados no app, se houver.
  - Manter o `turbopack.rules` do `@tailwindcss/turbopack` e o
    `AGENTS.md`.
- **Env**: arquivos em `apps/web/` (`.env.local`, `.env.example`).
  `API_URL` e `NEXT_PUBLIC_*` são declaradas em `env` da task `build` no
  `turbo.json`, sem o que o turbo não as repassa ao `next build`.
- **Previews**: `API_URL` de Preview é a API de staging do monorepo
  (`../../docs/deploy.md`).
- **Decisão "pnpm como único package manager"**: substituída (ver a nota
  na própria decisão).
- **Sessão**: `sessionSchema` vem de `@repo/contracts/session`.

### Correções à frente do app-nextjs v0.1.0

Achados do primeiro scaffold (turborepo-template 0.3.0) corrigidos aqui e
ainda **não** levados ao app-nextjs. Na próxima sincronização, não
sobrescrever estes trechos sem conferir se o app-nextjs já os incorporou
(lista também em `../../docs/CHANGELOG.md`):

- ky 2 (`baseUrl`, `beforeError({ error })`, `error.data`).
- shadcn 4 (`-b radix -p nova`, pacote `cn`, `Field` + `Controller`).
- O que remover depois do `create-next-app` 16.4.
- Sessão validada com schema Zod; rotas e shell de base.
- Setup do Vitest com MSW 3, shim de `Request` e mock de
  `next/navigation` (`docs/testing.md`).
