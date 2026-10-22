+++
date = '2026-10-22T20:14:20-03:00'
draft = false
title = 'Tecnologias de Armazenamento'
type = 'blog'
tags = ['Arquitetura', 'Armazenamento']
+++

Todo sistema de computação guarda dados em várias tecnologias diferentes, cada uma com um equilíbrio próprio entre velocidade, capacidade e custo. Este post resume as principais: as memórias de acesso aleatório (SRAM e DRAM), as memórias não voláteis (ROM, PROM, EPROM, EEPROM e flash), os discos, os SSDs e, por fim, como as memórias cache são organizadas e como escrever código que tire proveito delas.

## Memória de acesso aleatório (RAM)

Existem dois tipos de RAM: a estática (SRAM) e a dinâmica (DRAM). A SRAM é mais rápida e bem mais cara, e é usada nas memórias cache. A DRAM é usada na memória principal.

### SRAM

A SRAM guarda cada bit em uma célula *biestável*, feita com seis transistores. Essa célula pode ficar em um de dois estados estáveis, e permanece nele enquanto houver energia. Mesmo que uma interferência elétrica a perturbe, ela volta sozinha para o estado estável.

### DRAM

A DRAM guarda cada bit como uma carga em um capacitor muito pequeno, controlado por um único transistor. Por isso ela é muito mais densa e barata que a SRAM. Em compensação, é sensível: o capacitor perde a carga com o tempo, e cada célula precisa ser lida e regravada (*refresh*) a cada poucas dezenas de milissegundos.

|      | Transistores por bit | Tempo de acesso relativo | Mantém os dados sem refresh? | Custo relativo | Uso              |
| ---- | -------------------- | ------------------------ | ---------------------------- | -------------- | ---------------- |
| SRAM | 6                    | 1×                       | Sim                          | 1.000×         | Memória cache    |
| DRAM | 1                    | 10×                      | Não                          | 1×             | Memória principal |

As duas são **voláteis**: perdem os dados quando a energia é desligada.

### Organização de um chip de DRAM

Um chip de DRAM é dividido em *supercélulas*, cada uma com alguns bits (8, por exemplo), organizadas em uma matriz de linhas e colunas. Para ler uma supercélula, o controlador de memória envia primeiro o número da linha (RAS, *row access strobe*), e o chip copia a linha inteira para um buffer interno. Depois envia o número da coluna (CAS, *column access strobe*), e o chip devolve a supercélula daquela coluna.

Organizar em matriz reduz o número de pinos de endereço do chip, mas o acesso passa a ter dois passos, o que aumenta o tempo de acesso.

Os chips são vendidos em **módulos de memória** (DIMMs). Um módulo típico tem 8 chips, e cada um fornece 1 byte de uma palavra de 64 bits. Para ler uma palavra, o controlador acessa a mesma supercélula nos 8 chips ao mesmo tempo e junta os bytes.

### DRAMs melhoradas

Como a DRAM é a memória principal, ela foi sendo melhorada ao longo do tempo para acompanhar os processadores:

- **FPM DRAM (Fast Page Mode):** permite ler várias supercélulas da mesma linha sem reenviar o número da linha.
- **EDO DRAM (Extended Data Out):** parecida com a FPM, mas com os sinais de coluna mais próximos no tempo.
- **SDRAM (Synchronous DRAM):** sincronizada com o clock do controlador de memória, o que permite transferir dados mais rápido.
- **DDR SDRAM (Double Data-Rate):** transfere dados nas duas bordas do clock, dobrando a velocidade. As gerações DDR2, DDR3, DDR4 e seguintes aumentam o buffer de pré-busca, transferindo cada vez mais bits por acesso.
- **VRAM (Video RAM):** usada em placas de vídeo, permite ler e escrever ao mesmo tempo.

## Memória não volátil

Memórias não voláteis mantêm os dados mesmo sem energia. Por razões históricas, todas são chamadas de ROM (*read-only memory*), mesmo as que podem ser escritas. Elas se diferenciam pelo número de vezes que podem ser regravadas e pelo jeito como isso é feito:

- **ROM:** gravada na fábrica, não pode ser alterada.
- **PROM (Programmable ROM):** pode ser programada uma única vez. Cada célula tem um fusível, que é queimado com uma corrente alta.
- **EPROM (Erasable Programmable ROM):** tem uma janela de quartzo no chip. Para apagar, o chip é exposto à luz ultravioleta. Aguenta cerca de mil ciclos de gravação.
- **EEPROM (Electrically Erasable PROM):** apagada eletricamente, sem precisar tirar o chip da placa. Aguenta cerca de 100 mil ciclos de gravação.
- **Memória flash:** baseada na EEPROM, é a tecnologia usada em pendrives, cartões de memória, celulares e SSDs.

