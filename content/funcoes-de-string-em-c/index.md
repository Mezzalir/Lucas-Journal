+++
date = '2026-10-27T16:09:42-03:00'
draft = false
title = 'Funções de string em C'
type = 'blog'
tags = ['C', 'Memória', 'Segurança']
+++

C não tem um tipo string. Uma string é só um vetor de `char` que termina no caractere nulo `'\0'`, e quem trabalha com ela recebe um `char *` apontando para o primeiro caractere. Nada guarda o tamanho da string nem o tamanho do buffer onde ela está: **o fim é descoberto procurando o `'\0'`**.

As funções do `<string.h>` partem dessa convenção. Este post explica as cinco principais:

| Função | O que faz |
|---|---|
| `strlen` | Conta os caracteres até o `'\0'` |
| `strcpy` | Copia uma string para outro buffer |
| `strcat` | Junta uma string no fim de outra |
| `strcmp` | Compara duas strings |
| `strchr` | Procura um caractere numa string |

Para cada uma: o protótipo, o que significa cada argumento, o que devolve, um exemplo e as armadilhas. Algumas parentes próximas (`strncpy`, `strncat`, `strdup`, `strncmp` e `strrchr`) aparecem dentro dessas seções. Todos os exemplos foram compilados e executados no Linux (x86-64), e as saídas mostradas são as reais.

## strlen: o tamanho da string

```c
#include <string.h>

size_t strlen(const char *s);
```

- **`s`**: ponteiro para o início da string. É `const` porque a função só lê.
- **Retorno:** o número de caracteres antes do `'\0'`, **sem contar o `'\0'`**. O tipo `size_t` é um inteiro sem sinal (8 bytes no x86-64), por isso o `printf` usa `%zu`.

O `strlen` começa em `s` e avança byte a byte até encontrar o `'\0'`. Ele não sabe nada sobre o tamanho do vetor, e é isso que o diferencia do `sizeof`:

```c
char s[20] = "casa";
char *p = "olá";

printf("strlen(s) = %zu   sizeof(s) = %zu\n", strlen(s), sizeof(s));
printf("strlen(p) = %zu   sizeof(p) = %zu\n", strlen(p), sizeof(p));
printf("strlen(\"\") = %zu\n", strlen(""));
```

```text
strlen(s) = 4   sizeof(s) = 20
strlen(p) = 4   sizeof(p) = 8
strlen("") = 0
```

- **`s`** tem espaço para 20 bytes, mas a string guardada nele tem 4 caracteres. O `sizeof` mede o buffer, e o `strlen` mede o conteúdo.
- **`p`** é um ponteiro, então `sizeof(p)` é 8, o tamanho de um endereço. Já `strlen(p)` deu 4 para `"olá"`, que tem 3 letras. O `strlen` conta **bytes**, não letras: em UTF-8, o `á` ocupa 2 bytes.
- **A string vazia** `""` tem só o `'\0'`, então o tamanho é 0.

A regra prática é: **para guardar uma string de tamanho `strlen(s)`, é preciso `strlen(s) + 1` bytes**. O `+ 1` é o lugar do `'\0'`, e esquecê-lo é um dos erros mais comuns em C.

### Armadilhas do strlen

**Sem `'\0'`, o `strlen` não para.** Se o vetor não tiver o terminador, a função continua lendo a memória depois dele até achar algum byte zero por acaso. O resultado é um tamanho errado, ou um segmentation fault se a leitura chegar a uma página não mapeada.

**O `strlen` custa O(n).** Ele percorre a string inteira toda vez que é chamado, sem guardar resultado nenhum. O problema aparece quando ele está na condição de um laço:

```c
for (size_t i = 0; i < strlen(s); i++)   /* strlen a cada volta */
    s[i] = toupper(s[i]);
```

A cada volta o `strlen` percorre a string de novo, e o laço inteiro vira O(n²). Calculando antes, ele fica O(n):

