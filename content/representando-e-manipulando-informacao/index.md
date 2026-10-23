+++
date = '2026-10-22T20:34:51-03:00'
draft = false
title = 'Representando e Manipulando Informação'
type = 'blog'
tags = ['Arquitetura', 'Informação']
+++

Computadores guardam e processam tudo como bits, sinais de dois valores. Um bit sozinho não diz muito, mas um grupo de bits com uma interpretação pode representar qualquer coisa: números, letras, instruções. Existem três representações de números que importam mais:

- **Sem sinal (unsigned):** baseada na notação binária tradicional, representa números maiores ou iguais a zero.
- **Complemento de dois:** a forma mais comum de representar inteiros com sinal.
- **Ponto flutuante:** uma versão em base 2 da notação científica, usada para aproximar números reais.

Como os computadores usam um número limitado de bits, algumas operações dão resultados surpreendentes. Em uma máquina em que `int` tem 32 bits, `200 * 300 * 400 * 500` dá **−884.901.888**: o produto de números positivos ficou negativo. Já em ponto flutuante, `(3.14 + 1e20) - 1e20` dá **0.0**, enquanto `3.14 + (1e20 - 1e20)` dá **3.14**. A diferença vem de como cada representação lida com o limite de bits: inteiros representam uma faixa pequena de valores, mas de forma exata; ponto flutuante representa uma faixa enorme, mas só de forma aproximada.

Como diz o Patterson, **bits não têm significado próprio**. O mesmo padrão de bits pode ser um inteiro com sinal, um inteiro sem sinal, um número de ponto flutuante, um texto ou uma instrução. O que define o significado é a operação que é feita sobre ele.

## Armazenamento de informação

A menor unidade endereçável da memória é o **byte** (8 bits). Para o programa, a memória é um grande vetor de bytes, cada um identificado por um número único, o seu **endereço**.

### Notação hexadecimal

Escrever bits em binário é longo demais, e em decimal é difícil converter. Por isso se usa a base 16 (hexadecimal), com os dígitos 0 a 9 e A a F. Cada dígito hexadecimal corresponde a exatamente 4 bits, então a conversão é feita dígito a dígito. Em C, números hexadecimais começam com `0x`:

| Hexadecimal | 1    | 7    | 3    | A    | 4    | C    |
| ----------- | ---- | ---- | ---- | ---- | ---- | ---- |
| Binário     | 0001 | 0111 | 0011 | 1010 | 0100 | 1100 |

Ou seja, `0x173A4C` é `000101110011101001001100` em binário. Um byte vai de `0x00` a `0xFF`.

### Tamanho da palavra

Todo computador tem um **tamanho de palavra** (*word size*), que é o tamanho de um ponteiro. Ele define o tamanho máximo do espaço de endereçamento: com uma palavra de w bits, o programa consegue acessar até 2 elevado a w bytes. Com 32 bits, são 4 GB. Com 64 bits, são 16 exabytes. Por isso houve a migração para máquinas de 64 bits.

Vale notar que 2 elevado a 64 não é o dobro de 2 elevado a 32. Cada bit a mais dobra o número de endereços, então o dobro de 2 elevado a 32 é 2 elevado a 33, ou seja, 33 bits, que dão 8 GB. Os 32 bits extras dobram o espaço 32 vezes seguidas, o que dá cerca de 4 bilhões de vezes mais memória endereçável.

Os tipos de dados em C têm tamanhos típicos que dependem de como o programa é compilado:

| Tipo em C         | 32 bits | 64 bits |
| ----------------- | ------- | ------- |
| `char`            | 1       | 1       |
| `short`           | 2       | 2       |
| `int`             | 4       | 4       |
| `long`            | 4       | 8       |
| `char *` (ponteiro) | 4     | 8       |
| `float`           | 4       | 4       |
| `double`          | 8       | 8       |

Para não depender do tamanho "típico", o C99 criou tipos com tamanho fixo, como `int32_t` e `int64_t`.

