+++
date = '2026-10-27T13:51:50-03:00'
draft = false
title = 'Variáveis de Ambiente'
type = 'blog'
tags = ['Arquitetura', 'Sistemas Operacionais', 'Memória', 'C', 'CSAPP']
+++

Todo processo carrega, além do código e dos dados, uma pequena lista de configurações no formato `NOME=valor`: as **variáveis de ambiente**. Elas dizem ao programa coisas sobre o "ambiente" em que ele está rodando, como quem é o usuário, onde fica a pasta pessoal dele, em que idioma mostrar as mensagens e em quais diretórios procurar outros programas. Alguns exemplos:

| Variável | Exemplo | Para que serve |
|---|---|---|
| `PATH` | `/usr/local/bin:/usr/bin:/bin` | Diretórios onde o shell procura os programas |
| `HOME` | `/home/lucas` | Pasta pessoal do usuário |
| `USER` | `lucas` | Nome do usuário |
| `SHELL` | `/bin/bash` | Shell padrão do usuário |
| `LANG` | `pt_BR.UTF-8` | Idioma e codificação de caracteres |
| `PWD` | `/home/lucas/projetos` | Diretório atual |

No terminal, `env` (ou `printenv`) lista todas elas, e `echo $HOME` mostra o valor de uma só.

Este post parte do que vimos em [Exceptional Control Flow](/exceptional-control-flow/): como as variáveis de ambiente passam de um processo para outro, como um programa em C as enxerga e onde elas ficam na memória.

## O papel das variáveis de ambiente nos processos

**Cada processo tem a sua própria cópia das variáveis de ambiente.** Elas fazem parte do espaço de endereçamento privado do processo, como qualquer outro dado. O que muda é como elas chegam lá: elas são **herdadas** do processo que criou o processo atual.

- **No `fork`**, o filho recebe uma cópia do espaço de endereçamento do pai, e as variáveis de ambiente vêm junto.
- **No `execve`**, o programa antigo é apagado, e o ambiente do programa novo é exatamente o vetor `envp` passado como terceiro argumento. O kernel copia esse vetor para a pilha do programa novo.

Como cada processo tem a sua cópia, **mudar uma variável em um processo não afeta nenhum outro**. Um filho nunca consegue mudar o ambiente do pai. É por isso que um script que faz `export X=1` não muda o `X` do terminal que o chamou: o script roda em um processo filho, e a mudança morre com ele. A herança só vai **de pai para filho**, e só no momento em que o filho é criado.

Isso faz das variáveis de ambiente uma forma simples de **configurar programas sem mudar o código**: quem roda o programa ajusta o ambiente, e o programa lê o que precisa. O shell usa o `PATH` para achar os executáveis, os programas usam o `LANG` para escolher o idioma, um editor de texto pode ser escolhido pelo `EDITOR`, e assim por diante.

### Variável do shell x variável de ambiente

No shell, definir uma variável não basta para que ela vá para os filhos:

```bash
X=1                       # variável do shell: só o shell enxerga
bash -c 'echo "[$X]"'     # o filho não recebe o X
export X                  # agora X faz parte do ambiente
bash -c 'echo "[$X]"'     # o filho recebe o X
```

```
[]
[1]
```

O `bash` guarda as suas variáveis internamente, e só as marcadas com `export` entram no ambiente que ele passa aos filhos. Também dá para definir uma variável **só para um comando**, sem mexer no shell: `LANG=C ls` roda o `ls` com `LANG=C`, e o shell continua com o valor de antes. Já o `env -i` roda um programa com o ambiente **vazio**, o que é útil para testes.

## Como um programa em C enxerga o ambiente

Em C, existem três formas de acessar as variáveis de ambiente.

**1. O terceiro parâmetro da `main`.** Além de `argc` e `argv`, a `main` pode receber `envp`:

```c
int main(int argc, char *argv[], char *envp[]);
```

Isso não faz parte do padrão C, mas funciona nos sistemas Unix.

**2. A variável global `environ`.** Ela é definida pela biblioteca C e aponta para o mesmo vetor. Basta declará-la com `extern`:

```c
#include <stdio.h>

extern char **environ;

int main(void)
{
    for (char **p = environ; *p != NULL; p++)
        printf("%s\n", *p);
    return 0;
}
```

```
$ env -i PATH=/usr/bin HOME=/home/lucas ./ambiente
PATH=/usr/bin
HOME=/home/lucas
```

**3. As funções `getenv`, `setenv` e `unsetenv`.** São a forma mais prática quando se quer uma variável específica:

```c
#include <stdlib.h>

char *getenv(const char *name);
int setenv(const char *name, const char *newvalue, int overwrite);
void unsetenv(const char *name);
```

- **`getenv`** procura `name=valor` e devolve um ponteiro para o `valor`, ou `NULL` se a variável não existir.
- **`setenv`** troca o valor de `name` por `newvalue` (só se `overwrite` for diferente de zero) ou cria a variável se ela não existir.
- **`unsetenv`** apaga a variável.

### O tipo `char **`

Tanto `envp` quanto `environ` têm o tipo `char **`, ou seja, **ponteiro para ponteiro para `char`**. Eles não apontam para as strings, e sim para um **vetor de ponteiros**, em que cada elemento aponta para uma string `NOME=valor`. O vetor termina com um ponteiro `NULL`.

Esse formato tem um motivo. As strings têm tamanhos diferentes, então não dá para achar "a terceira variável" fazendo uma conta. Com um vetor de ponteiros, cada elemento ocupa sempre 8 bytes, e `environ[2]` é só "pegar o ponteiro na posição 2". O `NULL` no final substitui um contador: como não existe um "`envc`", a única forma de saber onde o vetor acaba é parar no `NULL`, exatamente o que o `for` do exemplo acima faz.

## O ambiente na pilha

Quando um programa começa, onde estão essas strings e esses ponteiros? A resposta vem do `execve`. Ele **apaga todo o espaço de endereçamento** do processo para carregar o programa novo, inclusive a memória onde estavam os argumentos e as variáveis de ambiente que ele recebeu. Por isso, antes de apagar, o kernel **copia** as strings dos argumentos e do ambiente, monta vetores novos de ponteiros para elas e coloca tudo no **fundo da pilha** do programa novo. Quando a `main` começa, tudo já está lá.

Para ver isso de verdade, este programa imprime os endereços de tudo o que a `main` recebe:

```c
#include <stdio.h>

extern char **environ;

int main(int argc, char *argv[], char *envp[])
{
    printf("argc = %d\n\n", argc);

    printf("argv    = %p  (endereço de argv[0])\n", (void *)argv);
    for (int i = 0; i <= argc; i++)
        printf("&argv[%d] = %p  contém %p -> %s\n", i, (void *)&argv[i],
               (void *)argv[i], argv[i] ? argv[i] : "(NULL)");

    printf("\nenvp    = %p  (endereço de envp[0])\n", (void *)envp);
    printf("environ = %p\n", (void *)environ);
    for (int i = 0; envp[i] != NULL; i++)
        printf("&envp[%d] = %p  contém %p -> %s\n", i, (void *)&envp[i],
               (void *)envp[i], envp[i]);
    return 0;
}
```

Rodando com `env -i`, para o ambiente ter só duas variáveis:

```
$ env -i PATH=/usr/bin HOME=/home/lucas ./pilha -l
argc = 2

argv    = 0x7ffc6c04eb28  (endereço de argv[0])
&argv[0] = 0x7ffc6c04eb28  contém 0x7ffc6c04ffc6 -> ./pilha
&argv[1] = 0x7ffc6c04eb30  contém 0x7ffc6c04ffce -> -l
&argv[2] = 0x7ffc6c04eb38  contém (nil) -> (NULL)

envp    = 0x7ffc6c04eb40  (endereço de envp[0])
environ = 0x7ffc6c04eb40
&envp[0] = 0x7ffc6c04eb40  contém 0x7ffc6c04ffd1 -> PATH=/usr/bin
&envp[1] = 0x7ffc6c04eb48  contém 0x7ffc6c04ffdf -> HOME=/home/lucas
```

Todos os endereços começam com `0x7ffc6c04`, então o desenho mostra só os quatro últimos dígitos. Por causa do ASLR, os endereços mudam a cada execução, mas a organização é sempre a mesma:

![A pilha do usuário quando a main começa, com os endereços reais do exemplo: no fundo, as strings "HOME=/home/lucas", "PATH=/usr/bin", "-l" e "./pilha"; acima, o vetor envp[] (apontado por envp em %rdx e pela global environ) e o vetor argv[] (apontado por argv em %rsi), cada elemento apontando para a sua string e terminando em NULL; depois, o stack frame de libc_start_main e o futuro stack frame da main; argc fica só no registrador %rdi](01.png)

A pilha cresce para **endereços menores**. O **fundo** é onde ela começa, no endereço mais alto, e o **topo** é onde está o `%rsp`, no endereço mais baixo. O que o kernel coloca primeiro fica no fundo, então, no desenho, o fundo aparece em cima. Do fundo para o topo:

1. **As strings** (em amarelo). São só bytes, cada string terminada em `\0` e todas grudadas: `"./pilha\0"` tem 8 bytes e vai de `…ffc6` a `…ffcd`, e o `"-l"` começa logo em `…ffce`. Primeiro vêm as do ambiente, e depois as dos argumentos.
2. **O vetor `envp[]`** (em verde). Cada elemento ocupa 8 bytes e guarda o **endereço** de uma string `NOME=valor`. Os elementos ficam a 8 bytes um do outro (`…eb40`, `…eb48`, `…eb50`), e o último é `NULL`.
3. **O vetor `argv[]`** (em azul). É igual, mas para os argumentos.
4. **Os stack frames.** Acima de tudo entra o frame da `libc_start_main`, a função da biblioteca C que chama a `main`, e depois o da própria `main`.

A `main` não recebe a pilha inteira, só três valores em registradores, seguindo a convenção do x86-64:

| Parâmetro | Registrador | Valor no exemplo | O que é |
|---|---|---|---|
| `argc` | `%rdi` | `2` | Um número: quantos argumentos existem |
| `argv` | `%rsi` | `0x…eb28` | O endereço do vetor `argv[]` |
| `envp` | `%rdx` | `0x…eb40` | O endereço do vetor `envp[]` |

Para ler `envp[1]`, o programa dá **dois saltos**. Primeiro, vai do registrador para o vetor: `envp + 1` é `…eb40` mais 8 bytes, ou seja, `…eb48`. Depois, lê os 8 bytes guardados ali, `…ffdf`, e vai para a string que está nesse endereço, `"HOME=/home/lucas"`.

Entre o `NULL` do `envp[]` e as strings, o Linux ainda coloca o **vetor auxiliar**, com informações para o dynamic linker e a biblioteca C, como o tamanho de página. É a área tracejada do desenho, e não é preciso entendê-la para usar o ambiente.

O kernel também mostra essa região da pilha em `/proc/<pid>/environ`, com as strings separadas por `\0`:

```
$ cat /proc/self/environ | tr '\0' '\n'
```

Esse arquivo mostra o ambiente **com que o processo começou**, e não as mudanças que ele fez depois com `setenv`, que, como veremos a seguir, nem ficam mais na pilha.

## `environ` x `envp`

No exemplo, `envp` e `environ` têm o mesmo valor (`0x…eb40`), porque os dois apontam para o mesmo vetor na pilha. A diferença está em **quem consegue enxergá-los**:

- **`envp`** é um parâmetro da `main`. Só a `main` tem acesso a ele, a menos que o passe adiante.
- **`environ`** é uma variável global da biblioteca C, que qualquer função do programa pode usar.

E existe uma diferença mais importante: **o `environ` acompanha as mudanças, o `envp` não**. O vetor na pilha tem tamanho fixo. Quando o programa cria uma variável nova com `setenv`, a biblioteca precisa de um vetor maior, então cria um novo no heap e atualiza o `environ`. O `envp` continua apontando para o vetor antigo, na pilha:

```c
#include <stdio.h>
#include <stdlib.h>

extern char **environ;

int main(int argc, char *argv[], char *envp[])
{
    printf("antes:  envp = %p, environ = %p\n", (void *)envp, (void *)environ);

    setenv("NOVA", "1", 1);

    printf("depois: envp = %p, environ = %p\n", (void *)envp, (void *)environ);
    printf("getenv(\"NOVA\") = %s\n", getenv("NOVA"));
    return 0;
}
```

```
$ env -i PATH=/usr/bin HOME=/home/lucas ./muda
antes:  envp = 0x7ffdbd0582d8, environ = 0x7ffdbd0582d8
depois: envp = 0x7ffdbd0582d8, environ = 0x5ecd02009020
getenv("NOVA") = 1
```

Depois do `setenv`, o `environ` passou a apontar para o heap (`0x5ecd…`), e o `envp` ficou para trás, na pilha (`0x7ffd…`), sem a variável `NOVA`. Por isso, fora da `main` e em qualquer programa que mude o ambiente, o certo é usar `environ` ou `getenv`.

## Por que o shell passa `environ` para o `execve`

No shell simples do livro, o filho roda o programa pedido com esta chamada:

```c
if (execve(argv[0], argv, environ) < 0) {
    printf("%s: Command not found.\n", argv[0]);
    exit(0);
}
```

