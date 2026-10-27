+++
date = '2026-10-27T16:00:49-03:00'
draft = false
title = 'Vetor de ponteiros'
type = 'blog'
tags = ['C', 'Memória']
+++

Um **vetor de ponteiros** é um vetor comum cujos elementos são endereços. Ele já apareceu várias vezes nos posts anteriores sem ganhar um nome próprio: o `argv` e o `envp` da `main`, o `environ`, o segundo e o terceiro argumentos do `execve`. Todos são vetores de ponteiros para strings, terminados em `NULL`.

Este post explica como eles funcionam em C: como declarar e ler o tipo, como ficam na memória, a diferença para uma matriz, como passá-los para funções, como alocá-los dinamicamente e onde costumam aparecer os erros. Todos os exemplos foram compilados e executados no Linux (x86-64), e as saídas mostradas são as reais.

## Declaração

```c
int *v[3];
```

Essa linha declara `v` como um **vetor de 3 elementos, cada um do tipo `int *`**. A forma de ler é começar pelo nome e seguir a precedência: `[]` tem precedência maior que `*`, então `v` é primeiro um vetor (`v[3]`), e o que sobra (`int *`) é o tipo de cada elemento.

Com parênteses, o sentido muda:

| Declaração | Leitura | `sizeof` (x86-64) |
|---|---|---|
| `int *v[3]` | `v` é um vetor de 3 ponteiros para `int` | 24 (3 × 8) |
| `int (*p)[3]` | `p` é um ponteiro para um vetor de 3 `int` | 8 |

O segundo tipo é o que aparece ao passar uma matriz `int m[2][3]` para uma função: `p + 1` avança uma linha inteira, ou 12 bytes.

```c
int m[2][3] = {{1, 2, 3}, {4, 5, 6}};
int *v[3];                      /* vetor de 3 ponteiros para int */
int (*p)[3] = m;                /* ponteiro para um vetor de 3 ints */

printf("sizeof(v) = %zu   sizeof(p) = %zu\n", sizeof(v), sizeof(p));
printf("p[1][2] = %d\n", p[1][2]);
printf("p = %p   p + 1 = %p\n", (void *)p, (void *)(p + 1));
```

```text
sizeof(v) = 24   sizeof(p) = 8
p[1][2] = 6
p = 0x7ffdd7d01640   p + 1 = 0x7ffdd7d0164c
```

O resto do post trata do primeiro caso, o vetor de ponteiros.

## Como fica na memória

Um vetor de ponteiros é guardado como qualquer outro vetor: os elementos ficam **lado a lado** na memória. A diferença é que cada elemento é um endereço de 8 bytes, e os dados para os quais eles apontam podem estar em qualquer lugar.

```c
int main(void) {
    int a = 10, b = 20, c = 30;
    int *v[3] = {&a, &b, &c};

    printf("sizeof(v)    = %zu\n", sizeof(v));
    printf("sizeof(v[0]) = %zu\n\n", sizeof(v[0]));

    for (int i = 0; i < 3; i++)
        printf("&v[%d] = %p   v[%d] = %p   *v[%d] = %d\n",
               i, (void *)&v[i], i, (void *)v[i], i, *v[i]);

    *v[1] = 99;                     /* escreve em b através do ponteiro */
    printf("\nb = %d\n", b);
    return 0;
}
```

```text
sizeof(v)    = 24
sizeof(v[0]) = 8

&v[0] = 0x7ffeec4db9c0   v[0] = 0x7ffeec4db9b0   *v[0] = 10
&v[1] = 0x7ffeec4db9c8   v[1] = 0x7ffeec4db9b4   *v[1] = 20
&v[2] = 0x7ffeec4db9d0   v[2] = 0x7ffeec4db9b8   *v[2] = 30

b = 99
```

Cada coluna mostra uma coisa diferente:

