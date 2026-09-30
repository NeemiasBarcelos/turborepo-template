# Arquitetura

> Baseado em api-bun v0.16.0, adaptado ao monorepo. Onde este arquivo
> conflita com a raiz, vale a raiz (`../../CLAUDE.md`) e as seções
> "## Diferenças no monorepo" de `CLAUDE.md` e `docs/architecture.md`.

## Estrutura: Modules + Features

```
src/
  modules/
    <module-name>/
      features/
        <action-name>/
          <action-name>.ts           # feature: schema + handler
          <action-name>.service.ts   # regra de negócio + acesso ao banco
      <module-name>.routes.ts        # instância Elysia que agrega as features do módulo
  db/
    schema/      # definição das tabelas Drizzle
    migrations/  # migrations geradas (não editar à mão)
  plugins/       # plugins Elysia (auth, error handler, openapi, etc)
  lib/           # utilitários puros, sem dependência de framework
  types/         # tipos compartilhados
```

Exemplo real (módulo `tasks`) — uma pasta por feature, como no diagrama
acima:

```
modules/tasks/features/create-task/create-task.ts
modules/tasks/features/create-task/create-task.service.ts
modules/tasks/features/update-task/update-task.ts
modules/tasks/features/update-task/update-task.service.ts
modules/tasks/features/delete-task/delete-task.ts
modules/tasks/features/delete-task/delete-task.service.ts
modules/tasks/tasks.routes.ts
```

Cada feature é um par de arquivos:

**`<action>.ts`** — feature (camada de borda, agnóstica de framework)
- Define `inputSchema` e `outputSchema` (Zod).
- Recebe o input e o `ctx` (`TenantContext`: `userId`, `organizationId`,
  `isOwner` — ver "### Multi-tenancy"), chama o service, devolve o output.
- Não trata exception aqui — exception sobe e é tratada na raiz (`.onError`
  do Elysia).
- Não conhece Elysia — não importa `Elysia`, não recebe `context`. Isso
  mantém a feature testável sem subir servidor e reutilizável se o
  framework HTTP mudar algum dia.

**`<action>.service.ts`** — regra de negócio
- Recebe o input já validado pela feature e o mesmo `ctx` — todo acesso a
  tabela de negócio é filtrado pelo tenant do `ctx` (`inTenant`).
- Aplica as regras de negócio e transforma dados.
- Fala diretamente com o banco (Drizzle) — **não existe camada de repository**
  neste padrão.
- Devolve o output.

**`<module-name>.routes.ts`** — wiring HTTP (única camada que conhece Elysia)
- Uma instância `Elysia` por módulo, com `prefix` do módulo.
- Registra cada feature como rota, passando `inputSchema`/`outputSchema`
  direto como `body`/`response` (Elysia aceita Zod nativamente via Standard
  Schema — sem adapter).
- Rota de negócio marca `tenant: true` (macro de `plugins/auth.ts`, ver
  "## Autenticação e autorização"): exige sessão, organização ativa e
  membership, e expõe `tenant` (o `TenantContext`). A rota repassa esse
  `ctx` para a feature — a feature recebe dado puro, nunca o `context` do
  Elysia. `auth: true` (só sessão) fica para o que não é de um tenant:
  listar, criar e trocar de organização.

```ts
// modules/tasks/tasks.routes.ts
import { Elysia } from 'elysia';
import { authPlugin } from '@/plugins/auth';
import {
  createTask,
  inputSchema as createTaskInput,
  outputSchema as createTaskOutput,
} from './features/create-task/create-task';
import { deleteTask } from './features/delete-task/delete-task';

export const tasksRoutes = new Elysia({ prefix: '/tasks' })
  .use(authPlugin)
  .post('/', ({ body, tenant }) => createTask(body, tenant), {
    tenant: true,
    body: createTaskInput,
    response: createTaskOutput,
  })
  .delete('/:id', ({ params, tenant }) => deleteTask(params.id, tenant), {
    tenant: true,
  });
```

```ts
// index.ts
import { Elysia } from 'elysia';
import { env } from '@/lib/env';
import { tasksRoutes } from '@/modules/tasks/tasks.routes';

new Elysia().use(tasksRoutes).listen(env.PORT);
```

Os snippets desta documentação são ilustrativos: não seguem à risca o
formatter do Biome (ver `docs/conventions.md`, "Lint e formatação").

Fluxo: `<module>.routes.ts (wiring HTTP) → feature (input/output schema) → service (regra + banco) → output | exception sobe para o .onError na raiz`

## Por que essa separação

- **Coesão por feature**: tudo que uma ação precisa (schema, handler, regra,
  query) fica em dois arquivos próximos, fácil de achar e de deletar quando a
  feature morre.
- **Sem indireção desnecessária**: sem repository genérico no meio — o service
  já é o dono da query, evita camada que só repassa chamada sem agregar valor.
- **Testabilidade**: o service é testável isoladamente (banco de teste
  limpo entre testes, ver `docs/testing.md`), sem precisar subir o
  servidor HTTP.

