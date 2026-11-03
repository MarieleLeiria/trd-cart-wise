---
id: feat-remove-item
title: "Remover um produto salvo"
type: feature
status: proposed
consult_when: [change-removal, change-ui]
applies_to: ["src/ui/**", "src/storage/**"]
related: [entities, feat-item-list, agentflow-architecture:data-model, agentflow-architecture:components]
tags: [remocao]
---

# Remover um produto salvo

## Objetivo

O usuário apaga um produto da lista do CartWise. A remoção é definitiva, não desfaz e não mexe no carrinho da loja.

## Escopo

Cobre a confirmação, o apagamento físico e o efeito sobre as outras variações. A lista que permanece na tela é `feat-item-list`.

## Requisitos funcionais

### FR-REM-001 Dá para remover no dashboard e no popup

Toda linha do dashboard e toda linha mostrada no popup tem a ação "Remover".

### FR-REM-002 A remoção pede confirmação

A ação abre a pergunta "Remover este produto da lista?" com os botões "Remover" e "Cancelar". "Cancelar" fecha o diálogo e o item continua. "Remover" chama `SavedItemRepository.remove` com o `id` daquela linha.

### FR-REM-003 O registro some de verdade

A remoção apaga o `SavedItem`. Não há campo de item inativo, lixeira nem desfazer. A linha sai da lista assim que o repositório confirma.

### FR-REM-004 Só a variação escolhida sai

Apagar um item não apaga outro `SavedItem` com a mesma `productUrl` e outra `variantKey`.

### FR-REM-005 A loja não é chamada

A remoção não abre a URL do produto, não chama a API da loja e não altera o carrinho da loja.

### FR-REM-006 Capturar de novo cria outro id

Depois de apagado, uma captura do mesmo `dedupeKey` cria um `SavedItem` com outro `id`. `capturedAt` é o instante dessa captura nova.

### FR-REM-007 A moeda da lista se ajusta

Se o item removido era o último da moeda selecionada, a moeda volta ao valor padrão de `FR-LST-002`.

## Critérios de aceitação

### FR-REM-001

**Dado** um item visível no dashboard e entre os cinco do popup
**Quando** o usuário olha as duas telas
**Então** as duas linhas têm "Remover"

### FR-REM-002

**Dado** um item salvo
**Quando** o usuário aciona "Remover" e depois "Cancelar"
**Então** o diálogo fecha e o item continua na lista e no IndexedDB

**Dado** o mesmo item
**Quando** o usuário confirma "Remover"
**Então** `remove` é chamado com o `id` dessa linha e com nenhum outro

### FR-REM-003

**Dado** um único item
**Quando** o usuário confirma a remoção
**Então** a lista passa a mostrar "Nenhum produto salvo." e uma leitura do repositório não devolve esse `id`

### FR-REM-004

**Dado** dois itens com a mesma `productUrl`, um com `variantKey` `size=m` e outro com `size=g`
**Quando** o usuário remove o `size=m`
**Então** o `size=g` continua na lista, com o mesmo `id` de antes

### FR-REM-005

**Dado** um item cujo `productUrl` é de `mercadolivre.com.br`
**Quando** o usuário confirma a remoção
**Então** a extensão não emite requisição de rede e o carrinho da loja não é lido nem alterado

### FR-REM-006

**Dado** um item removido cujo `dedupeKey` era `shein.com|https://br.shein.com/p|size=m`
**Quando** o usuário captura de novo essa mesma variação
**Então** o novo `SavedItem` tem outro `id` e o `capturedAt` é o dessa captura

### FR-REM-007

**Dado** um item `USD` e um item `BRL` mais antigo, com `USD` selecionado
**Quando** o usuário remove o único item `USD`
**Então** a moeda selecionada passa a ser `BRL`, que é a moeda do item restante de maior `updatedAt`

## Restrições

- Sem soft delete e sem histórico de preço (`entities`, `agentflow-architecture:data-model`).
- A UI não apaga direto no IndexedDB. Usa `SavedItemRepository.remove` (`agentflow-architecture:components`).
- Não enviar dados para fora (`NFR-PRIV-001`).

## Questões em aberto

- Nenhuma.

## Regras para agentes

- DEVE pedir confirmação antes de chamar `remove`.
- NÃO DEVE marcar o item como inativo no lugar de apagar.
- NÃO DEVE apagar outros `dedupeKey` no mesmo gesto.
- NÃO DEVE contactar a loja durante a remoção.
- DEVE citar o id `FR-REM-*` no teste que o cobre.