- **`&v[i]`** é o endereço do elemento dentro do vetor. Os endereços sobem de 8 em 8 (`…9c0`, `…9c8`, `…9d0`), porque cada elemento é um ponteiro de 8 bytes.
- **`v[i]`** é o valor guardado no elemento: o endereço de `a`, `b` ou `c`. Esses sobem de 4 em 4, porque são `int`.
- **`*v[i]`** é o valor no endereço apontado. A leitura dá **dois saltos**: primeiro busca `v[i]` no vetor, depois vai até o endereço lido.

Desenhando:

```text
          v (na pilha)                        variáveis (na pilha)
        ┌────────────────┐
…9c0    │ 0x7ffeec4db9b0 │ ── v[0] ──►  …9b0  │ 10 │  a
        ├────────────────┤
…9c8    │ 0x7ffeec4db9b4 │ ── v[1] ──►  …9b4  │ 99 │  b
        ├────────────────┤
…9d0    │ 0x7ffeec4db9b8 │ ── v[2] ──►  …9b8  │ 30 │  c
        └────────────────┘
```

A linha `*v[1] = 99` não mexeu em `v`. Ela seguiu o ponteiro e escreveu em `b`, e por isso o `printf` final mostra `b = 99`.

## Vetor de strings

O uso mais comum de um vetor de ponteiros é guardar strings. Em C, uma string é um vetor de `char` terminado em `'\0'`, e um `char *` aponta para o primeiro caractere dela. Um vetor de `char *` é então uma lista de strings.

Existem duas formas de declarar uma lista de nomes, e elas parecem iguais no uso, mas são bem diferentes na memória:

```c
char *p[] = {"Ana", "Bruno", "Carla"};    /* vetor de ponteiros */
char  m[][6] = {"Ana", "Bruno", "Carla"}; /* matriz de caracteres */
```

```c
printf("sizeof(p) = %zu   sizeof(m) = %zu\n\n", sizeof(p), sizeof(m));

for (int i = 0; i < 3; i++)
    printf("p[%d] em %p aponta para %p \"%s\"\n",
           i, (void *)&p[i], (void *)p[i], p[i]);
printf("\n");
for (int i = 0; i < 3; i++)
    printf("m[%d] em %p \"%s\"\n", i, (void *)m[i], m[i]);

printf("\np[1][2] = '%c'   m[1][2] = '%c'\n", p[1][2], m[1][2]);
```

```text
sizeof(p) = 24   sizeof(m) = 18

p[0] em 0x7ffe707ecbc0 aponta para 0x5f29321df008 "Ana"
p[1] em 0x7ffe707ecbc8 aponta para 0x5f29321df00c "Bruno"
p[2] em 0x7ffe707ecbd0 aponta para 0x5f29321df012 "Carla"

m[0] em 0x7ffe707ecba0 "Ana"
m[1] em 0x7ffe707ecba6 "Bruno"
m[2] em 0x7ffe707ecbac "Carla"

p[1][2] = 'u'   m[1][2] = 'u'
```

- **`p`** tem 24 bytes: só os três ponteiros, que ficam na pilha (`0x7ffe…`). As strings ficam em outro lugar, na área de dados somente leitura do programa (`0x5f29…`), uma logo depois da outra, cada uma com o tamanho exato.
- **`m`** tem 18 bytes: 3 linhas de 6 `char`, com os caracteres copiados para dentro da própria matriz, na pilha. Todas as linhas têm o tamanho da maior string (`"Bruno"` e `"Carla"`, 5 letras + `'\0'`). `"Ana"` usa só 4 dos seus 6 bytes.

```text
p (pilha)                   strings (somente leitura)
┌──────────┐
│ …df008 ──┼──────────►  A n a \0
├──────────┤
│ …df00c ──┼──────────►  B r u n o \0
├──────────┤
│ …df012 ──┼──────────►  C a r l a \0
└──────────┘

m (pilha, 3 linhas de 6 bytes)
┌─────────────┬─────────────┬─────────────┐
│ A n a \0 · ·│ B r u n o \0│ C a r l a \0│
└─────────────┴─────────────┴─────────────┘
```

