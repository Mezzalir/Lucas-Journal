+++
date = '2026-10-28T12:00:00-03:00'
draft = false
title = 'Unix e o projeto GNU/Linux'
type = 'blog'
tags = ['História da computação', 'Linux', 'Sistemas Operacionais']
+++

Quase todo sistema operacional que roda hoje em servidores, celulares e supercomputadores descende, no código ou nas ideias, de um sistema escrito por dois programadores da Bell Labs em 1969. O Unix virou família, foi disputado por empresas, foi padronizado e acabou reimplementado do zero por um estudante finlandês, que usou as ferramentas de um projeto de software livre começado anos antes. Este post conta essa história: o Unix, as variantes e a guerra entre elas, o padrão POSIX, o projeto GNU e o Linux.

## Antes do Unix: o Multics

Em 1964, o MIT, a General Electric e a Bell Labs (o laboratório de pesquisa da AT&T) começaram o **Multics**, um sistema de tempo compartilhado (*time-sharing*) que pretendia ser uma espécie de "serviço público" de computação: muitos usuários ao mesmo tempo, conectados a uma única máquina grande, com segurança, memória virtual e um sistema de arquivos hierárquico.

As ideias eram boas, mas o projeto era grande demais. Em 1969, a Bell Labs concluiu que o Multics não ia entregar o que prometia a tempo e saiu do projeto. Dois dos pesquisadores que trabalhavam nele, **Ken Thompson** e **Dennis Ritchie**, ficaram sem o ambiente de programação ao qual tinham se acostumado e decidiram fazer algo menor.

## O nascimento do Unix

No verão de 1969, Thompson escreveu, em assembly, um sistema operacional simples para um **PDP-7** da DEC que estava encostado no laboratório. Parte da motivação era rodar um jogo que ele tinha escrito, o *Space Travel*. O sistema tinha um kernel, um shell, um editor e um assembler, e herdava do Multics as ideias que valiam a pena sem a complexidade: o sistema de arquivos em árvore, o interpretador de comandos (o shell) como um programa separado e a noção de arquivo como uma sequência de bytes sem estrutura.

O nome veio de um trocadilho atribuído a Brian Kernighan: se o Multics era "multi", aquele sistema era "uni", **Unics**, que depois virou **Unix**.

Em 1970 e 1971 o sistema foi levado para o **PDP-11**, e a Bell Labs justificou a compra da máquina com um uso prático: processamento de texto para o departamento de patentes. A primeira edição do manual do Unix é de novembro de 1971.

### C e a portabilidade

O detalhe que mudou tudo foi a linguagem. Thompson criou a linguagem **B**, derivada do BCPL, e Ritchie a evoluiu para **C** entre 1972 e 1973, acrescentando tipos e estruturas. Em 1973, na Version 4, **o kernel do Unix foi reescrito quase inteiro em C**.

Na época, sistemas operacionais eram escritos em assembly, presos à máquina para a qual foram feitos. Um sistema escrito em linguagem de alto nível podia ser levado para outro hardware reescrevendo só a parte pequena que dependia da arquitetura e recompilando o resto. Essa portabilidade é o motivo de o Unix ter se espalhado por dezenas de computadores diferentes nas décadas seguintes, e de C ter se tornado a linguagem de sistemas por excelência.

### A filosofia Unix

Junto com o código, o Unix espalhou um jeito de construir software. Em 1973, por insistência de **Doug McIlroy**, entrou no sistema o **pipe** (`|`), que liga a saída de um programa à entrada de outro. Com ele, a forma natural de resolver um problema passou a ser combinar programas pequenos:

```bash
cat acessos.log | grep "404" | sort | uniq -c | sort -rn | head
```