Os programas guardados em ROM são chamados de *firmware*. Quando o computador liga, ele executa o firmware guardado em ROM (como a BIOS ou a UEFI). Dispositivos como controladores de disco e placas de rede também têm seu próprio firmware.

## Acessando a memória principal

### Barramentos

Os dados viajam entre o processador e a memória por **barramentos**. Um barramento é um conjunto de fios paralelos, chamados de *linhas*, e cada linha transmite um bit por vez. As linhas são divididas em três grupos, de acordo com o que carregam:

- **Linhas de endereço:** indicam qual posição de memória vai ser acessada. O número de linhas define quanta memória pode ser endereçada: com n linhas, é possível endereçar \(2^n\) posições. Com 32 linhas, por exemplo, são \(2^{32}\) bytes, ou 4 GB, que era o limite de memória dos processadores de 32 bits.
- **Linhas de dados:** carregam os dados que estão sendo lidos ou escritos. O número de linhas é a *largura* do barramento, e define quantos bits são transferidos de uma vez. Um barramento de 64 bits transfere uma palavra de 8 bytes por transferência.
- **Linhas de controle:** carregam os sinais que coordenam a comunicação. Elas dizem se a operação é uma leitura ou uma escrita, se o acesso é à memória ou a um dispositivo de E/S, quando o endereço e os dados são válidos (sincronizando tudo pelo clock), e também transmitem pedidos de interrupção e de uso do barramento.

Em alguns projetos, endereço e dados compartilham as mesmas linhas, enviados um depois do outro. Isso economiza fios, mas deixa cada transferência mais lenta.

No modelo do livro, existem dois barramentos ligados por uma ponte, a *I/O bridge*: o **barramento do sistema**, que liga o processador à ponte, e o **barramento de memória**, que liga a ponte à memória principal. A ponte traduz os sinais de um barramento para o outro, e também liga os dois ao barramento de E/S.

### Transações de leitura e escrita

Cada troca de dados pelo barramento é chamada de *transação*. Uma leitura como `movq A, %rax` vira uma **transação de leitura**:

1. O processador coloca o endereço `A` nas linhas de endereço do barramento do sistema, e sinaliza uma leitura nas linhas de controle.
2. A ponte repassa o endereço para o barramento de memória.
3. A memória lê o endereço, busca a palavra na DRAM e a coloca nas linhas de dados do barramento de memória.
4. A ponte repassa a palavra para o barramento do sistema.
5. O processador lê a palavra das linhas de dados e a copia para o registrador `%rax`.

Uma escrita como `movq %rax, A` é uma **transação de escrita**: o processador envia o endereço, sinaliza uma escrita, envia a palavra pelas linhas de dados, e a memória grava a palavra na posição indicada.

### Tempo de acesso

Uma leitura da memória principal leva em torno de 50 a 100 ns. Parece pouco, mas para um processador de 3 GHz, em que cada ciclo dura cerca de 0,33 ns, 100 ns equivalem a cerca de 300 ciclos parado esperando o dado. Esse tempo soma vários passos: a viagem pelos barramentos, a ponte, o acesso em dois passos da DRAM (linha e depois coluna) e a volta.

Existem duas medidas diferentes de desempenho da memória:

- **Latência:** quanto tempo leva para o primeiro dado chegar depois do pedido. É o que pesa quando o programa precisa de um dado específico e não tem o que fazer enquanto espera.
- **Largura de banda:** quantos bytes podem ser transferidos por segundo depois que a transferência começa. Ela é a largura do barramento de dados vezes o número de transferências por segundo. Uma memória DDR4-3200, por exemplo, faz 3,2 bilhões de transferências por segundo em um barramento de 64 bits (8 bytes), o que dá 25,6 GB/s.

A largura de banda melhorou muito mais que a latência ao longo dos anos. As DRAMs melhoradas, como a DDR, transferem cada vez mais dados por segundo, mas o tempo para o primeiro dado chegar caiu pouco. Essa latência alta é a principal razão para existirem as memórias cache.

## Discos

### Geometria

Um disco rígido é formado por *pratos*, cada um com duas *superfícies* cobertas de material magnético. Os pratos giram presos a um eixo central, normalmente entre 5.400 e 15.000 rotações por minuto (RPM). Cada superfície é dividida em círculos concêntricos chamados *trilhas*, e cada trilha é dividida em *setores*, que normalmente guardam 512 bytes. O conjunto das trilhas na mesma posição em todas as superfícies é chamado de *cilindro*.

A capacidade de um disco é calculada multiplicando:

```
bytes por setor × setores por trilha × trilhas por superfície × superfícies por prato × pratos
```

Os fabricantes de discos usam GB = \(10^9\) bytes, e não \(2^{30}\). Por isso um disco de "1 TB" aparece com menos espaço no sistema operacional.

### Tempo de acesso

