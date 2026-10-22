+++
date = '2026-10-22T18:40:00-03:00'
draft = true
title = 'Otimizando o Desempenho de Programas'
type = 'blog'
tags = ['Arquitetura', 'CSAPP']
+++

Compiladores modernos, como o GCC, aplicam várias otimizações no código. O nível é escolhido na compilação: `-Og` aplica só otimizações básicas, enquanto `-O1`, `-O2` e `-O3` aplicam otimizações cada vez mais agressivas. Mas existe uma regra que o compilador nunca quebra: ele só pode fazer otimizações **seguras**. O programa otimizado precisa se comportar exatamente igual ao original em todos os casos possíveis, mesmo nos que parecem improváveis. Por isso, o compilador às vezes deixa de fazer uma otimização que parece óbvia para o programador. Esses obstáculos são chamados de *optimization blockers*.

### Memory aliasing

Considere as duas funções abaixo:

```c
void twiddle1(long *xp, long *yp) {
    *xp += *yp;
    *xp += *yp;
}

void twiddle2(long *xp, long *yp) {
    *xp += 2 * *yp;
}
```

À primeira vista, as duas fazem a mesma coisa: somam duas vezes o valor de `*yp` em `*xp`. E `twiddle2` é mais eficiente, já que faz 3 acessos à memória (ler `*xp`, ler `*yp`, escrever `*xp`), enquanto `twiddle1` faz 6. Seria natural o compilador transformar a primeira na segunda.

O problema é o caso em que `xp` e `yp` apontam para o mesmo endereço. Se `*xp` vale 1:

- `twiddle1` faz 1 + 1 = 2 e depois 2 + 2 = 4. O valor é **quadruplicado**.
- `twiddle2` faz 1 + 2 × 1 = 3. O valor é **triplicado**.

Quando dois ponteiros podem apontar para a mesma posição de memória, dizemos que existe **memory aliasing**. Como o compilador não sabe se `xp` e `yp` vão ser iguais, ele precisa assumir que podem ser, e não pode transformar `twiddle1` em `twiddle2`.

### Chamadas de função

O segundo blocker são as chamadas de função:

```c
long f();

long func1() {
    return f() + f() + f() + f();
}

long func2() {
    return 4 * f();
}
```

Parece que `func2` é só uma versão mais rápida de `func1`, já que chama `f` uma vez em vez de quatro. Mas se `f` tiver efeitos colaterais, os resultados mudam:

```c
long counter = 0;

long f() {
    return counter++;
}
```

Com essa `f`, `func1` retorna 0 + 1 + 2 + 3 = 6, e `func2` retorna 4 × 0 = 0. Como o compilador normalmente não sabe o que a função faz, ele assume o pior caso e mantém todas as chamadas. Uma exceção é o *inlining*: quando o compilador substitui a chamada pelo corpo da função, ele passa a enxergar o que ela faz e pode otimizar o resultado.

> **Otimizando chamadas de função com inline substitution**
>
> Código com chamadas de função pode ser otimizado por um processo chamado *inline substitution* (ou simplesmente *inlining*), em que a chamada da função é substituída pelo código do corpo da função. Por exemplo, podemos expandir o código de `func1` substituindo as quatro chamadas pelo corpo de `f`:
>
> ```c
> /* Resultado de fazer inlining de f em func1 */
> long func1in() {
>     long t = counter++; /* +0 */
>     t += counter++;     /* +1 */
>     t += counter++;     /* +2 */
>     t += counter++;     /* +3 */
>     return t;
> }
> ```
>
> Essa transformação reduz o custo das chamadas de função e ainda permite otimizar o código expandido. Por exemplo, o compilador pode juntar as atualizações da variável global `counter` em `func1in` e gerar uma versão otimizada da função:
>
> ```c
> /* Otimização do código após o inlining */
> long func1opt() {
>     long t = 4 * counter + 6;
>     counter += 4;
>     return t;
> }
> ```
>
> Esse código reproduz fielmente o comportamento de `func1` para essa definição específica de `f`.
>
> Versões recentes do GCC tentam fazer essa otimização quando recebem a opção `-finline` ou com nível de otimização `-O1` ou maior. Infelizmente, o GCC só tenta fazer inlining de funções definidas no mesmo arquivo. Ou seja, ela não é aplicada no caso comum em que um conjunto de funções de biblioteca é definido em um arquivo e chamado por funções em outros arquivos.
>
> Há situações em que é melhor impedir o compilador de fazer inlining. Uma é quando o código vai ser analisado com um depurador simbólico, como o GDB. Se uma chamada de função foi eliminada pelo inlining, qualquer tentativa de rastrear essa chamada ou de colocar um breakpoint nela vai falhar. A outra é quando o desempenho do programa vai ser avaliado com *profiling*. As chamadas de funções que foram eliminadas pelo inlining não vão aparecer corretamente no resultado.

