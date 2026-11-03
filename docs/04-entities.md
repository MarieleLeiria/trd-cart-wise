---
id: entities
title: "Regras de negócio das entidades"
type: entity-rules
status: proposed
consult_when: [change-data-model, change-capture, change-storage]
applies_to: ["src/shared/**", "src/storage/**", "src/background/**"]
related: [contracts, stores, feat-auto-capture, feat-item-list, feat-remove-item, agentflow-architecture:data-model, agentflow-architecture:adr-0004]
tags: [entidades, dedupe, variante, preco]
---

# Regras de negócio das entidades

## Objetivo

Dizer como o service worker transforma um `CapturedProduct` em `SavedItem`, de um jeito que dois agentes produzam a mesma `dedupeKey`. Os campos e os índices estão em `agentflow-architecture:data-model`. Não há histórico de preço.

## Regras de negócio

### Preço e moeda

O adaptador coloca em `priceText` só o preço atual. Preço riscado, "de" e "a partir de" não entram. Faixa sem preço único é `price_not_found` no adaptador.

O service worker resolve a moeda nesta ordem:

1. `currencyHint` com três letras, se o código estiver na lista ISO 4217 empacotada na extensão.
2. Símbolo inequívoco em `priceText`: `R$` → `BRL`, `US$` → `USD`, `€` → `EUR`, `£` → `GBP`.
3. Qualquer outro caso, inclusive `$` sozinho, `¥` e a localidade do navegador: `currency_unknown`.

O expoente da unidade menor é 2, exceto:

- expoente 0: `JPY`, `KRW`, `CLP`, `VND`;
- expoente 3: `BHD`, `KWD`, `OMR`.

A lista de códigos válidos vem no repositório. Não há consulta de rede para obtê-la.

Para o valor:

1. Remova símbolos e letras de `priceText`.
2. Se sobrarem `.` e `,`, o último é o separador decimal.
3. Se sobrar só um, ele é decimal quando é seguido de exatamente `expoente` dígitos no fim; senão é separador de milhar.
4. Monte o inteiro: parte inteira vezes `10^expoente`, mais a parte fracionária ajustada ao expoente.
5. Mais casas do que o expoente, texto vazio ou valor negativo viram `invalid_payload`. Zero é válido.

| Entrada | Moeda | `amount` |
| ------- | ----- | -------- |
| `R$ 1.299,90` | `BRL` | `129990` |
| `R$ 10` | `BRL` | `1000` |
| `$1,299.90` com hint `USD` | `USD` | `129990` |
| `1,299` com hint `JPY` | `JPY` | `1299` |
| `1.234` com hint `KWD` | `KWD` | `1234` |

### Nome, imagens e loja

- Nome: sem tags, entidades HTML decodificadas, trim, espaços internos colapsados, no máximo 300 caracteres. Vazio depois disso é `name_not_found`.
- Imagens: absolutas contra `pageUrl`, só `https`, sem repetir, ordem da página, no máximo 10. Zero depois do filtro é `no_image`.
- `storeName`: trim, no máximo 80 caracteres, ou `null`.
- `storeDomain`: eTLD+1 pela lista de sufixos públicos. `amazon.com.br` permanece `amazon.com.br`. `br.shein.com` vira `shein.com`. Não recorte o host no último ponto.

### URL

`productUrl` começa em `canonicalUrl` quando ela é `https` e não vazia; senão, em `pageUrl`. Em seguida:

- host em minúsculas, sem porta padrão;
- sem fragmento;
- sem estes parâmetros de rastreamento, comparação sem distinção de maiúsculas, e sem qualquer chave com prefixo `utm_`: `gclid`, `fbclid`, `msclkid`, `srsltid`, `mc_cid`, `mc_eid`, `gbraid`, `wbraid`, `yclid`, `twclid`.

Os outros parâmetros ficam. A variante não é inferida da query string: ela só entra por `variant`, preenchido pelo adaptador.

### Ciclo de vida

1. Primeira vez daquele `dedupeKey`: cria `SavedItem` com UUID v4 novo. `capturedAt` e `updatedAt` são o mesmo instante.
2. Mesmo `dedupeKey`: atualiza nome, imagens, preço, `storeName`, `variant`, `source` e `updatedAt`. Substitui o preço. Não muda `id`, `dedupeKey` nem `capturedAt`. A resposta leva `created: false`.
3. Mesma `productUrl` com outra `variantKey`: outro `SavedItem`.
4. Remoção: apaga o registro. Não há soft delete. Uma captura posterior do mesmo `dedupeKey` recebe outro `id`.

## Identidade e variantes

`variant` chega do adaptador. O service worker normaliza antes de serializar:

- chave e valor: trim, minúsculas, espaços internos colapsados;
- descarta valor vazio;
- `variantKey`: chaves em ordem UTF-16; `encodeURIComponent` na chave e no valor; pares `chave=valor` unidos por vírgula;
- `variant` vazio produz `variantKey` `""`.

`dedupeKey` é `storeDomain`, `productUrl` e `variantKey` unidos por `|`. Se uma parte contiver `|`, troque por `%7C` nessa parte antes de unir.

Exemplo: `{ Color: "Azul", Size: "M" }` produz a chave `color=azul,size=m` e não depende da ordem em que o adaptador montou o objeto.

Quando a página exige uma opção e nenhuma está selecionada, o adaptador específico falha com `variant_not_selected`. Não manda `variant: {}` nesse caso. `{}` significa "este produto não tem variação", não "o usuário ainda não escolheu".

## Questões em aberto

- Nenhuma. A lista de parâmetros de rastreamento só cresce por uma mudança deste documento.

## Regras para agentes

- DEVE calcular preço, `productUrl`, `variantKey` e `dedupeKey` só no service worker.
- NÃO DEVE inferir moeda pela localidade do navegador nem mapear `$` ou `¥` sem hint.
- NÃO DEVE adicionar `priceHistory` ou campo equivalente.
- DEVE usar a lista de sufixos públicos para `storeDomain`. Recortar o host na mão não cumpre a regra.
- DEVE tratar outra `variantKey` como outro item, mesmo com o mesmo nome e a mesma URL.
- DEVE atualizar este documento na mesma mudança que alterar a lista de parâmetros, os expoentes ou o algoritmo de `variantKey`.
