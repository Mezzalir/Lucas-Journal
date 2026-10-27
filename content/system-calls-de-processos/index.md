+++
date = '2026-10-27T15:56:29-03:00'
draft = false
title = 'System calls de Processos no Linux'
type = 'blog'
tags = ['Sistemas Operacionais', 'Linux', 'C']
+++

Um processo não consegue criar outro processo, trocar de programa ou terminar sozinho: tudo isso passa pelo kernel, através de **system calls**. Em [Exceptional Control Flow](/exceptional-control-flow/) vimos essas chamadas pelo ponto de vista do livro, e em [Variáveis de Ambiente](/variaveis-de-ambiente/) olhamos de perto o terceiro argumento do `execve`. Este post é um guia de consulta: as principais system calls de processos do Linux, cada uma com o protótipo, o significado de cada argumento, o retorno e os erros mais comuns, como uma man page explicada.

O ciclo de vida de um processo passa por quatro etapas, e cada uma tem as suas chamadas:

| Etapa | System calls |
|---|---|
| Criar | `fork` |
| Trocar de programa | `execve` |
| Terminar | `_exit` / `exit_group` |
| Ser recolhido pelo pai | `waitpid` / `wait4` / `waitid` |

Em volta delas ficam as chamadas de identidade (`getpid`, `getppid`), a de sinais (`kill`) e as de grupos e sessões (`setpgid`, `getpgid`, `setsid`), que o shell usa para controlar os jobs.

Todos os exemplos foram compilados e executados no Linux (x86-64), e as saídas mostradas são as reais.

## Antes de começar: system call x função da libc

Um programa em C quase nunca faz uma system call diretamente. Ele chama uma **função da libc** com o mesmo nome (ou um nome parecido), e essa função coloca os argumentos nos registradores, executa a instrução `syscall` e devolve o resultado. Por isso a documentação fica dividida em duas seções do `man`:

- **`man 2`**: system calls, como `man 2 fork`, `man 2 execve`, `man 2 kill`.
- **`man 3`**: funções da biblioteca, como `man 3 exit`, `man 3 printf`.

Nem sempre o nome da função é o nome da system call que o kernel recebe. O `strace` mostra as chamadas reais. Este programa faz `fork`, `exit` e `waitpid`:

```c
int main(void) {
    pid_t pid = fork();
    if (pid == 0)
        exit(7);
    int status;
    waitpid(pid, &status, 0);
    printf("filho %d saiu com %d\n", pid, WEXITSTATUS(status));
    return 0;
}
```

E o `strace` mostra o que o kernel recebeu:

```text
$ strace -f -e trace=fork,clone,execve,wait4,exit_group ./wrappers
execve("./wrappers", ["./wrappers"], 0x7fff7cd32508 /* 221 vars */) = 0
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, ...) = 134028
wait4(134028, <unfinished ...>
[pid 134028] exit_group(7)              = ?
[pid 134028] +++ exited with 7 +++
<... wait4 resumed>, [{WIFEXITED(s) && WEXITSTATUS(s) == 7}], 0, NULL) = 134028
filho 134028 saiu com 7
exit_group(0)                           = ?
```

- O `fork()` da libc virou a system call `clone`, que é a forma geral de criar processos no Linux.
- O `waitpid()` virou `wait4`.
- O `exit()` virou `exit_group`.

A convenção de erro é a mesma em todas as chamadas deste post: **em caso de erro, a função devolve `-1` e coloca o código do erro na variável global `errno`**. As funções `perror` e `strerror` transformam esse código em texto. Os códigos mais comuns de cada chamada aparecem nas seções abaixo.

## getpid e getppid: quem sou eu e quem é meu pai

```c
#include <unistd.h>

pid_t getpid(void);
pid_t getppid(void);
```

- **Argumentos:** nenhum.
- **Retorno:** `getpid` devolve o PID do próprio processo, e `getppid` devolve o PID do pai. O tipo `pid_t` é um inteiro com sinal (`int` no Linux).
- **Erros:** nenhum. As duas chamadas sempre funcionam.

O PID do pai pode mudar durante a vida do processo. Se o pai termina antes do filho, o filho vira **órfão** e é adotado por outro processo. Tradicionalmente, esse processo é o `init` (PID 1). Mas o Linux também permite que um processo se declare **subreaper**, e aí os órfãos dos seus descendentes vão para ele. Num desktop com systemd, quem adota é o `systemd --user` da sessão:

