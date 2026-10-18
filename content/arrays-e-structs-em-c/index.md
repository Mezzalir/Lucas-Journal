+++
date = '2026-10-18T15:03:43-03:00'
draft = false
title = 'Arrays e Structs em C'
type = 'blog'
tags = ['Arquitetura', 'Assembly', 'Memória', 'CSAPP']
+++

Arrays em C são uma forma de agrupar dados escalares em tipos de dados maiores. C usa uma implementação de arrays particularmente simples, então a tradução para código de máquina é bem direta. Uma característica incomum de C é que podemos gerar ponteiros para elementos dentro de arrays e fazer aritmética com esses ponteiros. Essas operações são traduzidas em cálculos de endereço no código de máquina.

Compiladores com otimização são particularmente bons em simplificar os cálculos de endereço usados na indexação de arrays. Isso pode tornar a correspondência entre o código C e sua tradução em código de máquina um tanto difícil de decifrar.

---

## Princípios básicos

Para um tipo de dado `T` e uma constante inteira `N`, considere uma declaração da forma:

```c
T A[N];
```

Vamos chamar a posição inicial de `x_A`. A declaração tem dois efeitos. *Primeiro, ela aloca uma região contígua de `L · N` bytes na memória, onde `L` é o tamanho (em bytes) do tipo `T`. Segundo, ela introduz um identificador `A` que pode ser usado como um ponteiro para o início do array.* O valor desse ponteiro será `x_A`. Os elementos do array podem ser acessados com um índice inteiro que vai de 0 até `N - 1`. O elemento `i` do array será armazenado no endereço `x_A + L · i`.

Como exemplo, considere as seguintes declarações:

```c
char    A[12];
char   *B[8];
int     C[6];
double *D[5];
```

Essas declarações vão gerar arrays com os seguintes parâmetros:

| Array | Tamanho do elemento | Tamanho total | Endereço inicial | Elemento `i` |
|-------|---------------------|---------------|------------------|--------------|
| `A`   | 1                   | 12            | `x_A`            | `x_A + i`    |
| `B`   | 8                   | 64            | `x_B`            | `x_B + 8i`   |
| `C`   | 4                   | 24            | `x_C`            | `x_C + 4i`   |
| `D`   | 8                   | 40            | `x_D`            | `x_D + 8i`   |

O array `A` é composto por 12 elementos de um byte (`char`). O array `C` é composto por 6 inteiros, cada um ocupando 4 bytes. `B` e `D` são arrays de ponteiros, então cada elemento ocupa 8 bytes.

As instruções de acesso à memória do x86-64 foram projetadas para simplificar o acesso a arrays. Por exemplo, suponha que `E` é um array de valores do tipo `int` e queremos calcular `E[i]`, onde o endereço de `E` está no registrador `%rdx` e `i` está no registrador `%rcx`. Então a instrução

```asm
movl (%rdx,%rcx,4), %eax
```

vai fazer o cálculo de endereço `x_E + 4i`, ler essa posição de memória e copiar o resultado para o registrador `%eax`. Os fatores de escala permitidos, 1, 2, 4 e 8, cobrem os tamanhos dos tipos de dados primitivos mais comuns.

## Aritmética de ponteiros

C permite aritmética com ponteiros, onde o valor calculado é escalado de acordo com o tamanho do tipo de dado referenciado pelo ponteiro. Ou seja, se `p` é um ponteiro para um dado do tipo `T` e o valor de `p` é `x_p`, então a expressão `p+i` tem valor `x_p + L · i`, onde `L` é o tamanho do tipo `T`.

Os operadores unários `&` e `*` permitem gerar e desreferenciar ponteiros. Ou seja, para uma expressão `Expr` que denota algum objeto, `&Expr` é um ponteiro que dá o endereço do objeto. Para uma expressão `AExpr` que denota um endereço, `*AExpr` dá o valor naquele endereço. As expressões `Expr` e `*&Expr` são, portanto, equivalentes. A operação de indexação pode ser aplicada tanto a arrays quanto a ponteiros. A referência `A[i]` é idêntica à expressão `*(A+i)`. Ela calcula o endereço do `i`-ésimo elemento do array e depois acessa essa posição de memória.

Expandindo o exemplo anterior, suponha que o endereço inicial do array de inteiros `E` e o índice inteiro `i` estão nos registradores `%rdx` e `%rcx`, respectivamente. A seguir estão algumas expressões envolvendo `E`. Também mostramos uma implementação em assembly de cada expressão, com o resultado armazenado no registrador `%eax` (para dados) ou no registrador `%rax` (para ponteiros).

