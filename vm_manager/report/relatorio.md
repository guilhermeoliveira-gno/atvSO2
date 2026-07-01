# Relatório Técnico
**Trabalho Prático 2: Simulador de Memória Virtual**  
**Disciplina:** Sistemas Operacionais II  
**Integrantes Responsáveis:** Eduardo, Giovana, Guilherme  

---

## 1. Introdução ao Tema

A tradução de endereços é o mecanismo central que possibilita a virtualização de memória, permitindo que processos executem em um espaço de endereçamento lógico isolado da memória física real. Neste projeto, simulamos um sistema com:
* **Espaço de Endereçamento Virtual:** $2^{16} = 65.536$ bytes (endereços de 16 bits).
* **Tamanho de Página e Quadro:** 256 bytes (bits de deslocamento: 8).
* **Memória Física:** 32.768 bytes (128 quadros).
* **Tabela de Páginas:** 256 entradas (mapeamento direto).

O objetivo principal deste simulador é demonstrar, na prática, os passos envolvidos na resolução de endereços utilizando Tabela de Páginas, Translation Lookaside Buffer (TLB) e paginação por demanda com um Backing Store.

---

## 2. Descrição do Desenvolvimento

O desenvolvimento do simulador foi dividido em módulos focados em componentes específicos da arquitetura. A seguir, detalhamos as decisões de projeto e as dificuldades encontradas em cada componente.

### 2.1. Extração de Página e Deslocamento (Offset)
Os endereços fornecidos nos arquivos de teste possuem 32 bits em nível de sistema, porém a arquitetura simulada utiliza apenas 16 bits. A segmentação do endereço lógico ocorre da seguinte forma:
* **Passo 1 (Máscara):** Isolam-se os 16 bits da extrema direita utilizando uma operação lógica `AND` com a máscara `0xFFFF`.
* **Passo 2 (Número da Página):** Desloca-se o endereço mascarado em 8 bits para a direita (`>> 8`) e aplica-se uma máscara de 8 bits (`& 0xFF`).
* **Passo 3 (Offset):** Aplica-se uma máscara nos 8 bits menos significativos (`& 0xFF`).

```c
int logical_address_masked = logical_address & 0xFFFF;
int page = (logical_address_masked >> 8) & 0xFF;
int offset = logical_address_masked & 0xFF;
```

O cálculo do endereço físico resultante utiliza o quadro alocado (`frame`) e o deslocamento original (`offset`):
$$\text{Endereço Físico} = (\text{quadro} \times \text{Tamanho do Quadro}) + \text{offset}$$
```c
int physical_address = (frame * FRAME_SIZE) + offset;
```

---

### 2.2. Estrutura e Gerenciamento da Tabela de Páginas
A tabela de páginas foi implementada como um array estático de structs com 256 entradas, onde cada entrada contém o número do quadro físico associado, um bit de validade e metadados para a substituição de página (algoritmo de envelhecimento/LRU aproximado):

```c
typedef struct {
    int frame;
    int valid;
    unsigned char reference_bit;
    unsigned char aging_counter;
} page_table_entry_t;
```

As principais operações desenvolvidas para esta estrutura foram:
1. **`page_table_lookup`**: Consulta direta pelo índice da página. Se `valid == 1`, retorna o número do quadro (`frame`). Caso contrário (miss), retorna `-1` para disparar a falta de página.
2. **`page_table_update`**: Chamada quando uma página é carregada na memória física. Associa a página ao quadro físico correspondente, define o bit de validade como `1`, o bit de referência como `1` e inicializa o contador de envelhecimento (`aging_counter`) com `0`.
3. **`page_table_invalidate`**: Chamada quando a página associada a um quadro é selecionada para substituição. A entrada é redefinida com `valid = 0`, `frame = -1`, `reference_bit = 0` e `aging_counter = 0`.
4. **`page_table_set_reference`**: Marca o `reference_bit = 1` a cada acesso à página, sinalizando atividade recente.

---

### 2.3. Algoritmo de Envelhecimento (Aging Algorithm)
Para apoiar a política de substituição LRU aproximada (desenvolvida em conjunto com a Giovana), a função `page_table_update_aging` atualiza periodicamente os contadores das páginas residentes na memória física da seguinte forma:
1. O contador de envelhecimento (`aging_counter`) de 8 bits é deslocado 1 bit para a direita.
2. O valor atual do bit de referência da página é inserido no bit mais significativo (MSB) do contador (ou seja, realiza-se uma operação `OR` com `0x80` se o bit estiver ativo).
3. O bit de referência é zerado.