Em 1994, o historiador Peter Salus resumiu as ideias de McIlroy em três frases: **escreva programas que façam uma coisa e a façam bem; escreva programas que trabalhem juntos; escreva programas que lidem com fluxos de texto, porque essa é uma interface universal**. Outra ideia central é a de que **tudo é arquivo**: dispositivos, terminais e, depois, até conexões de rede são acessados com as mesmas chamadas `open`, `read`, `write` e `close`.

## A AT&T, as universidades e Berkeley

Em 1956, a AT&T tinha assinado um acordo antitruste com o governo americano (o *consent decree*) que a proibia de entrar em mercados fora da telefonia, incluindo o de computadores. Ela não podia vender o Unix como produto. Depois que Thompson e Ritchie apresentaram o sistema em 1973 e publicaram o artigo *The UNIX Time-Sharing System* em 1974, a AT&T passou a licenciar o Unix para universidades por um valor simbólico, **com o código-fonte incluído**. Isso começou com a Fifth Edition (1974) e ganhou força com a Sixth Edition (1975).

Em 1977, o Unix já rodava em cerca de 500 instalações, 125 delas universidades. Isso transformou o Unix no sistema que uma geração inteira de estudantes de computação leu, modificou e estudou. Também deu origem à primeira grande variante.

### BSD

Em 1975, Thompson passou um ano sabático na **Universidade da Califórnia em Berkeley**. Lá, um estudante de pós-graduação, **Bill Joy**, começou a juntar melhorias ao sistema e a distribuí-las: em 1978 saiu a primeira **Berkeley Software Distribution (BSD)**. As duas primeiras versões eram coleções de ferramentas para instalar em cima do Unix da AT&T. A primeira distribuição completa do sistema foi o **3BSD**, em dezembro de 1979. Ao longo dos anos, o BSD acrescentou o editor `vi`, o `csh`, memória virtual paginada (no VAX, em 1979) e, com financiamento da DARPA, a pilha **TCP/IP**. A versão **4.2BSD**, de 1983, trouxe a interface de **sockets**, que até hoje é a forma de programar rede em praticamente todo sistema operacional. A internet cresceu em cima de máquinas rodando BSD.

Do lado da Bell Labs, a **Version 7** (1979) foi a última versão de pesquisa amplamente distribuída, e é considerada por muitos o ancestral comum de todos os Unix modernos. Ela também marcou uma mudança: a licença passou a restringir o uso do código-fonte em sala de aula. Foi essa restrição que levou Andrew Tanenbaum, professor em Amsterdã, a escrever o **MINIX** (1987), um sistema parecido com o Unix feito para ensino. O MINIX vai reaparecer mais adiante.

## A guerra do Unix

Em 1982, a AT&T fez um novo acordo com o governo e foi dividida: a partir de 1984, deixou de ser dona das companhias telefônicas regionais e, em troca, ficou livre para vender computadores e software. O Unix virou produto. A AT&T lançou o **System III** (anunciado em 1981) e, em seguida, o **System V** (1983), anunciado com o slogan *"Consider it standard"*.

O problema é que, a essa altura, já havia muitos Unix. Cada fabricante de hardware pegava uma base (System V, BSD ou uma mistura das duas), adaptava para suas máquinas e acrescentava extensões próprias:

- **SunOS**, da Sun Microsystems, empresa que Bill Joy ajudou a fundar em 1982, baseado em BSD;
- **Xenix**, da Microsoft (1980), depois mantido pela Santa Cruz Operation (SCO);
- **AIX**, da IBM;
- **HP-UX**, da HP;
- **Ultrix**, da DEC;
- **IRIX**, da Silicon Graphics.

Todos eram "Unix", mas um programa escrito para um não necessariamente compilava no outro. Chamadas de sistema, bibliotecas, opções de comandos e até o comportamento de sinais variavam. Para quem desenvolvia software, isso anulava boa parte da vantagem de portabilidade que tinha tornado o Unix especial.

### System V Release 4 e a reação

