+++
date = '2026-10-14T13:49:39-03:00'
draft = false
title = 'Concorrência x Paralelismo'
type = 'blog'
tags = ['Arquitetura', 'História']
+++

## Resumo

Um único núcleo roda apenas UM programa por vez. O sistema operacional alterna entre programas: roda um por alguns milissegundos, salva o estado, carrega o próximo. É tão rápido que parece simultâneo, mas em cada instante só um programa está usando o núcleo. Isso se chama concorrência. As tarefas se intercalam, mas não executam ao mesmo tempo. O paralelismo em nível de instrução é executar múltiplas instruções ao mesmo tempo, DO MESMO PROGRAMA.

_Em termos de software_, concorrência é ter várias tarefas em andamento no mesmo período, mas não necessariamente executando no mesmo instante. Elas podem se revezar no processador. No paralelismo, várias tarefas estão executando literalmente no mesmo instante, em unidades de hardware diferentes (vários núcleos, por exemplo).

---

Vamos entender um pouco sobre concorrência e paralelismo e a diferença entre os dois. Analisando a evolução dos computadores, podemos aprender mais sobre o assunto.

Este post será atualizado conforme os estudos avançarem.

## Antes do time-sharing: processamento em lote

Antigamente, computadores eram caríssimos e enormes. Ocupavam facilmente uma sala inteira, e havia dezenas de pessoas trabalhando com a máquina. Apenas governos, militares, grandes empresas e universidades possuíam esses computadores. Ainda não havia surgido o PC (Personal Computer).

Nos anos 50 e 60, o comum era o processamento de jobs em lote. O programador escrevia o programa no papel, em alguma linguagem de alto nível, como o Fortran, e depois ele era perfurado em cartões por um perfurador (keypunch operator) ou pelo próprio programador. Esse maço de cartões, com o programa, os dados e alguns cartões de controle dizendo à máquina o que fazer, formava um job.

O programador entregava o maço no centro de computação, e lá o operador do computador juntava diversos jobs em um lote (batch) e os colocava para a máquina executar, um atrás do outro. O tempo de máquina era caríssimo, e o lote existia justamente para que ela não ficasse parada entre um job e outro. O custo ficava com o programador: a operação levava horas, e ele só descobria a saída do seu programa no dia seguinte. Se houvesse um erro de digitação, era preciso corrigir e esperar tudo de novo.

## Time-sharing

O compartilhamento de tempo (time-sharing) permite que vários usuários utilizem o computador ao mesmo tempo, cada um no seu terminal, de forma interativa.

É importante dizer que o processador continua executando um programa por vez. O sistema operacional dá a cada usuário uma fatia de tempo curta (na ordem de milissegundos) e depois passa para o próximo. Essa passagem de um programa para outro é a troca de contexto. Como a troca é muito rápida, o usuário tem a impressão de ter o computador só para ele.

Como a troca acontece:

1. Quando a fatia de tempo do usuário acaba, um timer dispara uma interrupção.
2. O processador interrompe o programa A e pula para o código do sistema operacional.
3. O sistema salva os registradores e as flags de A em uma estrutura própria para ele. O contexto do programa precisa ser salvo!
4. O sistema escolhe o próximo usuário.
5. O sistema carrega o próximo programa e restaura o contexto dele.

Quando chega de novo a vez de A, o contexto dele é restaurado, e o programa continua exatamente de onde tinha parado, como se nada tivesse acontecido.

## Paralelismo em nível de instrução

O ILP (Instruction-Level Parallelism) é a capacidade de executar mais de uma instrução ao mesmo tempo, a partir de um único programa sequencial. O programador escreve as instruções em sequência, e o processador descobre quais delas não dependem umas das outras para executá-las juntas.

### Pipeline

Executar uma instrução envolve várias etapas.
