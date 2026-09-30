# Git workflow

Mesmo fluxo dos templates app-nextjs e api-bun, num repositório só.

## Remote ao derivar do template

Clonar o template (`turborepo-template`) preserva o remote `origin`
apontando para o repositório do template. Antes do primeiro commit de
uma instância, confirmar com `git remote -v`. Nunca commitar nem dar push
com `origin` ainda apontando para o template. Resolver de um dos dois
jeitos, perguntando ao usuário qual prefere se não estiver claro:

- Criar um repositório novo no GitHub para a instância e trocar o remote:
  `git remote set-url origin <novo-url>`.
- Manter o template como remoto adicional (`git remote add template
  <url-do-template>`) e apontar `origin` para um repositório novo.

Vale o mesmo para as integrações: o projeto da Vercel, o host da API e o
projeto Neon são **da instância**, nunca compartilhados com o template ou
com outra instância.

## Branching

Trunk-based: `main` é sempre deployável. A Vercel publica o web a partir
dela e o `api-deploy.yml` publica a API. Sem `develop` nem `release/*`.

Branch curta a partir de `main`, nomeada `<tipo>/<descricao-curta>`:

```
feat/tasks-bulk-actions
fix/api-org-switch-cache
chore/upgrade-turbo
docs/fill-domain-doc
```

A branch morre depois do merge. Cada PR ganha um branch Neon
`ci/pr-<n>`, apagado quando a PR fecha (`docs/neon.md`).

## Commits

[Conventional Commits](https://www.conventionalcommits.org/):
`<tipo>(<escopo>): <descrição>`, validado pelo hook `commit-msg` (Husky +
commitlint). Nunca `git commit --no-verify`.

| Tipo | Quando usar |
|---|---|
| `feat` | nova capacidade (tela, endpoint, fluxo) |
| `fix` | correção de bug |
| `refactor` | mudança de código sem mudar comportamento |
| `perf` | melhoria de performance |
| `style` | só visual/CSS no web, sem mudar comportamento |
| `test` | adiciona/ajusta teste |
| `docs` | só documentação |
| `chore` | deps, config, tooling |
| `ci` | pipeline de CI/CD |

Escopo, em ordem de preferência:

1. **Módulo de negócio**, quando a mudança é de um módulo (mesmo nome em
   api, web e contracts): `feat(tasks): …`, mesmo que toque os dois apps.
2. **Workspace**, quando é transversal dentro de um app ou pacote: `web`,
   `api`, `contracts`.
3. **Área de infra**: `neon`, `ci`, `deps`, `turbo`, `docker`, `auth`.

```
feat(tasks): add bulk status change
fix(api): return 404 for other tenant on task update
refactor(contracts): split error schema from permissions
ci(neon): renew ci branch expiration on synchronize
chore(deps): bump next to 16.4
```

## Pull Request

PR é obrigatório mesmo solo. É o gate do CI, gera o preview da Vercel e o
comentário de schema diff do Neon.

- Título em Conventional Commits (vira o commit de squash).
- Descrição com o que mudou, por quê e, para mudança visual, print ou
  link do preview.
- **Mudou `apps/api` ou `packages/contracts`?** Responder na descrição as
  duas perguntas de compatibilidade de `docs/deploy.md` ("## Ordem de
  deploy entre web e API").
- **Tem migration?** Conferir o comentário do schema diff do Neon e se a
  migration é *expand/contract*.
- CI verde antes do merge.
- Mudança em "Regras não-negociáveis" de qualquer `CLAUDE.md` segue
  `docs/versioning.md`. Não é PR comum.

## Merge

Sempre **squash merge**: um commit por PR em `main`. Branch apagada
automaticamente. Nunca merge commit nem rebase merge.

## Husky

```bash
# .husky/pre-commit
bun run lint && bunx turbo run typecheck --affected
```

```bash
# .husky/commit-msg
bunx commitlint --edit "$1"
```

```js
// commitlint.config.mjs
export default {
	extends: ['@commitlint/config-conventional'],
	rules: {
		'type-enum': [
			2,
			'always',
			['feat', 'fix', 'refactor', 'perf', 'style', 'test', 'docs', 'chore', 'ci', 'revert'],
		],
	},
}
```

- Instalação (no scaffold), na raiz: `bun add -d husky @commitlint/cli
  @commitlint/config-conventional && bunx husky init`. O `husky init`
  cria o script `prepare`, e o `bun install` roda o `prepare` da raiz.
- `typecheck --affected` compara com `main`. Ele fica rápido em branch
  pequena e não pula nada que a PR mudou.
- Um `.husky/` só, na raiz. Os apps não têm hooks próprios.

## Branch protection (GitHub)

Proteger `main` exigindo os checks de `docs/ci-cd.md` ("## Branch
protection"). O `pr-title` é obrigatório porque o título vira o commit em
`main` e o hook local só valida os commits da branch.

Settings → General → Pull Requests:

- Permitir **só** *squash merging*.
- **Default commit message**: *Pull request title*.
- Marcar *Automatically delete head branches*.

Settings → Environments:

- `staging`: sem aprovação.
- `production`: *required reviewers* (mesmo solo, é o "botão" de
  promover a API) e restrito à branch `main`.
