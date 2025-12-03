---
title: ASC 718 Valuation Overview
date: 2025-12-03 00:00:00 +0000
categories: [asc718]
tags: [saas]
description: ASC 718 - SBA Memo
pin: true
math: true
---


## Introduction

Under **ASC 718 – Compensation—Stock Compensation**, entities must measure equity-based compensation awards at their **grant-date fair value** and recognize compensation cost over the service period.

A valuation is generally required when:

- Awards contain market conditions or complex economic features (multi-tier hurdles, catch-up provisions...etc).
- Awards are issued by pass-through entities (LLCs or partnerships), where incentive units or profits interests must be measured at fair value.
- Existing awards are modified, requiring reassessment of fair value.
- Changes in capital structure materially affect award economics.

For service-based awards, compensation cost is recognized on a straight-line basis or according to the vesting schedule. For awards with market conditions, the full compensation cost is fixed at the grant date and recognized **regardless** of whether the market condition is met.

A **Monte Carlo simulation** is used to estimate fair value when payouts depend on market-based performance conditions. ASC 718 identifies Monte Carlo as the appropriate technique since it incorporates the probability distribution of future equity values. The model uses a Geometric Brownian Motion (GBM) process to project equity values.

### Math Expression
$$
\begin{equation}
  S_t = S_0 \exp\left[\left(\mu - \frac{1}{2}\sigma^2\right)t + \sigma W_t\right]
  \label{eq:gbm_formula}
\end{equation}
$$

$$
\begin{equation}
  W_t \sim \mathcal{N}(0, t)
  \label{eq:brownian_motion}
\end{equation}
$$

### Valuation Procedure

1. Simulate equity value paths over the expected term.
2. Incorporate award terms (waterfalls, vesting, triggers, etc.).
3. Evaluate payouts for each simulated path.
4. Average simulated payouts and discount to present value.

### Key Model Inputs

- Equity or common value  
- Volatility of the underlying equity  
- Risk-free interest rate  
- Expected time to liquidity  
- Distribution thresholds (strike prices, hurdles, IRR targets)  

---

## Technical Valuation Memo

If a company has a simple capital structure or senior preferences have limited impact, equity-based awards may be valued by applying Monte Carlo directly to common equity without modeling the full capitalization. Auditors often scrutinize this approach when the common price relies on older 409A valuations, as private-company common may not fully reflect capital structure economics.

Since equity-based awards behave like call options on common stock, their valuation can be interpreted using Black–Scholes “Greeks.” The two inputs with the largest impact are:

- **Underlying price (delta):** Higher underlying value increases the award value.  
- **Volatility (vega):** Higher volatility increases the award value.  

A common point of confusion is **intrinsic value vs. fair value**:

- **Intrinsic value** = max(common price – strike price, 0). Measures moneyness and immediate proceeds at liquidity.
- **Fair value** includes intrinsic value **plus** time value, volatility, term, and risk-free rates.

Longer remaining term increases the award’s time value and moves it closer to the underlying common value.

---

## How Our Platform Helps

Our platform uses AI to extract and structure a company’s cap table and dynamically build a waterfall model. It incorporates valuation-specialist logic to handle challenges such as dilution, equity volatility, asset volatility errors, and back-solving.

The pricing engine runs on a single platform, producing results in seconds at a reasonable cost. It is designed to cover the majority of plain to intermediate valuation needs, reducing reliance on external valuation firms.

For context, a standard ASC 718 valuation typically costs **$10,000–$15,000**, and can reach **$20,000** depending on complexity. Many awards with minimal option value or small ownership percentages still incur these costs. Our platform allows companies to capture fair value efficiently without unnecessary spending.

Valuation firms also use the platform, allowing continuous refinement based on real-world pain points. Users access the same tools and methodologies applied by valuation specialists.

---

## Target Audiences

As a fintech valuation platform, we focus on models and automation. We aim to be open-source-friendly, enabling users to work with a valuation firm or perform valuations internally. With the same inputs, users should expect results consistent with those from Big Four or specialist valuation firms.

Our target users include:

- Controllers and finance teams with experience across fundraising rounds  
- Companies not expecting major structural changes  
- Users who understand valuation inputs and can defend assumptions  
- Valuation firms without complex-instrument teams  

In reporting contexts, documenting and defending inputs is as important as the model itself. We do not provide sign-off services, but we can refer clients to partner valuation firms when a formal opinion is required.
