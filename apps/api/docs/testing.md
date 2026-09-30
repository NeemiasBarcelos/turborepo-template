# Testes

> Baseado em api-bun v0.16.0, adaptado ao monorepo. Onde este arquivo
> conflita com a raiz, vale a raiz (`../../CLAUDE.md`) e as seções
> "## Diferenças no monorepo" de `CLAUDE.md` e `docs/architecture.md`.

Este documento formaliza e expande a seção "Testes" de
`docs/conventions.md`. `bun test` é o runner (nativo do Bun, sem lib
adicional).

## Filosofia

Pirâmide leve, sem camadas em excesso:

- **Maioria**: teste unitário de `<action>.service.ts` — regra de negócio
  isolada, sem subir servidor HTTP.
- **Parte relevante**: teste de contrato HTTP de `<action>.ts` — sobe a
  instância Elysia em memória e confere status/shape de resposta.
- **Poucos, e só quando justificado**: teste end-to-end cruzando múltiplos
  módulos ou envolvendo infra externa real. Não é o padrão — é exceção
  documentada em `../../docs/features/<nome>.md` quando a feature exigir.

> **Skill opcional**: [`tdd`](https://github.com/mattpocock/skills/blob/main/skills/engineering/tdd/SKILL.md)
> do pacote [mattpocock/skills](https://github.com/mattpocock/skills) —
> operacionaliza o loop red-green-refactor para o teste unitário de
> service e o teste de contrato HTTP que esta seção já exige. Não é parte
> da baseline do template, é uma sugestão de ferramenta.

## Obrigatório vs. opcional

**Obrigatório** (isso é a régua de "toda feature nova precisa de teste
correspondente" do `CLAUDE.md`):

- Todo `<action>.service.ts` com regra de negócio não trivial (validação
  além do schema, cálculo, decisão condicional) tem teste unitário
  cobrindo o caminho feliz e pelo menos um caminho de erro relevante
  (ex: conflito, not found).
- Toda feature exposta via rota (`<action>.ts` registrada em
  `<module>.routes.ts`) tem pelo menos um teste de contrato HTTP com
  caminho feliz e um caso de erro (ex: 400 de validação, 403 de
  autorização quando aplicável).
- Toda feature de negócio tem **teste de isolamento entre tenants**: a
  organização B não lê, altera nem exclui dado da organização A (ver
  "## Isolamento entre tenants"). É a única defesa do isolamento — a
  baseline não usa RLS —, então não é opcional.

**Opcional** — vale a pena, mas não bloqueia PR por ausência:

- Teste de integração cruzando módulos.
- Teste de carga/performance.
- Teste end-to-end completo (banco + auth + rede) fora do already-covered
  por unit + contrato HTTP.

Quando um desses opcionais existir para uma feature específica, documentar
o motivo em `../../docs/features/<nome>.md`.

## Banco de teste

Padrão: **truncar as tabelas entre testes**, no `afterEach` — não
transação por teste com rollback. Os services importam o singleton
`db` (`lib/db.ts`) direto, nunca recebem uma transação como parâmetro
(é o padrão "sem repository" de `docs/architecture.md`), então uma
transação aberta só pelo teste não contém as escritas feitas pelo
service sob teste — elas rodam numa conexão separada e não seriam
desfeitas pelo rollback. Arrange/act/assert usam todos o mesmo `db`
normal; quem garante isolamento entre testes é a limpeza, não uma
transação.

```ts
// tests/helpers/db.ts
import { sql } from 'drizzle-orm';
import { db } from '@/lib/db';
import { organizations, users } from '@/db/schema';

export async function resetDatabase() {
  await db.execute(sql`
    truncate table ${organizations}, ${users}
    restart identity cascade
  `);
}
```

O `cascade` do `truncate` limpa **toda tabela com FK** para as tabelas
citadas. Como toda tabela de negócio tem `organization_id` (FK para
`organizations`), e `members`, `invitations`, `sessions`, `accounts` e
`user_module_roles` apontam para `organizations`/`users`, essa única
instrução esvazia o banco de dados de teste inteiro — sem lista para manter.

```ts
afterEach(async () => {
  await resetDatabase();
});
```

Só uma tabela **sem** FK para `organizations`/`users` (exceção rara, ex: uma
tabela de sistema) precisaria entrar à mão em `resetDatabase()`. Mesmo
princípio já usado em "## Redis em teste" abaixo (`redis.flushdb()` no
`afterEach`) — os dois recursos externos de teste seguem o mesmo
padrão de limpeza pós-teste, não isolamento por transação.

`.env.test` (já descrito em `docs/architecture.md`) aponta para um banco
Postgres de teste dedicado — nunca o banco de desenvolvimento.

### Rodando localmente

```bash
docker compose up -d                 # cria também o app_test (docs/docker.md)
NODE_ENV=test bun run db:migrate     # aplica as migrations no banco de teste
bun test                             # o Bun já define NODE_ENV=test
```

O `bun run db:migrate` sem `NODE_ENV=test` migra o banco de
**desenvolvimento** — o Bun só carrega `.env.test` com `NODE_ENV=test`.

### Guarda contra rodar teste no banco errado

Variável de ambiente já exportada no shell **vence** o `.env.test`
(arquivos `.env*` nunca sobrescrevem `process.env`). Como `resetDatabase()`
faz `truncate` e o `afterEach` faz `flushdb()`, rodar `bun test` num shell
com a `DATABASE_URL` do Neon (ou de qualquer banco real) exportada **apaga
os dados desse ambiente**. Confiar em "lembrar de conferir" não basta: a
baseline exige um *preload* que aborta a execução antes de qualquer teste.

```toml
# bunfig.toml
[run]
bun = true

[test]
preload = ["./tests/setup.ts"]
```

```ts
// tests/setup.ts
import { env } from '@/lib/env';

const database = new URL(env.DATABASE_URL);
const dbName = database.pathname.slice(1);
const redisIndex = new URL(env.REDIS_URL).pathname.slice(1) || '0';

const problems = [
  env.NODE_ENV !== 'test' && `NODE_ENV="${env.NODE_ENV}" (esperado "test")`,
  !dbName.endsWith('_test') &&
    `banco "${dbName}" em ${database.host} (o nome deve terminar em "_test")`,
  redisIndex === '0' && 'Redis no índice 0 (testes usam outro, ex: /1)',
].filter(Boolean);

if (problems.length > 0) {
  console.error('Recusando rodar os testes — apagariam dados reais:');
  for (const problem of problems) console.error(`  - ${problem}`);
  process.exit(1);
}
```

A mensagem imprime só o nome e o host do banco, nunca a URL (que carrega
usuário e senha). Esse arquivo é código de teste, não de produção — o
`console.error` aqui não fere a regra de logger. O `.env.test` e o CI
(`docs/ci-cd.md`) já seguem a convenção que a guarda exige (`app_test`,
Redis em `/1`).

## Redis em teste

Redis real (via `docker-compose.yml` local e via service container no CI —
ver `docs/docker.md` e `docs/ci-cd.md`), nunca mockado. Justificativa:
Redis já é infraestrutura leve e rápida de subir, e mockar cache/rate limit
esconde exatamente os bugs de TTL e invalidação que esse tipo de teste
deveria pegar. Limpar as chaves usadas pelo teste no `afterEach` (ex:
`redis.flushdb()` num Redis de teste isolado, nunca em um Redis
compartilhado com outro ambiente). Localmente o isolamento é por índice
de banco (`REDIS_URL=redis://localhost:6379/1` no `.env.test`): `flushdb()`
limpa só o índice selecionado, então as chaves de dev (índice `0`) ficam
intactas.

## Isolamento entre tenants

Dois tenants reais no banco de teste, criados por um helper que grava a
organização, o usuário e a linha de `members`. O dono entra com `role:
'owner'` — e `is_owner` é gerada pelo banco a partir disso, então o helper
não escreve a flag:

```ts
// tests/helpers/tenants.ts
import { db } from '@/lib/db';
import { members, organizations, users } from '@/db/schema';
import type { TenantContext } from '@/lib/tenant';

/** `isOwner: true` (padrão) = dono da organização; `false` = membro comum. */
export async function createTenant(name: string, { isOwner = true } = {}) {
  const [org] = await db.insert(organizations).values({ name, slug: name }).returning();
  const [user] = await db.insert(users).values({ email: `${name}@test.dev`, name }).returning();
  const [member] = await db
    .insert(members)
    .values({ organizationId: org.id, userId: user.id, role: isOwner ? 'owner' : 'member' })
    .returning();
  const ctx: TenantContext = { userId: user.id, organizationId: org.id, isOwner: member.isOwner };
  return { org, user, ctx };
}
```

Todo `<action>.service.test.ts` de feature de negócio cobre, além do caminho
feliz:

- **Leitura**: a listagem da organização B **não** contém as linhas da A.
- **Acesso por id**: buscar, alterar ou excluir, na organização B, o id de um
  registro da A responde `NotFoundError` (404, nunca 403) — **e o registro da
  A continua intacto**.
- **Unicidade por organização**: o mesmo valor "único" (ex: título) pode
  existir nas duas organizações sem `ConflictError`.
- **Escrita**: o `organization_id` gravado é o do `ctx`, nunca um valor vindo
  do input (o `inputSchema` nem declara esse campo).

```ts
test('organização B não apaga a task da organização A', async () => {
  const a = await createTenant('org-a');
  const b = await createTenant('org-b');
  const project = await createProjectService({ name: 'p' }, a.ctx);
  const task = await createTaskService({ projectId: project.id, title: 'segredo' }, a.ctx);

  await expect(deleteTaskService(task.id, b.ctx)).rejects.toBeInstanceOf(NotFoundError);

  // a task da organização A continua lá
  const still = await db.query.tasks.findFirst({ where: eq(tasks.id, task.id) });
  expect(still).toBeDefined();
});
```

Para testar uma role de módulo, criar um membro comum (`createTenant('x', {
isOwner: false })`) e uma linha em `user_module_roles` **daquela organização**;
para testar o caminho de `admin`, usar o dono (o padrão de `createTenant`) —
nunca um campo na sessão. Um dono numa organização é `admin` em qualquer
módulo, sem linha em `user_module_roles`.

## Mock de sessão (teste de contrato HTTP)

Teste de contrato HTTP de uma feature de negócio não sobe o fluxo de login
real do Better Auth: ele **espia** `auth.api.getSession` e devolve a sessão
desejada. O macro `tenant: true` roda de verdade — inclusive a checagem de
membership na tabela `members` — então o teste exercita o caminho real de
autorização, só sem cookie/login:

```ts
// tests/helpers/mock-session.ts
import { spyOn } from 'bun:test';
import { auth } from '@/lib/auth';
import type { TenantContext } from '@/lib/tenant';

/** Faz o macro real de plugins/auth.ts enxergar este usuário e esta organização ativa.
 *  `null` = sem sessão. Restaurar com `.mockRestore()` no afterEach. */
export function mockSession(ctx: Pick<TenantContext, 'userId' | 'organizationId'> | null) {
  return spyOn(auth.api, 'getSession').mockResolvedValue(
    ctx
      ? ({ user: { id: ctx.userId }, session: { activeOrganizationId: ctx.organizationId } } as never)
      : null,
  );
}
```

```ts
test('DELETE /tasks/:id de outro tenant → 404', async () => {
  const a = await createTenant('org-a');
  const b = await createTenant('org-b');
  // …task criada na organização A…
  const spy = mockSession(b.ctx); // a requisição chega "logada" na organização B
  const res = await app.handle(new Request(`http://localhost/tasks/${task.id}`, { method: 'DELETE' }));
  expect(res.status).toBe(404);
  spy.mockRestore();
});
```

Por que espiar `getSession` e **não** `mock.module` do `plugins/auth`: no
Bun, `mock.module` vale para o processo inteiro de `bun test` e **vaza entre
arquivos** (nem `mock.restore()` o desfaz) — um teste do próprio macro
passaria a ver o plugin falso. `spyOn` numa função de objeto é restaurável.
Também não dá para trocar o plugin registrando outro com o mesmo `name`
antes: a rota registra o seu por conta própria e o real vence.

O macro em si (sem sessão → 401, sem organização ativa → 403, usuário que não
é membro da organização ativa → 403) é testado uma vez, no módulo de auth,
com o mesmo `mockSession`. Isso mantém o teste rápido e focado no contrato da
feature, sem depender de fluxo de login real. Fluxo de auth em si (login,
logout, expiração de sessão, convite) é testado separadamente, no módulo de
auth.

## Cobertura

`bun test --coverage` roda no CI como sinal (visível no resultado do
pipeline), sem gate de percentual mínimo rígido por enquanto — cobertura
baixa em um arquivo específico é um sinal para revisar, não um bloqueio
automático de merge.

## Diferenças no monorepo

- **Local**: `docker compose up -d` na raiz cria `app_dev` e `app_test`.
  Dentro de `apps/api`, `NODE_ENV=test bun run db:migrate` e depois
  `bun test`. Pela raiz, `turbo run test --filter=@repo/api`.
- **CI**: os testes rodam contra o banco `app_test` **dentro do branch
  Neon da PR** (`ci/pr-<n>`), recriado vazio a cada execução e migrado do
  zero. Só o Redis é service container (índice `1`). Sequência completa
  em `../../docs/neon.md`, "## Branch por PR no CI".
- **A guarda de `tests/setup.ts` não muda.** Ela aborta fora de banco
  `_test` e Redis índice `0`, e é exatamente o que impede um `bun test`
  de truncar o banco `app` (cópia de produção) do mesmo branch Neon.
- **E2E do web**: usa outro banco no mesmo branch (`app_e2e_test`) e o
  Redis índice `2`, para nunca disputar dados com estes testes.
- **`packages/contracts`** tem testes próprios (`bun test` no pacote), por
  exemplo para a matriz de `can()`. Os testes da API não repetem esses
  casos, só testam o uso deles (403 na rota).
