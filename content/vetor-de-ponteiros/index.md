+++
date = '2026-10-27T16:00:49-03:00'
draft = false
title = 'Vetor de ponteiros'
type = 'blog'
tags = ['C', 'Memória']
+++

Um **vetor de ponteiros** é um vetor comum cujos elementos, em vez de números ou caracteres, são **endereços de memória**. Cada posição do vetor não guarda o dado em si: guarda o lugar onde o dado está.

Essa estrutura já apareceu várias vezes nos posts anteriores sem ganhar um nome próprio. O `argv` e o `envp` da `main`, o `environ` e os argumentos do `execve` são todos vetores de ponteiros para strings. Este post explica como eles funcionam: como o tipo é lido, como ficam na memória, por que são tão usados para guardar strings e quais cuidados exigem.

## Lendo a declaração

Uma declaração como `int *v[3]` diz que `v` é um **vetor de 3 elementos, e cada elemento é um ponteiro para `int`**. A forma de ler é começar pelo nome e seguir a precedência dos operadores. Os colchetes têm precedência maior que o asterisco, então `v` é primeiro um vetor, e o que sobra (`int *`) é o tipo de cada elemento.

Com parênteses, a leitura se inverte. Em `int (*p)[3]`, o asterisco é aplicado primeiro, e `p` passa a ser **um único ponteiro**, que aponta para um vetor de 3 inteiros. O primeiro ocupa 24 bytes no x86-64 (três endereços de 8 bytes). O segundo ocupa só 8, porque é um endereço só. A diferença parece um detalhe de sintaxe, mas são estruturas completamente diferentes.

## Como fica na memória

Na memória, um vetor de ponteiros é guardado como qualquer outro vetor: os elementos ficam **lado a lado**, em endereços consecutivos. Como cada elemento é um endereço de 8 bytes, as posições do vetor avançam de 8 em 8.

O que muda é que **os dados apontados podem estar em qualquer lugar**: na pilha, no heap, na área de dados do programa. O vetor é contíguo, mas os dados não precisam ser. Essa separação entre "onde está a lista" e "onde estão os itens" é a ideia central da estrutura.

Ela também tem um custo. Para ler o dado da posição `i`, o processador faz **dois acessos à memória**: primeiro lê o elemento `v[i]`, que é um endereço, e depois vai até esse endereço buscar o valor. Num vetor comum, um único acesso basta. Esse nível extra de indireção é o preço da flexibilidade.

Pela mesma razão, escrever através de um elemento, como em `*v[1] = 99`, não altera o vetor. A escrita segue o ponteiro e modifica a variável apontada. O vetor continua guardando o mesmo endereço.

## Vetor de strings

O uso mais comum de um vetor de ponteiros é guardar uma lista de strings. Em C, uma string é um vetor de `char` terminado em `'\0'`, e um `char *` aponta para o primeiro caractere dela. Um vetor de `char *` é, então, uma lista de strings.

Existe outra forma de guardar uma lista de nomes: uma **matriz de caracteres**, como `char m[3][6]`. No uso, as duas parecem iguais, porque as duas aceitam a sintaxe `nomes[i][j]`. Mas a organização na memória é bem diferente:

- No **vetor de ponteiros**, o vetor guarda só os endereços. Cada string fica em outro lugar e ocupa exatamente o seu tamanho. Uma lista com "Ana", "Bruno" e "Carla" ocupa 24 bytes de ponteiros, e as strings ficam guardadas uma depois da outra, sem desperdício.
- Na **matriz**, os caracteres ficam dentro da própria matriz, em linhas de tamanho fixo. Toda linha tem o tamanho da maior string, então "Ana" ocupa uma linha de 6 bytes e usa só 4.

O acesso a um caractere também é diferente. Na matriz, o compilador calcula o endereço com uma conta (linha vezes tamanho da linha, mais a coluna) e faz uma só leitura. No vetor de ponteiros, ele primeiro lê o ponteiro da linha e só depois chega ao caractere.

| | Vetor de ponteiros | Matriz de caracteres |
|---|---|---|
| Tamanho das strings | Cada uma do seu tamanho | Todas do tamanho da maior |
| Onde ficam os caracteres | Em qualquer lugar | Dentro da própria matriz |
| Trocar duas strings de lugar | Barato: troca dois endereços | Caro: copia as linhas inteiras |
| Acesso a um caractere | Duas leituras | Uma leitura |

### Strings literais

Quando um vetor de ponteiros é inicializado com strings entre aspas, os ponteiros apontam para **strings literais**. Os literais ficam numa área de memória marcada como **somente leitura**. Trocar um elemento do vetor para apontar para outra string é permitido, porque o vetor pertence ao programa. Mas tentar alterar um caractere do literal faz o processador gerar uma falha de página, e o kernel encerra o programa com segmentation fault.

