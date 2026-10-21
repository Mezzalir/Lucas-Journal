+++
date = '2026-10-20T20:53:39-03:00'
draft = false
title = 'Buffer Overflow Attacks'
type = 'blog'
tags = ['Segurança', 'Arquitetura', 'CSAPP']
+++

Buffer overflow é uma das vulnerabilidades mais antigas e mais exploradas da computação. Ela acontece quando um programa escreve num array (o *buffer*) mais bytes do que ele comporta. Os bytes que sobram não são descartados: são gravados nas posições de memória logo depois do array, apagando o que estava lá.

Em C isso é possível porque a linguagem não verifica os limites de arrays. Se o array tem 8 posições e você escreve na posição 20, o C simplesmente escreve no endereço correspondente, sem nenhum erro ou aviso. O programa continua executando como se nada tivesse acontecido, só que agora com parte da memória corrompida.

Um atacante pode se aproveitar disso para sobrescrever dados importantes guardados na pilha, como o endereço de retorno de uma função, e desviar a execução do programa para um código que ele mesmo injetou. Para dificultar esse tipo de ataque, o GCC e o Linux adotam alguns mecanismos de proteção.

## Stack Randomization

Para inserir código de exploração em um sistema, o atacante precisa injetar tanto o código quanto um ponteiro para esse código como parte da string de ataque. Gerar esse ponteiro exige conhecer o endereço da pilha onde a string vai ficar. Historicamente, os endereços da pilha de um programa eram altamente previsíveis. Em todos os sistemas que rodavam a mesma combinação de versão do programa e do sistema operacional, as posições da pilha eram bastante estáveis entre as máquinas. Assim, por exemplo, se um atacante conseguisse determinar os endereços de pilha usados por um servidor Web comum, ele poderia criar um ataque que funcionaria em muitas máquinas. Usando doenças infecciosas como analogia, muitos sistemas eram vulneráveis exatamente à mesma cepa de um vírus, um fenômeno frequentemente chamado de *monocultura de segurança*.

> **Worms e vírus**
>
> Tanto worms quanto vírus são trechos de código que tentam se espalhar entre computadores. Como descrito por Spafford, um *worm* é um programa que pode rodar sozinho e propagar uma versão totalmente funcional de si mesmo para outras máquinas. Um *vírus* é um trecho de código que se adiciona a outros programas, incluindo sistemas operacionais. Ele não consegue rodar de forma independente. Na imprensa popular, o termo "vírus" é usado para se referir a uma variedade de estratégias diferentes de espalhar código de ataque entre sistemas, e por isso você vai ouvir pessoas dizendo "vírus" para o que, mais corretamente, deveria ser chamado de "worm".

A ideia da *stack randomization* é fazer a posição da pilha variar de uma execução do programa para outra. Assim, mesmo que muitas máquinas estejam rodando código idêntico, todas elas vão usar endereços de pilha diferentes. Isso é implementado alocando uma quantidade aleatória de espaço, entre 0 e *n* bytes, na pilha no início do programa, por exemplo usando a função de alocação `alloca`, que aloca na pilha um número especificado de bytes. Esse espaço alocado não é usado pelo programa, mas faz com que todas as posições seguintes da pilha variem de uma execução do programa para outra. O intervalo de alocação *n* precisa ser grande o suficiente para gerar variação suficiente nos endereços da pilha, mas pequeno o bastante para não desperdiçar espaço demais no programa.

O código a seguir mostra uma forma simples de determinar um endereço "típico" da pilha:

```c
int main() {
    long local;
    printf("local at %p\n", &local);
    return 0;
}
```

Esse código simplesmente imprime o endereço de uma variável local na função `main`. Rodando o código 10.000 vezes em uma máquina Linux no modo 32 bits, os endereços variaram de `0xff7fc59c` a `0xffffd09c`, uma faixa de cerca de 2²³. Rodando no modo 64 bits na máquina mais nova, os endereços variaram de `0x7fff0001b698` a `0x7ffffffaa4a8`, uma faixa de quase 2³².

A stack randomization se tornou prática padrão em sistemas Linux. Ela faz parte de uma classe maior de técnicas conhecida como *address-space layout randomization*, ou ASLR. Com a ASLR, diferentes partes do programa, incluindo o código do programa, o código das bibliotecas, a pilha, as variáveis globais e os dados do heap, são carregadas em regiões diferentes da memória a cada vez que o programa é executado. Isso significa que um programa rodando em uma máquina vai ter mapeamentos de endereço muito diferentes do mesmo programa rodando em outras máquinas. Isso consegue frustrar algumas formas de ataque.