A lição do capítulo é que o compilador é limitado pelo que ele consegue provar sobre o código. Cabe ao programador escrever o código de um jeito que facilite as otimizações, por exemplo guardando um valor em uma variável local em vez de ler e escrever na memória repetidas vezes.

## O exemplo do livro

As próximas seções usam um mesmo exemplo: um vetor guardado em uma struct, com o tamanho e um ponteiro para os dados. O tipo dos elementos (`data_t`), a operação (`OP`, que pode ser `+` ou `*`) e o valor inicial (`IDENT`, que é 0 para soma e 1 para produto) são definidos na compilação. A função `combine1` combina todos os elementos do vetor:

```c
typedef struct {
    long len;
    data_t *data;
} vec_rec, *vec_ptr;

void combine1(vec_ptr v, data_t *dest) {
    long i;
    *dest = IDENT;
    for (i = 0; i < vec_length(v); i++) {
        data_t val;
        get_vec_element(v, i, &val);
        *dest = *dest OP val;
    }
}
```

A `vec_length` retorna o tamanho do vetor, e a `get_vec_element` copia um elemento para `val`, verificando antes se o índice está dentro dos limites. A cada seção, o código é melhorado um pouco.

## Eliminando ineficiências em laços

A `combine1` chama `vec_length` na condição do laço, ou seja, em toda iteração. Mas o tamanho do vetor não muda durante o laço, então basta calcular uma vez:

```c
void combine2(vec_ptr v, data_t *dest) {
    long i;
    long length = vec_length(v);
    *dest = IDENT;
    for (i = 0; i < length; i++) {
        data_t val;
        get_vec_element(v, i, &val);
        *dest = *dest OP val;
    }
}
```

Essa otimização se chama **code motion**: identificar um cálculo que é feito várias vezes, mas cujo resultado não muda, e movê-lo para um ponto do código onde ele é feito só uma vez. O compilador não faz isso sozinho por causa do blocker das chamadas de função: ele não sabe se `vec_length` tem efeitos colaterais ou se o resultado pode mudar entre as iterações.

O livro mostra um caso extremo com uma função que converte uma string para minúsculas:

```c
void lower1(char *s) {
    long i;
    for (i = 0; i < strlen(s); i++)
        if (s[i] >= 'A' && s[i] <= 'Z')
            s[i] -= ('A' - 'a');
}
```

O `strlen` percorre a string inteira para descobrir o tamanho, e ele é chamado em toda iteração. Com uma string de n caracteres, o tempo cresce com n². Para strings pequenas, ninguém percebe. Para strings grandes, a função fica inutilizável. A correção é a mesma, calcular o tamanho uma vez antes do laço:

```c
void lower2(char *s) {
    long i;
    long len = strlen(s);
    for (i = 0; i < len; i++)
        if (s[i] >= 'A' && s[i] <= 'Z')
            s[i] -= ('A' - 'a');
}
```

Aqui o compilador tem ainda menos chance de otimizar, porque o laço altera a string, e ele não consegue provar que o tamanho continua o mesmo. É o tipo de ineficiência escondida que só aparece quando o programa recebe uma entrada grande.

## Reduzindo chamadas de procedimento

