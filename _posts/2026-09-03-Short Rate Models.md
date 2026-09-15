---
title: Short Rate Models and Curve Fitting
date: 2026-09-07 00:00:00 +0000
categories: [Quant]
tags: [Interest Rate, BDT, Hull White, Black-Karasinski, Black-Derman-Toy, Cubic Spline, Nelson-Siegel-Svensson]
description: Short Rate Model Comparisons
pin: true
math: true
---
![Short Rate Models](/assets/img/posts/short-rate/bdt.png)

## Introduction

This document compares three foundational one-factor short-rate models: **Black-Derman-Toy (BDT)**, **Hull-White (HW)**, and **Black-Karasinski (BK)**. It highlights their structural trade-offs across distribution assumptions, mean-reversion mechanics, calibration speed, and practical implementation limits.

**Refer to Interest Rate Modeling Notebooks:**
* [Nelson-Siegel-Svensson and Cubic Spline Curve Fitting](https://github.com/jhqntdv/Interest-Rate-Modeling/blob/main/nss-spline-main.ipynb)
* [Short Rate Models Notebook](https://github.com/jhqntdv/Interest-Rate-Modeling/blob/main/bdt-main.ipynb)

## Short Rate Models

### 1. Executive Summary Table

| Dimension | Black-Derman-Toy (BDT, 1990) | Hull-White (HW, 1990) | Black-Karasinski (BK, 1991) | Practical & Trading Impact |
| :--- | :--- | :--- | :--- | :--- |
| **Short-Rate SDE** | $d\ln(r) = \left[\theta(t) + \frac{\sigma'(t)}{\sigma(t)}\ln(r)\right]dt + \sigma(t)dW_t$ | $dr(t) = \left[\theta(t) - a(t)r(t)\right]dt + \sigma(t)dW_t$ | $d\ln(r) = \left[\theta(t) - a(t)\ln(r)\right]dt + \sigma(t)dW_t$ | BDT and BK model $\ln(r)$; HW models $r$ directly in nominal terms. |
| **Distribution** | Lognormal ($r > 0$ strictly; vol scales with rate level: $\sigma_{\text{bp}} \approx \sigma_{\text{LN}} \cdot r$) | Normal / Gaussian ($r \in (-\infty, +\infty)$; absolute vol is rate-independent) | Lognormal ($r > 0$ strictly; vol scales with rate level: $\sigma_{\text{bp}} \approx \sigma_{\text{LN}} \cdot r$) | Lognormal (BDT, BK) preferred when rates > 3% as basis-point vol scales with rates; Normal (HW) required when rates are near zero or negative. |
| **Negative Interest Rates** | No | Yes | No | HW is required in negative rate regimes; BDT and BK fail if rates are non-positive. |
| **Mean Reversion** | No | Yes | Yes | BDT drift is tied to volatility ($\sigma'/\sigma$); HW and BK have explicit, independent mean reversion parameter $a$. |
| **Branching Probabilities** | Fixed ($p = 0.5$) | State-dependent | State-dependent | HW and BK adjust probabilities to enforce drift without lattice distortion. |
| **Long-Horizon Stability (10Y-30Y)** | Lack of mean reversion; not practical | Viable | Viable | For long-dated contracts, desks avoid BDT in favor of HW or BK. |
| **Calibration Complexity** | 2D non-linear solver per time step | Analytical (no numerical solver needed) | 1D numerical solver per time step | HW is fastest; BK solves 1D shift per slice; BDT solves 2D system per step. |

### 2. The Evolution: Why Black & Karasinski Created the BK Model

In 1991, Fischer Black and Piotr Karasinski addressed a fundamental structural limitation in the original Black-Derman-Toy (BDT) model. In BDT, the drift coefficient governing $\ln(r)$ is tied directly to the term $\frac{\sigma'(t)}{\sigma(t)}$. Consequently, whenever market volatility is flat over time ($\sigma'(t) = 0$), the model's effective mean-reversion speed drops to zero. Over long maturities (10 to 30 years), the absence of mean reversion causes the lognormal rate distribution to diffuse uncontrollably, yielding unrealistic interest rate scenarios in the tails.

The Black-Karasinski (BK) model resolves this limitation by introducing an explicit, independent mean-reversion speed parameter $a(t)$ alongside the time-dependent level $\theta(t)/a(t)$. By decoupling mean reversion from volatility, BK preserves strictly positive interest rates while introducing the stabilizing pull characteristic of the Hull-White model.

### 3. Transition Probabilities & Lattice Construction

The geometric construction of the underlying lattice differs substantially across the three models:

* **Black-Derman-Toy**: Built on a standard recombining binomial tree with fixed branching probabilities ($p = 0.5$) at every node. Rates at each time step follow a geometric progression, with the vertical spacing determined entirely by calibration to market yields and volatilities.
* **Hull-White**: Constructed on a uniform arithmetic trinomial grid for the nominal short rate $r$. The mean-reverting drift $-a \cdot r$ is enforced by dynamically adjusting the branching probabilities $(p_u, p_m, p_d)$ across states, shifting downward at high interest rate levels to prevent rates from exploding.
* **Black-Karasinski**: Implemented using a trinomial grid in log-space ($x = \ln r$). The state-dependent branching probabilities absorb the mean-reverting drift $-a \cdot x$, and the node values are mapped back to nominal discounting rates via $r = \exp(x)$.

#### Arrow-Debreu Forward Induction

Arrow-Debreu forward induction is a sequential calibration technique that propagates state prices level by level through the lattice. An Arrow-Debreu price represents the present value at time zero of a contract that pays \$1 if a specific node is reached at a future time step and \$0 otherwise.

In traditional backward induction, calibrating interest rates at step $t$ would require repeatedly rolling back prices across the entire tree from the horizon, leading to computationally prohibitive global root-finding. By contrast, Arrow-Debreu forward induction works forward in time: once state prices are determined up to level $t-1$, the price of any zero-coupon bond maturing at step $t$ can be expressed as a simple dot product of the known state prices and the discount factors at step $t$. This isolates calibration to a localized, one-step numerical problem at each time slice, dramatically reducing overall computational complexity to $O(n^2)$ while guaranteeing an exact fit to the market term structure.

### 4. Implementation and Computation Speed: BDT vs. BK

In terms of engineering implementation and computational speed, BDT and BK present distinct operational trade-offs stemming from their lattice geometry and parameter calibration.

| Dimension | Black-Derman-Toy (BDT) | Black-Karasinski (BK) |
| :--- | :--- | :--- |
| **Lattice Structure** | Standard Uniform Binomial Tree (fixed $\Delta t$) | Non-uniform time-step Binomial Tree, or Trinomial Tree |
| **Calibration Algorithm** | Forward Induction / State Prices (Arrow-Debreu) | Numerical root-finding iteration or 2D grid calibration |
| **Implementation Complexity** | Moderate | High |
| **Calibration & Pricing Speed** | Fast | Slow |

#### Tree Construction Geometry

With equal time steps (fixed $\Delta t$), BDT nodes naturally recombine geometrically ($r_{ud} = r_{du}$), allowing node rates at each time step to be expressed analytically as a geometric sequence. This makes the lattice geometry highly uniform and straightforward to implement. Conversely, enforcing both an independent mean-reversion speed $a(t)$ and recombining nodes within a binomial tree forces the BK model to use non-uniform time steps $\Delta t_i$. This irregular spacing complicates cash flow interpolation for fixed coupon dates. In practice, BK is usually implemented via Hull-White style trinomial trees or PDE solvers, which require additional branching logic and boundary handling to guard against negative probabilities.

#### Algorithmic Complexity & Speed

BDT leverages Arrow-Debreu forward induction to isolate calibration to a stable two-dimensional root-solver per time step, maintaining an $O(n^2)$ runtime and minimal memory requirements. BK requires calibrating multiple time-dependent parameters $(\theta(t), a(t), \sigma(t))$, which typically involves outer optimizers coupled with inner backward induction or multidimensional root-finding. On trinomial lattices, each node evaluates three branching paths rather than two, making numerical calibration noticeably slower and more computationally intensive than BDT.

### 5. Decision Matrix: When to Use Which Model

1. **Black-Derman-Toy (BDT):** Choose when you are pricing short- to medium-term contracts ($T \le 5\text{ years}$). Interest rates are comfortably positive ($> 3\%$). You are working with classical binomial lattice architectures where only yield volatilities (not full swaption surfaces) are available.
2. **Hull-White (HW):** Choose when you need maximum computational speed and analytical tractability for vanilla instruments (Caps/Floors, European Swaptions). You must support zero or negative interest rates (e.g., EUR, JPY, CHF post-2014). You are calibrating to large market swaption volatility cubes. You are pricing complex Bermudan swaptions or callable floaters across long horizons (10Y-30Y).
3. **Black-Karasinski (BK):** Choose when you require strictly positive interest rates ($r > 0$) while avoiding the long-horizon volatility explosion of BDT. You are pricing long-dated callable debt (10Y-30Y callable bonds, step-up floaters, municipal bonds) in positive-rate currency regimes (e.g., USD, emerging markets). You want volatility to scale with the level of interest rates, reflecting realistic proportional volatility dynamics across different economic cycles.

### 6. Market Calibration & Data Preparation

In trading environments, short-rate models are typically calibrated using implied volatilities from OTC interest rate derivatives like caps, floors, and swaptions. However, maintaining a clean, stripped caplet-level volatility surface requires expensive data subscriptions (e.g., Bloomberg VCUB, LSEG/Refinitiv) and sophisticated processing infrastructure. For illustration purposes, and as is standard practice for many bank risk and Asset Liability Management (ALM) teams, we will use US Treasury data as our calibration benchmark for the BDT model. 

While using historical Treasury data is a generally accepted proxy for risk-modeling, the data preparation process requires a few critical considerations:

* Zero-Coupon Curve Construction: Treasury CMT rates are par yields, but the BDT model requires zero-coupon (spot) rates. Instead of using iterative bootstrapping (which strips zero rates sequentially), a robust alternative is the Nelson-Siegel-Svensson (NSS) parametric model, which fits a functional form to the instantaneous forward-rate curve. To avoid smoothing data that has already been smoothed by the Treasury (the CMT curve), the cleanest approach is to use the Federal Reserve's Gürkaynak-Sack-Wright (GSW) dataset, which publishes daily NSS zero-coupon curves fitted directly to underlying Treasury bond prices.
* Lognormal Volatility Alignment: BDT assumes a lognormal distribution, meaning it requires *relative* volatility ($\Delta \ln(y_t)$), not absolute normal volatility ($\Delta y_t$). This distinction is critical in near-zero interest rate environments (like 2020-2021) where relative changes can numerically explode. Practical implementations often apply a floor to yields before taking the log to prevent instability.
* Annualization & Lookback Windows: Volatility must be correctly annualized by scaling with the square root of time (e.g., multiplying by $\sqrt{252}$ for daily data, mirroring the $\sqrt{\Delta t}$ scaling inside the model). Furthermore, the choice of the historical lookback window drastically shifts volatility estimates across different policy cycles. A rolling window is generally preferred to capture dynamic market regimes.
* Physical vs. Risk-Neutral Measure: Using historical yield standard deviations measures *physical* (realized) volatility, whereas calibrating to market options measures *risk-neutral* (implied) volatility. While historical volatility is perfectly adequate for internal ALM and scenario generation, it will systematically misprice market-traded options that contain a volatility risk premium.

### 7. Curve Construction: Nelson-Siegel-Svensson vs. Cubic Spline

#### 7.1 Nelson-Siegel-Svensson (NSS): Parametric Smoothing

If you fit every market point directly, the resulting instantaneous forward rate curve often exhibits wild, un-economic oscillations. NSS solves this by imposing a rigid, economically motivated mathematical shape that filters out noise and guarantees asymptotic stability across long horizons.

##### Mathematical Formulation
The complete model is represented by 6 parameters:

$$y(t) = \beta_0 + \beta_1 \left(\frac{1 - e^{-t/\tau_1}}{t/\tau_1}\right) + \beta_2 \left(\frac{1 - e^{-t/\tau_1}}{t/\tau_1} - e^{-t/\tau_1}\right) + \beta_3 \left(\frac{1 - e^{-t/\tau_2}}{t/\tau_2} - e^{-t/\tau_2}\right)$$

| Parameter | Factor Interpretation | Asymptotic Behavior & Economic Role |
| :--- | :--- | :--- |
| **$\beta_0$** | **Level** (Long-term) | Long-term rate level ($\lim_{t \to \infty} y(t) = \beta_0$). |
| **$\beta_1$** | **Slope** (Short-term) | Short-term slope ($y(0) - y(\infty) = \beta_1$); decays to $0$ as $t \to \infty$. |
| **$\beta_2$** | **First Curvature / Hump** (Medium-term) | Medium-term hump or trough; peaks at $t \approx \tau_1$. |
| **$\beta_3$** | **Second Curvature / Twist** (Long-term) | Second hump or twist (long-end); peaks at $t \approx \tau_2$. |
| **$\tau_1, \tau_2$** | **Scale / Decay Parameters** | Maturities where each respective hump reaches its peak. |

The method does not fit all data points exactly and is not suitable for derivatives pricing.

![Short Rate Models](/assets/img/posts/short-rate/yield_curve_evolution.gif)

---

#### 7.2 Cubic Spline: Local Piecewise Interpolation

Cubic spline is a piecewise interpolation method that fits a 3rd-degree polynomial to the observed market yields:

$$S_i(t) = a_i + b_i(t - t_{i-1}) + c_i(t - t_{i-1})^2 + d_i(t - t_{i-1})^3$$

To ensure the pieces form a cohesive, smooth curve, the parameters are solved subject to:
1. **Exact Interpolation:** The spline passes through the market yields at each knot ($S_i(t_i) = y_i$).
2. **$C^0$ Continuity:** The adjacent curve segments meet continuously ($S_i(t_i) = S_{i+1}(t_i)$).
3. **$C^1$ Continuity:** The first derivatives match ($S_i'(t_i) = S_{i+1}'(t_i)$), ensuring a continuous forward rate curve.
4. **$C^2$ Continuity:** The second derivatives match ($S_i''(t_i) = S_{i+1}''(t_i)$), ensuring smooth rate curvature across maturities.

![Short Rate Models](/assets/img/posts/short-rate/yield_curve_evolution_cb.gif)