No geral, porém, um atacante persistente consegue superar a randomização por força bruta, tentando ataques repetidamente com endereços diferentes. Um truque comum é incluir uma longa sequência de instruções `nop` (pronuncia-se "no op", abreviação de *no operation*) antes do código de exploração propriamente dito. Executar essa instrução não tem nenhum efeito além de incrementar o contador de programa para a próxima instrução. Desde que o atacante consiga adivinhar um endereço em algum lugar dentro dessa sequência, o programa vai percorrer a sequência e então atingir o código de exploração. O termo comum para essa sequência é *nop sled*, expressando a ideia de que o programa "desliza" pela sequência. Se montarmos um nop sled de 256 bytes, a randomização sobre *n* = 2²³ pode ser quebrada enumerando 2¹⁵ = 32.768 endereços iniciais, o que é totalmente viável para um atacante determinado. Para o caso de 64 bits, tentar enumerar 2²⁴ = 16.777.216 é um pouco mais intimidador. Podemos ver que a stack randomization e outros aspectos da ASLR podem aumentar o esforço necessário para atacar um sistema com sucesso e, portanto, reduzir muito a taxa com que um vírus ou worm consegue se espalhar, mas não conseguem oferecer uma proteção completa.

## Stack Corruption Detection

Uma segunda linha de defesa é conseguir detectar quando a pilha foi corrompida. Tome como exemplo a função `echo`, que lê uma linha da entrada para um buffer local de 8 bytes e a imprime de volta:

```c
void echo() {
    char buf[8];  /* Muito pequeno! */
    gets(buf);
    puts(buf);
}
```

A corrupção normalmente acontece quando o programa ultrapassa os limites de um buffer local, como o `buf` acima. Em C, não existe uma forma confiável de impedir a escrita além dos limites de um array. Em vez disso, o programa pode tentar detectar quando uma escrita dessas aconteceu, antes que ela tenha algum efeito nocivo.

Versões recentes do GCC incorporam ao código gerado um mecanismo conhecido como *stack protector* para detectar estouros de buffer. A ideia é guardar um valor especial chamado *canário* no stack frame, entre qualquer buffer local e o resto do estado da pilha, como ilustrado abaixo. (O nome vem do uso histórico desses pássaros para detectar a presença de gases perigosos em minas de carvão.) Esse valor canário, também chamado de *valor de guarda*, é gerado aleatoriamente a cada execução do programa, e por isso não há uma forma fácil de um atacante descobrir qual é.

```
                ┌──────────────────────────────┐
  Stack frame   │                              │
  do chamador   │                              │
                ├──────────────────────────────┤
                │       Endereço de retorno    │ ← %rsp+24
                ├──────────────────────────────┤
  Stack frame   │                              │
  de echo       │            Canário           │
                ├───┬───┬───┬───┬───┬───┬───┬───┤
                │[7]│[6]│[5]│[4]│[3]│[2]│[1]│[0]│ ← buf = %rsp
                └───┴───┴───┴───┴───┴───┴───┴───┘
```

*Organização da pilha para a função `echo` com o stack protector ativado. Um valor especial, o "canário", fica posicionado entre o array `buf` e o estado salvo. O código verifica o valor do canário para determinar se o estado da pilha foi corrompido ou não.*

Antes de restaurar o estado dos registradores e retornar da função, o programa verifica se o canário foi alterado por alguma operação dessa função ou de alguma função que ela chamou. Se foi, o programa é abortado com um erro.

Versões recentes do GCC tentam determinar se uma função é vulnerável a um estouro de pilha e inserem esse tipo de detecção automaticamente. Na verdade, para demonstrar um estouro de pilha em um programa compilado com o GCC, é preciso usar a opção de linha de comando `-fno-stack-protector` para impedir que ele insira esse código. Compilando a função `echo` sem essa opção, e portanto com o stack protector ativado, obtemos o seguinte código assembly:

```asm
# void echo()
echo:
  subq    $24, %rsp            # Aloca 24 bytes na pilha
  movq    %fs:40, %rax         # Recupera o canário
  movq    %rax, 8(%rsp)        # Guarda na pilha
  xorl    %eax, %eax           # Zera o registrador
  movq    %rsp, %rdi           # Calcula buf como %rsp
  call    gets                 # Chama gets
  movq    %rsp, %rdi           # Calcula buf como %rsp
  call    puts                 # Chama puts
  movq    8(%rsp), %rax        # Recupera o canário
  xorq    %fs:40, %rax         # Compara com o valor armazenado
  je      .L9                  # Se =, vai para ok
  call    __stack_chk_fail     # Pilha corrompida!
.L9:                           # ok:
  addq    $24, %rsp            # Desaloca o espaço da pilha
  ret
```

Vemos que essa versão da função recupera um valor da memória (`movq %fs:40, %rax`) e o guarda na pilha no deslocamento 8 a partir de `%rsp`, logo depois da região alocada para `buf`. O argumento da instrução `%fs:40` indica que o valor canário é lido da memória usando *endereçamento segmentado*, um mecanismo de endereçamento que vem desde o 80286 e raramente é encontrado em programas rodando em sistemas modernos. Ao guardar o canário em um segmento especial, ele pode ser marcado como "somente leitura", de forma que um atacante não consiga sobrescrever o valor canário armazenado. Antes de restaurar o estado dos registradores e retornar, a função compara o valor guardado na posição da pilha com o valor canário (através da instrução `xorq`). Se os dois forem idênticos, a instrução `xorq` vai resultar em zero, e a função vai terminar normalmente. Um valor diferente de zero indica que o canário na pilha foi modificado, e então o código vai chamar uma rotina de erro.

A proteção de pilha faz um bom trabalho em impedir que um ataque de buffer overflow corrompa o estado armazenado na pilha do programa. Ela gera apenas uma pequena perda de desempenho, principalmente porque o GCC só a insere quando existe um buffer local do tipo `char` na função. É claro que existem outras formas de corromper o estado de um programa em execução, mas reduzir a vulnerabilidade da pilha frustra muitas estratégias de ataque comuns.

## Limiting Executable Code Regions

Um passo final é eliminar a capacidade de um atacante inserir código executável em um sistema. Um método é limitar quais regiões da memória podem conter código executável. Em programas típicos, só a parte da memória que guarda o código gerado pelo compilador precisa ser executável. As outras partes podem ser restritas para permitir apenas leitura e escrita. O espaço de memória virtual é logicamente dividido em *páginas*, normalmente com 2.048 ou 4.096 bytes por página. O hardware oferece diferentes formas de *proteção de memória*, indicando os tipos de acesso permitidos tanto para os programas de usuário quanto para o kernel do sistema operacional. Muitos sistemas permitem controlar três formas de acesso: leitura (ler dados da memória), escrita (guardar dados na memória) e execução (tratar o conteúdo da memória como código de máquina). Historicamente, a arquitetura x86 juntava os controles de leitura e execução em uma única flag de 1 bit, de modo que qualquer página marcada como legível também era executável. A pilha precisava ser mantida tanto legível quanto gravável, e por isso os bytes na pilha também eram executáveis. Vários esquemas foram implementados para conseguir limitar algumas páginas a serem legíveis, mas não executáveis, mas eles geralmente introduziam ineficiências significativas.

Mais recentemente, a AMD introduziu um bit NX (de *no-execute*, "não executar") na proteção de memória dos seus processadores de 64 bits, separando os modos de acesso de leitura e de execução, e a Intel fez o mesmo. Com esse recurso, a pilha pode ser marcada como legível e gravável, mas não executável, e a verificação de se uma página é executável é feita em hardware, sem nenhuma perda de eficiência.

Alguns tipos de programa precisam da capacidade de gerar e executar código dinamicamente. Por exemplo, técnicas de compilação *just-in-time* geram código dinamicamente para programas escritos em linguagens interpretadas, como Java, para melhorar o desempenho da execução. Se o sistema de tempo de execução consegue ou não restringir o código executável apenas à parte gerada pelo compilador ao criar o programa original depende da linguagem e do sistema operacional.

As técnicas que apresentamos (randomização, proteção de pilha e limitação de quais partes da memória podem conter código executável) são três dos mecanismos mais comuns usados para minimizar a vulnerabilidade de programas a ataques de buffer overflow. Todas elas têm em comum o fato de não exigirem nenhum esforço especial por parte do programador e de terem pouca ou nenhuma perda de desempenho. Cada uma, separadamente, reduz o nível de vulnerabilidade, e em combinação elas se tornam ainda mais eficazes. Infelizmente, ainda existem formas de atacar computadores, e por isso worms e vírus continuam a comprometer a integridade de muitas máquinas.