A `combine2` ainda chama `get_vec_element` em toda iteração, e essa função verifica os limites do índice toda vez, mesmo sabendo que o índice é sempre válido. A `combine3` troca essa chamada por uma função que retorna o ponteiro para o início dos dados, e acessa os elementos diretamente:

```c
data_t *get_vec_start(vec_ptr v) {
    return v->data;
}

void combine3(vec_ptr v, data_t *dest) {
    long i;
    long length = vec_length(v);
    data_t *data = get_vec_start(v);
    *dest = IDENT;
    for (i = 0; i < length; i++) {
        *dest = *dest OP data[i];
    }
}
```

O resultado surpreende: o ganho de desempenho é praticamente nenhum. O gargalo está em outras operações do laço, que a próxima seção resolve. Além disso, essa mudança tem um custo: o código passa a depender de como o vetor é guardado na memória, quebrando a abstração.

## Eliminando referências desnecessárias à memória

Olhando o assembly da `combine3`, a cada iteração o programa lê `*dest` da memória, faz a operação e escreve o resultado de volta em `*dest`. Na iteração seguinte, ele lê de novo o valor que acabou de escrever. É um desperdício. A `combine4` acumula o resultado em uma variável local, que fica em um registrador, e só escreve em `*dest` no final:

```c
void combine4(vec_ptr v, data_t *dest) {
    long i;
    long length = vec_length(v);
    data_t *data = get_vec_start(v);
    data_t acc = IDENT;
    for (i = 0; i < length; i++) {
        acc = acc OP data[i];
    }
    *dest = acc;
}
```

Os acessos à memória por iteração caem de duas leituras e uma escrita para uma leitura só, e o CPE cai bastante.

Por que o compilador não faz isso sozinho? Por causa do **memory aliasing**. Se `dest` apontar para um elemento do próprio vetor, as duas versões dão resultados diferentes. Com o vetor `[2, 3, 5]`, a operação de produto e `dest` apontando para o último elemento:

- `combine3` escreve em `v[2]` a cada iteração e lê o valor alterado: 1, depois 2, depois 6, e por fim 6 × 6 = **36**.
- `combine4` só escreve no final: 2 × 3 × 5 = **30**.

Como o compilador não sabe se esse caso vai acontecer, ele mantém as leituras e escritas na memória. Cabe ao programador, que sabe que `dest` não aponta para dentro do vetor, fazer essa otimização.

## Entendendo os processadores modernos

Até aqui, as otimizações não dependiam do processador. Para ir além, é preciso entender o que o hardware faz para executar o código mais rápido. Processadores modernos são *superescalares*: executam várias instruções por ciclo, e fora de ordem (*out-of-order*), ou seja, não necessariamente na ordem em que aparecem no programa. Tudo isso acontece sem o programador ver, e o resultado final é sempre o mesmo que o da execução sequencial.

### Branch prediction e speculative execution

Para manter o processador ocupado, ele precisa buscar e decodificar instruções bem antes de executá-las. O problema são os desvios (um `if` ou a condição de um laço): o processador só sabe qual caminho seguir depois de calcular a condição, e esperar por isso desperdiçaria muitos ciclos.

A solução é a **branch prediction** (predição de desvio): o processador tenta adivinhar se o desvio vai ser tomado ou não e continua buscando instruções pelo caminho previsto. Junto com ela vem a **speculative execution** (execução especulativa): ele já começa a executar essas instruções antes de saber se a previsão está certa. Os resultados ficam guardados à parte e só passam a valer quando a previsão é confirmada.

Se a previsão estiver certa, o processador ganhou tempo. Se estiver errada, ele descarta tudo o que executou pelo caminho errado e recomeça pelo caminho certo. Isso tem um custo, a *misprediction penalty*, que no processador usado pelo livro fica por volta de 19 ciclos. Por isso, desvios difíceis de prever (por exemplo, que dependem de dados aleatórios) podem deixar um programa bem mais lento.

### Unidades funcionais: latência, issue time e capacidade