Na matriz isso não acontece, porque os caracteres são uma cópia que pertence a ela. Por isso, quando um vetor de ponteiros aponta para literais, o costume é declará-lo como `const char *`. Assim, uma tentativa de escrita vira erro de compilação, e não uma falha durante a execução.

## Passando para uma função

Em quase toda expressão, o nome de um vetor **decai** para um ponteiro para o seu primeiro elemento. Num vetor de `char *`, o primeiro elemento é um `char *`, então o ponteiro para ele é um **`char **`**, um ponteiro para ponteiro. É por isso que, como parâmetro de função, `char *argv[]` e `char **argv` significam exatamente a mesma coisa.

A consequência é que **a função recebe só um endereço e perde o tamanho do vetor**. O `sizeof`, que fora da função mediria o vetor inteiro, dentro dela mede só o ponteiro: 8 bytes. A função precisa de outra forma de saber onde a lista acaba, e há duas convenções:

1. **Passar o tamanho junto**, num argumento separado. É o papel do `argc` na `main`.
2. **Terminar o vetor com `NULL`**, um ponteiro que não aponta para lugar nenhum e funciona como marcador de fim. Quem recebe percorre os elementos até encontrar o `NULL`.

A segunda convenção é a usada pelo `argv`, pelo `envp`, pelo `environ` e pelo `execve`. Ela é a mesma ideia do `'\0'` no fim de uma string, só que aplicada a uma lista de ponteiros.

## Alocação dinâmica

Quando a quantidade de strings só é conhecida durante a execução (linhas lidas de um arquivo, por exemplo), o vetor é criado com `malloc`. Nesse caso existem **dois níveis de alocação**: um bloco para o vetor de ponteiros e um bloco separado para cada string.

Essa separação traz algumas consequências:

- **O vetor pode crescer sem mover as strings.** Se a lista enche e é aumentada com `realloc`, o bloco de ponteiros pode mudar de endereço, mas as strings continuam onde estavam. Só os endereços delas são copiados.
- **Cada string precisa da sua própria memória.** Se o programa lê todas as linhas no mesmo buffer e guarda o endereço desse buffer no vetor, todos os elementos apontam para o mesmo lugar, e a lista inteira mostra só a última linha lida. Por isso cada string é copiada para um bloco próprio, geralmente com `strdup`.
- **A liberação segue a ordem inversa da alocação.** Primeiro se libera cada string, depois o vetor. Se o vetor for liberado antes, os endereços das strings se perdem junto com ele, e a memória delas nunca mais pode ser devolvida.

## Ordenar trocando ponteiros

Como cada elemento é só um endereço, reorganizar a lista é barato. Para ordenar um vetor de strings, basta trocar ponteiros de 8 bytes de lugar, e os caracteres nunca são copiados. Depois da ordenação, cada string continua no mesmo endereço de antes. O que mudou foi a posição do seu ponteiro dentro do vetor.

O `qsort` da biblioteca padrão ordena qualquer tipo de vetor, e por isso não sabe o que são os elementos. Ele passa para a função de comparação **ponteiros para os elementos**. Num vetor de `char *`, isso significa ponteiros para ponteiros. A função de comparação precisa desfazer esse nível extra antes de comparar as strings. Se ela comparar os ponteiros recebidos diretamente, vai comparar os bytes dos endereços, e não os nomes. O código compila sem aviso, e a lista simplesmente sai fora de ordem.

## Vetor de ponteiros para funções

Os elementos também podem ser **endereços de funções**. Um vetor desse tipo permite escolher qual função chamar por um índice, em vez de uma sequência de `if` ou `switch`. É uma tabela: a posição `i` guarda o endereço do código que trata o caso `i`.

O próprio sistema usa essa ideia. A **tabela de exceções**, vista em [Exceptional Control Flow](/exceptional-control-flow/), é um vetor de endereços de tratadores indexado pelo número da exceção. Quando uma exceção acontece, o processador consulta a posição correspondente e salta para o endereço que está nela.

## Resumo

- **Um vetor de ponteiros guarda endereços.** Os elementos ficam lado a lado, mas os dados apontados podem estar em qualquer lugar, e cada leitura de um dado faz dois acessos à memória.
- **A declaração se lê pela precedência.** `int *v[3]` é um vetor de ponteiros; `int (*p)[3]` é um ponteiro para um vetor.
- **Para strings, ele economiza espaço e facilita reorganizar a lista** em relação a uma matriz, mas os literais apontados não podem ser alterados.
- **Numa função, o vetor vira `char **` e perde o tamanho.** Por isso existem o `argc` e a convenção de terminar com `NULL`.
- **Na alocação dinâmica, há dois níveis**, o vetor e cada string, liberados na ordem inversa.
- **Com ponteiros para funções**, o vetor vira uma tabela de despacho, como a tabela de exceções do processador.

---

*Escrito pelo Claude*