## Supporting Variable-Size Stack Frames

Antes de tudo, uma breve recordação. Quando uma função precisa de mais armazenamento do que cabe nos registradores, ela reserva uma região na pilha só para ela: o seu *stack frame*. É ali que ficam as variáveis locais, os registradores que precisam ser salvos, os argumentos que não couberam em registradores e o endereço de retorno para quem a chamou. Normalmente o frame é alocado no início da função, subtraindo um valor fixo do stack pointer (`%rsp`), e liberado no final, somando o mesmo valor de volta. Como a pilha cresce em direção aos endereços menores, cada nova chamada empilha o seu frame abaixo do frame de quem a chamou. Esse assunto foi visto com mais detalhes no post sobre [procedimentos e a running stack](/procedimento-e-a-running-stack/).

Todas as funções que vimos até aqui têm a propriedade de que o compilador consegue determinar de antemão a quantidade de espaço que precisa ser alocada para seus stack frames. Algumas funções, porém, precisam de uma quantidade variável de armazenamento local. Isso pode acontecer, por exemplo, quando a função chama `alloca`, uma função da biblioteca padrão que pode alocar um número arbitrário de bytes na pilha. Também pode acontecer quando o código declara um array local de tamanho variável.

Este assunto é, na verdade, um aspecto de como procedimentos são implementados, mas exige um entendimento de [arrays](/arrays-e-structs-em-c/) e alinhamento.

O código da Figura 3.43(a) dá um exemplo de uma função contendo um array de tamanho variável. A função declara o array local `p` de *n* ponteiros, onde *n* é dado pelo primeiro argumento. Isso exige alocar 8*n* bytes na pilha, e o valor de *n* pode variar de uma chamada da função para outra. O compilador, portanto, não consegue determinar quanto espaço precisa alocar para o stack frame da função. Além disso, o programa gera uma referência ao endereço da variável local `i`, e por isso essa variável também precisa ser guardada na pilha. Durante a execução, o programa precisa conseguir acessar tanto a variável local `i` quanto os elementos do array `p`. Ao retornar, a função precisa desalocar o stack frame e posicionar o ponteiro de pilha na posição do endereço de retorno armazenado.

Para gerenciar um stack frame de tamanho variável, o código x86-64 usa o registrador `%rbp` como *frame pointer* (às vezes chamado de *base pointer*, daí as letras `bp` em `%rbp`). Quando se usa um frame pointer, o stack frame é organizado como mostrado na Figura 3.44 para o caso da função `vframe`. Vemos que o código precisa salvar a versão anterior de `%rbp` na pilha, já que ele é um registrador *callee-saved*. Depois, ele mantém `%rbp` apontando para essa posição durante toda a execução da função, e referencia variáveis locais de tamanho fixo, como `i`, por deslocamentos relativos a `%rbp`.

**(a) Código C**

```c
long vframe(long n, long idx, long *q) {
    long i;
    long *p[n];
    p[0] = &i;
    for (i = 1; i < n; i++)
        p[i] = q;
    return *p[idx];
}
```

**(b) Partes do código assembly gerado**

```asm
# long vframe(long n, long idx, long *q)
# n em %rdi, idx em %rsi, q em %rdx
# Apenas partes do código são mostradas
vframe:
  pushq   %rbp                  # Salva o %rbp antigo
  movq    %rsp, %rbp            # Define o frame pointer
  subq    $16, %rsp             # Aloca espaço para i (%rsp = s1)
  leaq    22(,%rdi,8), %rax
  andq    $-16, %rax
  subq    %rax, %rsp            # Aloca espaço para o array p (%rsp = s2)
  leaq    7(%rsp), %rax
  shrq    $3, %rax
  leaq    0(,%rax,8), %r8       # Define %r8 como &p[0]
  movq    %r8, %rcx             # Define %rcx como &p[0] (%rcx = p)
  # ...
# Código do laço de inicialização
# i em %rax e na pilha, n em %rdi, p em %rcx, q em %rdx
.L3:                            # loop:
  movq    %rdx, (%rcx,%rax,8)   # Define p[i] como q
  addq    $1, %rax              # Incrementa i
  movq    %rax, -8(%rbp)        # Guarda na pilha
.L2:
  movq    -8(%rbp), %rax        # Recupera i da pilha
  cmpq    %rdi, %rax            # Compara i:n
  jl      .L3                   # Se <, vai para loop
  # ...
# Código de saída da função
  leave                         # Restaura %rbp e %rsp
  ret                           # Retorna
```

