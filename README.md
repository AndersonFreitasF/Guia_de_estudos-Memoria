# Um guia de estudo com todo o essencial sobre memória e como ela funciona com o computador e com os códigos compilados

Olá, me chamo Anderson e estou escrevendo essa guia para tentar ajudar pessoas que querem entender mais a fundo como a memória do seu computador funciona.

Esse guia existe porque anteriormente estava fazendo todas as anotações na mão, o que era tedioso, então resolvi escrever no teclado e pensei em criar um repositório no Github para manter minhas anotações públicas no caso de alguém se interessar.

Todo o conteúdo aqui é escrito por mim e são meus entendimentos sobre o artigo **_What Every Programmer Should Know About Memory_** do Ulrich Drepper, escrito em 2007, e está sujeito a mudanças e erros, coisas que serão revisadas com o passar do tempo!

Como estou escrevendo enquanto leio o artigo vou seguir um modelo onde escrevo tudo que estou entendendo no momento e ao terminar tudo volto para revisar

## Sumário
1. [Caminho da memória](#capítulo-1---caminho-da-memória)
   - [Tabela de latência da memória](#tabela-de-latência-da-memória)
   - [Processo de acesso da DRAM](#processo-de-acesso-da-dram)
   - [Impacto no desempenho](#impacto-no-desempenho)
2. [CPU Caches](#capítulo-2---cpu-caches)
   - [Hierarquia de cache e localidade temporal/espacial](#hierarquia-de-cache-e-localidade-temporalespacial)
   - [Mapeamento e associatividade](#mapeamento-e-associatividade)
   - [Políticas de escrita](#políticas-de-escrita)
   - [Multi-core e coerência](#multi-core-e-coerência)
      - [Estados de coerência](#estados-de-coerência)
      - [Comparativo de protocolos](#comparativo-de-protocolos)
3. [Glossário](#glossário)

---

# Capítulo 1 - Caminho da memória

Esse bloco tem como intuito entender onde o tempo é gasto quando a CPU lê ou escreve um dado, além de entendermos o motivo do gargalo em sistemas atuais.

Quando a CPU lê algo da RAM, ela segue o seguinte caminho:

| Registrador → Cache L1 → Cache L2 → Cache L3 → Controlador de Memória → DRAM |
|------------------------------------------------------------|

Cada etapa desse processo é mais lenta e mais pesada que a anterior, o que causa uma ida da CPU até a RAM ser extremamente custosa.

---

## Tabela de latência da memória

| Camada         | Tipo                      | Tamanho (típico) | Latência (aprox.) |
|:--------------:|:-------------------------:|:----------------:|:----------------:|
| Registradores  | memória dedicada dentro da CPU | ~1KB | 1 ciclo |
| L1 Cache       | SRAM | 32–64KB | 3–5 ciclos |
| L2 Cache       | SRAM | 256KB–1MB | 10–20 ciclos |
| L3 Cache       | SRAM | 2–64MB | 40–70 ciclos |
| DRAM           | Dinâmica (capacitores) | GBs | 150–300 ciclos |

> *A memória dos registradores é mais rápida que as SRAMs*  
> *Entenda 1 ciclo como 1 clock da sua CPU*

---

A DRAM é construída em um formato de grade com linhas e colunas, e dentro dessa grade estão contidas as células de memória, onde cada célula de DRAM é **1 transistor + 1 capacitor**.  
O transistor é a portinha de entrada/saída, e o capacitor armazena a carga (0 ou 1).

Mas o capacitor vaza e tem que ser regravado (processo chamado de *refresh*) a cada **64 ms**, onde durante esse período a linha não pode ser acessada.

---

## Processo de acesso da DRAM

Para acessar um único bit, o controlador precisa:

1. Ativar a linha (**Row**) com o sinal **RAS** (*Row Address Strobe*)
2. Ativar a coluna (**Column**) com o sinal **CAS** (*Column Address Strobe*)
3. Depois vem o burst: vários dados consecutivos são transmitidos, preenchendo uma cache line de **64 B**

Vou falar mais sobre cache lines no Capítulo 2. Caso não tenha entendido e não queira ir no glossário, pode continuar a leitura normalmente.

Cada célula só responde quando **as duas** estão ativas, então o bit da memória só pode ser lido ao ser selecionado.

---

Apesar da arquitetura complicada, existe um motivo para a DRAM ser feita em linhas e colunas:

Cada célula da DRAM precisa ser endereçada, e para isso o chip tem pinos de endereço (chamados de *address lines*). Quanto mais memória o chip tem, mais bits de endereço ele precisa receber, o que resulta em mais linhas físicas e consequentemente em mais dinheiro gasto.

Por isso, os engenheiros bolaram um truque para economizar:

Eles **multiplexaram os endereços**, assim reusam as mesmas linhas para o CAS e para o RAS (enviam metade primeiro e metade depois).

Esse processo reduz custo, mas **adiciona tempo de espera**, o que se acumula com outros processos elétricos que geram ainda mais tempo de espera, tais como:

1. O capacitor da célula carrega e descarrega lentamente (milissegundos em escala de tempo da CPU)
2. O sinal que ele gera é fraco, então precisa de um amplificador (*sense amplifier*) para detectar se é 0 ou 1
3. Depois, o dado precisa viajar do chip até o controlador (vários centímetros em trilhas físicas)

---

Esses delays cumulativos são medidos através de:

- **tRCD** (*RAS → CAS Delay*)
- **CL** (*CAS Latency*)
- **tRP** (*Row Precharge*)
- **tRAS** (*Row Active Time*)

Esses tempos são o motivo pelo qual a memória é “lenta”, mesmo quando a frequência parece alta.

---

## Impacto no desempenho

A nível de performance de código:

- Acessos sequenciais são rápidos porque a linha já está aberta (não precisa novo RAS), e o burst preenche uma cache line inteira
- Acessos aleatórios acabam com a performance porque têm que fechar a linha (tRP) e abrir outra (tRCD), e o cache prefetcher não consegue prever o padrão

---

O acesso à memória é lento não por falta de frequência, mas por **física e arquitetura**.

Cada bit da DRAM vive em um capacitor isolado, que precisa ser selecionado, amplificado e recarregado. Para isso, o controlador faz uma sequência de sinais (RAS, CAS, precharge) que **consome tempo** e **exige linhas físicas de endereço**, encarecendo o hardware.

Enquanto isso, a CPU (milhares de vezes mais rápida) fica parada esperando.

---

**TLDR do capítulo:**  
> A diferença entre o ritmo do processador e o da DRAM é o maior gargalo dos sistemas modernos e o papel das caches é mascarar essa diferença.

---

# Capítulo 2 - CPU Caches

---

## Hierarquia de cache e localidade temporal/espacial

Esse bloco tem como intuito entender como é feito o cache e como o gargalo da CPU é mascarado para o usuário final (você e eu).

Em máquinas antigas, o clock da CPU e das memórias eram equivalentes, mas com o avanço da CPU as memórias não conseguiram acompanhar a frequência do núcleo e tornou-se inevitável a criação das camadas de cache.

As camadas de cache funcionam copiando e armazenando temporariamente os dados e as instruções lidas que ainda estão em uso e são construídas usando SRAM, se baseando em 2 princípios fundamentais que podemos ver em qualquer sistema real:

- localidade temporal
- localidade espacial

A localidade temporal afirma que dados usados recentemente tendem a ser usados de novo, enquanto a localidade espacial afirma que dados próximos na memória costumam ser usados juntos (como um array de elementos, por exemplo).

---

| CPU Core → Cache → Memory bus → Memória principal |
|--------------------------------------------------|
> *Modelo mínimo de cache em arquitetura descrito por Drepper*

Na prática, existem vários níveis, onde nenhum acesso vai direto da CPU para a RAM. Tudo passa pelo cache.

Um exemplo de modelo de caches em sistemas atuais é:

| Nível | Função | Tamanho (típico) | Latência (aprox.) |
|:-----:|:------:|:----------------:|:----------------:|
| L1I | Cache de instruções, guarda o código executado | 32KB | 3–5 ciclos |
| L1D | Cache de dados, guarda os dados acessados | 32KB | 3–5 ciclos |
| L2  | Cache unificada, código + dados | 256KB–1MB | 10–20 ciclos |
| L3  | Cache unificada, compartilhada entre cores | 2–64MB | 40–70 ciclos |

---

Usando como base a tabela, o fluxo de acesso ocorre da seguinte forma:

1. A CPU pede algum dado que está no endereço X
2. A L1 é consultada:
   - Se o dado for encontrado, ocorre um **cache hit**
3. Caso contrário, ocorre um **cache miss** e buscamos o dado na L2
4. A L2 é consultada:
   - Se o dado for encontrado, copiamos ele para a L1 e continuamos a execução
   - Se não for encontrado, buscamos na L3 ou direto na RAM caso não exista L3

> *Lembrando que cada nível é mais lento e mais pesado do que o anterior, como comentado no primeiro capítulo*

Se o dado chegar ao ponto de ser buscado na RAM, a CPU vai literalmente parar enquanto aguarda o dado ser encontrado, pois o processo pode demorar cerca de 200 ciclos, enquanto buscas em caches demoram ~15 ciclos.

Esse atraso é o motivo do conceito de *stall*, onde ocorre parada do pipeline.

Nossa meta para gerar ganho de performance vem de maximizar hits e evitar misses.

---

## Mapeamento e associatividade

Entrando na estrutura física interna das caches, vemos que cada cache é dividida em linhas: as cache lines que foram mencionadas lá no Capítulo 1, e que essas cache lines costumam ter cerca de 64B e guardam uma faixa de endereços na memória.

Quando um dado é lido, a cache carrega toda essa linha do dado, não só o byte.

Então, quando você tenta ler um array no valor [0] em um loop de repetição normal (`for (let i = 1; i < array.length; i++)`, por exemplo), a cache carrega toda a linha e os 64 bytes que estão nesse endereço da memória.

Ao acessar o valor [1], ele já vai estar carregado na memória e você terá praticamente zero latência para ler esse dado.

Essa técnica é chamada de **explorar a localidade espacial**.

---

As caches são estruturadas em um modelo de conjuntos (sets), e cada conjunto é estruturado em vias (ways) que podem armazenar várias cache lines. Essa estrutura é usada para definir os tipos de cache, sendo eles:

| Tipo | Força de endereçamento | Características |
|:--------------:|:-------------------------:|:----------------:|
| Directed map  | 1 linha por endereço | simples e rápida, mas de fácil colisão |
| n-way set associative | um endereço pode ir para n posições | reduz colisões, mas o controle é mais complexo |
| fully associative | qualquer linha pode ir para qualquer posição | lenta e cara, mas extremamente flexível |

> *segunda tabela do capítulo, espero não ter que fazer uma terceira*  
> *As caches modernas são tipicamente 8-way ou 16-way*

---

Quando um endereço chega na cache, ele é mapeado e dividido em 3 partes: **Tag, Index e Offset**

- Tag → identifica o bloco de memória armazenado dentro do conjunto
- Index → define qual conjunto (set) será consultado
- Offset → indica o byte dentro da linha de cache (posição 0–63)

---

Na leitura, a cache compara a tag do endereço pedido com a tag armazenada:

- Se bate → **hit**
- Se não → **miss** e a linha é substituída

Quando dois endereços se mapeiam para o mesmo conjunto, eles competem por espaço e ocorre **conflict miss**, o que gera **cache thrashing**, processo onde os dados ficam se substituindo sem parar e não permanecem tempo suficiente para serem reutilizados.

---

## Políticas de escrita

Existem duas políticas de escrita principais: Write-through e Write-back

**Write-through:**  
cada escrita vai direto para a RAM e para a cache, o que garante coerência, mas é mais lento

**Write-back:**  
a escrita fica só na cache; a RAM é atualizada depois, o que é muito mais rápido, mas requer mecanismos de controle (bit “dirty”)

Sistemas modernos usam Write-back nas L1/L2, e às vezes usam Write-through na L3.

---

## Multi-core e coerência

Falando em L1/L2 e L3, em CPUs multi-core as caches L1/L2 são individuais por core, mas eles compartilham a L3, o que exige protocolos de coerência de cache para garantir que todos os núcleos vejam os mesmos dados atualizados.

Os principais protocolos são MESI, MOESI e MESIF, onde cada letra tem um significado e juntas formam um protocolo.

---

### Estados de coerência

| Letra | Nome | Significado | Consequência |
|:-----:|:------:|:-------------|:--------------|
| M | Modified | Linha foi modificada nesta cache. Diferente da RAM. Nenhuma outra cache tem cópia | Precisa fazer write-back antes de liberar. Escrita livre. |
| E | Exclusive | Linha igual à RAM e só existe nesta cache | Pode ser modificada sem avisar ninguém (vira M) |
| S | Shared | Linha igual à RAM e pode estar em várias caches ao mesmo tempo | Somente leitura. Para escrever → precisa pedir exclusividade |
| I | Invalid | Linha desatualizada ou vazia | Qualquer acesso gera miss |
| O | Owned | (MOESI) Linha modificada e esta cache é a fonte oficial. Outras têm Shared | Evita write-back imediato na RAM |
| F | Forward | (MESIF) Linha Shared, mas esta cache responde pelos outros para evitar colisões | Reduz tráfego do barramento na leitura |
>*Aff*

> *essa é a penúltima tabela do capítulo, eu juro*

---

### Comparativo de protocolos

| Protocolo | Letras usadas | Vantagem | Desvantagem | Usado por |
|:---------:|:-------------:|:--------:|:-----------:|:---------:|
| MESI | M, E, S, I | Base simples e eficiente para coerência | Write-back na leitura de linha Modified | Intel, ARM, x86 clássico |
| MOESI | M, O, E, S, I | Evita write-back usando estado Owned → esta cache serve as outras | Hardware mais complexo | AMD (Athlon, EPYC, Ryzen) |
| MESIF | M, E, S, I, F | Estado Forward define quem responde → reduz tráfego entre caches Shared | Ainda exige write-back como MESI | Intel (Nehalem em diante) |

> *chega de tabelas por enquanto*

Sem coerência, um núcleo poderia trabalhar com uma cópia desatualizada da memória.

A coerência é mantida por sinais de controle no barramento interno do chip.

---

**TLDR do capítulo:**  
As otimizações de performance vêm do aproveitamento da localidade espacial e da localidade temporal, agrupando dados que serão usados juntos e fazendo com que dados frequentes caibam na L2 (qualquer coisa além disso já é uma perda enorme de desempenho).

Laços que percorrem arrays em sequência aproveitam o prefetch e as cache lines, sendo mais rápidos do que estruturas de dados dispersas como linked lists, grafos e árvores que têm nós alocados aleatoriamente, pois causam cache misses constantes.

---

# Glossário

| Termo | Significado |
|------|-------------|
| CPU | Executa instruções |
| Registrador | Memória mais rápida da CPU |
| Cache | Memória intermediária super rápida |
| L1 / L2 / L3 | Níveis de cache — maior = mais lento |
| DRAM | Memória principal baseada em capacitores |
| SRAM | Memória rápida de caches e registradores |
| Controlador de Memória | Coordena acesso CPU ↔ RAM |
| Endereço | Identificação numérica de um dado |
| Latência | Tempo de espera por um acesso |
| Ciclo | Pulso do clock da CPU |
| Cache Line | Bloco de transferência (~64B) |
| Prefetcher | Tenta prever acessos futuros |
| RAS | Ativa linha da DRAM |
| CAS | Ativa coluna da DRAM |
| tRCD | Delay RAS → CAS |
| CL | Latência da coluna |
| tRP | Pré-carregar nova linha |
| tRAS | Tempo mínimo de linha ativa |
| Refresh | Recarrega capacitores da DRAM |
| Sense Amplifier | Detecta o valor 0/1 do capacitor |
| Burst | Transferir vários bytes seguidos |
| Address Lines | Linhas físicas de endereço no chip |
| Stall | Parada do pipeline |
| Associatividade | Quantidade de vias por conjunto |
| Tag | Identifica o bloco armazenado |
| Index | Seleciona o conjunto |
| Offset | Seleciona o byte dentro da linha |
| Conflict Miss | Miss por colisão de endereço |
| Cache Thrashing | Substituições sucessivas das mesmas linhas |
| Barramento | Caminho físico elétrico para transmissão de dados |
| Write-through | Política onde a escrita vai para a RAM na hora |
| Write-back | Política onde a RAM só recebe a atualização depois |
| Dirty | Dado que foi alterado e difere da RAM |
| MESI | Protocolo de coerência |
| MOESI | Protocolo de coerência |
| MESIF | Protocolo de coerência |
| Coerência | Garantir que os núcleos vejam o mesmo dado |