Em 1987, a AT&T e a Sun começaram a trabalhar juntas para unificar System V, BSD, SunOS e Xenix em uma única versão, o **System V Release 4 (SVR4)**, anunciado em outubro de 1988 e incorporado aos produtos comerciais a partir de 1989. Em janeiro de 1988, a AT&T ainda anunciou que ia comprar até 20% da Sun.

Os outros fabricantes enxergaram nisso uma ameaça: o Unix "oficial" passaria a ser controlado por um concorrente direto. Em 1988, IBM, DEC, HP e outras empresas fundaram a **Open Software Foundation (OSF)** para desenvolver um Unix alternativo, o **OSF/1**. A AT&T e a Sun responderam criando a **Unix International**. Esse confronto entre os dois blocos, cada um tentando definir o padrão, é o que ficou conhecido como **a guerra do Unix** (*Unix wars*).

Enquanto os fabricantes brigavam, o mercado de computadores pessoais crescia com MS-DOS e, depois, Windows NT, que a Microsoft lançou em 1993 também mirando servidores. A fragmentação do Unix deixou espaço para isso.

A guerra foi se dissolvendo no início dos anos 1990. Em 1993, os principais fabricantes anunciaram uma iniciativa comum (a COSE) para alinhar suas interfaces, e a AT&T vendeu sua divisão de Unix, a Unix System Laboratories, para a **Novell**. Ainda em 1993, a Novell anunciou que ia transferir a marca "UNIX" para o consórcio **X/Open**, o que se concretizou no início de 1994. O X/Open, em 1996, se fundiu com a OSF formando o **The Open Group**. Desde então, "UNIX" é uma marca registrada: só pode usar o nome o sistema que passa pela certificação da **Single UNIX Specification**. O macOS, por exemplo, é certificado.

### O processo contra o BSD

Uma segunda frente de briga envolveu o BSD. No fim dos anos 1980, Berkeley vinha reescrevendo, arquivo por arquivo, todo o código que ainda vinha da AT&T, com a intenção de distribuir um BSD totalmente livre. Em 1991 saiu o Net/2, quase completo, e logo apareceram o **386BSD**, para PCs, e a empresa **BSDi**, que vendia uma versão comercial.

Em 1992, a Unix System Laboratories processou a BSDi e a Universidade da Califórnia alegando uso indevido de código do Unix. O processo foi resolvido por acordo em janeiro de 1994, já com a Novell como dona. Dos 18 mil arquivos do Net/2, a universidade teve de remover só 3, alterar alguns outros e acrescentar avisos de copyright da USL em cerca de 70. O resultado foi o **4.4BSD-Lite**, lançado em junho de 1994, e os BSDs livres tiveram de refazer sua base a partir dele. Mas foram dois anos de incerteza jurídica, exatamente no período em que o Linux estava nascendo e ganhando usuários. Muita gente atribui ao processo o fato de o Linux, e não o BSD, ter se tornado o Unix livre dominante.

## As variantes hoje

Da árvore do Unix, hoje sobrevivem principalmente três galhos.

**A família System V.** O **Solaris** (cujo núcleo, o SunOS 5, a Sun passou a basear no SVR4), o **AIX** e o **HP-UX** ainda existem, principalmente em servidores corporativos antigos, mas perderam quase todo o mercado para o Linux.

**A família BSD.** Do 386BSD e do 4.4BSD-Lite vieram o **FreeBSD** e o **NetBSD** (1993), e do NetBSD saiu o **OpenBSD** (fork de 1995, com a primeira versão em 1996), focado em segurança. O FreeBSD está, por exemplo, na base do sistema do PlayStation 4 e do PlayStation 5. O ramo mais usado de todos é o do **macOS**: ele vem do NeXTSTEP, o sistema da NeXT de Steve Jobs, que combinava o microkernel Mach com código BSD. O núcleo, chamado **Darwin**, também é a base do iOS.

