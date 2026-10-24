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

Existem duas formas de fazer o linking:

- **Linking estático:** acontece na compilação. As bibliotecas são arquivos `.a`, uma coleção de relocatable object files (`.o`) empacotados em um só arquivo, e o linker copia para dentro do executável só os `.o` que o programa usa. O executável fica maior, mas é autossuficiente e não depende de nada para rodar.
- **Linking dinâmico:** o código das bibliotecas fica de fora do executável, em uma *shared library* (`.so` no Linux, `.dll` no Windows), e só é ligado ao programa quando ele é carregado ou enquanto roda. Vários programas podem usar a mesma cópia da biblioteca na memória, e atualizar a biblioteca não exige refazer o linking dos programas.

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

## Símbolos e tabelas de símbolos

Um **símbolo** é um nome que aparece no nível global do código C: uma função ou uma variável global (ou `static`). No exemplo, `main`, `array` e `sum` são símbolos. Já as variáveis declaradas dentro das funções, como `val`, `i` e `s`, não são.

A **tabela de símbolos** é uma lista guardada dentro do próprio `.o`, na seção `.symtab`, que diz para cada nome:

- onde ele está (em qual seção e em qual posição dela);
- quantos bytes ocupa;
- o que ele é (função, variável...);
- se outros arquivos podem usá-lo.

Ela existe para o **linker**. Quando você junta vários `.o`, o linker usa essas tabelas para descobrir, por exemplo, que o `sum` chamado em `main.o` é o `sum` definido em `sum.o`.

Para entender os tipos de símbolo, é preciso saber o que é um **módulo**: é simplesmente um arquivo do programa. Cada `.c`, depois de compilado, vira um `.o`, e cada `.o` é um módulo (o livro chama de *object module*). No exemplo, o programa tem dois módulos: `main.o` e `sum.o`. Então, quando o texto diz "definido pelo módulo", quer dizer "definido **naquele arquivo**".

Para o linker, existem três tipos de símbolo, sempre olhando do ponto de vista de um módulo:

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

### Resolução de símbolos

O linker liga cada referência a exatamente uma definição, procurando nas tabelas de símbolos dos object files. Para **símbolos locais** é fácil, porque a definição está no próprio módulo. Para **símbolos globais**, o compilador só assume que a definição está em outro módulo e deixa o problema para o linker. Se o linker não a encontra em nenhum arquivo, ele dá erro, como o `undefined reference to 'foo'` de uma função que foi declarada mas nunca definida.

O outro problema é quando vários módulos definem um símbolo global com o mesmo nome.

#### Símbolos com o mesmo nome: fortes e fracos

Cada símbolo global é **forte** (funções e variáveis globais inicializadas) ou **fraco** (variáveis globais não inicializadas). O linker segue três regras:

1. **Vários fortes com o mesmo nome:** erro (`multiple definition`).
2. **Um forte e vários fracos:** escolhe o forte.
3. **Vários fracos:** escolhe qualquer um.

As regras 2 e 3 são perigosas porque o linker escolhe **sem avisar**:

```c
/* foo3.c */
#include <stdio.h>
void f(void);

int x = 15213;

int main()
{
    f();
    printf("x = %d\n", x);
    return 0;
}
```

```c
/* bar3.c */
int x;

void f()
{
    x = 15212;
}
```

O `x` forte de `foo3.c` e o fraco de `bar3.c` viram a mesma variável, então `f` altera o `x` do `main` e o programa imprime 15212 em vez de 15213. Com tipos diferentes é pior: se um arquivo define `x` como `int` e o outro como `double`, a escrita de 8 bytes do `double` sobrescreve também a variável vizinha na memória. Para se proteger, o livro recomenda compilar com `-fno-common` ou `-Werror`.

É por isso que variáveis globais não inicializadas iam para COMMON: o compilador não sabe se outro módulo define a mesma variável e deixa a decisão para o linker. **Desde o GCC 10, o padrão é `-fno-common`**, então hoje esse exemplo dá `multiple definition of 'x'` em vez de escolher em silêncio. O comportamento do livro só aparece com `-fcommon`.

