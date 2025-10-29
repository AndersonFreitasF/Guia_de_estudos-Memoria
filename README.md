# Um guia de estudo com todo o essencial sobre memória e como ela funciona com o computador e com os códigos compilados

Olá, me chamo Anderson e estou escrevendo essa guia para tentar ajudar pessoas que querem entender mais a fundo como a memória do seu computador funciona.

Esse guia existe porque anteriormente estava fazendo todas as anotações na mão, o que era tedioso, então resolvi escrever no teclado e pensei em criar um repositório no Github para manter minhas anotações públicas no caso de alguem se interessar

Todo o conteúdo aqui é escrito por mim e são meus entendimentos sobre o artigo **_What Every Programmer Should Know About Memory_** do Ulrich Drepper, escrito em 2007, e está sujeito a mudanças e erros, coisas que serão revisadas com o passar do tempo!

## Sumário
1. [Caminho da memória](#capítulo-1---caminho-da-memória)
   - [Tabela de latência da memória](#tabela-de-latência-da-memória)
   - [Processo de acesso da DRAM](#processo-de-acesso-da-dram)
   - [Impacto no desempenho](#impacto-no-desempenho)
2. CPU Caches *(em desenvolvimento)*

---

# Capítulo 1 - Caminho da memória

Esse bloco tem como intuito entender onde o tempo é gasto quando a CPU lê ou escreve um dado.

Quando a CPU lê algo da RAM, ela segue o seguinte caminho:

**Registrador → Cache L1 → Cache L2 → Cache L3 → Controlador de Memória → DRAM**

Cada etapa desse processo é mais lenta e mais pesada que a anterior, o que causa uma ida da CPU até a RAM ser extremamente custosa

---

## Tabela de latência da memória

| Camada         | Tipo                      | Tamanho (típico) | Latência (aprox.) |
|:--------------:|:-------------------------:|:----------------:|:----------------:|
| Registradores  | SRAM*                     | ~1KB             | 1 ciclo |
| L1 Cache       | SRAM                      | 32–64KB          | 3–5 ciclos |
| L2 Cache       | SRAM                      | 256KB–1MB        | 10–20 ciclos |
| L3 Cache       | SRAM                      | 2–64MB           | 40–70 ciclos |
| DRAM           | Dinâmica (capacitores)    | GBs              | 150–300 ciclos |

> *A SRAM dos registradores em específico é mais rápida que as demais*  
> *Entenda 1 ciclo como 1 clock da sua CPU*

---

A DRAM é construída em um formato de grade com linhas e colunas, e dentro dessa grade estão contidas as células de memória, onde cada célula de DRAM é **1 transistor + 1 capacitor**  
O transistor é a portinha de entrada/saída, e o capacitor armazena a carga (0 ou 1).

Mas o capacitor vaza e tem que ser regravado (processo chamado de _refresh_) a cada 64 ms, onde durante esse periodo a linha não pode ser acessada

---

## Processo de acesso da DRAM

Pra acessar um único bit, o controlador precisa:

1. Ativar a linha (**Row**) com o sinal **RAS** (*Row Address Strobe*)
2. Ativar a coluna (**Column**) com o sinal **CAS** (*Column Address Strobe*)
3. Depois vem o burst: vários dados consecutivos são transmitidos, preenchendo uma cache line de 64 B

Cada célula só responde quando **as duas** estão ativas, então o bit da memória só pode ser lido ao ser selecionado

---

Apesar da arquitetura complicada, existe um motivo para a DRAM ser feita em linhas e colunas:

Cada célula da DRAM precisa ser endereçada, e pra isso o chip tem pinos de endereço (chamados de *address lines*). Quanto mais memória o chip tem, mais bits de endereço ele precisa receber, o que resulta em mais linhas físicas e consequentemente em mais dinheiro gasto

Por isso, os engenheiros bolaram um truque para economizar:

Eles multiplexaram os endereços, assim reusam as mesmas linhas para o CAS e para o RAS (enviam metade primeiro e metade depois)

Esse processo reduz custo, mas adiciona tempo de espera, o que se acumula com outros processos elétricos que geram ainda mais tempo de espera, tais como:

1. O capacitor da célula carrega e descarrega lentamente (milissegundos em escala de tempo da CPU)
2. O sinal que ele gera é fraco, então precisa de um amplificador (sense amplifier) pra detectar se é 0 ou 1
3. Depois, o dado precisa viajar do chip até o controlador (vários centímetros em trilhas)

---

Esses delays cumulativos são medidos através de:

- **tRCD** (*RAS → CAS Delay*)
- **CL** (*CAS Latency*)
- **tRP** (*Row Precharge*)
- **tRAS** (*Row Active Time*)

Esses tempos são o motivo pelo qual a memória é “lenta”, mesmo quando a frequência parece alta

---

## Impacto no desempenho

A nível de performance de código:

- Acessos sequenciais são rápidos porque a linha já está aberta (não precisa novo RAS), e o burst preenche uma cache line inteira
- Acessos aleatórios acabam com a performance porque têm que fechar a linha (tRP) e abrir outra (tRCD) e o cache prefetcher não consegue prever o padrão

---

O acesso à memória é lento não por falta de frequência, mas por física e arquitetura

Cada bit da DRAM vive em um capacitor isolado, que precisa ser selecionado, amplificado e recarregado, e pra isso, o controlador faz uma sequência de sinais (RAS, CAS, precharge) que consome tempo e exige linhas físicas de endereço

Enquanto isso, a CPU (milhares de vezes mais rápida) fica parada esperando.

---

**Resultado final:**
> A diferença entre o ritmo do processador e o da DRAM é o maior gargalo dos sistemas modernos e o papel das caches é mascarar essa diferença.










## Glossário
| Termo | Significado |
|------|-------------|
| [CPU](#cpu) | Executa instruções |
| [Registrador](#registrador) | Memória mais rápida da CPU |
| [Cache](#cache) | Memória intermediária super rápida |
| [L1 / L2 / L3](#l1--l2--l3) | Níveis de cache — maior = mais lento |
| [DRAM](#dram) | Memória principal baseada em capacitores |
| [SRAM](#sram) | Memória rápida de caches e registradores |
| [Controlador de Memória](#controlador-de-memória) | Coordena acesso CPU ↔ RAM |
| [Endereço](#endereço) | Identificação numérica de um dado |
| [Latência](#latência) | Tempo de espera por um acesso |
| [Linha da Cache](#linha-da-cache) | Bloco de transferência (~64B) |
| [Prefetcher](#prefetcher) | Tenta prever acessos futuros |
| [RAS](#ras-row-address-strobe) | Ativa linha da DRAM |
| [CAS](#cas-column-address-strobe) | Ativa coluna da DRAM |
| [tRCD](#trcd) | Delay RAS → CAS |
| [CL](#cl-cas-latency) | Latência da coluna |
| [tRP](#trp) | Pré-carregar nova linha |
| [tRAS](#tras) | Tempo mínimo de linha ativa |
| [Refresh](#refresh) | Recarrega capacitores da DRAM |
| [Sense Amplifier](#sense-amplifier) | Detecta o valor 0/1 do capacitor |
| [Burst](#burst) | Transferir vários dados seguidos |
| [Address Lines](#address-lines) | Linhas físicas de endereço no chip |
