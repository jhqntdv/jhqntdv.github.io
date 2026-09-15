---
title: Convertible Note Model Selection
date: 2026-08-19 00:00:00 +0000
categories: [Valuation]
tags: [Accounting, Valuation, Convertible Note]
description: Guide to Convertible Note Valuation Models and Accounting
pin: false
math: true
---

## 1. Accounting Summary: ASC 470/815 vs ASC 825

When dealing with convertible notes, companies must decide on the appropriate accounting treatment. The two primary frameworks are the bifurcation method (ASC 470/815) and the Fair Value Option (ASC 825).

| Comparison Metric | ASC 470 / 815 (Bifurcation Method) | ASC 825 (Fair Value Option - FVO) |
| :--- | :--- | :--- |
| **Income Statement Volatility** | Lower (the host debt is amortized at a fixed effective rate, while the bifurcated embedded derivative is still marked to fair value through earnings each period) | Higher (fair value fluctuations of the entire instrument hit earnings) |
| **Valuation Complexity** | High | Low |
| **Accounting Burden for Modifications** | High | Low |
| **Financial Statement Presentation** | Complex (separates host debt and embedded derivative on the balance sheet and in footnotes) | Simplified (single liability line item, integrated with ASC 820 disclosures) |

**Conclusion & Strategic Insights:**
- If an enterprise wishes to **smooth interest expense on the income statement** and can handle the complex valuation and accounting assessments, the **ASC 470/815 Bifurcation Method** is appropriate.
- The ASC 815-40 "fixed-for-fixed" test determines whether a conversion option is indexed to the issuer's own stock. Conversion terms whose ratio floats based on a *future, external* event — most notably **Valuation Caps** and **Qualified Financing Discounts (QFD)**, common in early-stage/private convertible notes — still fail this test and remain subject to bifurcation under ASC 815-15.

---

## 2. AICPA PE/VC Guide

The **AICPA Accounting and Valuation Guide: Valuation of Portfolio Company Investments of Venture Capital and Private Equity Funds and Other Investment Companies (PE/VC Guide)** provides best practices (the "how") for estimating the fair value of investments.

Valuation specialists reference the AICPA PE/VC Guide when:
- Determining the fair value of a convertible note under **ASC 825 (Fair Value Option)**.
- Valuing the standalone embedded derivative required to be bifurcated under **ASC 815**.
- Assessing the fair value of the host debt under **ASC 470**.
- Justifying the selection of valuation models, such as Binomial and Monte Carlo.

---

## 3. Common Four Approaches to Valuing Convertible Notes

Depending on the complexity of the instrument and underlying assumptions, valuation specialists typically employ one of four primary methodologies:

1. **Bond Plus Call Option Model**
2. **As-Converted Plus Risky Put Option Model**
3. **Binomial / Lattice Model**
4. **Monte Carlo Simulation**

These approaches are used across accounting and financial reporting functions (e.g., Big 4, corporate finance teams, and controllers).

---

## 4. Details of the Approaches and When to Use Them

According to AICPA guidance (the *PE/VC Guide* and the *Cheap Stock Guide*), valuation specialists select specific quantitative models based on the following dimensions:

### 4.1 Moneyness (ITM vs. OTM)
- **Deep Out-of-the-Money (Deep OTM):** conversion probability is extremely low; the note's economics are close to a straight bond. Specialists typically use the **Bond Plus Call** combination model — host debt valued on credit risk/market yield, plus a low-value call option under Black-Scholes.
  $$ V_{CN} = \text{PV}(\text{Principal} + \text{Interest}) + n \cdot C_{BSM}(S, K, t, r, \sigma) $$
- **Deep In-the-Money (Deep ITM):** conversion is nearly certain; the note's economics approach equity. Specialists lean toward the **As-Converted Plus Risky Put** model — the underlying treated as converted equity, plus a put option providing downside protection.
  $$ V_{CN} = n \cdot S + n \cdot P_{BSM}(S, K, t, r, \sigma) $$

### 4.2 Path-Dependency & Volatility
- **High Volatility & Path-Dependent:** startup notes often carry floating conversion prices tied to future financing size, Valuation Caps, or Qualified Financing Discounts (QFD) — terms whose value depends on the sequence of future triggering events. **Monte Carlo Simulation** is the industry standard here.
- **Early Exercise / American Option Features:** callable or puttable notes call for a **Binomial Model**.

### 4.3 Underlying Asset Properties (Common vs. Preferred Shares)
- **Underlying is Common Shares:** for simple capital structures or liquid public companies, stock value can follow a single Geometric Brownian Motion. The traditional **Binomial Model** or **Bond Plus Call** is generally applicable.
- **Underlying is Preferred Shares:** private-company notes often convert into the next round's preferred stock, which carries liquidation preferences and participation rights that make its value non-linear. Specialists typically use the **Option Pricing Method (OPM)** to allocate enterprise value across equity classes, then layer this into a **Monte Carlo Simulation**.

## 5. Embedded Derivative: The With-and-Without Method
 
The most common technique specialists use to isolate the embedded derivative's value is the **with-and-without method**. The **"With" value** represents the fair value of the entire hybrid instrument and the **"Without" value** represents the fair value of a hypothetical plain-vanilla debt instrument. The residual is the value of the embedded derivative which represents the value of the optionality embedded in the note. If the convertible note has multiple features, the residual reflects the net value of all embedded features.

 
### Can the embedded derivative be negative or zero?
For a standalone conversion feature, the embedded derivative cannot be negative — it is essentially a call option on the underlying stock, and a call option's value is always greater than or equal to zero. However, if the convertible note bundles multiple features together, the embedded derivative can come out negative, for two reasons.

First, an unfavorable feature can outweigh the favorable conversion feature. For example, an issuer soft-call right that lets the company redeem the note at par once the stock trades above a threshold effectively caps the holder's upside, offsetting some or all of the conversion option's value.

Second — and more commonly in practice — a negative result comes from inconsistent inputs across the with-and-without split, often surfacing during specialist-management discussions. For example, using a distressed credit spread to discount the "without" (straight debt) leg while implicitly assuming improved credit quality in the "with" leg, or modeling the "with" leg to a short expected time-to-next-financing while the "without" leg runs to the note's full contractual maturity — either mismatch can produce a distorted, sometimes negative, residual that isn't really about optionality at all.

### What does the embedded derivative represent when the note is deep ITM?
The embedded derivative here is overwhelmingly the excess of as-converted equity value over the straight-debt value.