## Bibliotecas estáticas

Até aqui, o linker recebia só os object files do próprio programa. Na prática, todo sistema também permite empacotar vários object files relacionados em um único arquivo, a **biblioteca estática** (*static library*), que também é passada para o linker. A vantagem é que o linker **copia para o executável só os módulos da biblioteca que o programa realmente usa**.

Um bom exemplo é a biblioteca padrão do C, a `libc.a`, que tem funções como `printf`, `scanf`, `strcpy` e `atoi`, e a `libm.a`, com funções matemáticas como `sin`, `cos` e `sqrt`. Sem bibliotecas estáticas, as alternativas seriam ruins:

- **O compilador reconhecer as funções padrão e gerar o código delas direto**, como o Pascal faz. No C, que tem muitas funções padrão, isso deixaria o compilador muito complicado, e cada mudança em uma função exigiria uma nova versão do compilador.
- **Colocar todas as funções em um único object file**, como um `libc.o`. Aí todo executável carregaria uma cópia da biblioteca inteira, desperdiçando disco e memória (a `libc.a` tem cerca de 5 MB), e qualquer mudança em uma função obrigaria a recompilar o arquivo todo.
- **Deixar cada função em um object file separado** e o programador passar ao linker cada um que usa, como `printf.o` e `scanf.o`. Funciona, mas é trabalhoso e fácil de errar.

**A biblioteca estática resolve tudo isso: cada função fica em um módulo separado, mas todos ficam juntos em um só arquivo.** No Linux, esse arquivo tem o formato **archive** (extensão `.a`): um conjunto de relocatable object files concatenados, com um cabeçalho que descreve onde está cada um. Basta passar a biblioteca para o linker:

```
linux> gcc main.c /usr/lib/libm.a /usr/lib/libc.a
```

(Na verdade, o `gcc` já passa a `libc.a` sozinho, então ela nem precisa aparecer no comando.)

### Exemplo: a biblioteca `libvector`

O livro cria uma biblioteca com duas funções, `addvec` (soma dois vetores) e `multvec` (multiplica dois vetores), cada uma no seu arquivo. Cada uma também conta quantas vezes foi chamada, em uma variável global:

```c
/* addvec.c */
int addcnt = 0;

void addvec(int *x, int *y,
            int *z, int n)
{
    int i;

    addcnt++;

    for (i = 0; i < n; i++)
        z[i] = x[i] + y[i];
}
```

O `multvec.c` é igual, só que com `multcnt` e `z[i] = x[i] * y[i]`. Para criar a biblioteca, compilamos os dois arquivos e juntamos os object files com o programa `ar`:

```
linux> gcc -c addvec.c multvec.c
linux> ar rcs libvector.a addvec.o multvec.o
```

Um programa que usa a biblioteca:

```c
/* main2.c */
#include <stdio.h>
#include "vector.h"

int x[2] = {1, 2};
int y[2] = {3, 4};
int z[2];

int main()
{
    addvec(x, y, z, 2);
    printf("z = [%d %d]\n", z[0], z[1]);
    return 0;
}
```

O arquivo `vector.h` só tem as declarações das funções da `libvector`. Para gerar o executável:

```
linux> gcc -c main2.c
linux> gcc -static -o prog2c main2.o ./libvector.a
```

ou, de forma equivalente:

```
linux> gcc -static -o prog2c main2.o -L. -lvector
```

- **`-static`:** diz para gerar um executável totalmente ligado, que não precisa de mais nenhum linking quando for carregado.
- **`-lvector`:** é um atalho para `libvector.a`.
- **`-L.`:** diz para o linker procurar a biblioteca no diretório atual.

