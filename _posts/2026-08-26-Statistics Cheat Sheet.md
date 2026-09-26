---
title: Statistics Cheat Sheet
date: 2026-08-26 00:00:00 +0000
categories: [Notes]
tags: [Statistics]
description: Statistics Cheat Sheet
pin: false
hidden: true
math: true
---

## Regression & Linear Models

### 1. Linear Regression Core Theory & OLS
Predicts a continuous target variable $y$ as a linear combination of input features $X$:
$$ y = X\beta + \epsilon $$
* Ordinary Least Squares (OLS) Solution:
  $$ \hat{\beta} = (X^T X)^{-1} X^T y $$

---

### 2. LR Checklist

#### Phase 1: Data Preparation

1. Missing Data Handling
2. Outliers & Influential Points:
   * **Leverage**: Measures how far predictor values deviate from the mean.
   * **Cook's Distance**: Measures the actual influence on the coefficients.
3. Feature Scaling & Encoding:
   * One-Hot Encoding and Drop First for non-ordinal categorical variables. Ex: Cities/Genders
   * Ordinal variables. Ex: High school (1), Bachelor (2), Master (3)
   * Standardization ($Z = \frac{X - \mu}{\sigma}$).

#### Phase 2: Testing LINE Assumptions

| Assumption | What It Means | Diagnostic Tool / Test | How to Fix / Remedy |
| :--- | :--- | :--- | :--- |
| Linearity | Expected value of $y$ is a linear combination of $X$. | • Residuals vs. Fitted Plot (flat horizontal band, no curvature) | • Log transform on $y$ |
| Independence | Residuals are independent (No Autocorrelation). | • Durbin-Watson Test <br>• ACF / PACF Plots of residuals | • First-differencing <br>• Add lag features<br>• Newey-West (HAC) Standard Errors |
| Normality | Residuals $\epsilon_i \sim \mathcal{N}(0, \sigma^2)$. | • Q-Q Plot | • Apply Box-Cox or Log transformation on $y$ |
| Equal Variance | $\text{Var}(\epsilon_i) = \sigma^2$ | • Residuals vs. Fitted Plot | • Weighted Least Squares (WLS) |

#### Phase 3: Multicollinearity Diagnostics & Fixes

Multicollinearity leads to unstable parameter estimates without affecting overall prediction:
* Although overall $R^2$ and prediction power may still be high, standard errors of coefficients inflate drastically, and p-values become unreliable.

Detection Methods:
1. Pairwise Pearson correlation ($\lvert r \rvert > 0.8$ implies severe collinearity)
2. **Variance Inflation Factor (VIF)**:
   * Regresses each feature $X_j$ against all other remaining predictors:
     $$ VIF_j = \frac{1}{1 - R_j^2} $$
   * $VIF = 1$: Independent. $1 < VIF < 5$: Moderate correlation. $VIF \ge 5 \text{ to } 10$: Multicollinearity.

To Resolve Multicollinearity:
* Feature Removal: Drop one of the redundant variables (typically the one with the higher p-value / lower t-statistic).
* Regularization: Switch to Ridge Regression (L2) or Elastic Net.

#### Phase 4: Post-Fit Diagnostics & Model Evaluation

1. Goodness-of-Fit ($R^2$ and Adjusted $R^2$):
   $$ R^2 = 1 - \frac{SS_{\text{res}}}{SS_{\text{tot}}} $$
   $$ R^2_{\text{adj}} = 1 - \left[ \frac{(1 - R^2)(n - 1)}{n - p - 1} \right] $$
   * Adjusted $R^2$ penalizes irrelevant features.
2. Information Criteria:
   * AIC = $2k - 2\ln(\hat{L})$
   * BIC = $k\ln(n) - 2\ln(\hat{L})$
3. Statistical Significance: t-statistic and F-statistic

---

### 3. Logistic Regression
* Log-Odds (Logit):
  $$ \ln\left(\frac{p}{1 - p}\right) = X\beta $$
* Key Distinctions from Linear Regression:
  * Estimated via **Maximum Likelihood Estimation (MLE)**.
  * Log-odds (not $y$) is linear in $X$.
  * Probability output $[0, 1]$.

* Multinomial Logistic Regression (Softmax Regression):
  * **Sigmoid** ($K = 2$, Binary Classification):
    $$ P(Y=1 \mid X) = \sigma(z) = \frac{1}{1 + e^{-X\beta}} $$
    $$ P(Y=0 \mid X) = 1 - P(Y=1 \mid X) $$
  * **Softmax** ($K \ge 3$, Multiclass Classification):
    $$ P(Y=k \mid X) = \frac{e^{X\beta_k}}{\sum_{j=1}^K e^{X\beta_j}} \quad \text{where } \sum_{k=1}^K P(Y=k \mid X) = 1 $$
  * Softmax outputs a probability distribution across $K$ classes. Sigmoid is Softmax when $K=2$.

---

## Principal Component Analysis (Dimensionality Reduction)

