---
id: stores
title: "Lojas suportadas e fixtures"
type: store-catalog
status: proposed
consult_when: [add-store-adapter, change-capture]
applies_to: ["src/adapters/**", "manifest.config.*"]
related: [contracts, entities, nfr, feat-auto-capture, agentflow-architecture:adr-0003]
tags: [lojas, fixtures, permissoes]
---

# Lojas suportadas e fixtures

## Objetivo

Fechar a lista de lojas do MVP, os hosts que a extensão pode ler sozinha e o que cada adaptador precisa provar com fixtures.

## Lojas do MVP

Cada linha é um adaptador em `src/adapters/<id>/`, registrado em `src/adapters/registry.ts`. `matches(url)` é verdadeiro só quando o eTLD+1 da URL é o domínio da linha.

| Loja | Id do adaptador | eTLD+1 | `host_permissions` |
| ---- | --------------- | ------ | ------------------ |
| Magazine Luiza | `magazine-luiza` | `magazineluiza.com.br` | `https://*.magazineluiza.com.br/*` |
| Shein | `shein` | `shein.com` | `https://*.shein.com/*` |
| Shopee | `shopee` | `shopee.com.br` | `https://*.shopee.com.br/*` |
| Mercado Livre | `mercado-livre` | `mercadolivre.com.br` | `https://*.mercadolivre.com.br/*` |
| Amazon Brasil | `amazon-com-br` | `amazon.com.br` | `https://*.amazon.com.br/*` |

A captura automática só roda nesses hosts. Eles já estão em `host_permissions`, então a captura manual neles não pede permissão de novo.

`optional_host_permissions` contém apenas `https://*/*`. Esse padrão é pedido na primeira captura manual num site fora da tabela. A concessão não liga a captura automática fora das cinco lojas.

O manifest pede também `storage`, `activeTab` e `scripting`, e nada além disso (`NFR-PRIV-003`).

## Adaptador genérico

O id é `generic`. Ele só entra quando nenhum adaptador específico deu `matches(url)`, ou quando o específico retornou `status: "unhandled"`.

Ordem das fontes, a primeira que tiver o campo ganha: JSON-LD `schema.org/Product` (e `Offer`), Open Graph (`og:title`, `og:image`, `product:price:amount`, `product:price:currency`) e microdata.

Limites:

- Não tem meta de cobertura. Um site que ele não entende falha com um `reason` de `contracts` e a tarefa está correta.
- Não emite `variant_not_selected`. Se não conseguir ler opções, manda `variant: {}`.
- Se só houver faixa ("a partir de", mínimo e máximo) e nenhum preço único, falha com `price_not_found`.
- Não lê o preço riscado quando existe um preço atual.

## Fixtures

Cada um dos cinco adaptadores versiona, junto do código, pelo menos três HTML anonimizados (sem cookie, sem dado de usuário):

| Arquivo | O que a página contém | Resultado exigido |
| ------- | --------------------- | ----------------- |
| `simple.html` | Um produto, sem seletor de variação, um preço, ao menos uma imagem. | `status: "ok"`, `variant` vazio, `priceText` igual ao preço visível. |
| `variant.html` | Ao menos uma opção selecionada e o preço dessa opção. | `status: "ok"` e `variant` com essa opção. O mesmo HTML com a seleção removida retorna `variant_not_selected`. |
| `promo.html` | Preço riscado e preço atual. | `priceText` é o preço atual, nunca o riscado. |

Ao lado de cada HTML há o resultado esperado: nome, `amount`, `currency`, `variant` e a contagem de imagens (mínimo 1). O teste de fixture usa o HTML estático e não espera os 2 s de `NFR-PERF-002`. A espera de 2 s vale na página ao vivo, quando o preço ainda não entrou no DOM.

Shein e Shopee: o HTML de fixture é o DOM depois da hidratação, não o documento vazio que o servidor entrega.

## Questões em aberto

- Confirmar os cinco padrões de host. Pontos ainda não fechados: incluir `https://*.magalu.com/*`; restringir a Shein a `br.shein.com`; incluir `https://*.mercadolivre.com/*`.

## Regras para agentes

- DEVE usar os ids da tabela. Não crie um segundo id para a mesma loja.
- DEVE entregar as três fixtures e o caso sem seleção em `variant.html` no mesmo PR que cria o adaptador.
- NÃO DEVE alargar `host_permissions` para um host fora da tabela sem atualizar este documento.
- NÃO DEVE fazer a captura automática depender de `optional_host_permissions`.
- DEVE tratar o `generic` como melhor esforço: não reprove uma tarefa porque ele falhou numa loja fora da tabela.