Quando o linker roda, ele percebe que `main2.o` usa `addvec`, que está definida em `addvec.o`, e copia `addvec.o` para o executável. Como o programa nunca usa `multvec`, o `multvec.o` **não** é copiado. O mesmo acontece com a `libc.a`: o linker copia o `printf.o` e os outros módulos de que o `printf` precisa, e mais nada. O executável `prog2c` fica com:

- `main2.o`, o código do próprio programa;
- `addvec.o`, vindo da `libvector.a`;
- `printf.o` e os módulos que ele usa, vindos da `libc.a`.

![Linking com bibliotecas estáticas: main2.o, addvec.o (da libvector.a) e printf.o (da libc.a) entram no linker, que gera o executável prog2c (CS:APP, Figura 7.8)](01.png)

### Resumindo: o que é uma biblioteca estática

Uma **biblioteca estática** é um arquivo `.a` que junta vários relocatable object files (`.o`) em um só, junto com um índice que diz qual `.o` define cada função. O `s` do `ar rcs` é o que cria esse índice.

Na hora de gerar o executável, o linker (`ld`, chamado pelo `gcc`) faz o seguinte:

1. Olha o programa e anota as funções que ele usa mas não define, como `addvec`.
2. Procura essas funções no índice da biblioteca.
3. **Copia para o executável só os `.o` necessários, e inteiros.** Se o programa usa uma função de `addvec.o`, vem o `addvec.o` todo. Se nenhuma função de `multvec.o` é usada, ele fica de fora.

Disso saem três consequências:

- **Depois do linking, o executável não precisa mais do `.a`**, porque o código foi copiado para dentro dele.
- **A unidade de cópia é o `.o`, não a função.** Por isso bibliotecas como a `libc` costumam ter uma função por arquivo `.c`.
- **A ordem no comando importa.** O linker lê da esquerda para a direita e só pega da biblioteca o que já sabe que está faltando, então a biblioteca precisa vir **depois** de quem a usa. A próxima seção explica isso em detalhe.

### Como o linker usa as bibliotecas estáticas

As bibliotecas estáticas são úteis, mas também confundem muita gente por causa do jeito como o linker do Linux as usa. O linker lê os arquivos **da esquerda para a direita, na ordem em que aparecem no comando**, e vai mantendo três conjuntos:

- **E:** os object files que vão entrar no executável.
- **U:** os símbolos **não resolvidos**, que já foram usados mas ainda não têm definição.
- **D:** os símbolos que já foram definidos pelos arquivos lidos até agora.

No começo, os três estão vazios. Para cada arquivo do comando:

- **Se for um object file**, o linker o coloca em E e atualiza U e D com o que ele define e usa.
- **Se for uma biblioteca**, o linker procura dentro dela módulos que definam algum símbolo de U. Cada módulo que resolve alguma coisa entra em E, e U e D são atualizados. Ele repete isso até não encontrar mais nada útil na biblioteca. **Os módulos que não resolveram nada são descartados.**

No fim, se U não estiver vazio, sobrou algum símbolo sem definição, e o linker para com um erro. Senão, ele junta e reloca os object files de E e gera o executável.

O problema é que, por isso, **a ordem dos arquivos no comando importa**. Se a biblioteca vier antes do arquivo que usa as funções dela, o linker falha:

```
linux> gcc -static ./libvector.a main2.c
/tmp/cc9XH6Rp.o: In function 'main':
/tmp/cc9XH6Rp.o(.text+0x18): undefined reference to 'addvec'
```

Quando o linker leu a `libvector.a`, o U ainda estava vazio, porque o `main2.c` não tinha sido lido. Então nenhum módulo da biblioteca parecia útil e todos foram descartados. Quando o `main2.o` apareceu pedindo `addvec`, a biblioteca já tinha ficado para trás.

A regra geral é **colocar as bibliotecas no fim do comando**. Se uma biblioteca depende de outra, a ordem entre elas também importa: quem usa vem antes de quem define. Se `foo.c` chama funções de `libx.a` e de `libz.a`, e essas chamam funções de `liby.a`, então `libx.a` e `libz.a` têm que vir antes de `liby.a`:

```
linux> gcc foo.c libx.a libz.a liby.a
```

Se a dependência for circular, a biblioteca pode ser repetida no comando. Se `foo.c` chama `libx.a`, que chama `liby.a`, que chama `libx.a` de novo:

```
linux> gcc foo.c libx.a liby.a libx.a
```

Outra saída é juntar `libx.a` e `liby.a` em uma única biblioteca.

## Relocação

Terminada a resolução de símbolos, cada referência já está ligada a uma única definição. Na relocação, o linker junta as seções do mesmo tipo de todos os módulos (todas as `.text` viram uma só, todas as `.data` também), decide o endereço de execução de cada seção e de cada símbolo e, por fim, corrige as referências no código para apontarem para esses endereços.

Para saber o que corrigir, o linker usa as **entradas de relocação** (`.rela.text` e `.rela.data`). O montador cria uma entrada sempre que encontra uma referência a algo cujo endereço final ele ainda não conhece. Cada entrada diz **onde** está a referência dentro da seção (*offset*), **a qual símbolo** ela se refere, **como** corrigir (*type*) e uma constante que entra no cálculo (*addend*). Os dois tipos básicos são:

- **`R_X86_64_PC32`:** endereço **relativo ao PC**, ou seja, a distância entre o símbolo e a instrução. É o usado pelo `call`.
- **`R_X86_64_32`:** endereço **absoluto** de 32 bits, usado diretamente.

### Relocando referências a símbolos

O linker percorre cada seção e cada entrada de relocação dela. Para cada entrada, ele acha o ponto da seção que precisa ser corrigido e escreve ali o valor certo, de acordo com o tipo:

- **Referência relativa ao PC:** o linker escreve a **distância** entre a instrução e o símbolo. Na execução, o processador soma essa distância ao PC e chega ao símbolo. O addend serve para compensar o fato de que, quando a instrução executa, o PC já aponta para a instrução seguinte.
- **Referência absoluta:** o linker escreve o **endereço** do símbolo direto, e a instrução o usa sem nenhum cálculo.

## Executable object files

Depois do linking, o programa, que começou como alguns arquivos de texto em C, virou um único arquivo binário com tudo o que é preciso para ser carregado na memória e rodar.

![Formato de um executable object file ELF típico: cabeçalho ELF, tabela de cabeçalhos de segmento, as seções .init, .text, .rodata, .data, .bss, .symtab, .debug, .line, .strtab e a tabela de cabeçalhos de seção (CS:APP, Figura 7.13)](02.png)

**O formato de um executable object file é parecido com o de um relocatable object file.** O cabeçalho ELF descreve o formato geral do arquivo e também guarda o **entry point**, o endereço da primeira instrução que vai ser executada quando o programa rodar. As diferenças em relação ao relocatable object file são:

- **As seções `.text`, `.rodata` e `.data` já foram relocadas** para os endereços que vão ter na memória durante a execução.
- **Existe uma seção `.init`**, com uma pequena função `_init`, chamada pelo código de inicialização do programa.
- **Não existem seções `.rel`**, porque o executável já está totalmente ligado e não há mais nada para corrigir.

O executável foi feito para ser fácil de carregar: pedaços contíguos do arquivo correspondem a pedaços contíguos da memória, os **segmentos**. Quem descreve essa correspondência é a **tabela de cabeçalhos de programa** (*program header table*), que dá para ver com `objdump -p`. No exemplo, ela tem dois segmentos:

- **Segmento de código** (leitura e execução): contém o cabeçalho ELF, a tabela de cabeçalhos de programa e as seções `.init`, `.text` e `.rodata`.
- **Segmento de dados** (leitura e escrita): contém a seção `.data` e a `.bss`. A `.bss` não ocupa espaço no arquivo, então a parte do segmento que corresponde a ela é preenchida com zeros quando o programa é carregado.

As seções `.symtab`, `.debug`, `.line` e `.strtab` ficam no arquivo, mas **não são carregadas na memória**.

