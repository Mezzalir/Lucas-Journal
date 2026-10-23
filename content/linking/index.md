+++
date = '2026-10-22T22:59:03-03:00'
draft = false
title = 'Linking'
type = 'blog'
tags = ['Arquitetura', 'Sistemas Operacionais', 'Bibliotecas', 'Compiladores']
+++

*Linking* (ligação) é o processo de reunir e combinar vários pedaços de código e de dados em um único arquivo, que pode ser *carregado* (copiado) para a memória e executado. O linking pode acontecer em três momentos:

- **Em tempo de compilação** (*compile time*), quando o código-fonte é traduzido para código de máquina.
- **Em tempo de carregamento** (*load time*), quando o programa é carregado na memória e executado pelo *loader*.
- **Em tempo de execução** (*run time*), feito pelos próprios programas enquanto rodam.

Nos primeiros computadores, o linking era feito manualmente. Nos sistemas modernos, ele é feito automaticamente por programas chamados de **linkers**.

Os linkers têm um papel essencial no desenvolvimento de software porque permitem a **compilação separada**. Em vez de organizar uma aplicação grande em um único arquivo-fonte gigante, é possível dividi-la em módulos menores e mais fáceis de gerenciar, que podem ser modificados e compilados separadamente. **Quando um desses módulos muda, basta recompilar só ele e refazer o linking da aplicação, sem precisar recompilar todos os outros arquivos.**

## O programa de exemplo

Ao longo do post, vamos usar como exemplo um programa formado por dois arquivos-fonte, `main.c` e `sum.c`. A função `main` inicializa um vetor de `int` e depois chama a função `sum` para somar os elementos do vetor.

**main.c**

```c
int sum(int *a, int n);

int array[2] = {1, 2};

int main()
{
    int val = sum(array, 2);
    return val;
}
```

**sum.c**

```c
int sum(int *a, int n)
{
    int i, s = 0;

    for (i = 0; i < n; i++) {
        s += a[i];
    }
    return s;
}
```

Repare que `main.c` só declara a função `sum` (na primeira linha), sem defini-la. A definição está em `sum.c`. Cada arquivo pode ser compilado separadamente, e é o linker que junta os dois em um único programa executável.

## Compiler drivers

A maioria dos sistemas de compilação tem um **compiler driver**, um programa que chama o pré-processador, o compilador, o montador (*assembler*) e o linker conforme necessário. No Linux, o driver é o próprio `gcc`. Para gerar o executável do nosso exemplo, basta um comando:

```
linux> gcc -Og -o prog main.c sum.c
```

Por trás dele, o driver passa cada arquivo-fonte pelas mesmas etapas, até chegar no executável. Para `main.c`, elas são:

| Etapa                 | Programa | Entrada → Saída       | O que faz                                          |
| --------------------- | -------- | --------------------- | -------------------------------------------------- |
| 1. Pré-processamento  | `cpp`    | `main.c` → `main.i`   | Expande `#include` e `#define`, remove comentários |
| 2. Compilação         | `cc1`    | `main.i` → `main.s`   | Traduz C para Assembly (em texto)                  |
| 3. Montagem           | `as`     | `main.s` → `main.o`   | Traduz o Assembly para código de máquina (binário) |
| 4. Linkagem           | `ld`     | `main.o` + `sum.o` → `prog` | Junta os objetos e resolve os endereços      |

As três primeiras etapas são feitas separadamente para cada arquivo, e o mesmo processo gera o `sum.o`. Os arquivos `.o` são chamados de **relocatable object files**: já são código de máquina, mas ainda não podem ser executados sozinhos. Na última etapa, o linker `ld` combina `main.o` e `sum.o`, junto com alguns object files do sistema, e cria o **executable object file** `prog`:

```
ld -o prog [object files do sistema e argumentos] /tmp/main.o /tmp/sum.o
```

Para rodar o programa, basta digitar o nome dele no terminal:

```
linux> ./prog
```