* Eigendecomposition:
  $$ \Sigma v_i = \lambda_i v_i \quad \Longleftrightarrow \quad \Sigma V = V \Lambda $$
  where $V = [v_1, v_2, \dots, v_p]$ is the orthogonal matrix of eigenvectors ($V^T V = I$), $\lambda_1 \ge \lambda_2 \ge \dots \ge \lambda_p \ge 0$, and $\Sigma$ is the sample covariance matrix.

* Principal Component Scores (Projected Data):
  $$ Z = X V \quad (Z_k = X v_k) $$

* **Explained Variance Ratio (EVR)**:
  $$ \text{EVR}_k = \frac{\lambda_k}{\sum_{j=1}^p \lambda_j} $$

* Preconditions & Data Requirements:
  1. Zero Mean (Mandatory): components represent directions of maximum variance.
  2. Standardization (Recommended)
  3. Linearity

* Matrix $\Lambda = \text{diag}(\lambda_1, \dots, \lambda_p)$:
  * Covariance of Transformed Features: $\text{Cov}(Z) = \frac{1}{n-1} Z^T Z = V^T \Sigma V = \Lambda$.
  * Principal components are **strictly orthogonal and uncorrelated**.
  * $\text{Var}(Z_k) = \lambda_k$.

* Quick Notes:
  * PCA can be reversed to recover the original data if $m=p$.
  * PCA works on linear problems.
  * PCA does not assume any distribution of variables. However:
    * If $X$ are jointly-normal, the PCs, which are linear combinations of normal variables, are also normal ($Z$ is jointly normal).

---

## Time Series Analysis

* **Stationarity (Weak / Covariance Stationary)**:
  * **Conditions**: Constant mean $E[y_t] = \mu$, constant variance $\text{Var}(y_t) = \sigma^2$, and autocovariance $\text{Cov}(y_t, y_{t-k})$ dependent only on lag $k$ (not time $t$).
  * **How to Check**:
    1. *Visual*: Non-decaying ACF (suggests non-stationarity) or trending rolling mean/variance.
    2. *Statistical Test — Augmented Dickey-Fuller (ADF)*:
       * Tests for a unit root: $\Delta y_t = \alpha + \beta t + \gamma y_{t-1} + \sum_{i=1}^p \delta_i \Delta y_{t-i} + \epsilon_t$.
       * $H_0: \gamma = 0$ (Unit root exists $\implies$ Non-Stationary).
       * $H_1: \gamma < 0$ (No unit root $\implies$ Stationary).
       * **ADF Statistic Interpretation**:
         * Test statistic is always negative; **more negative $\implies$ stronger rejection of unit root**.
         * **Statistic < Critical Value** (or **$p\text{-value} < 0.05$**): **Reject $H_0 \implies$ Series is Stationary**.
         * **Statistic > Critical Value** ($p \ge 0.05$): **Fail to reject $H_0 \implies$ Non-Stationary** (needs differencing).

* **ARIMA$(p, d, q)$**:
  * **Parameters**:
    * **$p$ (Auto-Regressive, AR)**: Number of lag observations included in the model ($y_t$ depends on past values $y_{t-1}, \dots, y_{t-p}$).
    * **$d$ (Integrated, I)**: Degree of differencing needed to achieve stationarity ($\Delta^d y_t = (1-B)^d y_t$).
    * **$q$ (Moving Average, MA)**: Number of lagged forecast error terms included ($y_t$ depends on past white noise shocks $\epsilon_{t-1}, \dots, \epsilon_{t-q}$).
  * **Equation**: $\phi(B)(1-B)^d y_t = c + \theta(B)\epsilon_t$.
  * **Common Specifications & Real-Life Data**:
    * **$\text{ARIMA}(1,0,0)$ — $\text{AR}(1)$**:
      * *Formula*: $y_t = c + \phi_1 y_{t-1} + \epsilon_t$ ($\lvert \phi_1 \rvert < 1$).
      * *Properties*: Mean-reverting process with exponentially decaying autocorrelation.
      * *Real-Life Fit*: **Short-term interest rates** (e.g., Vasicek model), **credit default swap (CDS) spreads**, and commodity mean-reverting basis.
    * **$\text{ARIMA}(0,1,0)$ — Random Walk**:
      * *Formula*: $y_t = y_{t-1} + c + \epsilon_t \implies \Delta y_t = c + \epsilon_t$.
      * *Properties*: Non-stationary; best prediction of tomorrow is today's level plus drift $c$.
      * *Real-Life Fit*: **Stock prices / FX exchange rates** under the Efficient Market Hypothesis.
    * **$\text{ARIMA}(1,1,0)$ — Differenced $\text{AR}(1)$**:
      * *Formula*: $\Delta y_t = c + \phi_1 \Delta y_{t-1} + \epsilon_t$.
      * *Properties*: Changes/returns exhibit momentum or autocorrelation before decaying.
      * *Real-Life Fit*: **Macroeconomic inflation rates**, **quarterly GDP levels**, and trending asset series.
    * **$\text{ARIMA}(0,0,1)$ — $\text{MA}(1)$**:
      * *Formula*: $y_t = c + \epsilon_t + \theta_1 \epsilon_{t-1}$.
      * *Properties*: Finite 1-period shock memory; autocorrelation cuts off abruptly after lag 1.
      * *Real-Life Fit*: **Microstructure bid-ask bounce** in high-frequency returns, inventory adjustment deviations.