```c
size_t n = strlen(s);                     /* strlen uma vez só */
for (size_t i = 0; i < n; i++)
    s[i] = tolower(s[i]);
```

Com uma string de 200 mil caracteres, compilado com `-O2`:

```text
strlen na condição: 0.430 s
strlen antes do for: 0.000 s
```

O compilador não consegue tirar o `strlen` do laço sozinho nesse caso, porque o laço escreve na string, e ele não tem como provar que nenhuma escrita coloca um `'\0'` no meio dela.

## strcpy: copiar uma string

```c
#include <string.h>

char *strcpy(char *dest, const char *src);
```

- **`dest`**: o buffer de destino, onde a cópia vai ser escrita. Precisa ter pelo menos `strlen(src) + 1` bytes.
- **`src`**: a string de origem. É `const` porque só é lida.
- **Retorno:** o próprio `dest`. Isso permite encadear chamadas, mas não traz nenhuma informação nova: o retorno não indica erro.

O `strcpy` copia os caracteres de `src` para `dest`, **incluindo o `'\0'`**, e para logo depois de copiá-lo.

```c
char origem[] = "computação";
char destino[20];

char *r = strcpy(destino, origem);
printf("destino = \"%s\"\n", destino);
printf("r == destino? %s\n", r == destino ? "sim" : "não");
```

```text
destino = "computação"
r == destino? sim
```

Também é por isso que o `strcpy` existe: em C não dá para escrever `destino = origem` com vetores. O nome de um vetor não pode receber uma atribuição, e mesmo com ponteiros, `p = q` só faz `p` apontar para o mesmo lugar, sem copiar nenhum caractere.

### Armadilha: o strcpy não conhece o tamanho do destino

`dest` é só um endereço. O `strcpy` não tem como saber quantos bytes cabem ali e copia até o `'\0'` da origem, aconteça o que acontecer. Se a origem for maior que o destino, a cópia continua por cima do que estiver depois na memória:

```c
struct pessoa {
    char nome[8];
    int idade;
};

int main(void) {
    struct pessoa p;
    p.idade = 20;
    printf("antes:  idade = %d\n", p.idade);

    strcpy(p.nome, "Bartolomeu");   /* 10 letras + '\0' = 11 bytes em 8 */

    printf("depois: nome = \"%s\", idade = %d (0x%x)\n", p.nome, p.idade, p.idade);
    return 0;
}
```

```text
antes:  idade = 20
depois: nome = "Bartolomeu", idade = 30053 (0x7565)
```

O campo `idade` fica logo depois de `nome` na struct. Os bytes que não couberam (`e`, `u` e o `'\0'`) foram escritos em cima dele. Em hexadecimal, `0x65` é `e` e `0x75` é `u`: como o x86-64 é little-endian, o `int` lido foi `0x00007565`. O programa não deu erro nenhum, só passou a ter uma idade errada.

Isso é um **buffer overflow**. Quando o destino é um vetor na pilha e a origem vem do usuário, os bytes extras podem sobrescrever o endereço de retorno da função, que é o ataque explicado no post [Buffer Overflow Attacks](/buffer-overflow-attacks/). A regra para usar `strcpy` com segurança é **verificar o tamanho antes**: `strlen(src) < sizeof dest`.

### Alternativas: strncpy e strdup

```c
char *strncpy(char *dest, const char *src, size_t n);
```

O `strncpy` recebe um terceiro argumento, **`n`**, o número máximo de bytes que ele vai escrever em `dest`. Parece a versão segura do `strcpy`, mas o comportamento tem duas surpresas:

1. Se `src` for **menor** que `n`, ele completa o resto de `dest` com `'\0'` até escrever `n` bytes.
2. Se `src` tiver `n` caracteres ou mais, ele copia `n` caracteres e **não coloca o `'\0'`**. O resultado não é uma string.