```c
void page_table_update_aging(void)
{
    for (int i = 0; i < PAGE_TABLE_SIZE; i++) {
        if (page_table[i].valid) {
            page_table[i].aging_counter >>= 1;
            if (page_table[i].reference_bit) {
                page_table[i].aging_counter |= 0x80;
            }
            page_table[i].reference_bit = 0;
        }
    }
}
```
Esta técnica permite registrar o histórico recente de acessos nos 8 períodos anteriores de forma compacta e rápida, permitindo que a página menos usada recentemente seja removida no page fault (aquela com o menor valor inteiro em seu `aging_counter`).

---

### 2.4. Acesso ao Backing Store
Quando ocorre uma falta de página (*page fault*), a página requerida é recuperada do arquivo `BACKING_STORE.bin`. Este arquivo simula a memória secundária (disco) e possui tamanho total correspondente ao espaço virtual ($65.536$ bytes).

A leitura do arquivo foi implementada utilizando acesso aleatório via chamadas padrão da biblioteca C:
1. **Posicionamento (`fseek`)**: O ponteiro do arquivo é movido para o byte inicial da página desejada: $\text{posição} = \text{page} \times \text{PAGE\_SIZE}$.
2. **Leitura (`fread`)**: Lêem-se `PAGE_SIZE` (256) bytes do arquivo diretamente para o array que representa o quadro na memória física.

```c
if (fseek(backing, page * PAGE_SIZE, SEEK_SET) != 0) {
    fprintf(stderr, "Erro no fseek do BACKING_STORE.\n");
    exit(1);
}

if (fread(physical_memory[frame], sizeof(signed char), PAGE_SIZE, backing) != PAGE_SIZE) {
    fprintf(stderr, "Erro no fread do BACKING_STORE.\n");
    exit(1);
}
```


## 2.5. Translation Lookaside Buffer (TLB)

Foi implementado o módulo responsável pelo **Translation Lookaside Buffer (TLB)**, utilizado como uma memória cache para armazenar traduções recentes entre páginas virtuais e quadros físicos. O objetivo do TLB é reduzir a quantidade de consultas realizadas à Tabela de Páginas, melhorando o desempenho da tradução de endereços.

A estrutura foi implementada como um vetor estático de entradas, onde cada posição armazena o número da página, o quadro físico correspondente e um indicador de validade.

```c
typedef struct {
    int page;
    int frame;
    int valid;
} tlb_entry_t;
```

Além disso, foi utilizada uma variável auxiliar (`fifo_next`) para controlar a política de substituição FIFO (First-In, First-Out).

```c
static int fifo_next = 0;
```

### Busca no TLB

A função `tlb_lookup` percorre todas as entradas válidas do TLB procurando pela página solicitada.

Caso a página seja encontrada, o quadro físico correspondente é retornado imediatamente. Caso contrário, a função retorna `-1`, indicando um **TLB Miss**.

```c
int tlb_lookup(int page)
{
    for (int i = 0; i < TLB_SIZE; i++) {
        if (tlb[i].valid && tlb[i].page == page) {
            return tlb[i].frame;
        }
    }

    return -1;
}
```

### Inserção utilizando FIFO

A inserção de novas traduções é realizada pela função `tlb_insert`.

A implementação considera três situações:

1. Atualização do quadro físico caso a página já esteja presente no TLB;
2. Utilização de uma posição livre, caso exista;
3. Substituição utilizando a política FIFO quando todas as posições estiverem ocupadas.

Quando o TLB está cheio, a entrada indicada por `fifo_next` é substituída e o índice é atualizado de forma circular.

```c
tlb[fifo_next].page = page;
tlb[fifo_next].frame = frame;
tlb[fifo_next].valid = 1;

fifo_next = (fifo_next + 1) % TLB_SIZE;
```

### Remoção de Entradas

Também foi implementada a função `tlb_remove`, responsável por invalidar uma entrada quando a página correspondente deixa de estar presente na memória física.

A remoção consiste em marcar a entrada como inválida e redefinir seus campos.

```c
tlb[i].valid = 0;
tlb[i].page = -1;
tlb[i].frame = -1;
```

---