**Os sistemas "tipo Unix" (*Unix-like*).** São sistemas que não herdaram código do Unix original, mas se comportam como ele: oferecem as mesmas interfaces, os mesmos comandos e a mesma filosofia. O MINIX e o **Linux** são os exemplos mais conhecidos. O que tornou possível reimplementar o Unix sem copiar código foi um padrão.

## POSIX

Com tantas variantes, ficou claro nos anos 1980 que era preciso definir o que, afinal, um sistema "tipo Unix" deveria oferecer a um programa. O IEEE assumiu essa tarefa e publicou em 1988 o padrão **IEEE 1003.1**, que ficou conhecido como **POSIX**, *Portable Operating System Interface*. O "X" no final aparece porque o nome da maioria das variantes do Unix terminava em X. O nome, curiosamente, foi sugerido por Richard Stallman, que aparece na próxima seção. O padrão foi adotado pela ISO em 1990 e partiu de um padrão não oficial de 1984, criado por uma associação de fabricantes de Unix, a /usr/group.

O ponto importante do POSIX é que ele **especifica a interface, não a implementação**. Ele não diz como o kernel deve funcionar por dentro, e sim quais funções existem, quais argumentos recebem, o que devolvem e quais erros podem acontecer. Entre outras coisas, o padrão define:

- a interface de programação em C: `fork`, `execve`, `waitpid`, `open`, `read`, `write`, `kill`, sinais, variáveis de ambiente, e assim por diante;
- o shell e os utilitários de linha de comando (`sh`, `ls`, `grep`, `awk`, `sed`...), padronizados em 1992;
- as threads (**pthreads**), padronizadas em 1995.

Boa parte do que apareceu nos posts sobre [system calls de processos](/system-calls-de-processos/), [variáveis de ambiente](/variaveis-de-ambiente/) e [exceptional control flow](/exceptional-control-flow/) é, na verdade, POSIX: um programa que usa só essas interfaces compila e roda no Linux, no macOS e no FreeBSD.

O POSIX acabou incorporado pela Single UNIX Specification, que é um superconjunto dele. Um sistema pode ser **compatível com POSIX** na prática sem ter a certificação formal, que é paga. É o caso da maioria das distribuições Linux: elas seguem o padrão quase inteiramente, mas não são oficialmente "UNIX". Um empurrão importante veio do governo americano, um dos maiores compradores de computadores do mundo: a partir de 1989, o padrão de compras federal FIPS 151-1 passou a exigir conformidade com o POSIX.1, e a maioria dos fabricantes adaptou seus sistemas. Até o Windows NT chegou a ter um subsistema POSIX para poder disputar esses contratos.

## GNU

Em paralelo a tudo isso, no MIT, **Richard Stallman** via o software ficar cada vez mais fechado. No laboratório de inteligência artificial onde trabalhava, compartilhar código era o normal, e a mudança para licenças proprietárias o incomodava. Em 27 de setembro de 1983, ele anunciou o projeto **GNU**, cujo objetivo era escrever **um sistema operacional completo, compatível com o Unix e inteiramente livre**. O trabalho começou em janeiro de 1984. O BSD não resolvia o problema: na época, usá-lo ainda exigia uma licença da AT&T, e o código da AT&T que fazia parte dele não podia ser modificado e redistribuído livremente.

O nome é um acrônimo recursivo: **GNU's Not Unix**. Ser compatível com o Unix era uma escolha prática: as pessoas já conheciam aquele modelo, e um sistema compatível podia substituí-lo peça por peça.

Em 1985, Stallman publicou o **Manifesto GNU** e fundou a **Free Software Foundation (FSF)**. A definição de software livre da FSF foi publicada pela primeira vez em 1986, com duas liberdades, e foi sendo ampliada até chegar, em 1999, às quatro de hoje: executar o programa para qualquer propósito, estudar como ele funciona, redistribuir cópias e distribuir versões modificadas. Para garantir que essas liberdades não se perdessem quando alguém modificasse o código, a FSF criou em 1989 a **GPL** (*GNU General Public License*). A GPL usa o direito autoral ao contrário, no que se chamou de *copyleft*: quem distribui uma versão modificada de um programa GPL precisa distribuí-la também sob a GPL, com o código-fonte.

