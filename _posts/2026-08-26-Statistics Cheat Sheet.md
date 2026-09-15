---
title: Statistics Cheat Sheet
date: 2026-08-26 00:00:00 +0000
categories: [Quant]
tags: [Statistics]
description: Statistics Cheat Sheet
pin: false
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
1. Pairwise Pearson correlation ($|r| > 0.8$ implies severe collinearity)
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
    $$ P(Y=1|X) = \sigma(z) = \frac{1}{1 + e^{-X\beta}} $$
    $$ P(Y=0|X) = 1 - P(Y=1|X) $$
  * **Softmax** ($K \ge 3$, Multiclass Classification):
    $$ P(Y=k|X) = \frac{e^{X\beta_k}}{\sum_{j=1}^K e^{X\beta_j}} \quad \text{where } \sum_{k=1}^K P(Y=k|X) = 1 $$
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
* Autocorrelation - mean of a series at time $t$ is related to mean of series at time $t-k$
  * Check with Durbin-Watson test.

## Statistical Distribution
* Independent vs Uncorrelated
  * Random Variables can be uncorrelated but not independent. Examples:
    * $Y = X^2$, where $X$ is normally distributed with 0 mean.
    * $Y = +X$ if $|X| < c$, $Y = -X$ if $|X| \geq c$
  * Independent: $P(X|Y) = P(X)$ and $P(Y|X) = P(Y) $
  * Uncorrelated: $Cov(X,Y) = 0 $

* Normal Distribution
  * 1-dimensional normal also known as Univariate Normal
  * $X_i$'s are Jointly Normal iff $a_1 X_1 + \dots + a_n X_n$ are Normal for ANY $a_i$
  * Multivariate Normal (Gaussian) $\implies$ any subset (even single variable) is also Gaussian
  * A given set of normal variables $X_i$ does not imply Multivariate Normality
     