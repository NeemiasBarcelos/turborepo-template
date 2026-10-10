# Testes

> Baseado em app-nextjs v0.1.0, adaptado ao monorepo. Onde este arquivo
> conflita com a raiz, vale a raiz (`../../CLAUDE.md`) e as seções
> "## Diferenças no monorepo" de `CLAUDE.md` e `docs/architecture.md`.

## Filosofia

O front não tem regra de negócio: ela está no api-bun, que tem seus
próprios testes. Aqui o teste garante que **a interface faz o que promete
com o contrato da API**: renderiza o estado certo, envia o payload certo,
trata os erros documentados e não mistura dado entre organizações.

Dois níveis:

- **Vitest + Testing Library** (jsdom): componentes, hooks, schemas,
  parsers e helpers. A API é mockada com **MSW**, nunca com mock manual do
  ky.
- **Playwright**: fluxos de ponta a ponta no app rodando (`next build &&
  next start`), contra o api-bun real ou contra MSW (ver "## E2E").

Testar comportamento visível ao usuário (texto, role, label), não detalhe
de implementação (estado interno, nome de classe, chamada de hook).

## Obrigatório vs. opcional

**Obrigatório** em toda feature nova:

- Schema Zod do formulário: casos válidos e inválidos relevantes.
- Componente de formulário: envia o payload esperado; mostra erro de
  campo vindo de um 400 com `issues`; desabilita submit enquanto envia.
- Componente de listagem: estado vazio, estado com dados e estado de erro.
- Componente condicionado a permissão: esconde ou desabilita a ação para
  a role sem permissão.
- Parsers nuqs com default ou parser customizado: valor ausente, válido e
  inválido na URL.
- Qualquer helper em `lib/` (`can()`, `toApiError`, `applyApiIssues`, …).

**Obrigatório por fluxo crítico** (Playwright), além dos da baseline:

- Login e logout.
- Troca de organização ativa **limpa os dados da anterior** (ver "##
  Isolamento entre tenants").
- Rota autenticada sem sessão redireciona para `/sign-in`.

**No scaffold** (sem módulo de negócio), só os fluxos de auth são
obrigatórios: rota protegida → `/sign-in?next=`, cadastro → logout → login
(com a organização ativa visível) e senha errada. O teste de troca de
organização exige dado de negócio e um usuário em duas organizações, então
entra **com a primeira feature de negócio**.

**Opcional**: componentes puramente visuais sem lógica, wrappers finos de
componente do shadcn.

## Vitest

Setup conforme o guia de Vitest do Next (`node_modules/next/dist/docs/01-app/02-guides/testing/vitest.md`):
`vitest`, `@vitejs/plugin-react`, `jsdom`, `@testing-library/react`,
`@testing-library/dom`, `@testing-library/user-event`,
`@testing-library/jest-dom`, `vite-tsconfig-paths` (alias `@/`) e `msw`.

```ts
// vitest.config.mts
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: {
    environment: 'jsdom',
    // URL da página no jsdom: base para caminhos relativos como /api/
    environmentOptions: { jsdom: { url: 'http://localhost:3000' } },
    setupFiles: ['./tests/setup.ts', './tests/next-navigation.ts'],
    include: ['src/**/*.test.{ts,tsx}'],
    env: { NEXT_PUBLIC_APP_URL: 'http://localhost:3000' },
  },
});
```

```ts
// tests/setup.ts
import '@testing-library/jest-dom/vitest';
import { cleanup } from '@testing-library/react';
import { afterAll, afterEach } from 'vitest';
import { server } from './mocks/server';

// no browser, `new Request('/api/')` resolve contra a página; o Request do Node (que o
// jsdom do Vitest mantém) não. O ky resolve `baseUrl` assim, então o teste imita o browser.
const NodeRequest = globalThis.Request;
globalThis.Request = class extends NodeRequest {
  constructor(input: RequestInfo | URL, init?: RequestInit) {
    super(
      typeof input === 'string' && input.startsWith('/')
        ? new URL(input, window.location.origin)
        : input,
      init,
    );
  }
};

// no topo, não em beforeAll: módulos que guardam o `fetch` ao carregar
// (o client do Better Auth) já pegam a versão interceptada
server.listen({ onUnhandledFrame: 'error' });
afterEach(() => {
  cleanup(); // sem `globals: true`, o Testing Library não limpa sozinho
  server.resetHandlers();
});
afterAll(() => server.close());
```

```ts
// tests/next-navigation.ts — componentes client usam useRouter fora do Next
import { vi } from 'vitest';

export const router = { push: vi.fn(), refresh: vi.fn(), replace: vi.fn(), back: vi.fn() };

vi.mock('next/navigation', () => ({
  useRouter: () => router,
  usePathname: () => '/',
  useSearchParams: () => new URLSearchParams(),
}));
```

O teste que precisa conferir navegação importa `router` de
`tests/next-navigation` e checa `router.push`.

Pontos que o setup resolve (todos validados com Vitest 5, MSW 3 e jsdom 30):

- **`onUnhandledFrame: 'error'`** é intencional: request sem handler é bug
  de teste, nunca chamada real à rede. No MSW 3, a opção antiga
  `onUnhandledRequest` é **ignorada em silêncio** em runtime (o padrão
  volta a ser `warn`), então um teste poderia bater na rede sem falhar.
- **`server.listen` no topo do arquivo**: em `beforeAll`, o client do
  Better Auth (que captura o `fetch` quando o módulo carrega) escapa do
  MSW e tenta `localhost:3000` (`ECONNREFUSED`).
- **Shim do `Request`**: sem ele, todo teste que passa por
  `lib/api/client.ts` falha com `Failed to parse URL from /api/`.
- **`cleanup()`** no `afterEach` e o **mock de `next/navigation`** como
  setup file.

**Limitação**: o Vitest não renderiza **Server Components `async`**. Eles
são cobertos pelo E2E; a lógica que dá para extrair (parse, mapeamento)
vira função pura testada no Vitest.

### Helper de render

`tests/render.tsx` exporta um `renderWithProviders` que envolve o
componente em `QueryClientProvider` (um `QueryClient` **novo por teste**,
com `retry: false`) e no adapter de teste do nuqs
(`nuqs/adapters/testing`), aceitando a query string inicial.

## Mocks da API (MSW)

- Handlers em `tests/mocks/handlers/<modulo>.ts`, respondendo no formato
  real do api-bun, **inclusive o formato de erro**
  (`{ statusCode, error, message, issues? }`).
- Fixtures geradas a partir dos schemas Zod do front (factory por
  entidade), para o mock não divergir do tipo usado pelo componente.
- Cenário de erro é sobrescrito no próprio teste com `server.use(...)`.

## Isolamento entre tenants

A garantia de isolamento é do api-bun. O front testa o que depende dele:

- **Vitest**: toda fábrica de query key do módulo começa pelo
  `organizationId` (teste da própria `taskKeys`).
- **Playwright**: usuário membro de duas organizações vê a lista da A,
  troca para a B e **não** vê nenhum item da A, nem por um frame, nem
  voltando com o botão "voltar" do browser.

## E2E (Playwright)

Setup conforme `node_modules/next/dist/docs/01-app/02-guides/testing/playwright.md`.
Specs em `e2e/`, `playwright.config.ts` com `webServer` rodando
`bun run build && bun run start` (E2E contra build de produção, não contra
`next dev`).

Contra o quê roda:

- **Local e CI**: api-bun real, subido via Docker Compose do próprio
  api-bun, com banco de **teste** (as guardas do api-bun garantem que não
  é o banco de dev). Usuários e organizações de teste criados por um
  `globalSetup` via API.
- Se a instância ainda não tem api-bun disponível no CI, os fluxos críticos
  rodam com MSW no Node (via `instrumentation.ts` condicionado a
  `NODE_ENV=test`) e a PR registra isso como débito.

Autenticação: login uma vez no `globalSetup` por usuário de teste, salvo em
`storageState`, e reaproveitado pelas specs. Não fazer login pela UI em
todo teste. Exceção: as specs **do próprio fluxo de auth** (cadastro,
login, logout) fazem tudo pela UI, porque é isso que testam; no scaffold,
que só tem essas specs, o `globalSetup` ainda não é necessário.

Seletores: `getByRole`, `getByLabel`, `getByText`. `data-testid` só
quando não houver alternativa acessível.

## Cobertura

Sem meta numérica na baseline. Métrica de cobertura não mede se o teste
certo existe; a lista de "Obrigatório" acima e o checklist de PR
(`../../docs/checklists.md`) é que medem. `bun run test --coverage` fica disponível
para inspeção.

## Diferenças no monorepo

- **API do E2E**: é o `apps/api` do mesmo repositório, não um api-bun
  subido de outro repo. O `playwright.config.ts` declara dois
  `webServer`:

  ```ts
  // apps/web/playwright.config.ts (trecho)
  webServer: [
  	{
  		// API em modo teste: carrega apps/api/.env.test (banco *_test, Redis índice ≠ 0)
  		command: 'bun --env-file .env.test src/index.ts',
  		cwd: '../api',
  		url: 'http://localhost:3333/health',
  		reuseExistingServer: !process.env.CI,
  	},
  	{
  		command: 'bun run build && bun run start',
  		url: 'http://localhost:3000',
  		reuseExistingServer: !process.env.CI,
  	},
  ],
  ```

- **Banco**: local, o `app_test` do Postgres do Docker (migrado com
  `NODE_ENV=test bun run db:migrate` em `apps/api`). No CI, o banco
  `app_e2e_test` dentro do branch Neon da PR, com Redis no índice `2`.
  As variáveis exportadas pelo job vencem o `.env.test`
  (`../../docs/ci-cd.md`, job `e2e`).
- **Mocks MSW**: continuam valendo para testes de componente. O formato
  deles vem dos schemas de `@repo/contracts`, e um mock fora do contrato
  falha no `typecheck`.
- **Fallback "sem api-bun no CI"**: não se aplica. A API está sempre
  disponível no monorepo.