```c
int main(void) {
    if (fork() == 0) {
        printf("filho: meu pai é %d\n", getppid());
        sleep(1);                   /* o pai termina nesse meio-tempo */
        printf("filho: agora meu pai é %d\n", getppid());
        exit(0);
    }
    printf("pai:   eu sou %d e vou terminar\n", getpid());
    return 0;
}
```

```text
pai:   eu sou 134031 e vou terminar
filho: meu pai é 134031
filho: agora meu pai é 1082
```

O processo 1082 é o `/usr/lib/systemd/systemd --user`. Quem adota o órfão passa a ser responsável por recolhê-lo com `wait` quando ele terminar.

## fork: cria um processo

```c
#include <unistd.h>

pid_t fork(void);
```

- **Argumentos:** nenhum. O filho é sempre uma cópia de quem chamou, então não há nada para configurar.
- **Retorno:** o `fork` é chamado uma vez e **retorna duas vezes**, uma em cada processo:
  - **no pai**, devolve o PID do filho;
  - **no filho**, devolve `0`;
  - em caso de erro, devolve `-1` só no pai, porque nenhum filho foi criado.
- **Erros comuns:**
  - `EAGAIN`: o limite de processos foi atingido (o limite do usuário, `RLIMIT_NPROC`, ou o do sistema).
  - `ENOMEM`: não há memória para as estruturas do kernel do novo processo.

O retorno é a única forma de o código saber em qual processo está, por isso todo `fork` é seguido de um teste do valor devolvido:

```c
int main(void) {
    int x = 1;
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        exit(1);
    }
    if (pid == 0) {                 /* filho */
        x++;
        printf("filho: pid=%d, ppid=%d, fork devolveu %d, x=%d\n",
               getpid(), getppid(), pid, x);
        exit(0);
    }
    waitpid(pid, NULL, 0);          /* pai */
    x--;
    printf("pai:   pid=%d, ppid=%d, fork devolveu %d, x=%d\n",
           getpid(), getppid(), pid, x);
    return 0;
}
```

```text
filho: pid=134030, ppid=134029, fork devolveu 0, x=2
pai:   pid=134029, ppid=134018, fork devolveu 134030, x=0
```

O filho incrementou `x` e o pai decrementou, e cada um viu o seu próprio resultado: **os espaços de endereçamento são separados**. O PPID do pai (134018) é o shell que rodou o programa.

O que o filho recebe do pai:

| Herda (uma cópia) | Não herda |
|---|---|
| Memória: código, dados, heap, pilha | O PID (ganha um novo) |
| Descritores de arquivos abertos, apontando para as mesmas entradas, com o mesmo offset | Sinais pendentes (começa sem nenhum) |
| Variáveis de ambiente | Timers de `alarm` |
| Tratadores de sinais e a máscara de sinais bloqueados | Locks de arquivos (`fcntl`) |
| Diretório atual, `umask`, grupo de processos e sessão | Uso de CPU acumulado (zera) |

Copiar a memória inteira a cada `fork` seria caro. Por isso o Linux usa **copy-on-write**: pai e filho começam compartilhando as mesmas páginas físicas, marcadas como somente leitura, e uma página só é copiada de verdade quando um dos dois escreve nela. No exemplo acima, só a página que contém `x` (e as da pilha) precisaram ser copiadas.

## execve: troca o programa do processo

```c
#include <unistd.h>

int execve(const char *pathname, char *const argv[], char *const envp[]);
```

O `execve` carrega um programa novo **no processo atual**. O PID continua o mesmo, mas o código, os dados, o heap e a pilha do programa antigo são descartados e substituídos pelos do novo.

- **`pathname`**: o caminho do arquivo executável.
  - O `execve` **não procura no `PATH`**. Se você passar só `"ls"`, ele procura um arquivo chamado `ls` no diretório atual. Para usar o `PATH`, existe a função da libc `execvp`, que tenta cada diretório e chama o `execve` por baixo.
  - O arquivo pode ser um binário ELF ou um script que começa com `#!`. No caso do script, o kernel executa o interpretador indicado na primeira linha (`#!/bin/sh`, por exemplo) e passa o caminho do script para ele.
- **`argv`**: o vetor de argumentos que vira o `argv` da `main` do programa novo.
  - É um vetor de ponteiros para strings, terminado em `NULL`.
  - Por convenção, `argv[0]` é o nome do programa. O kernel não confere isso: o valor é só o que você passar.
