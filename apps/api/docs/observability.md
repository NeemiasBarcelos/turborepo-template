# Observabilidade

> Baseado em api-bun v0.16.0, adaptado ao monorepo. Onde este arquivo
> conflita com a raiz, vale a raiz (`../../CLAUDE.md`) e as seções
> "## Diferenças no monorepo" de `CLAUDE.md` e `docs/architecture.md`.

## Logging estruturado

[Pino](https://getpino.io/) como logger. JSON em produção (consumível por
qualquer agregador de log), `pino-pretty` só em `bun dev` (nunca em
produção — formatação legível tem custo de performance que não vale a pena
fora do terminal do desenvolvedor).

```ts
// lib/logger.ts
import pino from 'pino';
import { env } from '@/lib/env';

export const logger = pino({
  level: env.NODE_ENV === 'production' ? 'info' : 'debug',
  redact: {
    paths: [
      'req.headers.authorization',
      'req.headers.cookie', // cookie de sessão do Better Auth
      'res.headers["set-cookie"]',
      '*.password',
      '*.token',
      '*.secret',
      '*.accessToken',
      '*.refreshToken',
      '*.idToken',
    ],
    censor: '[Redacted]',
  },
  transport:
    env.NODE_ENV !== 'production'
      ? { target: 'pino-pretty' }
      : undefined,
});
```

`*` no `redact` cobre **um nível só** (`*.token` pega `body.token`, não
`body.user.token`). Campo sensível mais fundo precisa do caminho
explícito. A sessão do Better Auth viaja em **cookie**, por isso
`cookie`/`set-cookie` estão na lista — só `authorization` não bastaria.

Uso: contexto estruturado primeiro, mensagem depois — nunca string
interpolada com dado dentro:

```ts
// ✅ correto
logger.info({ userId, taskId }, 'task created');

// ❌ não fazer
logger.info(`task ${taskId} created by ${userId}`);
```

Erro vai sempre na chave `err` — o pino só serializa `Error` (message,
stack, type) nessa chave; `{ error }` sai como `{}`:

```ts
// ✅ correto
logger.error({ err: error, taskId }, 'failed to create task');

// ❌ não fazer — o stack se perde
logger.error({ error, taskId }, 'failed to create task');
```

Todo request ganha um `requestId` (gerado no plugin de entrada, propagado
pelo context do Elysia), correlacionando as linhas de log de uma mesma
request e aparecendo também no corpo de erro de resposta quando fizer
sentido para debug. Depois que o macro `tenant: true` resolve o contexto, a
request passa a logar com o tenant — `logger.child({ requestId,
organizationId: ctx.organizationId, userId: ctx.userId })` — de modo que
toda linha de uma request de negócio é filtrável por organização (útil
para suporte e para investigar um suposto vazamento entre tenants). Isso
reforça a regra já existente no `CLAUDE.md`:
nunca `console.log` em código de produção — sempre `logger`. Únicas
exceções: `lib/env.ts` e `lib/env-tooling.ts`, que rodam antes do `logger`
existir (ver `docs/architecture.md`, "Variáveis de ambiente").

## OpenTelemetry

Vendor-neutral via OTLP — o código nunca importa um SDK específico de
vendor (Datadog, New Relic, etc), só exporta para o endpoint configurado:

```ts
// lib/env.ts — já consta no envSchema (docs/architecture.md, "Variáveis de ambiente")
OTEL_EXPORTER_OTLP_ENDPOINT: z.url().optional(),
```

Se a variável não estiver definida, telemetria fica desligada — instância
que não precisa de tracing não paga o custo de configurar um coletor.

**Traces**: um span por request, criado no lifecycle do Elysia (`onRequest`
/ `onResponse`), com spans filhos em operações relevantes de service
(query cara, chamada a serviço externo). Não instrumentar toda função
trivial — span deve marcar uma unidade de trabalho que vale a pena isolar
num gráfico de trace, não cada linha de código. O span da request ganha o
atributo `organization.id` (e `enduser.id` para o usuário) assim que o
tenant é resolvido.

**Métricas básicas**: duração de request por rota, contagem de erro por
tipo (`AppError` subclass), cache hit/miss do Redis (ver
`docs/architecture.md`, seção "## Redis").

**Correlação log ↔ trace**: quando há um span ativo, `traceId`/`spanId`
entram no log estruturado (via o mesmo `requestId`/contexto usado pelo
logger), permitindo pular do log direto para o trace correspondente no
backend de observabilidade escolhido pela instância.
