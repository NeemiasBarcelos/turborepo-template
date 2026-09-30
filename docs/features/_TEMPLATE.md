<!-- Molde para docs/features/<nome>.md. Ver docs/features/README.md para
quando criar um doc de feature. Copiar, renomear e apagar os comentários
de instrução. -->

# <Nome da feature>

## Contexto e objetivo

<!-- Por que essa feature existe, que problema do usuário resolve. -->

## Contrato

<!-- Arquivo em packages/contracts, endpoints (método + rota), schemas de
request/response e erros esperados (status + quando). -->

## Regras de negócio (API)

<!-- Regras, estados e transições, o que é validado onde, efeitos no banco
(tabelas, migrations e etapas de expand/contract). -->

## Fluxo de tela (web)

<!-- Rotas, estados (loading, vazio, erro, sucesso), o que vai para a URL
(nuqs), queries/mutations e chaves invalidadas. -->

## Permissões

<!-- Qual role (por módulo) vê/faz o quê. A API decide, o web só
espelha. -->

## Ordem de deploy

<!-- Se muda o contrato: quais PRs, em que ordem (docs/deploy.md). -->

## Edge cases

<!-- Dado ausente, conflito (409), sessão expirada no meio do fluxo, troca
de organização com a tela aberta, concorrência, idempotência. -->

## Decisões específicas desta feature

<!-- Decisão / Contexto / Alternativas consideradas. -->

## Fora de escopo

<!-- O que essa feature explicitamente não faz. -->
