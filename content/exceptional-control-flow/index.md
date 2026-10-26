+++
date = '2026-10-26T16:25:59-03:00'
draft = true
title = 'Exceptional Control Flow'
type = 'blog'
tags = ['Arquitetura', 'Sistemas Operacionais', 'CSAPP']
+++

Do momento em que o computador é ligado até o momento em que é desligado, o *program counter* (PC) assume uma sequência de valores, cada um sendo o endereço de uma instrução. A passagem de um endereço para o próximo é chamada de **transferência de controle**, e a sequência inteira dessas transferências é o **fluxo de controle** (*control flow*) do processador.

O tipo mais simples de fluxo de controle é o "suave", em que cada instrução executada está logo depois da anterior na memória. Mudanças bruscas nesse fluxo, em que a próxima instrução não é a vizinha da atual, já são conhecidas: são causadas por instruções como **jumps**, **calls** e **returns**. Esses mecanismos permitem que o programa reaja a mudanças no seu próprio estado, representado pelas suas variáveis.

Só que o sistema também precisa reagir a mudanças que **não têm nada a ver com as variáveis do programa** e que muitas vezes nem estão relacionadas à execução dele. Alguns exemplos:

- Um timer de hardware dispara em intervalos regulares e isso precisa ser tratado.
- Pacotes chegam pela placa de rede e precisam ser guardados na memória.
- Um programa pede dados do disco e fica dormindo até ser avisado de que eles chegaram.
- Um processo pai que criou processos filhos precisa ser avisado quando um deles termina.

Os sistemas modernos reagem a essas situações fazendo mudanças bruscas no fluxo de controle. Essas mudanças são chamadas, em geral, de **exceptional control flow** (ECF), e elas acontecem em todos os níveis de um sistema computacional:

- **No hardware:** eventos detectados pelo hardware fazem o controle ser transferido para rotinas chamadas de *exception handlers*.
- **No sistema operacional:** o kernel transfere o controle de um processo para outro por meio das **trocas de contexto** (*context switches*).
- **Na aplicação:** um processo pode enviar um **sinal** para outro, fazendo o controle do processo que recebe ir para um *signal handler*.
- **Dentro de um programa:** um programa pode reagir a erros pulando para fora da disciplina normal da pilha, fazendo **saltos não locais** (*nonlocal jumps*) para pontos arbitrários de outras funções.

## Por que entender ECF

Existem alguns motivos para um programador se importar com ECF:

- **Para entender conceitos importantes de sistemas.** ECF é o mecanismo básico que o sistema operacional usa para implementar entrada e saída, processos e memória virtual.
- **Para entender como as aplicações conversam com o sistema operacional.** Quando um programa escreve no disco, lê dados da rede, cria um processo ou termina, ele faz uma **system call**, que é uma forma de ECF chamada de *trap*.
- **Para escrever programas mais interessantes.** O sistema operacional oferece mecanismos baseados em ECF para criar processos, esperar que terminem, avisar outros processos sobre eventos e reagir a eles. Com isso dá para escrever programas como um shell Unix ou um servidor web.
- **Para entender concorrência.** ECF é a base da concorrência: um exception handler que interrompe a aplicação, processos e threads que se revezam no processador, e um signal handler que interrompe a aplicação são todos exemplos de execução concorrente.
- **Para entender como as exceções de software funcionam.** Linguagens como C++ e Java oferecem exceções com `try`, `catch` e `throw`, que permitem fazer saltos não locais em resposta a erros. Em C, o equivalente são as funções `setjmp` e `longjmp`, e entender essas funções ajuda a entender como as exceções de mais alto nível são implementadas.

## O que vem pela frente

Até aqui, o foco foi em como as aplicações interagem com o hardware. A partir deste capítulo, o foco passa a ser em como elas interagem com o **sistema operacional**, e essas interações giram em torno de ECF. O caminho será:

1. **Exceções**, que ficam na fronteira entre o hardware e o sistema operacional.
2. **System calls**, exceções que dão às aplicações pontos de entrada no sistema operacional.
3. **Processos e sinais**, que ficam na fronteira entre a aplicação e o sistema operacional.
4. **Saltos não locais**, uma forma de ECF no nível da aplicação.

---

*Resumo do capítulo Exceptional Control Flow do livro Computer Systems: A Programmer's Perspective (CSAPP), de Randal E. Bryant e David R. O'Hallaron, compilado pelo Claude*
