# Domínio

> **Por instância.** Vazio no template, de propósito. Cada produto
> derivado preenche com o seu domínio e as suas decisões de infra. Um
> único arquivo para web e API: no monorepo não existem mais dois
> `domain.md`. Não inventar conteúdo aqui a partir do template.

## Visão geral do produto

<!-- Preencher: o que o produto faz, para quem, principais fluxos. -->

## Glossário

<!-- Preencher: termos de negócio, como aparecem na UI (rótulo) e como se
chamam no código/API/banco. -->

## Módulos

<!-- Preencher: um item por módulo, com o mesmo nome em
apps/api/src/modules, apps/web/src/features e packages/contracts. Para
cada um: entidades, rotas principais da API, telas do web e quem (qual
role) faz o quê. -->

## Regras de negócio

<!-- Preencher: invariantes do domínio que a API garante. -->

## Decisões de produto

<!-- Preencher: métodos de login, quem cria organização, fluxo de convite,
idioma(s), tema, error tracking/analytics. -->

## Infraestrutura desta instância

<!-- Preencher:
- Neon: id do projeto, região, plano (limites de branch/restore window),
  se o branch de CI usa --schema-only (e por quê).
- Host da API: qual (DigitalOcean/Railway/Render/…), nomes dos serviços
  api-staging e api-production, como o job deploy do api-deploy.yml
  aciona o host.
- Redis: provedor e região.
- Vercel: projeto, domínio de produção, domínio de preview fixo usado em
  TRUSTED_ORIGINS de staging.
- Domínios: web e API em produção e staging. -->

## Fora de escopo

<!-- Preencher. -->