## 2.6. Coleta de Estatísticas

Também foi desenvolvido o módulo responsável pela coleta das estatísticas de execução do simulador.

Foram implementadas funções para contabilizar:

- número total de endereços traduzidos;
- número de **Page Faults**;
- número de **TLB Hits**.

Cada contador é atualizado durante a execução do simulador por funções específicas.

```c
void count_address(void)
{
    total_addresses++;
}

void count_page_fault(void)
{
    page_faults++;
}

void count_tlb_hit(void)
{
    tlb_hits++;
}
```

Ao final da execução, são calculadas as taxas de **Page Fault** e **TLB Hit**, utilizando como base o número total de endereços traduzidos.

```c
page_fault_rate = (double) page_faults / total_addresses;
tlb_hit_rate = (double) tlb_hits / total_addresses;
```

Essas métricas permitem avaliar o desempenho do simulador e medir a eficiência do TLB durante a tradução de endereços.



## 2.7. Gerenciamento da Memória Física e Tratamento de Page Faults

O gerenciamento da memória física é responsável por alocar os quadros disponíveis para as páginas carregadas do arquivo `BACKING_STORE.bin`. Para isso, utilizamos o mapeamento direto através de um array `frame_to_page`, onde cada índice corresponde a um quadro físico (de `0` a `127`) e armazena o número da página nele contida (ou `-1` se o quadro estiver livre).

Quando ocorre um *Page Fault*, o fluxo do gerenciamento obedece às seguintes etapas na função `handle_page_fault`:
1. **Busca por quadro livre:** Utilizamos a função auxiliar `find_free_frame` para percorrer `frame_to_page` e encontrar o primeiro quadro disponível.
2. **Substituição de página (se necessário):** Caso a memória física esteja cheia (`frame == -1`), a função `select_victim_page` é invocada para determinar qual página deverá ser removida, seguindo a política de substituição (LRU Aproximado).
3. **Invalidação e Atualização:** A página escolhida como vítima tem seus mapeamentos invalidados tanto na Tabela de Páginas (`page_table_invalidate`) quanto na TLB (`tlb_remove`).
4. **Leitura e Atualização:** Por fim, a nova página é carregada do disco para a memória física no quadro alocado, e o vetor `frame_to_page` bem como a Tabela de Páginas são atualizados para refletir o novo mapeamento.

---

## 2.8. Seleção de Vítima com LRU Aproximado

O algoritmo de envelhecimento (Aging), cujos contadores são atualizados a cada ciclo, é consumido pelo algoritmo de substituição de páginas para simular de maneira eficiente o comportamento do *Least Recently Used* (LRU).

A escolha da página vítima ocorre na função `select_victim_page`:
* A função itera sobre todos os quadros ocupados da memória física.
* Para cada página presente, o seu `aging_counter` (contador de envelhecimento de 8 bits) é consultado via Tabela de Páginas.
* A página que apresentar o **menor valor no contador** (ou seja, aquela cujo histórico de bits de referência nos últimos ciclos representa o valor mais baixo e, portanto, menos acessada recentemente) é eleita como a vítima para dar lugar à nova página.

```c
int select_victim_page(void)
{
    int victim_page = -1;
    unsigned int min_aging = 256; 
    
    for (int i = 0; i < NUM_FRAMES; i++) {
        int page = frame_to_page[i];
        if (page != -1) {
            unsigned char aging = page_table_get_aging_counter(page);
            if (victim_page == -1 || aging < min_aging) {
                min_aging = aging;
                victim_page = page;
            }
        }
    }
    
    return victim_page;
}
```

---

## 3. Resultados Obtidos

Durante a validação funcional, o simulador foi testado utilizando os arquivos de entrada gerados pelo script Python fornecido (`addresses_random.txt` e `addresses_location.txt`). Os testes demonstraram que a lógica de isolamento de bits, a gerência da TLB via FIFO e a substituição de páginas baseada no algoritmo Aging (LRU Aproximado) operam de maneira correta para virtualizar a memória de 65.536 bytes em apenas 32.768 bytes físicos.

O simulador computou as estatísticas finais com precisão, contabilizando apropriadamente a taxa de erros de página (Page Faults) e a taxa de acertos na TLB (TLB Hits), comprovando a robustez da implementação tanto para cenários de acessos randômicos quanto com forte localidade de referência.

---

## 4. Conclusão

