# Relatório Técnico
**Trabalho Prático 2: Simulador de Memória Virtual**  
**Disciplina:** Sistemas Operacionais II  
**Integrantes Responsáveis:** Eduardo, Giovana, Guilherme  

---

## 1. Introdução à Tradução de Endereços e Tabela de Páginas

A tradução de endereços é o mecanismo central que possibilita a virtualização de memória, permitindo que processos executem em um espaço de endereçamento lógico isolado da memória física real. Neste projeto, simulamos um sistema com:
* **Espaço de Endereçamento Virtual:** $2^{16} = 65.536$ bytes (endereços de 16 bits).
* **Tamanho de Página e Quadro:** 256 bytes (bits de deslocamento: 8).
* **Memória Física:** 32.768 bytes (128 quadros).
* **Tabela de Páginas:** 256 entradas (mapeamento direto).

As responsabilidades deste integrante envolveram a extração correta dos campos de endereço, a gerência da Tabela de Páginas (incluindo o suporte para o algoritmo de envelhecimento) e a leitura sob demanda do armazenamento secundário (`BACKING_STORE.bin`).

---

## 2. Descrição das Estruturas e Algoritmos

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

---

## 3. Metodologia de Uso de IA (Spec-Driven Development)

O fluxo de desenvolvimento do projeto seguiu os seguintes passos organizacionais e metodológicos:

1. **Análise e Divisão de Tarefas:** Primeiramente, o grupo analisou a especificação completa do trabalho contida no PDF para compreender o funcionamento geral do simulador de memória virtual. Em seguida, o projeto foi segmentado em tarefas menores e distribuído equilibradamente entre os três integrantes (Guilherme, Giovana e Eduardo).
2. **Desenvolvimento Dirigido por Especificação (SDD):** Com as responsabilidades divididas, a Inteligência Artificial foi utilizada como ferramenta de suporte para construir os trechos de código C específicos de cada parte, partindo de prompts estruturados que descreviam exatamente os requisitos lógicos esperados.
3. **Escrita e Revisão do Relatório:** O conteúdo conceitual e a descrição das partes foram redigidos de forma manual pelos integrantes. Na etapa final, a IA foi utilizada para avaliar o texto elaborado, propor melhorias na clareza gramatical e técnica, além de verificar e corrigir eventuais erros de consistência lógica nas descrições de algoritmos.

### Prompts Utilizados

#### Prompt 1: Mapeamento de Bits (Tradução)
> **Definição do problema:** Extrair o número de página de 8 bits e o offset de 8 bits de um endereço lógico de 32 bits em C, limitando a análise aos 16 bits mais significativos da direita.
> **Restrições de implementação:** Linguagem C, operações bit-a-bit eficientes.
> **Prompt:** *"Em C, como extrair o número de página (bits 15-8) e o offset (bits 7-0) de uma variável inteira de 32 bits `logical_address` usando máscaras bit-a-bit e operadores de deslocamento? Dê também a expressão para recompor o endereço físico a partir do `frame` e do `offset`."*

#### Prompt 2: Lógica do Algoritmo de Envelhecimento (Aging)
> **Definição do problema:** Implementar a lógica de atualização do contador de envelhecimento (Aging) de 8 bits baseado nos bits de referência das entradas da tabela de páginas.
> **Restrições de implementação:** Operações rápidas com máscara de bits em C, apenas páginas válidas.
> **Prompt:** *"Escreva uma função em C que percorra um array de estruturas da tabela de páginas. Para cada entrada válida, ela deve deslocar o contador de envelhecimento de 8 bits (`aging_counter`) para a direita por 1, colocar o bit de referência da página (`reference_bit`) no bit mais significativo (MSB) do contador, e depois limpar o bit de referência."*

#### Prompt 3: Leitura de Arquivo Binário com Acesso Aleatório
> **Definição do problema:** Ler blocos de 256 bytes de posições arbitrárias dentro de um arquivo binário em C.
> **Restrições de implementação:** Usar `fseek` e `fread`, tratamento de erro integrado.
> **Prompt:** *"Como posicionar o ponteiro de leitura em um arquivo binário aberto e ler um bloco contíguo de 256 bytes para uma matriz que representa a memória física, tratando possíveis falhas de leitura ou posicionamento?"*

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

---

## 3. Complemento da Metodologia de Uso de IA

Durante o desenvolvimento desta etapa do projeto, a Inteligência Artificial foi utilizada como ferramenta de apoio para auxiliar na implementação do módulo do TLB, na construção das funções responsáveis pelas estatísticas e na revisão da integração dessas funcionalidades ao fluxo principal do simulador.

### Prompt 1 – Implementação do TLB

> **Definição do problema:** Implementar um Translation Lookaside Buffer (TLB) utilizando a política FIFO para substituição de entradas.
>
> **Restrições:** Linguagem C, vetor de estruturas e suporte às operações de busca, inserção e remoção.
>
> **Prompt:** *"Implemente um TLB em C utilizando um vetor de estruturas. A busca deve retornar o quadro físico correspondente à página ou -1 em caso de falha. A inserção deve atualizar entradas existentes, utilizar posições livres quando disponíveis e aplicar a política FIFO quando o TLB estiver cheio."*

### Prompt 2 – Estatísticas da Simulação

> **Definição do problema:** Implementar funções para contabilizar o número de endereços traduzidos, Page Faults e TLB Hits, além de calcular as respectivas taxas ao final da execução.
>
> **Prompt:** *"Implemente funções em C para registrar estatísticas de um simulador de memória virtual, contabilizando acessos, Page Faults, TLB Hits e calculando suas taxas ao final da execução."*

### Prompt 3 – Revisão da Integração

> **Definição do problema:** Verificar se o módulo do TLB e o sistema de estatísticas estão corretamente integrados ao fluxo principal de tradução de endereços.
>
> **Prompt:** *"Revise o fluxo principal de um simulador de memória virtual em C e verifique se a utilização do TLB e das estatísticas está consistente com a especificação do projeto."*
