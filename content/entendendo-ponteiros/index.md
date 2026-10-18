+++
date = '2026-10-18T17:02:29-03:00'
draft = false
title = 'Entendendo ponteiros'
type = 'blog'
tags = ['Arquitetura', 'Assembly', 'Memória', 'CSAPP']
+++

Ponteiros são uma característica central da linguagem C. Eles servem como uma forma uniforme de gerar referências a elementos dentro de diferentes estruturas de dados. Ponteiros são uma fonte de confusão para programadores iniciantes, mas os conceitos por trás deles são bem simples. Aqui destacamos alguns princípios importantes dos ponteiros e como eles são mapeados para código de máquina.

- *Todo ponteiro tem um tipo associado.* Esse tipo indica para que tipo de objeto o ponteiro aponta. Usando as seguintes declarações de ponteiros como exemplo:

  ```c
  int *ip;
  char **cpp;
  ```

  a variável `ip` é um ponteiro para um objeto do tipo `int`, enquanto `cpp` é um ponteiro para um objeto que, por sua vez, é um ponteiro para um objeto do tipo `char`. *Em geral, se o objeto tem tipo `T`, então o ponteiro tem tipo `*T`.* O tipo especial `void *` representa um ponteiro genérico. Por exemplo, a função `malloc` retorna um ponteiro genérico, que é convertido em um ponteiro com tipo por meio de um cast explícito ou pelo cast implícito da operação de atribuição. *Tipos de ponteiros não fazem parte do código de máquina; eles são uma abstração fornecida pelo C* para ajudar programadores a evitar erros de endereçamento.

- *Todo ponteiro tem um valor.* Esse valor é o endereço de algum objeto do tipo designado. O valor especial `NULL` (0) indica que o ponteiro não aponta para lugar nenhum.

- *Ponteiros são criados com o operador `&`.* Esse operador pode ser aplicado a qualquer expressão C classificada como *lvalue*, ou seja, uma expressão que pode aparecer do lado esquerdo de uma atribuição. Exemplos incluem variáveis e os elementos de structs, unions e arrays. Já vimos que a implementação em código de máquina do operador `&` costuma usar a instrução `leaq` para calcular o valor da expressão, já que essa instrução foi projetada para calcular o endereço de uma referência à memória.

- *Ponteiros são desreferenciados com o operador `*`.* O resultado é um valor com o tipo associado ao ponteiro. A desreferência é implementada por uma referência à memória, seja armazenando no endereço especificado, seja lendo dele.

- *Arrays e ponteiros são intimamente relacionados.* O nome de um array pode ser referenciado (mas não atualizado) como se fosse uma variável ponteiro. Indexar um array (por exemplo, `a[3]`) tem exatamente o mesmo efeito que aritmética de ponteiros seguida de desreferência (por exemplo, `*(a+3)`). Tanto a indexação de arrays quanto a aritmética de ponteiros exigem escalar os deslocamentos pelo tamanho do objeto. Quando escrevemos uma expressão `p+i` para um ponteiro `p` com valor `p`, o endereço resultante é calculado como `p + L · i`, onde `L` é o tamanho do tipo de dado associado a `p`.

- *Fazer cast de um tipo de ponteiro para outro muda o seu tipo, mas não o seu valor.* Um efeito do cast é mudar a escala da aritmética de ponteiros. Por exemplo, se `p` é um ponteiro do tipo `char *` com valor `p`, então a expressão `(int *) p+7` calcula `p + 28`, enquanto `(int *) (p+7)` calcula `p + 7`. (Lembre que o cast tem precedência maior que a soma.)

- *Ponteiros também podem apontar para funções.* Isso oferece uma capacidade poderosa de armazenar e passar referências para código, que pode ser invocado em alguma outra parte do programa. Por exemplo, se temos uma função definida pelo protótipo

  ```c
  int fun(int x, int *p);
  ```

  então podemos declarar um ponteiro `fp` e atribuir essa função a ele com a seguinte sequência de código:

  ```c
  int (*fp)(int, int *);
  fp = fun;
  ```

  Podemos então invocar a função usando esse ponteiro:

  ```c
  int y = 1;
  int result = fp(3, &y);
  ```

  O valor de um ponteiro para função é o endereço da primeira instrução na representação em código de máquina da função.

---

*Trecho extraído do livro CSAPP e traduzido pelo Claude*
