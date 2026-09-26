---
title: Principal Component Analysis of the Treasury Yield Curve
date: 2026-08-26 00:00:00 +0000
categories: [Quant]
tags: [Interest Rate, Fixed Income]
description: Principal Component Analysis of the Treasury Yield Curve
pin: true
math: true
---
![Historical Treasury Yield Curve](/assets/img/posts/pca/ch1.png)

## Introduction

The US Treasury curve is the benchmark of global interest rates. It serves as the foundation for pricing almost all other fixed-income instruments, such as corporate bonds, interest rate swaps and mortgage-backed securities. In addition, risk managers and Asset Liability Management (ALM) teams model curve shifts to measure how a 50-basis-point parallel rate hike would impact the bank's net interest margin or a portfolio's Value-at-Risk (VaR).

Principal Component Analysis (PCA) is a dimensionality reduction technique that simplifies complex, multi-dimensional data into a few key independent drivers (level, slope, curvature, etc.).

**Refer to Interest Rate Modeling Notebooks and PCA Fundamentals:**
* [Analysis on Treasury Yield Curve (PCA, VaR, PnL Attribution)](https://github.com/jhqntdv/Interest-Rate-Modeling/blob/main/pca-main.ipynb)
* [The impact of Fed Funds Rate on the curve (Linear Regression and ECM)](https://github.com/jhqntdv/Interest-Rate-Modeling/blob/main/fed-main.ipynb)
* [Statistics Cheat Sheet](/posts/Statistics-Cheat-Sheet/)

## Examples of PCA applications in Fixed Income

1. **PnL Explain:** A trader's daily Profit and Loss (PnL) on a <span>$</span>500M swap portfolio can be messy if different tenors moved in opposite directions. PCA allows the risk team to break down yesterday's -<span>$</span>50k PnL into concrete buckets: "We lost <span>$</span>60k because the overall curve shifted up (Level), made <span>$</span>15k because the curve steepened (Slope), and lost <span>$</span>5k from a change in the 5Y hump (Curvature)."
2. **Hedging:** If a swap dealer holds a massive, complex portfolio across 20 different maturities, hedging every single tenor individually is expensive and impractical due to bid-ask spreads. Instead, they calculate the portfolio's exposure to PC1, PC2, and PC3, and trade just three liquid instruments (like 2Y, 5Y, and 10Y Treasury futures) to completely neutralize the portfolio's dominant risks.
3. **Scenario Analysis:** Risk managers need to simulate stress tests, like a sudden Fed rate hike or a repeat of the 2008 liquidity crisis. Instead of randomly shocking 11 different tenors—which might create a physically impossible yield curve shape—they shock the three independent PCA components to generate mathematically realistic, historically consistent stress scenarios.

## PCA

### 1. The Economic Meaning of PCA (Litterman & Scheinkman)
PCA is a mathematical dimension-reduction technique with no inherent economic meaning. However, Litterman and Scheinkman (1991) in their seminal paper *"Yield Curve and Hedging Returns"* discovered that applying PCA to interest rates yields components with clear economic interpretations:
*   **PC1 (Level):** The plot is a relatively flat line across maturities; shifts up or down together across all tenors.
*   **PC2 (Slope):** Weights are oppositely signed between short and long tenors, crossing zero at intermediate maturities.
*   **PC3 (Curvature):** Weights share the same sign on the short and long wings, but take the opposite sign in the belly (U-shaped or inverted U-shaped).

![PCA Components](/assets/img/posts/pca/ch2.png){: style="display: block; margin: 0 auto;" }

#### 1.1 Factor Loadings vs. Maturity (Charts 1–3)

* **Chart 1 — PC1 (Level) Loadings vs. Maturity:**
  * 

* **Chart 2 — PC2 (Slope) Loadings vs. Maturity:**
  * 

* **Chart 3 — PC3 (Curvature) Loadings vs. Maturity:**
  * 

#### 1.2 Rolling Factor Loadings over Time (Charts 4–6)

> **Methodology Note — Factor Tracking & the Hungarian Algorithm:**  
> When computing PCA across rolling time windows, numerical eigensolvers suffer from two mathematical ambiguities:
> 1. **Sign Flipping:** Eigenvectors are unique only up to a sign flip ($v$ and $-v$ satisfy the same eigensystem).
> 2. **Factor Swapping (Eigenvalue Crossing):** During market regime shifts or when eigenvalues are close, the ranking of principal components can abruptly swap (e.g., PC2 and PC3 switching positions).
> 
> The **Hungarian Algorithm** (Kuhn-Munkres) solves this by treating factor tracking as a bipartite matching problem: it finds the optimal 1-to-1 assignment that maximizes the absolute inner product matrix between current eigenvectors ($t$) and reference eigenvectors ($t-1$). Combined with sign alignment ($\text{sgn}(v_t \cdot v_{t-1})$), it preserves smooth, continuous factor identities (Level, Slope, Curvature) across the entire rolling history without artificial jumps.

* **Chart 4 — Rolling PC1 Loadings over Time:**
  * 

* **Chart 5 — Rolling PC2 Loadings over Time:**
  * 

* **Chart 6 — Rolling PC3 Loadings over Time:**
  * 

#### 1.3 Variance Explained, Factor Scores & Curve Shapes (Charts 7–9)

* **Chart 7 — Rolling Explained Variance Ratio:**
  * 

* **Chart 8 — Rolling PCA Factor Scores:**
  * 

* **Chart 9 — Actual Yield Curves at 4 Sample Dates:**
  * 

#### 1.4 Hyperparameter Tuning: Rolling Windows & Decay Parameters (24 Scenarios)

* **Scenario Framework & Parameter Grid:**
  * Evaluating combinations of rolling lookback windows and exponential decay parameters ($\lambda$) across 24 test scenarios.

* **Performance & Stability Observations:**
  * 

### 2. CCAR and Regulatory Stress Testing
Under the **CCAR (Comprehensive Capital Analysis and Review)**, banks must simulate stress scenarios that are extreme yet plausible to satisfy Federal Reserve rules. Randomly shocking individual interest rate tenors is not allowed. PCA serves as a standardized approach because it helps modelers shock just the independent components (Level, Slope, Curvature) to ensure that the generated stress scenarios are mathematically sound and defensible to regulators.

### 3. Independent Risk Factors
A key mathematical property of PCA is that its components are independent of each other. This is helpful for risk management because it allows one to isolate and hedge the "Slope" risk of a portfolio without accidentally altering exposure to the "Level" risk. For example, traders break down the PnL into level, slope, and curvature risk factors to understand the sources of risk and can hedge "Slope" risk by trading instruments that are sensitive to "Slope" risk. 

Additionally, if there is unexplained PnL (the residual difference between actual PnL and what the PCA model predicts), it is handled rigorously in practice:

1. If the unexplained PnL remains high, the modelers (Quant/Front Office) would need to revisit and revise the model such as including more principal components or using a different model.
2. Middle office teams would initiate a formal investigation or valuation adjustment (risk reserves) if the unexplained PnL breaches predefined thresholds (Volcker Rule). 

    ![PnL Attribution](/assets/img/posts/pca/ch3.png){: style="display: block; margin: 0 auto;" }