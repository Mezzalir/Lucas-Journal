+++
date = '2026-09-21T10:17:29-03:00'
draft = false
title = 'Algoritmos genéticos'
type = 'blog'
tags = ['unioeste', 'otimização']
+++

## Primeiro de tudo

Antes de conversarmos sobre algoritmos genéticos, é importante definir, de maneira breve, computabilidade e complexidade.

### 1. Computabilidade: existe algoritmo?

- **Decidíveis:** existe um algoritmo que sempre para e responde corretamente, para qualquer entrada.
- **Indecidíveis:** nenhum algoritmo resolve o problema para toda entrada, por mais tempo que se dê. Ex.: o problema da parada.

### 2. Complexidade: dado que é decidível, quanto custa?

- **P:** problemas resolvíveis em tempo polinomial. Ex.: ordenação, caminho mínimo com o algoritmo de Dijkstra.
- **NP:** problemas em que uma solução candidata pode ser *verificada* em tempo polinomial, mesmo que encontrá-la seja difícil. Toda a classe P está contida em NP. Se P = NP, ou seja, se todo problema fácil de verificar também é fácil de resolver, continua sendo uma questão em aberto.
- **NP-completo:** os problemas mais difíceis dentro de NP. Todos os outros problemas de NP se reduzem a eles em tempo polinomial. Ex.: SAT, ou o caixeiro-viajante na versão de decisão ("existe uma rota de custo ≤ k?").
- **NP-difícil:** problemas pelo menos tão difíceis quanto os NP-completos, mas que não precisam estar em NP. A versão de otimização do caixeiro-viajante ("qual é a rota de menor custo?") é NP-difícil.

Para os problemas NP-difíceis, não se conhece nenhum algoritmo exato de tempo polinomial, e a força bruta explode rapidamente. O caixeiro-viajante com 70 cidades tem cerca de 10⁹⁸ rotas possíveis, muito mais do que os aproximadamente 10⁸⁰ átomos do universo observável. Testar todas está fora de questão.

Nesses casos, frequentemente trocamos a *melhor* solução por uma solução *boa o suficiente*, obtida em tempo viável. É um trade-off, como quase tudo na computação.

## Sobre o AG

Os algoritmos genéticos foram desenvolvidos nas décadas de 1960 e 1970 na Universidade de Michigan, por John Holland e seus alunos, e formalizados no livro *Adaptation in Natural and Artificial Systems* (1975). A ideia era simples: imitar a evolução biológica. Representar soluções candidatas como indivíduos de uma população e, ao longo de gerações, aplicar operadores de seleção, cruzamento e mutação, de modo que as melhores soluções tenham mais chance de passar suas características adiante.

AGs são **metaheurísticas**: estratégias de busca genéricas, que não são feitas para um problema específico. Para aplicá-las, basta saber representar uma solução e medir o quão boa ela é. O nicho deles são problemas de otimização em que os métodos exatos falham, seja por serem NP-difíceis, seja por não haver gradiente ou sequer uma expressão fechada para derivar. Ali, uma boa solução obtida em tempo razoável vale mais do que a ótima obtida tarde demais.

AGs não resolvem problemas indecidíveis, não mudam a classe de complexidade de nada e geralmente são uma má escolha quando já existe algoritmo exato e eficiente, como para ordenar uma lista. Vale acrescentar: diferente dos algoritmos de aproximação, um AG não oferece nenhuma garantia de quão perto do ótimo você chegou. E, como é estocástico, duas execuções podem chegar a resultados diferentes.

O AG troca garantia de otimalidade por tempo.

### O ciclo básico de um AG

```
gera população inicial aleatória
avalia o fitness de cada indivíduo
repita até o critério de parada:
    seleciona os pais
    cruza os pais → filhos           (com probabilidade de cruzamento)
    muta os filhos                   (com probabilidade de mutação)
    avalia o fitness dos filhos
    aplica elitismo                  (os melhores passam direto)
    nova geração substitui a anterior
retorna o melhor indivíduo encontrado
```

### Termos

| Biologia               | Computação                                                 |
| ---------------------- | ---------------------------------------------------------- |
| Indivíduo / cromossomo | uma solução candidata                                      |
| Gene                   | uma posição do cromossomo (ex.: um bit, na codificação binária) |
| Alelo                  | o valor que um gene assume (ex.: 0 ou 1)                   |
| População              | conjunto de soluções candidatas em avaliação               |
| Geração                | uma iteração do laço                                       |
| Fitness (aptidão)      | medida da qualidade de uma solução                         |

### Os operadores

- **Seleção:** escolhe quem vai se reproduzir, favorecendo os indivíduos com melhor fitness. As formas mais comuns são a *roleta*, em que a chance de ser escolhido é proporcional ao fitness, e o *torneio*, em que alguns indivíduos são sorteados e o melhor deles vence.
- **Cruzamento (crossover):** combina partes de dois pais para gerar filhos. No cruzamento de um ponto, por exemplo, escolhe-se uma posição de corte e os filhos herdam o início de um pai e o final do outro.
- **Mutação:** altera aleatoriamente alguns genes, como inverter um bit. Ela acontece com probabilidade baixa e serve para manter a diversidade da população, evitando que o AG fique preso em um ótimo local.
- **Elitismo:** copia os melhores indivíduos diretamente para a próxima geração, garantindo que a melhor solução encontrada até então não se perca.

O equilíbrio entre esses operadores é o que faz o AG funcionar. A seleção e o elitismo empurram a população para as regiões boas do espaço de busca (*intensificação*), enquanto a mutação e o cruzamento continuam explorando regiões novas (*diversificação*). Pressão seletiva demais faz a população convergir cedo para uma solução medíocre; de menos, e o AG vira uma busca aleatória.

### Critério de parada

O laço termina quando atinge um número máximo de gerações, um limite de tempo, um fitness considerado bom o suficiente, ou quando a melhor solução para de melhorar por várias gerações seguidas.

---

*Texto modificado pelo Claude*
