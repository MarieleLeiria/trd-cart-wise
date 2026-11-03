---
id: contracts
title: "Contratos de mensagem, falhas e feedback"
type: contract
status: proposed
consult_when: [change-capture, change-ui, change-storage]
applies_to: ["src/shared/**", "src/content/**", "src/background/**", "src/ui/**"]
related: [entities, stores, feat-auto-capture, feat-manual-save, agentflow-architecture:components, agentflow-architecture:adr-0008]
tags: [mensagens, erros, toast, badge]
---

# Contratos de mensagem, falhas e feedback

## Objetivo

Fixar o que o usuário lê quando a captura dá certo ou falha, e quem emite cada `reason`. A lista de `type` está em `agentflow-architecture:components`. A tabela abaixo acrescenta o domínio de `created`, `permission` e `reason`. Se um `type` existir só num dos dois documentos, pare e reporte os ids.

## Mensagens

| type | Origem | Destino | Payload | Resposta |
| ---- | ------ | ------- | ------- | -------- |
| `ITEM_CAPTURED` | content script | service worker | `CapturedProduct` | `{ ok: true, itemId, created: boolean }` ou `{ ok: false, reason }` |
| `CAPTURE_ACTIVE_TAB` | UI | service worker | `{ tabId }` | `{ ok: true, itemId, created: boolean, name }` ou `{ ok: false, reason }` ou `{ ok: false, permission: "denied" \| "not_https" }` |
| `RUN_CAPTURE` | service worker | content script | `{ trigger: "manual" }` | `CapturedProduct` ou `{ ok: false, reason }` |
| `CAPTURE_FAILED` | content script | service worker | `{ reason, trigger }` | `{ ok: true }` |

`created: true` é a primeira gravação daquele `dedupeKey`. `created: false` é recaptura. O texto do toast depende disso.

`permission` não é `reason`. A permissão é resolvida antes da extração, e o catálogo de falhas de captura não a inclui.

## Motivos de falha

O catálogo é fechado. O mapa código → texto fica em `src/shared/capture-messages.ts` e é a única fonte do toast, do popup e dos testes.

| `reason` | Quem emite | Texto em pt-BR |
| -------- | ---------- | -------------- |
| `currency_unknown` | service worker | Não foi possível identificar a moeda deste produto. |
| `no_image` | adaptador ou service worker | Não encontramos uma imagem do produto. |
| `variant_not_selected` | adaptador específico | Selecione cor, tamanho ou outra opção do produto antes de salvar. |
| `price_not_found` | adaptador | Não encontramos o preço deste produto. |
| `name_not_found` | adaptador ou service worker | Não encontramos o nome deste produto. |
| `adapter_error` | content script | Não foi possível ler esta página. |
| `invalid_payload` | service worker | Os dados capturados são inválidos. |
| `storage_error` | service worker | Não foi possível salvar o produto neste navegador. |

Quem emite o quê:

- O adaptador devolve `status: "fail"` com `variant_not_selected`, `price_not_found`, `name_not_found` ou `no_image`. Não lança exceção.
- O content script, se o específico e o `generic` devolverem `unhandled`, ou se um adaptador lançar, emite `adapter_error` e manda `CAPTURE_FAILED`.
- O service worker revalida o payload. Moeda, imagem que sobrou vazia depois do filtro `https`, nome vazio e schema inválido viram os `reason` da tabela na resposta de `ITEM_CAPTURED`. Falha do IndexedDB vira `storage_error`.
- `variant_not_selected` não passa pelo `generic`.

## Feedback

### Toast

Vale para captura automática e manual, no sucesso e na falha. As regras de DOM estão em `agentflow-architecture:adr-0008`.

| Situação | Texto |
| -------- | ----- |
| `created: true` | Produto salvo. |
| `created: false` | Produto atualizado. |
| falha com `reason` | O texto da tabela de motivos. |

O toast só aparece com o resultado definitivo: falha já conhecida pelo content script, ou resposta do service worker a `ITEM_CAPTURED`. Um toast novo substitui o anterior. O host sai após 4 s sem outro toast. Não há toast para `permission: "denied"` nem para `permission: "not_https"`, porque a página pode nem ter content script.

### Popup

Na captura manual, o popup mostra o mesmo texto do toast. No sucesso, mostra também o `name`. Para permissão:

| `permission` | Texto |
| ------------ | ----- |
| pedido | O CartWise precisa de permissão para ler esta página. |
| `denied` | Sem permissão, este site não pode ser salvo. |
| `not_https` | Só é possível salvar páginas HTTPS. |

O botão do pedido é "Permitir".

### Badge

Só o service worker escreve o badge (`chrome.action`), ao concluir `ITEM_CAPTURED` ou ao receber `CAPTURE_FAILED`.

| Resultado | Texto | Fundo |
| --------- | ----- | ----- |
| sucesso | `OK` | `#0B7A3B` |
| falha | `!` | `#B42318` |

O badge fica até a próxima captura. Não some sozinho.

## Questões em aberto

- Confirmar a duração de 4 s do toast antes de aceitar este documento. O mesmo valor está em `agentflow-architecture:adr-0008`.

## Regras para agentes

- DEVE usar só os oito `reason` desta tabela. Um código novo entra primeiro aqui e no enum Zod, na mesma mudança.
- DEVE ler o texto em `src/shared/capture-messages.ts`. Não duplique a frase no toast e no popup.
- NÃO DEVE tratar `permission` como `reason`.
- DEVE preencher `created` na resposta de sucesso para o toast distinguir "salvo" de "atualizado".
- NÃO DEVE escrever o badge a partir do content script ou da UI.