```c
char a[6], b[6], c[6];

strncpy(a, "oi", sizeof a);             /* origem curta: completa com '\0' */

memset(b, '#', sizeof b);
strncpy(b, "abacaxi", sizeof b);        /* origem longa: sem '\0' */

strncpy(c, "abacaxi", sizeof c - 1);    /* o jeito seguro */
c[sizeof c - 1] = '\0';
```

Mostrando os 6 bytes de cada vetor:

```text
a: o  i  \0 \0 \0 \0
b: a  b  a  c  a  x
c: a  b  a  c  a  \0
c = "abaca"
```

- Em **`a`**, o `strncpy` escreveu os 4 zeros que faltavam.
- Em **`b`**, os 6 bytes viraram `abacax`, sem terminador. Um `printf("%s", b)` ou um `strlen(b)` continuariam lendo depois do fim do vetor.
- Em **`c`**, está o padrão correto: copiar no máximo `sizeof c - 1` bytes e colocar o `'\0'` na última posição à mão. A string foi cortada em `"abaca"`, mas é uma string válida.

O gcc avisa no caso de `b` com `-Wall`: *"'strncpy' output truncated copying 6 bytes from a string of length 7"*.

```c
char *strdup(const char *s);
```

Quando o tamanho da origem não é conhecido de antemão, o **`strdup`** resolve o problema de outra forma: ele aloca com `malloc` um bloco de `strlen(s) + 1` bytes, copia a string para lá e devolve o ponteiro. O destino sempre tem o tamanho certo. A contrapartida é que o bloco precisa ser liberado com `free` depois. Se não houver memória, ele devolve `NULL`.

## strcat: juntar strings

```c
#include <string.h>

char *strcat(char *dest, const char *src);
```

- **`dest`**: uma string já existente, que vai ser aumentada. Precisa ter espaço para `strlen(dest) + strlen(src) + 1` bytes.
- **`src`**: a string que vai ser acrescentada no fim de `dest`.
- **Retorno:** o próprio `dest`, como no `strcpy`.

O `strcat` (de *concatenate*) trabalha em duas etapas:

1. percorre `dest` até achar o `'\0'`, que marca onde ela termina;
2. a partir dali, copia `src` inteira, incluindo o `'\0'` dela.

O `'\0'` antigo de `dest` é sobrescrito pelo primeiro caractere de `src`, e as duas strings viram uma só.

```c
char caminho[32] = "/home";

strcat(caminho, "/");
strcat(caminho, "lucas");
strcat(strcat(caminho, "/"), "posts");   /* strcat devolve o destino */
printf("caminho = \"%s\" (%zu de %zu bytes)\n",
       caminho, strlen(caminho) + 1, sizeof caminho);
```

```text
caminho = "/home/lucas/posts" (18 de 32 bytes)
```

A última chamada usa o retorno: o `strcat` de dentro devolve `caminho`, que vira o destino do `strcat` de fora.

### Armadilhas do strcat

**O mesmo overflow do `strcpy`.** O `strcat` também não sabe o tamanho de `dest`. A conta de espaço agora tem três parcelas: o que já está lá, o que vai entrar e o `'\0'`.

**O destino precisa ser uma string.** Como a primeira etapa procura o `'\0'`, `dest` precisa ter um. Um vetor declarado sem inicializar (`char buf[32];`) contém o lixo que estava na memória, e o `strcat` pode achar o primeiro zero em qualquer lugar, inclusive depois do fim do vetor. Antes de concatenar, o buffer precisa começar como string vazia: `char buf[32] = "";` ou `buf[0] = '\0';`.

**Concatenar muitas vezes é lento.** Cada `strcat` percorre o destino inteiro desde o começo só para achar o fim. Juntando `n` pedaços um a um, o trabalho de procurar o fim cresce a cada chamada, e o total vira O(n²), como no `strlen` dentro do laço. Quando o laço é grande, vale guardar um ponteiro para o fim atual e copiar a partir dele.