*Figura 3.43: Função que exige o uso de um frame pointer. O array de tamanho variável faz com que o tamanho do stack frame não possa ser determinado em tempo de compilação.*

```
                     ┌─────────────────────┐
                     │ Endereço de retorno │
                   8 ├─────────────────────┤
                     │     %rbp salvo      │
  Frame pointer →  0 ├─────────────────────┤
  (%rbp)             │          i          │
                  −8 ├─────────────────────┤
                     │     (Não usado)     │
                 −16 ├─────────────────────┤ ← s1
                     │░░░░░░░░░░░░░░░░░░░░░│ } e1
                     ├─────────────────────┤
                     │                     │
                     │                     │
                     │          p          │   8n bytes
                     │                     │
                     │                     │
                     ├─────────────────────┤ ← p
                     │░░░░░░░░░░░░░░░░░░░░░│ } e2
  Stack pointer →    └─────────────────────┘ ← s2
  (%rsp)
```

*Figura 3.44: Estrutura do stack frame da função `vframe`. A função usa o registrador `%rbp` como frame pointer. As anotações do lado direito (e1, e2, s1 e s2) fazem referência a um exercício do livro (Practice Problem 3.49).*

A Figura 3.43(b) mostra partes do código que o GCC gera para a função `vframe`. No início da função, vemos o código que monta o stack frame e aloca espaço para o array `p`. O código começa empilhando o valor atual de `%rbp` e fazendo `%rbp` apontar para essa posição da pilha (`pushq %rbp` e `movq %rsp, %rbp`). Em seguida, ele aloca 16 bytes na pilha: os primeiros 8 são usados para guardar a variável local `i`, e os outros 8 não são usados. Então ele aloca espaço para o array `p` (as instruções de `leaq 22(,%rdi,8), %rax` até `movq %r8, %rcx`). Os detalhes de quanto espaço é alocado e de onde `p` é posicionado dentro desse espaço são explorados no Practice Problem 3.49 do livro. Basta dizer que, quando o programa chega em `movq %r8, %rcx`, ele (1) alocou pelo menos 8*n* bytes na pilha e (2) posicionou o array `p` dentro da região alocada de forma que pelo menos 8*n* bytes estejam disponíveis para uso.

O código do laço de inicialização mostra exemplos de como as variáveis locais `i` e `p` são referenciadas. A instrução `movq %rdx, (%rcx,%rax,8)` mostra o elemento `p[i]` recebendo `q`. Essa instrução usa o valor do registrador `%rcx` como o endereço do início de `p`. Podemos ver situações em que a variável local `i` é atualizada (`movq %rax, -8(%rbp)`) e lida (`movq -8(%rbp), %rax`). O endereço de `i` é dado pela referência `-8(%rbp)`, ou seja, no deslocamento −8 em relação ao frame pointer.

No final da função, o frame pointer é restaurado ao seu valor anterior usando a instrução `leave`. Essa instrução não recebe argumentos. Ela é equivalente a executar as duas instruções a seguir:

```asm
movq  %rbp, %rsp    # Posiciona o stack pointer no início do frame
popq  %rbp          # Restaura o %rbp salvo e posiciona o stack pointer
                    # no fim do frame do chamador
```

Ou seja, o stack pointer primeiro é posicionado no endereço onde está o valor salvo de `%rbp`, e então esse valor é desempilhado para `%rbp`. Essa combinação de instruções tem o efeito de desalocar todo o stack frame.

Em versões anteriores do código x86, o frame pointer era usado em toda chamada de função. No código x86-64, ele é usado apenas nos casos em que o stack frame pode ter tamanho variável, como no caso da função `vframe`. Historicamente, a maioria dos compiladores usava frame pointers ao gerar código IA32. Versões recentes do GCC abandonaram essa convenção. Observe que é aceitável misturar código que usa frame pointers com código que não usa, desde que todas as funções tratem `%rbp` como um registrador callee-saved.

---

*Trecho das seções Stack Randomization, Stack Corruption Detection, Limiting Executable Code Regions e Supporting Variable-Size Stack Frames extraído do livro Computer Systems: A Programmer's Perspective (CSAPP), de Randal E. Bryant e David R. O'Hallaron, e traduzido pelo Claude*