O shell chama uma função do sistema operacional chamada **loader** (carregador), que copia o código e os dados do arquivo `prog` para a memória e transfere o controle para o início do programa.

## Static linking

Um **linker estático**, como o `ld` do Linux, pega vários relocatable object files e junta tudo em um único executable object file, pronto para ser carregado e rodar. Cada object file é dividido em seções: uma com as instruções, outra com as variáveis globais inicializadas, outra com as não inicializadas, e assim por diante.

Para fazer isso, o linker tem duas tarefas:

1. **Resolução de símbolos:** símbolos são os nomes de funções, variáveis globais e variáveis `static`. Um arquivo pode *definir* um símbolo ou só *referenciá-lo*. O linker liga cada referência a exatamente uma definição. No exemplo, `main.c` referencia `sum`, e o linker encontra a definição em `sum.c`. Variáveis locais não entram nessa conta, a não ser que sejam declaradas como `static`: as comuns ficam na pilha, e o linker nem sabe que elas existem.
2. **Relocação:** o compilador e o montador geram cada arquivo como se ele começasse no endereço 0, porque ainda não sabem onde ele vai ficar. O linker decide o endereço final de cada símbolo e corrige todas as referências para apontarem para lá. Ele faz isso seguindo instruções deixadas pelo montador, as *entradas de relocação*.

O ponto importante é que **o linker é bem simples**: para ele, um object file é só um conjunto de blocos de bytes (código, dados e informações para o linker e o loader). Ele junta os blocos, decide onde cada um fica na memória e ajusta os endereços. Quase todo o trabalho pesado já foi feito antes, pelo compilador e pelo montador.

## Object files

Existem três tipos de object file:

- **Relocatable object file:** código e dados binários que ainda vão ser combinados com outros relocatable object files para formar um executable object file. É o que o compilador e o montador geram (os `.o` do exemplo).
- **Executable object file:** código e dados prontos para serem copiados para a memória e executados. É o que o linker gera (o `prog`).
- **Shared object file:** um tipo especial de relocatable object file que pode ser carregado na memória e ligado dinamicamente, quando o programa é carregado ou durante a execução. São as bibliotecas compartilhadas, como os `.so` do Linux e as `.dll` do Windows.

O formato desses arquivos muda de sistema para sistema. Os primeiros Unix usavam o formato `a.out` (por isso até hoje o executável gerado pelo `gcc` sem `-o` se chama `a.out`). O Windows usa o **PE** (*Portable Executable*), o macOS usa o **Mach-O**, e o Linux e os Unix modernos usam o **ELF** (*Executable and Linkable Format*). O post segue o ELF, mas as ideias são as mesmas nos outros formatos.

### Relocatable object files no formato ELF

Um relocatable object file ELF começa com um **cabeçalho ELF**, termina com uma **tabela de cabeçalhos de seção**, e tem as seções no meio:

```
+-------------------------------+  0
| Cabeçalho ELF                 |
+-------------------------------+
| .text                         |
| .rodata                       |
| .data                         |
| .bss                          |
| .symtab                       |  Seções
| .rel.text                     |
| .rel.data                     |
| .debug                        |
| .line                         |
| .strtab                       |
+-------------------------------+
| Tabela de cabeçalhos de seção |  Descreve as seções
+-------------------------------+
```

O **cabeçalho ELF** diz o tamanho da palavra e a ordem dos bytes da máquina que gerou o arquivo, o tipo do arquivo (relocatable, executable ou shared object file), a arquitetura (x86-64, por exemplo) e onde fica a tabela de cabeçalhos de seção. Essa tabela, por sua vez, diz onde começa e qual o tamanho de cada seção.

As seções guardam o conteúdo do arquivo:

| Seção       | O que guarda                                                                 | No exemplo (`main.o`)            |
| ----------- | ---------------------------------------------------------------------------- | -------------------------------- |
| `.text`     | O código de máquina do programa                                              | O código da função `main`        |
| `.rodata`   | Dados somente leitura, como as strings de formato do `printf`                | (vazia)                          |
| `.data`     | Variáveis globais e `static` **inicializadas**                               | O vetor `array`, com `{1, 2}`    |
| `.bss`      | Variáveis globais e `static` **não inicializadas** (ou inicializadas com zero) | (vazia)                        |
| `.symtab`   | A tabela de símbolos: funções e variáveis globais definidas e referenciadas  | `main` e `array` (definidos), `sum` (referenciado) |
| `.rel.text` | Os pontos do `.text` que o linker vai ter que corrigir                       | A chamada a `sum` e o uso do endereço de `array` |
| `.rel.data` | Os pontos do `.data` que o linker vai ter que corrigir                       | (vazia)                          |
| `.debug`    | Informações para o depurador (só com `-g`)                                   |                                  |
| `.line`     | A relação entre as linhas do código C e as instruções (só com `-g`)          |                                  |
| `.strtab`   | Os nomes dos símbolos e das seções, como strings terminadas em nulo          | "main", "array", "sum"...        |

Alguns detalhes que valem destacar:

- **Variáveis locais não aparecem em nenhuma dessas seções.** Elas são criadas na pilha durante a execução, então o linker não precisa saber delas. A variável `val` de `main`, por exemplo, não está em `.data` nem na tabela de símbolos.
- **A seção `.bss` não ocupa espaço no arquivo.** Como as variáveis dela começam todas com zero, não faz sentido guardar esses zeros no disco: o arquivo só registra quanto espaço elas precisam, e a memória é zerada quando o programa é carregado. O nome vem de uma diretiva do montador do IBM 704, de 1957 (*block started by symbol*), mas o livro sugere lembrar como *Better Save Space*.
- **Todo relocatable object file tem tabela de símbolos**, mesmo compilado sem `-g`. O `-g` só acrescenta as informações de depuração (`.debug` e `.line`).

Dá para ver as seções de um object file com o comando `readelf -S main.o`, a tabela de símbolos com `readelf -s main.o` e as entradas de relocação com `readelf -r main.o`. Compilando o exemplo no x86-64, a seção de relocação aparece como `.rela.text` em vez de `.rel.text`: é uma variante do mesmo formato, usada no x86-64, que guarda um valor extra em cada entrada. E ela tem duas entradas: uma para a chamada a `sum` e outra para o endereço de `array`, já que o `main.o` ainda não sabe onde nenhum dos dois vai ficar na memória.

## Símbolos e tabelas de símbolos

Cada relocatable object file tem uma tabela de símbolos com tudo o que ele define e tudo o que ele usa de fora. Para o linker, existem três tipos de símbolo:

- **Globais definidos pelo módulo:** funções e variáveis globais que o arquivo define e que outros arquivos podem usar. No `main.o`, são `main` e `array`.
- **Externos:** funções e variáveis globais que o arquivo usa, mas que são definidas em outro arquivo. No `main.o`, é o `sum`.
- **Locais:** funções e variáveis globais declaradas com `static`. Elas só podem ser usadas dentro do próprio arquivo, e nenhum outro módulo consegue enxergá-las.

### `static` como "privado"

Em C, o `static` em uma variável global ou função faz o papel do `private` em Java ou C++: o arquivo é o módulo, e tudo o que é `static` fica escondido dentro dele. O que não é `static` é público e pode ser usado por qualquer outro arquivo. Por isso, é uma boa prática declarar como `static` tudo o que não precisa ser visto de fora.

### Símbolos locais não são variáveis locais

Um ponto que confunde: **símbolo local do linker não é o mesmo que variável local do programa**. As variáveis locais comuns (como `val`, `i` e `s` do exemplo) ficam na pilha durante a execução e não aparecem na tabela de símbolos. O linker nem sabe que elas existem.