As duas formas aceitam a mesma sintaxe, `p[1][2]` e `m[1][2]`, mas o caminho até o caractere é diferente:

- Em **`m[1][2]`**, o compilador calcula um único endereço, `m + 1 × 6 + 2`, e faz uma leitura.
- Em **`p[1][2]`**, o programa lê o ponteiro `p[1]` da memória e só depois soma 2 a ele: são duas leituras.

### O que dá para modificar

A diferença de memória decide o que pode ser alterado:

```c
char *p[] = {"Ana", "Bruno"};
char  m[][6] = {"Ana", "Bruno"};

m[0][0] = 'E';                  /* ok: a matriz é uma cópia na pilha */
printf("m[0] = %s\n", m[0]);

p[0] = "Eva";                   /* ok: só troca o ponteiro */
printf("p[0] = %s\n", p[0]);

p[1][0] = 'X';                  /* erro: string literal é somente leitura */
printf("p[1] = %s\n", p[1]);
```

```text
m[0] = Ena
p[0] = Eva
Segmentation fault         (core dumped)
```

- Mudar um caractere de `m` funciona, porque os caracteres são uma cópia que pertence à matriz.
- Trocar `p[0]` funciona, porque o vetor é nosso: só fizemos o elemento apontar para outra string.
- Escrever em `p[1][0]` derruba o programa. A string `"Bruno"` é um **literal**, e os literais ficam numa área de memória marcada como somente leitura. A escrita causa um page fault que o kernel transforma em `SIGSEGV`.

Para deixar esse erro visível já na compilação, declare o vetor como `const char *p[]`. Aí `p[1][0] = 'X'` nem compila.

### Quando usar cada um

| | Vetor de ponteiros (`char *p[]`) | Matriz (`char m[][N]`) |
|---|---|---|
| Tamanho das strings | Cada uma do seu tamanho | Todas do tamanho da maior |
| Onde ficam os caracteres | Em qualquer lugar (literais, heap…) | Dentro da própria matriz |
| Trocar a ordem | Barato: troca 8 bytes de ponteiro | Caro: copia a linha inteira |
| Acesso a um caractere | Duas leituras | Uma leitura |
| Alterar os caracteres | Só se não forem literais | Sempre |

## Passando para uma função

Quando o nome de um vetor é usado numa expressão, ele **decai** para um ponteiro para o primeiro elemento. O primeiro elemento de um vetor de `char *` é um `char *`, então um ponteiro para ele é um **`char **`**. É por isso que estas duas declarações de parâmetro são idênticas:

```c
void imprime(char *v[]);
void imprime(char **v);
```

O mesmo vale para a `main`: `char *argv[]` e `char **argv` são a mesma coisa.

A consequência é que **a função não sabe o tamanho do vetor**. Ela recebe só um endereço:

```c
void imprime(char *v[]) {           /* o mesmo que char **v */
    printf("dentro da função: sizeof(v) = %zu\n", sizeof(v));
    for (char **p = v; *p != NULL; p++)
        printf("  %s\n", *p);
}

int main(void) {
    char *cores[] = {"vermelho", "verde", "azul", NULL};
    printf("na main: sizeof(cores) = %zu\n", sizeof(cores));
    imprime(cores);
    return 0;
}
```

```text
na main: sizeof(cores) = 32
dentro da função: sizeof(v) = 8
  vermelho
  verde
  azul
```

Na `main`, `sizeof` vê o vetor inteiro (4 ponteiros, 32 bytes). Dentro da função, vê só um ponteiro (8 bytes). O próprio gcc avisa sobre isso com `-Wall`: *"'sizeof' on array function parameter 'v' will return size of 'char \*\*'"*.

Há duas formas de a função saber onde o vetor termina:

1. **Passar o tamanho junto**, como em `main(int argc, char *argv[])`.
2. **Terminar o vetor com `NULL`**, como no exemplo. É a convenção do `argv`, do `envp`, do `environ` e do `execve`. Por isso todos eles terminam em `NULL`: quem recebe percorre até achar o `NULL`.

O laço da função mostra a forma idiomática de percorrer um vetor terminado em `NULL`:

- `p` é um `char **` que começa no primeiro elemento;
- `*p` é a string atual (um `char *`);
- `p++` avança 8 bytes, para o próximo elemento;
- o laço para quando `*p` é `NULL`.

## Alocação dinâmica

Quando a quantidade de strings só é conhecida durante a execução, o vetor é alocado com `malloc`. Agora são **dois níveis de alocação**: um bloco para o vetor de ponteiros e um bloco para cada string.

O programa abaixo lê linhas da entrada padrão, guarda todas e imprime de trás para frente:

```c
int main(void) {
    char buf[100];
    int n = 0, cap = 2;
    char **linhas = malloc(cap * sizeof *linhas);

    while (fgets(buf, sizeof buf, stdin) != NULL) {
        buf[strcspn(buf, "\n")] = '\0';
        if (n == cap) {             /* cheio: dobra o vetor */
            cap *= 2;
            linhas = realloc(linhas, cap * sizeof *linhas);
        }
        linhas[n++] = strdup(buf);  /* cada string tem a sua memória */
    }

    for (int i = n - 1; i >= 0; i--)
        printf("%d: %s\n", i, linhas[i]);

    for (int i = 0; i < n; i++)     /* primeiro as strings... */
        free(linhas[i]);
    free(linhas);                   /* ...depois o vetor */
    return 0;
}
```

```text
$ printf 'primeira\nsegunda\nterceira\nquarta\nquinta\n' | ./dinamico
4: quinta
3: quarta
2: terceira
1: segunda
0: primeira
```

Alguns detalhes:

- **`sizeof *linhas`** é o tamanho de um elemento, ou seja, de um `char *` (8 bytes). Escrever assim, em vez de `sizeof(char *)`, continua certo mesmo se o tipo de `linhas` mudar.
- **`realloc`** aumenta o vetor de ponteiros quando ele enche. Ele pode mover o vetor para outro endereço, mas as strings não se movem: só os ponteiros para elas são copiados.
- **`strdup`** é necessário porque `buf` é reaproveitado a cada linha. Se o código fizesse `linhas[n++] = buf`, todos os elementos apontariam para o mesmo buffer, e todas as linhas impressas seriam a última lida.
- **A liberação segue a ordem inversa da alocação**: primeiro cada string, depois o vetor. Se o vetor fosse liberado primeiro, os endereços das strings se perderiam junto com ele, e elas nunca mais poderiam ser liberadas.

## Ordenando com qsort

Como cada elemento é só um ponteiro, ordenar um vetor de strings é rápido: o `qsort` troca ponteiros de 8 bytes e nunca copia os caracteres. Mas a função de comparação é um lugar clássico de erro.

```c
void qsort(void *base, size_t nmemb, size_t size,
           int (*compar)(const void *, const void *));
```

O `qsort` não sabe o tipo dos elementos. Ele passa para `compar` **ponteiros para os elementos**. Se cada elemento é um `char *`, o que chega é um ponteiro para um `char *`, ou seja, um `char **` disfarçado de `void *`. A comparação precisa desfazer esse nível antes de chamar `strcmp`:

```c
int compara(const void *a, const void *b) {
    const char *s1 = *(const char * const *)a;  /* a aponta para um char * */
    const char *s2 = *(const char * const *)b;
    return strcmp(s1, s2);
}

int main(void) {
    char *nomes[] = {"Carla", "Ana", "Eduardo", "Bruno", "Diana"};
    int n = sizeof nomes / sizeof nomes[0];

    printf("antes:  \"Ana\" está em %p\n", (void *)nomes[1]);
    qsort(nomes, n, sizeof nomes[0], compara);
    printf("depois: \"Ana\" está em %p\n\n", (void *)nomes[0]);

    for (int i = 0; i < n; i++)
        printf("%s ", nomes[i]);
    printf("\n");
    return 0;
}
```