### Ordem dos bytes: little endian e big endian

Um objeto com vários bytes é guardado em posições seguidas da memória, e o seu endereço é o do primeiro byte. Mas em que ordem os bytes ficam? Existem duas convenções. Para o `int` `0x01234567` guardado a partir do endereço `0x100`:

| Endereço      | 0x100 | 0x101 | 0x102 | 0x103 |
| ------------- | ----- | ----- | ----- | ----- |
| Big endian    | 01    | 23    | 45    | 67    |
| Little endian | 67    | 45    | 23    | 01    |

No **little endian**, o byte menos significativo vem primeiro. No **big endian**, o mais significativo. Os processadores Intel e a maioria dos ARM (Android e iOS) usam little endian. Os nomes vêm do livro *As Viagens de Gulliver*, em que dois povos guerreiam para decidir se um ovo deve ser quebrado pela ponta menor ou pela maior. Assim como no livro, não existe razão técnica para preferir uma ordem: basta escolher uma e ser consistente.

Para a maioria dos programas, a ordem é invisível. Ela aparece quando dados binários são enviados pela rede entre máquinas diferentes, quando se lê código de máquina (os bytes aparecem invertidos) e quando um programa contorna o sistema de tipos com casts.

### Strings e código

Uma string em C é um vetor de caracteres terminado pelo caractere nulo (valor 0). Cada caractere é codificado normalmente em **ASCII**, em que, por exemplo, os dígitos de 0 a 9 têm códigos `0x30` a `0x39`. Como o texto é guardado byte a byte, ele não depende da ordem dos bytes nem do tamanho da palavra: dados em texto são mais portáveis que dados binários.

O ASCII não tem caracteres como "ç" ou letras gregas, russas e chinesas. Para isso existe o **Unicode**, que cobre praticamente todas as línguas. A codificação mais usada é o **UTF-8**, criado por Ken Thompson e Rob Pike em 1992, que usa de 1 a 4 bytes por caractere e mantém os caracteres ASCII com o mesmo byte de sempre. Hoje quase todas as páginas da web usam UTF-8.

O código de um programa também é só uma sequência de bytes. A mesma função em C vira bytes completamente diferentes em processadores ou sistemas operacionais diferentes, por isso código binário raramente é portável.

### Álgebra booleana e operações com bits

Em 1850, George Boole criou uma álgebra para a lógica usando os valores 1 (verdadeiro) e 0 (falso). Em 1937, Claude Shannon mostrou que ela podia ser usada para projetar circuitos digitais. O C tem as quatro operações aplicadas bit a bit:

| Operação | Nome | Exemplo (`0110` e `1100`) |
| -------- | ---- | ------------------------- |
| `~`      | NOT  | `~1100` = `0011`          |
| `&`      | AND  | `0110 & 1100` = `0100`    |
| `\|`     | OR   | `0110 \| 1100` = `1110`   |
| `^`      | XOR  | `0110 ^ 1100` = `1010`    |

Um uso comum é o **mascaramento**: `x & 0xFF` mantém só o byte menos significativo de `x` e zera o resto.

Não confundir com os operadores **lógicos** `&&`, `||` e `!`. Eles tratam qualquer valor diferente de zero como verdadeiro e sempre retornam 0 ou 1. Além disso, não avaliam o segundo operando se o resultado já estiver decidido pelo primeiro: `p && *p` nunca acessa um ponteiro nulo.

### Deslocamentos

`x << k` desloca os bits de `x` k posições para a esquerda, preenchendo a direita com zeros. Para a direita, existem dois tipos:

- **Lógico:** preenche a esquerda com zeros.
- **Aritmético:** preenche a esquerda com cópias do bit mais significativo. Isso preserva o sinal de números negativos.

| Operação                | `01100011` | `10010101` |
| ----------------------- | ---------- | ---------- |
| `x << 4`                | `00110000` | `01010000` |
| `x >> 4` (lógico)       | `00000110` | `00001001` |
| `x >> 4` (aritmético)   | `00000110` | `11111001` |

