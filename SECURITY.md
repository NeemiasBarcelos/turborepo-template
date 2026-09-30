# Security Policy

Este repositório é um template base: não roda em produção diretamente, mas
instâncias derivadas dele rodam.

## Escopo

Cobre falhas nos **padrões que o template prescreve**, por exemplo:

- exemplo de código inseguro em qualquer `docs/*.md` (raiz ou apps);
- regra não-negociável que deixa brecha;
- configuração insegura sugerida:
  - workflows do GitHub Actions (permissões, secrets expostos em log,
    PR de fork com acesso a secrets);
  - uso do `neonctl` e dos branches do Neon (vazamento de connection
    string, cópia de produção acessível além do necessário);
  - Dockerfile, rewrite web → API, `TRUSTED_ORIGINS`, Vercel ou host da
    API.

**Fora de escopo**: bugs introduzidos por uma instância derivada, falhas
nos templates de origem (app-nextjs, api-bun: reportar no repositório de
cada um, salvo quando a falha só existe na adaptação feita aqui) e
vulnerabilidades em dependências de terceiros (Next.js, Bun, Elysia,
Better Auth, Turborepo, neonctl etc.; reportar ao projeto
correspondente).

## Como reportar

Não abrir issue pública. Preferir, nesta ordem:

1. GitHub Security Advisories (aba "Security" → "Report a
   vulnerability"), que mantém o relato privado até haver correção.
2. Se a opção acima não estiver disponível, contatar diretamente o
   mantenedor.

Incluir: qual arquivo/regra do template está envolvido, o cenário
concreto em que o padrão prescrito gera risco e como uma instância que
siga o template ao pé da letra seria afetada.

## Versões suportadas

Apenas a versão mais recente da baseline (`docs/CHANGELOG.md`) recebe
correção. Instância derivada de baseline antiga atualiza manualmente
(`docs/versioning.md`).

## Prazo de resposta

Sem SLA formal: projeto mantido por uma pessoa. A correção entra no fluxo
normal de versionamento da baseline, com registro em `docs/CHANGELOG.md`.