```text
antes:  "Ana" está em 0x59f11da0800a
depois: "Ana" está em 0x59f11da0800a

Ana Bruno Carla Diana Eduardo 
```

"Ana" mudou de posição no vetor (de `nomes[1]` para `nomes[0]`), mas a string continua no mesmo endereço. Só o ponteiro foi movido.

O erro comum é passar `a` e `b` direto para o `strcmp`:

```c
int compara(const void *a, const void *b) {
    return strcmp(a, b);            /* errado: a e b não são as strings */
}
```

```text
Carla Ana Eduardo Bruno Diana 
```

O código compila sem aviso, porque `void *` converte para `const char *` sem reclamar, mas o vetor continuou desordenado. `a` e `b` apontam para os elementos do vetor, então o `strcmp` comparou os bytes dos **endereços** guardados lá, e não os nomes.

## Vetor de ponteiros para funções

Os elementos também podem ser ponteiros para funções. A declaração segue a mesma regra de precedência:

```c
int soma(int a, int b)  { return a + b; }
int sub(int a, int b)   { return a - b; }
int mult(int a, int b)  { return a * b; }

int main(void) {
    int (*ops[])(int, int) = {soma, sub, mult};
    char simbolos[] = {'+', '-', '*'};

    for (int i = 0; i < 3; i++)
        printf("7 %c 3 = %d\n", simbolos[i], ops[i](7, 3));
    return 0;
}
```

```text
7 + 3 = 10
7 - 3 = 4
7 * 3 = 21
```

A declaração `int (*ops[])(int, int)` se lê de dentro para fora:

1. `ops[]`: `ops` é um vetor,
2. `*ops[]`: de ponteiros,
3. `(*ops[])(int, int)`: para funções que recebem dois `int`,
4. `int …`: e devolvem `int`.

`ops[i](7, 3)` busca o endereço no vetor e chama a função que está nele. Isso troca uma sequência de `if` ou `switch` por um índice. É a mesma ideia da tabela de exceções vista em [Exceptional Control Flow](/exceptional-control-flow/), que é um vetor de endereços de tratadores indexado pelo número da exceção.

## Resumo

- **`int *v[N]`** é um vetor de N ponteiros. Como `[]` tem precedência sobre `*`, `v` é primeiro um vetor. **`int (*p)[N]`** é o contrário: um único ponteiro para um vetor.
- **Na memória**, os ponteiros ficam lado a lado (8 bytes cada no x86-64), e os dados apontados podem estar em qualquer lugar. Ler `*v[i]` dá dois saltos: um até o elemento e outro até o dado.
- **Vetor de strings x matriz:**
  - `char *p[]` guarda só endereços, e cada string tem o seu tamanho.
  - `char m[][N]` guarda os caracteres, e todas as linhas têm o tamanho da maior.
  - Literais apontados por `p` são somente leitura: escrever neles causa segmentation fault.
- **Em funções**, o vetor decai para `char **`, e a função perde o tamanho. Ou se passa o tamanho junto (`argc`), ou se termina o vetor com `NULL` (`argv`, `envp`, `environ`).
- **Alocação dinâmica** tem dois níveis: o vetor de ponteiros e cada string. A liberação é na ordem inversa: primeiro as strings, depois o vetor.
- **No `qsort`**, a função de comparação recebe ponteiros para os elementos. Para um vetor de `char *`, isso é um `char **`, que precisa ser desreferenciado antes do `strcmp`.
- **Vetores de ponteiros para funções** permitem escolher a função a ser chamada por um índice, como numa tabela de tratadores.

---

*Exemplos testados no Linux, escrito pelo Claude*