Ao longo dos anos 1980, o projeto GNU produziu quase tudo que um sistema operacional precisa fora do kernel:

- o editor **GNU Emacs**;
- o compilador **GCC** (1987);
- a biblioteca C **glibc**;
- o shell **bash** (1989);
- o depurador **gdb**, o **make** e os utilitários básicos (`ls`, `cp`, `cat`...), hoje reunidos no **coreutils**.

Faltava o kernel. O projeto escolheu construir o **GNU Hurd**, começado em 1990 sobre o microkernel Mach. O desenho era ambicioso, um conjunto de servidores rodando em cima do microkernel, e o desenvolvimento foi lento. Mais de três décadas depois, o Hurd ainda não está pronto para uso geral. A peça que faltava veio de outro lugar.

## Linux

Em 1991, **Linus Torvalds**, estudante da Universidade de Helsinque, tinha comprado um PC com processador Intel 386 e usava o MINIX. Insatisfeito com as limitações do MINIX, começou a escrever o próprio kernel para aprender como o 386 funcionava. Em 25 de agosto de 1991, publicou no grupo de discussão `comp.os.minix` a mensagem que ficou famosa:

> Estou fazendo um sistema operacional (livre) (só um hobby, não vai ser grande e profissional como o gnu) para clones do AT 386(486).

(Tradução livre do original em inglês.)

A versão 0.01 saiu em setembro de 1991. Desde o começo, o kernel rodava o **bash** e o **GCC** do projeto GNU: Torvalds escreveu um kernel, e o resto do sistema já existia. Em 5 de outubro de 1991, ao anunciar a versão 0.02 e pedir ajuda a outros programadores, ele listou o que já funcionava: "bash, gcc, gnu-make, gnu-sed, compress, etc.". No início a licença proibia uso comercial, mas em 1992, com a versão 0.12, ele passou o Linux para a **GPL**. Essa decisão foi decisiva: permitiu que empresas usassem e contribuíssem com o kernel, sabendo que o código continuaria aberto.

### O debate com Tanenbaum

Em janeiro de 1992, Andrew Tanenbaum, o autor do MINIX, publicou no mesmo grupo uma mensagem com o título *"LINUX is obsolete"*. A crítica era de arquitetura: o Linux era um **kernel monolítico**, em que todo o sistema operacional (escalonador, sistema de arquivos, drivers) roda no modo kernel, no mesmo espaço de endereçamento. Para Tanenbaum, o futuro eram os **microkernels**, em que o kernel faz o mínimo e o resto roda como processos isolados, como no MINIX e no Mach. Ele também criticou o fato de o Linux estar preso à arquitetura 386.

Torvalds respondeu defendendo o desenho monolítico pela simplicidade e pelo desempenho. A discussão virou um clássico. Tecnicamente, os dois tinham bons argumentos, mas na prática o Linux venceu: continua monolítico (com módulos carregáveis em tempo de execução) e foi portado para praticamente toda arquitetura que existe.

### GNU/Linux e as distribuições

Juntando o kernel Linux com as ferramentas GNU, havia pela primeira vez um sistema operacional completo e livre. Mas instalar isso tudo à mão era trabalhoso, e logo surgiram as **distribuições**: pacotes que reúnem o kernel, as ferramentas GNU, um instalador e um gerenciador de programas. As primeiras, como MCC Interim Linux, TAMU e SLS, apareceram em 1992. O **Slackware** e o **Debian** são de 1993, e a **Red Hat** e a **SUSE** vieram logo depois. O Slackware é a mais antiga ainda em atividade. O Linux 1.0 saiu em março de 1994.