- **`envp`**: o vetor de variáveis de ambiente do programa novo, no formato `"NOME=valor"` e também terminado em `NULL`. Normalmente se passa `environ`, para o programa novo herdar o ambiente atual. O post [Variáveis de Ambiente](/variaveis-de-ambiente/) explica esse argumento em detalhe.
- **Retorno:** em caso de sucesso, o `execve` **não retorna**, porque o código que o chamou não existe mais. Se ele retornar, deu erro, e o valor é `-1`.
- **Erros comuns:**
  - `ENOENT`: o arquivo não existe.
  - `EACCES`: o arquivo existe, mas não tem permissão de execução (ou um diretório do caminho não pode ser percorrido).
  - `ENOEXEC`: o arquivo tem permissão de execução, mas o kernel não reconhece o formato.
  - `E2BIG`: `argv` e `envp` juntos são grandes demais.
  - `ENOTDIR`: alguma parte do caminho não é um diretório.

```c
extern char **environ;

int main(void) {
    char *argv[] = {"ls", "-l", "/etc/hostname", NULL};

    execve("ls", argv, environ);            /* sem caminho: não procura no PATH */
    printf("execve(\"ls\") falhou: %s\n", strerror(errno));

    execve("/etc/hostname", argv, environ); /* existe, mas não é executável */
    printf("execve(\"/etc/hostname\") falhou: %s\n", strerror(errno));

    execve("/bin/ls", argv, environ);       /* agora sim */
    printf("isso nunca é impresso\n");
    return 1;
}
```

```text
execve("ls") falhou: No such file or directory
execve("/etc/hostname") falhou: Permission denied
-rw-r--r-- 1 root root 6 Sep 24 18:19 /etc/hostname
```

As duas primeiras chamadas retornaram com erro, e o programa seguiu em frente. A terceira deu certo, o processo virou o `ls`, e a última linha do programa nunca foi executada.

O que sobrevive ao `execve`:

| Mantém | Perde |
|---|---|
| PID, PPID, grupo de processos e sessão | Código, dados, heap e pilha |
| Descritores de arquivos abertos, exceto os marcados com `FD_CLOEXEC` | Tratadores de sinais: os sinais tratados voltam à ação padrão |
| Sinais ignorados continuam ignorados | Memória mapeada e memória compartilhada |
| Máscara de sinais bloqueados e sinais pendentes | Buffers do `stdio` que não foram esvaziados |
| Diretório atual e `umask` | |

Os tratadores voltam ao padrão porque o endereço de um tratador aponta para o código do programa antigo, que não existe mais. Já "ignorar" não depende de código nenhum, então pode continuar valendo. É também por causa dos descritores herdados que o redirecionamento do shell funciona: o shell abre o arquivo no lugar do `stdout` entre o `fork` e o `execve`, e o programa novo escreve nele sem saber.

O último item da tabela também merece cuidado: o que estiver no buffer do `printf` e ainda não tiver sido escrito some no `execve`. Na prática, chame `fflush(stdout)` antes de um `execve` se ainda houver saída pendente.

### A família exec

`execve` é a única system call. As outras variantes são funções da libc que montam os argumentos e chamam o `execve`. Os nomes seguem um padrão:

| Letra | Significado |
|---|---|
| `l` | Argumentos em lista: `execl("/bin/ls", "ls", "-l", NULL)` |
| `v` | Argumentos em vetor: `execv("/bin/ls", argv)` |
| `p` | Procura o programa no `PATH` |
| `e` | Recebe o ambiente explicitamente. Sem o `e`, usa o `environ` |

Assim, `execvp(file, argv)` é "vetor, procura no `PATH` e usa o `environ`".

## _exit e exit_group: termina o processo

```c
#include <unistd.h>

[[noreturn]] void _exit(int status);
```

- **`status`**: o código de saída que o pai vai receber no `wait`. **Só os 8 bits mais baixos chegam ao pai**, então o valor visível vai de 0 a 255. Por convenção, `0` significa sucesso e qualquer outro valor significa algum tipo de falha.
- **Retorno:** nenhum. A chamada nunca retorna.
- **Erros:** nenhum.

O kernel tem duas system calls de término. A `exit` termina só a thread que a chamou, e a `exit_group` termina todas as threads do processo. A função `_exit` da glibc chama a `exit_group`, que é o que se espera ao terminar um processo.

### exit x _exit

Na maioria dos programas se usa `exit` (sem sublinhado), que é uma função da libc. Ela faz uma limpeza antes de chamar o `_exit`:

1. executa as funções registradas com `atexit`, na ordem inversa do registro;
2. esvazia e fecha os streams do `stdio`, escrevendo o que estava no buffer;
3. chama `_exit(status)`.

Retornar da `main` com `return n` é o mesmo que chamar `exit(n)`. A diferença aparece quando há algo no buffer:

```c
void tchau(void) {
    printf("tchau (atexit)\n");
}

int main(int argc, char *argv[]) {
    atexit(tchau);
    printf("olá");                  /* sem \n: fica no buffer do stdio */
    if (argc > 1)
        _exit(3);
    exit(3);
}
```

```text
$ ./exit; echo " [$?]"
olátchau (atexit)
 [3]
$ ./exit x; echo " [$?]"
 [3]
```

Com `exit`, o `tchau` rodou e o "olá" foi escrito. Com `_exit`, o processo terminou na hora: o "olá" ficou no buffer e se perdeu, e a função do `atexit` não foi chamada. Nos dois casos o shell recebeu o código 3 (a variável `$?`).

Então por que usar `_exit`? Num filho criado por `fork` que não chegou a fazer `execve`, ou cujo `execve` falhou. O filho herdou uma cópia dos buffers do pai: se ele chamar `exit`, escreve de novo o que o pai já tinha no buffer, e a saída aparece duplicada. Ele também rodaria as funções do `atexit` do pai, que podem apagar arquivos temporários que o pai ainda está usando.

Quando um processo termina, o kernel fecha todos os descritores, libera a memória e manda `SIGCHLD` para o pai. Mas o processo não some totalmente: ele vira um **zumbi**, que guarda o código de saída até o pai recolhê-lo com um `wait`.

## waitpid, wait4 e waitid: recolhe o filho

```c
#include <sys/wait.h>

pid_t waitpid(pid_t pid, int *wstatus, int options);
```

O `waitpid` espera um filho mudar de estado (normalmente, terminar), recolhe o zumbi e diz como ele terminou.

- **`pid`**: qual filho esperar. O valor tem quatro significados:

  | Valor | Espera |
  |---|---|
  | `pid > 0` | O filho com exatamente esse PID |
  | `-1` | Qualquer filho |
  | `0` | Qualquer filho do mesmo grupo de processos de quem chamou |
  | `pid < -1` | Qualquer filho do grupo de processos `-pid` |

- **`wstatus`**: um ponteiro para um `int`, onde o kernel escreve como o filho terminou. Pode ser `NULL` quando isso não importa. O valor é codificado, e deve ser lido com macros:

  | Macro | Pergunta / extrai |
  |---|---|
  | `WIFEXITED(s)` | O filho terminou normalmente (com `exit` ou `return`)? |
  | `WEXITSTATUS(s)` | O código de saída, de 0 a 255. Só vale se `WIFEXITED` |
  | `WIFSIGNALED(s)` | O filho foi morto por um sinal? |
  | `WTERMSIG(s)` | O número do sinal que matou. Só vale se `WIFSIGNALED` |
  | `WCOREDUMP(s)` | Gerou um core dump? Só vale se `WIFSIGNALED` |
  | `WIFSTOPPED(s)` | O filho foi parado? Só com a opção `WUNTRACED` |
  | `WSTOPSIG(s)` | O sinal que parou o filho. Só vale se `WIFSTOPPED` |
  | `WIFCONTINUED(s)` | O filho voltou a rodar com `SIGCONT`? Só com `WCONTINUED` |

- **`options`**: muda o comportamento. Zero ou mais opções combinadas com `|`:
  - `0`: bloqueia até um filho terminar.
  - `WNOHANG`: não bloqueia. Se nenhum filho terminou, devolve `0` na hora.
  - `WUNTRACED`: também retorna quando um filho for **parado** (por `SIGSTOP` ou `SIGTSTP`, o Ctrl+Z).
  - `WCONTINUED`: também retorna quando um filho parado voltar a rodar.
- **Retorno:**
  - o PID do filho que mudou de estado;
  - `0`, se foi usado `WNOHANG` e nenhum filho mudou de estado;
  - `-1` em caso de erro.
- **Erros comuns:**
  - `ECHILD`: não há filho que corresponda ao `pid` pedido (ou nenhum filho, no caso de `-1`).
  - `EINTR`: a espera foi interrompida por um sinal tratado.
  - `EINVAL`: `options` inválido.

