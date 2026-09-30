# Versionamento do template

Mesmo mecanismo dos templates `app-nextjs` e `api-bun`, com uma camada a
mais: este template **embute** as baselines dos dois.

## Por que versionar um template

Este repositório é a origem de N produtos. Sem versão, "atualizar uma
regra" vira decisão ad-hoc no meio de uma tarefa, e cada instância
diverge da baseline sem ninguém perceber quando nem por quê. Versionar
torna a divergência uma decisão explícita e rastreável.

## Três baselines, uma versão

```
turborepo-template vX.Y.Z
├── baseline do monorepo      raiz: CLAUDE.md, docs/
├── app-nextjs vA.B.C         apps/web/CLAUDE.md, apps/web/docs/
└── api-bun vD.E.F            apps/api/CLAUDE.md, apps/api/docs/
```

- A versão **deste template** é a única que uma instância rastreia.
- As docs de `apps/web` e `apps/api` são **cópias** das baselines de
  origem, com a versão no cabeçalho (`> Baseado em api-bun v0.16.0`) e as
  adaptações em "## Diferenças no monorepo".
- Cada entrada do `docs/CHANGELOG.md` declara as versões embutidas
  (ex: "Embute app-nextjs `0.1.0` e api-bun `0.16.0`").

### Incorporar uma versão nova de app-nextjs ou api-bun

1. Ler o CHANGELOG do template de origem desde a versão embutida.
2. Copiar os arquivos alterados para `apps/<app>/` e reaplicar as seções
   "## Diferenças no monorepo". Nunca sobrescrever essas seções.
3. Checar se a mudança conflita com uma regra do monorepo (package
   manager, Biome, contrato em `packages/contracts`, testes no Neon). Se
   conflitar, a regra do monorepo vence e o conflito vira item novo em
   "Diferenças no monorepo".
4. Atualizar o cabeçalho `> Baseado em …` e fazer o bump deste template
   (o nível segue a mudança de origem: breaking lá é breaking aqui).

Checklist em `docs/checklists.md`, "## Sincronizar com app-nextjs ou
api-bun".

## O que é baseline travada vs. o que é evolutivo por instância

**Baseline travada**: só muda com bump de versão do template, feito
neste repositório, nunca numa instância:

- "Regras não-negociáveis" do `CLAUDE.md` da raiz e dos `CLAUDE.md` de
  cada app.
- Decisões estruturais de `docs/architecture.md`, `docs/neon.md`,
  `docs/deploy.md`, `docs/ci-cd.md` e das docs dos apps: stack,
  workspaces, contrato compartilhado, modelo de branches do Neon, ordem de
  deploy, pipeline de CI.

**Evolutivo por instância**: cada produto ajusta livremente, sem bump:

- Todo o conteúdo de `docs/domain.md` e `docs/features/*.md`.
- Escolha do host da API e o passo concreto de deploy, registrados em
  `docs/domain.md`.
- Parâmetros dentro de um padrão definido (expiração do branch de CI,
  `--schema-only` por LGPD, região do Neon, tema visual, módulos de
  negócio).

Regra prática: se está dentro de um `<!-- Preencher -->` ou num arquivo
marcado "por instância", é evolutivo. Fora disso, é baseline.

## SemVer da baseline

- **MAJOR**: remove ou substitui uma regra não-negociável, ou troca uma
  peça da stack (ex: sair do Neon, trocar Bun workspaces, trocar a
  Vercel). Também quando embute uma versão breaking de app-nextjs ou
  api-bun. A instância revisa manualmente antes de incorporar.
- **MINOR**: adiciona regra nova, peça nova de stack ou workflow novo sem
  remover nada.
- **PATCH**: correção de texto, exemplo ou wording sem mudar o
  comportamento esperado.

### Convenção pré-1.0

Enquanto a versão for `0.x.y`, um bump que seria MAJOR incrementa o
dígito **MINOR** (ex: `0.4.2` → `0.5.0`). `1.0.0` fica reservado para uma
declaração deliberada de "primeira versão estável". A entrada no
`docs/CHANGELOG.md` precisa marcar explicitamente quando é mudança
estrutural (`### ⚠️ Mudança estrutural (breaking)`), porque o número
sozinho não deixa isso óbvio.

## Processo para propor mudança na baseline

1. A mudança é feita neste repositório, nunca numa instância.
2. Registrar a decisão em "## Decisões registradas" do doc certo
   (`docs/architecture.md`, `docs/neon.md` ou o `architecture.md` do app)
   no formato Decisão / Contexto / Alternativas consideradas.
3. Atualizar o `CLAUDE.md` afetado se mexe em "Regras não-negociáveis".
4. Bump de versão pelo checklist de `docs/checklists.md` ("## Bump de
   versão da baseline"). A versão aparece em `CLAUDE.md` (x2),
   `README.md` e `docs/CHANGELOG.md`.
5. Só depois propagar manualmente para as instâncias existentes (não há
   sincronização automática).

Mudança que também vale para o template de origem (ex: correção na regra
de multi-tenancy) é feita **primeiro** no api-bun ou no app-nextjs e
depois incorporada aqui pelo processo de sincronização. Assim os
templates não divergem.

Nota para quem, humano ou IA, estiver executando uma tarefa comum: mudar
"Regras não-negociáveis" de qualquer `CLAUDE.md` **não é decisão de
tarefa**. Se a tarefa parecer exigir isso, é proposta de mudança de
versão do template. Siga o processo acima e nunca edite a regra
silenciosamente.

## Como uma instância rastreia a versão de origem

Linha no topo do `CLAUDE.md` da raiz da instância:
`> Baseado no template turborepo-template vX.Y.Z`. Os cabeçalhos
`> Baseado em …` dos apps ficam como estão. Convenção manual, sem
tooling.