Disso vem uma discussão de nomenclatura que dura até hoje. A FSF defende que o sistema completo se chame **GNU/Linux**, porque Linux é só o kernel e boa parte do restante veio do GNU. Muita gente usa simplesmente "Linux" para tudo. As duas formas são comuns, mas a distinção é útil: o **Android**, por exemplo, usa o kernel Linux, mas não usa as ferramentas GNU nem a glibc, então é Linux sem ser GNU/Linux.

### O Linux hoje

O hobby de 1991 se tornou o sistema operacional mais usado do mundo em quase tudo que não é desktop. Roda na maioria dos servidores da internet e da nuvem, nos contêineres, em roteadores, televisões e sistemas embarcados, nos bilhões de celulares Android e, desde novembro de 2017, em **todos** os 500 supercomputadores mais rápidos do mundo. O desenvolvimento do kernel envolve milhares de pessoas, a maioria paga por empresas, e Torvalds continua coordenando o projeto.

É importante notar que o Linux **não descende do Unix no código**. Nenhuma linha veio da Bell Labs ou de Berkeley. Ele é uma reimplementação que segue as interfaces do POSIX e a filosofia do Unix. Por isso, um programa em C que usa `fork`, `execve` e `waitpid`, ou um script de shell com pipes, funciona nele do mesmo jeito que funcionaria no Unix da década de 1980.

A disputa pela herança do Unix ainda teve um último capítulo nos tribunais. Em 2003, o SCO Group (antiga Caldera, que tinha comprado o negócio de Unix da Santa Cruz Operation) dizia ter herdado os direitos do Unix. A empresa processou a IBM alegando que código do Unix tinha sido copiado para o Linux e chegou a cobrar licenças de usuários de Linux. Em 2007, um juiz decidiu que os direitos autorais do Unix nunca tinham saído da Novell, e a SCO pediu falência no mesmo ano. A decisão foi revertida em recurso, mas em 2010 um júri chegou à mesma conclusão. A SCO nunca conseguiu demonstrar a cópia alegada: o primeiro trecho que apresentou como prova era, na verdade, código distribuído sob licença BSD.

## Linha do tempo

| Ano | Acontecimento |
|---|---|
| 1964 | MIT, GE e Bell Labs começam o Multics |
| 1969 | Bell Labs sai do Multics; Thompson escreve o Unix no PDP-7 |
| 1971 | Primeira edição do manual do Unix, no PDP-11 |
| 1973 | Kernel reescrito em C; pipes entram no sistema |
| 1978 | Primeira distribuição do BSD |
| 1979 | Version 7 Unix |
| 1983 | System V; 4.2BSD com TCP/IP e sockets; anúncio do projeto GNU |
| 1984 | Divisão da AT&T, que passa a vender o Unix comercialmente |
| 1985 | Manifesto GNU e fundação da FSF |
| 1987 | MINIX; AT&T e Sun começam a unificar System V e BSD |
| 1988 | POSIX (IEEE 1003.1); anúncio do SVR4; criação da OSF e da Unix International |
| 1989 | GPL; SVR4 chega aos produtos comerciais |
| 1991 | Linus Torvalds anuncia o Linux |
| 1992 | Linux passa para a GPL; debate Tanenbaum x Torvalds; processo contra a BSDi |
| 1993 | FreeBSD, NetBSD, Slackware e Debian; Novell compra o Unix da AT&T |
| 1994 | Linux 1.0; 4.4BSD-Lite encerra o processo |
| 1996 | X/Open e OSF formam o The Open Group |
| 2017 | Linux passa a rodar em todos os supercomputadores do TOP500 |

## O que as distribuições têm em comum e o que muda entre elas

Existem centenas de distribuições, mas por baixo elas compartilham a mesma base. A única peça que **toda** distribuição Linux tem é o **kernel Linux**, que gerencia o processador, a memória, os processos, os drivers e os sistemas de arquivos. A versão e a configuração do kernel variam, mas ele é sempre o Linux.