Em C, deslocamentos de números sem sinal são sempre lógicos. Para números com sinal, o padrão não define, mas na prática quase todos os compiladores usam o aritmético.

## Representação de inteiros

### Inteiros sem sinal

Na representação sem sinal, o número é lido como um binário comum: cada bit vale uma potência de 2, começando por 1 no bit mais à direita e dobrando a cada posição (1, 2, 4, 8...). O valor é a soma das posições que têm bit 1. Por exemplo, `1011` vale 8 + 2 + 1 = 11.

Com w bits, o menor valor é 0 (todos os bits em 0) e o maior é o número com todos os bits em 1. Esse maior valor é chamado de **UMax**. Com 8 bits, por exemplo, vai de 0 a 255.

### Complemento de dois

Para representar números negativos, o complemento de dois muda só uma coisa: o bit mais significativo passa a ter **peso negativo**. Com 4 bits, os pesos são −8, 4, 2 e 1, em vez de 8, 4, 2 e 1. Esse bit também indica o sinal do número, por isso é chamado de **bit de sinal**: se ele for 1, soma −8 e o número fica negativo; se for 0, não soma nada e o número é positivo.

Assim, `1011` vale −8 + 2 + 1 = **−5**, e `1111` vale −8 + 4 + 2 + 1 = **−1**. Os números que começam com 0 continuam valendo o mesmo que na representação sem sinal.

**Como obter o negativo de um número:** basta **inverter todos os bits e somar 1**. Por exemplo, com 4 bits:

1. 5 é `0101`.
2. Invertendo os bits: `1010`.
3. Somando 1: `1011`, que é **−5**.

O caminho de volta é igual: invertendo `1011` fica `0100`, e somando 1 fica `0101`, que é 5 de novo. Em C, isso quer dizer que `-x` é igual a `~x + 1`.

Isso permite fazer **subtração usando soma**: `a - b` é o mesmo que `a + (-b)`, então basta calcular o complemento de dois do segundo operando e somar. Por exemplo, 7 − 6 com 4 bits:

1. 6 é `0110`. O complemento de dois é `1001` + 1 = `1010` (−6).
2. 7 + (−6): `0111` + `1010` = `10001`.
3. O quinto bit não cabe nos 4 bits e é descartado, sobrando `0001`, que é **1**.

Por isso o processador não precisa de um circuito separado para subtrair: a mesma unidade que soma faz a subtração, invertendo os bits do segundo operando e somando 1.

O menor valor (**TMin**) é o bit de sinal em 1 e o resto em 0. O maior (**TMax**) é o bit de sinal em 0 e o resto em 1:

| w  | TMin                       | TMax                      | UMax                       |
| -- | -------------------------- | ------------------------- | -------------------------- |
| 8  | −128                       | 127                       | 255                        |
| 16 | −32.768                    | 32.767                    | 65.535                     |
| 32 | −2.147.483.648             | 2.147.483.647             | 4.294.967.295              |
| 64 | −9.223.372.036.854.775.808 | 9.223.372.036.854.775.807 | 18.446.744.073.709.551.615 |

A faixa é assimétrica: existe um negativo a mais que positivos, porque o zero ocupa uma das combinações com bit de sinal 0. Por isso TMin não tem um positivo correspondente. Também vale notar que −1 tem a mesma representação que UMax: todos os bits iguais a 1.

Existiram outras formas de representar números com sinal, como **sinal e magnitude** (um bit só para o sinal) e **complemento de um** (só inverter os bits, sem somar 1). As duas têm dois zeros (+0 e −0) e exigem hardware mais complicado para somar. O complemento de dois ganhou porque deixa o hardware simples, e é a escolha unânime desde 1965. O sinal e magnitude sobreviveu no ponto flutuante.

### Conversão entre com e sem sinal