A função `wait(&status)` é só um atalho para `waitpid(-1, &status, 0)`.

O programa abaixo cria filhos que terminam de jeitos diferentes e decodifica cada `status`:

```c
void mostra(pid_t pid, int status) {
    if (WIFEXITED(status))
        printf("%d terminou normalmente, código %d\n", pid, WEXITSTATUS(status));
    else if (WIFSIGNALED(status))
        printf("%d morto pelo sinal %d (%s)\n", pid, WTERMSIG(status),
               strsignal(WTERMSIG(status)));
    else if (WIFSTOPPED(status))
        printf("%d parado pelo sinal %d (%s)\n", pid, WSTOPSIG(status),
               strsignal(WSTOPSIG(status)));
}

int main(void) {
    int status;
    pid_t pid;

    if ((pid = fork()) == 0)        /* 1: termina com exit(42) */
        exit(42);
    waitpid(pid, &status, 0);
    mostra(pid, status);

    if ((pid = fork()) == 0) {      /* 2: acessa memória inválida */
        *(int *)0 = 1;
        exit(0);
    }
    waitpid(pid, &status, 0);
    mostra(pid, status);

    if ((pid = fork()) == 0) {      /* 3: fica parado esperando */
        pause();
        exit(0);
    }
    kill(pid, SIGSTOP);
    waitpid(pid, &status, WUNTRACED);
    mostra(pid, status);

    printf("WNOHANG devolveu %d\n", waitpid(pid, &status, WNOHANG));
    kill(pid, SIGKILL);
    waitpid(pid, &status, 0);
    mostra(pid, status);

    if (waitpid(-1, &status, 0) < 0)
        perror("waitpid sem filhos");
    return 0;
}
```

```text
134099 terminou normalmente, código 42
134100 morto pelo sinal 11 (Segmentation fault)
134105 parado pelo sinal 19 (Stopped (signal))
WNOHANG devolveu 0
134105 morto pelo sinal 9 (Killed)
waitpid sem filhos: No child processes
```

1. O primeiro filho chamou `exit(42)`, e `WEXITSTATUS` devolveu 42.
2. O segundo escreveu no endereço 0 e foi morto pelo `SIGSEGV` (11).
3. O terceiro foi parado com `SIGSTOP`. Sem `WUNTRACED`, o `waitpid` continuaria bloqueado, porque um filho parado não terminou.
4. Com `WNOHANG`, o `waitpid` voltou na hora com `0`: o terceiro filho ainda estava parado, não tinha terminado.
5. Depois do `SIGKILL` (9), o filho foi recolhido.
6. Sem filhos restantes, o `waitpid` falhou com `ECHILD`.

É assim que o shell sabe o que mostrar em `$?`. Quando um programa é morto por um sinal, o bash coloca 128 + o número do sinal, como 139 para um segmentation fault.

### wait4: o mesmo, com uso de recursos

```c
#include <sys/wait.h>
#include <sys/resource.h>

pid_t wait4(pid_t pid, int *wstatus, int options, struct rusage *rusage);
```

O `wait4` é a system call que está por trás do `waitpid`, como o `strace` mostrou no início. Os três primeiros argumentos são os mesmos. O quarto, **`rusage`**, é um ponteiro para uma `struct rusage`, onde o kernel escreve quanto o filho consumiu: tempo de CPU em modo usuário (`ru_utime`) e em modo kernel (`ru_stime`), pico de memória (`ru_maxrss`), page faults, trocas de contexto. Se for `NULL`, nada é escrito, e a chamada vira exatamente o `waitpid`. É assim que o comando `/usr/bin/time` mede um programa.

### waitid: a versão mais flexível

```c
#include <sys/wait.h>

int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
```

O `waitid` faz o mesmo trabalho, mas separa as coisas que o `waitpid` misturava num número só.

- **`idtype`** e **`id`**: em vez de codificar tudo no sinal do `pid`, o tipo de espera vem num argumento separado:
  - `P_PID`: espera o filho cujo PID é `id`;
  - `P_PGID`: espera qualquer filho do grupo `id`;
  - `P_ALL`: espera qualquer filho, e `id` é ignorado.
- **`infop`**: um ponteiro para uma `siginfo_t`, preenchida com campos já separados, sem precisar de macros:
  - `si_pid`: o PID do filho;
  - `si_code`: o que aconteceu. Pode ser `CLD_EXITED` (terminou), `CLD_KILLED` (morto por sinal), `CLD_DUMPED` (morto com core dump), `CLD_STOPPED` (parado) ou `CLD_CONTINUED` (continuou);
  - `si_status`: o código de saída, ou o número do sinal, dependendo do `si_code`.
