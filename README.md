<div align="center">

<img src="logo.png" alt="Componentes internos de um HD" width="500"><br>
NIVEA SOFIA DA CAS

<br><br><br><br><br>

**O CAOS DO SHOW DE ROCK: PROCESSOS, THREADS E CONDIÇÃO DE CORRIDA EM UM SISTEMA DE VENDA DE INGRESSOS**

<br><br><br><br><br><br><br><br>

LONDRINA<br>
2026

</div>

<div style="page-break-after: always;"></div>

<div align="center">

<br><br>

NIVEA SOFIA DA CAS

<br><br><br>

**O CAOS DO SHOW DE ROCK: PROCESSOS, THREADS E CONDIÇÃO DE CORRIDA EM UM SISTEMA DE VENDA DE INGRESSOS**

<br><br>

</div>

<div style="margin-left: 8cm; text-align: justify;">

Desafio Prático de Sala de Aula Invertida: Arquitetura e Concorrência 

</div>

<br><br><br><br>

<div align="center">

LONDRINA<br>
2026

</div>

<div style="page-break-after: always;"></div>

## SUMÁRIO

- [INTRODUÇÃO](#introdução)
- 1. [DESENVOLVIMENTO](#1-desenvolvimento)
  - [1.1. Síntese Teórica (Pesquisa)](#11-síntese-teórica-pesquisa)
    - [1.1.1. Diferença entre Processo e Thread](#111-diferença-entre-processo-e-thread)
    - [1.1.2. Por que criar threads é mais leve do que criar processos](#112-por-que-criar-threads-é-mais-leve-do-que-criar-processos)
    - [1.1.3. Threads em Modo Usuário e em Modo Núcleo (Kernel)](#113-threads-em-modo-usuário-e-em-modo-núcleo-kernel)
  - [1.2. Diagnóstico do Problema (O gargalo)](#12-diagnóstico-do-problema-o-gargalo)
    - [1.2.1. A Condição de Corrida (Race Condition)](#121-a-condição-de-corrida-race-condition)
    - [1.2.2. Consequências para o sistema de ingressos](#122-consequências-para-o-sistema-de-ingressos)
  - [1.3. A Solução Arquitetural](#13-a-solução-arquitetural)
    - [1.3.1. Exclusão Mútua e Regiões Críticas](#131-exclusão-mútua-e-regiões-críticas)
    - [1.3.2. Aplicação ao Assento A-15](#132-aplicação-ao-assento-a-15)
- 2. [CONCLUSÃO](#2-conclusão)
- [REFERÊNCIAS](#referências)

<div style="page-break-after: always;"></div>

## INTRODUÇÃO

Este trabalho trata de um problema comum em sistemas de alta demanda: o que acontece quando muitas rotinas de um mesmo programa são executadas ao mesmo tempo e disputam o mesmo recurso. O cenário proposto é o de um sistema de venda de ingressos online para um show internacional, que deve suportar cerca de 100.000 acessos simultâneos no instante em que as vendas forem abertas.

Os temas centrais são a diferença entre processo e thread, o motivo de as threads serem mais leves para atender muitos usuários, as diferenças entre threads em modo usuário e em modo núcleo, a condição de corrida e a exclusão mútua. O problema que orienta o estudo é: como atender uma quantidade enorme de usuários ao mesmo tempo sem que dois deles consigam comprar a mesma cadeira?

O objetivo é, na função de arquitetos de software, explicar por que o uso de threads é adequado ao cenário, diagnosticar a falha que surge quando dois usuários tentam comprar o assento A-15 no mesmo instante e propor uma solução baseada em regiões críticas e exclusão mútua. 

A pesquisa foi feita a partir do estudo do material da Aula 7 (Modelo de Processos, Threads e Concorrência), da consulta ao livro *Sistemas Operacionais Modernos*, de Andrew S. Tanenbaum e Herbert Bos, e de materiais complementares disponíveis na internet, como documentações técnicas e notas de aula de universidades. 

## 1. DESENVOLVIMENTO

### 1.1. Síntese Teórica (Pesquisa)

#### 1.1.1. Diferença entre Processo e Thread

Um processo é uma instância de um programa, que reúne os recursos necessários para a execução de um software: o espaço de endereçamento (onde ficam código e dados), as variáveis globais, os arquivos abertos e as informações de controle que o sistema operacional mantém na tabela de processos. Quando o processo é interrompido, por exemplo para aguardar uma resposta da rede, o sistema guarda seu estado nessa tabela e a CPU passa a executar outro; ao ser escalonado novamente, ele retoma de onde parou. Por isso, o processo é a unidade de **agrupamento de recursos**.

A thread é uma linha de execução, uma sequência de instruções dentro de um processo. Um software é organizado em processos, e cada processo pode ser dividido em várias threads, que formam tarefas independentes, mas relacionadas entre si. Cada thread carrega apenas o necessário para executar por conta própria: seu contador de programa, seus registradores, sua pilha e seu estado (executando, pronta ou bloqueada). Todo o restante é compartilhado entre as threads do mesmo processo. Por isso, a thread é a unidade de **execução** e de escalonamento.

#### 1.1.2. Por que criar threads é mais leve do que criar processos

Para atender 100.000 usuários, o servidor precisa de 100.000 "linhas de atendimento". Se cada uma fosse um processo novo, o sistema operacional teria de criar, para cada um, um espaço de endereçamento completo, tabelas de controle, descritores de arquivos e outras estruturas. Além disso, a troca entre processos exige trocar o mapeamento de memória, o que pode invalidar caches e a tabela de tradução de endereços do processador.

Com threads, esse trabalho é evitado porque todas elas **usam o espaço de endereçamento que já existe**. As threads não possuem recursos próprios atrelados a elas, portanto, são dezenas a centenas de vezes mais rápidas para criar e destruir do que processos. Criar uma thread significa basicamente reservar uma pilha e um pequeno bloco de controle para guardar registradores e contador de programa. A troca entre threads do mesmo processo também é mais barata, pois a memória continua a mesma.

Outra vantagem é a comunicação: como todas as threads enxergam os mesmos dados, não é preciso copiar informações entre elas nem usar mecanismos de comunicação entre processos. No caso do sistema de ingressos, isso é útil porque todas as threads precisam consultar e atualizar o mesmo mapa de assentos. 

#### 1.1.3. Threads em Modo Usuário e em Modo Núcleo (Kernel)

**Threads em modo usuário.** Nesse modelo, quem cria e organiza as threads é o próprio programa, por meio de uma biblioteca. O sistema operacional não sabe que elas existem: para ele, o programa é um processo comum, com uma única linha de execução. Isso traz um ganho de velocidade, pois criar uma thread ou alternar entre elas não exige pedir nada ao núcleo, e o programa ainda pode escolher a sua própria forma de revezar as threads. O problema é que o núcleo enxerga apenas uma unidade de execução. Assim, se uma thread faz uma operação que fica esperando (como ler um arquivo no disco), o sistema bloqueia o processo inteiro e todas as outras threads param junto. Além disso, como o núcleo não participa da troca entre elas, uma thread que não libera a CPU pode travar as demais, e o programa não consegue usar vários processadores ao mesmo tempo.

**Threads em modo núcleo (kernel).** Aqui, o próprio sistema operacional cria, controla e escalona as threads, mantendo um registro de todas elas. Como o núcleo conhece cada thread individualmente, ele consegue bloquear apenas a que está esperando e continuar executando as outras, do mesmo processo ou de outros. Ele também consegue distribuí-las entre vários núcleos do processador, executando-as de fato ao mesmo tempo. Em contrapartida, criar, encerrar e alternar entre threads passa a exigir chamadas ao sistema operacional, o que é mais lento do que fazer o mesmo apenas dentro do programa.

### 1.2. Diagnóstico do Problema (O gargalo)

#### 1.2.1. A Condição de Corrida (Race Condition)

Uma condição de corrida acontece quando duas ou mais threads (ou processos) acessam o mesmo dado compartilhado ao mesmo tempo, e o resultado depende de qual delas executa suas instruções primeiro. Um exemplo clássico é a fila de impressão: dois processos consultam a mesma posição livre da fila e ambos gravam nela, de modo que o trabalho de um apaga o do outro. A causa do problema é que a sequência "ler o valor, tomar a decisão e gravar o novo valor" **não é indivisível**: o escalonador pode pausar uma thread no meio dessa sequência e passar a CPU para outra, que então trabalha com um dado já desatualizado.

No nosso caso, o Usuário A e o Usuário B clicam em "Comprar" para o assento A-15 no mesmo milissegundo. O código de reserva, de forma simplificada, segue três passos: verificar se o assento está livre, marcá-lo como vendido e emitir o ingresso. 

#### 1.2.2. Consequências para o sistema de ingressos

Se o problema não fosse tratado, o sistema:

1. **Venderia o mesmo assento duas vezes**, com dois clientes pagando por uma única cadeira.
2. **Teria dados inconsistentes**: o banco registraria apenas o último comprador, enquanto o primeiro teria um comprovante sem lugar correspondente.
3. **Geraria conflitos na portaria**, com pessoas sem lugar e necessidade de reembolsos.
4. **Causaria prejuízo à imagem da empresa**, especialmente porque, com 100.000 acessos simultâneos, a chance de ocorrerem disputas pelos mesmos assentos é muito alta.

### 1.3. A Solução Arquitetural

#### 1.3.1. Exclusão Mútua e Regiões Críticas

O conceito que resolve a condição de corrida é a **exclusão mútua**, aplicada sobre as **regiões críticas**. A região crítica é o trecho do programa que acessa o recurso compartilhado (nesse caso, o trecho que verifica e altera o estado do assento). A exclusão mútua garante que, enquanto uma thread está nesse trecho, nenhuma outra consiga entrar nele, enquanto as demais ficam esperando a sua vez.

Para que a solução funcione bem, ela precisa atender a quatro condições: duas threads nunca podem estar na região crítica ao mesmo tempo e a solução não pode depender da velocidade ou do número de processadores. Uma thread que está fora da região crítica não pode bloquear as outras e nenhuma thread pode esperar indefinidamente para entrar.

Na prática, isso é implementado com mecanismos como o **mutex**, os **semáforos** e os **monitores**. O mutex funciona como uma tranca com duas operações: *lock*, que tranca (e faz a thread esperar caso já esteja trancado), e *unlock*, que destranca e libera a próxima thread da fila. Essas operações são atômicas, ou seja, indivisíveis, o que elimina o intervalo de tempo em que a corrida acontecia.

#### 1.3.2. Aplicação ao Assento A-15

Para o assento A-15, toda a compra (verificar se está livre, marcar como vendido e registrar o comprador) passa a ser uma região crítica protegida por um mutex. Quando o Usuário A e o Usuário B clicam ao mesmo tempo, apenas um deles, por exemplo o A, consegue trancar o mutex e entra na região crítica. O B fica bloqueado, aguardando. O A verifica que o assento está livre, o marca como vendido e destranca. Só então o B entra, verifica que o assento já está vendido e recebe a mensagem de que ele não está mais disponível. Assim, apenas um ingresso é emitido e os dados continuam consistentes.

A trava deve proteger apenas o assento disputado (ou um setor), e não o sistema inteiro, para que compras de assentos diferentes continuem em paralelo.

## 2. CONCLUSÃO

O trabalho alcançou os objetivos propostos. Foi possível explicar que o processo é a unidade que agrupa recursos, enquanto a thread é a unidade de execução que compartilha esses recursos dentro do processo. Como as threads aproveitam o espaço de endereçamento já existente, elas são muito mais rápidas e baratas de criar do que processos, o que as torna adequadas para atender 100.000 usuários. Também foram comparadas as threads em modo usuário e em modo núcleo, concluindo-se que, para um servidor que realiza operações que esperam resposta (como acesso a banco de dados) e precisa aproveitar vários processadores, o modo núcleo é o mais adequado.

O diagnóstico mostrou que o mesmo compartilhamento de memória que dá desempenho às threads cria a condição de corrida: sem controle, dois usuários poderiam comprar o assento A-15 ao mesmo tempo, gerando venda duplicada e dados inconsistentes. A solução é tratar a verificação e a reserva do assento como uma região crítica protegida por exclusão mútua, usando mutex, semáforo ou monitor, de modo que apenas uma thread por vez altere o estado do assento. Para preservar o desempenho, a trava deve proteger apenas o assento disputado, e não o sistema inteiro.

A metodologia, baseada na Aula 7, no livro de Tanenbaum e Bos e em materiais complementares, foi adequada para responder à problemática, pois permitiu relacionar a teoria de sistemas operacionais a um caso concreto de arquitetura de software. O estudo tratou a concorrência dentro de um único servidor.

## REFERÊNCIAS

### Texto

ALECRIM, Emerson; MARQUES, Ana. **O que são threads do processador e quais os benefícios do multithreading?** Tecnoblog, 2023. Disponível em: https://tecnoblog.net/responde/o-que-e-thread-processador/. Acesso em: 8 out. 2026.


MICROSOFT. **Sobre processos e threads**. Microsoft Learn, [s.d.]. Disponível em: https://learn.microsoft.com/pt-br/windows/desktop/ProcThread/about-processes-and-threads. Acesso em: 8 out. 2026.


TANENBAUM, Andrew S.; BOS, Herbert. **Sistemas operacionais modernos**. Tradução: Jorge Ritter. Revisão técnica: Raphael Y. de Camargo. 4. ed. São Paulo: Pearson Education do Brasil, 2016.

### Vídeo

COUTO, Rodrigo de Souza. **EEL770 - 19 - Introdução aos problemas de concorrência**. [S. l.]: YouTube, [2020]. 1 vídeo ([24] min). Disponível em: [https://www.youtube.com/watch?v=ohCMTCfUBUs]. Acesso em: 8 out. 2026.