Em C, converter entre `int` e `unsigned` do mesmo tamanho **não muda os bits, só a interpretação**. Com 16 bits, −12.345 e 53.191 têm exatamente o mesmo padrão de bits (`0xCFC7`). Na prática, um número negativo convertido para sem sinal vira um número positivo muito grande, e um número sem sinal maior que TMax convertido para com sinal vira negativo.

O problema é que o C faz essa conversão **implicitamente**: quando uma operação mistura um valor com sinal e outro sem sinal, o com sinal é convertido para sem sinal. Isso gera resultados nada intuitivos:

| Expressão (32 bits)             | Resultado      |
| ------------------------------- | -------------- |
| `-1 < 0`                        | 1 (verdadeiro) |
| `-1 < 0U`                       | **0 (falso)**  |
| `2147483647U > -2147483647-1`   | **0 (falso)**  |

Em `-1 < 0U`, o −1 vira 4.294.967.295, que obviamente não é menor que 0.

### Extensão de sinal

Quando um número é convertido para um tipo **maior** (de `char` para `int`, por exemplo), os bits novos à esquerda precisam ser preenchidos de forma que o valor continue o mesmo. A regra depende do tipo:

- **Sem sinal:** os bits novos são preenchidos com **zeros** (*zero extension*).
- **Com sinal:** os bits novos são preenchidos com **cópias do bit de sinal** (*sign extension*).
  - Se o número é **positivo**, o bit de sinal é 0, então entram zeros: 5 em 8 bits é `00000101`, e em 16 bits vira `00000000 00000101`.
  - Se o número é **negativo**, o bit de sinal é 1, então entram uns: −5 em 8 bits é `11111011`, e em 16 bits vira `11111111 11111011`, que continua valendo −5.

Se os bits novos de um negativo fossem preenchidos com zeros, `11111011` viraria `00000000 11111011`, que é 251, e não −5.

O processador tem instruções separadas para cada caso. No x86-64, na sintaxe Intel, são a **`movzx`** (*move with zero extension*) e a **`movsx`** (*move with sign extension*). Na sintaxe AT&T, usada pelo GCC e pelo livro, elas aparecem como `movz` e `movs` seguidas de duas letras, que indicam o tamanho de origem e de destino (`b` = byte, `w` = 2 bytes, `l` = 4 bytes, `q` = 8 bytes):

| Instrução (AT&T) | Equivalente Intel | O que faz                                        |
| ---------------- | ----------------- | ------------------------------------------------ |
| `movzbl`         | `movzx`           | Copia 1 byte para 4 bytes, preenchendo com zeros |
| `movzwl`         | `movzx`           | Copia 2 bytes para 4 bytes, preenchendo com zeros |
| `movsbl`         | `movsx`           | Copia 1 byte para 4 bytes, estendendo o sinal    |
| `movswl`         | `movsx`           | Copia 2 bytes para 4 bytes, estendendo o sinal   |
| `movslq`         | `movsxd`          | Copia 4 bytes para 8 bytes, estendendo o sinal   |

Por exemplo, para converter um `char` para `int`, o compilador gera `movsbl`. Para converter um `unsigned char` para `unsigned`, gera `movzbl`. Não existe `movzlq`: no x86-64, qualquer instrução que escreve em um registrador de 4 bytes já zera os 4 bytes de cima, então um `movl` comum faz a extensão com zeros de 4 para 8 bytes.

### Truncamento

Para converter para um tipo **menor**, os bits mais significativos são simplesmente descartados. Isso pode mudar o valor, porque o bit que sobra na frente passa a ser o novo bit de sinal: o `int` 53.191 convertido para `short` vira −12.345.

### Cuidado com unsigned

Muitos bugs vêm dessas conversões implícitas. Um exemplo do livro:

```c
/* ATENÇÃO: código com bug */
float sum_elements(float a[], unsigned length) {
    int i;
    float result = 0;
    for (i = 0; i <= length - 1; i++)
        result += a[i];
    return result;
}
```