- **`options`**: aqui os eventos esperados precisam ser pedidos explicitamente. É preciso passar pelo menos um destes:
  - `WEXITED`: filhos que terminaram;
  - `WSTOPPED`: filhos parados;
  - `WCONTINUED`: filhos que continuaram.

  E dá para combinar com:
  - `WNOHANG`: não bloqueia, como no `waitpid`;
  - `WNOWAIT`: **só espia**. Devolve a informação, mas deixa o filho como zumbi, para que ele possa ser recolhido depois.
- **Retorno:** `0` em caso de sucesso e `-1` em caso de erro. Com `WNOHANG` e nenhum filho pronto, também devolve `0`, e no Linux o `si_pid` vem zerado.
- **Erros comuns:** `ECHILD`, `EINTR` e `EINVAL`, como no `waitpid`.

```c
int main(void) {
    siginfo_t info;
    pid_t pid = fork();
    if (pid == 0)
        exit(5);

    waitid(P_PID, pid, &info, WEXITED | WNOWAIT);   /* só espia */
    printf("espiando: pid=%d, si_code=%s, si_status=%d\n", info.si_pid,
           info.si_code == CLD_EXITED ? "CLD_EXITED" : "outro", info.si_status);

    waitid(P_PID, pid, &info, WEXITED);             /* agora recolhe */
    printf("recolhido: pid=%d, si_status=%d\n", info.si_pid, info.si_status);

    if (waitid(P_PID, pid, &info, WEXITED) < 0)
        perror("de novo");
    return 0;
}
```

```text
espiando: pid=134109, si_code=CLD_EXITED, si_status=5
recolhido: pid=134109, si_status=5
de novo: No child processes
```

Com `WNOWAIT`, o filho continuou zumbi e pôde ser recolhido na segunda chamada. A terceira falhou com `ECHILD`, porque o filho já tinha sido recolhido.

## kill: manda um sinal

```c
#include <signal.h>

int kill(pid_t pid, int sig);
```

Apesar do nome, o `kill` não serve só para matar: ele **manda um sinal qualquer** para um processo ou para um grupo de processos. Matar é só o efeito padrão da maioria dos sinais.

- **`pid`**: para quem mandar. Segue a mesma lógica do `waitpid`:

  | Valor | Destino |
  |---|---|
  | `pid > 0` | O processo com esse PID |
  | `0` | Todos os processos do grupo de quem chamou |
  | `-1` | Todos os processos para os quais quem chamou tem permissão, exceto o `init` (PID 1) e o próprio processo |
  | `pid < -1` | Todos os processos do grupo `-pid` |

- **`sig`**: o número do sinal, como `SIGTERM` (15), `SIGKILL` (9), `SIGINT` (2) ou `SIGSTOP` (19). O caso especial é o **`0`**: nenhum sinal é enviado, mas o kernel faz todas as verificações. Isso serve para testar se um processo existe e se você pode mandar sinais para ele.
- **Retorno:** `0` em caso de sucesso (pelo menos um sinal foi enviado) e `-1` em caso de erro.
- **Erros comuns:**
  - `EINVAL`: número de sinal inválido.
  - `EPERM`: sem permissão para mandar sinal a esse processo. Um processo comum só pode mandar sinais a processos do mesmo usuário. O `root` pode mandar para qualquer um.
  - `ESRCH`: o processo ou grupo não existe.

O `sig = 0` em ação:

```c
int main(int argc, char *argv[]) {
    pid_t pid = atoi(argv[1]);
    if (kill(pid, 0) == 0)
        printf("%d existe e posso mandar sinais para ele\n", pid);
    else
        printf("kill(%d, 0): %s\n", pid, strerror(errno));
    return 0;
}
```

```text
$ ./kill 1
kill(1, 0): Operation not permitted
$ ./kill $$
134018 existe e posso mandar sinais para ele
$ ./kill 999999
kill(999999, 0): No such process
```

- O PID 1 existe, mas pertence ao `root`, então deu `EPERM`.
- `$$` é o PID do próprio shell, que é do mesmo usuário.
- O PID 999999 não existe, e deu `ESRCH`.

O comando `kill` do terminal é uma interface para essa chamada. `kill -9 1234` é `kill(1234, SIGKILL)`, e `kill -TERM -- -1234` é `kill(-1234, SIGTERM)`, para o grupo inteiro.

