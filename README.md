<div align="center">

# UniCesumar
EDUCAÇÃO PRESENCIAL E A DISTÂNCIA

</div>

---

<div align="center">

YASMIN FERNANDA DE CARVALHO – R.A. 25061121-2

<br><br><br><br><br><br>

**O CAOS DO SHOW DE ROCK: PROCESSOS, THREADS E CONCORRÊNCIA EM UM SISTEMA DE VENDA DE INGRESSOS**

<br><br><br><br><br><br><br><br>

LONDRINA<br>
2026

</div>

---

<div align="center">

YASMIN FERNANDA DE CARVALHO – R.A. 25061121-2

<br><br><br>

**O CAOS DO SHOW DE ROCK: PROCESSOS, THREADS E CONCORRÊNCIA EM UM SISTEMA DE VENDA DE INGRESSOS**

</div>

<br><br>

<div align="right">

Desafio Prático de Sala de Aula Invertida<br>
(Encontro 7 – Arquitetura e Concorrência),<br>
apresentado ao Curso de Engenharia de Software<br>
para obtenção parcial de nota semestral.

</div>

<br><br><br>

<div align="center">

LONDRINA<br>
2026

</div>

---

## SUMÁRIO

[INTRODUÇÃO](#introdução)

[1. DESENVOLVIMENTO](#1-desenvolvimento)
- [1.1. Síntese Teórica](#11-síntese-teórica)
  - [1.1.1. Diferença entre Processo e Thread](#111-diferença-entre-processo-e-thread)
  - [1.1.2. Por que criar threads é mais leve do que criar processos](#112-por-que-criar-threads-é-mais-leve-do-que-criar-processos)
  - [1.1.3. Threads em Modo Usuário e em Modo Núcleo](#113-threads-em-modo-usuário-e-em-modo-núcleo-kernel)
- [1.2. Diagnóstico do Problema](#12-diagnóstico-do-problema-o-gargalo)
  - [1.2.1. O que é uma Condição de Corrida](#121-o-que-é-uma-condição-de-corrida)
  - [1.2.2. O caso do Assento A-15](#122-o-caso-do-assento-a-15)
  - [1.2.3. Consequências para o sistema de ingressos](#123-consequências-para-o-sistema-de-ingressos)
- [1.3. A Solução Arquitetural](#13-a-solução-arquitetural)
  - [1.3.1. Região Crítica e Exclusão Mútua](#131-região-crítica-e-exclusão-mútua)
  - [1.3.2. Como funciona no Assento A-15](#132-como-funciona-no-assento-a-15)
  - [1.3.3. Cuidados de arquitetura para 100.000 usuários](#133-cuidados-de-arquitetura-para-100000-usuários)

[2. CONCLUSÃO](#2-conclusão)

[REFERÊNCIAS BIBLIOGRÁFICAS](#referências-bibliográficas)

---

## INTRODUÇÃO

Este trabalho analisa um estudo de caso fictício: uma empresa de engenharia de software foi contratada para desenvolver o sistema de venda de ingressos online de um show internacional, que deve suportar 100.000 acessos simultâneos no minuto em que as vendas forem abertas. Nesse cenário, muitas rotinas executam ao mesmo tempo e disputam os mesmos dados, o que torna a concorrência o problema central do projeto.

O objetivo é compreender como o sistema operacional organiza essas execuções simultâneas e propor uma solução arquitetural para o principal risco do sistema: dois usuários comprarem a mesma cadeira ao mesmo tempo. Para isso, o trabalho explica a diferença entre processos e threads, discute por que threads são mais adequadas para atender a um grande volume de usuários, apresenta o problema da condição de corrida aplicado ao caso do Assento A-15 e mostra como a exclusão mútua sobre regiões críticas resolve esse problema.

A justificativa está no impacto real do tema: falhas de concorrência em sistemas de venda causam vendas duplicadas, prejuízo financeiro e perda de confiança dos clientes, e costumam aparecer apenas sob carga alta, justamente no momento mais crítico. A metodologia adotada foi a pesquisa bibliográfica, com base no material da Aula 7, em livros clássicos de sistemas operacionais, em materiais de aula de universidades públicas e em vídeos sobre concorrência e paralelismo.

---

## 1. DESENVOLVIMENTO

### 1.1. Síntese Teórica

#### 1.1.1. Diferença entre Processo e Thread

Um **processo** é um programa em execução. Ele não é só o código: o sistema operacional reserva para ele um conjunto próprio de recursos, como o espaço de endereçamento (código, dados, heap e pilha), arquivos abertos, permissões e informações de controle guardadas no Bloco de Controle de Processo (PCB) (TANENBAUM; BOS, 2015). Cada processo é isolado dos demais, ou seja, um processo não enxerga diretamente a memória do outro.

Uma **thread** é uma linha de execução dentro de um processo. Um mesmo processo pode ter várias threads rodando ao mesmo tempo, e todas elas compartilham os recursos desse processo, como memória, arquivos abertos e variáveis globais. O que cada thread tem de exclusivo é apenas o necessário para executar de forma independente: seu contador de programa, seus registradores e sua própria pilha (SILBERSCHATZ; GALVIN; GAGNE, 2018). Por compartilharem o mesmo espaço de endereçamento, as threads são menos independentes entre si do que os processos (ROCHA, 2026).

Uma forma simples de visualizar: o processo é a "empresa", com prédio, equipamentos e arquivos; as threads são os "funcionários" trabalhando dentro do mesmo prédio, usando os mesmos recursos, cada um executando a sua tarefa.

#### 1.1.2. Por que criar threads é mais leve do que criar processos

Para atender 100.000 usuários simultâneos, criar um processo novo para cada acesso seria muito custoso. Ao criar um processo, o sistema operacional precisa montar um novo espaço de endereçamento, configurar tabelas de páginas, criar um novo PCB e copiar ou mapear recursos. Além disso, a troca de contexto entre processos é cara, porque envolve trocar o mapeamento de memória e normalmente invalidar entradas da TLB (o cache de traduções de endereço), o que deixa os primeiros acessos à memória mais lentos depois da troca (MACHADO; MAIA, 2013).

Já as threads de um mesmo processo **compartilham o mesmo espaço de endereçamento**. Por isso, criar uma thread exige basicamente alocar uma pilha e uma estrutura pequena de controle, sem duplicar a memória, e a troca de contexto entre threads do mesmo processo também é mais barata (CAVALHEIRO; BALDASSIN; DU BOIS, 2025). Em algumas situações, criar e destruir threads chega a ser até 100 vezes mais rápido do que fazer o mesmo com processos (ROCHA, 2026). Outra vantagem é a comunicação: como as threads enxergam a mesma memória, elas trocam dados diretamente, sem precisar de mecanismos de comunicação entre processos, como pipes ou sockets (AKITA, 2019b).

No caso do show, isso significa que o servidor consegue atender muito mais requisições com o mesmo hardware usando threads. Na prática, ainda se usa um **pool de threads** (um conjunto fixo de threads reaproveitadas), porque mesmo threads têm custo, e criar 100.000 delas de uma vez também esgotaria a memória.

#### 1.1.3. Threads em Modo Usuário e em Modo Núcleo (Kernel)

**Threads de usuário** são criadas e gerenciadas por uma biblioteca no espaço do usuário, sem que o kernel saiba que elas existem. Para o sistema operacional, o processo inteiro parece ter uma única linha de execução: o núcleo escolhe um processo e é o próprio processo, por meio da biblioteca, que escolhe qual thread vai executar (FERRAZ, 2015; ROCHA, 2026).

- Vantagens: criação e troca entre threads muito rápidas, pois não exigem chamadas ao sistema; o escalonamento pode ser personalizado pela aplicação.
- Desvantagens: se uma thread fizer uma chamada bloqueante (como ler do disco ou da rede), o kernel bloqueia o processo inteiro, e todas as outras threads param junto. Também não aproveitam múltiplos núcleos de CPU, já que o kernel enxerga só um fluxo de execução.

**Threads de núcleo (kernel)** são criadas e gerenciadas diretamente pelo sistema operacional, que as conhece e as escalona individualmente.

- Vantagens: se uma thread bloqueia, o kernel pode executar outra do mesmo processo; várias threads podem rodar em paralelo em núcleos diferentes.
- Desvantagens: criação e troca são mais caras, pois envolvem chamadas ao sistema e passagem para o modo núcleo.

Existem modelos que relacionam os dois tipos: **muitos-para-um** (várias threads de usuário em uma de kernel), **um-para-um** (cada thread de usuário corresponde a uma de kernel, modelo usado pelo Linux e pelo Windows) e **muitos-para-muitos** (várias threads de usuário distribuídas entre várias de kernel) (SILBERSCHATZ; GALVIN; GAGNE, 2018). O material da Aula 7 chama esse último modelo de threads híbridas, citando como exemplos o Solaris até a versão 8, o HP-UX e o Tru64 Unix (ROCHA, 2026). Para um servidor de ingressos que faz muitas operações de rede e banco de dados, threads de kernel ou modelos híbridos são mais adequados, porque uma requisição esperando o banco não trava as outras.

### 1.2. Diagnóstico do Problema (o gargalo)

#### 1.2.1. O que é uma Condição de Corrida

Uma **condição de corrida** (*race condition*) acontece quando duas ou mais threads acessam e modificam um mesmo dado compartilhado ao mesmo tempo, e o resultado final depende da ordem exata em que cada uma executa (TANENBAUM; BOS, 2015). Como o escalonador pode interromper uma thread em qualquer instrução e passar a CPU para outra, essa ordem é imprevisível, e o resultado pode ficar errado (JOHANN, 2011). Na Aula 7, essa situação é descrita como uma disputa pelo recurso, em que o resultado depende de qual processo executa no momento propício (ROCHA, 2026).

#### 1.2.2. O caso do Assento A-15

De forma simplificada, a rotina de compra faz três passos:

```text
1. Ler o status do assento A-15
2. Se status == "LIVRE":
3.     Marcar status = "VENDIDO" e registrar o comprador
```

O problema é que esses passos não acontecem como uma operação única e indivisível. Se o Usuário A e o Usuário B clicam no mesmo milissegundo, uma sequência possível é:

| Tempo | Thread do Usuário A | Thread do Usuário B | Status do A-15 |
|---|---|---|---|
| t1 | Lê status → "LIVRE" | | LIVRE |
| t2 | *(interrompida pelo escalonador)* | Lê status → "LIVRE" | LIVRE |
| t3 | | Marca "VENDIDO" para B | VENDIDO (B) |
| t4 | Marca "VENDIDO" para A | | VENDIDO (A) |

As duas threads viram o assento como livre, as duas seguiram em frente e as duas "venderam" o ingresso. A gravação de A sobrescreveu a de B.

#### 1.2.3. Consequências para o sistema de ingressos

Se a condição de corrida não for tratada:

- **Venda duplicada (*overbooking*):** dois clientes pagam pela mesma cadeira e recebem confirmação. No dia do show, os dois aparecem com ingresso para o A-15.
- **Dados inconsistentes:** o banco registra apenas um dono, mas o sistema de pagamento cobrou os dois. Um dos clientes fica com uma cobrança sem ingresso válido.
- **Prejuízo financeiro e jurídico:** estornos, reclamações em órgãos de defesa do consumidor e possíveis processos.
- **Dano à reputação:** com 100.000 acessos simultâneos, o problema não aconteceria uma vez, mas em centenas de assentos, principalmente nos mais disputados.

O mais perigoso é que esse erro é intermitente: em testes com poucos usuários ele quase nunca aparece e só surge sob carga real, justamente no momento da abertura das vendas.

### 1.3. A Solução Arquitetural

#### 1.3.1. Região Crítica e Exclusão Mútua

A **região crítica** é o trecho do código em que a thread acessa o recurso compartilhado. No caso estudado, é o trecho "ler status → verificar → marcar como vendido". A solução é garantir **exclusão mútua**: enquanto uma thread está dentro da região crítica de um recurso, nenhuma outra pode entrar na região crítica do mesmo recurso (GARCIA, 2017). É a solução apresentada na Aula 7: impedir que mais de um processo leia e escreva em uma variável compartilhada ao mesmo tempo (ROCHA, 2026).

Segundo Tanenbaum e Bos (2015), uma boa solução precisa atender a quatro condições:

1. Duas threads nunca podem estar ao mesmo tempo dentro da mesma região crítica.
2. Não se pode fazer suposições sobre a velocidade ou o número de CPUs.
3. Nenhuma thread fora da região crítica pode bloquear outras threads.
4. Nenhuma thread deve esperar para sempre para entrar na região crítica.

Os mecanismos mais usados para isso são o **mutex** (uma trava com dois estados: livre ou ocupada) e o **semáforo** (um contador que controla quantas threads podem acessar um recurso) (OLIVEIRA; CARISSIMI; TOSCANI, 2010). Também existem instruções de hardware atômicas, como *test-and-set* e *compare-and-swap*, que servem de base para esses mecanismos (CAVALHEIRO; BALDASSIN; DU BOIS, 2025).

#### 1.3.2. Como funciona no Assento A-15

Com um mutex associado ao assento, a rotina fica assim:

```text
adquirir(trava_A15)              // entrada da região crítica
    se status_A15 == "LIVRE":
        status_A15 = "VENDIDO"
        comprador_A15 = usuario
        resultado = SUCESSO
    senão:
        resultado = "ASSENTO INDISPONÍVEL"
liberar(trava_A15)               // saída da região crítica
```

Repetindo o cenário do milissegundo:

| Tempo | Thread do Usuário A | Thread do Usuário B | Status do A-15 |
|---|---|---|---|
| t1 | Adquire a trava | Tenta adquirir → **bloqueada, espera** | LIVRE |
| t2 | Lê "LIVRE", marca VENDIDO (A) | *(esperando)* | VENDIDO (A) |
| t3 | Libera a trava | Adquire a trava | VENDIDO (A) |
| t4 | | Lê "VENDIDO" → recebe "indisponível" | VENDIDO (A) |

Agora só um usuário compra o A-15, e o outro recebe uma mensagem clara para escolher outra cadeira. A ordem entre A e B continua sendo decidida pelo escalonador, mas o resultado é sempre consistente.

#### 1.3.3. Cuidados de arquitetura para 100.000 usuários

- **Granularidade da trava:** usar uma única trava para o show inteiro faria as 100.000 requisições ficarem em fila, uma de cada vez, e o sistema viraria um gargalo. O correto é uma trava **por assento** (ou por setor), para que compras de cadeiras diferentes aconteçam em paralelo e só disputas pelo mesmo assento fiquem sequenciais. Cavalheiro, Baldassin e Du Bois (2025) chamam isso de granularidade fina das seções críticas.
- **Região crítica curta:** o pagamento não deve ficar dentro da região crítica. Uma estratégia comum é **reservar** o assento por alguns minutos dentro da trava e processar o pagamento depois; se o pagamento não for concluído, a reserva expira e o assento volta a ficar livre.
- **Vários servidores:** um sistema desse porte roda em várias máquinas, e um mutex só vale dentro de um processo. Nesse caso, a exclusão mútua precisa ser garantida no ponto que todos compartilham, normalmente o banco de dados, com uma atualização atômica condicional, por exemplo:

```sql
UPDATE assentos
   SET status = 'RESERVADO', usuario_id = ?
 WHERE codigo = 'A-15' AND status = 'LIVRE';
```

  Se o comando afetar 1 linha, a compra foi garantida; se afetar 0, outro usuário chegou antes. O próprio banco aplica a exclusão mútua sobre aquela linha, que é o mesmo conceito de região crítica, aplicado em outra camada.
- **Evitar *deadlock*:** se uma compra envolver vários assentos, as travas devem ser adquiridas sempre na mesma ordem (por exemplo, ordem alfabética do código do assento), para que duas threads não fiquem esperando uma pela outra para sempre.

---

## 2. CONCLUSÃO

O objetivo do trabalho foi alcançado: a partir dos conceitos de processos, threads e concorrência, foi possível explicar o comportamento do sistema de ingressos sob 100.000 acessos simultâneos e propor uma solução para o seu principal risco.

A pesquisa mostrou que threads são a escolha adequada para atender a esse volume, por compartilharem o espaço de endereçamento do processo e, por isso, serem mais leves de criar e de alternar do que processos. Esse mesmo compartilhamento, porém, é o que expõe o sistema à condição de corrida, como demonstrado no caso do Assento A-15, em que dois usuários podem comprar a mesma cadeira.

A metodologia de pesquisa bibliográfica foi suficiente para responder ao problema: a exclusão mútua sobre a região crítica de compra garante que apenas um usuário finalize a compra de um assento. Para que a solução funcione em escala, ela deve ser aplicada com travas por assento, regiões críticas curtas e, em um ambiente com vários servidores, com atualizações atômicas no banco de dados.

---

## REFERÊNCIAS BIBLIOGRÁFICAS

AKITA, Fabio. **[Akitando] #43 – Concorrência e Paralelismo (Parte 1) | Entendendo Back-End para Iniciantes (Parte 3)**. [*S. l.*]: Akitando, 13 mar. 2019a. 1 vídeo. Disponível em: https://akitaonrails.com/2019/03/13/akitando-43-concorrencia-e-paralelismo-parte-1-entendendo-back-end-para-iniciantes-parte-3/. Acesso em: 8 out. 2026.

<br><br>

AKITA, Fabio. **[Akitando] #44 – Concorrência e Paralelismo (Parte 2) | Entendendo Back-end para Iniciantes (Parte 4)**. [*S. l.*]: Akitando, 20 mar. 2019b. 1 vídeo. Disponível em: https://akitaonrails.com/2019/03/20/akitando-44-concorrencia-e-paralelismo-parte-2-entendendo-back-end-para-iniciantes-parte-4/. Acesso em: 8 out. 2026.

<br><br>

CAVALHEIRO, Gerson Geraldo H.; BALDASSIN, Alexandro; DU BOIS, André Rauber. Programação multithread: modelos e abstrações em linguagens contemporâneas. *In*: JORNADAS DE ATUALIZAÇÃO EM INFORMÁTICA, 2025. **[Anais]**. Porto Alegre: Sociedade Brasileira de Computação, 2025. cap. 3. Disponível em: https://books-sol.sbc.org.br/index.php/sbc/catalog/download/173/771/1483?inline=1. Acesso em: 8 out. 2026.

<br><br>

FERRAZ, Carlos. **Sistemas operacionais**: processos/threads. Recife: CIn-UFPE, 2015. Slides de aula. Disponível em: https://www.cin.ufpe.br/~jcbf/if677/2015-1/slides/Aula_05_Threads.pdf. Acesso em: 8 out. 2026.

<br><br>

GARCIA, Islene Calciolari. **MC504 – Sistemas operacionais**: processos e threads: exclusão mútua. Campinas: IC-Unicamp, 2017. Slides de aula. Disponível em: https://www.ic.unicamp.br/~islene/1s2017-mc504/aula03/aula03.pdf. Acesso em: 8 out. 2026.

<br><br>

JOHANN, Marcelo. **INF01151 – Sistemas operacionais II N**: race conditions and POSIX threads. Porto Alegre: UFRGS, 2011. Slides de aula. Disponível em: https://www.inf.ufrgs.br/~johann/sisop2/aula03.race.threads.pdf. Acesso em: 8 out. 2026.

<br><br>

MACHADO, Francis Berenger; MAIA, Luiz Paulo. **Arquitetura de sistemas operacionais**. 5. ed. Rio de Janeiro: LTC, 2013.

<br><br>

OLIVEIRA, Rômulo Silva de; CARISSIMI, Alexandre da Silva; TOSCANI, Simão Sirineo. **Sistemas operacionais**. 4. ed. Porto Alegre: Bookman, 2010.

<br><br>

ROCHA, Leonardo. **Sistemas operacionais**: o modelo de processos. Londrina: UniCesumar, 2026. Material de aula (Aula 7), slides. Disponível em: https://classroom.google.com/c/ODcxMzY4NzY5OTM3/m/ODg5NTQxNTkwMzgw/details. Acesso em: 8 out. 2026.

<br><br>

SILBERSCHATZ, Abraham; GALVIN, Peter Baer; GAGNE, Greg. **Fundamentos de sistemas operacionais**. 9. ed. Rio de Janeiro: LTC, 2018.

<br><br>

TANENBAUM, Andrew S.; BOS, Herbert. **Sistemas operacionais modernos**. 4. ed. São Paulo: Pearson, 2015.