| Expressão  | Tipo    | Valor                  | Código assembly                   |
|------------|---------|------------------------|-----------------------------------|
| `E`        | `int *` | `x_E`                  | `movl %rdx,%rax`                  |
| `E[0]`     | `int`   | `M[x_E]`               | `movl (%rdx),%eax`                |
| `E[i]`     | `int`   | `M[x_E + 4i]`          | `movl (%rdx,%rcx,4),%eax`         |
| `&E[2]`    | `int *` | `x_E + 8`              | `leaq 8(%rdx),%rax`               |
| `E+i-1`    | `int *` | `x_E + 4i - 4`         | `leaq -4(%rdx,%rcx,4),%rax`       |
| `*(E+i-3)` | `int`   | `M[x_E + 4i - 12]`     | `movl -12(%rdx,%rcx,4),%eax`      |
| `&E[i]-E`  | `long`  | `i`                    | `movq %rcx,%rax`                  |

## Structs

A declaração `struct` de C cria um tipo de dado que agrupa objetos de tipos possivelmente diferentes em um único objeto. Os diferentes componentes de uma struct são referenciados por nomes. *A implementação de structs é parecida com a de arrays: todos os componentes de uma struct são armazenados em uma região contígua de memória, e um ponteiro para uma struct é o endereço do seu primeiro byte.* O compilador mantém informações sobre cada tipo de struct indicando o deslocamento (*offset*) em bytes de cada campo. Ele gera referências aos elementos da struct usando esses offsets como deslocamentos nas instruções de acesso à memória.

Como exemplo, considere a seguinte declaração de struct:

```c
struct rec {
    int i;
    int j;
    int a[2];
    int *p;
};
```

Essa struct contém quatro campos: dois valores de 4 bytes do tipo `int`, um array de dois elementos do tipo `int` e um ponteiro para inteiro de 8 bytes, totalizando 24 bytes:

```text
Offset    0       4       8       12      16              24
          +-------+-------+-------+-------+---------------+
Conteúdo  |   i   |   j   | a[0]  | a[1]  |       p       |
          +-------+-------+-------+-------+---------------+
```

Observe que o array `a` está embutido dentro da struct. Os números no topo do diagrama dão os offsets em bytes dos campos a partir do início da struct.

Para acessar os campos de uma struct, o compilador gera código que soma o offset apropriado ao endereço da struct. Por exemplo, suponha que a variável `r`, do tipo `struct rec *`, está no registrador `%rdi`. Então o código a seguir copia o elemento `r->i` para o elemento `r->j`:

```asm
# Registradores: r em %rdi
movl    (%rdi), %eax       # Lê r->i
movl    %eax, 4(%rdi)      # Armazena em r->j
```

Como o offset do campo `i` é 0, o endereço desse campo é simplesmente o valor de `r`. Para armazenar no campo `j`, o código soma o offset 4 ao endereço de `r`.

Para gerar um ponteiro para um objeto dentro de uma struct, basta somar o offset do campo ao endereço da struct. Por exemplo, podemos gerar o ponteiro `&(r->a[1])` somando o offset `8 + 4 · 1 = 12`. Para o ponteiro `r` no registrador `%rdi` e a variável inteira `long i` no registrador `%rsi`, podemos gerar o valor do ponteiro `&(r->a[i])` com uma única instrução:

```asm
# Registradores: r em %rdi, i em %rsi
leaq    8(%rdi,%rsi,4), %rax    # %rax recebe &r->a[i]
```

Como exemplo final, o código a seguir implementa a instrução

```c
r->p = &r->a[r->i + r->j];
```

começando com `r` no registrador `%rdi`:

```asm
# Registradores: r em %rdi
movl    4(%rdi), %eax           # Lê r->j
addl    (%rdi), %eax            # Soma r->i
cltq                            # Estende para 8 bytes
leaq    8(%rdi,%rax,4), %rax    # Calcula &r->a[r->i + r->j]
movq    %rax, 16(%rdi)          # Armazena em r->p
```

Como esses exemplos mostram, a seleção dos diferentes campos de uma struct é resolvida inteiramente em tempo de compilação. O código de máquina não contém nenhuma informação sobre as declarações dos campos nem sobre os nomes deles.
