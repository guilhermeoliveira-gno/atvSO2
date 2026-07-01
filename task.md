# Lista de Tarefas - Parte do Guilherme

- `[x]` Extração do número de página e offset (`src/main.c`)
- `[x]` Leitura das páginas a partir de `BACKING_STORE.bin` (`src/memory.c`)
- `[x]` Implementação das funções na Tabela de Páginas (`src/page_table.c`)
  - `[x]` `page_table_lookup`
  - `[x]` `page_table_update`
  - `[x]` `page_table_invalidate`
  - `[x]` `page_table_set_reference`
  - `[x]` `page_table_update_aging`
- `[x]` Compilação e verificação básica (Implementação concluída; ambiente local Windows sem gcc/make para teste direto no terminal)

# Lista de Tarefas - Parte do Edu

- [x] Implementação do TLB FIFO (`src/tlb.c`)
  - [x] `tlb_lookup`
  - [x] `tlb_insert`
  - [x] `tlb_remove`
  - [x] `tlb_clear`
- [x] Integração da tradução de endereços (`src/main.c`)
  - [x] Consulta ao TLB antes da Tabela de Páginas
  - [x] Inserção no TLB após Page Table/Page Fault
  - [x] Contabilização de TLB Hits e Page Faults
  - [x] Cálculo do endereço físico
- [x] Implementação das estatísticas (`src/statistics.c`)
  - [x] `count_address`
  - [x] `count_page_fault`
  - [x] `count_tlb_hit`
  - [x] Cálculo de `Page Fault Rate`
  - [x] Cálculo de `TLB Hit Rate`
- [x] Compilação e verificação básica (`make` executado sem erros)

# Lista de Tarefas - Parte da Giovana

- [x] Gerenciamento da memória física (`src/memory.c`)
  - [x] Mapeamento de quadros via `frame_to_page`
  - [x] Busca por quadro livre (`find_free_frame`)
- [x] Algoritmo de substituição aging / LRU aproximado (`src/memory.c`)
  - [x] Função de seleção de vítima (`select_victim_page`) checando o menor contador
- [x] Tratamento de page faults integrado com TLB (`src/memory.c`)
  - [x] Implementação central em `handle_page_fault`
  - [x] Invalidação na Tabela de Páginas da página vítima
  - [x] Remoção da TLB da página vítima
  - [x] Leitura de disco e atualização correta
- [x] Redação do relatório (Seções de memória, page faults e prompts da IA)