* **Vector Autoregression (VAR)**:
  * **Definition**: A multivariate time-series extension of AR models that captures dynamic linear interdependencies among multiple time series.
  * **Core Concept**: Every variable is treated as endogenous and modeled as a function of its own lagged values and the lagged values of all other variables in the system.
  * **Model Form (for $k$ variables, $p$ lags)**:
    $$ Y_t = c + A_1 Y_{t-1} + A_2 Y_{t-2} + \dots + A_p Y_{t-p} + \epsilon_t $$
    where $Y_t \in \mathbb{R}^k$, $A_i$ are $k \times k$ coefficient matrices, and $\epsilon_t \sim \text{WN}(0, \Sigma)$.
  * **Requirements & Applications**:
    * Requires all series to be stationary (if cointegrated non-stationary series, use **VECM**).
    * Used for **macroeconomic policy analysis** (e.g., joint response of Fed Funds Rate, Inflation, and GDP growth), **cross-asset volatility spillover**, and **impulse response function (IRF)** analysis.

* **Autocorrelation & Diagnostics**:
  * Autocorrelation: Correlation between series $y_t$ and its lagged values $y_{t-k}$.
  * Durbin-Watson test: Detects first-order autocorrelation in residuals ($d \approx 2 \implies$ no autocorrelation, $d < 2 \implies$ positive autocorrelation).

---

## Statistical Distribution
* Independent vs Uncorrelated
  * Random Variables can be uncorrelated but not independent. Examples:
    * $Y = X^2$, where $X$ is normally distributed with 0 mean.
    * $Y = +X$ if $\lvert X \rvert < c$, $Y = -X$ if $\lvert X \rvert \geq c$
  * Independent: $P(X \mid Y) = P(X)$ and $P(Y \mid X) = P(Y) $
  * Uncorrelated: $Cov(X,Y) = 0 $

* Normal Distribution
  * 1-dimensional normal also known as Univariate Normal
  * $X_i$'s are Jointly Normal iff $a_1 X_1 + \dots + a_n X_n$ are Normal for ANY $a_i$
  * Multivariate Normal (Gaussian) $\implies$ any subset (even single variable) is also Gaussian
  * A given set of normal variables $X_i$ does not imply Multivariate Normality

---

## Hypothesis Testing & Standard Error

### 1. Standard Deviation (SD) vs. Standard Error (SE)
* **Standard Deviation ($s$)**: Measures the dispersion/variability of individual observations around the sample mean:
  $$ s = \sqrt{\frac{1}{N-1}\sum_{i=1}^N (x_i - \bar{x})^2} $$
* **Standard Error ($\text{SE}$)**: Measures the precision/variability of the sample mean estimator $\bar{x}$:
  $$ \text{SE} = \frac{s}{\sqrt{N}} $$
  * By the Central Limit Theorem (CLT), $\text{SE}$ decreases at rate $O(1/\sqrt{N})$ as sample size $N$ increases.

### 2. One-Sample $t$-Statistic
Tests whether an observed sample mean $\bar{x}$ differs significantly from a hypothesized population value $\mu_0$ (typically $H_0: \mu = \mu_0$):
$$ t = \frac{\bar{x} - \mu_0}{\text{SE}} = \frac{\bar{x} - \mu_0}{s / \sqrt{N}} $$

* **Intuition**:
  * Represents a **signal-to-noise ratio**: how many standard errors the sample mean lies away from the null hypothesis $\mu_0$.
  * Differentiates a **statistically significant effect** from **random sampling variation**.

* **Rule-of-Thumb Interpretations & Conclusions (Two-Tailed)**:
  * **$\lvert t \rvert < 1.0$ (Noise-Dominated)**:
    * The sample mean lies within $1\text{ SE}$ of $\mu_0$ ($<68\%$ coverage).
    * *Conclusion*: **Fail to reject $H_0$.** The deviation is well within expected sampling noise; no evidence of an effect.
  * **$1.0 \le \lvert t \rvert < 1.96$ (Borderline / Inconclusive)**:
    * The deviation exceeds $1\text{ SE}$ but remains within the $95\%$ confidence band ($\pm 1.96\,\text{SE}$).
    * *Conclusion*: **Fail to reject $H_0$ at $\alpha = 0.05$ ($p > 0.05$).** Statistically inconclusive; a larger sample size $N$ is required to detect small effects.
  * **$\lvert t \rvert \ge 1.96 \approx 2.0$ (Statistically Significant)**:
    * Falls in the critical rejection region at the $5\%$ level ($p \le 0.05$).
    * *Conclusion*: **Reject $H_0$.** Sampling variation alone is unlikely to explain the difference; evidence of a true non-zero effect.
  * **$\lvert t \rvert \ge 3.0$ (Highly Significant)**:
    * Far outside random chance ($p < 0.003$, $>99.7\%$ confidence level).
    * *Conclusion*: **Strongly reject $H_0$.** Strong evidence of a systematic effect.
     