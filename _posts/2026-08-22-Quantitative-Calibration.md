---
title: Quantitative Approach in Option Pricing Models (OPM)
date: 2026-08-22 00:00:00 +0000
categories: [Quant]
tags: [Stochastic Process, Quant, MFin, Calibration]
description: A quantitative approach to derive price, class-specific volatility, and delta simultaneously.
pin: false
math: true
---

## Introduction: A Quantitative Calibration Approach

The purpose of Option Pricing Model (OPM) analysis is typically to estimate the value of each class of equity in a private company. Specialists often perform this sequentially in Excel. However, this type of analysis can be formulated into a system of non-linear equations by treating equity as a value-weighted sum of all classes. 

**Different from the typical approach where specialists start from equity value and perform allocation to each class, the proposed approach starts from common stock price and makes sure the aggregated value sums up to the total equity value.**

This approach can derive price, class-specific volatility, and delta simultaneously. It translates the following principles into a programmatic system of equations:

1. Each security can be formulated as a mix of Black-Scholes call and put options with the underlying being the common stock price.
2. The value of each class sums to the total equity value.
3. The equity value should follow a log-normal distribution, and the equity volatility and common volatility relationship remains.

Note that this setup inherently derives class-specific delta and volatility. Specialists often neglect the fact that under the OPM framework, Merton's model already determines the link between common volatility, equity volatility, and the distribution of total equity value.

### The Objective Function

The objective function for this calibration can be written as:

$$ w_1 \left( \text{TEV} - \sum_{i} V_i \right)^2 + w_2 \left( \sigma_E - \frac{1}{\frac{\partial V_c}{\partial V}} \cdot \frac{V_c \, \sigma_c}{\sum_{i} V_i} \right)^2 $$

where:
- $\text{TEV}$ and $\sigma_E$ are the inputs (Total Equity Value and Total Equity Volatility)
- $V_i$ is the value of each equity class $i$ using common stock as the underlying
- $V_c$ is the value of common stock
- $\sigma_c$ is the common stock volatility
- $w_1, w_2$ are the weights for each component of the objective function

## Use Cases for the Proposed Method

The proposed method is particularly useful when:
- **Valuing multiple tranches of profits interests**, especially when thresholds are quoted on a common stock basis.
- **Handling capital structures** that consist of hybrid securities.
- **Applying marketability discounts**, where class-specific volatility is a required input for each security class.
- **Evaluating equity instruments with complex payoff structures**, such as when vesting percentages require linear interpolation.
- **Anchoring to a transaction common stock price** where a back-solve for implied equity value is required.

---

## Background Knowledge

To fully appreciate the calibration method above, it is helpful to review the foundational concepts of the Merton model, asset volatility, and how volatility behaves across different equity classes.

### Merton Model and Asset Volatility

The Merton model, developed by Robert C. Merton in 1974, treats a company's equity as a European call option on its total assets. The strike price of this option is the face value of the company's debt. This framework provides a structural link between the value of a firm's assets, its debt, and its equity. Specifically, the method provides a conversion between equity volatility and asset volatility.

**Asset Volatility** represents the true, underlying business risk of the company's operations, independent of its capital structure. 

**Equity Volatility** is the volatility of the stock price. It incorporates both the business risk (asset volatility) and the financial risk arising from leverage. Because debt acts as fixed financial leverage, equity is generally riskier and more volatile than the underlying assets.

### Application of the Merton Model for Private Companies

When performing equity-related instrument valuations for private companies with complex capital structures, the standard approach is to adopt the Option Pricing Model (OPM) framework where specialists treat total equity value as the underlying in the model and allocate the total equity value to different classes. Since the equity volatility, one of the key inputs in the model, is not market observable for private companies, specialists typically use market comparables with the Merton model to derive the company's equity volatility. 

Using common volatility or asset volatility intentionally or as a proxy is not appropriate since they do not reflect the true risk (capital structure and leverage) of the equity holder.

### The Relationship Between Equity Volatility and Common Volatility

The relationship between equity volatility and common stock volatility can be expressed mathematically:

$$ \sigma_C = \sigma_V \times \frac{\partial C}{\partial V} \times \frac{V}{C} $$

where $V$ is the total equity value, $C$ is the common stock value, $\sigma_V$ is the total equity volatility, and $\sigma_C$ is the common stock volatility. The term $\Omega = \frac{\partial C}{\partial V} \times \frac{V}{C}$ is the elasticity or "omega" of the option. 

As a company gets more leveraged (e.g., by issuing more debt or senior preferred shares), the ratio $V/C$ increases faster than the delta $\frac{\partial C}{\partial V}$ changes, leading to a higher $\Omega$. Consequently, **as leverage increases, the volatility of the common stock ($\sigma_C$) increases** relative to the underlying equity/asset volatility ($\sigma_V$). 

The key takeaway is that the "leverage" of common stock can be measured, and in fact, the same expression applies to all equity securities (Preferred Shares, Profits Interest Units, Warrants).

### Estimation of Class-Specific Volatility 

In practice, specialists segregate class-specific volatility by looking at the leverage ratio of each security. The method is typically referred to as delta-adjustment. 

In terms of dynamics, the class volatility changes with the company's volatility in the same direction but with a different magnitude. Also, the aggregated movement of each class ties back to the movement of equity following a log-normal distribution and the aggregate equity volatility.

The traditional process of estimating class-specific volatility typically involves building capital structure models, analyzing the breakpoints, deriving the value of each class, and applying the delta adjustment method. 