## Grupos de processos e sessões

Os processos são organizados em dois níveis acima do PID:

- Um **grupo de processos** reúne processos que trabalham juntos. Num pipeline como `ls | sort | head`, o shell coloca os três no mesmo grupo, e esse grupo é o **job**. O ID do grupo (PGID) é o PID do **líder**, o processo que criou o grupo.
- Uma **sessão** reúne grupos. Normalmente cada terminal é uma sessão: o shell e todos os jobs que ele cria. O ID da sessão (SID) é o PID do **líder da sessão**, em geral o shell.

Os grupos existem para mandar sinais a vários processos de uma vez. Quando você aperta Ctrl+C, o terminal manda `SIGINT` para o grupo que está em primeiro plano, e os três programas do pipeline recebem o sinal ao mesmo tempo.

### setpgid e getpgid

```c
#include <unistd.h>

int setpgid(pid_t pid, pid_t pgid);
pid_t getpgid(pid_t pid);
```

O `setpgid` coloca um processo num grupo.

- **`pid`**: o processo que vai mudar de grupo. `0` significa o próprio processo.
- **`pgid`**: o grupo de destino. `0` significa "use o PID do processo como PGID", o que **cria um grupo novo** com esse processo como líder.
- **Retorno:** `0` em caso de sucesso e `-1` em caso de erro.
- **Erros comuns:**
  - `EACCES`: `pid` é um filho que já fez `execve`. Depois disso, o pai não pode mais mudar o grupo dele.
  - `EPERM`: o processo é líder de sessão, ou o grupo de destino está em outra sessão.
  - `ESRCH`: `pid` não é o próprio processo nem um filho dele.
  - `EINVAL`: `pgid` é negativo.

O `getpgid` faz a consulta. O argumento **`pid`** diz de qual processo se quer o PGID, e `0` significa o próprio processo. Ele devolve o PGID, ou `-1` com `ESRCH` se o processo não existir.

O shell usa essas chamadas assim: o primeiro processo de um job cria o grupo, os demais entram nele, e depois um único `kill` com PID negativo atinge todos. O programa abaixo faz o mesmo com dois filhos:

```c
int main(void) {
    pid_t pids[2], pgid = 0;

    printf("pai:   pid=%d pgid=%d sid=%d\n", getpid(), getpgid(0), getsid(0));

    for (int i = 0; i < 2; i++) {
        pid_t pid = fork();
        if (pid == 0) {
            setpgid(0, pgid);       /* o 1º cria o grupo, o 2º entra nele */
            printf("filho: pid=%d pgid=%d\n", getpid(), getpgid(0));
            pause();                /* espera um sinal */
            exit(0);
        }
        if (pgid == 0)
            pgid = pid;             /* o grupo leva o pid do 1º filho */
        setpgid(pid, pgid);         /* o pai faz o mesmo, para não ter corrida */
        pids[i] = pid;
    }
    sleep(1);

    kill(-pgid, SIGTERM);           /* pid negativo: o grupo inteiro */
    for (int i = 0; i < 2; i++) {
        int status;
        waitpid(pids[i], &status, 0);
        printf("pai:   %d morreu com o sinal %d\n", pids[i], WTERMSIG(status));
    }
    return 0;
}
```

```text
pai:   pid=134110 pgid=134110 sid=134089
filho: pid=134113 pgid=134113
filho: pid=134114 pgid=134113
pai:   134113 morreu com o sinal 15
pai:   134114 morreu com o sinal 15
```

- O pai está no seu próprio grupo (134110), que o shell criou para ele, e na sessão do shell (134089).
- O primeiro filho chamou `setpgid(0, 0)` e virou líder do grupo 134113.
- O segundo filho entrou no grupo 134113.
- Um único `kill(-134113, SIGTERM)` matou os dois.

O `setpgid` aparece duas vezes, no filho e no pai, e isso é de propósito. Depois do `fork`, não dá para saber quem roda primeiro. Se só o filho chamasse, o pai poderia mandar o `kill` antes de o grupo existir. Se só o pai chamasse, o filho poderia fazer `execve` antes, e o `setpgid` do pai falharia com `EACCES`. Chamando nos dois, o grupo fica certo qualquer que seja a ordem. Os shells fazem exatamente isso.

### setsid

```c
#include <unistd.h>

pid_t setsid(void);
```

O `setsid` cria uma **sessão nova**.

