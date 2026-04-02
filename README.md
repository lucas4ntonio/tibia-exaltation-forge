# Tibia — Exaltation Forge Calculator

Calculadora de custos para o sistema de upgrade de equipamentos da **Exaltation Forge** no Tibia para itens Class 4, com modelagem probabilística completa via cadeia de Markov e simulação Monte Carlo.

---

## O que a calculadora faz

A ferramenta calcula o custo esperado de subir o tier de um equipamento Class 4 através de dois caminhos:

1. **Convergence Fusion** — método garantido, usando um item "food" (barato) como sacrifício
2. **Fusion Normal** — método probabilístico, fundindo dois itens idênticos com chance de falha

Para cada caminho, são apresentados três cenários:

| Cenário | Descrição |
|---|---|
| **Perfeito** | Nenhuma falha em nenhuma tentativa — custo mínimo teórico |
| **Médio** | Valor esperado calculado analiticamente via cadeia de Markov |
| **P99** | Percentil 99 de 60k simulações Monte Carlo — o "caso de azar extremo" |

---

## Taxas Oficiais (Class 4)

Todos os valores de gold foram extraídos diretamente da wiki do Tibia.

### Fusion Normal

| Tier | Taxa de Gold |
|------|-------------|
| T1 | 8kk |
| T2 | 20kk |
| T3 | 40kk |
| T4 | 65kk |
| T5 | 100kk |
| T6 | 250kk |
| T7 | 750kk |
| T8 | 2.500kk |
| T9 | 8.000kk |
| T10 | 15.000kk |

### Convergence Fusion

| Tier | Taxa de Gold |
|------|-------------|
| T1 | 55kk |
| T2 | 110kk |
| T3 | 170kk |
| T4 | 300kk |
| T5 | 875kk |
| T6 | 2.350kk |
| T7 | 6.950kk |
| T8 | 21.250kk |
| T9 | 50.000kk |
| T10 | 125.000kk |

> Convergence Fusion consome **130 Dust** por tentativa. Fusion Normal consome **100 Dust** por tentativa.

---

## Metodologia

### 1. Convergence Fusion + Food

A estratégia consiste em subir o **item caro** (BIS) via Convergence Fusion, usando como sacrifício um **item food** (barato do mesmo slot e tier), construído previamente via Fusion Normal.

O fluxo por tier é:

```
Item Caro T(n-1) + Food T(n-1) + Taxa Convergence(n) = Item Caro T(n)
```

Por exemplo, para chegar ao T4:

```
Item Caro T0 + Food T0 + 55kk  = Item Caro T1
Item Caro T1 + Food T1 + 110kk = Item Caro T2
Item Caro T2 + Food T2 + 170kk = Item Caro T3
Item Caro T3 + Food T3 + 300kk = Item Caro T4
```

O custo do **item caro** é pago apenas uma vez (T0). O custo do **food T(n-1)** é calculado probabilisticamente, pois ele é construído via Fusion Normal.

O **custo acumulado total** até o tier N é:

```
Total(N) = Preço Item Caro + Σ [Food(t-1) + TaxaConvergence(t)]  para t = 1..N
```

---

### 2. Fusion Normal — Modelo Probabilístico

#### Regras do jogo

Cada tentativa de Fusion Normal (com 1 Exalted Core) tem três resultados possíveis:

| Resultado | Probabilidade | Consequência |
|---|---|---|
| **Sucesso** | 65% | Os 2 itens T(n) fundem e viram 1 item T(n+1) |
| **Falha sem perda** | 17,5% | Ambos os itens permanecem em T(n). Perde apenas gold e Dust |
| **Falha com perda** | 17,5% | 1 item aleatório volta a T(n-1). O outro fica em T(n). Perde gold e Dust |

> A probabilidade de falha com perda é 35% × 50% = 17,5%, pois o Exalted Core reduz a chance de perda de tier de 100% para 50%.

#### Natureza recursiva do custo

A falha com perda de tier torna o problema **recursivo**: quando 1 item volta a T(n-1), ele precisa ser reconstruído do zero para poder tentar a fusão novamente. Essa reconstrução exige, por sua vez, outros itens T(n-2), que também podem falhar, e assim por diante.

