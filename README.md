# Busca Estocástica no Labirinto — Simulated Annealing

> Projeto desenvolvido para a disciplina de **Análise de Algoritmos e Inteligência Artificial**.
> Robôs partem do centro de um labirinto e aprendem a encontrar suas saídas usando **Simulated Annealing** — um algoritmo de busca estocástica inspirado no processo físico de recozimento de metais.

<br>

## Demonstração

A simulação roda diretamente no **Google Colab** e abre uma visualização interativa em nova aba do navegador, com animação fluida a 60fps, controles de Play/Pause/Reset e slider de iteração.

```
▶ Play  ⏸ Pause  ⏹ Reset    Velocidade: ━━●━━    Iter: 1234 / 5000

[ Labirinto 90×90 com os robôs navegando ]   [ Barras de progresso de aprendizado ]
```

<br>

## Como executar

**Pré-requisito:** conta Google (para usar o Colab).

1. Abra o notebook no Google Colab
2. Execute os blocos em ordem: **Bloco 1 → 2 → 3 → 4 → 5 → 6**
3. O Bloco 6 abre a visualização interativa em uma nova aba automaticamente

> **Sem instalação local necessária.** Todas as dependências já estão disponíveis no Colab.

<br>

## Ferramentas e Bibliotecas

| Ferramenta | Versão | Uso no projeto |
|---|---|---|
| **Python** | 3.10+ | Linguagem principal |
| **Google Colab** | — | Ambiente de execução em nuvem |
| **NumPy** | ≥1.23 | Representação e manipulação do grid (matriz N×N) |
| **Matplotlib** | ≥3.5 | Visualização estática do labirinto (Blocos 1–5) |
| **Flask** | ≥2.0 | Servidor web local para servir a animação interativa |
| **HTML5 Canvas** | — | Renderização da animação a 60fps no navegador |
| **JavaScript** | ES6+ | Loop de animação com `requestAnimationFrame` |
| **Google Colab Output API** | — | Abertura da visualização em nova aba (`serve_kernel_port_as_window`) |
| **Threading** | stdlib | Execução do Flask em thread separada sem bloquear o notebook |
| **Collections.deque** | stdlib | Fila do BFS para verificação de conectividade |
| **Colorsys** | stdlib | Geração automática de cores distintas para N robôs |

<br>

## Estrutura do Notebook

O notebook está organizado em **6 blocos sequenciais**, cada um com responsabilidade única:

```
Bloco 1 — Geração do Labirinto
    Cria o grid N×N, distribui paredes aleatoriamente,
    posiciona saídas nas bordas e valida conectividade via BFS.

Bloco 2 — Representação da Solução
    Define como um "caminho" é codificado: sequência de direções
    [↑↓←→] e as operações de perturbação (substituição, inversão, inserção).

Bloco 3 — Função Objetivo
    Define o custo de uma solução — o que o SA tenta minimizar.
    Usa Distância de Manhattan como heurística de proximidade.

Bloco 4 — Núcleo do Simulated Annealing (1 robô)
    Implementa o loop principal do SA com critério de Metropolis
    e resfriamento geométrico. Teste isolado para 1 robô.

Bloco 5 — N Robôs Independentes
    Instancia N agentes SA, cada um com sua saída exclusiva
    e temperatura inicial diferente. Registra histórico completo.

Bloco 6 — Visualização Interativa
    Serializa os históricos em JSON, sobe um servidor Flask,
    e serve uma página HTML5 com Canvas animado em nova aba.
```

<br>

## O Labirinto

Representado como uma **matriz N×N de inteiros**:

| Valor | Símbolo | Significado |
|---|---|---|
| `0` | ░ | Célula livre |
| `1` | █ | Parede |
| `2` | S | Saída |
| `3` | ★ | Ponto de início (centro do grid) |

**Geração em 5 passos:**
1. Inicializar grid totalmente livre — `O(N²)`
2. Fixar o centro como ponto de partida
3. Escolher posições de saída nas bordas
4. Preencher paredes por sorteio de Bernoulli — `O(N²)`
5. Verificar conectividade com BFS e abrir corredores se necessário — `O(N²)`

<br>

## 🤖 Representação da Solução

Cada robô carrega uma **sequência de movimentos** (seu "cromossomo"):

```python
solucao = [1, 3, 1, 0, 3, 3, 1, 2]
#          ↓  →  ↓  ↑  →  →  ↓  ←
```

**Três operações de perturbação** aplicadas pelo SA a cada iteração:

| Operação | O que faz | Intensidade |
|---|---|---|
| **Substituição** | Troca 1 movimento por outro diferente | Mínima — exploração local |
| **Inversão de trecho** | Inverte a ordem de um segmento aleatório | Moderada — desfaz loops |
| **Inserção / Remoção** | Adiciona ou remove 1 movimento | Flexível — ajusta comprimento |

<br>

## Função Objetivo

O SA minimiza um **custo** calculado em dois regimes:

**Regime 1 — Robô NÃO chegou à saída:**
```
custo = 10.000 + 50 × d_manhattan(posição_final, saída_mais_próxima)
```

**Regime 2 — Robô chegou à saída:**
```
custo = passos_válidos − 500
```

A separação de ~10.500 pontos entre os regimes garante que qualquer caminho que chega vale mais que qualquer caminho que não chega.

