+++
date = '2026-10-22T18:13:37-03:00'
draft = false
title = 'Memória Cache'
type = 'blog'
tags = ['Arquitetura']
+++

## Resumo

O processador é muito mais rápido que a memória principal. Se ele tivesse que buscar cada instrução e cada dado direto na RAM, passaria a maior parte do tempo esperando. A cache é uma memória pequena e rápida, colocada entre o processador e a memória principal. No fundo, ela nada mais é do que uma cópia de partes da memória RAM: guarda os dados usados com mais frequência, para que o processador não precise buscá-los lá toda vez. Ela funciona por causa do princípio de localidade: programas tendem a acessar os mesmos dados e dados próximos repetidas vezes.

## Hierarquia de memória

Toda memória envolve um equilíbrio entre três características: custo, capacidade e tempo de acesso. As relações são sempre as mesmas:

- Quanto mais rápida, maior o custo por bit.
- Quanto maior a capacidade, menor o custo por bit.
- Quanto maior a capacidade, mais lento o acesso.

Não existe uma memória que seja rápida, grande e barata ao mesmo tempo. A solução é usar várias, organizadas em uma hierarquia:

| Nível               | Capacidade típica | Tempo de acesso aproximado |
| ------------------- | ----------------- | -------------------------- |
| Registradores       | bytes             | menos de 1 ns              |
| Cache L1            | dezenas de KB     | ~1 ns                      |
| Cache L2            | centenas de KB    | ~3 a 5 ns                  |
| Cache L3            | dezenas de MB     | ~10 a 20 ns                |
| Memória principal   | GB                | ~60 a 100 ns               |
| SSD                 | centenas de GB    | ~100 µs                    |
| HD                  | TB                | ~5 a 10 ms                 |

Descendo na hierarquia, o custo por bit diminui, a capacidade aumenta, o tempo de acesso aumenta e o processador acessa aquele nível com menos frequência. Esse último ponto é o que faz a hierarquia funcionar: se a maior parte dos acessos acontecer nos níveis de cima, o sistema se comporta quase como se tivesse só memória rápida.

## Princípio de localidade

Durante a execução, os acessos de um programa à memória não são aleatórios. Eles tendem a se concentrar em poucas regiões. Existem dois tipos de localidade:

- **Localidade temporal:** um dado acessado agora provavelmente será acessado de novo em breve. Exemplo: as variáveis e as instruções de um laço.
- **Localidade espacial:** se um endereço foi acessado, os endereços próximos provavelmente também serão. Exemplo: percorrer um array ou executar instruções em sequência.

A cache aproveita as duas. Ela mantém os dados usados recentemente (localidade temporal) e, quando busca algo na memória, traz um bloco inteiro de palavras vizinhas, não só a palavra pedida (localidade espacial).

Quando o dado procurado está na cache, temos um **acerto** (hit). Quando não está, temos uma **falha** (miss), e o bloco precisa ser buscado na memória principal. O exemplo do Stallings mostra o efeito disso: com uma cache de 0,01 µs, uma memória principal de 0,1 µs e uma taxa de acerto de 95%, o tempo médio de acesso fica em cerca de 0,015 µs. É muito mais perto da velocidade da cache que da velocidade da memória.

## História e evolução

A ideia foi descrita por Maurice Wilkes em 1965, com o nome de "slave memory". O primeiro computador comercial com cache foi o IBM System/360 Model 85, de 1968.

Nos microprocessadores, a cache entrou no chip com o Intel 80486 (1989), que tinha 8 KB de cache interna. O Pentium (1993) dividiu a cache L1 em duas: uma para instruções e outra para dados. Depois, a L2 também foi para dentro do chip, e mais tarde veio a L3.

O motivo dessa evolução é que a velocidade dos processadores cresceu muito mais rápido que a da memória principal, e a diferença entre os dois só aumentou. Cada novo nível de cache surgiu para diminuir essa distância. Hoje, o arranjo comum nos processadores multi-core é uma L1 dividida (instruções e dados) e uma L2 para cada núcleo, e uma L3 compartilhada entre todos os núcleos.

## Formas de implementar

A memória principal é dividida em blocos, e a cache, em linhas. Cada linha guarda um bloco e uma tag, que identifica qual bloco da memória está ali. A função de mapeamento define em qual linha cada bloco pode ficar. Existem três formas:

- **Mapeamento direto:** cada bloco só pode ficar em uma linha específica (linha = número do bloco mod número de linhas). É simples e barato, mas se dois blocos muito usados caírem na mesma linha, um fica expulsando o outro.
- **Mapeamento associativo:** um bloco pode ficar em qualquer linha. É o mais flexível, mas para saber se um dado está na cache é preciso comparar a tag com todas as linhas ao mesmo tempo, o que exige muito hardware.
- **Mapeamento associativo por conjunto:** a cache é dividida em conjuntos de k linhas. Cada bloco vai para um conjunto específico, mas pode ocupar qualquer linha dentro dele. É o meio-termo entre os dois anteriores e a forma mais usada nos processadores atuais.

## Algoritmos de substituição

Quando a cache está cheia e um novo bloco precisa entrar, algum bloco tem que sair. No mapeamento direto não há escolha, porque cada bloco só tem uma linha possível. Nos outros dois, um algoritmo decide qual linha substituir:

- **LRU (Least Recently Used):** substitui o bloco que está há mais tempo sem ser usado. É o mais popular, porque aproveita bem a localidade temporal.
- **FIFO (First In, First Out):** substitui o bloco que está há mais tempo na cache.
- **LFU (Least Frequently Used):** substitui o bloco com menos acessos.
- **Aleatório:** escolhe uma linha ao acaso. Simulações mostram que o desempenho é só um pouco pior que o dos algoritmos baseados em uso.

Como o algoritmo precisa ser rápido, ele é implementado no próprio hardware. Na prática, muitos processadores usam aproximações do LRU, que são mais baratas.

## Políticas de escrita

Quando o processador escreve em um dado que está na cache, a cópia na memória principal fica desatualizada. As políticas de escrita definem quando a memória é atualizada:

- **Write-through:** toda escrita é feita na cache e na memória principal ao mesmo tempo. A memória está sempre atualizada, mas isso gera muito tráfego.
- **Write-back:** a escrita é feita só na cache, e a linha é marcada com um bit de "sujo" (dirty bit). A memória só é atualizada quando a linha é substituída. Gera menos tráfego, mas a memória fica desatualizada por um tempo.

Também é preciso decidir o que fazer em uma falha de escrita: trazer o bloco para a cache antes de escrever (write allocate) ou escrever direto na memória (no-write allocate). O mais comum é combinar write-back com write allocate.

## Coerência de cache

Em um processador multi-core, cada núcleo tem a sua própria cache. Se dois núcleos têm uma cópia do mesmo dado e um deles altera a sua, a cópia do outro fica errada. Garantir que todos os núcleos enxerguem o mesmo valor é o problema da coerência de cache.

A principal forma de resolver é o **protocolo MESI**, usado com snooping: cada cache "escuta" o barramento e observa os acessos dos outros núcleos. Cada linha fica em um de quatro estados:

| Estado            | Significado                                                       |
| ----------------- | ----------------------------------------------------------------- |
| **M** (Modified)  | Só esta cache tem a linha, e ela foi alterada (a memória está desatualizada). |
| **E** (Exclusive) | Só esta cache tem a linha, e ela é igual à memória.               |
| **S** (Shared)    | Outras caches também podem ter a linha, e todas são iguais à memória. |
| **I** (Invalid)   | A linha não é válida.                                             |

Quando um núcleo quer escrever em uma linha compartilhada, ele avisa as outras caches, que marcam as suas cópias como inválidas (I). A linha dele passa para o estado M. Se outro núcleo precisar desse dado depois, vai ter uma falha e buscar a versão atualizada. Existem variações do MESI, como o MOESI (AMD) e o MESIF (Intel). Em sistemas com muitos núcleos, onde escutar o barramento fica caro, usam-se protocolos de diretório, que guardam em um lugar central quais caches têm cada linha.

## Referências

- STALLINGS, William. _Arquitetura e Organização de Computadores_. Pearson.
- Wikipedia. [CPU cache](https://en.wikipedia.org/wiki/CPU_cache).
- Wikipedia. [Cache coherence](https://en.wikipedia.org/wiki/Cache_coherence).

_Partes do texto baseadas na Wikipedia foram compiladas com a ajuda do Claude._
