---
id: feat-auto-capture
title: "Captura automática ao adicionar ao carrinho"
type: feature
status: proposed
consult_when: [change-capture]
applies_to: ["src/content/**", "src/adapters/**", "src/background/**"]
related: [nfr, stores, contracts, entities, feat-manual-save, agentflow-architecture:components, agentflow-architecture:data-model]
tags: [captura, clique]
---

# Captura automática ao adicionar ao carrinho

## Objetivo

Quando o usuário clica em adicionar ao carrinho numa loja suportada, o CartWise salva o produto da página sem atrapalhar a loja. A URL não dispara esta feature.

## Escopo

Cobre detecção do clique, escolha do adaptador, extração, normalização, gravação e o feedback de toast e badge. O botão do popup é `feat-manual-save`. A lista é `feat-item-list`.

## Requisitos funcionais

### FR-CAP-001 O clique é o único gatilho

Um clique conta quando o adaptador resolvido responde `isAddToCartTarget` para o alvo do evento ou para um ancestral até `document.body`. Abrir a URL, mudar o DOM do carrinho ou ver uma requisição de rede não capturam.

### FR-CAP-002 A loja segue o fluxo dela

O listener é passivo, na fase de captura, e retorna sem esperar a extração. Não chama `preventDefault`, `stopPropagation` nem `stopImmediatePropagation`.

### FR-CAP-003 A cadeia de adaptadores para na falha definitiva

Entra o adaptador específico cujo `matches(url)` é verdadeiro. Se ele devolver `unhandled`, entra o `generic`. Se devolver `fail`, a cadeia para. Sem específico, entra o `generic`. Os dois `unhandled` viram `adapter_error`.

### FR-CAP-004 A extração traz o produto da opção selecionada

O `CapturedProduct` sai com `trigger: "auto"`, nome, de 1 a 10 imagens, `pageUrl`, `priceText` do preço atual, `variant` e o id do adaptador. Preço riscado não entra. A normalização segue `entities`.

### FR-CAP-005 O prazo depende de o preço já estar no DOM

Com o preço já no DOM, vale `NFR-PERF-001` (500 ms). Sem o preço no DOM, vale `NFR-PERF-002` (2 s) e a falha é `price_not_found`.

### FR-CAP-006 Variação obrigatória não cai no genérico

Se a loja exige opção e nenhuma está selecionada, o adaptador específico falha com `variant_not_selected`. O `generic` não roda.

### FR-CAP-007 Só um produto válido segue para o banco

O content script valida o `CapturedProduct` com Zod e envia `ITEM_CAPTURED`. O service worker valida de novo, normaliza e faz o upsert. Falha antes de existir produto gera `CAPTURE_FAILED`.

### FR-CAP-008 O usuário vê o resultado

Sucesso e falha mostram o toast e o badge definidos em `contracts`. Sucesso de item novo diz "Produto salvo."; recaptura do mesmo `dedupeKey` diz "Produto atualizado.".

### FR-CAP-009 A mesma variação atualiza; outra variação cria

O mesmo `dedupeKey` mantém o `id` e substitui o preço atual. Outra `variantKey` cria outro `SavedItem`.

## Critérios de aceitação

### FR-CAP-001

**Dado** uma página de produto da Amazon Brasil com o botão de adicionar ao carrinho visível
**Quando** o usuário clica nesse botão
**Então** a extensão inicia uma captura com `trigger: "auto"`

**Dado** a mesma página
**Quando** ela termina de carregar, sem clique
**Então** nenhum `ITEM_CAPTURED` é enviado

### FR-CAP-002

**Dado** um listener da loja que adiciona o produto ao carrinho dela
**Quando** o usuário clica em adicionar ao carrinho
**Então** o listener da loja roda, o carrinho da loja muda como sem a extensão, e o listener da extensão não chama `preventDefault` nem `stopPropagation`

### FR-CAP-003