A exceção são as variáveis locais declaradas com `static`. Como precisam manter o valor entre uma chamada e outra, elas não ficam na pilha: o compilador reserva espaço para elas em `.data` ou `.bss` e cria um símbolo local para cada uma. Se duas funções do mesmo arquivo tiverem uma variável `static` com o mesmo nome, o compilador dá nomes diferentes para cada uma:

```c
int f()
{
    static int x = 0;
    return x;
}

int g()
{
    static int x = 1;
    return x;
}
```

Aqui, o compilador cria dois símbolos locais, como `x.1` e `x.2`, um para cada `x`. Compilando no GCC atual, eles aparecem como `x.0` e `x.1`, mas a ideia é a mesma. O `x` de `f`, inicializado com 0, vai para `.bss`; o de `g`, inicializado com 1, vai para `.data`.

### Como é uma entrada da tabela de símbolos

A tabela de símbolos fica na seção `.symtab` e é um vetor de entradas com este formato:

```c
typedef struct {
    int   name;        /* Posição do nome na tabela de strings */
    char  type:4,      /* Função ou dado (4 bits) */
          binding:4;   /* Local ou global (4 bits) */
    char  reserved;    /* Não usado */
    short section;     /* Índice da seção */
    long  value;       /* Posição na seção ou endereço absoluto */
    long  size;        /* Tamanho do objeto em bytes */
} Elf64_Symbol;
```

- **`name`:** o nome não fica na entrada, e sim na seção `.strtab`. A entrada só guarda a posição dele lá.
- **`value`:** o endereço do símbolo. Em um relocatable object file, é a distância desde o início da seção onde o símbolo está, já que o endereço final ainda não existe. Em um executable object file, é o endereço real durante a execução.
- **`size`:** o tamanho em bytes.
- **`type`:** se o símbolo é uma função ou um dado.
- **`binding`:** se o símbolo é local ou global.
- **`section`:** em qual seção o símbolo está (`.text`, `.data`...).

Além das seções normais, existem três **pseudosseções**, que só aparecem em relocatable object files:

- **ABS:** símbolos que não devem ser relocados.
- **UNDEF:** símbolos usados no arquivo, mas definidos em outro lugar (os externos).
- **COMMON:** variáveis globais não inicializadas que ainda não receberam espaço.

A diferença entre COMMON e `.bss` é sutil. O livro descreve esta convenção do GCC:

| Pseudosseção/seção | O que vai para ela                                                        |
| ------------------ | ------------------------------------------------------------------------- |
| COMMON             | Variáveis globais não inicializadas                                       |
| `.bss`             | Variáveis `static` não inicializadas, e globais ou `static` inicializadas com zero |

O motivo tem a ver com a forma como o linker resolve símbolos com o mesmo nome em arquivos diferentes, que é o assunto da próxima seção. Mas vale notar que **isso mudou desde o livro**: a partir do GCC 10, o padrão passou a ser a opção `-fno-common`, e as variáveis globais não inicializadas vão direto para `.bss`. Para ter o comportamento antigo, é preciso compilar com `-fcommon`.

### Vendo a tabela de símbolos do exemplo

O programa `readelf` mostra o conteúdo de object files. As últimas entradas da tabela de símbolos do `main.o` são:

```
linux> readelf -s main.o
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     3: 0000000000000000    26 FUNC    GLOBAL DEFAULT    1 main
     4: 0000000000000000     8 OBJECT  GLOBAL DEFAULT    3 array
     5: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND sum
```

- **`main`:** uma função global de 26 bytes, no início (`Value` 0) da seção 1, que é a `.text`. No livro ela tem 24 bytes; o tamanho muda conforme a versão do compilador.
- **`array`:** um dado global de 8 bytes (dois `int`), no início da seção 3, que é a `.data`.
- **`sum`:** está como `UND` (UNDEF), porque o `main.o` usa `sum`, mas não a define. É o linker que vai encontrar a definição em `sum.o`.

---

*Resumo do capítulo Linking do livro Computer Systems: A Programmer's Perspective (CSAPP), de Randal E. Bryant e David R. O'Hallaron, compilado pelo Claude*
