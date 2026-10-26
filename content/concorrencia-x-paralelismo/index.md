+++
date = '2026-10-14T13:49:39-03:00'
draft = false
title = 'Concorrência x Paralelismo'
type = 'blog'
tags = ['Arquitetura', 'Concorrência', 'História']
+++

## Resumo

Concorrência e paralelismo parecem a mesma coisa, mas não são.

Um único núcleo roda apenas UM programa por vez. Para dar conta de vários, o sistema operacional alterna entre eles: roda um por alguns milissegundos, salva o estado dele e carrega o próximo. A troca é tão rápida que parece simultânea, mas em cada instante só um programa está usando o núcleo. Isso é concorrência: várias tarefas em andamento no mesmo período, se intercalando, mas sem executar ao mesmo tempo.

Paralelismo é executar coisas literalmente no mesmo instante, em partes diferentes do hardware. E ele não acontece de uma forma só. Existem diversas formas de paralelismo:

- **Paralelismo em nível de bit (bit-level):** o processador opera sobre mais bits de uma vez. Um processador de 8 bits precisa de duas instruções para somar dois números de 16 bits, enquanto um de 16 bits faz isso em uma só.
- **Paralelismo em nível de instrução (instruction-level):** várias instruções DO MESMO PROGRAMA executam ao mesmo tempo, como no pipelining.
- **Paralelismo de dados (data parallelism):** a mesma operação é aplicada a vários dados ao mesmo tempo. Por exemplo, somar dois vetores grandes dividindo os elementos entre os núcleos.
- **Paralelismo de tarefas (task parallelism):** tarefas diferentes executam ao mesmo tempo, cada uma em um núcleo, trabalhando sobre os mesmos dados ou não.

Durante décadas, os programas ficaram mais rápidos sem esforço do programador, graças ao aumento da frequência do clock. Quando esse caminho esbarrou no consumo de energia, em 2004, a saída foi colocar vários núcleos no mesmo processador. Desde então, ter vários núcleos não basta: o programador precisa escrever o código dividido em threads diferentes, para que o sistema operacional possa distribuí-las entre os núcleos. Um programa escrito de forma sequencial continua rodando normalmente, mas em um único núcleo, sem aproveitar os outros. E a lei de Amdahl mostra até onde vai o ganho de paralelizar.

Vamos entender um pouco sobre concorrência e paralelismo e a diferença entre os dois. Analisando a evolução dos computadores, podemos aprender mais sobre o assunto.

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

## Classificação de Flynn

Em 1966, Michael J. Flynn propôs uma forma de classificar arquiteturas de computadores que é usada até hoje. Ela olha para duas coisas: quantos fluxos de instruções (instruction streams) e quantos fluxos de dados (data streams) a máquina processa ao mesmo tempo. Combinando "único" (single) e "múltiplo" (multiple), saem quatro categorias:

|                             | Um fluxo de dados | Vários fluxos de dados |
| --------------------------- | ----------------- | ---------------------- |
| Um fluxo de instruções      | SISD              | SIMD                   |
| Vários fluxos de instruções | MISD              | MIMD                   |

- **SISD (Single Instruction, Single Data):** um único processador executa uma única sequência de instruções sobre dados guardados em uma única memória. É o computador sequencial clássico, como os das seções anteriores. Um processador com pipelining continua sendo SISD: as instruções se sobrepõem, mas ainda vêm de um único fluxo.
- **SIMD (Single Instruction, Multiple Data):** uma única instrução é aplicada a vários dados ao mesmo tempo. É o paralelismo de dados no hardware. Exemplos são os processadores vetoriais, as instruções SSE e AVX dos processadores x86 e as GPUs.
- **MISD (Multiple Instruction, Single Data):** várias instruções diferentes operam sobre o mesmo fluxo de dados. É a categoria com menos exemplos práticos. Às vezes ela é usada para descrever sistemas redundantes, em que vários processadores fazem o mesmo cálculo para detectar falhas.
- **MIMD (Multiple Instruction, Multiple Data):** vários processadores executam sequências de instruções diferentes sobre dados diferentes, ao mesmo tempo. É o caso dos processadores multi-core e dos clusters, e é onde entra o paralelismo de tarefas.

## Paralelismo em nível de instrução

O ILP (Instruction-Level Parallelism) é a capacidade de executar mais de uma instrução ao mesmo tempo, a partir de um único programa sequencial. O programador escreve as instruções em sequência, e o processador descobre quais delas não dependem umas das outras para executá-las juntas.

Ou seja, no paralelismo em nível de instrução, tudo acontece dentro do hardware. O programa continua sendo escrito de forma sequencial, sem threads e sem nenhuma mudança no código. Para o programador, esse paralelismo é invisível.

### Pipelining

Executar uma instrução envolve várias etapas. Em um processador RISC clássico, são cinco:

1. **IF (Instruction Fetch):** busca a instrução na memória.
2. **ID (Instruction Decode):** decodifica a instrução e lê os registradores.
3. **EX (Execute):** executa a operação, por exemplo uma soma na ULA.
4. **MEM (Memory Access):** lê ou escreve na memória, se a instrução precisar.
5. **WB (Write Back):** escreve o resultado no registrador.

Sem pipelining, uma instrução precisa passar por todas as etapas antes de a próxima começar. Enquanto a instrução está na etapa EX, por exemplo, a parte do processador que faz a busca fica parada, esperando.

O pipelining funciona como uma linha de montagem: assim que a primeira instrução sai da busca e vai para a decodificação, a segunda instrução já pode ser buscada. Cada etapa trabalha em uma instrução diferente ao mesmo tempo:

| Ciclo       | 1  | 2  | 3  | 4   | 5   | 6   | 7  |
| ----------- | -- | -- | -- | --- | --- | --- | -- |
| Instrução 1 | IF | ID | EX | MEM | WB  |     |    |
| Instrução 2 |    | IF | ID | EX  | MEM | WB  |    |
| Instrução 3 |    |    | IF | ID  | EX  | MEM | WB |

Sem pipelining, essas três instruções levariam 15 ciclos (5 cada uma). Com pipelining, levam 7. Depois que o pipeline está cheio, uma instrução termina a cada ciclo. Cada instrução continua levando os mesmos 5 ciclos para passar por todas as etapas. O que aumenta é a quantidade de instruções concluídas por unidade de tempo.

Na prática, o pipeline nem sempre flui assim. Se uma instrução depende do resultado da anterior, ela precisa esperar esse resultado ficar pronto. E em um desvio (um `if`, por exemplo), o processador só sabe qual é a próxima instrução depois de calcular a condição. Para não ficar parado, ele tenta adivinhar o caminho (predição de desvio) e, se errar, descarta as instruções que já tinha começado.

## Frequency scaling

Do meio dos anos 80 até 2004, o principal motivo de os computadores ficarem mais rápidos foi o aumento da frequência do clock (frequency scaling). O tempo de execução de um programa é o número de instruções multiplicado pelo tempo médio de cada instrução. Mantendo todo o resto igual, aumentar a frequência deixa cada ciclo mais curto, e cada instrução leva menos tempo. Assim, todo programa limitado pelo processador (compute-bound) ficava mais rápido sem que o programador mudasse uma linha de código. Bastava comprar um processador novo.

O problema é o consumo de energia. A potência consumida por um chip é dada por:

$$
P = C \times V^2 \times F
$$

Onde \(C\) é a capacitância chaveada a cada ciclo (proporcional ao número de transistores cujas entradas mudam), \(V\) é a tensão e \(F\) é a frequência (ciclos por segundo). Ou seja, aumentar a frequência aumenta a energia consumida, e toda essa energia vira calor. Chegou um ponto em que não dava mais para resfriar os processadores. O marco costuma ser 8 de maio de 2004, quando a Intel cancelou os processadores Tejas e Jayhawk, o que é considerado o fim do frequency scaling como principal caminho para ganhar desempenho.

## Processadores multi-core

A saída dos fabricantes foi colocar vários núcleos no mesmo processador (uma arquitetura MIMD, na classificação de Flynn). Cada núcleo é independente, e todos acessam a mesma memória de forma concorrente. **Com os processadores multi-core, a computação paralela chegou aos computadores pessoais**, e paralelizar programas sequenciais virou uma tarefa comum de programação. Em 2012, processadores com quatro núcleos já eram o padrão em desktops, e servidores tinham mais de dez. Em 2023, alguns processadores já passavam de cem núcleos. Há também projetos que misturam núcleos de desempenho e núcleos de eficiência, como o big.LITTLE da ARM, por causa das limitações térmicas.

### O papel do sistema operacional

Com vários núcleos, o sistema operacional deixa de apenas alternar programas em um único processador e passa também a distribuí-los entre os núcleos disponíveis. Se você abre o navegador, o editor de código e um player de música, o sistema pode colocar cada um em um núcleo diferente, e eles rodam em paralelo de verdade. Isso o sistema faz sozinho, sem que o programador precise mudar nada.

Mas o sistema operacional não consegue dividir um programa sequencial em pedaços. Um programa escrito como uma única sequência de instruções roda em um núcleo só, e não fica mais rápido porque o processador tem oito. Para aproveitar a arquitetura multi-core, o programador precisa reestruturar o código e paralelizá-lo, dividindo o trabalho em tarefas (threads, por exemplo) que o sistema possa espalhar pelos núcleos.

Ou seja, o ganho de desempenho não vem mais de graça. Com o frequency scaling, bastava esperar o próximo processador. Agora, para tirar proveito do poder de processamento cada vez maior dos processadores multi-core, os programadores precisam paralelizar o próprio código. E é aí que entra a lei de Amdahl.

## Lei de Amdahl

A lei de Amdahl, formulada por Gene Amdahl em 1967, mostra o limite do ganho que o paralelismo pode trazer. A ideia é simples: quase todo programa tem uma parte que pode ser dividida entre vários núcleos e uma parte que precisa rodar em sequência. Por mais núcleos que você adicione, a parte sequencial continua levando o mesmo tempo, e ela acaba limitando o ganho total.

Sendo \(p\) a fração do programa que pode ser paralelizada e \(n\) o número de núcleos, o ganho de velocidade (speedup) é:

$$
S = \frac{1}{(1 - p) + \frac{p}{n}}
$$

Por exemplo, se 90% do programa pode ser paralelizado (\(p = 0{,}9\)), com 10 núcleos o programa fica cerca de 5,3 vezes mais rápido, e não 10. Mesmo com infinitos núcleos, o termo \(\frac{p}{n}\) vai a zero, mas sobram os 10% sequenciais, então o ganho nunca passa de 10 vezes. Ou seja, depois de certo ponto, adicionar núcleos traz cada vez menos resultado, e o que mais importa é diminuir a parte sequencial do programa.

## Referências

- STALLINGS, William. _Arquitetura e Organização de Computadores_. Pearson.
- Wikipedia. [Parallel computing](https://en.wikipedia.org/wiki/Parallel_computing).

_Partes do texto baseadas na Wikipedia foram compiladas com a ajuda do Claude._