Se `length` for 0, `length - 1` é calculado como sem sinal e vira UMax, o maior número sem sinal. O laço tenta ler o vetor inteiro da memória e o programa quebra. Outro caso clássico é `strlen(s) - strlen(t) > 0`: como `strlen` retorna `size_t` (sem sinal), a subtração nunca é negativa.

Por isso, linguagens como Java só têm inteiros com sinal. Mas números sem sinal são úteis quando a palavra é só um conjunto de bits sem valor numérico, como flags, máscaras e endereços.

## Aritmética de inteiros

### Adição e overflow

Somar dois números de w bits pode precisar de w + 1 bits. Como o tamanho é fixo, o bit extra é descartado. Quando o resultado verdadeiro não cabe nos bits disponíveis, ocorre **overflow**:

- **Sem sinal:** a soma "dá a volta". Com 4 bits, 9 + 12 = 21, mas só cabem valores até 15, então o resultado é 21 − 16 = **5**. Dá para perceber o overflow porque o resultado ficou menor que os operandos.
- **Complemento de dois:** se a soma passar de TMax, ocorre **overflow positivo** e o resultado fica negativo. Se ficar abaixo de TMin, ocorre **overflow negativo** e o resultado fica positivo. Com 4 bits, 5 + 5 = 10, mas o resultado é `1010`, que vale **−6**.

Uma vantagem do complemento de dois é que a soma com sinal e a soma sem sinal produzem exatamente os mesmos bits, então o processador usa a mesma instrução para as duas. Em C, o overflow não gera erro: o programa simplesmente continua com o valor errado.

### A regra do overflow

O Stallings resume como detectar overflow no complemento de dois em uma regra:

> **Regra do overflow:** se dois números com o mesmo sinal (os dois positivos ou os dois negativos) são somados, ocorre overflow se, e somente se, o resultado tiver o sinal oposto.

Ou seja:

- **Positivo + positivo** dando **negativo**: overflow.
- **Negativo + negativo** dando **positivo**: overflow.
- **Positivo + negativo:** **nunca** dá overflow, porque o resultado sempre fica entre os dois operandos.

A mesma regra vale para a subtração, já que `a - b` é calculado como `a + (-b)`. Por exemplo, subtrair um negativo de um positivo vira uma soma de dois positivos, que pode dar overflow.

### Flags

Depois de cada operação aritmética, o processador atualiza alguns bits especiais, as **flags** (ou *condition codes*), que descrevem o resultado. Os desvios condicionais (`if`, laços) usam essas flags para decidir o caminho. No x86-64, as principais são:

| Flag | Nome          | Quando é ligada                                                          |
| ---- | ------------- | ------------------------------------------------------------------------ |
| CF   | Carry flag    | Houve "vai um" para fora do bit mais significativo. Indica overflow em números **sem sinal**. |
| OF   | Overflow flag | Houve overflow em complemento de dois, positivo ou negativo. É a regra do Stallings. |
| SF   | Sign flag     | O resultado é negativo (o bit mais significativo é 1).                   |
| ZF   | Zero flag     | O resultado é zero.                                                      |

O processador não sabe se os números são com ou sem sinal, por isso atualiza as duas flags de overflow, e o programa usa a que fizer sentido. Exemplos com 4 bits:

| Operação          | Bits                     | Resultado | CF | OF | SF | ZF |
| ----------------- | ------------------------ | --------- | -- | -- | -- | -- |
| 5 + 5             | `0101` + `0101` = `1010` | −6        | 0  | 1  | 1  | 0  |
| −8 + −5           | `1000` + `1011` = `1 0011` | 3       | 1  | 1  | 0  | 0  |
| −1 + 1            | `1111` + `0001` = `1 0000` | 0       | 1  | 0  | 0  | 1  |
| 2 + 3             | `0010` + `0011` = `0101` | 5         | 0  | 0  | 0  | 0  |