Para ler um setor, a cabeça de leitura precisa se mover até a trilha certa e esperar o setor passar por baixo dela. O tempo de acesso tem três partes:

1. **Seek time:** o tempo para mover o braço até a trilha. Normalmente de 3 a 9 ms.
2. **Latência rotacional:** o tempo esperando o setor chegar. Em média, meia rotação.
3. **Tempo de transferência:** o tempo para o setor passar pela cabeça.

No exemplo do livro, um disco de 7.200 RPM com seek time médio de 9 ms e 400 setores por trilha tem latência rotacional média de 4 ms e tempo de transferência de 0,02 ms, o que dá cerca de 13 ms no total. Duas conclusões saem disso: o tempo é dominado pelo seek e pela rotação, então o primeiro byte é caro e o resto do setor sai quase de graça. E um disco é dezenas de milhares de vezes mais lento que a SRAM e milhares de vezes mais lento que a DRAM.

### Blocos lógicos

O sistema operacional não precisa conhecer essa geometria. O disco se apresenta como uma sequência de *blocos lógicos* numerados, e o controlador do disco traduz cada número para a superfície, a trilha e o setor correspondentes.

## Conectando dispositivos de E/S

Discos, placas de vídeo, teclados e outros dispositivos se conectam ao processador e à memória por um barramento de E/S, como o PCI Express. Controladores e adaptadores fazem a ponte entre o barramento e cada dispositivo (USB, SATA, SCSI).

Quando o processador pede para ler um bloco do disco, ele não fica esperando. O disco faz a leitura e copia os dados direto para a memória principal, sem passar pelo processador. Isso se chama **DMA (Direct Memory Access)**. Quando termina, o controlador avisa o processador com uma *interrupção*. Enquanto isso, o processador pode executar outras coisas.

## SSDs

Um SSD (*Solid State Disk*) é feito de memória flash e se comporta como um disco para o sistema, mas sem partes móveis. Uma camada de software no próprio SSD, a *flash translation layer*, traduz os blocos lógicos para a memória flash.

A memória flash é dividida em *blocos*, e cada bloco em *páginas*. Os dados são lidos e escritos por página, mas uma página só pode ser escrita depois que o bloco inteiro foi apagado. Isso tem algumas consequências:

- **Leituras são mais rápidas que escritas.**
- **Escritas aleatórias são lentas**, porque apagar um bloco demora, e às vezes é preciso copiar as páginas que ainda são usadas para outro bloco antes.
- **Os blocos se desgastam** depois de um certo número de apagamentos. Para compensar, o SSD espalha as escritas entre todos os blocos (*wear leveling*), para que nenhum se desgaste antes dos outros.

Comparado com um disco rígido, o SSD é muito mais rápido, gasta menos energia e é mais resistente a impactos. Em compensação, o custo por byte é maior.

### SSD SATA x SSD NVMe x HD

Nem todo SSD tem o mesmo desempenho. Boa parte da diferença está na forma como ele se conecta ao computador:

- **SATA:** é a mesma interface usada pelos discos rígidos. Ela foi projetada para HDs, e o próprio padrão (SATA III, de 6 Gb/s) limita a transferência a cerca de 550 MB/s. O protocolo usado (AHCI) também foi pensado para discos, com uma única fila de 32 comandos.
- **NVMe:** o SSD se conecta direto ao barramento PCI Express, sem passar pela controladora SATA. O protocolo foi criado para memória flash e permite milhares de filas, cada uma com milhares de comandos, o que aproveita o paralelismo dos vários chips de flash do SSD.

Valores típicos de cada tecnologia:

| Tecnologia                  | Leitura sequencial | Tempo de acesso aleatório | Operações aleatórias por segundo (IOPS) |
| --------------------------- | ------------------ | ------------------------- | --------------------------------------- |
| HD (7.200 RPM)              | ~150 a 250 MB/s    | ~5 a 10 ms                | ~100 a 200                              |
| SSD SATA                    | ~550 MB/s          | ~0,1 ms (100 µs)          | ~100 mil                                |
| SSD NVMe (PCIe 4.0)         | ~7.000 MB/s        | ~0,02 ms (20 µs)          | ~1 milhão                               |

Em ordens de grandeza:

- **Leitura sequencial:** o SSD SATA é cerca de 3 vezes mais rápido que o HD, e o NVMe é mais de 10 vezes mais rápido que o SATA. Do HD para o NVMe, a diferença passa de 30 vezes.
- **Acesso aleatório:** aqui a diferença é muito maior, porque o HD precisa mover o braço e esperar o disco girar. O SSD SATA responde cerca de 100 vezes mais rápido que o HD, e o NVMe, cerca de 500 vezes.
- **IOPS:** o SSD SATA faz cerca de mil vezes mais operações aleatórias por segundo que o HD, e o NVMe, cerca de 10 mil vezes. São quatro ordens de grandeza.

