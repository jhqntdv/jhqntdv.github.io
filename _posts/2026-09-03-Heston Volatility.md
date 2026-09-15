---
title: "Heston Stochastic Volatility: Simulation, Pricing & Calibration"
date: 2026-08-25 00:00:00 +0000
categories: [Quant]
tags: [Heston Model, Stochastic Volatility, Andersen QE, Euler Full Truncation, Variance Swaps, Fourier Pricing, Monte Carlo, Calibration, PnL Attribution]
description: "A quantitative implementation of the Heston stochastic volatility model, covering Euler full truncation and Andersen QE simulation, martingale corrections, variance swap pricing, Fourier inversion, Monte Carlo diagnostics, calibration, and PnL attribution."
pin: true
math: true
---

## Introduction

This article covers the implementation, simulation, and pricing of derivatives under the Heston stochastic volatility model:

* [**Part 1: Heston Model Mechanics & Simulation Schemes**](#heston-model-mechanics--simulation-schemes): Heston SDE formulation, Feller condition, QE simulation, martingale corrections, and simulation diagnostics.
* [**Part 2: Pricing Variance Swaps**](#pricing-variance-swaps): Variance swap fundamentals, closed-form strike, Carr-Madan model-free pricing, and simulation error analysis.
* [**Part 3: Pricing European Vanilla Options & Calibration**](#pricing-european-vanilla-options--calibration): Fourier pricing, Monte Carlo diagnostics, Greeks, put-call parity, and Heston model calibration.
* [**Part 4: Appendix & Advanced Frontiers**](#appendix--advanced-frontiers): Local Stochastic Volatility (LSV) and rough volatility.

---

## 2. Heston Model Mechanics & Simulation Schemes

### Heston Model SDE Formulation
The Black-Scholes model assumes constant volatility, which does not capture the volatility skew and heavy tails observed in equity returns. The Heston model addresses this by introducing a stochastic process for variance, resulting in a bivariate SDE under the risk-neutral measure $\mathbb{Q}$:

$$ dS_t = rS_t dt + \sqrt{v_t} S_t dW_t^{(1)} $$
$$ dv_t = \kappa(\theta - v_t)dt + \xi \sqrt{v_t} dW_t^{(2)} $$

with $dW_t^{(1)} dW_t^{(2)} = \rho dt$. The variance process $v_t$ follows Cox-Ingersoll-Ross (CIR) dynamics, governed by the mean-reversion speed $\kappa$, long-term variance $\theta$, and the volatility-of-volatility $\xi$. In equity markets, $\rho$ is typically negative (around -0.7), generating the empirical leverage effect: falling asset prices coincide with variance spikes, producing the negative skew observed in index option smiles.

### The Feller Condition & Boundary Behavior
A key property of the CIR process is the Feller condition, $2\kappa\theta > \xi^2$. When this condition holds, the upward drift near zero guarantees that $v_t$ remains strictly positive. However, in equity calibrations where volatility-of-volatility ($\xi$) is high, this condition is frequently violated ($2\kappa\theta < \xi^2$).

When the Feller condition is violated, the variance process frequently hits the zero boundary. Standard Euler Full Truncation discretizes the process by applying a $\max(v_t, 0)$ cutoff to both the drift and diffusion terms:

$$ \tilde{v}_{t+\Delta t} = \tilde{v}_t + \kappa(\theta - \max(\tilde{v}_t, 0))\Delta t + \xi \sqrt{\max(\tilde{v}_t, 0)} \Delta W_t $$

Because the un-truncated state $\tilde{v}_t$ can drift below zero, the flooring operation $\max(\tilde{v}_t, 0)$ is repeatedly invoked along the path, creating an artificial accumulation of probability mass at (or near) zero.

### Andersen's Quadratic-Exponential (QE) Scheme
To resolve the boundary artifacts, Andersen's Quadratic-Exponential (QE) scheme abandons the standard Gaussian step. Instead, it adaptively switches between two conditional distributions that closely match the true non-central chi-square distribution of the CIR process. The algorithm computes the exact conditional mean ($m$) and variance ($s^2$) of $v_{t+\Delta t}$, then bases its switching logic on the normalized variance ratio $\psi = s^2 / m^2$.

For the **quadratic branch** ($\psi \le \psi_c$, with $\psi_c = 1.5$), the scheme uses a shifted quadratic Gaussian approximation:

$$ v_{t+\Delta t} = a(b + Z)^2, \quad Z \sim N(0,1) $$

where $a$ and $b$ are determined by moment-matching to $m$ and $s^2$. The squaring naturally ensures the variance remains strictly non-negative. This regime is mathematically valid as long as $\psi \le 2$. 

Conversely, for the **exponential branch** ($\psi > \psi_c$), the shifted quadratic approximation fails. The scheme instead switches to an exponential [mixture](https://en.wikipedia.org/wiki/Mixture_model) with a distinct probability mass $p$ at zero:

$$ \mathbb{P}(v_{t+\Delta t} \in [0, x]) = p + (1-p)(1 - e^{-\beta x}), \quad x \ge 0 $$

The exponential mixture provides a numerically convenient approximation to the near-zero behavior of the CIR transition distribution. The point mass at zero is a feature of the QE approximation, rather than an exact representation of the continuous-time CIR transition density. It is mathematically valid for $\psi \ge 1$. The threshold $\psi_c = 1.5$ is chosen deliberately as the midpoint of the overlapping valid interval $[1, 2]$, ensuring smooth numerical transitions between the two branches.

**Implementation Workflow:**
1. **Compute Moments:** For each step and path, compute the true mean $m$ and variance $s^2$ of $v_{t+\Delta t}$ given the current state $v_t$.
2. **Evaluate Switch:** Compute $\psi = s^2 / m^2$.
3. **Quadratic branch ($\psi \le \psi_c$):** Solve for $a$ and $b$, draw a standard normal random variable $Z_V$, and compute the next variance state.
4. **Exponential branch ($\psi > \psi_c$):** Compute the zero-mass probability $p$ and exponential parameter $\beta$. Draw a uniform random variable $U_V$. If $U_V \le p$, the variance hits exactly zero. Otherwise, inverse-transform $U_V$ to sample from the exponential tail.

```python
def _qe_variance_step(self, v_curr, dt, c1, c2, c3):
    """Vectorized QE scheme for variance evolution."""
    m = c1 + v_curr * np.exp(-self.params.kappa * dt)
    s2 = v_curr * c2 + c3
    psi = s2 / np.maximum(m, 1e-300)**2

    Zv = self.rng.standard_normal(v_curr.shape[0])
    Uv = self.rng.random(v_curr.shape[0])
    v_next = np.zeros_like(v_curr)

    low_mask = psi <= 1.5
    high_mask = ~low_mask

    # Branch 1: Quadratic branch (psi <= 1.5)
    if np.any(low_mask):
        psi_low = psi[low_mask]
        inv_psi = 1.0 / np.maximum(psi_low, 1e-300)
        b2 = 2.0 * inv_psi - 1.0 + np.sqrt(2.0 * inv_psi) * np.sqrt(np.maximum(2.0 * inv_psi - 1.0, 0.0))
        a = m[low_mask] / (1.0 + b2)
        v_next[low_mask] = a * (np.sqrt(np.maximum(b2, 0.0)) + Zv[low_mask])**2

    # Branch 2: Exponential branch (psi > 1.5)
    if np.any(high_mask):
        psi_high = psi[high_mask]
        p = (psi_high - 1.0) / (psi_high + 1.0)
        beta = (1.0 - p) / np.maximum(m[high_mask], 1e-300)
        
        u_high = Uv[high_mask]
        positive = u_high > p
        
        values = np.zeros_like(u_high)
        values[positive] = np.log((1.0 - p[positive]) / np.maximum(1.0 - u_high[positive], 1e-300)) / beta[positive]
        v_next[high_mask] = values

    return v_next
```

### Martingale Corrections: Local vs. Global Approaches
Simulating the joint $(S_t, v_t)$ process introduces drift error: discretizing the integrated variance in the log-spot SDE biases the asset price drift. Without correction, $\mathbb{E}[S_{t+\Delta t} \mid \mathcal{F}_t] \neq S_t e^{r\Delta t}$, meaning simulated asset prices systematically undershoot the theoretical forward.

Andersen proposes a local, path-by-path analytical drift correction, referred to as **Andersen's Conditional Martingale Correction**. By conditioning on $v_t$ and $v_{t+\Delta t}$, the discrete SDE is constrained to be a martingale by replacing the deterministic drift coefficient $K_0$ with a state-dependent term $K_0^*(v_t)$:

$$ K_0^*(v_t) = -\ln M(A; v_t) - \left(K_1 + \tfrac{1}{2}K_3\right)v_t $$

where $M(A; v_t)$ is the conditional moment generating function (MGF) of $v_{t+\Delta t}$, which admits a closed-form solution under the QE scheme. Because this correction is evaluated per-path, it is computationally intensive.

```python
# 1. Evolve Variance (QE Scheme)
v_next, low_mask, b2, a, p, beta = self._qe_variance_step(v_curr, dt, c1, c2, c3)

# 2. Andersen's Martingale Correction
if correction == PriceCorrection.ANDERSEN:
    K0, K1, K2, K3, K4 = self._qe_price_coefficients(dt)
    K0_star = self._andersen_martingale_corrected_k0(
        v_curr, low_mask, b2, a, p, beta, K1, K2, K3, K4
    )
    
    # 3. Evolve Log-Spot SDE
    Zs = self.rng.standard_normal(v_curr.shape[0])
    log_ret = (
        (r - q) * dt
        + K0_star
        + K1 * v_curr
        + K2 * v_next
        + np.sqrt(np.maximum(K3 * v_curr + K4 * v_next, 0.0)) * Zs
    )
```

An alternative is the **Empirical Martingale Simulation (EMS)** by Duan and Simonato, which applies a global, cross-sectional rescaling across all simulated paths ($\tilde{S}_{t_k}^{(m)} = S_{t_k}^{(m)} \times S_0 e^{rt_k}/\bar{S}_{t_k}$) to enforce the theoretical forward price. While EMS requires maintaining all paths in memory, it satisfies put-call parity and provides built-in variance reduction.

### Simulation Diagnostics & Error Analysis

When pricing variance-dependent derivatives via Monte Carlo, the total pricing error can be decoupled into two components: the numerical bias of the **discretization scheme** and the structural discrepancy from the **monitoring frequency**. Discretization error is a numerical artifact of simulating continuous SDEs over discrete time steps $\Delta t$. Under Feller violation, the Euler Full Truncation scheme exhibits $O(\sqrt{\Delta t})$ weak convergence, leaving bias even with thousands of steps. In contrast, the QE scheme achieves $O(\Delta t^2)$ convergence. As illustrated in the chart below, Euler's bias grows sharply, approaching an exponential blow-up as the Feller ratio approaches zero, while the QE scheme remains largely flat, with only a marginal uptick as the ratio nears the extreme boundary—reflecting its near-exact matching of the CIR transition density even under severe violation.

![Feller Ratio vs Bias](/assets/img/posts/heston/feller_error_chart.png)

#### The Impact of Feller Condition Violations

When the Feller condition is breached ($2\kappa\theta < \xi^2$), the continuous CIR variance process tests the zero boundary. Numerical methods like the Euler Full Truncation scheme force the variance to remain positive via arbitrary fixes, such as applying a $\max(\tilde{v}_t, 0)$ floor. This truncation distorts the transitional distribution, especially the accumulation of probability mass at zero. Because the variance trajectory governs the asset's return profile, this boundary distortion injects bias into path-dependent derivative pricing. Schemes like QE avoid this by matching the non-central chi-square boundary behavior, modeling the probability of exact zero hits.

#### Continuous Integral vs. Discretized Sum

Even if the discretization scheme is mathematically perfect, a gap remains between the theoretical continuous variance and the tradable contract. The theoretical model evaluates continuous integrated variance $\frac{1}{T}\int_0^T v_t dt$. Conversely, real-world variance swaps are settled against the discretely monitored sum of squared log returns:

$$ \sigma_{\text{realized}}^2 = \frac{A}{N} \sum_{i=1}^N \left( \ln \frac{S_{t_i}}{S_{t_{i-1}}} \right)^2 $$

The discrete sum inherently captures higher-order cross-variation terms and localized drift effects that are smoothed out in the continuous integral. While this discrepancy scales down at $O(\Delta t)$ as monitoring becomes more frequent, it remains a structural gap for daily monitored contracts. The table below isolates discretization bias from the monitoring frequency gap ($N = 10,000$ paths):

| Component | FT (bps) | QE (bps) | Root Cause |
| :--- | :--- | :--- | :--- |
| Continuous MC - Analytical | 4.085750 | -3.768472 | Euler truncation failure (Feller < 1) |
| Discrete Sum - Continuous MC | 0.157771 | 3.220665 | Sampling (Itô squared drift term) |
| Total Net Bias vs. Continuous Benchmark | 4.243521 | -0.547807 | Combined effect |

Note: At $N = 10,000$, these bias estimates carry standard errors of $\approx 5.3$ bps—comparable to the point estimates themselves—and are not yet statistically distinguishable from zero. See the $N = 100,000$ results below for a more reliable decomposition with suppressed sampling noise.

Increasing the simulation sample size to $N = 100,000$ paths reduces Monte Carlo sampling noise, revealing that the QE scheme's net bias (-0.07 bps) is not statistically distinguishable from zero at this sample size, whereas Full Truncation's bias (6.21 bps) remains statistically significant, confirming a genuine structural truncation effect rather than a sampling artifact:

| Component | FT (bps) | QE (bps) | Root Cause |
| :--- | :--- | :--- | :--- |
| Continuous MC - Analytical | 4.999492 | -0.837943 | Euler truncation failure (Feller < 1) |
| Discrete Sum - Continuous MC | 1.211601 | 0.771763 | Sampling (Itô squared drift term) |
| Total Net Bias vs. Continuous Benchmark | 6.211094 | -0.066181 | Combined effect |

Note: The bias figures above are derived from pricing a Variance Swap benchmark, introduced in the next section.

---

## 3. Pricing Variance Swaps

### Fundamentals of Variance Swaps
[Variance swaps](https://en.wikipedia.org/wiki/Variance_swap) are OTC contracts that allow desks and hedge funds to trade volatility directly without delta-hedging an options portfolio. The payoff is linear in realized variance: $N_{\text{var}} \times (\sigma_R^2 - K_{\text{var}})$. These are heavily traded in equity index and FX markets because they provide clean exposure to the volatility risk premium and are mathematically easier to replicate than volatility swaps.

### Closed-Form Strike Formula
Under continuous monitoring, the fair variance strike $K_{\text{var}}$ under the Heston model can be evaluated by taking the expectation of the CIR integrated variance. By solving the ordinary differential equation $\frac{d\mathbb{E}[v_t]}{dt} = \kappa(\theta - \mathbb{E}[v_t])$, we obtain a compact closed-form expression:

$$ K_{\text{var}} = \theta + (v_0 - \theta)\frac{1 - e^{-\kappa T}}{\kappa T} $$

This strike depends only on the mean-reversion and long-term variance parameters. It is independent of vol-of-vol $\xi$ and correlation $\rho$. Because it is exact, it serves as a benchmark for testing discretization bias in variance simulators.

### Model-Free Pricing: The Carr-Madan Spanning Formula

Carr and Madan (1998) showed that the expected integrated variance over $[0, T]$ can be replicated model-independently via a static portfolio of out-of-the-money European puts and calls across a continuum of strikes:

$$
\mathbb{E}\left[\frac{1}{T}\int_0^T \sigma_t^2 \, dt\right] = \frac{2}{T}\left[
\int_0^{F} \frac{1}{K^2} P(K) \, dK + \int_{F}^{\infty} \frac{1}{K^2} C(K) \, dK
\right]
$$

where:
- $F$ is the forward price of the underlying at maturity $T$
- $P(K)$ and $C(K)$ are the time-$0$ prices of European puts and calls struck at $K$
- The integral is split at the forward price $F$, using puts for $K < F$ and calls for $K \ge F$

This result holds **regardless of the underlying's dynamics**, provided the price process is continuous (no jumps) and options exist at all strikes $K \in (0, \infty)$. This framework is the theoretical basis of the CBOE VIX index. In practice, approximation errors arise from the discrete spacing of strike prices and the truncation of the integral tails.

### Immunity to Asset-Level Martingale Errors
Variance swap pricing is independent of asset-price drift errors. Under continuous monitoring, the payoff is purely a function of the integrated $v_t$ path; the asset price $S_t$ is never evaluated. Even under discrete monitoring, where realized variance is the sum of squared log returns, the drift error term scales at $O(\Delta t^2)$, which becomes negligible over the path. This explains why fixing the variance boundary with the QE scheme is sufficient for variance swaps, even without applying EMS or Andersen's martingale asset correction.

#### Variance Swap Pricing ($N = 10,000$ Paths)

| Pricing Method | Realized Variance ($K_{\text{var}}$) | Standard Error (SE) | Realized Volatility | Bias vs. Benchmark (bps) | Relative Error (%) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Benchmark | 0.064039 | 0.000000 | 25.305873 | 0.000000 | 0.000000 |
| FT (Continuous Integral) | 0.064447 | 0.000525 | 25.386472 | 4.085750 | 0.638012 |
| FT (Discrete Daily Sum) | 0.064463 | 0.000530 | 25.389579 | 4.243521 | 0.662649 |
| QE (Continuous Integral) | 0.063662 | 0.000530 | 25.231305 | -3.768472 | -0.588468 |
| QE (Discrete Daily Sum) | 0.063984 | 0.000546 | 25.295047 | -0.547807 | -0.085543 |

#### Variance Swap Pricing ($N = 100,000$ Paths)

| Pricing Method | Realized Variance ($K_{\text{var}}$) | Standard Error (SE) | Realized Volatility | Bias vs. Benchmark (bps) | Relative Error (%) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Benchmark | 0.064039 | 0.000000 | 25.305873 | 0.000000 | 0.000000 |
| FT (Continuous Integral) | 0.064539 | 0.000169 | 25.404462 | 4.999492 | 0.780698 |
| FT (Discrete Daily Sum) | 0.064660 | 0.000171 | 25.428297 | 6.211094 | 0.969897 |
| QE (Continuous Integral) | 0.063955 | 0.000169 | 25.289311 | -0.837943 | -0.130849 |
| QE (Discrete Daily Sum) | 0.064032 | 0.000173 | 25.304565 | -0.066181 | -0.010334 |

At 100,000 paths (N = 100,000), the QE scheme's bias (-0.07 to -0.84 bps) is not statistically distinguishable from zero (absolute t-stat < 1), consistent with its accurate matching of the CIR transition density even under severe Feller violation. In contrast, Full Truncation exhibits statistically significant bias (5.0 to 6.2 bps, t-stat approximately 3.0 to 3.6), confirming that Euler truncation introduces a systematic error that persists regardless of sample size.

![Variance Swap Step Size Analysis](/assets/img/posts/heston/step_size_analysis_var_swap.png)

---

## 4. Pricing European Vanilla Options & Calibration

We use a baseline 1-year at-the-money option ($S_0 = K = 120, r=2\%$). The Heston model is configured to simulate a high-volatility regime ($\sqrt{v_0} = 30\%$) mean-reverting ($\kappa = 1.70$) to a lower long-term volatility ($\sqrt{\theta} = 20\%$). By applying negative correlation ($\rho = -0.70$) and high vol-of-vol ($\xi = 0.60$), we capture typical equity index skew and fat tails. This configuration violates the Feller condition ($2\kappa\theta = 0.136 < \xi^2 = 0.36$), serving as a stress test for our numerical simulations ($N=100,000$ paths, $252$ steps).

### Option Pricing Dynamics and Semi-Analytical Solutions
European option payoffs depend entirely on the terminal distribution of $S_T$. Under Heston, the non-constant variance, combined with correlation and vol-of-vol, results in a skewed, fat-tailed terminal distribution.

The Heston model allows for semi-analytical vanilla option pricing via Fourier inversion. Similar to the Black-Scholes framework, the price of a European call option can be expressed as:

$$ C(S, K, T) = S_0 P_1 - K e^{-rT} P_2 $$

where $P_2$ represents the risk-neutral probability that the option expires in-the-money, and $P_1$ is the corresponding probability under the stock measure. 

While the probability density function of the terminal spot price is unknown, its characteristic function $\phi(u)$ can be derived in closed form by solving the pricing PDE using an affine guess. The **Gil-Pelaez inversion theorem** allows us to compute the required probabilities $P_j$ directly from the characteristic functions via numerical integration:

$$ P_j = \frac{1}{2} + \frac{1}{\pi} \int_0^\infty \text{Re}\left[ \frac{e^{-iu \ln K} \phi_j(u)}{iu} \right] du \quad \text{for } j=1, 2 $$

where $\phi_1(u)$ and $\phi_2(u)$ are the characteristic functions evaluated under the respective probability measures. Care must be taken to use the Albrecher formulation to handle the complex logarithm branch-cut discontinuity in $\phi_j(u)$, which otherwise causes numerical instability at long maturities. The infinite integral is truncated and evaluated using Gauss-Legendre quadrature (e.g., 64 or 128 points), providing sub-millisecond pricing. Furthermore, Greeks like Delta and Variance Vega ($\partial C/\partial v_0$) can be evaluated directly by differentiating under the Fourier integral.

### Monte Carlo Diagnostics

The table below compares our Monte Carlo pricer to the semi-analytical Fourier inversion benchmark for the ATM option configured earlier ($N = 100,000$ paths). While the Put estimate remains within one standard error ($z \approx 0.96$), the uncorrected Call exhibits a persistent constant dollar drift ($z \approx 3.19$, outside the 95% confidence interval $\text{Price}_{\text{MC}} \pm 1.96\,\text{SE}$), illustrating how discretization drift manifests when martingale corrections are omitted.

| Option Type | Analytical Price | MC Price | MC Error | MC Std Error |
| :--- | :--- | :--- | :--- | :--- |
| Call | 12.0277 | 12.1804 | 0.1526 | 0.0478 |
| Put | 9.6516 | 9.7035 | 0.0519 | 0.0541 |

Estimating Greeks via Monte Carlo finite difference (bump-and-reprice) requires care. While first-order spot Greeks ($\Delta$) are stable, variance and parameter sensitivities (e.g., Variance Vega $\frac{\partial C}{\partial v_0}$, $\frac{\partial C}{\partial \kappa}$) can be noisy. A parameter bump reshapes the variance trajectory and can push paths across the QE scheme's critical threshold ($\psi_c = 1.5$). This switch between quadratic and exponential distributions introduces discontinuities in the simulated payoff. To obtain stable estimates, implementing Common Random Numbers (CRN) is required, as demonstrated in the analytical versus Monte Carlo Greeks comparison table below:

| Method | Type | Delta ($\Delta$) | Gamma ($\Gamma$) | Rho ($\rho$) | Theta ($\Theta$) | Var Vega ($\partial C/\partial v_0$) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Analytical | Call | 0.680689 | 0.013656 | 69.654981 | -0.013578 | 43.401887 |
| Analytical | Put | -0.319311 | 0.013656 | -47.968860 | -0.007133 | 43.401887 |
| Monte Carlo | Call | 0.679114 | 0.013680 | 69.345927 | -0.013397 | 42.700574 |
| Monte Carlo | Put | -0.321725 | 0.013680 | -48.277914 | -0.006110 | 42.025733 |

#### Put-Call Parity as a Litmus Test

When evaluating Monte Carlo option pricers, put-call parity acts as a first-moment test. For European options, the model-free no-arbitrage identity must hold regardless of the underlying volatility dynamics:

$$ C(K, T) - P(K, T) = S_0 e^{-qT} - K e^{-rT} $$

Without a martingale correction, discretization of the Heston spot process can introduce a first-moment drift error, which directly appears as a put-call parity violation. Andersen's conditional correction removes this drift error at the conditional level; residual parity deviations are then primarily attributable to finite Monte Carlo sampling error and any remaining implementation/discretization effects. EMS enforces the unconditional forward exactly by construction.

| Method | Call | Put | $C - P$ | Target | Parity Error |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Heston Analytical | 12.027745 | 9.651586 | 2.376159 | 2.376159 | 0.000000 |
| Heston MC (FT None) | 12.180358 | 9.703519 | 2.476839 | 2.376159 | 0.100680 |
| Heston MC (FT EMS) | 12.112086 | 9.735926 | 2.376159 | 2.376159 | 0.000000 |
| Heston MC (QE Andersen) | 11.932602 | 9.725950 | 2.206652 | 2.376159 | -0.169507 |
| Black-Scholes Analytical | 5.557349 | 3.181189 | 2.376159 | 2.376159 | 0.000000 |

#### Asian Call Option Pricing & Martingale Correction Diagnostics

To evaluate how forward drift leakage affects path-dependent derivatives, we price an arithmetic Asian call option ($\text{Payoff} = e^{-rT} \max(\frac{1}{N}\sum_{i=1}^N S_{t_i} - K, 0)$) across 30 independent Monte Carlo seeds ($N = 5,000$ paths per seed) using Common Random Numbers (CRN). The table below compares the resulting option prices, first- and second-order Greeks, and estimator variances across the simulation schemes:

| Scheme | Price | Delta ($\Delta$) | Gamma ($\Gamma$) | Vega ($\mathcal{V}$) | Vega Std | True SE | Naive SE |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Heston FT (None) | 7.6188 | 0.6239 | 0.021035 | 34.0391 | 1.1032 | 0.1334 | 0.1356 |
| Heston QE (EMS) | 7.5912 | 0.6258 | 0.021618 | 34.4584 | 0.4717 | 0.0903 | 0.1342 |
| Heston QE (Andersen M-Corr) | 7.6181 | 0.6270 | 0.021500 | 34.6572 | 0.9500 | 0.1125 | 0.1345 |

The forward drift bias $\mathbb{E}[S_t] - S_0 e^{(r-q)t}$ across the simulated paths is visualized below:

![Forward Drift Martingale Deviation](/assets/img/posts/heston/drift_correction.png)

The forward drift trajectory confirms this decomposition visually: the FT (None) trajectory exhibits a persistent, direction-consistent drift bias that lies clearly outside the zero line even after accounting for Monte Carlo noise (shaded 95% confidence band across 30 independent seeds). In contrast, the Andersen (M-Corr) trajectory, despite showing a smooth deviation from zero due to the autocorrelation structure of path noise, remains fully contained within its 95% confidence band throughout the entire path. This indicates that the apparent drift is statistically indistinguishable from sampling noise, consistent with the conditional martingale property being correctly enforced at each time step. Meanwhile, the EMS trajectory tracks zero exactly by construction via cross-sectional rescaling.

### Model Calibration Workflow
Calibrating the parameters involves minimizing a loss function against market-quoted option surfaces. The calibration workflow is structured around the following steps:

1. **Analytical Pricing:** Inside the optimization loop, the objective function is evaluated thousands of times. Since Monte Carlo simulation is computationally intensive and noisy for gradient-based solvers, the calibration uses the semi-analytical Fourier inversion method to compute option prices.
2. **Two-Stage Optimization:** The parameter space is non-convex and prone to local minima. The workflow uses a two-stage approach: it first applies a global differential evolution optimizer to find a general parameter neighborhood, and then passes that result to a local gradient-based solver for final convergence.
3. **Relative Loss Weighting (MSRE):** Minimizing a standard sum of squared errors overweights in-the-money options and underweights out-of-the-money options. To address this, the calibration uses Mean Squared Relative Error (MSRE). By dividing the pricing error by the market price (capped by a small floor value), the optimizer weights the out-of-the-money wings, which contain volatility skew information:

    ```python
    if config.loss_type == "msre":
        weights = 1.0 / np.maximum(market_prices, config.price_floor_for_mape)
        total_loss += float(np.sum((diff * weights) ** 2))
    ```
4. **Soft Feller Penalties:** Enforcing the Feller condition as a strict boundary constraint can prevent the optimizer from fitting equity skews. Instead, a soft quadratic penalty is added to the loss function, allowing the solver to violate the condition if it improves the fit. This is critical in our validation setup because the true target parameters themselves violate the Feller condition ($2\kappa\theta = 0.136 < \xi^2 = 0.36$); a hard constraint would render the true optimum unreachable by construction.

To validate this workflow, we test the calibration against a synthesized option surface generated from our baseline parameters. This assesses how well the optimizer recovers the parameters from prices alone. The optimization converged with an RMSE of 0.3830, an MAE of 0.2545, and a Mean Absolute Percentage Error (MAPE) of 1.81%. 

The table below contrasts the calibrated parameters against the true underlying targets:

| Parameter | Target | Calibrated | Error (%) |
| :--- | :--- | :--- | :--- |
| $\kappa$ | 1.70 | 1.6628 | 2.19% |
| $\theta$ | 0.04 | 0.0401 | 0.36% |
| $\xi$ | 0.60 | 0.6013 | 0.22% |
| $\rho$ | -0.70 | -0.6990 | 0.14% |
| $v_0$ | 0.09 | 0.0897 | 0.35% |

The two-stage calibration achieves high parameter recovery across the board, with all parameter errors remaining under 2.2% (and under 0.4% for $\theta$, $\xi$, $\rho$, and $v_0$), demonstrating that combining global differential evolution with MSRE loss effectively navigates the non-convex surface to recover the ground-truth parameters even under Feller violation.

<div class="row g-3 mb-4" style="display: flex; gap: 15px; margin-bottom: 1.5rem;">
  <div class="col-md-6" style="flex: 1; min-width: 0;">
    <img src="/assets/img/posts/heston/vol_surface_heston.png" alt="Heston Implied Volatility Surface" class="w-100 rounded" style="width: 100%; height: auto;" />
  </div>
  <div class="col-md-6" style="flex: 1; min-width: 0;">
    <img src="/assets/img/posts/heston/vol_smile_heston.png" alt="Heston Implied Volatility Smile" class="w-100 rounded" style="width: 100%; height: auto;" />
  </div>
</div>

![Market vs Model Price After Calibration](/assets/img/posts/heston/market_model_price_after_calibration.png)

### Application: PnL Attribution (Greek Explain)
An application of Greek estimation is Profit and Loss (PnL) attribution. By expanding the portfolio's value change via Taylor series, we can break down the total daily PnL into buckets driven by market moves—often referred to as "Greek Explain." 

In the Heston framework, this involves mapping the change in option value to Delta (spot move), Gamma (spot curvature), Variance Vega/Volga (variance moves), and Theta (time decay). An accurate pricer ensures that the sum of these Greek-explained PnL components matches the actual observed PnL of the re-priced option, leaving a small residual.

#### Target vs Optimized PnL Attribution

| $T$ | $K$ | Parameters | Actual PnL | Delta ($\Delta$) | Gamma ($\Gamma$) | Var Vega ($\partial C/\partial v_0$) | Theta ($\Theta$) | Residual |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 0.5 | 100.0 | **Target** | 1.5256 | 1.5960 | 0.0097 | -0.0667 | -0.0169 | 0.0035 |
| | | **Opt.** | 1.5252 | 1.5962 | 0.0097 | -0.0672 | -0.0170 | 0.0035 |
| 0.5 | 120.0 | **Target** | 1.0497 | 1.1677 | 0.0286 | -0.1263 | -0.0201 | -0.0003 |
| | | **Opt.** | 1.0496 | 1.1686 | 0.0286 | -0.1273 | -0.0201 | -0.0002 |
| 0.5 | 140.0 | **Target** | 0.3237 | 0.3976 | 0.0365 | -0.0872 | -0.0107 | -0.0124 |
| | | **Opt.** | 0.3223 | 0.3966 | 0.0365 | -0.0876 | -0.0107 | -0.0125 |
| 1.0 | 100.0 | **Target** | 1.4904 | 1.5727 | 0.0085 | -0.0817 | -0.0115 | 0.0024 |
| | | **Opt.** | 1.4899 | 1.5732 | 0.0085 | -0.0826 | -0.0115 | 0.0024 |
| 1.0 | 120.0 | **Target** | 1.1032 | 1.2252 | 0.0221 | -0.1302 | -0.0136 | -0.0003 |
| | | **Opt.** | 1.1032 | 1.2268 | 0.0221 | -0.1317 | -0.0136 | -0.0003 |
| 1.0 | 140.0 | **Target** | 0.5103 | 0.6104 | 0.0343 | -0.1149 | -0.0101 | -0.0094 |
| | | **Opt.** | 0.5087 | 0.6099 | 0.0345 | -0.1161 | -0.0101 | -0.0095 |

Note: The non-zero residuals in out-of-the-money strikes are driven by the omission of cross-Greeks (e.g., Vanna $\partial^2 C / \partial S \partial v$) and Volga ($\partial^2 C / \partial v^2$) in the Taylor expansion during simultaneous spot and variance moves, rather than pricing inaccuracy.

![Heston PnL Attribution](/assets/img/posts/heston/heston_pnl_attribution.png)

---

## 5. Appendix & Advanced Frontiers

### From Stochastic Volatility to Local Stochastic Volatility (LSV)
While pure stochastic volatility models capture forward volatility dynamics, they cannot generally reproduce an arbitrary market surface exactly. Dupire's local volatility model $dS_t = r S_t dt + \sigma_{LV}(S_t, t) S_t dW_t$ fits the vanilla smile but assumes volatility is a deterministic function of spot and time, leading to flat forward smile dynamics.

The Local Stochastic Volatility (LSV) framework addresses this by introducing a local leverage function $L(S_t, t)$:

$$ dS_t = r S_t dt + L(S_t, t)\sqrt{v_t} S_t dW_t $$

This approach matches the European vanilla market via the leverage function while maintaining the stochastic forward volatility dynamics required for pricing path-dependent exotics.

### Rough Volatility Paradigms
Empirical studies of high-frequency tick data by Gatheral, Jaisson, and Rosenbaum indicate that volatility is a fractional process. 

In rough volatility models, the driving fractional Brownian motion has a Hurst parameter $H < 1/2$ (typically around $H \approx 0.1$). This generates the power-law behavior of the implied volatility skew at short maturities, a feature that the standard Heston model cannot replicate without parameter jumps. A challenge with rough volatility is computational: the fractional process is non-Markovian, meaning the entire past path history must be tracked for simulation, increasing the computational cost of traditional PDE and Monte Carlo approaches.

---

## 6. References

* **Andersen, L. (2008)**. *Simple and efficient simulation of the Heston stochastic volatility model*. Journal of Computational Finance, 11(3), 1–42.
* **Broadie, M., & Jain, M. (2008)**. *Pricing and Hedging of Volatility and Variance Swaps Under Stochastic Volatility*. Journal of Computational Finance, 11(3), 7–34.
* **Carr, P., & Madan, D. (1998)**. *Towards a Theory of Volatility Trading*. In *Volatility: New Estimation Techniques for Pricing Derivatives*, R. Jarrow (ed.), Risk Publications. Available at: [http://faculty.baruch.cuny.edu/lwu/890/CarrMadan1998.pdf](http://faculty.baruch.cuny.edu/lwu/890/CarrMadan1998.pdf)
* **Demeterfi, K., Derman, E., Kamal, M., & Zou, J. (1999)**. *More Than You Ever Wanted to Know About Volatility Swaps*. Goldman Sachs Quantitative Strategies Research Notes.
* **Duan, J.-C., & Simonato, J.-G. (1998)**. *Empirical Martingale Simulation for Asset Prices*. Management Science, 44(9), 1218–1233.
* **Gatheral, J. (2006)**. *The Volatility Surface: A Practitioner's Guide*. John Wiley & Sons (Chapter 11: Variance Swaps).
* **Heston, S. L. (1993)**. *A Closed-Form Solution for Options with Stochastic Volatility with Applications to Bond and Currency Options*. The Review of Financial Studies, 6(2), 327–343.
* **Lyuu, Y.-D. (2016)**. *Stochastic-Volatility Models and Continuous-Time Derivatives Pricing*. Lecture Notes, Department of Computer Science & Information Engineering, National Taiwan University. Available at: [https://www.csie.ntu.edu.tw/~lyuu/finance1/2016/20160420.pdf](https://www.csie.ntu.edu.tw/~lyuu/finance1/2016/20160420.pdf).

### My Own Python Notebooks
* [Heston Stochastic Volatility - Part 1](https://github.com/jhqntdv/Heston-Stochastic-Volatility-Modeling/blob/main/Heston%20Vol/heston-main.ipynb)
* [Heston Stochastic Volatility - Part 2](https://github.com/jhqntdv/Heston-Stochastic-Volatility-Modeling/blob/main/Heston%20Vol/heston-main-2.ipynb)