O terceiro exemplo mostra que **carry não é o mesmo que overflow**: houve "vai um" para fora (CF = 1), mas −1 + 1 = 0 está correto em complemento de dois, então OF = 0. Como lembra o Stallings, o overflow pode acontecer com ou sem carry.

### Negação

Como visto no complemento de dois, a negação é feita invertendo todos os bits e somando 1. A exceção é TMin: com 4 bits, −8 é `1000`, invertendo fica `0111`, e somando 1 volta a ser `1000`. O negativo de TMin é o próprio TMin, porque +8 não cabe em 4 bits com sinal.

### Multiplicação

O produto de dois números de w bits pode precisar de 2w bits. Em C, o resultado é truncado para w bits, tanto com sinal quanto sem sinal, e os bits do resultado são os mesmos nos dois casos. Isso também pode virar uma falha de segurança: em 2002, uma biblioteca da Sun (XDR) calculava o tamanho de um buffer como `quantidade * tamanho`, e com valores escolhidos o produto dava overflow. O programa alocava um buffer pequeno e depois copiava muito mais dados para ele.

### Multiplicação e divisão por constantes

Multiplicação de inteiros é mais lenta que soma e deslocamento (3 ciclos contra 1 no processador usado pelo livro), e divisão é ainda pior (30 ciclos ou mais). Por isso, o compilador troca essas operações por deslocamentos sempre que pode:

- **Multiplicar por uma potência de 2** (2, 4, 8, 16...) é o mesmo que deslocar para a esquerda: cada posição deslocada multiplica por 2, então `x << 3` é `x * 8`. Para outras constantes, o compilador combina deslocamentos e somas. Como 14 = 16 − 2, `x * 14` pode virar `(x << 4) - (x << 1)`.
- **Dividir por uma potência de 2** é feito com deslocamento para a direita: cada posição divide por 2. Usa-se o deslocamento lógico para sem sinal e o aritmético para com sinal.

Mas o deslocamento para a direita arredonda **para baixo**, e a divisão inteira deve arredondar **em direção ao zero**. Para positivos dá no mesmo, mas para negativos não: −12.340 >> 4 dá −772, e o correto seria −771. A correção é somar um valor de ajuste (*bias*) antes de deslocar, quando o número é negativo. Esse valor é o divisor menos 1: para dividir por 16, soma-se 15.

```c
(x < 0 ? x + (1 << k) - 1 : x) >> k
```

O Patterson lista como falácia justamente achar que deslocar para a direita é sempre igual a dividir por uma potência de 2.

## Ponto flutuante

### Números binários fracionários

Assim como no decimal, os bits depois da vírgula binária valem metade, um quarto, um oitavo, e assim por diante. Por exemplo, `101.11` vale 4 + 1 + ½ + ¼ = **5,75**.

Isso significa que só frações cujo denominador é uma potência de 2 (como ½, ¾ ou 5/8) podem ser representadas exatamente. Um número simples como 0,1 (ou 0,2) vira uma dízima infinita em binário (`0.000110011[0011]...`) e só pode ser aproximado.

### O padrão IEEE 754

Até os anos 80, cada fabricante tinha o seu próprio formato de ponto flutuante, e a precisão nem sempre era a prioridade. Em 1976, a Intel contratou William Kahan, professor de Berkeley, para projetar o ponto flutuante do coprocessador 8087. O trabalho virou a base do **IEEE 754**, publicado em 1985 e revisado em 2008 e 2019. Hoje praticamente todo computador usa esse padrão.

A ideia é a mesma da notação científica, só que em base 2. Na notação científica, 1.500 é escrito como 1,5 × 10³. No IEEE 754, um número é guardado em três partes:

- **Sinal:** se o número é positivo ou negativo.
- **Significando:** um número fracionário entre 1 e 2 (ou entre 0 e 1 nos denormalizados), que guarda os dígitos do número.
- **Expoente:** quantas vezes o significando é multiplicado (ou dividido) por 2.

Por exemplo, 5,0 é guardado como 1,25 vezes 4 (ou seja, 1,25 × 2²), e −0,75 como −1,5 vezes ½.

