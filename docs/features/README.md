# Docs de feature

Documentação de feature individual vive aqui, um arquivo por feature
complexa: `docs/features/<nome>.md`. **Um doc só para os dois lados**:
no monorepo não existe doc de feature separado para web e API. A maioria
das features não precisa de doc. Schemas do `contracts`, nomes claros e
testes já contam boa parte da história.

## Quando criar um doc de feature

Criar `docs/features/<nome>.md` quando a feature tiver pelo menos um
destes:

- Regra de negócio com várias condições ou máquina de estados.
- Fluxo de várias etapas (wizard, onboarding, checkout) com estado entre
  telas.
- Integração com serviço externo (pagamento, e-mail, storage, webhook).
- Processamento assíncrono, idempotência ou concorrência.
- Comportamento de interface não óbvio (atualização otimista com
  rollback, polling, edição concorrente) ou uso de zustand.
- Migration em várias etapas (*expand/contract* espalhado por PRs).
- Edge case que não cabe num comentário de uma linha.

Se nada disso se aplica, não criar o arquivo: doc sem motivo concreto só
vira mais uma coisa desatualizada.

## Convenção de nome

`docs/features/<nome>.md` em kebab-case: o nome do fluxo
(`org-onboarding.md`) ou do módulo quando a regra atravessa várias
partes dele (`tasks-bulk-edit.md`).

## Molde

Usar `docs/features/_TEMPLATE.md` como ponto de partida.
