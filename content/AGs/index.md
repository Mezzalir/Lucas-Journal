+++
date = '2026-09-21T10:17:29-03:00'
draft = false 
title = 'Algoritmos genéticos'
type = 'blog'
tags = ['unioeste', 'otimização']
+++

## Primeiro de tudo

Antes de conversarmos sobre algoritmos genéticos, é importante definir, de maneira breve, computabilidade e complexidade.

_1. Computabilidade_

Decidíveis: existe algoritmo que sempre para e responde corretamente.
Indecidíveis: nenhum algoritmo resolve para toda entrada, por mais tempo que se dê. Ex.: o problema da parada.

2. Complexidade — dado que é decidível, quanto custa?

P: resolvível em tempo polinomial. Ex.: ordenação, algoritmo de Dijkstra.

NP: uma solução candidata pode ser verificada em tempo polinomial. Toda a classe P está contida em NP.

NP-completo: os problemas mais difíceis dentro de NP — todos os outros de NP se reduzem a eles em tempo polinomial. Ex.: SAT, ou o caixeiro-viajante na versão decisão ("existe rota de custo ≤ k?").

Existem problemas que não conseguimos resolver de maneira computacionalmente viável. O caixeiro-viajante com 70 cidades tem mais rotas possíveis do que átomos na parte observável do universo. Trocamos, então, a melhor solução por uma boa solução, diminuindo o tempo computacional. É um trade-off, como quase tudo na computação.

## Sobre o AG

Os algoritmos genéticos foram consolidados na década de 1960 na Universidade de Michigan, por John Holland e seus alunos. A ideia era simples: imitar a evolução biológica. Definir um espaço de busca, representar soluções candidatas como indivíduos e aplicar operadores de seleção, cruzamento e mutação ao longo de gerações.

AGs são metaheurísticas: não resolvem uma classe formal de problemas, atacam uma situação prática. O nicho deles são problemas de otimização em que os métodos exatos falham — seja por serem NP-difíceis, seja por não haver gradiente ou sequer uma expressão fechada para derivar. Ali, uma boa solução obtida em tempo razoável vale mais do que a ótima obtida tarde demais.

AGs não resolvem problemas indecidíveis, não mudam a classe de complexidade de nada e geralmente são uma má escolha quando já existe algoritmo exato e eficiente, como ordenar uma lista. Vale acrescentar: diferente dos algoritmos aproximativos, um AG não oferece nenhuma garantia de quão perto do ótimo você chegou.

O AG troca garantia de otimalidade por tempo.

### O ciclo básico de um AG e termos

```
população inicial aleatória
repita até o critério de parada:
    avalia o fitness de cada indivíduo
    seleciona pais
    cruza os pais → filhos
    muta os filhos
    aplica elitismo
    nova geração substitui a anterior
```

| Biologia               | Computação                          |
| ---------------------- | ----------------------------------- |
| Indivíduo / cromossomo | uma solução candidata               |
| Gene                   | um bit da string                    |
| População              | conjunto de candidatas em avaliação |
| Geração                | uma iteração do laço                |
| Fitness                | qualidade da solução                |