### Alternativa: strncat

```c
char *strncat(char *dest, const char *src, size_t n);
```

O `strncat` copia **no máximo `n` caracteres de `src`** e **sempre** coloca o `'\0'` no fim. A diferença em relação ao `strncpy` está no significado do `n`:

| | O que o `n` limita | Coloca `'\0'`? |
|---|---|---|
| `strncpy(dest, src, n)` | O total de bytes escritos em `dest` | Só se couber |
| `strncat(dest, src, n)` | Quantos caracteres de `src` são copiados | Sempre, num byte a mais |

Ou seja, o `n` do `strncat` **não é o tamanho do destino**. É quanto ainda cabe, descontando o que já está lá e o `'\0'`:

```c
char s[10] = "bom";
const char *resto = " dia para todos";

size_t livre = sizeof s - strlen(s) - 1;   /* espaço que sobra, sem o '\0' */
strncat(s, resto, livre);
printf("livre = %zu, s = \"%s\" (strlen = %zu)\n", livre, s, strlen(s));
```

```text
livre = 6, s = "bom dia p" (strlen = 9)
```

O vetor tem 10 bytes, `"bom"` usa 3, e 1 fica reservado para o `'\0'`. Sobram 6, e foram copiados exatamente 6 caracteres de `" dia para todos"`. O resultado foi cortado, mas cabe no vetor e termina com `'\0'`. Passar `sizeof s` como `n`, um erro comum, permitiria escrever até 3 + 10 + 1 = 14 bytes num vetor de 10.

## strcmp: comparar strings

```c
#include <string.h>

int strcmp(const char *s1, const char *s2);
```

- **`s1`** e **`s2`**: as duas strings a comparar. Nenhuma é modificada.
- **Retorno:** um `int` que indica a ordem entre elas:
  - **negativo**, se `s1` vem antes de `s2`;
  - **zero**, se forem iguais;
  - **positivo**, se `s1` vem depois de `s2`.

### Por que não usar ==

O operador `==` aplicado a strings compara os **endereços**, não os caracteres:

```c
char a[] = "casa";
char b[] = "casa";

printf("a == b          -> %s\n", a == b ? "iguais" : "diferentes");
printf("strcmp(a, b)    -> %d\n", strcmp(a, b));
```

```text
a == b          -> diferentes
strcmp(a, b)    -> 0
```

`a` e `b` são dois vetores em lugares diferentes da memória, com o mesmo conteúdo. O `==` diz que são diferentes porque os endereços são diferentes. O `strcmp` compara os caracteres e devolve 0, ou seja, iguais. O gcc avisa sobre esse erro: *"comparison between two arrays"*.

Por isso o teste de igualdade se escreve `strcmp(a, b) == 0`. Um erro comum é escrever `if (strcmp(a, b))`, achando que significa "se forem iguais": como o retorno de strings iguais é 0, essa condição é verdadeira justamente quando elas são **diferentes**.

### Como a comparação funciona

O `strcmp` anda nas duas strings ao mesmo tempo, posição por posição, e para no primeiro byte diferente (ou quando as duas terminam juntas). O resultado depende só desse primeiro byte diferente, comparado como `unsigned char`. Este programa compara os dois argumentos da linha de comando:

```c
int main(int argc, char *argv[]) {
    if (argc != 3)
        return 1;
    printf("strcmp(\"%s\", \"%s\") = %d\n", argv[1], argv[2], strcmp(argv[1], argv[2]));
    return 0;
}
```

```text
strcmp("abc", "abd") = -1
strcmp("abd", "abc") = 1
strcmp("casa", "casamento") = -109
strcmp("Zebra", "abacate") = -7
strcmp("10", "9") = -8
strcmp("casa", "casa") = 0
```