Por isso, a troca de um HD por qualquer SSD é a que mais se sente no uso do dia a dia: o sistema e os programas fazem muitos acessos pequenos e espalhados, que são justamente o ponto fraco do HD. A diferença entre SATA e NVMe aparece mais em tarefas que transferem muitos dados de uma vez, como copiar arquivos grandes ou carregar jogos.

Esses números são valores típicos e mudam de modelo para modelo. SSDs NVMe com PCIe 5.0, por exemplo, já passam de 12.000 MB/s.

## Tendências

Cada tecnologia evolui em um ritmo diferente. A densidade e o custo por byte melhoram muito mais rápido que o tempo de acesso, principalmente na DRAM e nos discos. Enquanto isso, os processadores ficaram muito mais rápidos. A distância entre a velocidade do processador e a da memória só aumenta, e é por isso que os sistemas dependem cada vez mais das memórias cache.

## Memórias cache

O conceito de cache e as políticas de mapeamento, substituição e escrita estão no post sobre [Memória Cache](/memoria-cache/). Aqui fica a forma como o livro descreve a organização de uma cache, que é útil para entender o desempenho dos programas.

Uma cache é descrita por quatro números, (S, E, B, m):

- **\(S = 2^s\) conjuntos (sets).**
- **E linhas por conjunto.**
- **\(B = 2^b\) bytes por bloco.** Cada linha guarda um bloco, um bit de validade e uma tag.
- **m bits de endereço.**

A capacidade da cache é C = S × E × B. Cada endereço de memória é dividido em três partes:

| t bits | s bits | b bits |
| ------ | ------ | ------ |
| Tag    | Índice do conjunto | Deslocamento no bloco |

Para ler um dado, a cache usa o índice para escolher o conjunto, compara a tag com as linhas do conjunto e, se encontrar uma linha válida com a mesma tag, usa o deslocamento para achar o byte dentro do bloco. Com E = 1, a cache é de mapeamento direto. Com S = 1, é totalmente associativa. Entre os dois, é associativa por conjunto.

Em um processador como o Intel Core i7, cada núcleo tem uma L1 de instruções (*i-cache*) e uma L1 de dados (*d-cache*), separadas, e uma L2 unificada. A L3 é compartilhada por todos os núcleos. Todas usam blocos de 64 bytes.

## Escrevendo código amigável à cache

Programas com boa localidade têm menos falhas de cache e rodam mais rápido. Duas regras práticas:

1. **Concentrar a atenção nos laços internos**, onde está a maior parte dos acessos.
2. **Acessar a memória sequencialmente** (*stride-1*), aproveitando todos os bytes de cada bloco trazido para a cache.

Em C, matrizes são guardadas linha por linha na memória. Percorrer uma matriz por linhas acessa endereços seguidos. Percorrer por colunas pula de uma linha para outra a cada acesso e desperdiça quase todo o bloco:

```c
/* Boa localidade: percorre por linhas (stride-1) */
int sumarrayrows(int a[M][N]) {
    int i, j, sum = 0;
    for (i = 0; i < M; i++)
        for (j = 0; j < N; j++)
            sum += a[i][j];
    return sum;
}

/* Localidade ruim: percorre por colunas (stride-N) */
int sumarraycols(int a[M][N]) {
    int i, j, sum = 0;
    for (j = 0; j < N; j++)
        for (i = 0; i < M; i++)
            sum += a[i][j];
    return sum;
}
```

As duas funções fazem exatamente a mesma conta, mas a primeira pode ser muitas vezes mais rápida.

## O impacto da cache no desempenho

O livro mostra o efeito da cache com a **montanha de memória** (*memory mountain*): um gráfico da taxa de leitura de um programa em função de dois parâmetros, o tamanho dos dados e o stride dos acessos.

- **Variando o tamanho**, aparecem "degraus" que correspondem a cada nível da hierarquia. Enquanto os dados cabem na L1, a leitura é muito rápida. Quando passam a caber só na L2, na L3 ou só na memória principal, a taxa cai de degrau em degrau. É o efeito da **localidade temporal**.
- **Variando o stride**, a taxa cai conforme os acessos ficam mais espaçados, porque cada bloco trazido para a cache é menos aproveitado. É o efeito da **localidade espacial**.

O livro também mostra que a simples ordem dos laços em uma multiplicação de matrizes muda bastante o desempenho. As seis ordens possíveis (`ijk`, `ikj`, `kij` e assim por diante) fazem as mesmas operações, mas as que acessam as matrizes por linhas no laço interno são bem mais rápidas que as que acessam por colunas.

---

*Resumo do capítulo The Memory Hierarchy do livro Computer Systems: A Programmer's Perspective (CSAPP), de Randal E. Bryant e David R. O'Hallaron, feito pelo Claude*
