# agentflow-trd

Fonte de conhecimento de **requisitos técnicos** do CartWise para os agentes do AgentFlow: regras de negócio, contratos, lojas suportadas e critérios de aceitação.

O CartWise é uma extensão de navegador que, quando o usuário adiciona um item ao carrinho de uma loja, captura nome, fotos, site, preço e variação do produto e reúne os itens salvos em uma única lista.

Este repositório faz parte de um conjunto de três (ver `agentflow-architecture:adr-0007`):

| Repositório | Quando o agente consulta |
| ----------- | ------------------------ |
| `agentflow-architecture` | Antes de definir ou alterar a estrutura do sistema. |
| `agentflow-trd` (este) | Antes de implementar uma feature. |
| `agentflow-engineering-standards` | Durante a implementação e a validação de qualquer feature. |

Em conflito, a arquitetura prevalece em estrutura e dados; este repositório prevalece em comportamento, regras de negócio, contratos e critérios de aceitação. O processo está em `agentflow-architecture:docs-resolution`.

## Estrutura

```
manifest.yaml          índice legível por máquina (ponto de entrada do AgentFlow)
docs/                  requisitos transversais
docs/features/         uma feature do MVP por arquivo
templates/             modelo para uma feature nova
```

| Id | Documento | Conteúdo |
| -- | --------- | -------- |
| `overview` | [docs/00-overview.md](docs/00-overview.md) | Glossário, requisitos do MVP e fora de escopo. |
| `nfr` | [docs/01-nfr.md](docs/01-nfr.md) | Desempenho, privacidade, resiliência e offline. |
| `stores` | [docs/02-stores.md](docs/02-stores.md) | Cinco lojas, adaptador genérico, hosts e fixtures. |
| `contracts` | [docs/03-contracts.md](docs/03-contracts.md) | Mensagens, motivos de falha, toast e badge. |
| `entities` | [docs/04-entities.md](docs/04-entities.md) | Preço, `variantKey`, `dedupeKey` e ciclo de vida. |
| `feat-auto-capture` | [docs/features/feat-auto-capture.md](docs/features/feat-auto-capture.md) | Captura pelo clique em adicionar ao carrinho. |
| `feat-manual-save` | [docs/features/feat-manual-save.md](docs/features/feat-manual-save.md) | Botão "Salvar este produto". |
| `feat-item-list` | [docs/features/feat-item-list.md](docs/features/feat-item-list.md) | Lista, busca, moeda e ordenação. |
| `feat-remove-item` | [docs/features/feat-remove-item.md](docs/features/feat-remove-item.md) | Remoção física de um item. |

## Protocolo de consumo pelo AgentFlow

1. Ler `agentflow.docs.lock.yaml` no repositório do produto e obter o **commit SHA completo** deste repositório.
2. Ler `manifest.yaml` nesse commit.
3. Selecionar os documentos cujo `consult_when` contém `any-task` ou intersecta os tipos da tarefa, ou cujo `applies_to` casa com os caminhos que a tarefa toca.
4. Ignorar documentos `superseded`, `deprecated` e `rejected`. Tratar `accepted` como obrigatório e `proposed` como orientação sinalizada.
5. Entregar ao agente o conteúdo dos documentos, destacando `## Regras para agentes` e os critérios **Dado / Quando / Então**.
6. Registrar em `docs_pins` o commit e os ids usados.

Uma tarefa deve declarar os `consult_when` dos dois repositórios que ela precisa. Exemplos:

| Tarefa | `consult_when` |
| ------ | -------------- |
| Captura automática ou manual | `change-capture` e, se mexer na UI, `change-ui` |
| Nova loja ou fixture | `add-store-adapter`, `change-capture` |
| Lista, busca, moeda ou ordenação | `change-list`, `change-ui` |
| Remover item | `change-removal`, `change-ui` |

`change-list` e `change-removal` existem só neste repositório. A UI correspondente no `agentflow-architecture` continua sob `change-ui`.

## Contrato dos documentos

O mesmo de `agentflow-architecture:adr-0006`.

- `type`, `status` e `consult_when` vêm do `vocabulary` deste `manifest.yaml`.
- A ordem das seções H2 está em `section_order`.
- Todo documento termina a parte normativa com `## Regras para agentes`. Em ADR esse título fica no repositório de arquitetura; aqui, nas features, ele é a última seção H2.
- Requisitos funcionais usam id `FR-<SIGLA>-<NNN>` (`CAP`, `MAN`, `LST`, `REM`). Não funcionais usam `NFR-<SIGLA>-<NNN>` (`PERF`, `PRIV`, `RES`, `OFF`). O id não muda quando o texto muda.
- Cada critério de aceitação repete o id do requisito e usa **Dado**, **Quando** e **Então**.
- Referências cruzadas usam `<repositorio>:<id>`, por exemplo `agentflow-architecture:data-model`.

## Como contribuir

- **Nova feature:** copie [templates/feature.md](templates/feature.md), escolha a próxima sigla livre e registre o documento em `manifest.yaml` antes de usar um `consult_when` novo.
- **Novo `consult_when`:** entre primeiro no `vocabulary` do manifesto. Não invente o valor no frontmatter.
- **Novo motivo de falha:** só em [docs/03-contracts.md](docs/03-contracts.md). O catálogo é fechado.
- **Toda mudança:** mantenha frontmatter e `manifest.yaml` idênticos nos campos `id`, `type`, `status`, `consult_when`, `applies_to` e `tags`.

Os produtos só passam a usar a mudança quando o SHA for atualizado no `agentflow.docs.lock.yaml` deles.

## Estado atual

Todos os documentos estão como `proposed`. Valores assumidos, ainda não aceitos:

- Espera máxima de 2 s quando o preço ainda não está no DOM (`nfr`, `NFR-PERF-002`).
- Toast visível por 4 s (`contracts`, `agentflow-architecture:adr-0008`).
- Padrões de host das cinco lojas (`stores`).