**Distância de Manhattan** — heurística exata para movimentos ortogonais:
```
d(A, B) = |linhaA − linhaB| + |colunaA − colunaB|
```

<br>

## O Algoritmo — Simulated Annealing

Inspirado no processo físico de **recozimento de metais** (Kirkpatrick et al., 1983).

### Pseudocódigo

```
solução ← sequência aleatória de movimentos
T ← temperatura_inicial

enquanto T > T_mínima e iterações < MAX_IT:
    candidato ← perturbar(solução)
    Δ ← custo(candidato) − custo(solução)

    se Δ < 0:                    # melhoria → aceita sempre
        solução ← candidato
    senão:                        # piora → aceita com probabilidade
        p ← e^(−Δ / T)
        se random() < p:
            solução ← candidato

    T ← T × α                   # resfriamento geométrico
```

### Critério de Metropolis

A chave do SA é **aceitar soluções piores** com probabilidade decrescente:

```
P(aceitar piora) = e^(−Δ/T)
```

| Temperatura | P(aceitar piora de Δ=100) | Comportamento |
|---|---|---|
| T = 500 (início) | e^(−0.2) ≈ **82%** | Exploração intensa |
| T = 100 | e^(−1.0) ≈ **37%** | Equilíbrio |
| T = 5 (fim) | e^(−20) ≈ **0%** | Apenas aceita melhorias |

Isso permite escapar de **mínimos locais** — armadilhas onde um algoritmo guloso ficaria preso.

### Resfriamento Geométrico

A temperatura segue a recorrência:

```
T(k) = T(k−1) × α      ⟹      T(k) = T₀ × αᵏ
```

Com `α = 0.9985` e `T₀ = 500`: após 5.000 iterações, `T ≈ 0.08` — abaixo do limiar mínimo de `0.1`, o SA para naturalmente.

<br>

## Parâmetros Configuráveis

```python
TAMANHO_GRID      = 90      # dimensão N do labirinto (N×N células)
DENSIDADE_PAREDES = 0.40    # fração de células que viram paredes
NUM_ROBOS         = 6       # número de agentes e saídas
NUM_SAIDAS        = 6       # deve ser igual a NUM_ROBOS

ALPHA  = 0.9985             # taxa de resfriamento geométrico
MAX_IT = 5000               # máximo de iterações por robô
```

**Semente aleatória:** usa `time.time()` por padrão — cada execução gera um labirinto diferente. Para reproduzir um resultado específico, substitua por `SEMENTE = <número>`.

<br>

## Complexidade Assintótica

| Componente | Tempo | Espaço |
|---|---|---|
| Geração do labirinto | `Θ(N²)` | `Θ(N²)` |
| BFS de conectividade | `O(N²)` | `O(N²)` |
| Avaliação de uma solução | `O(N²)` | `O(N²)` |
| SA completo — 1 robô | `O(MAX_IT × N²)` | `O(MAX_IT)` |
| Simulação — K robôs | `O(K × MAX_IT × N²)` | `O(K × MAX_IT)` |

**Comparação com busca exaustiva:** o espaço de soluções tem `4^(N²)` possibilidades — para N=90, isso é um número com ~4.300 dígitos. Completamente intratável. O SA encontra boas soluções em tempo polinomial usando aleatoriedade controlada.

### Número real de operações neste projeto:

```
K=6 robôs × MAX_IT=5.000 × N²=8.100 ≈ 243.000.000 operações
```

Executado em ~30–60 segundos no Colab.

<br>

## A Visualização

A animação mostra:

- **Labirinto 90×90** com caminhos dos robôs pintados célula a célula, com gradiente de opacidade (rastro que vai sumindo no passado)
- **Emoji 🚤** na posição atual de cada robô, com sombra para destacar do fundo
- **Barras de progresso** oscilando em tempo real — quando o SA aceita uma piora pelo critério de Metropolis, a barra **cai** visivelmente
- **Temperatura atual** exibida abaixo de cada barra enquanto o robô ainda busca

**Arquitetura da visualização:**
```
Python (Colab)
    → serializa historicos[] em JSON (~5 MB)
    → Flask serve a página HTML na porta 5000
    → Colab abre nova aba via serve_kernel_port_as_window()

Navegador (nova aba)
    → carrega JSON com todos os frames
    → HTML5 Canvas renderiza labirinto base UMA VEZ (sem piscar)
    → requestAnimationFrame() atualiza só os caminhos e barras a cada frame
    → 60fps nativos, sem comunicação com Python
```

<br>

## Referências

- **Kirkpatrick, S., Gelatt, C. D., & Vecchi, M. P.** (1983). Optimization by Simulated Annealing. *Science*, 220(4598), 671–680. — artigo original do SA
- **University of Manchester** — An Application of Simulated Annealing to Maze Routing
- **IEEE** — Robot Path Planning in Dynamic Environments Using a Simulated Annealing Based Approach
- **Springer Nature** — Cooperative Simulated Annealing for Path Planning in Multi-Robot Systems
- **Cornell University Optimization Wiki** — Simulated Annealing

<br>

## Tecnologias

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Google Colab](https://img.shields.io/badge/Google%20Colab-F9AB00?style=flat&logo=googlecolab&logoColor=white)
![NumPy](https://img.shields.io/badge/NumPy-013243?style=flat&logo=numpy&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-000000?style=flat&logo=flask&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=flat&logo=javascript&logoColor=black)
![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=flat&logo=html5&logoColor=white)