> **Skill opcional**: [`improve-codebase-architecture`](https://github.com/mattpocock/skills/blob/main/skills/engineering/improve-codebase-architecture/SKILL.md)
> do pacote [mattpocock/skills](https://github.com/mattpocock/skills) —
> checagem periódica opcional de que o padrão modules+features sem
> repository não está degradando (módulo ficando raso, indireção
> voltando). Não é parte da baseline do template, é uma sugestão de
> ferramenta.

## Banco de dados

- PostgreSQL como banco principal.
- Drizzle ORM para schema e queries — schema é a fonte de verdade.
- Driver de conexão: `postgres.js` (pacote `postgres`), via
  `drizzle-orm/postgres-js`. Não usar `drizzle-orm/bun-sql` — apesar de
  nativo do Bun e sem dependência extra, a doc do Drizzle registra bug
  conhecido de execução de statements concorrentes no driver Bun SQL (a
  partir do Bun 1.2.0), risco real numa API que roda queries em paralelo.
  `postgres.js` é o driver mais maduro e battle-tested com Drizzle +
  Postgres:

```ts
// lib/db.ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import { env } from '@/lib/env';
import * as schema from '@/db/schema';

export const client = postgres(env.DATABASE_URL, {
  max: 10, // teto de conexões deste processo — ver "### Neon"
  prepare: false, // o pooler do Neon (PgBouncer, modo transação) não suporta PREPARE
  idle_timeout: 20, // fecha conexão ociosa (segundos) antes que o pooler o faça
  connect_timeout: 15, // tolera o cold start do scale-to-zero (segundos)
});
export const db = drizzle(client, { schema });
```

`client` é exportado porque o shutdown gracioso precisa chamar
`client.end()` (ver "## Shutdown gracioso").
- Migrations sempre via `drizzle-kit generate` + `migrate`, nunca `push` fora
  de ambiente local — isso inclui staging, CI e qualquer branch de banco
  hospedado.
- Toda tabela tem `created_at` e `updated_at`, sempre `timestamp` com
  timezone:

```ts
createdAt: timestamp('created_at', { withTimezone: true })
  .defaultNow()
  .notNull(),
updatedAt: timestamp('updated_at', { withTimezone: true })
  .defaultNow()
  .$onUpdate(() => new Date())
  .notNull(),
```
- Soft delete (`deletedAt`) não é regra fixa da baseline — é decisão por
  feature (ver `../../docs/checklists.md`, "Nova feature"). O modelo de
  permissão segue CRUD tradicional (`read`/`create`/`update`/`delete`,
  ver "## Autenticação e autorização") — não existe ação `revert`
  separada. Restaurar um registro com soft delete é uma operação de
  `update` normal (zera `deletedAt`), autorizada pela mesma ação
  `update` de sempre, não uma permissão própria. Fazer isso sem deixar
  rastro no histórico da entidade apaga silenciosamente a evidência de
  que a exclusão aconteceu — se a entidade não tem histórico, ou a
  exclusão precisa ser definitiva (compliance, regra de negócio), use
  hard delete.
- Primary key sempre `uuid`, nunca `serial`/incremental:

```ts
id: uuid('id').primaryKey().defaultRandom(),
```

- Coluna com valores fixos conhecidos (status, role, etc.) sempre via
  `pgEnum` — tipo nativo do Postgres, com constraint real no banco.
  Nunca `text`/`varchar` com a opção `{ enum: [...] }`: essa opção só
  restringe o tipo TypeScript de insert/select, não valida nada em
  runtime nem no banco — um valor fora da lista passa direto num
  insert cru, seed, ou numa migration futura, sem erro nenhum.

```ts
export const moduleRole = pgEnum('module_role', ['user', 'editor', 'manager', 'admin']);

export const userModuleRoles = pgTable('user_module_roles', {
  // ...
  role: moduleRole().notNull(),
});
```

Reaproveitar a lista de valores em código, nunca duplicando o array à
mão — ex: `outputSchema` (Zod) de uma feature que expõe essa coluna:

```ts
// list-tasks.ts
import { z } from 'zod/v4';
import { moduleRole } from '@/db/schema/roles';

export const outputSchema = z.object({
  // ...
  role: z.enum(moduleRole.enumValues),
});
```

Nunca declarar um `as const` array separado (`['user', 'editor',
'manager']` de novo) só pra alimentar o Zod — isso duplica a fonte de
verdade e os dois podem divergir com o tempo.

- Priorizar **relations** (`relations()` no schema + `db.query.x.findMany`)
  em vez de `leftJoin`/`rightJoin` manual. Drizzle cobre join manual também,
  mas pra quem lê o código, relations deixam a intenção clara sem precisar
  entender a query relacional por trás:

```ts
// ✅ preferir — relação declarada no schema, query legível
export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

const result = await db.query.users.findMany({
  with: { posts: true },
});
```

```ts
// ❌ evitar quando dá pra usar relation — join manual é mais difícil de ler
const result = await db
  .select()
  .from(users)
  .leftJoin(posts, eq(posts.userId, users.id));
```

Join manual continua válido para casos que `relations` não cobre bem (ex:
agregações complexas, `groupBy`) — a regra é preferir relations, não proibir
join.

### Multi-tenancy: isolamento por `organization_id`

A baseline é **somente multi-tenant**: não existe modo single-tenant nem
tenant opcional. **Tenant = organização** (plugin `organization` do Better
Auth, ver "## Autenticação e autorização"). Banco único e schema único —
o isolamento é por linha, com `organization_id`:

- **Toda tabela de negócio** tem `organization_id uuid NOT NULL`, FK para
  `organizations(id)` com `on delete cascade` — via o mixin `tenantColumns`.
  Só ficam de fora as tabelas do próprio Better Auth: `users`, `sessions`,
  `accounts` e `verifications` são globais (um usuário existe uma vez e
  pode pertencer a várias organizações), e `organizations`, `members` e
  `invitations` *são* o tenant.

```ts
// db/schema/tenant.ts
import { uuid } from 'drizzle-orm/pg-core';
import { organizations } from './auth';

export const tenantColumns = {
  organizationId: uuid('organization_id')
    .notNull()
    .references(() => organizations.id, { onDelete: 'cascade' }),
};
```

- **Todo acesso é escopado pelo `ctx`.** O helper `inTenant` monta o
  filtro obrigatório; consulta só por `id` é proibida, e todo `insert`
  grava `ctx.organizationId`:

```ts
// lib/tenant.ts
import { type AnyColumn, type SQL, and, eq } from 'drizzle-orm';

export type TenantContext = {
  userId: string;
  organizationId: string;
  /** É o dono da organização (`members.is_owner`)? O dono é `admin` em qualquer módulo. */
  isOwner: boolean;
};

/** Filtro obrigatório de tenant: `where: inTenant(tasks, ctx, eq(tasks.id, id))`. */
export function inTenant(
  table: { organizationId: AnyColumn },
  ctx: TenantContext,
  ...conditions: (SQL | undefined)[]
) {
  return and(eq(table.organizationId, ctx.organizationId), ...conditions);
}
```

```ts
// ✅ correto — sempre com o tenant
const task = await db.query.tasks.findFirst({
  where: inTenant(tasks, ctx, eq(tasks.id, id)),
});
await db.insert(tasks).values({ ...input, organizationId: ctx.organizationId });

// ❌ não fazer — por id sozinho, lê (ou apaga) a linha de qualquer tenant
const task = await db.query.tasks.findFirst({ where: eq(tasks.id, id) });
```

- **Índices e `unique` de negócio incluem `organization_id`** (e começam por
  ele nos índices de consulta): o título de uma task é único **por
  organização**, não no banco todo — senão um tenant descobre o dado do
  outro pelo erro de conflito.
- **FK entre tabelas de tenant é composta**, `(organization_id, <pai>_id)`,
  para que o banco impeça uma linha de apontar para o pai de **outro**
  tenant (a tabela pai declara `unique(organization_id, id)` como alvo):

```ts
export const projects = pgTable(
  'projects',
  { id: uuid('id').primaryKey().defaultRandom(), ...tenantColumns, name: text('name').notNull() },
  (t) => [
    unique('projects_org_id_uq').on(t.organizationId, t.id), // alvo da FK composta
    unique('projects_org_name_uq').on(t.organizationId, t.name),
  ],
);

export const tasks = pgTable(
  'tasks',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    ...tenantColumns,
    projectId: uuid('project_id').notNull(),
    title: text('title').notNull(),
  },
  (t) => [
    foreignKey({
      columns: [t.organizationId, t.projectId],
      foreignColumns: [projects.organizationId, projects.id],
    }).onDelete('cascade'),
    unique('tasks_org_title_uq').on(t.organizationId, t.title),
    index('tasks_org_project_idx').on(t.organizationId, t.projectId),
  ],
);
```

- **Recurso de outro tenant responde `404`, nunca `403`** — `403` confirmaria
  que o id existe. O service trata "não achei no meu tenant" e "não existe"
  do mesmo jeito (`NotFoundError`).
- **Limites assumidos** (ver "Decisões registradas", "Multi-tenant
  obrigatório"): não há segunda barreira no banco (sem RLS) — a segurança
  do isolamento depende do `inTenant` e dos testes de isolamento
  (`docs/testing.md`), que por isso são obrigatórios; o restore
  *point-in-time* do Neon é do banco inteiro, não de um tenant; e um tenant
  ruidoso divide compute com os demais. Excluir uma organização apaga seus
  dados em cascata.

### Neon (Postgres em produção)

O alvo de produção da baseline é **Postgres no [Neon](https://neon.com)**;
localmente é o Postgres do Docker e no CI um branch Neon por PR (`docs/docker.md`,
`docs/ci-cd.md`). O Neon expõe **duas connection strings** por branch, e a
baseline usa cada uma para uma coisa:

| Variável | Host | Uso |
|---|---|---|
| `DATABASE_URL` | com `-pooler` (PgBouncer, modo transação) | **runtime** da API (`lib/db.ts`) |
| `DATABASE_URL_UNPOOLED` | sem `-pooler` (conexão direta) | **migrations** e `drizzle-kit` (`db:migrate`, `db:studio`) |

- **Migrations nunca passam pelo pooler.** PgBouncer em modo transação não
  suporta recursos de sessão que ferramentas de migration usam; a própria
  doc do Neon manda usar a conexão direta para migrations.
- **`prepare: false` no `postgres.js`** quando a URL de runtime é a pooled:
  `PREPARE` de SQL não é suportado no endpoint pooled, e o driver usa
  prepared statements por padrão (a própria doc do `postgres.js` manda
  desligar para PgBouncer em modo transação). Versões recentes do
  PgBouncer suportam prepared statements de protocolo, mas
  `prepare: false` é a escolha conservadora, com custo baixo — reavaliar
  só com medição.
- **`sslmode=require`** nas duas URLs — o Neon só aceita TLS. **Sem
  `channel_binding`**: as URLs geradas pelo console do Neon trazem
  `&channel_binding=require` (parâmetro do libpq), e ele deve ser
  **removido** da URL antes de guardá-la em `DATABASE_URL` /
  `DATABASE_URL_UNPOOLED`. O `postgres.js` não implementa channel binding
  (só negocia `SCRAM-SHA-256`, sem `-PLUS`; há issue aberta pedindo
  suporte) e trata como opção de conexão só o que conhece: qualquer outro
  parâmetro da query string da URL vira **parâmetro de sessão enviado ao
  servidor** no startup. Ou seja, o driver mandaria `channel_binding=require`
  como se fosse uma configuração do Postgres, em vez de aplicar a proteção
  que o parâmetro promete. Remover não perde segurança — a proteção real
  aqui é o TLS de `sslmode=require`.

  Para conferir uma URL numa instância (com um branch/credencial
  descartável do Neon), um script avulso — **não commitar**:

  ```ts
  // check-db-url.ts
  import postgres from 'postgres';

  const sql = postgres(Bun.argv[2] ?? '', { prepare: false, max: 1, connect_timeout: 15 });
  console.log(await sql`select version(), current_database()`);
  await sql.end();
  ```

  `bun check-db-url.ts "<url>"`, uma vez com e outra sem
  `&channel_binding=require`.
- **Cold start**: com scale-to-zero, a primeira conexão depois de ociosidade
  leva algumas centenas de ms (o padrão do Neon suspende a compute após 5
  min sem atividade) — daí `connect_timeout` folgado. Por isso o `/health`
  (liveness) **não consulta o banco**: uma sonda a cada poucos segundos
  manteria a compute acordada, com custo, e um Neon suspenso derrubaria o
  container por restart. A checagem de banco fica em `/ready` (ver
  "## Health check").
- **`max`** é o teto **por processo**. Com o pooler o limite prático é
  folgado; sem pooler (URL direta), `réplicas × max` precisa caber no
  `max_connections` da compute.
- **Versão major do Postgres** do Docker local e do CI (hoje `postgres:16`)
  deve ser a mesma do projeto Neon — o Neon suporta várias majors; escolher
  na criação do projeto e manter as três alinhadas.

## Variáveis de ambiente

Nunca acessar `process.env` diretamente no código — sempre pelo helper
`env`, validado com Zod em `lib/env.ts`. Isso garante que a aplicação falha
rápido (na inicialização) se faltar variável ou vier em formato errado, em
vez de quebrar em runtime no meio de uma request.

Bun carrega `.env` automaticamente (sem precisar do pacote `dotenv`) — o
`env.ts` só precisa validar o que já está em `process.env`:

```ts
// lib/env.ts
import { z } from 'zod/v4';

const envSchema = z.object({
  NODE_ENV: z
    .enum(['development', 'production', 'test'])
    .default('development'),
  PORT: z.coerce.number().default(3333),
  DATABASE_URL: z.url('DATABASE_URL is required'),
  REDIS_URL: z.url('REDIS_URL is required'),
  BETTER_AUTH_SECRET: z.string().min(32, 'BETTER_AUTH_SECRET needs 32+ chars'),
  BETTER_AUTH_URL: z.url('BETTER_AUTH_URL is required'),
  // lista separada por vírgula; vira array (CORS + trustedOrigins do Better Auth)
  TRUSTED_ORIGINS: z
    .string()
    .default('')
    .transform((v) => v.split(',').map((s) => s.trim()).filter(Boolean)),
  CLIENT_IP_HEADER: z.string().optional(),
  OTEL_EXPORTER_OTLP_ENDPOINT: z.url().optional(),
});

const _env = envSchema.safeParse(process.env);

if (!_env.success) {
  console.error('Invalid environment variables:');
  console.error(z.prettifyError(_env.error));
  process.exit(1);
}

export const env = _env.data;
```

`lib/env.ts` (e `lib/env-tooling.ts`, ver "### CLI de ferramentas") são as
**únicas exceções** à regra "nunca `console.*`": rodam antes do `logger`
existir (o `logger` depende do `env`), então não há como logar de outra
forma. Usam `console.error` (stderr) e só imprimem as mensagens de
validação — nunca os valores das variáveis.

O `envSchema` acima é a lista completa das variáveis da **aplicação**; a
tabela abaixo documenta cada uma. `NODE_ENV` tem default `development`,
então o container de produção **precisa** definir `NODE_ENV=production`
(o `Dockerfile` faz isso, ver `docs/docker.md`) — sem isso, a API sobe em
produção com log `debug` e `pino-pretty`.

### Variáveis exigidas pela baseline

| Variável                      | Obrigatória | Default          | Usada em                                                          |
| ----------------------------- | ----------- | ---------------- | ----------------------------------------------------------------- |
| `NODE_ENV`                    | não         | `development`    | logger, carregamento de `.env.*`, OpenAPI, cookies seguros        |
| `PORT`                        | não         | `3333`           | `.listen(env.PORT)`                                               |
| `DATABASE_URL`                | sim         | —                | `lib/db.ts` (Neon: URL **pooled**, com `-pooler`, sem `channel_binding`) |
| `DATABASE_URL_UNPOOLED`       | só em prod  | —                | `db:migrate`/`db:studio` via `lib/env-tooling.ts` (URL direta, sem `channel_binding`) |
| `REDIS_URL`                   | sim         | —                | `lib/redis.ts` (cache, rate limit, sessão); `rediss://` em prod   |
| `BETTER_AUTH_SECRET`          | sim         | —                | Better Auth (mínimo 32 caracteres)                                |
| `BETTER_AUTH_URL`             | sim         | —                | Better Auth (`https://` em produção)                              |
| `TRUSTED_ORIGINS`             | não         | vazio            | CORS + `trustedOrigins` do Better Auth (mesma lista nos dois)     |
| `CLIENT_IP_HEADER`            | não         | — (IP do socket) | rate limit atrás de proxy (ver "## Produção: proxy, CORS e limites") |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | não         | — (desligado)    | telemetria (`docs/observability.md`)                              |

`DATABASE_URL_UNPOOLED` **não** entra no `envSchema` da aplicação — só o
tooling de CLI a lê (ver "### CLI de ferramentas"). Em desenvolvimento e
no CI, onde não há pooler, ela é opcional e o tooling cai em
`DATABASE_URL`.

Toda variável da tabela precisa estar no `.env.example` da instância (sem
valor real). O job `test` do CI (`docs/ci-cd.md`) fixa as obrigatórias
como `env:` do job, porque `lib/env.ts` falha na inicialização sem elas.

### Arquivos .env

Projeto é trabalhado sozinho (sem time), o que simplifica a estratégia.
Bun já carrega esses arquivos automaticamente por convenção — sem precisar
apontar caminho manualmente:

- **`.env.local`** — desenvolvimento do dia a dia. Valores reais de ambiente
  local (banco local, etc). Bun carrega com prioridade mais alta (exceto
  quando `NODE_ENV=test`). **Gitignored**, nunca commitado.
- **`.env.test`** — usado pelos testes (Bun carrega quando `NODE_ENV=test`).
  Pode ser **commitado**, desde que os valores sejam fake/isolados (banco de
  teste dedicado, chaves dummy) — evita ter que reconfigurar ambiente de
  teste em outra máquina.

  Exemplo — banco e Redis **isolados** dos de desenvolvimento (o Redis
  usa o índice de banco `/1`, porque o teste roda `flushdb()`; o
  `RedisClient` do Bun aceita o índice na URL):

  ```
  NODE_ENV=test
  DATABASE_URL=postgres://postgres:postgres@localhost:5432/app_test
  REDIS_URL=redis://localhost:6379/1
  BETTER_AUTH_SECRET=test-secret-not-for-production-0123456789
  BETTER_AUTH_URL=http://localhost:3333
  ```

- **`.env.example`** — documenta todas as vars existentes, sem valor real.
  Não é carregado automaticamente (não segue a convenção que o Bun
  reconhece), serve só de referência. **Commitado**.
- **Produção** — nunca em arquivo dentro do repositório. Variáveis
  injetadas diretamente pela plataforma de deploy (env vars do serviço,
  secrets do Docker/provider). Se por algum motivo existir um `.env.production`,
  ele vive só no servidor e fica no `.gitignore` — nunca é commitado, para
  não vazar secret por engano.

`.gitignore` deve conter:

```
.env.local
.env.production
```

Uso:

```ts
// ✅ correto
import { env } from '@/lib/env';
const port = env.PORT;

// ❌ não fazer
const port = process.env.PORT;
```

Toda variável nova precisa ser adicionada ao `envSchema` antes de ser usada.

### CLI de ferramentas (drizzle-kit) e `bunfig.toml`

`bun run db:generate`/`db:migrate`/`db:studio` invocam o binário do
`drizzle-kit`, que tem shebang `#!/usr/bin/env node` — por padrão o Bun
delega esse binário pra um subprocesso Node real, e o auto-load de `.env`
do Bun não atravessa esse subprocesso (`DATABASE_URL` chegaria vazio ali).
Corrigir isso é infra, não motivo pra abrir exceção na regra de env —
`bunfig.toml` na raiz do projeto:

```toml
[run]
bun = true
```

Isso força `bun run <script>` a rodar bins com shebang `node` sob o
próprio runtime do Bun, então o auto-load de `.env` passa a valer também
pro processo do `drizzle-kit`.

Com isso resolvido, `drizzle.config.ts` **não é exceção** à regra "nunca
`process.env` direto" — lê a env por um módulo validado com Zod e com o
mesmo fail-fast de qualquer outro lugar do projeto. Mas **não** pelo
`lib/env.ts` da aplicação: aquele exige `REDIS_URL`, `BETTER_AUTH_SECRET`
etc., e o job de migration do CD (`docs/ci-cd.md`) não deve receber
segredos que não usa. O tooling de CLI tem seu próprio módulo mínimo,
`lib/env-tooling.ts`, que exige só a URL do banco — e prefere a URL
**direta** (`DATABASE_URL_UNPOOLED`, ver "### Neon"), caindo em
`DATABASE_URL` onde não há pooler (local e CI):

```ts
// lib/env-tooling.ts — única leitura de env do tooling de CLI
import { z } from 'zod/v4';

const schema = z.object({
  DATABASE_URL: z.url().optional(),
  DATABASE_URL_UNPOOLED: z.url().optional(),
});

const parsed = schema.safeParse(process.env);
const url = parsed.success
  ? (parsed.data.DATABASE_URL_UNPOOLED ?? parsed.data.DATABASE_URL)
  : undefined;

if (!url) {
  console.error('DATABASE_URL_UNPOOLED (or DATABASE_URL, locally) is required');
  process.exit(1);
}

export const migrationUrl = url;
```

```ts
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';
import { migrationUrl } from './src/lib/env-tooling';

export default defineConfig({
  dialect: 'postgresql',
  schema: './src/db/schema',
  out: './src/db/migrations',
  dbCredentials: { url: migrationUrl },
});
```

## Redis

Redis faz parte da stack oficial, com três responsabilidades — e nenhuma
outra além destas três sem registrar uma nova decisão em "Decisões
registradas":

**Cache** — cache-aside manual, sem lib de decorator mágico escondendo
quando algo é lido do banco ou do cache. Chave no formato
`t:<organization_id>:<module>:<action>:<hash-dos-params>` — o prefixo de
tenant é obrigatório (sem ele, dois tenants com os mesmos params
compartilhariam a chave e um leria o cache do outro), TTL padrão curto (segundos a poucos
minutos, definido por feature conforme volatilidade do dado). Regra dura:
**Redis nunca é fonte de verdade** — todo valor em cache precisa ser
reconstruível a partir do Postgres a qualquer momento, sem perda de dado se
o Redis for zerado.

**Rate limit** — plugin Elysia dedicado, contador em Redis com TTL
correspondente à janela (`INCR` + `EXPIRE key ttl NX`). Chave por IP em
rotas públicas, por `(organization_id, user_id)` em rotas autenticadas. Excedeu o limite → 429
(`TooManyRequestsError`, mesmo shape de erro do `AppError`, ver "Tratamento
de erros"). O IP do cliente atrás de proxy/load balancer não é o do socket
— ver "## Produção: proxy, CORS e limites".

`EXPIRE ... NX` (Redis ≥ 7.0 — a baseline usa `redis:7`) só define o TTL se
a chave ainda não tem um. Isso é mais seguro que "`EXPIRE` só quando
`INCR` devolve 1": `INCR` e `EXPIRE` são dois comandos, e se o processo cair
entre eles a chave ficaria **sem TTL para sempre** (bloqueando o cliente
para sempre); com `NX`, a próxima requisição repõe o TTL sozinha.

**Sessão do Better Auth** — Postgres continua sendo a fonte de verdade da
sessão (mantém a auditoria e a invalidação por `user_id` já descritas
acima). Redis entra como *secondary storage*: cache de leitura na frente do
Postgres, reduzindo round-trip de banco em toda request autenticada.
Invalidação (logout, revogação) precisa atingir os dois — nunca só o cache.

Isso exige uma flag explícita: por padrão, quando `secondaryStorage` é
configurado, o Better Auth guarda a sessão **só** no Redis, não no Postgres
— o oposto do que a regra "Redis nunca é fonte de verdade" exige. A opção
`session.storeSessionInDatabase: true` é o que garante o comportamento
documentado acima (Postgres sempre grava, Redis é só cache):

```ts
// lib/auth.ts
export const auth = betterAuth({
  // ...
  session: {
    storeSessionInDatabase: true, // Postgres continua fonte de verdade
  },
  secondaryStorage: {
    get: (key) => redis.get(key),
    getAndDelete: (key) => redis.send('GETDEL', [key]),
    increment: async (key, ttl) => {
      const value = await redis.incr(key);
      await redis.send('EXPIRE', [key, String(ttl), 'NX']); // repõe o TTL se um EXPIRE anterior se perdeu
      return value;
    },
    set: async (key, value, ttl) => {
      if (ttl) await redis.send('SET', [key, value, 'EX', String(ttl)]); // atômico: valor + TTL
      else await redis.set(key, value);
    },
    delete: async (key) => {
      await redis.del(key); // RedisClient.del() resolve pra number; SecondaryStorage.delete espera void
    },
  },
});
```

As 5 funções (`get`/`getAndDelete`/`increment`/`set`/`delete`) são a
interface `SecondaryStorage` completa que o Better Auth exige — não é
opcional implementar só parte dela.

**Conexão**: client Redis nativo do Bun (`import { RedisClient } from
'bun'`) — sem dependência extra, mesmo princípio já usado para o driver
Postgres e para `.env`. `ioredis` não é necessário: o Better Auth trata
`secondaryStorage` como interface (`get`/`getAndDelete`/`increment`/
`set`/`delete`), implementada manualmente independente do client
escolhido; não estamos usando o pacote `@better-auth/redis-storage`
oficial, que é o único motivo pra precisar de `ioredis` aqui. Client
único em `lib/redis.ts`, reaproveitado pelas três responsabilidades.
`REDIS_URL` entra no `envSchema` (`lib/env.ts`), mesma regra de "nunca
`process.env` direto" que vale para qualquer outra variável:

```ts
// lib/redis.ts
import { RedisClient } from 'bun';
import { env } from '@/lib/env';

export const redis = new RedisClient(env.REDIS_URL);
```

### Redis em produção

Redis **gerenciado, com TLS** (ex: Upstash, Redis Cloud) — o Neon não
oferece Redis, e um container de Redis ao lado da API não tem persistência
nem HA gerenciada. Fora essa escolha de provedor (por instância), o que a
baseline fixa:

- `REDIS_URL` com esquema **`rediss://`** — o `RedisClient` do Bun ativa TLS
  por esse esquema, sem opção extra; `redis://` sem TLS só em dev/CI.
- Índice de banco por ambiente, na própria URL (`.../0` em dev/prod,
  `.../1` nos testes, ver `docs/testing.md`).
- Defaults do `RedisClient` já servem: `autoReconnect` ligado,
  `connectionTimeout` de 10 s. Se um provedor derrubar conexões ociosas,
  o reconnect automático cobre — **não** configurar `idleTimeout` (o Bun
  fecha a conexão por ociosidade e não reconecta sozinho).
- Redis fora do ar não corrompe nada (nunca é fonte de verdade), mas cache
  e rate limit ficam indisponíveis: `/ready` devolve 503 (ver
  "## Health check") e a decisão de fail-open/fail-closed do rate limit é
  da instância, registrada em `docs/features/`.

## Autenticação e autorização

**Autenticação**: [Better Auth](https://www.better-auth.com), com adapter
Drizzle e o plugin `organization` (cada organização é um tenant — ver
"### Tenant: organização ativa e contexto"). Sessão persistida no próprio
Postgres do projeto (tabela `sessions` do Better Auth), não é stateless —
isso permite:
- Deslogar todas as sessões de um usuário imediatamente (invalidar por
  `user_id`, sem esperar expiração de token).
- Rastreabilidade: cada sessão registra `user_id`, criado em, último uso, e
  idealmente IP/user-agent para auditoria.

**Schema e IDs**: as tabelas do Better Auth seguem as mesmas regras
não-negociáveis de qualquer tabela do projeto — PK `uuid` e nome de tabela
no plural (ver "## Banco de dados" e `docs/conventions.md`) — via
configuração da instância, não como exceção:

```ts
// lib/auth.ts
import { betterAuth } from 'better-auth';
import { drizzleAdapter } from 'better-auth/adapters/drizzle';
import { organization } from 'better-auth/plugins';
import { memberAc, ownerAc } from 'better-auth/plugins/organization/access';
import { asc, eq } from 'drizzle-orm';
import { db } from '@/lib/db';
import { env } from '@/lib/env';
import { ForbiddenError } from '@/lib/errors';
import * as schema from '@/db/schema';

export const auth = betterAuth({
  database: drizzleAdapter(db, {
    provider: 'pg',
    schema,
    usePlural: true, // tabelas no plural (users, sessions, accounts, organizations, members…)
  }),
  // passados explicitamente: o Better Auth leria process.env sozinho, e a
  // regra é que só lib/env.ts lê o ambiente
  baseURL: env.BETTER_AUTH_URL,
  secret: env.BETTER_AUTH_SECRET,
  trustedOrigins: env.TRUSTED_ORIGINS, // a mesma lista do CORS
  advanced: {
    database: {
      generateId: 'uuid', // PK uuid nativo, não text
    },
    useSecureCookies: env.NODE_ENV === 'production',
    ipAddress: {
      // só com proxy confiável na frente — ver "## Produção: proxy, CORS e limites"
      ipAddressHeaders: env.CLIENT_IP_HEADER ? [env.CLIENT_IP_HEADER] : undefined,
    },
  },
  plugins: [
    organization({
      // decisão de produto por instância: quem pode criar uma organização
      allowUserToCreateOrganization: true,
      // administração da organização = só o dono. O `admin` padrão do plugin não é
      // usado: o `admin` da baseline é role de MÓDULO (user_module_roles).
      roles: { owner: ownerAc, member: memberAc },
      organizationHooks: {
        // a role do plugin só pode ser 'member'; 'owner' nasce na criação da organização
        // e só muda pela feature transfer-ownership
        beforeUpdateMemberRole: async ({ member, newRole }) => {
          if (member.role === 'owner' || newRole !== 'member') {
            throw new ForbiddenError('A role do plugin é fixa; a propriedade muda por transfer-ownership');
          }
        },
        beforeCreateInvitation: async ({ invitation }) => {
          if (invitation.role !== 'member') throw new ForbiddenError('Convites entram como member');
        },
      },
    }),
  ],
  databaseHooks: {
    session: {
      create: {
        // sessão nova já nasce com uma organização ativa (a mais antiga do usuário)
        before: async (session) => {
          const membership = await db.query.members.findFirst({
            where: eq(schema.members.userId, session.userId),
            orderBy: asc(schema.members.createdAt),
          });
          return { data: { ...session, activeOrganizationId: membership?.organizationId } };
        },
      },
    },
  },
  // + session.storeSessionInDatabase e secondaryStorage, ver "## Redis"
});
```

Rodar `bunx @better-auth/cli generate` **depois** de configurar `usePlural`,
`generateId: 'uuid'` e o plugin `organization` acima — o gerador lê essa
configuração e já emite `uuid('id')`, nomes de tabela no plural e as tabelas
do plugin (`organizations`, `members`, `invitations`, mais a coluna
`active_organization_id` em `sessions`), sem precisar editar nada à mão
para isso.

Dois ajustes manuais adicionais, exigidos pelo modelo de tenant (ver "###
Tenant: organização ativa e contexto"), também sobrescritos a cada
`generate`:

- `members`: acrescentar a coluna derivada `isOwner` e o índice único parcial
  de um dono por organização:

```ts
isOwner: boolean('is_owner')
  .notNull()
  .generatedAlwaysAs((): SQL => sql`${members.role} = 'owner'`),
// …e, na configuração da tabela:
uniqueIndex('members_one_owner_uq')
  .on(table.organizationId)
  .where(sql`${table.role} = 'owner'`),
```

- `sessions.active_organization_id`: o gerador emite `text` sem FK; a baseline
  exige `uuid` e integridade —
  `uuid('active_organization_id').references(() => organizations.id, { onDelete: 'set null' })`.

Uma exceção real, sem solução via configuração: o gerador do Better Auth
sempre emite `timestamp('created_at')` **sem** `{ withTimezone: true }` —
isso é hardcoded no código-fonte do gerador, nenhuma opção do Better Auth
controla esse detalhe. Depois de rodar `generate`, editar à mão os campos
`createdAt`/`updatedAt` do schema gerado (`db/schema/auth.ts`) para
adicionar `{ withTimezone: true }` — e reaplicar esse ajuste sempre que o
schema for regenerado (o comando reescreve o arquivo inteiro).

Outra exceção real, também sem solução via configuração: a tabela
`accounts` ganhou a coluna `issuer` (`text`, obrigatória) no core do
Better Auth a partir da v1.7 — usada junto com `accountId` como índice
único composto pra identificar a conta externa. O gerador do CLI pode
ficar defasado em relação ao core instalado e não emitir essa coluna
mesmo com o core já exigindo ela em runtime, causando erro 500 real no
sign-up até adicionar a coluna à mão e regenerar a migration. Valor
convencionado pra contas de credential (`emailAndPassword`):
`local:credential`; pra OAuth/OIDC, o issuer real do provedor (ou
`local:oauth:<providerId>` quando o provedor não expõe um).

Ver seção "## Redis" acima para como a sessão usa Redis como cache de
leitura na frente do Postgres, sem deixar de ser Postgres a fonte de verdade.

### Tenant: organização ativa e contexto

**Tenant = organização** do Better Auth. O usuário é global (e-mail único) e
pode pertencer a várias organizações. A **organização ativa** vive na sessão
(`sessions.active_organization_id`); o usuário troca de organização por um
endpoint do plugin, e a sessão nova já nasce com uma organização ativa
(`databaseHooks` acima). Criar organização, convidar, aceitar convite,
remover membro e trocar a organização ativa são **endpoints do próprio
plugin** (montados com o resto do Better Auth em `/api/auth/…`) — a baseline
não reimplementa isso como feature. O que é decisão de produto fica com a
instância: quem pode criar organização (`allowUserToCreateOrganization`) e
como o e-mail de convite é enviado.

**O dono (`isOwner`).** `members.is_owner` é uma flag booleana: `true` só
para o proprietário da organização. Há **exatamente um por organização**
(índice único parcial), e o dono é `admin` em **qualquer módulo** dela (ver
"### Autorização por role, por módulo"). Quem cria a organização vira o dono.
`is_owner` é uma coluna **gerada** (`role = 'owner'`): ninguém a escreve, e
por isso ela nunca diverge da role do plugin.

**Dois planos, sem sobreposição de nomes:**

- `members.role` é encanaria do plugin de organização e só assume `owner` (o
  dono) ou `member` (todos os outros). É o que faz os endpoints do Better
  Auth — convidar, remover membro, excluir a organização — funcionarem só
  para o dono. A aplicação **nunca** lê `members.role` para autorizar: lê
  `is_owner`.
- O `admin` da baseline é uma role **de módulo** (`user_module_roles`), sem
  relação com o `admin` padrão do plugin, que **não é usado**. Os hooks de
  `lib/auth.ts` impedem atribuí-lo (ou `owner`) por convite ou por
  `updateMemberRole`: a role do plugin é sempre `member`, exceto a do dono.

A propriedade só muda pela feature `transfer-ownership` (só o dono; o alvo
precisa ser membro da organização), que numa `db.transaction` rebaixa o dono
atual (`role = 'member'`) e depois promove o alvo (`role = 'owner'`) — nessa
ordem, por causa do índice único:

```ts
// transfer-ownership.service.ts
export async function transferOwnership(newOwnerUserId: string, ctx: TenantContext) {
  if (!ctx.isOwner) throw new ForbiddenError('Só o dono transfere a propriedade');

  await db.transaction(async (tx) => {
    const target = await tx.query.members.findFirst({
      where: and(eq(members.organizationId, ctx.organizationId), eq(members.userId, newOwnerUserId)),
    });
    if (!target) throw new NotFoundError('Membro não encontrado nesta organização');

    await tx
      .update(members)
      .set({ role: 'member' })
      .where(and(eq(members.organizationId, ctx.organizationId), eq(members.userId, ctx.userId)));
    await tx.update(members).set({ role: 'owner' }).where(eq(members.id, target.id));
  });
}
```

Integração com Elysia via `.mount()` + dois `macro`, com escopos diferentes:

- **`auth: true`** — só exige sessão válida. É para o que **não** pertence a
  um tenant: listar as minhas organizações, criar uma, trocar a ativa.
- **`tenant: true`** — exige sessão **e** organização ativa **e** que o
  usuário seja mesmo membro dela; expõe `tenant`, o `TenantContext`
  (`userId`, `organizationId`, `isOwner`). **Toda rota de negócio usa
  `tenant: true`.** Sem organização ativa a resposta é `403`.

```ts
// plugins/auth.ts
import { Elysia } from 'elysia';
import { and, eq } from 'drizzle-orm';
import { members } from '@/db/schema';
import { auth } from '@/lib/auth';
import { db } from '@/lib/db';
import { ForbiddenError, UnauthorizedError } from '@/lib/errors';
import type { TenantContext } from '@/lib/tenant';

export const authPlugin = new Elysia({ name: 'better-auth' })
  .mount(auth.handler)
  .macro({
    // só sessão válida — listar/criar/trocar organização
    auth: {
      async resolve({ request: { headers } }) {
        const session = await auth.api.getSession({ headers });
        if (!session) throw new UnauthorizedError();
        return { user: session.user, session: session.session };
      },
    },
    // sessão + organização ativa + membership — toda rota de negócio
    tenant: {
      async resolve({ request: { headers } }) {
        const session = await auth.api.getSession({ headers });
        if (!session) throw new UnauthorizedError();

        const organizationId = session.session.activeOrganizationId;
        if (!organizationId) throw new ForbiddenError('Nenhuma organização ativa');

        // confirma a membership: activeOrganizationId pode estar velho (membro removido)
        const member = await db.query.members.findFirst({
          where: and(eq(members.organizationId, organizationId), eq(members.userId, session.user.id)),
        });
        if (!member) throw new ForbiddenError('Você não é membro desta organização');

        const tenant: TenantContext = {
          userId: session.user.id,
          organizationId,
          isOwner: member.isOwner,
        };
        return { user: session.user, tenant };
      },
    },
  });
```

Os dois macros **lançam** `UnauthorizedError`/`ForbiddenError` em vez de
devolver `status(401)`: assim o `.onError` global formata a resposta no mesmo
shape de erro de todo o resto (ver "## Tratamento de erros"). A checagem de
membership a cada request cobre o caso de a organização ativa ter ficado
velha na sessão depois de o usuário ser removido dela.

### Autorização por role, por módulo

Baseada em role, **por módulo dentro da organização ativa**: o mesmo usuário
pode ser `editor` em `tasks` e `user` em `billing`, e ter roles diferentes em
organizações diferentes. As roles, em ordem crescente de permissão:
`user` → `editor` → `manager` → `admin`.

- `user`: leitura.
- `editor`: pode criar e editar.
- `manager`: tudo do editor + excluir.
- `admin`: acesso total **ao módulo**.

**O dono da organização (`isOwner = true`) é tratado como `admin` em qualquer
módulo**, sem precisar de linha em `user_module_roles`. Membro sem linha para
o módulo tem a role `user`.

Exemplo (módulo `tasks`):

| Ação | user | editor | manager | admin |
|---|---|---|---|---|
| Ler | ✅ | ✅ | ✅ | ✅ |
| Criar | ❌ | ✅ | ✅ | ✅ |
| Editar | ❌ | ✅ | ✅ | ✅ |
| Excluir | ❌ | ❌ | ✅ | ✅ |

Uma única função (`resolveRole`) é a única leitora de `is_owner` e de
`user_module_roles` — nunca dois campos checados em paralelo, o que evita
lógica dupla de autorização (esquecer uma checagem é brecha de segurança) e
estado ambíguo. `isOwner` não é uma permissão a mais ao lado da role: é o que
faz `resolveRole` devolver `admin`.

**Não existe role que atravesse organizações** — nem "super admin" da
plataforma na API. Operação de plataforma (suporte, correção de dado) é feita
fora da API, com acesso direto ao banco e a auditoria da própria infra. Se
uma instância precisar de um papel entre tenants, é proposta de mudança de
baseline (`../../docs/versioning.md`), não uma exceção local.

### Modelagem no banco

```
organizations, members, invitations   (Better Auth — o próprio tenant)
  members: organization_id, user_id,
           role      ("owner" | "member" — encanaria do plugin),
           is_owner  (boolean GERADA: role = 'owner'; único por organização)

user_module_roles
  organization_id -> FK organizations
  user_id         -> FK users
  module          -> "tasks", "billing", etc
  role            -> "user" | "editor" | "manager" | "admin"   (pgEnum)
  unique (organization_id, user_id, module)
```

`(organization_id, user_id, module) → role`: o mesmo usuário pode ter roles
diferentes por módulo e por organização. Não há tabela de role global.

### Matriz de permissão

Fixa em código, não em banco. No monorepo, `permissions`, `Role`,
`Action` e `can` vivem em `@repo/contracts/permissions` (o web usa a mesma
matriz); `resolveRole` fica em `lib/permissions.ts` da API. `admin` é uma chave
normal da matriz — não um caso especial fora dela:

```ts
const permissions = {
  user:    ['read'],
  editor:  ['read', 'create', 'update'],
  manager: ['read', 'create', 'update', 'delete'],
  admin:   ['read', 'create', 'update', 'delete'],
} as const;

export type Role = keyof typeof permissions;
export type Action = (typeof permissions)[Role][number];

export function can(role: Role, action: Action) {
  return (permissions[role] as readonly Action[]).includes(action);
}

export async function resolveRole(ctx: TenantContext, module: string): Promise<Role> {
  // o dono da organização é 'admin' em qualquer módulo — só dentro desta org
  if (ctx.isOwner) return 'admin';

  const row = await db.query.userModuleRoles.findFirst({
    where: inTenant(
      userModuleRoles,
      ctx,
      eq(userModuleRoles.userId, ctx.userId),
      eq(userModuleRoles.module, module),
    ),
  });
  return row?.role ?? 'user';
}
```

**Default explícito**: membro sem linha em `user_module_roles` para o módulo,
naquela organização, tem a role `user` (só leitura) — `resolveRole` nunca
retorna `undefined`, então `permissions[role]` nunca lança por role ausente.
Módulo cuja leitura também precise de autorização explícita deve checar isso
na própria feature (ex: exigir `editor` até para `read`), não mudar o
default global. Custo: `resolveRole` faz no máximo uma query por request
(nenhuma para o dono); se virar gargalo, cachear em Redis com chave de
tenant e invalidar ao mudar a role.

### Onde o check acontece

Dentro do arquivo `<action>.ts` da feature, antes de chamar o service —
sempre `resolveRole` + `can`, nunca um campo de sessão checado à parte:

```ts
export async function deleteTask(id: string, ctx: TenantContext) {
  const role = await resolveRole(ctx, 'tasks');

  if (!can(role, 'delete')) {
    throw new ForbiddenError('Sem permissão para excluir tasks');
  }

  return deleteTaskService(id, ctx);
}
```

```ts
// delete-task.service.ts — 404 também para a task de outro tenant
export async function deleteTaskService(id: string, ctx: TenantContext) {
  const [deleted] = await db
    .delete(tasks)
    .where(inTenant(tasks, ctx, eq(tasks.id, id)))
    .returning();
  if (!deleted) throw new NotFoundError('Task não encontrada');
  return deleted;
}
```

### Gerenciamento de roles e membros

Há três frentes, com donos diferentes:

- **Organização e membros** (convidar, remover membro, excluir a organização,
  trocar a organização ativa): endpoints do plugin `organization` do Better
  Auth, permitidos só ao dono (role `owner` do plugin). O convite entra sempre
  como `member`. Não são features da baseline.
- **Propriedade**: a feature `transfer-ownership` (acima) — só o dono.
- **Role por módulo** (`user_module_roles`): CRUD normal, exposto como feature
  própria (ex: módulo `users`), sempre `tenant: true`:
  `assign-user-role.ts` / `remove-user-role.ts` / `list-user-roles.ts`.
  Exigem `admin` no módulo de administração — o dono já é. **Conceder a role
  `admin` é reservado ao dono**, para que um admin de módulo não escale outros
  membros a `admin`. Toda query usa `inTenant`, e `assign-user-role` valida
  que o usuário-alvo é **membro da mesma organização** antes de gravar.

O frontend consome tudo isso para montar a tela de administração da
organização.

**Primeiro admin**: não há bootstrap nem seed. Quem cria a organização vira o
dono — `admin` em todos os módulos — e a partir daí convida os demais e
atribui as roles.

## Documentação da API

O `inputSchema`/`outputSchema` (Zod) de cada feature é a fonte de verdade —
não escrever doc de contrato HTTP à parte, gerar a partir do schema.

Elysia aceita Zod nativamente como `body`/`response` (via Standard Schema)
já na validação em runtime — sem precisar de adapter. Pra gerar a doc
OpenAPI a partir desses mesmos schemas, falta só um passo: o Zod não expõe
`toJSONSchema` no formato que o plugin espera nativamente, então é
necessário mapear:

```ts
// plugins/openapi.ts
import { openapi } from '@elysia/openapi';
import { z } from 'zod/v4';
import { env } from '@/lib/env';

export const openapiPlugin = openapi({
  enabled: env.NODE_ENV !== 'production', // sem /openapi em produção
  mapJsonSchema: { zod: z.toJSONSchema },
  exclude: { paths: ['/health', '/ready'] },
});
```

A doc interativa expõe todas as rotas e schemas — em produção o plugin fica
**desligado** (`enabled: false` faz o plugin não registrar nenhuma rota).
Instância que precise publicar a doc em produção deve protegê-la (ex:
atrás de `auth: true`, ou só em ambiente interno) e registrar essa decisão.

```ts
// index.ts (trecho — montagem completa em "## Shutdown gracioso")
new Elysia()
  .use(openapiPlugin) // doc interativa em /openapi, só fora de produção
  .use(tasksRoutes)
  .listen(env.PORT);
```

O `inputSchema`/`outputSchema` que já existe em cada `<action>.ts` vira o
`body`/`response` da rota registrada em `<module>.routes.ts` — sem duplicar
definição. Usar `.describe('...')` nos campos do Zod quando precisar
enriquecer a doc com contexto que o nome do campo não deixa óbvio.

Isso é diferente de `docs/features/`: aquilo documenta a *regra de negócio*
(o "porquê"), o OpenAPI documenta o *contrato HTTP* (o "como chamar"). Os
dois se complementam.

## Tratamento de erros

Resposta de erro sempre um objeto JSON consistente, nunca string solta:

```json
{
  "statusCode": 409,
  "error": "ConflictError",
  "message": "Task já existe com esse título"
}
```

Erro de validação (`inputSchema` do Zod) ganha o campo `issues`:

```json
{
  "statusCode": 400,
  "error": "ValidationError",
  "message": "Dados inválidos",
  "issues": [{ "path": "title", "message": "Required" }]
}
```

`statusCode` no body é redundante com o status HTTP da resposta, mas vem da
mesma fonte que o `set.status` abaixo (a classe `AppError`), então nunca
diverge — e facilita quem só tem acesso fácil ao body (logs, debug).

As exceptions de domínio (`lib/errors.ts`) estendem uma classe base
`AppError` que já carrega o `statusCode` de cada uma:

| Classe | Status | Uso |
|---|---|---|
| `ValidationError` | 400 | regra de negócio violada além do schema |
| `UnauthorizedError` | 401 | sem sessão válida |
| `ForbiddenError` | 403 | role sem permissão (`can()` falhou) ou sem organização ativa |
| `NotFoundError` | 404 | recurso inexistente — **inclusive o de outro tenant** |
| `ConflictError` | 409 | conflito de estado (ex: duplicidade) |
| `TooManyRequestsError` | 429 | rate limit excedido |

Um único `.onError()` na raiz trata cinco casos:

```ts
// plugins/error-handler.ts
import { Elysia } from 'elysia';
import { AppError } from '@/lib/errors';
import { logger } from '@/lib/logger';

export const errorHandlerPlugin = new Elysia({ name: 'error-handler' }).onError(
  ({ code, error, set }) => {
    if (error instanceof AppError) {
      set.status = error.statusCode;
      return {
        statusCode: error.statusCode,
        error: error.name,
        message: error.message,
      };
    }

    if (code === 'VALIDATION') {
      set.status = 400;
      return {
        statusCode: 400,
        error: 'ValidationError',
        message: 'Dados inválidos',
        issues: error.all,
      };
    }

    if (code === 'NOT_FOUND') {
      set.status = 404;
      return {
        statusCode: 404,
        error: 'NotFoundError',
        message: 'Rota não encontrada',
      };
    }

    if (code === 'PARSE') {
      set.status = 400;
      return {
        statusCode: 400,
        error: 'ValidationError',
        message: 'Corpo da requisição inválido',
      };
    }

    // pino só serializa Error (message/stack) na chave `err` — `{ error }`
    // sairia como `{}` e o stack se perderia.
    logger.error({ err: error }, 'unhandled error');
    set.status = 500;
    return {
      statusCode: 500,
      error: 'InternalServerError',
      message: 'Erro interno',
    };
  },
);
```

```ts
// index.ts (trecho — montagem completa em "## Shutdown gracioso")
new Elysia().use(errorHandlerPlugin).use(tasksRoutes).listen(env.PORT);
```

1. **Exception de domínio** (`instanceof AppError`) → status e mensagem
   vêm da própria exception.
2. **Erro de validação** (`code === 'VALIDATION'`, do `inputSchema`/Zod) →
   400 com `issues`. Verificar o formato exato de `error.all` ao implementar
   — a estrutura pode variar conforme a versão do Elysia; se não vier no
   shape esperado, usar `error.message` (string) como fallback.
3. **Rota inexistente** (`code === 'NOT_FOUND'`) → 404. Sem esse branch, uma
   URL errada cairia no 500 e ainda seria logada como erro.
4. **Corpo ilegível** (`code === 'PARSE'`, ex: JSON malformado) → 400.
5. **Erro não mapeado** (bug, exception não tipada) → sempre 500 genérico;
   nunca vazar `error.message`/stack pro cliente, logar internamente.

## Health check

Duas rotas, sem autenticação, registradas na raiz (`plugins/health.ts`,
usado em `index.ts`), fora de `src/modules`, excluídas do OpenAPI e do rate
limit:

- **`GET /health`** — *liveness*: `200 { status: 'ok' }` se o processo está
  de pé. **Não consulta nenhuma dependência.** É a rota do `HEALTHCHECK` do
  Dockerfile (`docs/docker.md`) e de probes de liveness.
- **`GET /ready`** — *readiness*: checa Postgres (`select 1`) e Redis
  (`PING`); `200` se ambos respondem, `503` com o detalhe de cada check se
  algum falhar **ou se o processo está em shutdown** (ver "## Shutdown
  gracioso"). É a rota que load balancer/orquestrador usam para decidir se
  mandam tráfego.

Separar as duas importa: uma liveness que depende do banco reinicia o
container a cada oscilação do Postgres — e, com Neon em scale-to-zero, uma
sonda no banco a cada poucos segundos mantém a compute acordada, com custo
(ver "### Neon"). Cada check de `/ready` deve ter timeout curto (~2 s) para
não pendurar a probe.

```ts
// plugins/health.ts
import { Elysia } from 'elysia';
import { sql } from 'drizzle-orm';
import { db } from '@/lib/db';
import { redis } from '@/lib/redis';
import { isShuttingDown } from '@/lib/shutdown';

export const healthPlugin = new Elysia({ name: 'health' })
  .get('/health', () => ({ status: 'ok' }))
  .get('/ready', async ({ set }) => {
    const [database, cache] = await Promise.allSettled([
      db.execute(sql`select 1`),
      redis.send('PING', []),
    ]);
    const checks = {
      database: database.status === 'fulfilled',
      redis: cache.status === 'fulfilled',
    };
    if (isShuttingDown() || !Object.values(checks).every(Boolean)) {
      set.status = 503;
      return { status: 'unavailable', checks };
    }
    return { status: 'ready', checks };
  });
```

O `docker-compose.yml` da baseline só sobe Postgres e Redis, não a
aplicação (`docs/docker.md`), então não há healthcheck de compose apontando
para estas rotas — só existiria se a instância adicionar a app ao compose.

## Shutdown gracioso

No deploy, Docker/orquestrador manda `SIGTERM` e espera um prazo (`docker
stop`: 10 s por padrão) antes do `SIGKILL`. Sem handler, requests em
andamento são cortadas e conexões de banco ficam penduradas até o servidor
(Neon) expirá-las. O `CMD` do Dockerfile em forma *exec* faz o Bun ser o
PID 1 e receber o sinal diretamente.

```ts
// lib/shutdown.ts
import { logger } from '@/lib/logger';

let shuttingDown = false;
export const isShuttingDown = () => shuttingDown;

export function registerShutdown(stop: () => Promise<void>) {
  const handle = async (signal: string) => {
    if (shuttingDown) return;
    shuttingDown = true; // /ready passa a devolver 503 e o balanceador drena
    logger.info({ signal }, 'shutting down');
    try {
      await stop();
      process.exit(0);
    } catch (err) {
      logger.error({ err }, 'error during shutdown');
      process.exit(1);
    }
  };
  process.on('SIGTERM', () => void handle('SIGTERM'));
  process.on('SIGINT', () => void handle('SIGINT'));
}
```

```ts
// index.ts — montagem completa
import { Elysia } from 'elysia';
import { client } from '@/lib/db';
import { env } from '@/lib/env';
import { redis } from '@/lib/redis';
import { registerShutdown } from '@/lib/shutdown';
import { tasksRoutes } from '@/modules/tasks/tasks.routes';
import { corsPlugin } from '@/plugins/cors';
import { errorHandlerPlugin } from '@/plugins/error-handler';
import { healthPlugin } from '@/plugins/health';
import { openapiPlugin } from '@/plugins/openapi';

const app = new Elysia({ serve: { maxRequestBodySize: 1024 * 1024 } })
  .use(errorHandlerPlugin)
  .use(corsPlugin)
  .use(openapiPlugin)
  .use(healthPlugin)
  .use(tasksRoutes)
  .listen(env.PORT);

registerShutdown(async () => {
  await app.stop(); // para de aceitar conexões; requests em andamento terminam
  await client.end({ timeout: 5 }); // postgres.js: espera queries e fecha o pool
  redis.close();
});
```

`app.stop()` sem argumento **não** cancela requests em andamento — é o
comportamento desejado. Se o balanceador demora a perceber o `503` de
`/ready`, adicionar uma espera curta (ex: 5 s) antes de `app.stop()`; o
total precisa caber no prazo do orquestrador (`stop_grace_period`).

## Produção: proxy, CORS e limites

A API roda em container, atrás de um proxy/load balancer que termina TLS.
Isso muda o que o processo enxerga, e a baseline fixa o seguinte:

**IP do cliente.** `server.requestIP(request)` devolve o IP do **proxy**, não
o do cliente — sem tratamento, o rate limit "por IP" coloca todos os
usuários no mesmo bucket. `CLIENT_IP_HEADER` (ex: `x-real-ip`,
`cf-connecting-ip`, `fly-client-ip`) aponta o header que o proxy da
instância **sobrescreve**; o mesmo valor alimenta o rate limit e o
`advanced.ipAddress.ipAddressHeaders` do Better Auth:

```ts
// lib/client-ip.ts
import { env } from '@/lib/env';

type ServerLike = { requestIP(req: Request): { address: string } | null } | null;

export function getClientIp(request: Request, server: ServerLike) {
  const header = env.CLIENT_IP_HEADER;
  const forwarded = header ? request.headers.get(header)?.trim() : undefined;
  return forwarded || server?.requestIP(request)?.address || null;
}
```

Só definir `CLIENT_IP_HEADER` quando o proxy realmente sobrescreve esse
header (nunca repassa o valor do cliente) — caso contrário qualquer cliente
forja o IP e contorna o rate limit. Preferir um header de valor único a
`x-forwarded-for`, cujo primeiro item é forjável. API exposta direto, sem
proxy: deixar a variável vazia.

**CORS.** Frontend em outra origem precisa de CORS, e o Better Auth exige
que as mesmas origens estejam em `trustedOrigins`. Uma única lista
(`TRUSTED_ORIGINS`) alimenta os dois, sempre com origens **explícitas** —
nunca `origin: true`/`*` com `credentials: true`:

```ts
// plugins/cors.ts
import { cors } from '@elysia/cors';
import { env } from '@/lib/env';

export const corsPlugin = cors({
  origin: env.TRUSTED_ORIGINS,
  credentials: true,
});
```

O pacote é `@elysia/cors` (idem `@elysia/openapi`, ver "## Documentação da
API"). O nome antigo, `@elysiajs/*`, ainda é publicado no npm com a mesma
versão, mas o nome canônico — o do `package.json` dos repositórios e da
documentação do Elysia — é `@elysia/*`.

**HTTPS e cookies.** `BETTER_AUTH_URL` é `https://` em produção e
`advanced.useSecureCookies` fica ligado quando `NODE_ENV=production` (ver
"## Autenticação e autorização").

**Limite de body.** O padrão do Bun/Elysia é 128 MB por request; a baseline
fixa 1 MB (`serve.maxRequestBodySize`, ver `index.ts` acima). Feature de
upload sobe o limite por instância e registra o motivo.

**Segredos.** Nunca na imagem, em `ENV` ou `ARG` do Dockerfile — a
plataforma de deploy injeta no runtime. `BETTER_AUTH_SECRET` é gerado por
ambiente (`openssl rand -base64 32`), nunca reaproveitado entre dev e
produção.

## Filas / processamento assíncrono

Fora da baseline por padrão — este template não assume fila/broker até que
uma instância precise de processamento assíncrono de verdade (job longo,
retry, agendamento). Quando precisar: como Redis já está disponível na
stack (ver "## Redis"), preferir um worker Redis-backed (ex: BullMQ) antes
de introduzir um broker dedicado novo (RabbitMQ, SQS, etc). Se a instância
tiver um motivo concreto para um broker dedicado (throughput, garantias de
entrega que BullMQ não cobre, integração já existente), registrar como uma
nova entrada em "Decisões registradas" abaixo, com o contexto específico —
isso é uma decisão por instância, não uma mudança na baseline do template.

## Decisões registradas

Registre aqui decisões arquiteturais importantes com o formato:
**Decisão**: o que foi decidido.
**Contexto**: por que foi necessário decidir.
**Alternativas consideradas**: o que mais foi avaliado.

### Modules + features sem camada de repository

**Decisão**: cada feature é um par `<action>.ts` + `<action>.service.ts`
dentro do módulo; o service fala direto com o banco via Drizzle, sem
camada de repository intermediária.
**Contexto**: reduzir indireção e manter tudo que uma ação precisa
próximo, fácil de achar e de deletar quando a feature morre.
**Alternativas consideradas**: camadas horizontais com repository
genérico (rejeitado por gerar indireção sem ganho claro no tamanho do
projeto — o service já é o único consumidor da query, um repository ali
só repassaria a chamada).

### Redis como parte oficial da stack (cache + rate limit + sessão)

**Decisão**: adotar Redis desde a baseline do template, com três usos
definidos — cache-aside, rate limit e secondary storage da sessão do
Better Auth (ver "## Redis").
**Contexto**: rate limit de alta frequência e cache de leitura são
padrões que toda API real acaba precisando, e Postgres sozinho não atende
bem esse perfil de acesso (muitas escritas pequenas de contador, TTL
nativo, leitura sub-milissegundo).
**Alternativas consideradas**: cache in-memory por processo (rejeitado —
não escala horizontalmente, cada instância teria seu próprio estado);
resolver rate limit e cache só com Postgres (rejeitado — gera lock/IO
desnecessário em uma tabela de alta escrita para um caso de uso que não
precisa de durabilidade).

### Filas fora da baseline por padrão

**Decisão**: não incluir fila/broker na baseline do template; documentar
apenas a recomendação de usar um worker Redis-backed (BullMQ) quando uma
instância precisar (ver "## Filas / processamento assíncrono").
**Contexto**: fila é infraestrutura com custo operacional real (mais um
processo, mais um ponto de falha) que nem toda instância derivada vai
precisar; forçar isso na baseline seria inchar o template para um caso de
uso que não é universal.
**Alternativas consideradas**: incluir BullMQ já configurado na baseline
(rejeitado — adicionaria peça de infra e worker process a instâncias que
nunca vão processar job assíncrono).

### Versionamento semântico do template como mecanismo de governança

**Decisão**: versionar a baseline do template com SemVer
(`../../docs/versioning.md`) e manter `../../docs/CHANGELOG.md`, distinguindo
explicitamente regra "não-negociável" (trava, só muda com bump de versão)
de configuração evolutiva por instância.
**Contexto**: sem um mecanismo formal, "atualizar uma regra" vira decisão
ad-hoc tomada no meio de uma tarefa qualquer, e cada instância deriva
silenciosamente da baseline sem que ninguém perceba quando ou por quê.
**Alternativas consideradas**: apenas git tags no template sem changelog
estruturado (rejeitado — não registra o "por quê" de cada mudança, só o
"quando"; instância derivada precisaria ler o diff inteiro pra entender o
motivo de uma mudança de baseline).

### Driver Postgres: postgres.js em vez de drizzle-orm/bun-sql

**Decisão**: fixar `postgres.js` como driver oficial de conexão ao
Postgres (ver "## Banco de dados").
**Contexto**: esta seção não fixava o driver, e isso gerou uma pausa real
durante o scaffold de uma instância derivada (o agente parou pra
perguntar qual usar, como o próprio processo de scaffold prescreve). A
documentação do Drizzle registra bug conhecido de execução de statements
concorrentes no driver `bun-sql` (Bun ≥ 1.2.0), risco real numa API que
roda queries em paralelo.
**Alternativas consideradas**: `drizzle-orm/bun-sql` (rejeitado — apesar
de nativo do Bun e sem dependência extra, o risco de bug em queries
concorrentes não compensa numa baseline que se propõe "sólida"; pode ser
revisitado numa futura MINOR se o bug for resolvido e confirmado numa
versão específica do Bun).

### Client Redis: RedisClient nativo do Bun em vez de ioredis

**Decisão**: fixar o `RedisClient` nativo do Bun como client oficial de
Redis (ver "## Redis").
**Contexto**: a seção "## Redis" definia as três responsabilidades e o
client único, mas nunca fixava a biblioteca — mesma lacuna que gerou
pausa no scaffold do driver Postgres, agora repetida aqui. Ao contrário
do `bun-sql`, não há bug conhecido documentado para o `RedisClient`; e o
argumento "é o que o Better Auth usa oficialmente" não se aplica, porque
não estamos usando o pacote oficial `@better-auth/redis-storage` (que é o
único motivo do exemplo oficial usar `ioredis`) — a baseline já decidiu
ter client único compartilhado, então o adapter de `secondaryStorage` é
manual de qualquer forma.
**Alternativas consideradas**: `ioredis` (rejeitado — adicionaria
dependência que o Bun já cobre nativamente, sem ganho concreto dado que
não usamos o pacote oficial do Better Auth de qualquer forma).

### Better Auth: uuid nativo e tabelas no plural via configuração, não exceção

**Decisão**: configurar `advanced.database.generateId: 'uuid'` e
`usePlural: true` no adapter Drizzle de `lib/auth.ts`, para que as tabelas
do Better Auth sigam as mesmas regras de PK `uuid` e tabela no plural do
resto do projeto — em vez de aceitar `text`/singular como limitação da
biblioteca (ver "## Autenticação e autorização").
**Contexto**: um teste real de scaffold gerou o schema do Better Auth com
`text`/singular e tratou isso como limitação inevitável. Pesquisa na
documentação oficial e no código-fonte do gerador Drizzle do CLI mostrou
que `text`/singular é só o *default* sem configuração — `generateId:
'uuid'` e `usePlural: true` são opções de primeira classe, documentadas,
que o próprio gerador do CLI lê para emitir o schema já correto. Duas
limitações reais permanecem, sem solução via configuração:
- o gerador nunca emite `{ withTimezone: true }` nos campos de timestamp
  (hardcoded no código-fonte);
- a coluna `issuer` da tabela `accounts` (obrigatória a partir do core
  Better Auth v1.7, parte do índice único `issuer` + `accountId`) pode
  não ser emitida pelo gerador do CLI quando ele está defasado em
  relação ao core instalado — confirmado num scaffold real
  (`@better-auth/cli@1.4.21` vs. core `1.7.1`), com erro 500 real no
  sign-up até adicionar a coluna à mão.

Os dois ajustes continuam manuais, pós-`generate`.
**Alternativas consideradas**: aceitar `text`/singular como exceção
documentada para as tabelas do Better Auth (rejeitado — era a suposição
inicial, baseada em como um scaffold de teste rodou sem configurar essas
opções antes de gerar o schema; a pesquisa mostrou que a exceção não era
necessária).

### tsconfig.json: paths sem baseUrl para o alias `@`

**Decisão**: configurar o alias `@` só com `paths` (`"@/*": ["./src/*"]`),
sem `baseUrl` (ver `docs/conventions.md`, seção "Imports").
**Contexto**: um scaffold de teste gerou `tsconfig.json` com o par
`baseUrl` + `paths` (o padrão pré-TS 4.1), disparando aviso de
depreciação: `baseUrl` foi descontinuado no TS 6.0 e será removido no TS
7.0. Desde o TS 4.1, `paths` funciona sozinho, resolvido relativo ao
próprio `tsconfig.json`; confirmado que o resolver do Bun já lê `paths`
sem `baseUrl` corretamente (usa a base implícita do arquivo).
**Alternativas consideradas**: manter `baseUrl` e silenciar o aviso com
`"ignoreDeprecations": "6.0"` (rejeitado — adia o problema até o TS 7.0
remover a opção de vez, em vez de corrigir agora que a correção é trivial).

### drizzle.config.ts usa env validado, não process.env direto

**Decisão**: `bunfig.toml` com `[run] bun = true` na baseline, e
`drizzle.config.ts` importando `env` de `lib/env.ts` em vez de
`process.env` direto (ver "## Variáveis de ambiente"). *Atualizado na
0.15.0:* o import passou a ser `migrationUrl` de `lib/env-tooling.ts` (env
mínima, sem `REDIS_URL`/`BETTER_AUTH_*`) — a regra "nunca `process.env`
direto" não mudou, só o módulo validado que o tooling usa (ver "Migrations
rodam no CD, com env mínimo").
**Contexto**: um scaffold de teste usou `process.env.DATABASE_URL` direto
em `drizzle.config.ts`, violando a regra não-negociável "nunca
`process.env` direto" sem que a doc abrisse exceção em lugar nenhum. A
causa raiz é técnica, não de regra: o binário do `drizzle-kit` tem shebang
`node`, e por padrão o Bun delega pra um subprocesso Node real onde o
auto-load de `.env` do Bun não chega — `bunfig.toml` corrige isso na raiz,
sem precisar de exceção na regra de env.
**Alternativas consideradas**: documentar `drizzle.config.ts` como
exceção aceita à regra de `process.env` (rejeitado — mesma categoria de
erro do caso do Better Auth: tratar como limitação inevitável algo que
tinha correção de infra simples).

### session.storeSessionInDatabase: true — sem isso, Redis vira a fonte de verdade

**Decisão**: fixar `session.storeSessionInDatabase: true` na config do
Better Auth como parte não-negociável de "## Redis" / seção "Sessão do
Better Auth" — não é detalhe opcional, é o que garante a regra "Redis
nunca é fonte de verdade" na prática.
**Contexto**: a doc já afirmava que "Postgres continua sendo a fonte de
verdade da sessão" quando Redis é `secondaryStorage`, mas nunca verificou
o comportamento real do Better Auth: por padrão, configurar
`secondaryStorage` faz a sessão ser gravada **só** no Redis, não no
Postgres — o oposto do que a doc afirmava. Sem essa flag, um Redis
zerado apagaria todas as sessões ativas, quebrando a garantia de
auditoria/rastreabilidade já documentada. Pesquisa confirmou a flag via
Context7 antes de fixar isso como regra.
**Alternativas consideradas**: nenhuma — não existe forma de ter Redis
como cache (não fonte de verdade) de sessão sem essa flag; a alternativa
seria não usar `secondaryStorage` pra sessão, o que contradiria a decisão
já tomada de ter Redis cacheando leitura de sessão (ver "Redis como parte
oficial da stack" acima).

### Admin como role global (user_global_roles), não flag booleana

> **Substituída na 0.16.0** por "Roles por organização, sem role global"
> (abaixo): com multi-tenancy obrigatório, `admin` deixa de ser global e
> `user_global_roles` deixa de existir. O princípio de fundo — `admin` é
> role resolvida por uma única função, nunca flag ao lado da role — segue
> valendo. Mantida como registro histórico.

**Decisão**: substituir a coluna `is_admin` em `users` por
`user_global_roles(user_id, role)`, resolvida junto com a role por módulo
através de uma única função (`resolveRole`) — `admin` é role, nunca uma
flag ao lado da role (ver "## Autenticação e autorização").
**Contexto** (levantado pelo usuário, revisando o design original):
- **Lógica dupla de autorização** — com `isAdmin` separado, todo lugar
  que checa permissão precisaria lembrar de checar os dois campos; um
  esquecimento vira brecha de segurança.
- **Estados ambíguos** — `role: 'editor'` + `isAdmin: true` não tem
  significado único definido; com `admin` como valor de role num escopo
  próprio, esse estado simplesmente não existe.
**Alternativas consideradas**: manter `is_admin` como coluna booleana
(rejeitado pelos dois motivos acima); usar um valor sentinela de módulo
(ex: `module: '*'`) dentro da própria `user_module_roles` pra representar
"admin global" (rejeitado — reintroduz o mesmo risco da flag: é um caso
especial que quem escreve o check pode esquecer de considerar, só que
escondido dentro dos dados em vez de num campo separado).

### Single-tenant por design — sem modelagem antecipando multi-tenancy futura

> **Substituída na 0.16.0** por "Multi-tenant obrigatório: banco único com
> `organization_id`" (abaixo): a baseline passou a ser somente
> multi-tenant, então a proibição de modelar tenant deixou de existir.
> Mantida como registro histórico do raciocínio anterior.

**Decisão**: o template nunca modela, documenta ou justifica uma
decisão de código citando compatibilidade com um futuro fork
multi-tenant — nem como razão de design, nem como nota especulativa em
"Decisões registradas". Se multi-tenancy virar necessidade real de
alguma instância, é proposta de mudança de baseline a desenhar do zero
então (`../../docs/versioning.md`), não algo preparado hoje "por garantia".
**Contexto**: revendo o histórico do template, mais de um ponto usava
"isso ajuda um futuro fork multi-tenant" como justificativa parcial de
design (a decisão "Admin como role global" citava `organization_members`
como precedente; "Modelagem no banco" especulava como as tabelas de role
se dividiriam num cenário multi-tenant). Design especulativo pra um
requisito que não existe adiciona complexidade conceitual sem benefício
verificável — a forma real que multi-tenancy exigiria só fica clara
quando (e se) o requisito aparecer de verdade, com escopo e escala reais
na mão.
**Alternativas consideradas**: manter essas referências como
"documentação de intenção futura" (rejeitado — é exatamente o padrão que
se decidiu nunca mais fazer; também trava contribuidores futuros numa
forma específica de resolver algo que ainda nem é requisito real).

### Error handler usa logger, não console.error

**Decisão**: o snippet de `plugins/error-handler.ts` (seção "Tratamento
de erros") usa `logger.error({ err: error }, 'unhandled error')` no
branch de erro não mapeado, nunca `console.error`. A chave é `err` (não
`error`) porque o pino só aplica o serializer de `Error` a essa chave —
com `{ error }` o log sairia `{}`, sem message nem stack.
**Contexto**: uma instância derivada gerou o error handler copiando o
snippet do template literalmente, incluindo `console.error(error)` —
violação direta da regra "nunca console.* em produção, usar o logger"
que o próprio `CLAUDE.md`/`docs/observability.md` já exigem. O bug
estava no exemplo da doc, não na execução do scaffold.
**Alternativas consideradas**: nenhuma — é correção de um exemplo que
contradizia uma regra não-negociável já existente, não uma decisão de
design nova.

### ci.yml do job de teste fixa env vars e nome do banco (app_test)

> **Adaptada no monorepo**: o banco de teste do CI é `app_test` dentro do
> branch Neon da PR, não um service container
> (`../../docs/architecture.md`, "### Testes do CI em branch Neon, não em
> service container"). O princípio (nome fixo, igual ao `.env.test`)
> continua.

**Decisão**: o snippet de `ci.yml` (`docs/ci-cd.md`, job `test`) declara
explicitamente `DATABASE_URL`/`REDIS_URL`/`BETTER_AUTH_SECRET`/
`BETTER_AUTH_URL` como `env:` do job, e o Postgres do service container
usa `POSTGRES_DB: app_test` — mesmo nome usado em `.env.test`.
**Contexto**: o snippet nunca definia essas variáveis, então
`bun run db:migrate`/`bun test` não tinham como funcionar sem que cada
instância inventasse os valores por conta própria — uma instância
derivada preencheu com o banco default `postgres`, divergindo do nome
`app_test` que `.env.test`/`docs/testing.md` já usam como convenção
de "banco de teste dedicado".
**Alternativas consideradas**: deixar cada instância decidir o nome do
banco de CI livremente (rejeitado — gera a mesma deriva silenciosa que
`../../docs/versioning.md` já existe para evitar, só que no nome de um
recurso em vez de numa regra).

### Remover ação `revert` do modelo de permissão — CRUD tradicional

**Decisão**: o modelo de autorização (`lib/permissions.ts`, matriz de
permissão) usa só as quatro ações de CRUD tradicional — `read`,
`create`, `update`, `delete` — sem ação `revert` separada. `manager`/
`admin` ganham `delete`, não mais `delete` + `revert`. A linha da matriz
"Excluir / reverter" vira só "Excluir".
**Contexto**: `revert` existia na baseline desde o início do modelo de
autorização sem nunca ter semântica clara, e a v0.10.3 tentou consertar
isso condicionando `revert` a exigir histórico da entidade. Revisando de
novo, ficou claro que não existe uma operação "reverter" distinta de
exclusão — o que existe é soft vs. hard delete (decisão por feature, já
documentada em "## Banco de dados"), e desfazer um soft delete é só
`update` normal do registro, não uma capacidade de autorização própria.
Manter `revert` como ação separada cria uma permissão fantasma que não
mapeia pra nenhum verbo HTTP/operação real.
**Alternativas consideradas**: manter `revert` como ação condicionada a
ter histórico, como a v0.10.3 propôs (rejeitado — o usuário preferiu
simplificar pro CRUD tradicional em vez de manter uma ação especial com
pré-condição; restaurar um soft delete já é coberto por `update`).

### Banco de teste: truncar entre testes, não transação por teste

**Decisão**: `docs/testing.md` documenta truncar tabelas no `afterEach`
como estratégia de isolamento de banco de teste — substitui o padrão
anterior de transação por teste com rollback (`withTestTransaction`).
**Contexto**: o padrão anterior pressupunha que o código sob teste
recebesse `tx` por parâmetro, mas "Modules + features sem camada de
repository" (acima) manda todo `<action>.service.ts` importar o
singleton `db` direto, nunca receber conexão por parâmetro. Uma
transação aberta só pelo teste nunca contém a escrita feita pelo
service — ele grava por uma conexão separada, fora da transação do
teste. Como todo teste de service obrigatório chama o service de
verdade, isso quebrava o isolamento no caso comum inteiro, não numa
borda. Encontrado construindo o primeiro módulo de negócio real numa
instância derivada.
**Alternativas consideradas**: propagar a transação via
`AsyncLocalStorage` pra fazer o singleton `db` resolver pra `tx` quando
uma estiver ativa (rejeitado por agora — funcionaria, mas adiciona
indireção/mágica implícita numa baseline que preza simplicidade
explícita; revisitar se o custo de truncar (tempo de teste, manutenção
da lista de tabelas) virar problema real).

### Enum de banco via pgEnum, nunca text/varchar com opção `enum`

**Decisão**: coluna com valores fixos conhecidos (status, role, etc.)
sempre usa `pgEnum` (tipo nativo do Postgres) — nunca `text`/`varchar`
com a opção `{ enum: [...] }`.
**Contexto**: "### Modelagem no banco" só descrevia `role` como
pseudocódigo (`role -> "user" | "editor" | "manager"`), sem fixar a
implementação real em Drizzle. Construindo o primeiro módulo de
negócio real (coluna `status` com valores fixos), a instância derivada
implementou como `text('status', { enum: TODO_STATUSES })` — sintaxe
válida do Drizzle, mas que só restringe o tipo TypeScript; não gera
constraint nenhuma no Postgres, então um valor fora da lista passa
direto num insert cru ou numa migration futura. `pgEnum` gera um tipo
`ENUM` de verdade no Postgres (`CREATE TYPE ... AS ENUM (...)`), com
constraint real — confirmado via Context7 contra a doc oficial do
Drizzle.
**Alternativas consideradas**: manter `text`/`varchar` + `{ enum: [...]
}`, aceitando que a validação fica só no Zod (`inputSchema`) da camada
HTTP (rejeitado — não protege contra escrita fora da API, como seed,
migration manual, ou acesso direto ao banco em debug; `pgEnum` cobre
esse caso sem custo extra).

### drizzle-kit push proibido fora de ambiente local, não só em produção

**Decisão**: a regra não-negociável do `CLAUDE.md` passa a dizer "nunca
`drizzle-kit push` fora de ambiente local (staging, CI e produção
incluídos)", alinhando com "## Banco de dados" — que já dizia "fora de
ambiente local".
**Contexto**: `CLAUDE.md` proibia `push` só "em produção", enquanto
`docs/architecture.md` proibia "fora de ambiente local". Com banco
hospedado, ambientes intermediários (staging, branch de banco por PR)
também têm dado que não pode ser alterado por diff de schema sem
migration versionada; deixar essa lacuna entre as duas frases abria
exatamente a exceção silenciosa que `../../docs/versioning.md` quer evitar.
**Alternativas consideradas**: afrouxar `docs/architecture.md` para
"em produção" (rejeitado — `push` compara o schema com o banco e aplica
o diff direto, sem histórico de migration em nenhum ambiente que não
seja descartável).

### Neon: duas connection strings (pooled no runtime, direta nas migrations)

**Decisão**: o alvo de produção da baseline é Postgres no Neon. O runtime
usa a URL **pooled** (`DATABASE_URL`, host com `-pooler`) com
`prepare: false` no `postgres.js`; migrations e `drizzle-kit` usam a URL
**direta** (`DATABASE_URL_UNPOOLED`). Ver "### Neon".
**Contexto**: a baseline nunca mencionava o Neon, mas é onde a API vai
rodar. O pooler do Neon é PgBouncer em modo transação: não suporta
`PREPARE` de SQL nem recursos de sessão que ferramentas de migration usam
— a doc do Neon manda usar a conexão direta para migrations e a pooled
para o runtime. Confirmado via Context7 contra a doc do Neon e do
`postgres.js` (que manda `prepare: false` para PgBouncer em modo
transação).
**Alternativas consideradas**: usar só a URL direta no runtime
(rejeitado — sem pooler, `réplicas × max` disputa o `max_connections` da
compute); usar só a pooled em tudo (rejeitado — migrations falham ou
ficam frágeis pelo pooler).

### Migrations rodam no CD, com env mínimo — nunca no start do container

**Decisão**: `db:migrate` roda como job do GitHub Actions
(`deploy.yml`, ambiente `production`) antes do deploy da imagem, com a
URL direta do Neon. A imagem de runtime não contém `drizzle-kit`, a
config nem a pasta de migrations. O tooling lê a env por
`lib/env-tooling.ts` (só a URL do banco), não por `lib/env.ts`.
**Contexto**: o Dockerfile só copiava `dist`, então `bun run db:migrate`
não tinha onde rodar em produção. Migrar no start do container faria N
réplicas competirem pela migration e daria ao processo da API uma
credencial de DDL. E o `drizzle.config.ts` importava `lib/env.ts`, que
exige `REDIS_URL`, `BETTER_AUTH_SECRET` etc. — o job de migration teria que
receber segredos que não usa.
**Alternativas consideradas**: script de migration programático dentro da
imagem, rodando no start (rejeitado pelos motivos acima; pode ser
revisitado por instância com lock consultivo); manter `lib/env.ts` único
e passar valores dummy para as variáveis não usadas (rejeitado —
mascara a validação e espalha segredo desnecessário no job).

### Redis gerenciado com TLS em produção

**Decisão**: em produção o Redis é um serviço gerenciado acessado por
`rediss://`; o provedor é escolha da instância. Ver "### Redis em produção".
**Contexto**: o Neon não oferece Redis e a doc só dizia que `REDIS_URL` é
obrigatória, sem dizer onde ele roda. Um container de Redis ao lado da API
não dá persistência nem HA gerenciada, e Redis nunca é fonte de verdade
de qualquer forma.
**Alternativas consideradas**: Redis como sidecar no mesmo host do Docker
(rejeitado como default — sem HA, some junto com o host); deixar sem
recomendação (rejeitado — repete a lacuna que gerou pausa de scaffold no
driver Postgres e no client Redis).

### Health check separado em liveness (`/health`) e readiness (`/ready`)

**Decisão**: `/health` só responde 200 se o processo está de pé, sem tocar
em dependências; `/ready` checa Postgres e Redis (e vira 503 durante o
shutdown). O `HEALTHCHECK` do Dockerfile usa `/health`; o balanceador usa
`/ready`.
**Contexto**: a doc dizia que `/health` "pode opcionalmente checar
Postgres e Redis". Liveness que depende de banco reinicia o container a
cada oscilação — e com Neon em scale-to-zero uma sonda periódica no banco
mantém a compute acordada, com custo.
**Alternativas consideradas**: uma rota só, com checagem opcional
(rejeitado — mistura duas perguntas diferentes: "reinicio este processo?"
e "mando tráfego para ele?").

### Guarda de teste: `bun test` aborta fora de banco/Redis de teste

**Decisão**: `bunfig.toml` com `[test] preload = ["./tests/setup.ts"]`; o
setup aborta a execução se `NODE_ENV !== 'test'`, se o banco não termina
em `_test` ou se o Redis está no índice `0`. Ver `docs/testing.md`.
**Contexto**: `resetDatabase()` faz `truncate ... cascade` e o `afterEach`
faz `flushdb()`. Variável de ambiente exportada no shell vence qualquer
`.env*` (confirmado na doc/testes do Bun), então um shell com a
`DATABASE_URL` do Neon exportada faria `bun test` apagar dados reais —
risco que passa a existir de verdade agora que há um banco hospedado.
**Alternativas consideradas**: só documentar o aviso (rejeitado — depende
de lembrar, e o custo do erro é perda de dado).

### Multi-tenant obrigatório: banco único com `organization_id`

**Decisão**: a baseline é **somente multi-tenant** — não existe modo
single-tenant nem tenant opcional. O isolamento é por linha, num banco e
schema únicos: toda tabela de negócio tem `organization_id NOT NULL` (FK
`organizations`), todo acesso passa por `inTenant(table, ctx, …)`, FK entre
tabelas de tenant é composta, e o recurso de outro tenant responde 404.
Todo `<action>.ts`/`<action>.service.ts` recebe um `TenantContext`. Ver
"### Multi-tenancy: isolamento por `organization_id`".
**Contexto**: pedido do usuário — a baseline foi desenhada para uma API
single-tenant, mas o uso pretendido é multi-tenant. Isso inverte a decisão
"Single-tenant por design" e a regra do `CLAUDE.md` que proibia modelar
tenant. Por `../../docs/versioning.md` é uma mudança que substitui regra
não-negociável (MAJOR; pré-1.0, vira MINOR marcada como estrutural).
**Alternativas consideradas**: RLS do Postgres como segunda barreira
(rejeitado por ora — a doc do Drizzle mostra o padrão com `set_config` por
transação, o mesmo problema que já derrubou o teste por transação: os
services importam o `db` singleton, sem repository e sem `tx` por
parâmetro; e complica o pooler do Neon. Continua possível como
endurecimento futuro por instância); um banco ou branch Neon por tenant
(rejeitado — provisionamento, migrations em N bancos e roteamento de
conexão por request são custo operacional grande demais para uma baseline);
tenant opcional (rejeitado — dois modos duplicam docs, testes e
superfície de erro).
**Consequências assumidas**: sem RLS, a segurança do isolamento depende do
helper `inTenant` e dos testes de isolamento (por isso obrigatórios); o
restore *point-in-time* do Neon é do banco todo, não de um tenant; tenants
dividem a mesma compute.

### Tenant é a organização do Better Auth, ativa na sessão

**Decisão**: tenant = organização do plugin `organization` do Better Auth;
a organização ativa vive em `sessions.active_organization_id`. O macro
`tenant: true` exige sessão, organização ativa e membership, e expõe o
`TenantContext`; `auth: true` fica só para o que não é de um tenant. Criar
organização, convite e membros são endpoints do plugin, não features da
baseline. Ver "### Tenant: organização ativa e contexto".
**Contexto**: o plugin já entrega organizações, membros, convites, roles
`owner`/`admin`/`member` e a org ativa na sessão (confirmado via Context7);
reimplementar isso seria manter uma segunda versão do mesmo modelo.
**Alternativas consideradas**: id da organização na URL
(`/orgs/:orgId/…`, rejeitado por ora — é stateless e funciona com várias
abas, mas todo endpoint carrega o parâmetro; a sessão foi a escolha do
usuário); header `X-Organization-Id` (rejeitado — implícito, fácil de
esquecer no cliente e exige tratamento no CORS); tabela `tenants` própria
sem o plugin (rejeitado — duplica o que o Better Auth já modela).

### Roles por módulo na organização; dono por flag `isOwner`

**Decisão**: `user_module_roles(organization_id, user_id, module, role)`,
`role` ∈ `user | editor | manager | admin` (`pgEnum`). O dono da organização
é identificado por uma flag booleana **gerada**, `members.is_owner = (role =
'owner')`, com índice único parcial garantindo **um** dono por organização;
`resolveRole` trata `isOwner` como `admin` em qualquer módulo, sem linha em
`user_module_roles`. `members.role` (do plugin Better Auth) fica restrito a
`owner`/`member` — é encanaria de administração da organização (convite,
remoção, exclusão), nunca lido pela aplicação para autorizar features de
negócio. `user_global_roles` e o bootstrap do primeiro admin deixam de
existir: quem cria a organização vira o dono. Nenhuma role atravessa
organizações. Ver "### Tenant: organização ativa e contexto" e "###
Autorização por role, por módulo".
**Contexto**: pedido do usuário, revisando o design inicial desta baseline
multi-tenant (que mapeava `owner`/`admin` do plugin direto para a role
`admin` de negócio). Separar os dois planos evita que uma mudança no
vocabulário de roles do plugin (o `admin` embutido dele) altere sem querer
quem é `admin` de módulo na aplicação, e a flag booleana torna "é o dono"
uma pergunta de uma coluna, sem repetir a string `'owner'` pela base de
código. Validado de ponta a ponta (Better Auth + Drizzle + Postgres reais,
scaffold descartável): coluna gerada rejeita escrita direta, o índice único
barra um segundo dono, os hooks de `lib/auth.ts` bloqueiam promover alguém a
`owner`/`admin` do plugin por convite ou por `updateMemberRole`, e
`transfer-ownership` move a propriedade dentro de uma transação.
**Alternativas consideradas**: usar `members.role` diretamente como a role
de negócio (rejeitado — é o pedido do usuário: role de negócio tem 4
valores por módulo, e a administração da organização precisa de um
vocabulário próprio e menor); permitir mais de um dono (rejeitado pela
resposta do usuário — um dono simplifica exclusão da organização e
transferência de propriedade, sem ambiguidade de "qual dono decide");
`is_owner` como coluna gravada, mantida em sincronia por um `CHECK
((role = 'owner') = is_owner)` e por um hook `afterCreateOrganization`
(rejeitado — o plugin insere o criador com `role = 'owner'` antes de
qualquer hook, e a flag entraria com o default `false`, violando o `CHECK`
na própria criação da organização; a coluna gerada dispensa hook e nunca
diverge); `isOwner` independente da role do membro (rejeitado — permitiria um dono com
role `user`/`member` sem poder administrar a própria organização, o mesmo
tipo de estado ambíguo que a decisão "Admin como role global" já tinha
identificado); manter um "super admin" da plataforma em `user_global_roles`
(rejeitado — brecha de isolamento de propósito, exigiria auditoria própria
de cada acesso; operação de plataforma fica fora da API).

## Diferenças no monorepo

- **Estrutura**: este app é o workspace `@repo/api` em `apps/api`. A
  árvore de `src/` acima vale como está.
- **Contrato compartilhado** (`../../docs/architecture.md`,
  "### `packages/contracts`"):
  - `permissions`, `Role`, `Action` e `can` são importados de
    `@repo/contracts/permissions`. `lib/permissions.ts` da API mantém só
    `resolveRole` (que consulta o banco), sem reexportar a matriz.
  - O formato de erro `{ statusCode, error, message, issues? }` é o
    `apiErrorSchema` de `@repo/contracts/errors`, e o error handler
    responde nesse formato.
  - Enum que aparece na borda da API nasce como array `as const` no
    contracts e é consumido pelo `pgEnum`:

    ```ts
    // apps/api/src/db/schema/roles.ts
    import { moduleRoles } from '@repo/contracts/roles'
    import { pgEnum } from 'drizzle-orm/pg-core'

    export const moduleRole = pgEnum('module_role', moduleRoles)
    ```

    A regra "nunca duplicar o array" continua valendo. A fonte única
    passa a ser o contracts, e `moduleRole.enumValues` continua
    disponível dentro da API.
  - `outputSchema` das features usa os schemas de resposta de
    `@repo/contracts/<modulo>`. O `inputSchema` parte do schema de
    request do contracts e pode acrescentar regra só da API.
- **Neon**: local usa o Postgres do Docker (`app_dev`/`app_test`). CI,
  staging e produção são branches do mesmo projeto Neon
  (`../../docs/neon.md`). As regras de "### Neon" (pooled/unpooled,
  `prepare: false`, sem `channel_binding`) valem para todos.
- **Env**: arquivos em `apps/api/` (`.env.local`, `.env.test`,
  `.env.example`). Variáveis lidas em `test`/`db:migrate` são declaradas
  em `env` da task no `turbo.json`.
- **`tsconfig.json`**: estende `@repo/tsconfig/bun.json` e mantém
  `paths` `@/*` → `./src/*` aqui (sem `baseUrl`, como antes).
- **`TRUSTED_ORIGINS`**: local, `http://localhost:3000` (o web do mesmo
  repo). Staging, o domínio de preview fixo da Vercel. Produção, o
  domínio do web.