#### Custo médio analítico (Cadeia de Markov)

Seja `E(t)` o custo esperado para se obter 1 item no tier `t`. Definindo `X` como o custo esperado de *completar* a fusão dado que já se possui 2 itens em T(t-1):

Cada tentativa pode resultar em:
- Sucesso (65%) → custo zero adicional, fim
- Falha sem perda (17,5%) → paga a taxa e retenta, ainda com 2 itens T(t-1)
- Falha com perda (17,5%) → paga a taxa, 1 item vai a T(t-2), precisa reconstruir gastando `E(t-1)` e retenta

Montando a equação e resolvendo:

```
X = Taxa(t) + P_fail_safe × X + P_fail_loss × (E(t-1) + X)
X × (1 - P_fail_safe - P_fail_loss) = Taxa(t) + P_fail_loss × E(t-1)
X × P_sucesso = Taxa(t) + P_fail_loss × E(t-1)
X = [Taxa(t) + 0,175 × E(t-1)] / 0,65
```

O custo total esperado para obter 1 item no tier `t` é então:

```
E(t) = 2 × E(t-1) + [Taxa(t) + 0,175 × E(t-1)] / 0,65
E(0) = Preço do item T0
```

Esta recorrência é calculada analiticamente para cada tier, produzindo o valor **médio esperado** de forma exata.

#### Custo perfeito (0 falhas)

O cenário sem nenhuma falha segue uma recorrência simples:

```
Perfeito(t) = 2 × Perfeito(t-1) + Taxa(t)
Perfeito(0) = Preço do item T0
```

Exemplos com item T0 = 4kk:

| Tier | Cálculo | Total |
|------|---------|-------|
| T1 | 4 + 4 + 8 | 16kk |
| T2 | 16 + 16 + 20 | 52kk |
| T3 | 52 + 52 + 40 | 144kk |
| T4 | 144 + 144 + 65 | 353kk |

#### Simulação Monte Carlo (P99)

Para o percentil 99, a calculadora executa **60.000 simulações** independentes. Em cada simulação:

1. Para obter 1 item no tier `t`, começa construindo 2 itens em T(t-1) (recursivamente até T0)
2. Tenta a fusão, sorteando um número aleatório para determinar o resultado
3. Em caso de falha com perda, reconstrói o item perdido recursivamente e retenta
4. Acumula o gold gasto ao longo de toda a cadeia

Ao final das 60k simulações, os custos são ordenados e o **percentil 99** é extraído — ou seja, 99% das simulações custaram igual ou menos que esse valor.

---

## Inputs da Calculadora

| Campo | Descrição |
|---|---|
| **Food T0 (kk)** | Preço de mercado do item barato T0 (usado como sacrifício na Convergence) |
| **Item Caro T0 (kk)** | Preço de mercado do item BIS que você quer subir de tier |
| **Tier máximo** | Até qual tier deseja calcular (T1 a T10) |
| **Simulações MC** | Número de runs do Monte Carlo (mais runs = mais preciso, mais lento) |

---

## Interpretação dos Resultados

- **Perfeito**: referência de custo mínimo absoluto. Improvável na prática, mas útil como baseline.
- **Médio**: o valor que você deve esperar gastar na média ao longo de muitas tentativas. Use este para planejamento financeiro.
- **P99**: o "pior caso realista". Em 99% das vezes você gastará menos que isso. Use para saber o quanto de gold reservar para não ficar sem recurso no meio do processo.
- **Multiplicador (Médio / Perfeito)**: indica o overhead das falhas. Um multiplicador de 2× significa que, na média, você gastará o dobro do custo perfeito.

---

## Tecnologias

- HTML5 + CSS3 + JavaScript vanilla (sem dependências externas)
- Fonte: [Cinzel + Crimson Pro](https://fonts.google.com) via Google Fonts
- Matemática: cadeia de Markov analítica + Monte Carlo com `Float64Array` para performance

---

## Dados de Referência

- [TibiaWiki — Equipment Upgrade](https://tibia.fandom.com/wiki/Equipment_Upgrade)
- [TibiaWiki BR — Exaltation Forge](https://www.tibiawiki.com.br/wiki/Exaltation_Forge)