O processador tem várias unidades funcionais, cada uma especializada em um tipo de operação (soma de inteiros, multiplicação, ponto flutuante, leitura e escrita na memória). O desempenho de cada operação é descrito por três números:

- **Latência:** quantos ciclos uma operação leva do início ao fim.
- **Issue time:** o intervalo mínimo, em ciclos, entre o início de duas operações do mesmo tipo. Quando é 1, a unidade é *pipelined*: começa uma operação nova a cada ciclo, mesmo que as anteriores ainda não tenham terminado.
- **Capacidade:** quantas unidades daquele tipo existem.

Uma multiplicação de ponto flutuante, por exemplo, tem latência de 5 ciclos, mas como a unidade é pipelined, uma nova multiplicação pode começar a cada ciclo.

### Latency bound e throughput bound

Esses números definem dois limites para o CPE:

- **Latency bound:** quando cada operação depende do resultado da anterior, elas precisam ser feitas estritamente em sequência, e o CPE não consegue ser menor que a latência da operação.
- **Throughput bound:** mesmo sem dependências, o processador não consegue passar do número de operações que as unidades funcionais conseguem iniciar por ciclo. É o limite máximo do hardware.

A `combine4` está presa no latency bound. Cada iteração precisa do valor de `acc` calculado na iteração anterior, então as operações sobre `acc` formam uma corrente de dependências, o *caminho crítico* do laço. Não importa quantas unidades funcionais o processador tenha: ele não consegue começar a próxima operação antes de a anterior terminar. Para passar desse limite, é preciso reescrever o código para quebrar essa corrente, que é o assunto das próximas seções do livro.

## Aumentando o paralelismo

Na `combine4`, a soma (ou o produto) de cada elemento depende do resultado do elemento anterior, e o processador fica esperando a latência de cada operação. As unidades funcionais conseguiriam fazer mais, mas o código não deixa. A saída é reescrever o laço para que existam operações independentes.

### Múltiplos acumuladores

A ideia é dividir o trabalho entre dois (ou mais) acumuladores. Um combina os elementos de índice par e o outro, os de índice ímpar. Para isso, o laço processa dois elementos por iteração (*loop unrolling*):

```c
void combine6(vec_ptr v, data_t *dest) {
    long i;
    long length = vec_length(v);
    long limit = length - 1;
    data_t *data = get_vec_start(v);
    data_t acc0 = IDENT;
    data_t acc1 = IDENT;

    for (i = 0; i < limit; i += 2) {
        acc0 = acc0 OP data[i];
        acc1 = acc1 OP data[i+1];
    }
    /* Elemento que sobrou, se o tamanho for ímpar */
    for (; i < length; i++)
        acc0 = acc0 OP data[i];

    *dest = acc0 OP acc1;
}
```

Agora existem duas correntes de dependência independentes, e o processador consegue executar as duas ao mesmo tempo. O CPE cai pela metade. Com mais acumuladores, o desempenho se aproxima do throughput bound.

O compilador não faz isso sozinho com números de ponto flutuante, porque a soma e a multiplicação de ponto flutuante não são associativas: mudar a ordem das operações pode mudar o arredondamento e, em casos extremos, o resultado.

### Reassociação

Outra forma de quebrar a corrente é só mudar os parênteses. Em vez de:

```c
acc = (acc OP data[i]) OP data[i+1];
```

escrever:

```c
acc = acc OP (data[i] OP data[i+1]);
```

Na segunda versão, a operação `data[i] OP data[i+1]` não depende de `acc`, então pode ser feita em paralelo com a iteração anterior. Só uma operação a cada dois elementos fica no caminho crítico. O ganho é parecido com o dos múltiplos acumuladores.

## Resumo dos resultados

Partindo da `combine1` e aplicando as otimizações uma a uma (code motion, menos chamadas de função, menos acessos à memória, loop unrolling e paralelismo), o CPE cai mais de dez vezes, chegando perto do throughput bound do processador. Para ir além, só com instruções vetoriais (SIMD), que fazem a mesma operação sobre vários dados de uma vez.

## Fatores limitantes

### Register spilling

