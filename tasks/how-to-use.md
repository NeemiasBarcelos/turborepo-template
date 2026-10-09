# Como usar o fluxo de tasks

Cada task vive numa pasta própria e passa por comandos do Claude Code, um por etapa. A ideia: entender antes de fatiar, fatiar em tickets pequenos, e cada ticket sair testado e commitado. O contexto fica gravado no `spec.md` para não reler o código a cada sessão (economia de tokens).

## Estrutura

```
tasks/
├── _templates/        modelos (não editar por task)
├── how-to-use.md      este arquivo
└── <ID>/              <slug> da branch <tipo>/<slug>
    ├── spec.md        cabeçalho que liga tudo (branch, workspaces, links, PR) + objetivo, contexto, decisões, questões, fora de escopo
    ├── tickets.md     "Estado atual" no topo + tickets T1..Tn com checkboxes, "Pronto quando:" e commit
    └── log.md         uma entrada curta por ticket + o fechamento da task
```

- **spec.md** responde *o quê, por quê e onde*. O "Contexto" mapeia os arquivos (path:linha) por workspace e as tabelas da task, para os próximos comandos não precisarem procurar de novo. Regras dos `CLAUDE.md` e de `docs/` não se repetem aqui.
- **tickets.md** responde *em que ordem*. Cada ticket tem workspaces, decisões (D#), itens `[ ]/[x]`, "Pronto quando:" e o hash do commit.
- **log.md** responde *o que foi feito e como foi verificado*. Descoberta que muda decisão vai para o spec, não para o log.

`tasks/<ID>/` é **versionada**: entra nos commits da branch e na PR, e fica como histórico da decisão depois do squash.

## Passo a passo

| # | Comando | Quando | O que faz | Escreve |
|---|---------|--------|-----------|---------|
| 0 | `/new-branch [<tipo>/<slug>]` | começo | pergunta o tipo e o slug se não vierem no argumento, derruba o `bun run dev`, atualiza a `main`, cria a branch e sobe `docker compose` + `bun run dev` | git |
| 1 | `/new-task <slug> <título>` | logo depois | cria `tasks/<slug>/` com `spec.md`, `tickets.md` e `log.md`, já com a branch no cabeçalho | spec/tickets/log |
| 2 | `/investigate [ID] [tema]` | antes de fatiar | **primeiro pergunta** do que trata a task; depois lê só os arquivos e tabelas relacionados (só leitura, banco `app_dev`), grava o contexto e faz **uma pergunta por vez** para o que o código não responde | spec.md |
| 3 | `/create-tickets [ID]` | sem questão aberta que mude o fatiamento | fatia a task em tickets pequenos com checkbox; grava só depois do seu ok | tickets.md |
| 4 | `/implement-ticket [ID] [Tn]` | um ticket por sessão | pergunta qual ticket, implementa, testa, roda `lint`/`typecheck`/`test`, escreve o log e commita **depois do seu ok** | código + tickets/log + commit |
| 5 | `/close-task [ID]` | todos os tickets feitos | para o dev, confere os tickets, commita o que sobrou (com ok), roda os gates do CI localmente (incluindo `build --affected` e o `docker build` da API) e abre a PR (com ok) | commit, push, PR, spec/log |
| — | `/adjust <descrição>` | a qualquer momento | classifica o pedido (decisão, escopo novo ou bug), atualiza spec/tickets e só depois mexe no código | spec.md / tickets.md |
| — | `/update-main` | a qualquer momento | volta para a `main` e faz `pull`, avisando de `bun.lock` e migrations novas | git |

**Por que branch e pasta são dois passos:** o `/new-branch` mexe no ambiente (git, processos, containers) e também serve para retomar o ambiente de uma task que já existe. O `/new-task` só cria arquivos em `tasks/`.

`ID` é o `<slug>` da branch `<tipo>/<slug>` (ex.: branch `feat/tasks-bulk-actions` → `tasks/tasks-bulk-actions/`). Sem `ID`, os comandos usam a branch atual. No `/implement-ticket`, `T3` escolhe o ticket; sem ele, o comando pergunta.

**Correção (`fix/*`):** mesmo fluxo e mesmos templates (ex.: `/new-branch fix/api-org-switch-cache` e `/new-task api-org-switch-cache Cache do org switch`). Costuma haver um ticket só.

Depois de cada `/implement-ticket`, faça `/clear` (ou abra outra sessão) antes do próximo.

## Decisões e a coluna "Base"

Toda decisão no spec tem de onde ela veio:

| Base | Exemplo |
|------|---------|
| `usuário` | você respondeu no `/investigate` ou no `/adjust` |
| `código: caminho:linha` | `código: apps/api/src/modules/tasks/list.ts:42` |
| `dado: consulta (banco, data)` | `dado: count(*) where archived_at is null (app_dev, 2026-10-09)` |
| `doc: arquivo §N` | `doc: docs/deploy.md "Ordem de deploy entre web e API"` |

Decisão revista não é apagada: o texto antigo fica riscado (`~~…~~`), o novo entra na mesma linha e a data muda. Os números (D1, D2…) nunca são renumerados.

## Como os tickets são fatiados

- **Pequenos**: cada ticket cabe numa sessão e vira um commit.
- **T1 = fatia fina de ponta a ponta**: o menor caminho que atravessa contracts → api → web e dá para ver funcionando (normalmente a listagem só leitura, com o schema no `contracts`, a rota na API e a tela).
- **Risco em ticket próprio, com testes antes**: migration, isolamento entre tenants, permissões, efeito colateral, concorrência e escrita em lote ganham ticket separado, com testes escritos (e falhando) antes do código: `bun test` na API, Vitest/MSW no web.
- **Mudança incompatível de API ou contrato** vira duas PRs (`docs/deploy.md`, "## Ordem de deploy entre web e API"), então duas tasks.
- **Todo ticket termina com "Pronto quando:"**: um comportamento observável (comando, teste ou URL + resultado esperado). Sem essa evidência, o ticket não é commitado.

## Teste e commit de cada ticket

1. Checagem do "Pronto quando:" (teste ou smoke no browser com `bun run dev`).
2. Na raiz: `bun run lint`, `bun run typecheck` e `turbo run test --filter=<workspace>` nos workspaces tocados.
3. Migration: `bun run db:generate`, revisada (expand/contract) e aplicada só no `app_dev` com seu ok. Nunca `drizzle-kit push`.
4. Escreve a entrada no `log.md`, mostra a mensagem de commit (Conventional Commits, `docs/git-workflow.md`) e commita depois do seu ok. Os hooks do Husky rodam; nunca `--no-verify`. O hash vai para o ticket e para o log no commit seguinte.

## Quando algo muda no meio

Use `/adjust` em vez de pedir a mudança direto:

- **Decisão**: muda ou refina um D# → a linha é revista no spec.
- **Escopo novo**: entra como decisão nova + itens num ticket aberto (ou num ticket novo). Se for grande, vira outra task.
- **Bug**: vira item do ticket aberto, ou um ticket novo `Tn.1: correção` se o ticket já foi commitado.

Se o `/implement-ticket` encontrar algo fora do spec, ele para e sugere `/adjust` em vez de resolver sozinho.

## Regras que valem sempre

- Uma branch `<tipo>/<slug>` por task; os comandos conferem a branch antes de mexer em código.
- Um ticket por sessão; o próximo só começa com seu ok.
- `/investigate` roda **fora do plan mode** (no plan mode ele não consegue gravar o spec).
- Banco: só o local (`app_dev`/`app_test`), só leitura no `/investigate`; qualquer escrita (migration, seed, script) só com confirmação. Nunca apontar para branch Neon.
- Commit e PR só com seu ok; push só no `/close-task`, depois do ok. Merge sempre por squash, com CI verde.
- Regras completas: `CLAUDE.md` da raiz e dos apps, `docs/git-workflow.md`, `docs/checklists.md`.