Para cada segmento, o linker escolhe um endereço inicial que fique **alinhado** com a posição do segmento dentro do arquivo: os dois precisam ter o mesmo resto na divisão pelo alinhamento indicado no cabeçalho, que no exemplo é 2 MB. Isso é uma otimização que permite copiar os segmentos do arquivo para a memória de forma eficiente, por causa do jeito como a memória virtual é organizada, em blocos grandes e contíguos com tamanho potência de 2 (assunto do capítulo 9).

## Resumo

- **Object file:** é o arquivo que o compilador gera a partir de um `.c`. Ele já tem o código em linguagem de máquina, mas ainda não é um programa pronto. Cada object file também é chamado de **módulo**: no exemplo deste post, `main.o` e `sum.o` são dois módulos.
- **Tipos de object file:**
  - **Relocatable object file** (`.o`): gerado pelo compilador, ainda precisa ser juntado com outros para virar um programa.
  - **Executable object file**: o programa pronto, gerado pelo linker, que pode ser carregado na memória e executado.
  - **Shared object file** (`.so` no Linux, `.dll` no Windows): uma biblioteca que é ligada ao programa só na hora de carregar ou durante a execução.
- **O que tem dentro de um object file:** o código de máquina (`.text`), as variáveis globais inicializadas (`.data`) e não inicializadas (`.bss`), a **tabela de símbolos** (`.symtab`) e as informações de relocação (`.rel.text`, `.rel.data`), que dizem ao linker o que corrigir.
- **Carregar x executar:** **carregar** é copiar o código e os dados do executable object file do disco para a memória, preparando tudo para o programa começar. Quem faz isso é o **loader**, uma parte do sistema operacional. **Executar** é o que vem depois: o processador começa a rodar as instruções que já estão na memória, a partir da primeira instrução do programa. Ou seja, carregar é colocar o programa na memória, e executar é rodar o programa que já está lá.
- **Símbolo:** o nome de uma função ou variável global (ou `static`). Variáveis locais comuns não são símbolos. Do ponto de vista de um módulo, existem três tipos:
  - **Globais:** definidos no módulo e visíveis para os outros (`main` e `array` em `main.o`).
  - **Externos:** usados no módulo, mas definidos em outro (`sum` em `main.o`).
  - **Locais:** declarados com `static`, visíveis só dentro do próprio módulo.
  - Os globais também podem ser **fortes** (funções e variáveis inicializadas) ou **fracos** (variáveis não inicializadas), o que decide qual definição o linker escolhe quando há nomes repetidos.
- **O que o linker faz:** junta os módulos em um executável em duas etapas. Na **resolução de símbolos**, liga cada nome usado à sua definição, que pode estar em outro módulo. Na **relocação**, decide o endereço final de cada coisa e corrige todas as referências.
- **Biblioteca estática** (`.a`): um pacote com vários relocatable object files, como a `libc.a`. O linker copia para o executável só os módulos que o programa usa. Como o linker lê os arquivos da esquerda para a direita, **as bibliotecas devem vir no fim do comando**, depois dos arquivos que usam as funções delas.
- **Relocação:** para cada referência cujo endereço não conhece, o montador deixa uma **entrada de relocação**. Depois de decidir os endereços, o linker usa essas entradas para corrigir o código: referências **relativas ao PC** (como o `call`) recebem a distância até o símbolo, e referências **absolutas** recebem o endereço dele.
- **Executable object file:** tem formato parecido com o do relocatable object file, mas já está relocado, não tem seções `.rel` e o cabeçalho ELF guarda o **entry point**, o endereço da primeira instrução. A **tabela de cabeçalhos de programa** diz quais partes do arquivo viram os segmentos de código e de dados na memória.

---

*Resumo do capítulo Linking do livro Computer Systems: A Programmer's Perspective (CSAPP), de Randal E. Bryant e David R. O'Hallaron, compilado pelo Claude*