Os acumuladores precisam ficar em registradores, e o número de registradores é limitado. Se o laço usar mais variáveis do que existem registradores, o compilador precisa guardar algumas na pilha (*register spilling*), e elas voltam a ser lidas e escritas na memória. Nesse caso, aumentar o número de acumuladores deixa o programa mais lento, não mais rápido.

### Branch misprediction

Não é preciso se preocupar com todos os desvios. Os que são fáceis de prever, como o desvio que fecha um laço ou a verificação de limites de um vetor, quase sempre são acertados pelo processador e custam pouco.

O problema são os desvios que dependem de dados imprevisíveis. Nesses casos, vale escrever o código de forma que o compilador consiga usar *conditional moves* (a instrução `cmov`), que calculam os dois resultados e escolhem um, sem desvio nenhum. Compare:

```c
/* Com desvio: lento se a comparação for imprevisível */
void minmax1(long a[], long b[], long n) {
    long i;
    for (i = 0; i < n; i++) {
        if (a[i] > b[i]) {
            long t = a[i];
            a[i] = b[i];
            b[i] = t;
        }
    }
}

/* Sem desvio: o compilador pode usar conditional moves */
void minmax2(long a[], long b[], long n) {
    long i;
    for (i = 0; i < n; i++) {
        long min = a[i] < b[i] ? a[i] : b[i];
        long max = a[i] < b[i] ? b[i] : a[i];
        a[i] = min;
        b[i] = max;
    }
}
```

A `minmax2` tem o mesmo desempenho com dados previsíveis ou aleatórios, enquanto a `minmax1` fica bem mais lenta quando os dados são aleatórios.

## Entendendo o desempenho da memória

Leituras da memória também têm latência, mesmo quando o dado está na cache L1. Quando cada leitura depende do resultado da anterior, como ao percorrer uma lista encadeada (o endereço do próximo nó só é conhecido depois de ler o nó atual), o CPE fica preso na latência da leitura.

As escritas, por outro lado, normalmente não atrasam o programa: elas vão para um *store buffer* e são feitas na memória depois, enquanto o processador continua. O problema aparece quando o programa lê um endereço que acabou de escrever. A leitura precisa esperar a escrita, o que cria uma dependência entre as duas, e o desempenho cai.

## Na prática: técnicas para melhorar o desempenho

O livro resume as técnicas do capítulo em três níveis:

1. **Projeto de alto nível:** escolher algoritmos e estruturas de dados adequados. Nenhum truque compensa um algoritmo com complexidade ruim.
2. **Princípios básicos de código:** evitar os optimization blockers.
   - Tirar de dentro dos laços as chamadas de função e os cálculos que não mudam.
   - Usar variáveis locais para guardar resultados intermediários, em vez de ler e escrever na memória o tempo todo.
3. **Otimizações de baixo nível:**
   - Loop unrolling.
   - Múltiplos acumuladores e reassociação, para aumentar o paralelismo.
   - Escrever condicionais em um estilo que permita conditional moves.

Também é preciso cuidado para não introduzir erros ao otimizar. Código otimizado costuma ser mais difícil de ler, então vale testar bem e comparar com a versão original.

## Identificando e eliminando gargalos

Em programas grandes, é difícil saber onde o tempo está sendo gasto só olhando o código. Para isso existem os *profilers*, como o `gprof`: o programa é compilado com a opção `-pg`, executado normalmente, e o `gprof` mostra quanto tempo foi gasto em cada função e quantas vezes cada uma foi chamada.

No exemplo do livro, um programa que conta as sequências de palavras mais frequentes nas obras de Shakespeare, o profiling mostra que quase todo o tempo ia para uma ordenação ineficiente. Trocando o algoritmo de ordenação, o gargalo passa para outras partes do programa, que vão sendo melhoradas uma de cada vez.

A [lei de Amdahl](/concorrencia-x-paralelismo/#lei-de-amdahl) explica por que vale a pena começar pelos gargalos: o ganho total é limitado pela parte do programa que não foi otimizada. Acelerar muito uma função que ocupa 5% do tempo quase não muda o resultado final.