- **`"abc"` x `"abd"`**: a primeira diferença é `c` (99) contra `d` (100), e `"abc"` vem antes.
- **`"casa"` x `"casamento"`**: as quatro primeiras letras são iguais. Na quinta posição, `"casa"` já tem o `'\0'` (0) e `"casamento"` tem `m` (109). Como 0 é menor que qualquer caractere, **um prefixo sempre vem antes** da string maior. O resultado foi exatamente 0 − 109 = −109.
- **`"Zebra"` x `"abacate"`**: `Z` vem antes de `a`, porque na tabela ASCII **todas as maiúsculas (65–90) vêm antes das minúsculas (97–122)**. A ordem do `strcmp` é a ordem dos códigos, não a ordem alfabética do dicionário.
- **`"10"` x `"9"`**: `1` (49) vem antes de `9` (57), então `"10"` vem antes de `"9"`. O `strcmp` compara texto, não números.

Os retornos −109, −7 e −8 mostram outro ponto: **o valor não é necessariamente −1 ou 1**. O padrão de C só garante o sinal, e a glibc devolve a diferença entre os bytes em vários casos. Testar `strcmp(a, b) == -1` é um erro; o certo é `< 0`.

Esse retorno é exatamente o formato que o `qsort` espera da função de comparação. O post [Vetor de ponteiros](/vetor-de-ponteiros/) mostra o `strcmp` sendo usado para ordenar um vetor de strings.

### strncmp: comparar só o começo

```c
int strncmp(const char *s1, const char *s2, size_t n);
```

O `strncmp` compara **no máximo `n` caracteres**. O uso típico é testar se uma string começa com um prefixo:

```text
strncmp("casa", "casamento", 4) = 0
```

As 4 primeiras letras são iguais, então para o `strncmp` as strings são iguais. Para verificar se `linha` começa com `"PATH="`, basta `strncmp(linha, "PATH=", 5) == 0`.

## strchr: procurar um caractere

```c
#include <string.h>

char *strchr(const char *s, int c);
```

- **`s`**: a string onde procurar.
- **`c`**: o caractere procurado. O tipo é `int` por motivos históricos, mas o valor é convertido para `char` antes da comparação.
- **Retorno:** um **ponteiro para a primeira ocorrência** de `c` em `s`, ou **`NULL`** se o caractere não aparecer.

O retorno é um ponteiro, e não uma posição. Isso permite usar o resultado direto como uma string que começa no caractere encontrado. A posição, se for necessária, sai de uma subtração de ponteiros:

```c
char email[] = "lucas@exemplo.com";

char *arroba = strchr(email, '@');
if (arroba != NULL) {
    printf("achou '%c' no índice %td\n", *arroba, arroba - email);
    printf("domínio: %s\n", arroba + 1);
    *arroba = '\0';                         /* corta a string ali */
    printf("usuário: %s\n", email);
}

printf("strchr(\"abc\", 'z') = %p\n", (void *)strchr("abc", 'z'));

const char *s = "abc";
printf("strchr(s, '\\0') - s = %td\n", strchr(s, '\0') - s);
```

```text
achou '@' no índice 5
domínio: exemplo.com
usuário: lucas
strchr("abc", 'z') = (nil)
strchr(s, '\0') - s = 3
```

- **`arroba - email`** é a distância entre os dois ponteiros, ou seja, o índice do `@`. O resultado de uma subtração de ponteiros tem o tipo `ptrdiff_t`, impresso com `%td`.
- **`arroba + 1`** aponta para o caractere seguinte ao `@`. A partir dali, até o `'\0'`, está o domínio. Nenhuma cópia foi feita: é a mesma memória vista a partir de outro ponto.
- **`*arroba = '\0'`** escreve um terminador no lugar do `@`. Agora `email` termina ali e mostra só o usuário. Partir uma string trocando um caractere por `'\0'` é um truque comum em C, mas só funciona se a string puder ser modificada: `email` é um vetor, não um literal.
- **Caractere ausente**: o `strchr` devolveu `NULL`. **Sempre teste o retorno antes de usar.** Desreferenciar o `NULL` derruba o programa.
- **Procurar o `'\0'`** funciona: o terminador é considerado parte da string, e o `strchr` devolve um ponteiro para o fim. `strchr(s, '\0') - s` dá o mesmo que `strlen(s)`.