- **Argumentos:** nenhum. Quem chama é sempre o processo que vai para a sessão nova.
- **O que faz:**
  - o processo vira líder de uma sessão nova;
  - o processo também vira líder de um grupo novo dentro dela;
  - a sessão nova começa **sem terminal de controle**.
- **Retorno:** o ID da nova sessão (que é o PID do processo), ou `-1` em caso de erro.
- **Erros:** só `EPERM`, quando o processo **já é líder de um grupo de processos**.

Essa restrição explica um padrão comum. Um programa rodado pelo shell é líder do próprio grupo, então não pode chamar `setsid`. A solução é fazer um `fork` e chamar `setsid` no filho, que tem um PID novo e não lidera nada:

```c
int main(void) {
    if (setsid() < 0)
        perror("setsid");           /* eu sou líder do meu grupo */

    if (fork() == 0) {
        pid_t sid = setsid();       /* o filho não é líder de nada */
        printf("filho: nova sessão %d, pgid=%d, pid=%d\n",
               sid, getpgid(0), getpid());
        exit(0);
    }
    sleep(1);
    return 0;
}
```

```text
setsid: Operation not permitted
filho: nova sessão 134119, pgid=134119, pid=134119
```

Depois do `setsid`, o filho tem SID, PGID e PID iguais e não tem terminal. Fechar o terminal não manda mais `SIGHUP` para ele. É assim que **daemons** se desligam do terminal, e é o que faz o comando `setsid` do shell (`setsid programa`).

Também existe `getsid(pid)`, que devolve o ID da sessão de um processo (`0` para o próprio processo). O exemplo de grupos usou essa chamada para mostrar o SID do pai.

## Resumo

| System call | Argumentos | Retorno |
|---|---|---|
| `getpid()` / `getppid()` | nenhum | O PID do processo / do pai |
| `fork()` | nenhum | PID do filho no pai, `0` no filho, `-1` em erro |
| `execve(pathname, argv, envp)` | caminho do executável, vetor de argumentos e vetor de ambiente, os dois terminados em `NULL` | Não retorna. `-1` em erro |
| `_exit(status)` | código de saída (só 8 bits chegam ao pai) | Não retorna |
| `waitpid(pid, &status, options)` | qual filho (`>0`, `-1`, `0`, `<-1`), onde guardar o status, `WNOHANG`/`WUNTRACED`/`WCONTINUED` | PID do filho, `0` com `WNOHANG`, `-1` em erro |
| `wait4(pid, &status, options, &rusage)` | os mesmos do `waitpid`, mais o uso de recursos | Igual ao `waitpid` |
| `waitid(idtype, id, &info, options)` | `P_PID`/`P_PGID`/`P_ALL`, o ID, a `siginfo_t`, `WEXITED`/`WSTOPPED`/`WCONTINUED`/`WNOHANG`/`WNOWAIT` | `0`, ou `-1` em erro |
| `kill(pid, sig)` | destino (`>0`, `0`, `-1`, `<-1`), o sinal (`0` só testa) | `0`, ou `-1` em erro |
| `setpgid(pid, pgid)` | processo (`0` = eu) e grupo (`0` = grupo novo) | `0`, ou `-1` em erro |
| `getpgid(pid)` | processo (`0` = eu) | O PGID |
| `setsid()` | nenhum | O ID da nova sessão, ou `-1` se já for líder de grupo |

- **Criar, trocar e terminar:**
  - O `fork` duplica o processo e retorna duas vezes.
  - O `execve` troca o programa mantendo o PID, e só retorna em caso de erro.
  - O `_exit` termina na hora. O `exit` da libc faz a limpeza antes.
- **Recolher:**
  - Todo filho que termina vira zumbi até o pai recolhê-lo com uma chamada da família `wait`.
  - O `status` diz se ele terminou normalmente, foi morto ou foi parado.
- **Sinais e jobs:**
  - O `kill` manda sinais para processos ou para grupos inteiros.
  - O `setpgid` e o `setsid` organizam os processos em grupos e sessões, que é como o shell controla os jobs.
- **Libc x kernel:**
  - Várias dessas funções são wrappers: `fork` vira `clone`, `waitpid` vira `wait4` e `exit` vira `exit_group`.
  - Em caso de erro, todas devolvem `-1` e deixam o motivo em `errno`.

---

*Baseado nas man pages do Linux (seção 2), com exemplos testados no Linux, escrito pelo Claude*
