# Medidas DAX / DAX Measures

Documentação das medidas criadas para este projeto, com o raciocínio por trás de cada uma.
Documentation of the measures built for this project, with the reasoning behind each one.

---

## 1. Ponte financeira (Waterfall) / Financial bridge

**Tabela auxiliar `Cascata`** — tabela pequena, digitada manualmente (Inserir dados), com as colunas `Categoria` e `Ordem`:

| Categoria | Ordem |
|---|---|
| Gross sales | 1 |
| Discounts | 2 |
| COGS | 3 |

`Categoria` usa **Classificar por coluna → Ordem**, garantindo que o gráfico de cascata sempre respeite essa sequência, e não a ordem alfabética.

```dax
Valor Cascata =
SWITCH(
    SELECTEDVALUE(Cascata[Categoria]),
    "Gross sales", SUM(financials[Gross Sales]),
    "Discounts",  -SUM(financials[Discounts]),
    "COGS",       -SUM(financials[COGS])
)
```

**Por que assim, e não com múltiplas medidas direto no Eixo Y:** o visual nativo de cascata do Power BI só aceita uma medida no Eixo Y quando o poço Categoria está preenchido. A saída foi mapear cada categoria da ponte para o valor certo (com sinal invertido nas deduções) através de uma única medida via `SWITCH`. A barra final ("Total"), que fecha em `Gross Sales − Discounts − COGS`, é calculada automaticamente pelo próprio visual e bate exatamente com o Profit real da base.

**Why this approach instead of multiple measures directly on the Y-axis:** Power BI's native waterfall visual only accepts a single measure on the Y-axis once the Category well is populated. The workaround was to map each bridge category to the correct value (with inverted sign on the deductions) through a single `SWITCH`-based measure. The final "Total" bar — which closes at `Gross Sales − Discounts − COGS` — is computed automatically by the visual itself and matches the dataset's real Profit exactly.

---

## 2. País com maior / menor margem (cartões dinâmicos) / Highest & lowest margin country (dynamic cards)

```dax
Pais Maior Margem =
VAR TabelaPaises = ADDCOLUMNS(VALUES(financials[Country]), "M", CALCULATE([Margem Real]))
VAR Top1 = TOPN(1, TabelaPaises, [M], DESC)
RETURN CONCATENATEX(Top1, financials[Country])
```

```dax
Pais Menor Margem =
VAR TabelaPaises = ADDCOLUMNS(VALUES(financials[Country]), "M", CALCULATE([Margem Real]))
VAR Top1 = TOPN(1, TabelaPaises, [M], ASC)
RETURN CONCATENATEX(Top1, financials[Country])
```

**Por que medida em vez de filtro Top N no visual:** um filtro Top N no cartão funcionaria, mas amarra a lógica à configuração do visual (fácil de quebrar sem querer, difícil de reaproveitar). Como medida, o resultado é portátil — pode ser usado em qualquer visual, cartão, tooltip ou outra medida, e se recalcula sozinho se o dataset mudar.

**Why a measure instead of a visual-level Top N filter:** a Top N filter on the card would work, but ties the logic to that specific visual's configuration (easy to break by accident, hard to reuse). As a measure, the result is portable — it can be dropped into any card, tooltip, or other measure, and it recalculates on its own if the underlying data changes.

---

## 3. Margem por faixa de desconto / Margin by discount band

O gráfico de barras `Margem Real` por `Discount Band` usa a medida pré-existente `Margem Real` (não documentada aqui em detalhe, pois já fazia parte do modelo). A ordenação das barras (do maior para o menor valor) é o comportamento padrão do Power BI para um gráfico de colunas recém-criado — não foi necessária nenhuma coluna de apoio.

The `Margem Real` by `Discount Band` bar chart uses the pre-existing `Margem Real` measure (not detailed here, as it was already part of the model). Bar ordering (highest to lowest) is Power BI's default behavior for a newly created column chart — no helper column was needed.

---

## 4. Destaques de produto (cartões dinâmicos) / Product highlights (dynamic cards)

Mesmo padrão do item 2, aplicado a `Product` em vez de `Country` — na página **Analise de produto**:

```dax
Produto Mais Vendido =
VAR TabelaProdutos = ADDCOLUMNS(VALUES(financials[Product]), "U", CALCULATE(SUM(financials[Units Sold])))
VAR Top1 = TOPN(1, TabelaProdutos, [U], DESC)
RETURN CONCATENATEX(Top1, financials[Product])
```

```dax
Produto Menos Vendido =
VAR TabelaProdutos = ADDCOLUMNS(VALUES(financials[Product]), "U", CALCULATE(SUM(financials[Units Sold])))
VAR Top1 = TOPN(1, TabelaProdutos, [U], ASC)
RETURN CONCATENATEX(Top1, financials[Product])
```

```dax
Produto Maior Margem =
VAR TabelaProdutos = ADDCOLUMNS(VALUES(financials[Product]), "M", CALCULATE([Margem Real]))
VAR Top1 = TOPN(1, TabelaProdutos, [M], DESC)
RETURN CONCATENATEX(Top1, financials[Product])
```

```dax
Produto Menor Margem =
VAR TabelaProdutos = ADDCOLUMNS(VALUES(financials[Product]), "M", CALCULATE([Margem Real]))
VAR Top1 = TOPN(1, TabelaProdutos, [M], ASC)
RETURN CONCATENATEX(Top1, financials[Product])
```

---

## 5. Curva ABC de Lucro por Produto (Pareto) / ABC Profit Curve

Três medidas encadeadas para construir uma análise de Pareto clássica: quais poucos produtos concentram a maior parte do lucro.
Three chained measures building a classic Pareto analysis: which few products concentrate most of the profit.

```dax
Rank Produto Lucro =
RANKX(ALL(financials[Product]), CALCULATE(SUM(financials[Profit])), , DESC)
```

```dax
Lucro Acumulado =
VAR RankAtual = [Rank Produto Lucro]
RETURN
CALCULATE(
    SUM(financials[Profit]),
    TOPN(RankAtual, ALL(financials[Product]), CALCULATE(SUM(financials[Profit])), DESC)
)
```

```dax
% Lucro Acumulado =
DIVIDE([Lucro Acumulado], CALCULATE(SUM(financials[Profit]), ALL(financials[Product])))
```

Usadas num gráfico de colunas (Lucro por produto) + linha no eixo secundário (`% Lucro Acumulado`), ordenado por lucro decrescente — o formato clássico de curva ABC/Pareto.

Used in a column chart (Profit per product) + secondary-axis line (`% Lucro Acumulado`), sorted by descending profit — the classic ABC/Pareto curve layout.

---

## Interações entre visuais / Visual interactions

Na página **Executiva**, o slicer (`Country`, `Month Number`, `Product`) está sobreposto visualmente ao gráfico de tendência para parecer um controle nativo da própria página. Suas interações com os dois cartões de destaque (`Pais Maior Margem` e `Pais Menor Margem`) foram explicitamente desativadas via **Formato → Editar interações → Nenhum**, para que a seleção no slicer não altere esses dois cartões — que devem sempre refletir o panorama geral, não o recorte do filtro.

On the **Executiva** page, the slicer (`Country`, `Month Number`, `Product`) is visually overlaid on the trend chart to look like a native page control. Its interaction with the two highlight cards (`Pais Maior Margem` and `Pais Menor Margem`) was explicitly disabled via **Format → Edit interactions → None**, so that a slicer selection never changes those two cards — which should always reflect the overall picture, not the filtered slice.