### Achar todas as ocorrências e a última

Como o retorno é um ponteiro, dá para continuar a busca do ponto seguinte ao último resultado. O `strrchr` (o `r` vem de *reverse*) faz o oposto: devolve a **última** ocorrência.

```c
const char *frase = "a banana amadureceu";
int n = 0;

for (const char *p = strchr(frase, 'a'); p != NULL; p = strchr(p + 1, 'a')) {
    printf("%td ", p - frase);
    n++;
}
printf("\n%d letras 'a'\n", n);

const char *arquivo = "relatorio.final.pdf";
printf("extensão: %s\n", strrchr(arquivo, '.'));
```

```text
0 3 5 7 9 11
6 letras 'a'
extensão: .pdf
```

O laço começa na primeira ocorrência e, a cada volta, procura a partir de `p + 1`, o caractere seguinte ao último encontrado. Sem o `+ 1`, o `strchr` acharia o mesmo `a` de novo, e o laço nunca terminaria. Para a extensão do arquivo, o `strrchr` pega o último ponto, ignorando o do meio do nome.

### O const que some

Repare na assinatura: o `strchr` recebe um `const char *` e devolve um `char *` sem `const`. Isso existe para que a mesma função sirva para strings modificáveis e somente leitura, mas abre uma brecha. É possível passar um literal e escrever pelo ponteiro devolvido sem nenhum aviso do compilador, e isso causa um segmentation fault, como escrever em qualquer literal. Ao procurar numa string `const`, guarde o resultado num `const char *`, como no exemplo da frase. O C23 corrigiu isso: o `strchr` passou a devolver `const char *` quando recebe um `const char *`.

## Resumo

| Função | Argumentos | Retorno | Cuidado principal |
|---|---|---|---|
| `strlen(s)` | a string | Número de bytes antes do `'\0'` | Não conta o `'\0'`. É O(n), então não deve ficar na condição de um laço |
| `strcpy(dest, src)` | destino e origem | `dest` | Não conhece o tamanho de `dest`: overflow |
| `strcat(dest, src)` | destino (já uma string) e origem | `dest` | Overflow, e `dest` precisa ter `'\0'` |
| `strcmp(s1, s2)` | as duas strings | <0, 0 ou >0 | Compara bytes. Não use `==`, e não espere só −1 ou 1 |
| `strchr(s, c)` | a string e o caractere | Ponteiro para a primeira ocorrência, ou `NULL` | Teste o `NULL`. O retorno perde o `const` |

- **Tudo depende do `'\0'`.** Nenhuma dessas funções sabe o tamanho dos buffers: todas andam na memória até achar o terminador. Uma string sem `'\0'` faz qualquer uma delas ler ou escrever além do fim.
- **Espaço:** uma string de `n` caracteres ocupa `n + 1` bytes. Para `strcpy` e `strcat`, quem chama é que precisa garantir que o destino comporta o resultado.
- **As versões com `n` têm regras diferentes:**
  - o `strncpy` pode deixar o destino sem `'\0'`;
  - o `strncat` sempre termina, mas o `n` é o espaço que sobra, e não o tamanho do destino;
  - o `strncmp` compara só os `n` primeiros caracteres.
- **`strdup`** aloca o destino com o tamanho certo, em troca de um `free` depois.
- **`strchr` e `strrchr`** devolvem ponteiros para dentro da própria string, que podem ser usados como o começo de uma nova string ou subtraídos para dar a posição.

---

*Exemplos testados no Linux, escrito pelo Claude*