**Dado** uma URL da Magazine Luiza cujo adaptador `magazine-luiza` devolve `status: "unhandled"`
**Quando** o usuário clica em adicionar ao carrinho
**Então** o adaptador `generic` é chamado em seguida

**Dado** a mesma loja e um adaptador que devolve `status: "fail"` com `price_not_found`
**Quando** o usuário clica em adicionar ao carrinho
**Então** o `generic` não é chamado e a falha permanece `price_not_found`

### FR-CAP-004

**Dado** a fixture `promo.html` de uma das cinco lojas, com preço riscado e preço atual
**Quando** o adaptador extrai
**Então** `priceText` é o preço atual, `images` tem ao menos uma URL e `trigger` é `"auto"` na captura por clique

### FR-CAP-005

**Dado** uma página das cinco lojas cujo preço já está no DOM
**Quando** o usuário clica em adicionar ao carrinho
**Então** `ITEM_CAPTURED` é enviado em até 500 ms e o handler síncrono do clique já retornou

**Dado** uma página da Shein cujo preço entra no DOM 1 s depois do clique
**Quando** o usuário clica em adicionar ao carrinho
**Então** a captura espera esse preço e envia `ITEM_CAPTURED` em até 2 s

**Dado** uma página da Shopee que não mostra preço
**Quando** passam 2 s do clique
**Então** a captura falha com `price_not_found` e não espera mais

### FR-CAP-006

**Dado** a fixture `variant.html` de Mercado Livre com a seleção removida
**Quando** o adaptador extrai
**Então** o resultado é `variant_not_selected` e o `generic` não roda

### FR-CAP-007

**Dado** um adaptador que devolve um `CapturedProduct` sem imagem
**Quando** o content script valida
**Então** ele não envia `ITEM_CAPTURED` e envia `CAPTURE_FAILED` com `no_image`

**Dado** um `CapturedProduct` válido cuja moeda o service worker não reconhece
**Quando** o service worker normaliza
**Então** a resposta é `{ ok: false, reason: "currency_unknown" }` e nada é gravado

### FR-CAP-008

**Dado** uma primeira captura bem-sucedida
**Quando** o service worker responde `{ ok: true, created: true }`
**Então** o toast mostra "Produto salvo." e o badge fica `OK` com fundo `#0B7A3B`

**Dado** uma captura que falha com `no_image`
**Quando** a falha é definitiva
**Então** o toast mostra "Não encontramos uma imagem do produto." e o badge fica `!` com fundo `#B42318`

### FR-CAP-009

**Dado** um `SavedItem` de `amazon.com.br` com `variantKey` `size=m`
**Quando** o mesmo produto e a mesma variação são capturados de novo por um preço diferente
**Então** o `id` se mantém, o `amount` passa a ser o novo e não existe lista de preços anteriores

**Dado** esse mesmo item
**Quando** a captura traz `variantKey` `size=g`
**Então** surge um segundo `SavedItem`, e o primeiro permanece

## Restrições

- Cumprir `NFR-PERF-001`, `NFR-PERF-002`, `NFR-PERF-004`, `NFR-PRIV-001` e `NFR-RES-001`.
- Não interceptar rede nem injetar script no mundo `MAIN` (`agentflow-architecture:adr-0003`).
- Não gravar histórico de preço (`entities`).
- O toast obedece `agentflow-architecture:adr-0008`.

## Questões em aberto

- Nenhuma. O prazo de 2 s está em `nfr` e os hosts estão em `stores`.

## Regras para agentes

- DEVE citar o id `FR-CAP-*` no teste que o cobre.
- DEVE manter a extração fora do handler síncrono do clique.
- NÃO DEVE chamar o `generic` depois de `status: "fail"`.
- NÃO DEVE capturar no carregamento da página.
- DEVE rejeitar variação obrigatória sem seleção com `variant_not_selected`, e não com `variant` vazio.