Os bits são divididos em três campos:

| Formato                       | Sinal | Expoente (`exp`) | Fração (`frac`) | Bias |
| ----------------------------- | ----- | ---------------- | --------------- | ---- |
| Precisão simples (`float`)    | 1 bit | 8 bits           | 23 bits         | 127  |
| Precisão dupla (`double`)     | 1 bit | 11 bits          | 52 bits         | 1023 |

O tamanho dos campos é um compromisso: mais bits na fração aumentam a **precisão**, e mais bits no expoente aumentam a **faixa** de valores.

O expoente é guardado com um *bias* (127 no `float`): o valor real é `exp − 127`. Assim, o expoente não precisa de bit de sinal, e números de ponto flutuante positivos podem ser comparados como se fossem inteiros. O sinal fica no bit mais significativo e o expoente vem antes da fração exatamente por isso.

### Normalizados, denormalizados e valores especiais

O valor do campo `exp` define como o número é interpretado:

| `exp`                 | `frac`       | Significado                |
| --------------------- | ------------ | -------------------------- |
| Nem só zeros, nem só uns | qualquer  | Número normalizado         |
| Só zeros              | qualquer     | Número denormalizado (ou zero) |
| Só uns                | zero         | Infinito (+∞ ou −∞)        |
| Só uns                | diferente de zero | NaN (*Not a Number*)  |

- **Normalizados:** é o caso comum. O significando é 1 mais a fração guardada no campo `frac`. Como o primeiro bit é sempre 1, ele não é guardado: é o "1 implícito", que dá um bit de precisão de graça.
- **Denormalizados:** quando `exp` é só zeros, o significando é só a fração, sem o 1 implícito. Servem para representar o **zero** (que tem +0 e −0) e números muito próximos de zero, espaçados de forma uniforme (*gradual underflow*).
- **Infinito:** representa resultados que estouraram, como `1.0 / 0.0`.
- **NaN:** representa resultados que não são números reais, como a raiz quadrada de −1 ou ∞ − ∞.

Os números representáveis não ficam distribuídos de forma uniforme: eles são mais densos perto do zero e cada vez mais espaçados conforme crescem.

### Exemplo: 12.345 como float

O inteiro 12.345 é `11000000111001` em binário. Normalizando, a vírgula anda 13 casas para a esquerda, e o número fica 1,1000000111001 (em binário) vezes 2 elevado a 13. Para montar o `float`:

1. **Fração:** tira-se o 1 implícito e completa-se com zeros até 23 bits: `10000001110010000000000`.
2. **Expoente:** 13 + 127 = 140, que é `10001100`.
3. **Sinal:** 0.

O resultado é `0x4640E400`. Comparando com o inteiro `0x00003039`, os bits do inteiro depois do primeiro 1 aparecem iguais no começo da fração do float.

### Arredondamento

Como o ponto flutuante só aproxima os números reais, é preciso arredondar. O IEEE 754 define vários modos:

| Modo                           | 1,40 | 1,60 | 1,50 | 2,50 | −1,50 |
| ------------------------------ | ---- | ---- | ---- | ---- | ----- |
| Para o par mais próximo (padrão) | 1  | 2    | 2    | 2    | −2    |
| Em direção ao zero             | 1    | 1    | 1    | 2    | −1    |
| Para baixo                     | 1    | 1    | 1    | 2    | −2    |
| Para cima                      | 2    | 2    | 2    | 3    | −1    |

O modo padrão arredonda para o mais próximo e, no empate, escolhe o valor **par**. Se o empate fosse sempre arredondado para cima, a média de muitos valores arredondados ficaria um pouco maior que a real. Arredondando para o par, metade dos empates sobe e metade desce, e o viés some.

Para arredondar corretamente, o hardware guarda bits extras durante os cálculos (os bits *guard*, *round* e *sticky*). Com eles, o IEEE 754 garante que o resultado fica a no máximo meia unidade na última casa (*ulp*) do valor exato.

