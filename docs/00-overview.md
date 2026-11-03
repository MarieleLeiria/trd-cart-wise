---
id: overview
title: "Requisitos gerais do CartWise"
type: overview
status: proposed
consult_when: [any-task]
applies_to: ["**"]
related: [nfr, stores, contracts, entities, feat-auto-capture, feat-manual-save, feat-item-list, feat-remove-item, agentflow-architecture:overview]
tags: [produto, escopo, glossario]
---

# Requisitos gerais do CartWise

## Objetivo

Definir o que o MVP do CartWise deve fazer, o vocabulário comum dos requisitos e o que está bloqueado. A estrutura do sistema está em `agentflow-architecture:overview` e `agentflow-architecture:components`. Este documento não a repete.

O usuário navega em lojas, salva produtos do carrinho e consulta tudo numa lista só. Há dois jeitos de salvar: o clique em "adicionar ao carrinho" e o botão "Salvar este produto". A URL identifica o produto; ela não dispara a captura.

## Glossário

| Termo | Significado |
| ----- | ----------- |
| Captura automática | Extração disparada pelo clique em adicionar ao carrinho. `trigger: "auto"`. |
| Captura manual | Extração disparada pelo botão do popup. `trigger: "manual"`. |
| `CapturedProduct` | Dado bruto do adaptador, antes da normalização. Ver `agentflow-architecture:data-model`. |
| `NormalizedProduct` | Dado já com preço inteiro, URL canônica e `variantKey`, produzido pelo service worker. |
| `SavedItem` | Registro persistido. Um por `dedupeKey`. |
| `variant` | Mapa das opções selecionadas (cor, tamanho, voltagem). `{}` quando o produto não tem variação. |
| `variantKey` | Serialização estável de `variant`. Vazia quando `variant` é `{}`. Ver `entities`. |
| `dedupeKey` | `storeDomain\|productUrl\|variantKey`. Identidade do item. |
| `reason` | Código de falha do catálogo fechado em `contracts`. |
| Adaptador específico | Implementação de uma das cinco lojas do MVP. |
| Adaptador `generic` | Fallback de melhor esforço para os outros sites HTTPS. |
| Toast | Aviso na página, dentro do Shadow DOM fechado. Ver `agentflow-architecture:adr-0008`. |
| Badge | Texto curto no ícone da extensão, atualizado pelo service worker. |

## Requisitos do MVP

| Feature | Id | O que o usuário consegue fazer |
| ------- | -- | ------------------------------ |
| Captura automática | `feat-auto-capture` | Salvar o produto ao clicar em adicionar ao carrinho nas lojas suportadas. |
| Captura manual | `feat-manual-save` | Salvar o produto da aba atual pelo popup, quando o clique não foi detectado. |
| Lista | `feat-item-list` | Ver, buscar, filtrar por loja, escolher a moeda e ordenar por preço. |
| Remoção | `feat-remove-item` | Apagar um item salvo, sem desfazer. |

Requisitos funcionais usam `FR-CAP`, `FR-MAN`, `FR-LST` e `FR-REM`. Requisitos não funcionais usam `NFR-PERF`, `NFR-PRIV`, `NFR-RES` e `NFR-OFF`, em `nfr`.

As lojas com critério de aceitação completo são Magazine Luiza, Shein, Shopee, Mercado Livre e Amazon Brasil (`stores`).

## Fora de escopo

- Histórico de preço, alerta de queda e comparação do mesmo produto entre lojas.
- Finalizar a compra ou alterar o carrinho da loja.
- Backend, conta, sincronização entre dispositivos e telemetria.
- Captura disparada só por abrir uma URL, por mutação do DOM do carrinho ou por interceptar a rede.
- Firefox e qualquer navegador que não seja Chrome/Chromium.
- Gravar o binário das imagens.

## Regras para agentes

- DEVE tratar cada item de "Fora de escopo" como bloqueado. Se a tarefa exigir um deles, pare e reporte o conflito.
- DEVE citar os ids `FR-*` e `NFR-*` nos testes e no resultado da tarefa.
- NÃO DEVE criar um `reason` que não esteja em `contracts`.
- NÃO DEVE copiar regras de estrutura para cá. Aponte `agentflow-architecture:<id>`.
- DEVE declarar, na tarefa de lista, `change-list` e `change-ui`. Na de remoção, `change-removal` e `change-ui`.