O desenvolvimento deste simulador permitiu consolidar de forma prática os conceitos de virtualização de memória, paginação sob demanda e algoritmos de substituição lecionados na disciplina. A arquitetura descentralizada das estruturas em C exigiu cuidado na integração do Backing Store com a Tabela de Páginas e TLB, garantindo que o estado de cada quadro se mantivesse consistente mesmo sob sucessivas faltas de página. O uso do bit de referência e do contador de envelhecimento demonstrou ser uma técnica eficiente para aproximar o comportamento LRU clássico. O trabalho em equipe e o uso da metodologia SDD asseguraram uma entrega estável e otimizada.

---

## 5. Uso de Inteligência Artificial

O fluxo de desenvolvimento do projeto seguiu os preceitos do Spec-Driven Development (SDD):
1. **Análise e Divisão de Tarefas:** O grupo segmentou o simulador e distribuiu equilibradamente as funcionalidades.
2. **Desenvolvimento Dirigido por Especificação:** A IA foi utilizada para gerar trechos específicos (buscas eficientes em arrays, fseek/fread, bitwise), partindo de prompts restritivos.
3. **Escrita e Revisão:** A IA avaliou e formatou seções do relatório técnico final.

### 5.1. Extração, Envelhecimento e Backing Store (Guilherme)
* **Prompt 1:** *"Em C, como extrair o número de página (bits 15-8) e o offset (bits 7-0) de uma variável inteira de 32 bits `logical_address` usando máscaras bit-a-bit e operadores de deslocamento? Dê também a expressão para recompor o endereço físico a partir do `frame` e do `offset`."*
* **Prompt 2:** *"Escreva uma função em C que percorra um array de estruturas da tabela de páginas. Para cada entrada válida, ela deve deslocar o contador de envelhecimento de 8 bits (`aging_counter`) para a direita por 1, colocar o bit de referência da página no bit mais significativo (MSB) do contador, e depois limpar o bit de referência."*
* **Prompt 3:** *"Como posicionar o ponteiro de leitura em um arquivo binário aberto e ler um bloco contíguo de 256 bytes para uma matriz que representa a memória física, tratando possíveis falhas de leitura ou posicionamento?"*

### 5.2. TLB e Estatísticas (Edu)
* **Prompt 1:** *"Implemente um TLB em C utilizando um vetor de estruturas. A busca deve retornar o quadro físico correspondente à página ou -1 em caso de falha. A inserção deve atualizar entradas existentes, utilizar posições livres quando disponíveis e aplicar a política FIFO quando o TLB estiver cheio."*
* **Prompt 2:** *"Implemente funções em C para registrar estatísticas de um simulador de memória virtual, contabilizando acessos, Page Faults, TLB Hits e calculando suas taxas ao final da execução."*
* **Prompt 3:** *"Revise o fluxo principal de um simulador de memória virtual em C e verifique se a utilização do TLB e das estatísticas está consistente com a especificação do projeto."*

### 5.3. Gestão de Memória Física e Tratamento de Page Fault (Giovana)
* **Prompt 1:** *"Em linguagem C, como implementar a função `handle_page_fault` considerando que eu preciso buscar um quadro livre. Se a memória estiver cheia, preciso selecionar uma vítima, pegar seu quadro associado e usar funções `page_table_invalidate(victim_page)` e `tlb_remove(victim_page)`. Na sequência, faça a leitura da página com fseek e fread e então atualize os arrays `frame_to_page` e a tabela de páginas."*
* **Prompt 2:** *"Como escrever a função `select_victim_page` em C para percorrer o array `frame_to_page` (de 0 a `NUM_FRAMES - 1`), consultar o contador de envelhecimento (0 a 255) de cada página carregada e retornar o número da página que possui o menor valor (ou seja, a candidata a ser removida da memória pelo algoritmo de LRU aproximado)?"*

---

## 6. Repositório no GitHub

O código-fonte deste simulador, contendo todas as implementações e mantendo a arquitetura original exigida, pode ser encontrado publicamente no GitHub. Todos os integrantes colaboraram ativamente com pelo menos 1 (um) commit ao longo do ciclo de desenvolvimento.

**Link do Repositório:** [https://github.com/usuario/atvSO2](https://github.com/usuario/atvSO2) *(Atenção: Lembre-se de substituir este link pelo link real do repositório público do seu grupo!)*
