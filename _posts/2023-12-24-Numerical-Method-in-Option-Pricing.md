---
title: Numerical Methods in Option Pricing
date: 2023-12-24 00:00:00 +0000
categories: [Quant]
tags: [Option Pricing, Numerical Methods, Valuation, PIU]
description: A practical guide to numerical integration and the Likelihood Ratio Method (LRM) for pricing complex equity derivatives and Profits Interest Units (PIUs).
pin: false
math: true
---

## Introduction

While the Black-Scholes formula provides exact analytical solutions for vanilla options, complex equity derivatives—such as **Profits Interest Units (PIUs)** with distribution hurdles and tiered vesting—often break closed-form tractability. In practice, valuation specialists rely on numerical methods. This post reviews common pricing frameworks, highlights the practical advantages of **composite trapezoidal numerical integration**, and demonstrates how to evaluate fair value and Delta simultaneously via the **Likelihood Ratio Method (LRM)**.

### Outline

* [**Ways to Price Vanilla Options**](#ways-to-price-vanilla-options)
* [**Composite Trapezoidal Numerical Integration**](#composite-trapezoidal-numerical-integration)
* [**Profits Interest Units Pricing under Numerical Integration**](#profits-interest-units-pricing-under-numerical-integration)
* [**Python Implementation**](#python-implementation)
* [**Key Takeaways**](#key-takeaways)

---

## Ways to Price Vanilla Options

### 1. Black-Scholes Closed-Form Formula

Assumes the underlying asset follows Geometric Brownian Motion and constructs a continuously delta-hedged riskless portfolio to obtain an exact analytical solution (zero numerical pricing error):

$$ C(S_0, K, T) = S_0 N(d_1) - K e^{-rT} N(d_2) $$

### 2. Monte Carlo Simulation

Estimates the option price by generating independent random paths of the underlying asset under the risk-neutral measure and computing the sample average of the discounted terminal payoffs (standard error converges at $\mathcal{O}(M^{-1/2})$):

$$ C_0 \approx e^{-rT} \frac{1}{M} \sum_{i=1}^{M} \max\left(S_T^{(i)} - K, \, 0\right) $$

### 3. Lattice Models (Binomial Trees)

Discretizes underlying price paths into a recombining tree to backward-induct European option values (evaluating $\max(\text{payoff}, \text{continuation})$ at each node for American exercise), converging at first-order rate $\mathcal{O}(\Delta t)$ with oscillatory errors driven by strike-node alignment:

$$ V_{i,j} = e^{-r \Delta t} \left[ p_u V_{i+1, j+1} + (1 - p_u) V_{i+1, j} \right] \quad \left(\text{or } V_{i,j} = \max\left( (S_{i,j} - K)^+, \, e^{-r \Delta t} \mathbb{E}[V_{i+1}] \right) \text{ for American}\right) $$

### 4. Fourier Transform Methods (Carr-Madan)

Inverts the damped option price ($e^{\alpha k} C_0(k)$ with $\alpha > 0$) via FFT to price across an entire strike grid simultaneously (frequency and log-strike spacings are linked by $\eta \lambda = 2\pi / N$, with polynomial integrand decay $\sim 1/u^2$ dictating truncation error alongside quadrature error):

$$ C_0(k) = \frac{e^{-\alpha k}}{\pi} \int_{0}^{\infty} \text{Re}\left( e^{-i u k} \psi(u) \right) du, \quad \psi(u) = \frac{e^{-rT} \phi(u - (\alpha + 1)i)}{\alpha^2 + \alpha - u^2 + i(2\alpha + 1)u} $$

### 5. Numerical Integration (Quadrature of Risk-Neutral Density)

Directly integrates the discounted payoff against the risk-neutral transition probability density function $q(S_T \mid S_0)$ using numerical quadrature ($\mathcal{O}(h^2)$ for composite trapezoidal, or exponential convergence for Gaussian quadrature on smooth domains):

$$ C_0 = e^{-rT} \int_{K}^{\infty} (S_T - K) q(S_T \mid S_0) \, dS_T \approx e^{-rT} \sum_{k=1}^{N_q} w_k \left(x_k - K\right) q\left(x_k \mid S_0\right) $$

#### Composite Trapezoidal vs. Gaussian Quadrature
For a call option, the composite trapezoidal rule approximates the density integral on a uniformly spaced grid $[K, S_{\max}]$ with step $h$. The integration grid starts at $K$ rather than $0$ because the call payoff $\max(S - K, 0)$ is identically zero for $S < K$, avoiding wasting nodes on zero-payoff regions.
$$ C_0 \approx e^{-rT} h \left( \frac{1}{2}f(K) + \sum_{k=1}^{N-1} f(x_k) + \frac{1}{2}f(S_{\max}) \right), \quad f(x) = (x-K)q(x \mid S_0) $$
While trapezoidal rules use uniform spacing and typically require hundreds of nodes ($500-2000$), Gaussian quadrature evaluates optimally chosen non-uniform nodes. Gaussian quadrature offers exponentially faster convergence with far fewer nodes ($20-100$) but requires sophisticated node and weight selection.

| Feature | Composite Trapezoidal | Gaussian Quadrature |
| :--- | :--- | :--- |
| **Nodes & Placement** | $500 - 2,000$ (Uniformly spaced) | $20 - 100$ (Non-uniform, orthogonal roots) |
| **Convergence** | $\mathcal{O}(h^2)$ | Exponential (for smooth integrands) |
| **Complexity** | Simple to implement | Requires sophisticated node picking |

---

## Composite Trapezoidal Numerical Integration

Numerical integration using the composite trapezoidal rule is simple and practical. Compared to Monte Carlo simulation and Gaussian quadrature:

1. vs. Monte Carlo
   * zero sampling noise
   * produces smooth prices and stable finite-difference Greeks for continuous payoffs
   * limited applicability for path-dependent and multi-asset products

2. vs. Gaussian Quadrature 
   * equal spacing, no domain-dependent rule selection
   * local error behavior: kinks are handled by placing a node on the strike, and discontinuities by splitting the domain
   * requires 1,000+ nodes to achieve high accuracy, while Gaussian quadrature needs only 20–100 nodes.

### Summary Comparison

| Method | Best suited for | Accuracy / convergence | Random noise | Main limitation |
| :--- | :--- | :--- | :--- | :--- |
| Monte Carlo | Path-dependent, high-dimensional | $\mathcal{O}(M^{-1/2})$ | Yes | Slow; noisy Greeks without extra techniques |
| Gaussian quadrature | Smooth integrands, known weights/domains | Exponential when smooth | None | Must split at kinks and discontinuities |
| Composite trapezoid | Any European payoff, single underlying, known density | $\mathcal{O}(h^2)$ plus truncation error; faster if smooth and decaying | None | Low dimension only; slower than Gauss when smooth |
---

## Profits Interest Units Pricing under Numerical Integration

**Profits Interest Units (PIUs)** can be viewed as European-style call options subject to partnership distribution hurdles and market/performance-based vesting conditions. In corporate equity compensation and financial reporting, these incentive awards are formally governed by **ASC 718** (Compensation—Stock Compensation).

> For accounting guidelines, valuation triggers, and mandatory financial reporting inputs under U.S. GAAP, see our companion guide on [ASC 718 Stock Compensation](/posts/ASC718/).

At an exit or liquidity event at time $T$, the general payoff $\Phi(S_T)$ of a Profits Interest Unit can be expressed mathematically as:

$$
\Phi(S_T) = \max\left(S_T - K, \, 0\right) \cdot \text{Vesting}(S_T) \cdot \mathbf{1}_{\{S_T \ge S_{\text{vest}}\}}
$$

and can be priced with composite trapezoidal rule as:

$$
\begin{aligned}
V_0 &= e^{-rT} \int_{\max(K, \, S_{\text{vest}})}^{S_{\max}} (S_T - K) \cdot \text{Vesting}(S_T) \cdot q(S_T \mid S_0) \, dS_T \\
&\approx e^{-rT} h \left( \frac{1}{2} f(x_0) + \sum_{k=1}^{N-1} f(x_k) + \frac{1}{2} f(x_N) \right)
\end{aligned}
$$

where $f(x) = (x - K) \cdot \text{Vesting}(x) \cdot q(x \mid S_0)$, with $x_0 = \max(K, S_{\text{vest}})$.

Beyond fair value, **Delta** ($\Delta = \frac{\partial V_0}{\partial S_0}$) is a critical metric in PIU valuation. It measures the sensitivity of the unit's value with respect to changes in the underlying equity price, reflects the unit's effective moneyness, and serves as a key input for downstream quantitative calculations—such as determining the unit's **leverage ratio** ($\Omega = \Delta \cdot \frac{S_0}{V_0}$) and **unit-specific volatility** ($\sigma_{\text{PIU}} = \Omega \cdot \sigma$).

Delta can be computed directly by differentiating the transition density under the integral:

$$
\begin{aligned}
\Delta &= e^{-rT} \int_{\max(K, \, S_{\text{vest}})}^{S_{\max}} (S_T - K) \cdot \text{Vesting}(S_T) \cdot \frac{\partial q(S_T \mid S_0)}{\partial S_0} \, dS_T \\
&\approx e^{-rT} h \left( \frac{1}{2} g(x_0) + \sum_{k=1}^{N-1} g(x_k) + \frac{1}{2} g(x_N) \right)
\end{aligned}
$$

where $g(x) = (x - K) \cdot \text{Vesting}(x) \cdot \frac{\partial q(x \mid S_0)}{\partial S_0}$.

By applying integration by parts (transferring the derivative from the non-smooth payoff to the density, or directly differentiating the lognormal transition density with respect to $S_0$), we get:

$$
\frac{\partial q(S_T \mid S_0)}{\partial S_0} = q(S_T \mid S_0) \cdot \frac{\ln(S_T / S_0) - \mu}{S_0 \sigma^2 T} = q(S_T \mid S_0) \cdot \frac{z(S_T)}{S_0 \sigma \sqrt{T}}
$$

where $\mu = (r - q - \frac{1}{2}\sigma^2)T$ and $z(S_T) = \frac{\ln(S_T / S_0) - \mu}{\sigma \sqrt{T}}$. In derivative pricing and quantitative finance, this technique of differentiating the density kernel rather than the non-smooth payoff is known as the **Likelihood Ratio Method (LRM)**.

---

## Python Implementation

Below is a simple Python implementation using `scipy.integrate.trapezoid` to compute the PIU price and Delta:

```python
import numpy as np
from scipy.stats import norm
from scipy.integrate import trapezoid

# Model and market inputs
S0, K, r, q, sigma, T = 100.0, 100.0, 0.05, 0.0, 0.30, 3.0
thresholds = np.array([150.0, 175.0, 200.0])
cum_weights = np.array([0.50, 0.75, 1.00])   # cumulative vesting fraction

# 1. Standard normal grid z ~ N(0, 1) and market parameters
z_grid = np.linspace(-5.0, 5.0, 1000)
sqrt_t = np.sqrt(T)
disc = np.exp(-r * T)

# 2. Terminal asset prices and PIU payoff values
mu = (r - q - 0.5 * sigma**2) * T
st_grid = S0 * np.exp(mu + sigma * sqrt_t * z_grid)

# Vesting schedule: 50%, 75%, 100% cliff vest when stock price hits 150, 175, 200
vesting = np.select([st_grid >= t for t in thresholds[::-1]], list(cum_weights[::-1]), default=0.0)
payoff_values = np.maximum(st_grid - K, 0.0) * vesting

# 3. Likelihood Ratio Method (LRM) weights
z_pdf = norm.pdf(z_grid)
delta_weight = z_pdf * z_grid / (S0 * sigma * sqrt_t)

# 4. Numerical integration for Price and Delta
price = trapezoid(payoff_values * disc * z_pdf, z_grid)
delta = trapezoid(payoff_values * disc * delta_weight, z_grid)
```

### Numerical Results

| $N_{\text{grid}}$ | Price | Delta | Price Error | Delta Error |
| :---: | :---: | :---: | :---: | :---: |
| 500 | 18.41092006 | 0.60100754 | 9.552e-02 | 1.573e-03 |
| 1,000 | 18.32258557 | 0.59959437 | 7.181e-03 | 1.600e-04 |
| 5,000 | 18.31322352 | 0.59940347 | 2.181e-03 | 3.086e-05 |
| 10,000 | 18.31738783 | 0.59943198 | 1.983e-03 | 2.345e-06 |
| 50,000 | 18.31588194 | 0.59941606 | 4.775e-04 | 1.827e-05 |

---

## Key Takeaways

* Beyond Black-Scholes: For complex payoffs like Profits Interest Units (PIUs), common practices rely on Black-Scholes extensions (combining multiple calls) or Monte Carlo simulation.
* Trapezoidal Numerical Integration is an advanced technique that specialists in valuation firms use as an alternative to Monte Carlo simulation when terms do not include path-dependent features, since:
  * Numerical integration decouples the payoff function and the probability distribution. With appropriate grid selection on the probability distribution, calculating multiple options with different vesting, strikes, and terms, etc., in one go is feasible.
  * The calculation is fast and deterministic, unlike the sampling variance from Monte Carlo simulation.
  * Delta, which measures the sensitivity of the unit's value with respect to changes in the underlying equity price, can be calculated using LRM along with the price in one run.