Por que `environ`? Porque **o `execve` não herda o ambiente sozinho**. O terceiro argumento é o ambiente inteiro do programa novo: o kernel copia para a pilha nova exatamente o vetor que recebe, e mais nada. O `fork` até copiou o ambiente do shell para o filho, mas essa cópia está na memória que o `execve` vai apagar. Se o filho não a passar adiante, ela se perde.

Dá para ver isso montando um ambiente na mão:

```c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    char *argv[] = { "/usr/bin/printenv", NULL };
    char *envp[] = { "MSG=oi", "CURSO=computacao", NULL };

    execve(argv[0], argv, envp);
    perror("execve");   /* só chega aqui se o execve falhar */
    return 1;
}
```

```
$ ./executa
MSG=oi
CURSO=computacao
```

O `printenv` imprimiu só as duas variáveis que passamos, sem `PATH`, `HOME` ou qualquer outra que o terminal tinha. **O ambiente do programa novo é só o vetor passado ao `execve`.**

Então, ao passar `environ`, o shell está dizendo: "rode este programa com **o mesmo ambiente que eu tenho**". É assim que um programa rodado pelo shell recebe o `PATH`, o `HOME`, o `LANG` e as variáveis que o usuário configurou. Se o shell passasse um vetor vazio, todo programa rodaria como se tivesse sido chamado com `env -i`: sem saber o idioma, a pasta pessoal ou o tipo do terminal.

E por que `environ`, e não `envp`? Por dois motivos que já vimos:

- **O `eval` não é a `main`.** Ele não recebeu o `envp`, mas pode usar a global `environ`.
- **O `environ` está sempre atualizado.** Se o shell criasse variáveis com `setenv`, o `envp` poderia continuar apontando para o vetor antigo, e o filho receberia um ambiente desatualizado.

Na prática, nem sempre é preciso passar o `environ` à mão. As outras funções da família `exec` que não terminam em `e` (`execv`, `execvp`, `execl`…) usam o `environ` automaticamente, e o `execve` é a versão em que o ambiente é passado explicitamente. Os shells de verdade, como o `bash`, também não passam o `environ` puro: eles montam um vetor com as variáveis marcadas com `export` e com as definidas só para aquele comando (`LANG=C ls`), e passam esse vetor ao `execve`. O shell do livro não tem variáveis próprias, então passar o `environ` é o suficiente.

O caminho completo de uma variável, do terminal até o programa:

1. O shell tem o seu ambiente, acessível por `environ`.
2. O `fork` cria o filho com uma cópia desse ambiente.
3. O filho chama `execve(programa, argv, environ)`.
4. O kernel copia as strings do ambiente e dos argumentos para o fundo da pilha nova e monta os vetores `argv[]` e `envp[]`.
5. A memória antiga é apagada, e o programa novo começa com `envp` (e `environ`) apontando para as cópias na pilha.

## Resumo

- **Variáveis de ambiente** são pares `NOME=valor` que configuram um processo, como `PATH`, `HOME` e `LANG`.
- **Cada processo tem a sua cópia.** Ela é herdada do pai no `fork`, e no `execve` vira exatamente o vetor `envp` recebido. Um filho nunca muda o ambiente do pai.
- **No shell**, só as variáveis com `export` vão para os filhos, `VAR=valor comando` define uma variável só para aquele comando, e `env -i` roda um programa com o ambiente vazio.
- **Em C**, o ambiente é acessado pelo terceiro parâmetro da `main` (`envp`), pela global `environ` ou pelas funções `getenv`, `setenv` e `unsetenv`.
- **`char **`:** `envp` e `environ` apontam para um **vetor de ponteiros** terminado em `NULL`, e cada ponteiro aponta para uma string `NOME=valor`.
- **Na pilha:** o `execve` apaga a memória antiga, então o kernel copia as strings para o fundo da pilha nova e monta, acima delas, os vetores `envp[]` e `argv[]`. A `main` recebe `argc`, `argv` e `envp` em `%rdi`, `%rsi` e `%rdx`.
- **`environ` x `envp`:** começam iguais, mas o `environ` é global e acompanha as mudanças (o `setenv` pode mover o vetor para o heap), enquanto o `envp` fica preso ao vetor original.
- **O shell passa `environ` ao `execve`** porque o `execve` não herda o ambiente sozinho: o programa novo recebe só o vetor passado. Passar `environ` faz o programa rodar com o mesmo ambiente do shell, e atualizado.

---

*Baseado no capítulo Exceptional Control Flow do livro Computer Systems: A Programmer's Perspective (CSAPP), de Randal E. Bryant e David R. O'Hallaron, com exemplos testados no Linux, escrito pelo Claude*
