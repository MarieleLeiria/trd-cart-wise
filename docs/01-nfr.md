---
id: nfr
title: "Requisitos não funcionais"
type: nfr
status: proposed
consult_when: [any-task]
applies_to: ["**"]
related: [overview, stores, contracts, feat-auto-capture, feat-item-list, agentflow-architecture:overview]
tags: [nfr, desempenho, privacidade]
---

# Requisitos não funcionais

## Objetivo

Fixar limites mensuráveis que valem para todas as features. Um critério de aceitação de feature não pode afrouxar um `NFR`.

## Requisitos

### NFR-PERF-001 Extração com o preço já no DOM

Quando o preço atual já está no DOM no instante do clique (ou do `RUN_CAPTURE`), a extração, a validação Zod e o envio de `ITEM_CAPTURED` terminam em até 500 ms. O listener síncrono do clique retorna sem esperar essa extração. O listener é passivo e está na fase de captura.

### NFR-PERF-002 Espera quando o preço ainda não está no DOM

Quando o preço ainda não está no DOM, o adaptador espera um sinal de preço por no máximo 2 s contados do clique ou do `RUN_CAPTURE`. Esse prazo substitui o de 500 ms; os dois não se somam. Esgotadas as 2 s sem preço, a captura falha com `price_not_found`. Shein e Shopee caem neste caso com frequência. Nas outras três lojas ele só vale se o preço realmente ainda não estiver no DOM.

### NFR-PERF-003 Lista em memória

Com 1.000 `SavedItem` já em memória, aplicar busca, filtro de loja, filtro de moeda e ordenação termina em até 200 ms. A leitura do IndexedDB fica fora dessa medida.

### NFR-PERF-004 Não interferência

A captura não chama `preventDefault`, `stopPropagation` nem `stopImmediatePropagation`. Não atrasa o handler da loja e não altera o DOM, exceto o host do toast (`agentflow-architecture:adr-0008`).

### NFR-PRIV-001 Sem saída de dados

A extensão não envia dados capturados, URLs de página, DOM ou identificadores para qualquer servidor. Não há `fetch`, `XMLHttpRequest`, `WebSocket` nem `sendBeacon` originados pela extensão. O navegador pode carregar as URLs de imagem que a UI pede para mostrar.

### NFR-PRIV-002 Sem telemetria

Não há analytics, log remoto nem relatório de erro. Falha de captura vai para o console da extensão, para o toast e para o badge.

### NFR-PRIV-003 Permissões mínimas

O manifest pede só `storage`, `activeTab` e `scripting`, mais os `host_permissions` e `optional_host_permissions` definidos em `stores`. Não pede `<all_urls>` em `host_permissions`.

### NFR-PRIV-004 Imagens por URL

O IndexedDB guarda URLs `https`. Não guarda o binário da imagem, miniatura nem data URL.

### NFR-RES-001 Falha isolada

A exceção de um adaptador vira `status: "fail"` ou `adapter_error`. Não derruba a página, os outros adaptadores nem o service worker. `status: "fail"` de um adaptador específico não aciona o `generic`.

### NFR-RES-002 Loja não suportada

Uma loja fora das cinco não impede a captura nas cinco. O `generic` é melhor esforço e não entra nos critérios de cobertura.

### NFR-OFF-001 Lista offline

Buscar, filtrar, ordenar e remover funcionam sem rede. Imagem que não carrega usa o placeholder de `feat-item-list`.

## Questões em aberto

- Confirmar o limite de 2 s de `NFR-PERF-002` antes de mudar este documento para `accepted`.

## Regras para agentes

- DEVE escrever um teste que falhe se `NFR-PERF-001`, `NFR-PERF-002` ou `NFR-PERF-003` estourar o prazo. O teste cita o id.
- NÃO DEVE adicionar permissão, host ou chamada de rede para cumprir uma feature.
- NÃO DEVE somar 500 ms e 2 s num único orçamento.
- DEVE manter a extração fora do handler síncrono do clique.
