---
id: feat-item-list
title: "Lista de produtos salvos"
type: feature
status: proposed
consult_when: [change-list, change-ui]
applies_to: ["src/ui/**", "src/storage/**"]
related: [nfr, entities, feat-remove-item, agentflow-architecture:components, agentflow-architecture:adr-0002]
tags: [lista, busca, moeda]
---

# Lista de produtos salvos

## Objetivo

O usuário vê, numa lista só, os produtos salvos de todas as lojas. Ele busca pelo nome, filtra pela loja, escolhe a moeda e ordena por preço dentro dessa moeda.

## Escopo

Cobre o dashboard e o recorte do popup. Não cobre salvar (`feat-auto-capture`, `feat-manual-save`) nem apagar (`feat-remove-item`), mas a lista reage aos dois.

## Requisitos funcionais

### FR-LST-001 Cada item mostra o que foi salvo

Cada linha tem a imagem, o nome, as opções de `variant` unidas por ` · ` (nada, se `variant` for vazio), `storeName` ou, se for `null`, `storeDomain`, o preço formatado e um link para `productUrl`.

### FR-LST-002 O usuário escolhe a moeda

O seletor lista os códigos `price.currency` presentes, em ordem alfabética. A lista mostra só itens desse código, e a ordenação por preço não mistura moedas. A escolha fica em `chrome.storage.local` na chave `ui.listCurrency`.

O valor padrão, quando não há escolha ou a escolha não existe mais nos itens, é a moeda do item com maior `updatedAt`. Sem itens, o valor padrão é `BRL` e o seletor mostra só `BRL`.

### FR-LST-003 Dá para filtrar pela loja

Dentro da moeda escolhida, o filtro oferece "Todas" e uma entrada por `storeDomain` presente. O valor padrão é "Todas".

### FR-LST-004 A busca é pelo nome

A busca é substring do nome, sem distinção de maiúsculas e sem distinção de acentos (NFD, remover marcas `Mn`). Não busca loja nem variação. Texto vazio não filtra. Com 1.000 itens já em memória, busca, filtro de loja, filtro de moeda e ordenação terminam em até 200 ms (`NFR-PERF-003`).

### FR-LST-005 A ordenação é por preço

Há duas ordens: preço crescente, que é o valor padrão, e preço decrescente. A escolha fica em `chrome.storage.local` na chave `ui.listSort`, com os valores `price-asc` e `price-desc`. Empate desempata por `updatedAt` decrescente e depois por `id` crescente.

### FR-LST-006 A lista vazia explica o motivo

Sem nenhum item: "Nenhum produto salvo." Com itens na moeda escolhida, mas busca ou loja sem resultado: "Nenhum produto encontrado." Moeda escolhida que não tem mais itens volta ao valor padrão de `FR-LST-002`, em vez de mostrar uma lista vazia de moeda.

### FR-LST-007 Preço e imagem quebrada têm forma estável

O preço usa `Intl.NumberFormat` com locale `pt-BR` e a moeda do item. A imagem que falha ao carregar é trocada por um placeholder de caixa fixa, sem ícone de imagem quebrada.

### FR-LST-008 O popup mostra um recorte

O popup lista no máximo cinco itens, os de maior `updatedAt` na moeda escolhida, além do botão de `feat-manual-save` e do link "Ver todos", que abre o dashboard. O popup não tem busca nem filtro de loja.

### FR-LST-009 A lista acompanha captura e remoção

Dashboard e popup usam `SavedItemRepository.observe`. Um item capturado ou removido aparece ou some sem recarregar a página da extensão.

### FR-LST-010 A lista funciona offline

Sem rede, busca, filtros, ordenação e abertura do dashboard continuam. A imagem que não carrega usa o placeholder. O link do produto não passa por servidor da extensão.

## Critérios de aceitação

### FR-LST-001

**Dado** um item `BRL` de nome "Camisa", loja "Shein", variação `{ color: "azul", size: "m" }` e preço `amount` `4990`
**Quando** o usuário abre o dashboard na moeda `BRL`
**Então** a linha mostra a imagem, "Camisa", "azul · m", "Shein" e `R$ 49,90`, e o link aponta para `productUrl`

