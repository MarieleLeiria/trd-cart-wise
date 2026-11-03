---
id: feat-nome-curto
title: "<Nome da feature>"
type: feature
status: proposed            # proposed | accepted | superseded | deprecated | rejected
consult_when: []            # valores de manifest.yaml > vocabulary.consult_when
applies_to: []              # globs de código no repositório do produto
related: []                 # ids deste repositório, ou <repositorio>:<id>
tags: []
---

# <Nome da feature>

## Objetivo

<O comportamento que o usuário passa a ter.>

## Escopo

<O que esta feature cobre e o que fica para outra feature.>

## Requisitos funcionais

### FR-XXX-001 <título curto>

<Frase testável. Um requisito, um comportamento.>

## Critérios de aceitação

### FR-XXX-001

**Dado** <estado inicial>
**Quando** <ação>
**Então** <resultado observável>

## Restrições

- <Limites que a feature não pode cruzar. Aponte o id de `nfr`, `entities` ou `agentflow-architecture`.>

## Questões em aberto

- <Itens que impedem `status: accepted`. Escreva "Nenhuma." se não houver.>

## Regras para agentes

- DEVE manter o id `FR-...` estável; testes e o resultado da tarefa citam esse id.
- DEVE atualizar a entrada em `manifest.yaml` ao criar este documento ou mudar `status` ou `consult_when`.
- NÃO DEVE implementar itens de "Fora de escopo" do `overview`.
