# Observabilidade

> Baseado em app-nextjs v0.1.0, adaptado ao monorepo. Onde este arquivo
> conflita com a raiz, vale a raiz (`../../CLAUDE.md`) e as seções
> "## Diferenças no monorepo" de `CLAUDE.md` e `docs/architecture.md`.

O front tem menos a observar que a API, mas precisa responder duas
perguntas: **"o usuário viu erro?"** e **"a página está lenta?"**.

## Logging

- **Nunca `console.log` em código de produção.** O Biome sinaliza
  (`noConsole`), e a exceção precisa de `biome-ignore` comentado.
- No server (RSC, server actions, route handlers), erro inesperado sobe
  para o `error.tsx` e é logado pelo runtime da Vercel. Para log com
  contexto, usar `lib/logger.ts`: um wrapper fino que emite **JSON de uma
  linha** (`level`, `msg`, `route`, `organizationId`, `requestId`), lido
  pelos Runtime Logs da Vercel.
- **Nunca logar** cookie, header `Authorization`, token, corpo de
  formulário de login/cadastro nem dado pessoal. `organizationId` e
  `userId` podem; e-mail não.
- No client, nada de log. Erro vai para o `error.tsx`/toast e, se a
  instância tiver, para a ferramenta de error tracking.

## `instrumentation.ts`

`src/instrumentation.ts` é o ponto de entrada de observabilidade no server
(função `register()`, e `onRequestError` para capturar erro de render com
a rota). A baseline deixa o arquivo criado com `onRequestError` enviando
para o `lib/logger.ts`. Integração com um provedor (Sentry, OpenTelemetry
via `@vercel/otel`, …) é decisão de cada instância, registrada em
`../../docs/domain.md`.

Se o api-bun da instância exporta traces OpenTelemetry, usar
`@vercel/otel` no front propaga o `traceparent` nas chamadas do
`serverApi()`, e o trace liga a página à query no banco.

## Métricas de front

- **Vercel Speed Insights** (`@vercel/speed-insights`): Core Web Vitals
  reais por rota. Recomendado em produção.
- **Vercel Analytics** (`@vercel/analytics`): opcional, decisão de produto
  da instância (privacidade/LGPD).

Os dois entram no `app/layout.tsx` e são no-op fora da Vercel.