### FR-LST-002

**Dado** itens em `BRL` e `USD`, e o de maior `updatedAt` em `USD`, sem `ui.listCurrency`
**Quando** o usuário abre o dashboard
**Então** a moeda selecionada é `USD` e nenhum item `BRL` aparece

**Dado** essa lista
**Quando** o usuário escolhe `BRL` e fecha o dashboard
**Então** `ui.listCurrency` é `BRL` e, ao reabrir, a moeda continua `BRL`

**Dado** nenhum item salvo
**Quando** o usuário abre o dashboard
**Então** a moeda selecionada é `BRL`

### FR-LST-003

**Dado** itens `BRL` de `shein.com` e de `amazon.com.br`
**Quando** o usuário filtra por `shein.com`
**Então** só os itens desse `storeDomain` aparecem, e o filtro não oferece domínio de outra moeda

### FR-LST-004

**Dado** itens chamados "Café especial" e "Camisa"
**Quando** o usuário busca `cafe`
**Então** só "Café especial" permanece

**Dado** 1.000 itens já carregados em memória
**Quando** o usuário digita uma busca e combina com moeda, loja e ordenação
**Então** o resultado fica pronto em até 200 ms, sem contar a leitura do IndexedDB

### FR-LST-005

**Dado** três itens `BRL` de 1000, 3000 e 2000 centavos e `ui.listSort` ausente
**Quando** o usuário abre o dashboard
**Então** a ordem é 1000, 2000, 3000

**Dado** esses itens
**Quando** o usuário escolhe preço decrescente
**Então** a ordem é 3000, 2000, 1000 e `ui.listSort` passa a `price-desc`

**Dado** um item `USD` de 100 centavos e um item `BRL` de 100000 centavos
**Quando** a moeda selecionada é `BRL`
**Então** o item `USD` não entra na ordenação

### FR-LST-006

**Dado** nenhum item
**Quando** o usuário abre o dashboard
**Então** lê "Nenhum produto salvo."

**Dado** itens só em `USD` e `ui.listCurrency` igual a `BRL`
**Quando** o usuário abre o dashboard
**Então** a moeda selecionada passa a ser `USD` e os itens `USD` aparecem

**Dado** itens `BRL` e uma busca sem correspondência
**Quando** o usuário busca `zzzz`
**Então** lê "Nenhum produto encontrado."

### FR-LST-007

**Dado** um item `BRL` de `amount` `129990`
**Quando** a linha é renderizada
**Então** o preço visível é `R$ 1.299,90`

**Dado** uma `images[0]` que responde erro de rede
**Quando** a linha é renderizada
**Então** o espaço da imagem mostra o placeholder e não o ícone de imagem quebrada do navegador

### FR-LST-008

**Dado** oito itens na moeda selecionada
**Quando** o usuário abre o popup
**Então** vê os cinco de maior `updatedAt`, o botão "Salvar este produto" e o link "Ver todos", e não vê campo de busca

### FR-LST-009

**Dado** o dashboard aberto
**Quando** uma captura grava um item novo da moeda selecionada
**Então** a linha aparece sem recarregar a página

### FR-LST-010

**Dado** itens já gravados e o navegador offline
**Quando** o usuário abre o dashboard, busca e ordena
**Então** a lista responde com os dados locais e a imagem que falha usa o placeholder

## Restrições

- A moeda e a ordenação não vão para o IndexedDB (`agentflow-architecture:adr-0002`).
- Não há histórico de preço na linha.
- Não comparar preços de moedas diferentes nem converter câmbio.
- A UI não grava item novo (`agentflow-architecture:components`, `comp-ui`).

## Questões em aberto

- Nenhuma.

## Regras para agentes

- DEVE aplicar busca, loja e ordenação só depois do filtro de moeda.
- DEVE persistir `ui.listCurrency` e `ui.listSort` em `chrome.storage.local`.
- NÃO DEVE ordenar uma lista que misture moedas.
- DEVE tratar "cafe" e "Café" como a mesma busca.
- DEVE citar o id `FR-LST-*` no teste que o cobre.
