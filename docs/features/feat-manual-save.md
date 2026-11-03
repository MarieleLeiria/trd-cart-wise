---
id: feat-manual-save
title: "Salvar o produto da aba atual"
type: feature
status: proposed
consult_when: [change-capture, change-ui]
applies_to: ["src/ui/**", "src/content/**", "src/background/**"]
related: [contracts, stores, entities, nfr, feat-auto-capture, agentflow-architecture:components]
tags: [captura, manual, popup]
---

# Salvar o produto da aba atual

## Objetivo

O usuário salva o produto da aba atual pelo popup, quando o clique em adicionar ao carrinho não foi detectado. A extração é a mesma da captura automática, com `trigger: "manual"`, e não depende do botão da loja.

## Escopo

Cobre o botão do popup, a permissão em site fora das cinco lojas, a injeção do content script e o texto de sucesso ou falha. Não cobre a lista nem a remoção.

## Requisitos funcionais

### FR-MAN-001 O popup oferece o botão

O popup tem o botão "Salvar este produto". Ele fica desabilitado quando a aba ativa não é `https`.

### FR-MAN-002 O popup não lê a loja

O clique envia `CAPTURE_ACTIVE_TAB` com o `tabId` da aba ativa. O popup não lê o DOM da loja nem chama o adaptador.

### FR-MAN-003 Site novo pede permissão antes da captura

Se o host ainda não foi concedido, o service worker pede `https://*/*` e mostra "O CartWise precisa de permissão para ler esta página.". Se o usuário negar, a resposta é `{ ok: false, permission: "denied" }`, o popup mostra "Sem permissão, este site não pode ser salvo." e nada é extraído. Página que não é `https` responde `{ ok: false, permission: "not_https" }` com o texto de `contracts`.

### FR-MAN-004 As cinco lojas não pedem de novo

Em Magazine Luiza, Shein, Shopee, Mercado Livre e Amazon Brasil a captura manual não abre pedido de permissão.

### FR-MAN-005 A extração manual reusa a cadeia

Com permissão, se o content script não estiver na aba, o service worker o injeta com `chrome.scripting.executeScript` e então envia `RUN_CAPTURE` com `{ trigger: "manual" }`. A cadeia de adaptadores, os prazos e a variação obrigatória são os de `feat-auto-capture`. `isAddToCartTarget` não participa.

### FR-MAN-006 O popup e a página mostram o mesmo resultado

No sucesso, o popup mostra o texto de `contracts` e o `name`. Na falha, mostra o texto do `reason`. Quando o content script chegou a rodar, a página também mostra o toast. O badge é atualizado pelo service worker. Negação de permissão não gera toast.

## Critérios de aceitação

### FR-MAN-001

**Dado** o popup aberto numa aba `https` de produto
**Quando** o usuário olha o popup
**Então** o botão "Salvar este produto" está habilitado

**Dado** o popup aberto numa aba `http` ou numa página interna do navegador
**Quando** o usuário olha o popup
**Então** o botão está desabilitado

### FR-MAN-002

**Dado** o popup aberto
**Quando** o usuário clica em "Salvar este produto"
**Então** a UI envia `CAPTURE_ACTIVE_TAB` com o `tabId` e não executa `extract`

### FR-MAN-003

**Dado** uma aba `https` de uma loja fora das cinco, sem permissão concedida
**Quando** o usuário clica em "Salvar este produto" e nega o pedido
**Então** a resposta é `{ ok: false, permission: "denied" }`, o popup mostra "Sem permissão, este site não pode ser salvo." e não há `RUN_CAPTURE`

**Dado** uma aba que não é `https`
**Quando** o botão é acionado por teste, mesmo desabilitado na UI
**Então** a resposta é `{ ok: false, permission: "not_https" }` e o texto é "Só é possível salvar páginas HTTPS."

### FR-MAN-004

**Dado** uma aba de produto em `amazon.com.br`
**Quando** o usuário clica em "Salvar este produto"
**Então** nenhum pedido de `host` aparece e a captura começa

### FR-MAN-005

**Dado** uma aba `https` já permitida, sem content script injetado, com uma variação obrigatória não selecionada
**Quando** o usuário clica em "Salvar este produto"
**Então** o service worker injeta o content script, envia `RUN_CAPTURE` com `trigger: "manual"` e a falha é `variant_not_selected`

**Dado** uma página da Shopee cujo preço só aparece 1 s depois do pedido
**Quando** o usuário salva pelo popup
**Então** a espera segue `NFR-PERF-002` e não depende de um clique na loja

### FR-MAN-006

**Dado** uma captura manual que grava um item novo de nome "Tênis"
**Quando** o service worker responde sucesso com `created: true`
**Então** o popup mostra "Produto salvo." e "Tênis", o toast da página mostra "Produto salvo." e o badge fica `OK`

**Dado** a mesma variação já salva
**Quando** a captura manual sucede com `created: false`
**Então** o popup e o toast mostram "Produto atualizado."

**Dado** uma falha `currency_unknown` depois que o content script rodou
**Quando** a resposta chega
**Então** o popup e o toast mostram "Não foi possível identificar a moeda deste produto." e o badge fica `!`

## Restrições

- A permissão opcional não liga captura automática fora das cinco lojas (`stores`).
- Valem `NFR-PRIV-001`, `NFR-PRIV-003` e os mesmos motivos de `contracts`.
- O popup não grava no IndexedDB. Quem grava é o service worker.

## Questões em aberto

- Nenhuma.

## Regras para agentes

- DEVE reutilizar a extração de `feat-auto-capture` com `trigger: "manual"`. Não crie um segundo extrator para o popup.
- NÃO DEVE ler o DOM da loja a partir de `src/ui/**`.
- DEVE pedir permissão só antes da primeira captura manual num host ainda não concedido.
- NÃO DEVE emitir toast quando a resposta for `permission`.
- DEVE citar o id `FR-MAN-*` no teste que o cobre.