### Operações em ponto flutuante

O resultado de uma operação é o resultado exato arredondado. Isso faz o ponto flutuante perder propriedades da matemática:

- **Não é associativo:** `(3.14 + 1e10) - 1e10` dá 0.0, porque o 3.14 se perde no arredondamento, enquanto `3.14 + (1e10 - 1e10)` dá 3.14. Na multiplicação, `(1e20 * 1e20) * 1e-20` dá +∞, e `1e20 * (1e20 * 1e-20)` dá 1e20.
- **Não é distributivo:** `1e20 * (1e20 - 1e20)` dá 0.0, e `1e20 * 1e20 - 1e20 * 1e20` dá NaN.

Por causa disso, o compilador evita otimizações que mudem a ordem das operações de ponto flutuante, mesmo as que parecem inofensivas. O Patterson lembra outra consequência: somar um milhão de números em paralelo pode dar um resultado ligeiramente diferente da soma sequencial, e até variar de uma execução para outra, se o número de processadores usados mudar. Em compensação, o ponto flutuante mantém a **monotonicidade**: se a ≥ b, então x + a ≥ x + b. Isso não vale para inteiros, por causa do overflow.

### Ponto flutuante em C

Em máquinas com IEEE 754, `float` é a precisão simples e `double` é a precisão dupla. Nas conversões entre tipos:

- **`int` → `float`:** não dá overflow, mas pode arredondar (o `float` tem só 24 bits de precisão).
- **`int` ou `float` → `double`:** o valor é preservado exatamente.
- **`double` → `float`:** pode dar overflow para ±∞ ou arredondar.
- **`float` ou `double` → `int`:** o valor é arredondado em direção ao zero (1,999 vira 1, e −1,999 vira −1), e pode dar overflow.

## Quando a aritmética dá errado

Os livros mostram que esses detalhes têm consequências reais:

- **Míssil Patriot (1991):** durante a Guerra do Golfo, uma bateria de mísseis Patriot falhou em interceptar um míssil Scud, que matou 28 soldados americanos. O relógio do sistema contava em décimos de segundo, e 0,1 era aproximado com 23 bits. Depois de 100 horas ligado, o erro acumulado chegou a cerca de 0,34 segundo, o suficiente para errar a posição do Scud em quase 700 metros.
- **Ariane 5 (1996):** 37 segundos depois do lançamento, o foguete saiu da rota e explodiu. O software, reaproveitado do Ariane 4, convertia a velocidade horizontal de um número de ponto flutuante de 64 bits para um inteiro de 16 bits com sinal. O Ariane 5 era mais rápido, a conversão deu overflow, e o sistema de navegação enviou dados inválidos para o controle dos motores. Havia satélites de 500 milhões de dólares a bordo.
- **Bug do Pentium (1994):** a divisão de ponto flutuante do Pentium usava uma tabela de consulta com cinco entradas erradas, e alguns resultados saíam errados a partir da quarta casa decimal. A Intel primeiro disse que só matemáticos seriam afetados, mas acabou trocando os processadores, com um custo de centenas de milhões de dólares.

## Referências

- BRYANT, Randal E.; O'HALLARON, David R. _Computer Systems: A Programmer's Perspective_. 3. ed. Pearson. Capítulo 2.
- PATTERSON, David A.; HENNESSY, John L. _Computer Organization and Design: The Hardware/Software Interface_. 5. ed. Morgan Kaufmann. Seções 2.4, 2.9 e capítulo 3.
- STALLINGS, William. _Computer Organization and Architecture_. 11. ed. Pearson. Capítulo 11.
- Wikipedia. [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754).
- Wikipedia. [UTF-8](https://en.wikipedia.org/wiki/UTF-8).

---

*Resumo do capítulo Representing and Manipulating Information do livro Computer Systems: A Programmer's Perspective (CSAPP), complementado com o Patterson e a Wikipedia, feito pelo Claude*
