---
title: Principal Component Analysis of the Treasury Yield Curve
date: 2026-08-26 00:00:00 +0000
categories: [Quant]
tags: [Interest Rate, Fixed Income]
description: Principal Component Analysis of the Treasury Yield Curve
pin: false
math: true
---
![Historical Treasury Yield Curve](/assets/img/posts/pca/ch1.png)

## Introduction

The US Treasury curve is the benchmark of global interest rates. It serves as the foundation for pricing almost all other fixed-income instruments, such as corporate bonds, interest rate swaps and mortgage-backed securities. Another example is that the risk managers and Asset Liability Management (ALM) teams model curve shifts to measure how a 50-basis-point parallel rate hike would impact the bank's net interest margin or a portfolio's Value-at-Risk (VaR).

Principal Component Analysis (PCA) is a dimensionality reduction technique that simplifies complex, multi-dimensional data into a few key independent drivers (level, slope, curvature, etc.).

**Refer to Interest Rate modeling Notebook and PCA fundamentals:**
* [Analysis on Treasury Yield Curve (PCA, VaR, PnL Attribution)](https://github.com/jhqntdv/Interest-Rate-Modeling/blob/main/pca-main.ipynb)
* [The impact of Fed Funds Rate on the curve (Linear Regression and ECM)](https://github.com/jhqntdv/Interest-Rate-Modeling/blob/main/fed-main.ipynb)
* [Statistics Cheat Sheet](./2026-08-26-Statistics%20Cheat%20Sheet.md)

## Examples of PCA applications in Fixed Income

1. **PnL Explain:** A trader's daily Profit and Loss (PnL) on a $500M swap portfolio can be messy if different tenors moved in opposite directions. PCA allows the risk team to break down yesterday's -$50k PnL into concrete buckets: "We lost $60k because the overall curve shifted up (Level), made $15k because the curve steepened (Slope), and lost $5k from a change in the 5Y hump (Curvature)."
2. **Hedging:** If a swap dealer holds a massive, complex portfolio across 20 different maturities, hedging every single tenor individually is expensive and impractical due to bid-ask spreads. Instead, they calculate the portfolio's exposure to PC1, PC2, and PC3, and trade just three liquid instruments (like 2Y, 5Y, and 10Y Treasury futures) to completely neutralize the portfolio's dominant risks.
3. **Scenario Analysis:** Risk managers need to simulate stress tests, like a sudden Fed rate hike or a repeat of the 2008 liquidity crisis. Instead of randomly shocking 11 different tenors—which might create a physically impossible yield curve shape—they shock the three independent PCA components to generate mathematically realistic, historically consistent stress scenarios.

## PCA

### 1. The Economic Meaning of PCA (Litterman & Scheinkman)
PCA is a mathematical dimension-reduction technique with no inherent economic meaning. However, Litterman and Scheinkman (1991) in their seminal paper *"Yield Curve and Hedging Returns"* discovered that applying PCA to interest rates yields components with clear economic interpretations:
*   **PC1 (Level):** The plot is a relatively flat horizontal line (across all tenors)and shifts up or down together.
*   **PC2 (Slope):** Weights are negative for short tenors and positive for long tenors. Its plot is a slanted line crossing zero.
*   **PC3 (Curvature):** Weights are positive on the short and long ends but negative in the middle. Its plot is U-shaped.

![PCA Components](/assets/img/posts/pca/ch2.png)

### 2. CCAR and Regulatory Stress Testing
Under the **CCAR (Comprehensive Capital Analysis and Review)**, banks must simulate stress scenarios that are extreme yet plausible to satisfy Federal Reserve rules. Randomly shocking individual interest rate tenors is not allowed. PCA serves as a standardized approach because it helps modelers shock just the independent components (Level, Slope, Curvature) to ensure that the generated stress scenarios are mathematically sound and defensible to regulators.

### 3. Independent Risk Factors
A key mathematical property of PCA is that its components are independent of each other. This is helpful for risk management because it allow one to isolate and hedge the "Slope" risk of a portfolio without accidentally altering exposure to the "Level" risk. For example, traders breakdown the PnL into level, slope and curvature risk factors to understand the sources of risk and could hedge "Slope" risk by trading instruments that are sensitive to "Slope" risk. 

Additionally, if there is unexplained PnL (the residual difference between actual PnL and what the PCA model predicts), it is handled rigorously in practice:

1. If the unexplained PnL remains high, the modelders (Quant/Front Office) would need to revisit and revise the model such as including more principal components or using a different model.
2. Middle office teams would initialte a formal investigation or valuation adjustment (risk reserves) if the unexplained PnL breaches predefined thresholds (Volcker Rule). 

    ![PnL Attribution](/assets/img/posts/pca/ch3.png){: style="display: block; margin: 0 auto;" }