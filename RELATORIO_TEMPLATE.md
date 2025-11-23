# Relatório do Projeto 2: Simulador de Memória Virtual

**Disciplina:** Sistemas Operacionais
**Professor:** Lucas Figueiredo
**Data:**

## Integrantes do Grupo

- Mateus Kage Moya – 10332608

- João Vitor Garcia Aguiar Mintz – 10440421

- Yan Andreotti dos Santos – 10439766

- Giovanni Castro – 10435745

---

## 1. Instruções de Compilação e Execução

### 1.1 Compilação

Para compilar o simulador, utilizamos o compilador GCC. Todos os arquivos fonte estão dentro da pasta src/. A compilação é feita com o seguinte comando:
```bash
gcc src/*.c -o simulador
```
   
Esse comando gera o executável chamado simulador, que será utilizado em todas as execuções.
### 1.2 Execução
A execução do simulador exige três parâmetros:
```bash
./simulador <config.txt> <acessos.txt> <fifo|clock>
```

O primeiro arquivo contém os parâmetros do sistema (tamanho da página, número de frames e configuração dos processos).

O segundo arquivo contém a sequência de acessos.

O último parâmetro define qual algoritmo de substituição será utilizado.

Exemplos:
```bash
./simulador tests/config_1.txt tests/acessos_1.txt fifo

./simulador tests/config_1.txt tests/acessos_1.txt clock
```
## 2. Decisões de Design

2.1 Estruturas de Dados

Tabela de Páginas: Cada processo possui uma tabela de páginas implementada implicitamente nos arrays de frames e na fila/lista de substituição. Em vez de criar uma estrutura separada por processo, mantivemos um vetor de frames físicos onde cada entrada guarda pid, pagina e um flag ocupado. Para descobrir se uma página de um processo está carregada, percorremos esse vetor procurando por (pid,pagina). Para o algoritmo Clock mantemos também um R‑bit (referenced) na estrutura circular para saber se a página foi acessada recentemente.

Frames Físicos: Os frames da memória física são representados por um vetor de estruturas com três campos: pid (identificador do processo), pagina (número da página virtual) e ocupado (0 para livre e 1 para ocupado). O acesso a esse vetor é feito por índice, o que facilita encontrar frames livres e atualizar os mapeamentos. Optamos por esse design pela simplicidade de percorrer e atualizar valores em O(n) e por ser suficiente para os casos de teste fornecidos.

Estrutura para FIFO: Para implementar o algoritmo FIFO usamos uma fila encadeada. Cada nó da fila guarda o índice do frame que foi alocado e um ponteiro para o próximo nó. Quando uma página entra na memória pela primeira vez, inserimos seu frame no final da fila (push). Quando precisamos substituir uma página porque não há frames livres, removemos o elemento no início da fila (pop), pois ele corresponde à página mais antiga
. Essa implementação garante inserção e remoção em tempo constante e reflete exatamente a lógica “primeiro a entrar, primeiro a sair”.

Estrutura para Clock: O algoritmo Clock foi implementado com uma lista circular simples. Cada nó dessa lista contém o número da página e um bit uso que representa o R‑bit. Existe um ponteiro que percorre a lista circularmente; quando ocorre um page fault e não há frame livre, chamamos clock_replace(). Essa função avança o ponteiro: se o bit uso for 1, ele o zera e continua; ao encontrar um nó com uso igual a 0, essa página é selecionada como vítima
. Optamos por guardar apenas o número da página na lista circular e, ao substituir, procuramos o frame correspondente no vetor de frames físicos. Essa escolha simplificou a estrutura e evitou redundância de informação.

### 2.2 Organização do Código
```bash
 simulador.c
├── main()                         - ponto de entrada; lê argumentos e coordena execução
├── lerConfiguracao()              - chamada via utils.c para carregar config
├── lerAcessos()                   - chamada via utils.c para carregar lista de acessos
├── inicializarMemoria()           - inicializa vetor de frames (mmu.c)
├── acessarPaginaFIFO()            - executa lógica do FIFO para cada acesso
├── acessarPaginaCLOCK()           - executa lógica do Clock para cada acesso
└── liberarMemoria()               - libera estruturas alocadas no fim da execução

mmu.c
├── inicializarMemoria()           - monta vetor de frames e reseta estruturas
├── liberarMemoria()               - libera FIFO/Clock e zera estruturas
├── acessarPaginaFIFO()            - trata HIT, PF e substituição com FIFO
├── acessarPaginaCLOCK()           - trata HIT, PF e substituição com Clock
├── procurarPagina()               - busca (pid,pagina) nos frames
├── encontrarFrameLivre()          - retorna índice do primeiro frame desocupado
└── carregarPaginaNoFrame()        - atualiza frame, imprime ações e seta R-bit

FIFO.c
├── criarFila()                    - cria estrutura da fila encadeada
├── vazia()                        - verifica se a fila está vazia
├── push()                         - adiciona frame ao final da fila
├── pop()                          - remove frame mais antigo (vítima)
└── liberaFila()                   - libera a fila inteira

CLOCK.c
├── clock_criar()                  - cria lista circular para o Clock
├── clock_push()                   - insere página enquanto há frames livres
├── clock_replace()                - executa segunda chance e escolhe vítima
└── clock_libera()                 - desaloca lista circular

utils.c
├── lerConfiguracao()              - interpreta config.txt (frames, page size, processos)
├── lerAcessos()                   - lê pares (PID, endereço) do arquivo de acessos
└── validarLinha()                 - checa formatação e auxilia na leitura
```

