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

This article is organized into the following sections:

* [**Examples of PCA Applications in Fixed Income**](#examples-of-pca-applications-in-fixed-income): Practical use cases in PnL attribution, multi-tenor hedging with liquid futures, and scenario stress testing.
* [**Part 1: The Economic Meaning of PCA**](#1-the-economic-meaning-of-pca-litterman--scheinkman): Empirical interpretation of Level, Slope, and Curvature across macro regimes, rolling eigensolvers with the Hungarian algorithm, and hyperparameter tuning.
* [**Part 2: CCAR and Regulatory Stress Testing**](#2-ccar-and-regulatory-stress-testing): Macroeconomic scenario design, Historical Simulation Value-at-Risk (VaR), Component VaR decomposition, and matrix algebra formulation.
* [**Part 3: Independent Risk Factors**](#3-independent-risk-factors): Orthogonal risk isolation, directional vs. curve spread PnL dynamics, and residual monitoring for risk governance.

**Refer to Interest Rate Modeling Notebooks and PCA Fundamentals:**
* [Analysis on Treasury Yield Curve (PCA, VaR, PnL Attribution)](https://github.com/jhqntdv/Interest-Rate-Modeling/blob/main/pca-main.ipynb)
* [The impact of Fed Funds Rate on the curve (Linear Regression and ECM)](https://github.com/jhqntdv/Interest-Rate-Modeling/blob/main/fed-main.ipynb)
* [Statistics Cheat Sheet](/posts/Statistics-Cheat-Sheet/)

---

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

**Baseline Model Setup (3Y Window / 3Y ESS):**  
The empirical results presented below (Charts 1–9) are estimated using a **rolling lookback window of 756 trading days** ($252 \times 3 \approx 3\text{ years}$) paired with an **Effective Sample Size (ESS) of 756 days** ($252 \times 3 \approx 3\text{ years}$). Under exponential weighting ($\lambda = \frac{\text{ESS} - 1}{\text{ESS} + 1}$), this translates to an effective decay factor of **$\lambda \approx 0.99736$** (implying a half-life of $\approx 262$ trading days / $\approx 1$ year). The following results are presented based on this **3-year / 3-year** baseline configuration.

#### 1.1 Factor Loadings vs. Maturity (Charts 1–3)

* **Chart 1 — PC1 (Level) Loadings vs. Maturity:**
  * Cross-sectional snapshots across four macro regimes (2013 Taper Tantrum, 2017 mid-hiking cycle, 2022 hiking shock, and 2026 current).
  * Loadings are strictly positive and monotonically upward-sloping from short to long tenors (~0.01–0.15 at 3M up to ~0.40–0.62 at 10Y–30Y), proving that level shocks structurally shift the entire curve in the same direction across varying regimes.

* **Chart 2 — PC2 (Slope) Loadings vs. Maturity:**
  * Exhibits a consistent downward-to-upward steepener tilt: negative weights on short tenors (~-0.40 to -0.60 at 2Y–5Y) and positive weights on the long end (~+0.50 to +0.60 at 30Y).
  * Consistently pivots through zero between intermediate maturities (~3Y–6Y), confirming that a 1-unit slope shift twists short and long rates in opposite directions.

* **Chart 3 — PC3 (Curvature) Loadings vs. Maturity:**
  * Demonstrates the classic butterfly hump: positive loadings on the belly (~+0.35 to +0.42 at 2Y–5Y) flanked by negative loadings on both the short and long wings (~-0.30 to -0.70 at 3M; ~-0.30 to -0.38 at 30Y).
  * Structural stability across all four regimes verifies that the model is well-specified and regime-independent.

#### 1.2 Rolling Factor Loadings over Time (Charts 4–6)

**Methodology Note — Factor Tracking & the Hungarian Algorithm:**  
When computing PCA across rolling time windows, numerical eigensolvers suffer from two mathematical ambiguities:
1. **Sign Flipping:** Eigenvectors are unique only up to a sign flip ($v$ and $-v$ satisfy the same eigensystem).
2. **Factor Swapping (Eigenvalue Crossing):** During market regime shifts or when eigenvalues are close, the ranking of principal components can abruptly swap (e.g., PC2 and PC3 switching positions).

The **Hungarian Algorithm** (Kuhn-Munkres) solves this by treating factor tracking as a bipartite matching problem: it finds the optimal 1-to-1 assignment that maximizes the absolute inner product matrix between current eigenvectors (at time $t$) and reference eigenvectors (at time $t-1$). Combined with sign alignment ($\text{sgn}(v_t \cdot v_{t-1})$), it preserves smooth, continuous factor identities (Level, Slope, Curvature) across the entire rolling history without artificial jumps.

* **Chart 4 — Rolling PC1 Loadings over Time:**
  * Tracks each tenor's sensitivity to Level over 2013–2026: long tenors (5Y–30Y) maintain high, stable exposures (~0.45–0.62), while front-end tenors (3M–1Y) remain low (<0.15 pre-2020) before rising to 0.30–0.45 during the 2022–2024 Fed tightening cycle.
  * Time series are materially smooth with zero vertical discontinuities or line-crossing artifacts.

* **Chart 5 — Rolling PC2 Loadings over Time:**
  * Persistently separates long-end steepeners (30Y at +0.35 to +0.65) from short-end anchors (3M–2Y at -0.20 to -0.60), with the 5Y–10Y sector acting as the stable transition zone.
  * Shows robust factor identity preservation through the 2015–2018 hiking cycle, 2020 COVID shock, and 2022–2023 yield curve inversion.

* **Chart 6 — Rolling PC3 Loadings over Time:**
  * Belly tenors (2Y, 5Y) consistently maintain positive loadings (+0.10 to +0.55), while short and long wings (3M, 30Y) remain negative (-0.10 to -0.80).
  * Confirms that identity swaps and eigenvalue crossings are fully eliminated across the 13-year rolling sample.

#### 1.3 Variance Explained, Factor Scores & Curve Shapes (Charts 7–9)

* **Chart 7 — Rolling Explained Variance Ratio:**
  * Total explanatory power (PC1+PC2+PC3) stays consistently high at ~92%–97% (PC1: ~75%–85%, PC2: ~10%–15%, PC3: ~3%–6%).
  * The transient dip in total variance to ~90% (with PC1 dropping to ~72%–75%) during 2020–2022 is an expected economic phenomenon: front-end rates were pinned near the zero lower bound (ZLB) while long-end yields moved independently, causing a decoupling that a linear 3-factor model cannot fully span.

* **Chart 8 — Rolling PCA Factor Scores:**
  * Represents daily standardized factor realizations (typically fluctuating within $[-20, +20]$).
  * Sharp spikes in March 2020 (emergency liquidity dislocation and rate cuts) and 2022–2023 (rapid 75 bps Fed rate hike cycle, with scores reaching $\pm 60$ to $-75$) are intentionally retained rather than smoothed away, preserving authentic market shocks critical for risk management and PnL attribution.

* **Chart 9 — Actual Yield Curves at 4 Sample Dates:**
  * Provides the raw curve context underlying Charts 1–3: 2013 (steep curve; 3M at 0.06%, 30Y at 3.06%, spread $\approx 300$ bps), 2017 (moderately steep mid-cycle), 2022 (pre-hiking flat baseline), and 2026 (elevated, inverted curve with 3M at 3.86% and 10Y at 4.65%).
  * Confirms factor shapes align directly with observed yield curve geometries across varying monetary regimes.

#### 1.4 Hyperparameter Tuning: Rolling Windows & Decay Parameters (24 Scenarios)

* **Scenario Framework & Parameter Grid:**
  * Rolling PCA has two free hyperparameters: lookback window length ($W \in [252, 1260]$ days) and Effective Sample Size ($\text{ESS} \in [252, 756]$ days, or decay $\lambda \in [0.99209, 0.99736]$).
  * Optimizing purely on in-sample explained variance is fundamentally biased: shorter windows mechanically inflate in-sample fit (from 94.59% at $W=1260$ to 95.15% at $W=252$) by memorizing sample-specific noise without improving true explanatory power.
  * The 24 scenarios were evaluated along three distinct quantitative dimensions:
    1. **Out-of-Sample (OOS) Explained Variance:** Measures how effectively yesterday's estimated loadings explain today's realized curve moves, testing whether shorter windows capture genuine predictive signal.
    2. **Loading Instability (Turnover):** Quantifies day-to-day volatility in factor weights. Because loadings determine hedge ratios and risk attribution, unstable loadings generate false rebalancing signals and excessive transaction costs.
    3. **Overfit Gap ($R^2_{\text{IS}} - R^2_{\text{OOS}}$):** Monitored as a diagnostic for structural overfitting rather than a direct objective.

![Stability vs. OOS Explanatory Power (Pareto Frontier)](/assets/img/posts/pca/ch4.png){: style="display: block; margin: 0 auto;" }

* **Stability vs. Out-of-Sample Explanatory Power (Pareto Frontier):**
  * **Axes & Coordinates:** Each point represents one Window/ESS combination, plotting Instability Penalty ($x$-axis, lower is better) against OOS Explained Variance ($y$-axis, higher is better).
  * **Dominated Configurations (Gray):** Discarded because alternative combinations achieve both equal-or-better OOS variance and equal-or-better stability; no rational objective function would select them regardless of weighting.
  * **The Pareto Frontier (Red):** The remaining non-dominated frontier where improving one metric strictly costs the other. Choosing an operating point is explicitly an economic judgment call on how much stability is worth in predictive power, rather than an automated decision from a scalar z-score (especially given the narrow 0.866–0.870 OOS variance spread):
    * $W=1260, \text{ESS}=756$: Maximum stability (penalty of 1.68), OOS variance of 86.60%.
    * $W=756, \text{ESS}=756$ (Baseline): Near-identical stability (penalty of 1.81), OOS variance of 86.67%.
    * $W=756, \text{ESS}=504$: Penalty rises to 2.33, OOS variance of 86.73%.
    * $W=504, \text{ESS}=252$: Penalty jumps to 4.14, OOS variance of 86.76%.
    * $W=252, \text{ESS}=252$: Peak OOS variance (87.02%), but instability surges over 3.5× to 5.93.

![Hyperparameter Tuning Top Configurations & Metrics](/assets/img/posts/pca/ch5.png){: style="display: block; margin: 0 auto;" }

* **Top Hyperparameter Configurations & Score Ranking:**
  * Summarizes the 5 non-dominated Pareto configurations sorted from lowest instability to highest OOS variance.
  * **Overfit Gap Invariance:** The overfit gap is virtually constant across all window/ESS configurations at ~0.080–0.082 (0.0799 at $W=1260$ vs. 0.0814 at $W=252$). This stability indicates that the ~8% gap represents irreducible, idiosyncratic daily rate noise rather than tunable overfitting.
  * **Compressed OOS Spread:** Out-of-sample explanatory power spans a tight 42 bps range (86.60% to 87.02%), showing that aggressive lookback shortening delivers rapidly diminishing predictive gains.

* **Performance & Stability Observations:**
  * **Flaw of the Scalar "Combined Score":** The z-score Combined Score (ranging from 0.186 to 2.462) artificially amplifies microscopic OOS variance gains (+0.42%) over massive increases in turnover (+253% in instability), unfairly penalizing long, stable windows.

* **Why 756/756 Was Selected:**
  * **Pareto Frontier Trade-Off:** 756/756 sits directly on the Pareto frontier, conceding only ~0.35 percentage points of OOS explained variance (86.67% vs. 87.02%) relative to the highest-scoring 252/252 setup in exchange for a ~3.3× improvement in stability (instability penalty of 1.81 vs. 5.93).
  * **Noise vs. Operational Reality:** OOS variance differences across the entire grid are economically immaterial (well within market noise), whereas factor loading instability directly and consequentially destabilizes downstream hedge ratios and PnL attribution.
  * **Equal Weighting Across 3 Years:** A 756-day lookback represents a 3-year window, and $\text{ESS} = 756$ implies an effective decay of $\lambda \approx 0.99736$—amounting to near-equal weighting across the full 3 years with essentially no meaningful decay.

### 2. CCAR and Regulatory Stress Testing
Under the **CCAR (Comprehensive Capital Analysis and Review)**, banks must simulate stress scenarios that are extreme yet plausible to satisfy Federal Reserve rules. Randomly shocking individual interest rate tenors is not allowed. PCA serves as a standardized approach because it helps modelers shock just the independent components (Level, Slope, Curvature) to ensure that the generated stress scenarios are mathematically sound and defensible to regulators.

#### 2.1 From Scenario Shocks to Factor Value-at-Risk (Factor VaR)
For ongoing market risk management and desk limit monitoring, regulatory stress testing extends directly to **Factor Value-at-Risk (Factor VaR)**. Rather than monitoring an aggregate, opaque loss metric, Factor VaR isolates portfolio tail risk into the three independent economic drivers over an $N$-day horizon:
* **$\text{VaR}_{\text{PC1}}$ (Level VaR):** Tail loss driven purely by parallel curve shifts (outright duration risk).
* **$\text{VaR}_{\text{PC2}}$ (Slope VaR):** Tail loss driven by curve steepening or flattening pivots (curve spread risk).
* **$\text{VaR}_{\text{PC3}}$ (Curvature VaR):** Tail loss driven by belly humping or wing movements (butterfly risk).

#### 2.2 Matrix Formulation of Factor VaR
The historical simulation of $N$-day Factor VaR is computed via three vectorized matrix operations:

1. **Factor Sensitivity Vector ($\mathbf{dv01}_{\text{factor}}$):**  
   Project the portfolio tenor DV01 vector $\mathbf{dv01} \in \mathbb{R}^M$ ($M = 7$ tenors) onto the latest rolling loading matrix $\mathbf{L} \in \mathbb{R}^{K \times M}$ ($K = 3$ components):
   $$ \mathbf{dv01}_{\text{factor}} = \mathbf{L} \, \mathbf{dv01} = \begin{bmatrix} \text{DV01}_{\text{PC1}} \\ \text{DV01}_{\text{PC2}} \\ \text{DV01}_{\text{PC3}} \end{bmatrix} \in \mathbb{R}^K $$

2. **Historical Component PnL Matrix ($\mathbf{\Pi}$):**  
   Given the matrix of rolling $N$-day cumulative factor shocks $\mathbf{S} \in \mathbb{R}^{T \times K}$ ($\mathbf{S}_{t, :} = \sum_{i=0}^{N-1} \mathbf{f}_{t-i}$ across $T$ historical days), scale each factor column by its respective sensitivity:
   $$ \mathbf{\Pi} = - \mathbf{S} \, \text{diag}(\mathbf{dv01}_{\text{factor}}) \in \mathbb{R}^{T \times K} $$
   where $\Pi_{t, k} = - S_{t, k} \cdot \text{DV01}_{\text{factor}, k}$ is the simulated PnL from factor $k$ on historical day $t$.

3. **95% Factor VaR Vector:**  
   Extract the 5th percentile across each factor column independently:
   $$ \mathbf{VaR}_{95\%} = \text{Percentile}_{5} \left( \mathbf{\Pi}, \text{axis}=0 \right) = \begin{bmatrix} \text{VaR}_{\text{PC1}} \\ \text{VaR}_{\text{PC2}} \\ \text{VaR}_{\text{PC3}} \end{bmatrix} \in \mathbb{R}^K $$

### 3. Independent Risk Factors

A key advantage of PCA in portfolio risk management is the **orthogonality** of its components. Because raw Treasury yields across tenors are highly collinear, individual tenor DV01 exposures obscure the underlying curve drivers. With orthogonal factors, daily portfolio PnL can be linearly decomposed into independent risk dimensions:

$$ \Delta \text{PnL}_{\text{Actual}} \approx \text{PnL}_{\text{Level}} + \text{PnL}_{\text{Slope}} + \text{PnL}_{\text{Curvature}} + \text{Residual} $$

where factor PnL is computed as $\text{PnL}_{k} = - \Delta f_k \cdot \text{DV01}_{\text{factor}, k}$, and the residual captures anything not accounted for by the first three components.

![PnL Attribution](/assets/img/posts/pca/ch3.png){: style="display: block; margin: 0 auto;" }

#### Portfolio Comparison & Numerical Walkthrough

The tables above illustrate this decomposition across two benchmark portfolios over the same five-day sample:

1. **Table 1: Outright Duration Position (Directional Risk)**
   * **PC1 Dominance & Factor Trimming:** Daily performance is driven almost entirely by Level shifts. On 2026-05-04, PC1 accounts for -<span>$</span>6,527 of the -<span>$</span>7,200 loss. On 2026-05-06, PC1 slightly overshoots the actual gain (+<span>$</span>6,601 vs. +<span>$</span>6,200), with negative contributions from Curvature (-<span>$</span>345) and Slope (-<span>$</span>19) trimming total PCA PnL to +<span>$</span>6,237.
   * **Tight Residuals:** Across all five days, unexplained residuals stay within -<span>$</span>37 to +<span>$</span>40 (< 1% of daily swings), confirming that three macro factors cleanly explain standard directional duration.

2. **Table 2: 2s10s Steepener Position (Curve Spread Risk)**
   *(Filtering out May 5 and May 7 where actual portfolio PnL was flat at <span>$</span>0 to focus strictly on active trading sessions where Actual PnL = -<span>$</span>1,000):*
   * **Duration Immunization:** Across all active sessions (2026-05-01 at +<span>$</span>11, 2026-05-04 at -<span>$</span>283, and 2026-05-06 at +<span>$</span>263), PC1 PnL remains tightly constrained in the low hundreds of dollars—a stark contrast to the multi-thousand-dollar swings in Table 1—verifying effective protection against parallel yield shifts.
   * **Slope Dominance vs. Curvature Twists:** On May 1, performance cleanly follows the intended curve steepening (PC2 = -<span>$</span>1,108, residual = +<span>$</span>153). On May 4 and May 6, however, Curvature swings (PC3 = +<span>$</span>921 and +<span>$</span>604) actively counter Slope, pulling PCA PnL away from actual performance.
   * **Residual Range on Active Days:** Unexplained residuals on active trading days range from -<span>$</span>1,652 to +<span>$</span>153. Once directional duration is stripped away, higher-order curve twists, local tenor basis, and convexity make up a significant portion of net performance.

#### Practical Takeaway on Residuals
PCA is an effective linear diagnostic tool, but not an all-encompassing pricing model. In spread trading, where level risk is neutralized, higher-order effects—such as belly twists, roll-down differentials, and local tenor basis—naturally emerge. Tracking residuals helps risk managers separate genuine curve-spread performance from unmodeled structural basis and position mismatch.