As distribuições **GNU/Linux**, que são a grande maioria (Debian, Ubuntu, Fedora, Arch, openSUSE...), também têm em comum:

- **o userland GNU**: a biblioteca C **glibc**, usada por quase todos os programas; os utilitários do **coreutils** (`ls`, `cp`, `mv`, `cat`, `chmod`...); o shell **bash**; ferramentas como `grep`, `sed`, `tar` e `find`; e o conjunto de compilação, com **GCC**, `make` e `binutils`;
- **licenças de software livre**: o kernel e as ferramentas GNU são distribuídos principalmente sob a **GPL**, então o código-fonte está disponível e qualquer pessoa pode estudá-lo, modificá-lo e redistribuí-lo;
- **a mesma estrutura de diretórios**: quase todas seguem o *Filesystem Hierarchy Standard* (FHS), com `/etc` para configurações, `/home` para os usuários, `/usr` para os programas, `/var` para dados que mudam, como logs, e `/tmp` para arquivos temporários;
- **o comportamento de um Unix**: seguem o POSIX quase inteiramente, então têm usuários e permissões (`rwx`), processos, sinais, pipes e a ideia de que tudo é arquivo.

O que muda de uma distribuição para outra são as escolhas feitas em cima dessa base:

| Aspecto | Exemplos |
|---|---|
| Gerenciador de pacotes | `apt` (Debian, Ubuntu), `dnf` (Fedora, Red Hat), `pacman` (Arch), `zypper` (openSUSE) |
| Formato dos pacotes | `.deb`, `.rpm`, `.pkg.tar.zst` |
| Sistema de init | `systemd` (a maioria), `OpenRC` (Gentoo, Alpine), `runit` (Void) |
| Ambiente gráfico | GNOME, KDE Plasma, XFCE ou gerenciadores de janelas como o Hyprland |
| Modelo de atualização | *rolling release*, sempre na versão mais nova (Arch, openSUSE Tumbleweed), ou versões fixas lançadas periodicamente (Debian, Ubuntu LTS) |
| Público e objetivo | estabilidade para servidores (Debian, RHEL), facilidade para iniciantes (Ubuntu, Linux Mint), controle total para quem monta o sistema (Arch, Gentoo) |

Algumas distribuições trocam até a parte que parecia fixa. O **Alpine Linux**, muito usado em contêineres, substitui a glibc pela **musl** e os utilitários GNU pelo **BusyBox**, para ficar pequeno. Assim como o Android, ele é Linux sem ser GNU/Linux. Por isso o kernel é a única coisa que realmente toda distribuição Linux tem em comum.

## Resumo

O **Unix** nasceu em 1969, na Bell Labs, como uma reação à complexidade do Multics. Ser reescrito em **C** o tornou portável, e as licenças baratas para universidades o espalharam. Em Berkeley nasceu o **BSD**, que trouxe o TCP/IP e os sockets.

Quando a AT&T pôde vender o Unix, cada fabricante já tinha a sua versão, e a disputa entre o bloco AT&T/Sun (SVR4) e o bloco IBM/DEC/HP (OSF) ficou conhecida como a **guerra do Unix**. Ela terminou nos anos 1990, com a marca UNIX nas mãos do The Open Group. Hoje sobrevivem a família **System V** (Solaris, AIX, HP-UX), a família **BSD** (FreeBSD, NetBSD, OpenBSD, macOS) e os sistemas **tipo Unix**.

O **POSIX** padronizou a interface que um sistema tipo Unix deve oferecer aos programas, e isso permitiu reimplementar o Unix sem copiar seu código. O projeto **GNU**, de Richard Stallman, escreveu quase todo esse sistema de forma livre, protegido pela GPL, mas não conseguiu terminar o kernel. O **Linux**, de Linus Torvalds, preencheu essa lacuna em 1991. Juntos, GNU e Linux formaram o sistema livre que hoje roda na maior parte da infraestrutura computacional do mundo.