### 2.3 Algoritmo FIFO

O algoritmo FIFO (First In First Out) mantém uma fila com a ordem de chegada das páginas carregadas na memória. Quando ocorre um page fault e ainda existem frames livres, a página é simplesmente alocada no primeiro frame livre e o índice desse frame é inserido no final da fila. Caso a memória esteja cheia, removemos o índice no início da fila (a página mais antiga) e usamos esse frame para carregar a nova página. A estrutura de fila garante que a ordem de entrada seja preservada e simplifica a escolha da vítima.

### 2.4 Algoritmo Clock

O algoritmo Clock é uma melhoria do FIFO que leva em consideração o histórico de uso das páginas. Ele usa uma lista circular de páginas com um ponteiro que percorre essa lista. Cada página possui um R‑bit que indica se ela foi acessada recentemente. Quando precisamos substituir uma página, avançamos o ponteiro:

Se a página apontada tem R = 1, zeramos esse bit (dando uma “segunda chance”) e seguimos para o próximo nó.

Se a página tem R = 0, ela é escolhida como vítima e é substituída pela nova página.

Depois de carregar a nova página, seu R‑bit é inicializado como 1, pois ela acabou de ser acessada. Esse mecanismo evita remover páginas que foram usadas há pouco tempo e, por isso, costuma produzir menos page faults.

### 2.5 Tratamento de Page Fault

Quando um processo acessa um endereço virtual, o simulador calcula a página e verifica se essa página está nos frames físicos. Se estiver, ocorre um HIT e apenas atualizamos o R‑bit (no caso do Clock). Se não estiver, ocorre um PAGE FAULT. O tratamento segue dois cenários:

Frame livre disponível: percorremos o vetor de frames para encontrar a primeira posição livre (ocupado = 0). A página é carregada nesse frame, o R‑bit é setado para 1 e, conforme o algoritmo, inserimos o frame na fila FIFO ou a página na lista do Clock.

Memória cheia: se todos os frames estão ocupados, o simulador precisa escolher uma página vítima. No FIFO removemos o elemento do início da fila e reutilizamos esse frame. No Clock usamos clock_replace() para percorrer a lista circular até encontrar uma página com R‑bit 0. A página vítima é desalocada, a nova página ocupa o frame e seu R‑bit é setado para 1. Em ambos os casos o simulador imprime na saída qual página foi removida e qual foi carregada.
## 3. Análise Comparativa FIFO vs Clock

### 3.1 Resultados dos Testes

| Descrição do Teste              | Total de Acessos | Page Faults FIFO | Page Faults Clock | Diferença |
| ------------------------------- | ---------------- | ---------------- | ----------------- | --------- |
| **Teste 1 – Básico**            | 8                | 5                | 5                 | 0         |
| **Teste 2 – Memória Pequena**   | 10               | 10               | 10                | 0         |
| **Teste 3 – Simples**           | 7                | 4                | 4                 | 0         |
| **Teste 4 – Múltiplos Padrões** | 100              | 39               | 38                | −1        |
| **Teste 5 – Stress**            | 150              | 78               | 77                | −1        |
| **Teste 6 – Segunda Chance**    | 41               | 8                | 8                 | 0         |


### 3.2 Análise

Com base nos resultados acima, responda:

1. **Qual algoritmo teve melhor desempenho (menos page faults)?**

Nos testes mais simples (1, 2, 3 e 6) ambos os algoritmos apresentaram o mesmo número de page faults, pois havia frames suficientes ou o padrão de acessos não forçou substituições. Nos testes com competição por frames (4 e 5), o algoritmo Clock teve um page fault a menos que o FIFO
. Portanto, o Clock mostrou desempenho ligeiramente superior. 

2. **Por que você acha que isso aconteceu?** 

