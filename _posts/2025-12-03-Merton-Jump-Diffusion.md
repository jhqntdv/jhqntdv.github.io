---
title: Merton-Jump-Diffusion Model
date: 2025-12-03 00:00:00 +0000
categories: [Quant]
tags: [Stochastic Process, Quant, MFin]
description: Jump Process
pin: true
math: true
---

## Introduction

In derivative markets, many asset pricing models are still based on the Black–Scholes (BS) framework. However, the Black–Scholes model is overly "fixed-parameterized": volatility is treated as constant, the price path is continuous, and returns follow a log-normal distribution. While this setup makes the model simple, it ignores many observed market phenomena, such as fat tails and high peaks in the return distribution, sudden price jumps, and news-driven price shocks.

To address this, academics and practitioners have introduced richer stochastic structures to describe asset prices. The two most typical approaches are:
1. **Stochastic Volatility (SV) Models**：Models like Heston and SABR. These can explain features like the volatility smile and the dynamics of the implied volatility surface. They are often preferred in practice because volatility is a continuously evolving state variable, making calibration stable and parameters interpretable.
2. **Jump Processes**：Models like the Merton Jump-Diffusion (MJD) and Kou Double-Exponential. These explicitly model the discrete jumps in asset prices. Jumps are particularly applicable in the credit market (e.g., default events, sudden changes in credit spreads). While pure Jump-Diffusion models (MJD) are rarely used exclusively, their concepts are embedded in both trading desk and risk management. For example hybrid frameworks like the Stochastic Volatility with Jumps (SVJ) model are used to accurately price derivatives by capturing both volatility dynamics and the market's implied volatility smile/skew. In risk management, one can see the jump component in Value-at-Risk (VaR) measurment.

## Poisson Process
The **Poisson Distribution**describes the number of times an event occurs within a unit time, with the core parameter being the intensity $\lambda$. Given a time interval of length $t$, the number of events $N_t$ in a Poisson process satisfies:

$$\mathbb{P}(N_t = k) = \frac{(\lambda t)^k e^{-\lambda t}}{k!}$$

The Poisson process has three main characteristics: 
- events are independent
- the probability of an event occurring in a unit time is proportional to the interval length
- there are no simultaneous jumps

For example, if $\lambda = \frac{5}{252}$, it means "an average of 5 jumps are expected per year." Therefore, the approximate daily jump probability is $\lambda \approx 0.02$.


## SDE of Stock with Jump
The classic Black–Scholes Stochastic Differential Equation (SDE) for asset prices is:

$$ dS_t = \mu S_t\,dt + \sigma S_t\, dW_t $$

Here, the drift term $\mu$ controls the average growth, and the diffusion term $\sigma$ controls the random volatility. By introducing a jump component, the Merton Jump-Diffusion (MJD) model extends the SDE to:

$$ dS_t = \mu S_t\,dt + \sigma S_t\, dW_t + S_{t^-}(J - 1)\, dN_t $$

Where: $N_t$: The Poisson process, which dictates whether a jump occurs. $J$: The jump magnitude, usually assumed to be log-normally distributed. $S_{t^-}$: The stock price just before the jump. $(J - 1)\, dN_t$: The multiplicative change in the asset's value when a jump occurs.

Intuitively, this SDE means the price typically evolves by continuous diffusion, but when $dN_t = 1$, the price suddenly multiplies by $J$, causing a jump. The jump size and frequency are entirely controlled by extra parameters, allowing the model to better reflect discrete shocks like news announcements or credit events.


## Option Pricing under MJD
Merton demonstrated that, even with the inclusion of jumps, the price of a European option can still be written in a "semi-closed form" solution. The method treats the option price as a weighted sum of Black–Scholes prices, conditional on the number of jumps that occur in a continuous diffusion process:

$$
C(S_0,K,T)
=
\sum_{k=0}^{\infty}
\frac{e^{-\lambda' T}(\lambda' T)^k}{k!}
\, C_{\text{BS}}\!\left(S_0, K, T, \sigma_k\right)
$$

Where:
- $\lambda' = \lambda (1 + \mu_J)$: The risk-neutral jump intensity
- $\sigma_k^2 = \sigma^2 + \frac{k \delta_J^2}{T}$: The effective volatility caused by jumps
- $C_{\text{BS}}(\cdot)$: The standard Black–Scholes European Call Option price

This structure implies that the jump model can be computed by mixing multiple BS models, avoiding the need for complex simulation (Monte Carlo) or Partial Differential Equation (PDE) methods.

### Why are Jump Parameters not Often Directly Estimated by Practitioners?
Empirical research shows that:
- Jump frequency and jump size are difficult to separate from daily price data.
- "Abnormal data points" in log returns could be either a jump or a sudden spike in volatility. 
- Parameters are sensitive to the sample interval, unstable, and have large estimation errors (findings consistent with research by Bates 1996, among others).
- Market microstructure noise can significantly interfere with jump test statistics.

Therefore, most practical teams calibrate jump parameters using the option market (implied volatility data) rather than inferring them solely from historical prices.

### Limitations of the Model
Classic research (e.g., Andersen & Benzoni 2008) points out that the Merton model can improve the fit of short-term option implied volatility, especially when event risk is high. However, it fits the long-term term structure poorly, as the assumption of Independent and Identically Distributed (IID) jumps cannot explain the shape of the long-term volatility surface. Empirically, the fit improvement is evident for short tenors ( $\le$ 1 month), but the error remains significant for options with more than a year to expiration. Consequently, while MJD is very effective in theoretical and short-term market environments, **Stochastic Volatility (SV) models remain the mainstream** for comprehensive modeling. Jump models are typically used as a complement or as part of a hybrid model (such as the SVJ model).

## Convertible Bond Pricing under MJD
The valuation model utilizing the Merton Jump-Diffusion (MJD) framework is an advancement of the widely-adopted Tsiveriotis-Fernandes (TF) model (1998). The TF model simplifies the problem by splitting the CB into a "bond component" (subject to a risky discount rate) and an "equity component" (discounted at the risk-free rate). The key assumption that makes the model popular but flawed is the stock price is unaffected by the issuer's default. On the other hand, MJD captures these elements:
- Volatility Smile/Skew: The jump component captures the fat-tailed, skewed nature of equity returns, improving the pricing of the embedded option
- Stock-Price Discontinuity (Jumps): It explicitly models sudden, large price movements (up or down) driven by news or events
- Equity-to-Credit Linkage: MJD framework links the possibility of a stock price jump to a default event that result in sudden change in the equity value portion of the convertible.

### Trinomial lattice (tree) model
The convertible bond pricing under MJD is implemented through trinomial lattice (tree) as it captures early exercise features and the mechanism of default risk. For the stock tree, each node can be transitioned immediately to three states with derived probability under risk-neutral framewrok: the price going up $p_u$, the price going down $p_d$, and the price got hit by credit event $p_{\lambda _D}$. The payoff of the convertible bond on each node will also be modeled accordingly that reflects the stock movements as well as issuer's and holder's optimal decision (conversion, redemption, callable, puttable..etc).

The key parameters include:

- Volatility $\sigma$: Calibrated from the implied volatility of traded options or historical volatility.

- Risk-Free Rate $r$: Observed from the government bond yield curve.

- Credit Spread (Hazard Rate) $\lambda_D$: In reduced-form models, this intensity (or hazard rate) is backed out from the issuer's Credit Default Swaps (CDS) spread or their non-convertible corporate bond yields.

- Recovery Rate $\eta$: Assumed percentage of par value (or market value) recovered in the event of default.Jump

- Default Size $1 - \theta$: The assumed percentage drop in stock price upon default. Setting to $\theta = 1$ means no jumps