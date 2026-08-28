---
title: Machine Learning Cheat Sheet
date: 2026-08-26 00:00:00 +0000
categories: [Machine Learning]
tags: [Machine Learning]
description: Machine Learning Cheat Sheet
pin: false
math: true
---

## Regularization (L1 vs. L2)

Regularization adds a penalty term to the loss function to prevent overfitting:

* **L1 Regularization (Lasso)**: Adds absolute value penalty ($\lambda \sum |w_i|$). Shrinks coefficients strictly to zero $\implies$ **Can be used for feature selection** (creates sparsity).
* **L2 Regularization (Ridge)**: Adds squared penalty ($\lambda \sum w_i^2$). Shrinks coefficients close to zero but never to zero $\implies$ **Cannot perform feature selection** (keeps all features).
* **Elastic Net**: Combines both L1 and L2 penalties ($\lambda_1 \sum |w_i| + \lambda_2 \sum w_i^2$) for feature selection with correlated predictors.

## Model Evaluation Metrics

### 1. Classification Metrics & Confusion Matrix

* **Confusion Matrix**:

  | | Predicted Positive ($\hat{Y}=1$) | Predicted Negative ($\hat{Y}=0$) |
  | :--- | :--- | :--- |
  | **Actual Positive ($Y=1$)** | **TP** (True Positive) | **FN** (False Negative / Type II Error) |
  | **Actual Negative ($Y=0$)** | **FP** (False Positive / Type I Error) | **TN** (True Negative) |

* **Precision**: Proportion of predicted positives that are truly positive (avoids false positives, e.g., spam detection).
  $$ Precision = \frac{TP}{TP + FP} $$

* **Recall (Sensitivity / TPR)**: Proportion of actual positives that are correctly identified (avoids false negatives, e.g., disease detection).
  $$ Recall = \frac{TP}{TP + FN} $$

* **F1-Score**: Harmonic mean of Precision and Recall, balancing both especially on imbalanced datasets.
  $$ F1 = 2 \times \frac{Precision \times Recall}{Precision + Recall} $$

* **ROC Curve**: Plots True Positive Rate (TPR) vs. False Positive Rate (FPR) across different decision thresholds.
  $$ \text{TPR (Recall)} = \frac{TP}{TP + FN}, \quad \text{FPR} = \frac{FP}{FP + TN} $$

* **AUC**: Area under the ROC curve, measuring overall discrimination capability ($AUC = 1$: perfect classifier, $AUC = 0.5$: random guess).

### 2. Regression: MSE vs. MAE
* **Mean Squared Error (MSE)**: Penalizes large errors heavily (due to squaring).
  $$ MSE = \frac{1}{n} \sum_{i=1}^n (y_i - \hat{y}_i)^2 $$
  * *Choose MSE when*: Large errors are especially undesirable/costly, and the dataset is relatively clean with few extreme outliers.
* **Mean Absolute Error (MAE)**: Penalizes errors linearly and is robust to extreme values.
  $$ MAE = \frac{1}{n} \sum_{i=1}^n |y_i - \hat{y}_i| $$
  * *Choose MAE when*: The dataset has significant outliers or heavy-tailed noise, and you want predictions to be robust against extreme observations.


## Ensemble Learning

Ensemble learning combines multiple base models (weak learners) to produce a single stronger predictive model. The fundamental difference lies in their objective: **Bagging** trains independent models in parallel on random data subsets to reduce **variance** (prevent overfitting), while **Boosting** trains models sequentially to iteratively correct the errors of previous models to reduce **bias** (prevent underfitting).

* **Bagging (Bootstrap Aggregating)**: Trains multiple models in parallel on random bootstrap samples (with replacement) and aggregates predictions via averaging or voting.
  * Classic Application: Random Forest
* **Boosting**: Trains models sequentially where each subsequent model focuses on the residual errors of the previous ones.
  * Classic Application: Gradient Boosting