O Clock considera se a página foi acessada recentemente por meio do R‑bit. Páginas com R = 1 recebem uma segunda chance antes de serem substituídas, o que evita remover páginas que ainda fazem parte do conjunto ativo do processo. 
O FIFO, por outro lado, remove sempre a página mais antiga, mesmo que ela esteja sendo muito usada, podendo provocar um page fault desnecessário

3. **Em que situações Clock é melhor que FIFO?**

O Clock tende a ter melhor desempenho quando há localidade temporal, ou seja, quando o padrão de acessos volta a utilizar páginas recentemente referenciadas. Por exemplo, em uma sequência onde um conjunto pequeno de páginas é acessado repetidamente intercalado com acessos ocasionais a páginas novas, o Clock preserva essas páginas “quentes” enquanto o FIFO poderia removê‑las prematuramente.

4. **Houve casos onde FIFO e Clock tiveram o mesmo resultado?**

Sim, nos testes 1, 2, 3 e 6 tanto FIFO quanto Clock produziram a mesma quantidade de page faults. Isso ocorreu porque havia frames suficientes para acomodar todas as páginas ativas ou porque os acessos não criaram competição intensa; assim, a política de substituição não foi exercida ou não afetou o resultado.

5. **Qual algoritmo você escolheria para um sistema real e por quê?**

Escolheríamos o algoritmo Clock porque ele introduz apenas um pequeno custo adicional (armazenar o R‑bit e percorrer a lista circular), mas em troca reduz a probabilidade de remover páginas recém‑utilizadas. Na prática, sistemas reais usam políticas semelhantes (Clock, LRU aproximado) porque oferecem melhor desempenho médio do que FIFO, que pode sofrer da anomalia de Belady. Contudo, em sistemas com recursos muito limitados ou em que a simplicidade absoluta é necessária, o FIFO ainda pode ser aceitável.

---

## 4. Desafios e Aprendizados

### 4.1 Maior Desafio Técnico

O maior desafio do grupo foi implementar o algoritmo Clock de forma correta e eficiente. Inicialmente pensamos em armazenar todas as informações (PID, número da página e R‑bit) no próprio nó da lista circular, mas isso criava redundância com o vetor de frames. Optamos por guardar apenas o número da página e, ao substituir, buscar o frame correspondente no vetor. Foi necessário cuidar para avançar o ponteiro circular corretamente, zerar o R‑bit ao dar a segunda chance e reconfigurar o ponteiro após cada substituição. Também tivemos dificuldades em garantir que o R‑bit fosse setado em todo acesso (tanto em hits quanto em page faults). Para superar essas dificuldades, escrevemos casos de teste próprios com poucos frames e poucas páginas, imprimindo o estado da memória e dos R‑bits a cada acesso. Isso nos permitiu identificar quando o ponteiro “dava voltas” sem encontrar uma vítima ou quando o R‑bit não era atualizado.

### 4.2 Principal Aprendizado

Antes deste projeto, nossa compreensão de memória virtual era majoritariamente teórica. Implementar a tradução de endereços e a detecção de hits e page faults na prática nos ajudou a consolidar o conceito de tabelas de páginas e a diferença entre páginas virtuais e frames físicos. Ao programar o FIFO, vimos como uma política simples pode ter comportamento previsível, mas também percebemos suas limitações. Com o Clock entendemos de forma concreta como o R‑bit funciona como um marcador de uso recente e como isso influencia a escolha das vítimas. Além disso, o processo de comparar os algoritmos nos mostrou que pequenas melhorias na política de substituição podem resultar em economia de page faults, especialmente em padrões de acesso com localidade temporal.

---

## 5. Vídeo de Demonstração

[**Link do vídeo:** Para enviar](https://youtu.be/CN2p2OqaVlQ)

### Conteúdo do vídeo:

Confirme que o vídeo contém:

- [X] Demonstração da compilação do projeto
- [X] Execução do simulador com algoritmo FIFO
- [X] Execução do simulador com algoritmo Clock
- [X] Explicação da saída produzida
- [X] Comparação dos resultados FIFO vs Clock
- [X] Breve explicação de uma decisão de design importante

---

## Checklist de Entrega

Antes de submeter, verifique:

- [X] Código compila sem erros conforme instruções da seção 1.1
- [X] Simulador funciona corretamente com FIFO
- [X] Simulador funciona corretamente com Clock
- [X] Formato de saída segue EXATAMENTE a especificação do ENUNCIADO.md
- [X] Testamos com os casos fornecidos em tests/
- [X] Todas as seções deste relatório foram preenchidas
- [X] Análise comparativa foi realizada com dados reais
- [X] Vídeo de demonstração foi gravado e link está funcionando
- [X] Todos os integrantes participaram e concordam com a submissão

---
## Referências

## Comentários Finais

